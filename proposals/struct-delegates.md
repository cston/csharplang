# Struct delegates

## Summary

Allow delegates and closures on the stack.
Allow allocation-free LINQ.

## Detailed design

Extend `allows ref struct` to parameters with `interface` and `delegate` types, and to method returns. Rather than introducing syntax for the new cases, assume an `AllowsRefStructAttribute` exists as an alternative.

Consider `Enumerable.Where<TSource>()`:

```csharp
static class Enumerable
{
    [return: AllowsRefStruct]
    static WhereEnumerable<TSource> Where<TSource>(
        [AllowsRefStruct] this IEnumerable<TSource> source,
        [AllowsRefStruct] Func<TSource, bool> predicate);
}
```

*Within the method body*, the compiler supports the following:
* An annotated parameter cannot be boxed or captured.
* Access to instance members on an annotated parameter are limited to members on the declared interface and base interfaces, and to `object.Equals(object)`.

The *runtime* supports the following:
* Allows taking a `ref` to an annotated parameter or field of an annotated parameter. The `ref` is either a managed reference to an instance on the heap, or a reference to the stack, depending on whether the parameter is a `ref struct`.
* Allows returning a `struct` rather than a `ref struct`.
* Allows a `ref struct` to be used instead of a `System.MulticastDelegate`.

The `ref struct` returned from `Enumerable.Where<TSource>()` is something like:

```csharp
public /*ref*/ struct WhereEnumerable<TSource> : IEnumerable<TSource>
    where TSource : allows ref struct
{
    private readonly Ref<IEnumerable<TSource>> _source;
    private readonly Ref<Func<TSource, bool>> _predicate;
}
```

The *runtime* treats `WhereEnumerable<TSource>` as a `struct`, and relies on the compiler to ensure that instances that hold references to the stack have a lifetime that does extend beyond the lifetime of the referenced items. That is, the compiler is responsible for ensuring the instance is treated as a `ref struct`.

A `Ref<T>` is either a managed reference to an instance on the heap, or a reference to the stack, depending on whether the type parameter is a `ref struct`.

*Describe the runtime handling of an annotated `delegate` parameter.*

The `Enumerable.Where<TSource>()` example does include complex relative lifetimes &mdash; the lifetime of the return value is simply assumed to be no longer than shortest lifetime of the arguments.