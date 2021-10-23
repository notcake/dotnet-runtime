# Struct Marshalling

As part of the new source-generated direction for .NET Interop, we are looking at various options for supporting marshaling user-defined struct types.

These types pose an interesting problem for a number of reasons listed below. With a few constraints, I believe we can create a system that will enable users to use their own user-defined types and pass them by-value to native code.

## Problems

- Unmanaged vs Blittable
  - The C# language (and Roslyn) do not have a concept of "blittable types". It only has the concept of "unmanaged types", which is similar to blittable, but differs for `bool`s and `char`s. `bool` and `char` types are "unmanaged", but are never (in the case of `bool`), or only sometimes (in the case of `char`) blittable. As a result, we cannot use the "is this type unmanaged" check in Roslyn for structures.
- Limited type information in ref assemblies.
  - In the ref assemblies generated by dotnet/runtime, we save space and prevent users from relying on private implementation details of structures by emitting limited information about their fields. Structures that have at least one non-object field are given a private `int` field, and structures that have at least one field that transitively contains an object are given one private `object`-typed field. As a result, we do not have full type information at code-generation time for any structures defined in the BCL when compiling a library that uses the ref assemblies.
- Private reflection
  - Even when we do have information about all of the fields, we can't emit code that references them if they are private, so we would have to emit unsafe code and calculate offsets manually to support marshaling them.

## Opt-in Interop

We've been working around another problem for a while in the runtime-integrated interop design: The user can use any type that is non-auto layout and has fields that can be marshaled in interop. This has lead to various issues where types that were not intended for interop usage became usable and then we couldn't update their behavior to be special-cased since users may have been relying on the generic behavior (`Span<T>`, `Vector<T>` come to mind).

I propose an opt-in design where the owner of a struct has to explicitly opt-in to usage for interop. This enables our team to add special support as desired for various types such as `Span<T>` while also avoiding the private reflection and limited type information issues mentioned above.

This design would use these attributes:

```csharp

[AttributeUsage(AttributeTargets.Struct | AttributeTargets.Class)]
public class GeneratedMarshallingAttribute : Attribute {}

[AttributeUsage(AttributeTargets.Struct | AttributeTargets.Class)]
public class BlittableTypeAttribute : Attribute {}

[AttributeUsage(AttributeTargets.Struct | AttributeTargets.Class)]
public class NativeMarshallingAttribute : Attribute
{
     public NativeMarshallingAttribute(Type nativeType) {}
}

[AttributeUsage(AttributeTargets.Parameter | AttributeTargets.ReturnValue | AttributeTargets.Field)]
public class MarshalUsingAttribute : Attribute
{
     public MarshalUsingAttribute(Type nativeType) {}
}
```

The `NativeMarshallingAttribute` and `MarshalUsingAttribute` attributes would require that the provided native type `TNative` is a blittable `struct` and has a subset of three methods with the following names and shapes (with the managed type named TManaged):

```csharp
partial struct TNative
{
     public TNative(TManaged managed) {}
     public TManaged ToManaged() {}

     public void FreeNative() {}
}
```

The analyzer will report an error if neither the construtor nor the ToManaged method is defined. When one of those two methods is missing, the direction of marshalling (managed to native/native to managed) that relies on the missing method is considered unsupported for the corresponding managed type. The FreeNative method is only required when there are resources that need to be released.


> :question: Does this API surface and shape work for all marshalling scenarios we plan on supporting? It may have issues with the current "layout class" by-value `[Out]` parameter marshalling where the runtime updates a `class` typed object in place. We already recommend against using classes for interop for performance reasons and a struct value passed via `ref` or `out` with the same members would cover this scenario.

If the native type `TNative` also has a public `Value` property, then the value of the `Value` property will be passed to native code instead of the `TNative` value itself. As a result, the type `TNative` will be allowed to be non-blittable and the type of the `Value` property will be required to be blittable. If the `Value` property is settable, then when marshalling in the native-to-managed direction, a default value of `TNative` will have its `Value` property set to the native value. If `Value` does not have a setter, then marshalling from native to managed is not supported.

A `ref` or `ref readonly` typed `Value` property is unsupported. If a ref-return is required, the type author can supply a `GetPinnableReference` method on the native type. If a `GetPinnableReference` method is supplied, then the `Value` property must have a pointer-sized primitive type.

```csharp
[NativeMarshalling(typeof(TMarshaler))]
public struct TManaged
{
     // ...
}

public struct TMarshaler
{
     public TNative(TManaged managed) {}
     public TManaged ToManaged() {}

     public void FreeNative() {}

     public ref TNative GetPinnableReference() {}

     public TNative Value { get; set; }
}

```

### Performance features

#### Pinning

Since C# 7.3 added a feature to enable custom pinning logic for user types, we should also add support for custom pinning logic. If the user provides a `GetPinnableReference` method that matches the requirements to be used in a `fixed` statement and the pointed-to type is blittable, then we will support using pinning to marshal the managed value when possible. The analyzer should issue a warning when the pointed-to type would not match the final native type, accounting for the `Value` property on the native type. Since `MarshalUsingAttribute` is applied at usage time instead of at type authoring time, we will not enable the pinning feature since the implementation of `GetPinnableReference` unless the pointed-to return type matches the native type.

#### Caller-allocated memory

Custom marshalers of collection-like types or custom string encodings (such as UTF-32) may want to use stack space for extra storage for additional performance when possible. If the `TNative` type provides additional members with the following signatures, then it will opt in to using a caller-allocated buffer:

```csharp
partial struct TNative
{
     public TNative(TManaged managed, Span<byte> buffer) {}

     public const int BufferSize = /* */;
    
     public const bool RequiresStackBuffer = /* */;
}
```

When these members are present, the source generator will call the two-parameter constructor with a possibly stack-allocated buffer of `BufferSize` bytes when a stack-allocated buffer is usable. If a stack-allocated buffer is a requirement, the `RequiresStackBuffer` field should be set to `true` and the `buffer` will be guaranteed to be allocated on the stack. Setting the `RequiresStackBuffer` field to `false` is the same as omitting the field definition. Since a dynamically allocated buffer is not usable in all scenarios, for example Reverse P/Invoke and struct marshalling, a one-parameter constructor must also be provided for usage in those scenarios. This may also be provided by providing a two-parameter constructor with a default value for the second parameter.

Type authors can pass down the `buffer` pointer to native code by defining a `GetPinnableReference()` method on the native type that returns a reference to the first element of the span. When the `RequiresStackBuffer` field is set to `true`, the type author is free to use APIs that would be dangerous in non-stack-allocated scenarios such as `MemoryMarshal.GetReference()` and `Unsafe.AsPointer()`.

### Usage

There are 2 usage mechanisms of these attributes.

#### Usage 1, Source-generated interop

The user can apply the `GeneratedMarshallingAttribute` to their structure `S`. The source generator will determine if the type is blittable. If it is blittable, the source generator will generate a partial definition and apply the `BlittableTypeAttribute` to the struct type `S`. Otherwise, it will generate a blittable representation of the struct with the aformentioned requried shape and apply the `NativeMarshallingAttribute` and point it to the blittable representation. The blittable representation can either be generated as a separate top-level type or as a nested type on `S`.

#### Usage 2, Manual interop

The user may want to manually mark their types as marshalable in this system due to specific restrictions in their code base around marshaling specific types that the source generator does not account for. We could also use this internally to support custom types in source instead of in the code generator. In this scenario, the user would apply either the `BlittableTypeAttribute` or the `NativeMarshallingAttribute` attribute to their struct type. An analyzer would validate that the struct is blittable if the `BlittableTypeAttribute` is applied or validate that the native struct type is blittable and has marshalling methods of the required shape when the `NativeMarshallingAttribute` is applied.

The P/Invoke source generator (as well as the struct source generator when nested struct types are used) would use the `BlittableTypeAttribute` and `NativeMarshallingAttribute` to determine how to marshal a value type parameter or field instead of looking at the fields of the struct directly.

If a structure type does not have either the `BlittableTypeAttribute` or the `NativeMarshallingAttribute` applied at the type definition, the user can supply a `MarshalUsingAttribute` at the marshalling location (field, parameter, or return value) with a native type matching the same requirements as `NativeMarshallingAttribute`'s native type.

All generated stubs will be marked with [`SkipLocalsInitAttribute`](https://docs.microsoft.com/dotnet/api/system.runtime.compilerservices.skiplocalsinitattribute) on supported frameworks. This does require attention when performing custom marshalling as the state of stub allocated memory will be in an undefined state.

### Why do we need `BlittableTypeAttribute`?

Based on the design above, it seems that we wouldn't need `BlittableTypeAttribute`. However, due to the ref assembly issue above in combination with the desire to enable manual interop, we need to provide a way for users to signal that a given type should be blittable and that the source generator should not generate marshalling code.

I'll give a specific example for where we need the `BlittableTypeAttribute` below. Let's take a scenario where we don't have `BlittableTypeAttribute`.

In Foo.csproj, we have the following types:

```csharp
public struct Foo
{
     private bool b;
}

public struct Bar
{
     private short s;
}
```

We compile these types into an assembly Foo.dll and we want to publish a package. We decide to use infrastructure similar to dotnet/runtime and produce a ref assembly. The ref assembly will have the following types:

```csharp
struct Foo
{
     private int dummy;
}
struct Bar
{
     private int dummy;
}
```

We package up the ref and impl assemblies and ship them in a NuGet package.

Someone else pulls down this package and writes their own struct type Baz1 and Baz2:

```csharp
struct Baz1
{
     private Foo f;
}
struct Baz2
{
     private Bar b;
}
```

Since the source generator only sees ref assemblies, it would think that both `Baz1` and `Baz2` are blittable, when in reality only `Baz2` is blittable. This is the ref assembly issue mentioned above. The source generator cannot trust the shape of structures in other assemblies since those types may have private implementation details hidden.

Now let's take this scenario again with `BlittableTypeAttribute`:

```csharp
[BlittableType]
public struct Foo
{
     private bool b;
}

[BlittableType]
public struct Bar
{
     private short s;
}
```

This time, we produce an error since Foo is not blittable. We need to either apply the `GeneratedMarshalling` attribute (to generate marshalling code) or the `NativeMarshallingAttribute` attribute (so provide manually written marshalling code) to Foo. This is also why we require each type used in interop to have either a `[BlittableType]` attribute or a `[NativeMarshallingAttribute]` attribute; we can't validate the shape of a type not defined in the current assembly because its shape may be different between its reference assembly and the runtime type.

Now there's another question: Why we can't just say that a type with `[GeneratedMarshalling]` and not `[NativeMarshallingAttribute]` has been considered blittable?

We don't want to require usage of `[GeneratedMarshalling]` to mark that a type is blittable because then there is no way to enforce that the type is blittable. If we require usage of `[GeneratedMarshalling]`, then we will automatically generate marshalling code if the type is not blittable. By also having the `[BlittableType]` attribute, we enable users to mark types that they want to ensure are blittable and an analyzer will validate the blittability.

Basically, the design of this feature is as follows:

At build time, the user can apply either `[GeneratedMarshallling]`, `[BlittableType]`, or `[NativeMarshallingAttribute]`. If they apply `[GeneratedMarshalling]`, then the source generator will run and generate marshalling code as needed and apply either `[BlittableType]` or `[NativeMarshallingAttribute]`. If the user manually applies `[BlittableType]` or `[NativeMarshallingAttribute]` instead of `[GeneratedMarshalling]`, then an analyzer validates that the type is blittable (for `[BlitttableType]`) or that the marshalling methods and types have the required shapes (for `[NativeMarshallingAttribute]`).

When the source generator (either Struct, P/Invoke, Reverse P/Invoke, etc.) encounters a struct type, it will look for either the `[BlittableType]` or the `[NativeMarshallingAttribute]` attributes to determine how to marshal the structure. If neither of these attributes are applied, then the struct cannot be passed by value.


If someone actively disables the analyzer or writes their types in IL, then they have stepped out of the supported scenarios and marshalling code generated for their types may be inaccurate.

#### Exception: Generics

Because the Roslyn compiler needs to be able to validate that there are not recursive struct definitions, reference assemblies have to contain a field of a type parameter type in the reference assembly if they do in the runtime assembly. As a result, we can inspect private generic fields reliably.

To enable blittable generics support in this struct marshalling model, we extend `[BlittableType]` as follows:

- In a generic type definition, we consider all type parameters that can be value types as blittable for the purposes of validating that `[BlittableType]` is only applied to blittable types.
- When the source generator discovers a generic type marked with `[BlittableType]` it will look through the fields on the type and validate that they are blittable.

Since all fields typed with non-parameterized types are validated to be blittable at type definition time, we know that they are all blittable at type usage time. So, we only need to validate that the generic fields are instantiated with blittable types.

### Special case: Transparent Structures

There has been discussion about Transparent Structures, structure types that are treated as their underlying types when passed to native code. The support for a `Value` property on a generated marshalling type supports the transparent struct support. For example, we could support strongly typed `HRESULT` returns with this model as shown below:

```csharp
[NativeMarshalling(typeof(HRESULT))]
struct HResult
{
     public HResult(int result)
     {
          Result = result;
     }
     public readonly int Result;
}

struct HRESULT
{
     public HRESULT(HResult hr)
     {
          Value = hr;
     }

     public HResult ToManaged() => new HResult(Value);
     public int Value { get; set; }
}
```

In this case, the underlying native type would actually be an `int`, but the user could use the strongly-typed `HResult` type as the public surface area.

> :question: Should we support transparent structures on manually annotated blittable types? If we do, we should do so in an opt-in manner to make it possible to have a `Value` property on the blittable type.

#### Example: ComWrappers marshalling with Transparent Structures

Building on this Transparent Structures support, we can also support ComWrappers marshalling with this proposal via the manually-decorated types approach:

```csharp
[NativeMarshalling(typeof(ComWrappersMarshaler<Foo, FooComWrappers>))]
class Foo
{}

struct ComWrappersMarshaler<TClass, TComWrappers>
     where TComWrappers : ComWrappers, new()
{
     private static readonly TComWrappers ComWrappers = new TComWrappers();

     private IntPtr nativeObj;

     public ComWrappersMarshaler(TClass obj)
     {
          nativeObj = ComWrappers.GetOrCreateComInterfaceForObject(obj, CreateComInterfaceFlags.None);
     }

     public IntPtr Value { get => nativeObj; set => nativeObj = value; }

     public TClass ToManaged() => (TClass)ComWrappers.GetOrCreateObjectForComInstance(nativeObj, CreateObjectFlags.None);

     public unsafe void FreeNative()
     {
          ((delegate* unmanaged[Stdcall]<IntPtr, uint>)((*(void***)nativeObj)[2 /* Release */]))(nativeObj);
     }
}
```

This ComWrappers-based marshaller works with all `ComWrappers`-derived types that have a parameterless constructor and correctly passes down a native integer (not a structure) to native code to match the expected ABI.