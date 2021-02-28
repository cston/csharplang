# Stack allocation of reference types

## Summary
Allow stack allocation of reference type instances that do not escape the allocation scope.

## Motivation
There are several common patterns where a reference type instance has a lifetime that is limited to the scope in which it is allocated. Some examples: boxing, `params` arrays, delegates, closure classes, and builders.
In those cases, the compiler should indicate to the runtime that the instances can be allocated on the callstack rather than the heap.

For a given application, the fraction of allocations moved to the callstack may be small, but in some cases the code changes required may be small too.
For calling code that relies on approriately annotated libraries, there may be fewer heap allocations without any code changes.

## Detailed design
The compiler will use type and method annotations and escape analysis to determine the lifetime of reference type instances.
If the compiler can determine the lifetime of an instance is limited to the scope in which it was allocated, the compiler will emit a hint to the runtime that the instance can be allocated on the stack.

### Attributes
This proposal assumes a new `LifetimeAttribute` for annotations.

```csharp
namespace System.Diagnostics.CodeAnalysis
{
    public enum LifetimeScope { Unlimited = 0, Container = 1, Related = 2 };

    public sealed class LifetimeAttribute : System.Attribute
    {
        public LifetimeAttribute(LifetimeScope scope, int related = 0)
        {
            Scope = scope;
            Related = related;
        }
        public readonly LifetimeScope Scope;
        public readonly int Related;
    }
}
```

A `class` or `struct`:
- annotated with `[Lifetime(LifetimeScope.Container)]` contains only instance methods annotated `[Lifetime(LifetimeScope.Container)]`;
- annotated with `[Lifetime(LifetimeScope.Unlimited)]` may contain instance methods annotated `[Lifetime(LifetimeScope.Unlimited)]`.

An instance method:
- annotated with `[Lifetime(LifetimeScope.Container)]` does not create references to `this`, or to owned fields reachable from `this`, other than on the callstack;
- annotated with `[Lifetime(LifetimeScope.Unlimited)]` may create references to `this`, or to owned fields reachable from `this`, outside of the callstack.

A parameter or method return value:
- annotated with `[Lifetime(LifetimeScope.Container)]` indicates that the containing method does not create references to the parameter or return value, or to owned fields reachable from the parameter or return value, other than on the callstack;
- annotated with `[Lifetime(LifetimeScope.Unlimited)]` indicates that the containing method may create references to the parameter or return value, or to owned fields reachable from the parameter or return value, outside the callstack;
- annotated with `[Lifetime(LifetimeScope.Related, parameters)]` indicates that the parameter or return value lifetime is no longer than the lifetime of the parameters in the `parameters` bit array where the 0-th bit represents `this`.

A field:
- annotated with `[Lifetime(LifetimeScope.Container)]` is an instance that has a lifetime no longer than the lifetime of the containing instance;
- annotated with `[Lifetime(LifetimeScope.Unlimited)]` is an instance that has a lifetime independent of the containing instance.

If a virtual method is overridden in an annotated derived type, the annotations on the derived method must be at least as strong as the annotations on the base method.
If an interface is implemented in a `class` or `struct`, the annotations on the implemented methods must be at least as strong as the annotations on the interface.

Lifetime attributes should be considered part of the contract of an annotated API and should be versioned along with the rest of the API contract.

An unannotated member is treated as `[Lifetime(LifetimeScope.Unlimited)]`. But unlike a member with an explicit `[Lifetime(LifetimeScope.Unlimited)]` annotation, an unannotated member is treated as _not annotated yet_. An unannotated member may be annotated in a later release without a breaking change.

### Escape analysis
The compiler analyzes the lifetime of objects in each method.

A few examples:
- if an object is assigned to a global instance or field reachable from a global instance, then the object lifetime is `Unlimited`;
- if an object is used as the receiver of an instance method annotated with `[Lifetime(LifetimeScope.Unlimited)]`, then the object lifetime is `Unlimited`;
- if an object lifetime is changed to `Unlimited`, then lifetime of any container through which the object is an owned field is also `Unlimited`.

If the compiler determines the lifetime of a reference does not match an associated annotation, an error is reported.

If the compiler can determine the lifetime of a reference type instance allocated in the current scope is limited to that scope, the compiler can mark the allocation as allowed on the callstack rather than the heap.

To determine that a reference type instance can be allocated on the stack, the compiler will typically require the following:
- the methods invoked within the scope of the allocation are annotated;
- the parameters and return types of those methods have lifetimes that are independent of the allocated instance or are limited to the lifetime of the instance; and
- the runtime types of all related arguments to methods are known at compile time.

Those requirements are significant - for a given project perhaps only a fraction of the source and assemblies will be annotated for instance - but there still should be useful scenarios where the requirements are met. Some potential examples are described later.

### Opt-in and opt-out
Types can be annotated separately and over multiple releases.
In practice, base types and interfaces may need to be annotated before derived types and interface implementations.

It should be possible to opt-out from (or opt-in to) runtime stack allocation of reference types.
Perhaps there is a `[MethodImpl(MethodImplOptions.NoStackReferenceTypes)]` attribute to opt-out in the calling method for instance.

It should be possible to assert that allocations occur on the stack rather than leaving that decision to the compiler and runtime.
Perhaps `stackalloc` rather than `new` could be allowed for those cases, at least for explicit allocations.
`stackalloc` could also be used for lambda delegates: `stackalloc x => x == y`.

_Can the format of the hints to the runtime be made compatible with earlier runtimes?_

### Example: Boxing
In the call to `Console.WriteLine()` below, the compiler boxes the `int` argument.

_Consider using a string interpolation example instead._

```csharp
// Console.WriteLine
public static void WriteLine(
    [Lifetime(LifetimeScope.Container)] string format,
    [Lifetime(LifetimeScope.Container)] object arg);

Console.WriteLine("{0}", 42);
```

The compiler allocates an `object` instance indirectly with:
```
ldc.i4 42
box System.Int32
```

The annotation on the `object arg` parameter to `Console.WriteLine()` indicates that no references to `arg` are created in `WriteLine()` that outlive the method call.

When compiling the implementation of `Console.WriteLine()` and helper methods, the compiler verified the annotation on `object arg` by determining that:
- the direct uses of `arg` in `WriteLine()` are as the receiver in calls to `Object.ToString()` and `IFormattable.ToString(string format, IFormatProvider formatProvider)` which are both annotated `[Lifetime(LifetimeScope.Container)]`; and
- the indirect uses of `arg` as arguments to helper methods in `WriteLine()` are to methods where the parameters are annotated `[Lifetime(LifetimeScope.Container)]`.

When compiling the call to `Console.WriteLine("{0}", 42)`, the compiler verified there are no references to the boxed instance outside of the invocation expression since that instance was a temporary created by the compiler.

The combination of the annotation on `WriteLine()` plus the (trivial) lifetime analysis of the local boxed instance means the compiler can safely mark the `box` instruction as a potential stack allocation.

### Example: `params` array
In the call to the overload of `Console.WriteLine()` that takes a `params object[]` argument, the compiler implicitly allocates an array.

```csharp
// Console.WriteLine
public static void WriteLine(
    [Lifetime(LifetimeScope.Container)] string format,
    [Lifetime(LifetimeScope.Container)] params object[] arg);

Console.WriteLine("{0}, {1}, {2}, {3}", a, b, c, d);
```

The compiler allocates an `object[]` instance with:
```
ldc.i4.4
newarr System.Object
```

The annotation on the `params object[] arg` parameter to `Console.WriteLine()` indicates that no references to the array are created in `WriteLine()` that outlive the method call.

When compiling the implementation of `Console.WriteLine()` and helper methods, the compiler verified the annotation on `params object[] arg` by determining that:
- the direct uses of `arg` in `WriteLine()` are as an argument to `ldlen` and `ldelem.ref` which do not affect lifetime; and
- the indirect uses of `arg` as arguments to helper methods in `WriteLine()` are to methods where the parameters are annotated `[Lifetime(LifetimeScope.Container)]`.

When compiling the call to `Console.WriteLine("{0}, {1}, {2}, {3}", a, b, c, d)`, the compiler verified there are no references to the array outside of the invocation expression since that array instance was a temporary created by the compiler.

The combination of the annotation on `WriteLine()` plus the trivial lifetime analysis of the local array instance means the compiler can safely mark the `newarr` instruction as a potential stack allocation.

_Can the runtime support stack allocation for a `newarr` instruction where the array size is a constant previously pushed onto the stack?_

### Example: Delegate and closure class
The `ContainsTypeParameter()` method below relies on a helper method `VisitType()` which invokes the delegate `predicate` on each component of `type`.

```csharp
// Returns first component of compound 'type' that satisfies 'predicate'.
[return: Lifetime(LifetimeScope.Related, parameters: 0b10)]
static TypeSymbol VisitType(
    [Lifetime(LifetimeScope.Container)] TypeSymbol type,
    [Lifetime(LifetimeScope.Container)] Func<TypeSymbol, bool> predicate)
{
    ...
}

// Returns true if 'type' contains 'typeParameter'.
static bool ContainsTypeParameter(TypeSymbol type, TypeParameterSymbol typeParameter)
{
    return VisitType(type, t => t == typeParameter) != null;
}
```

For the lambda expression in `ContainsTypeParameter()`, the compiler generates a closure class for `typeParameter` and an instance method on the closure class for the delegate passed to `VisitType()`.
The compiler allocates an instance of the closure class and a delegate instance for the argument to `VisitType()`.

```
.class <>c__DisplayClass1_0
{
    .field class TypeParameterSymbol typeParameter

    .method bool <ContainsTypeParameter>b__0(class TypeSymbol t) { ... }
}

.method ContainsTypeParameter(class TypeSymbol type, class TypeParameterSymbol typeParameter)
{
    newobj <>c__DisplayClass1_0::.ctor()
    ldftn <>c__DisplayClass1_0::<ContainsTypeParameter>b__0(class TypeSymbol)
    newobj System.Func`2<class TypeSymbol, bool>::.ctor()
    call VisitType()
    ...
}
```

The annotation on the `Func<TypeSymbol, bool> predicate` parameter to `VisitType()` indicates that no references to the delegate are created in `VisitType()` that outlive the method call.

When compiling `VisitType()`, the compiler verified the annotation on `predicate` by determining that:
- `predicate` is only invoked directly and `Func<T, TResult>.Invoke()` is annotated `[Lifetime(LifetimeScope.Container)]`.

When compiling `ContainsTypeParameter()`, the compiler verified that the only reference to the closure class instance generated by the compiler is as the instance for the delegate passed to `VisitType()` and the type `System.Func<T, TResult>` and base type `System.Delegate` are annotated `[Lifetime(LifetimeScope.Container)]`.

When compiling the call to `VisitType(type, t => t == typeParameter)`, the compiler verified there are no references to the delegate outside of the invocation expression .

The combination of the local analysis and annotations on `VisitType()` means the compiler can safely mark both `newobj` instructions (for the closure class and the delegate) as potential stack allocations.

### Example: Builder
The `Concat()` method below uses a `System.Text.StringBuilder` instance to concatenate an array of strings.
The earlier examples described implicit allocations generated by the compiler. In this example there is an explicit allocation in source: `new StringBuilder()`.

```csharp
[Lifetime(LifetimeScope.Container)]
public sealed class StringBuilder
{
    public int Length { get; }

    [return: Lifetime(LifetimeScope.Related, parameters: 0b01)]
    public StringBuilder Append([Lifetime(LifetimeScope.Container)] string value) { ...; return this; }

    [return: Lifetime(LifetimeScope.Related, parameters: 0)] public string ToString() { ... }

    ...
}

static string Concat(string[] values)
{
    var builder = new StringBuilder();
    foreach (var value in values)
    {
        if (builder.Length > 0) builder.Append(", ");
        builder.Append(value);
    }
    return builder.ToString();
}
```

The annotation on the `StringBuilder` type indicates that the instance methods in this class and base classes do not generate any references to `this`, or to fields reachable from `this`, other than on the callstack.

The `[return: Lifetime(LifetimeScope.Related, parameters: 0b01)]` annotation on `Append(string)` indicates the lifetime of the return value does not extend beyond the lifetime of `this`. That is clearly true since `Append(string)` returns `this`.

When compiling the implementation of `StringBuilder`, the compiler verified the annotations on the various methods.

When compiling `Concat()`, the compiler verified the lifetime of the `builder` instance is the scope in which it was allocated by determining that:
- the direct uses of `builder` are as the receiver in calls to `Length`, `Append(string)` and `ToString()` which are all annotated `[Lifetime(LifetimeScope.Container)]`;
- the return value from `builder.ToString()` is the only reference returned from the scope and the `[return: Lifetime(LifetimeScope.Related, parameters: 0)]` annotation on `ToString()` indicates that reference is independent of the `StringBuilder` instance.

## See also

Various issues have discussed allocating allocating reference types on the stack, including
[coreclr/issues/1784](https://github.com/dotnet/coreclr/issues/1784), [corefxlab/pull/2595#comment](https://github.com/dotnet/corefxlab/pull/2595#discussion_r235208262).

