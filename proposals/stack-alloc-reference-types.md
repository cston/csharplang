## Summary
Allow stack allocation of reference type instances that do not escape the allocation scope.

## Motivation
There are several common patterns where a reference type instance has a lifetime limited to the method in which it is allocated: builders, `params` arrays, delegates and closure classes.
In some cases it should be possible to allocate those instances on the callstack rather than the heap.

For a given application, the fraction of allocations moved to the callstack may be small, but ideally the code changes required would be small too.
For calling code that relies on approriately annotated libraries, there may be fewer heap allocations without any code changes.

## Detailed design
The compiler will rely on method annotations and escape analysis to determine if the lifetime of a reference type instance is limited to the method in which it was allocated.
If the lifetime is limited to allocation scope, the compiler will emit a hint to the runtime that the instance can be allocated on the callstack.

For simplicity, assume classes and methods can be annotated with a `[DoesNotEscape]` attribute to indicate that receivers or parameters do not escape the corresponding methods.

A `[DoesNotEscape]` attribute:
- on a method indicates `this` does not escape the method,
- on a parameter indicates the parameter does not escape the containing method,
- on a class or interface indicates `this` does not escape any instance methods (equivalent to annotating all instance methods).

In an annotated method, the compiler will report an error if an annotated variable may escape the containing method.

In calling code, when a reference type is allocated, the compiler will use escape analysis to determine if the instance escapes the scope in which it was allocated. If the instance does not escape the allocation scope, the compiler will emit a hint to the runtime to allow stack allocation. If the instance may escape the allocation scope, no hint is included.
Allocations may be explicit or implicit.

Escape analysis will consider calls to unannotated methods with annotated variables as arguments or receiver. 

Base classes and derived types can be annotated independently. And a derived type may have a stronger or weaker contract than the base type.
When the compiler determines if an instance `Derived d` escapes the current scope, if there is a call to virtual method `d.M1()`, the compiler checks that the implementation of `Derived.M1()` is marked `[DoesNotEscape]`. If there is a call `M2(d)` to `M2([DoesNotEscape] Base b)`, the compiler checks that all virtual methods in `Base` that are annotated `[DoesNotEscape]` are also annotated in `Derived`.

It may be necessary to mark types that are considered annotated but contain no methods with `[DoesNotEscape]` attributes, so the compiler can differentiate an annotated type with no annotations from an unannotated type.

It should be possible to opt-out from (or opt-in to) runtime stack allocation of reference types.
Perhaps there is a `[MethodImpl(MethodImplOptions.NoStackReferenceTypes)]` attribute to opt-out in the calling method for instance.

It should be possible to assert that allocations occur on the stack rather than leaving that decision to the compiler and runtime.
Perhaps `stackalloc` rather than `new` could be allowed for those cases, at least for explicit allocations.
`stackalloc` could also be used for lambda delegates: `stackalloc x => x == y`.

## Examples

### Example: `params` array and boxing

Consider the call to `Console.WriteLine()` below that formats two value types, where the `ToString()` implementations are annotated with `[DoesNotEscape]`, and the `params object[] args` is annotated, and where the `params` annotation means the `args` array does not escape the method, nor do the items in the array. The `object` instances for the boxing conversions of `1` and `new MyStruct()` can be allocated on the stack, and the `object[2]` array can also be allocated on the stack.

```csharp
// Object.ToString()
[DoesNotEscape] public virtual string ToString() { ... }

// Int32.ToString(), MyStruct.ToString()
[DoesNotEscape] public override string ToString() { ... }

// Console.WriteLine()
public static void WriteLine(string format, [DoesNotEscape] params object[] args) { ... }

Console.WriteLine("{0}, {1}", 1, new MyStruct());
```

When compiling `Console.WriteLine()`, the compiler will report an error if references to `args` or items in `args` are copied outside the method. If `WriteLine()` only calls `args[i].ToString()`, analysis should succeed since `Object.ToString()` is annotated.

When compiling the caller, if the compiler determines an argument type satisfies the contract on `System.Object` - that is, methods overridden from `System.Object` have `[DoesNotEscape]` annotations that are at least as strong as those on `System.Object` - that argument can be boxed in an `object` instance on the stack.

### Example: Delegate and closure class

`ContainsTypeParameter()` below relies on a helper method `VisitType()` which invokes `predicate` on each component type in `type`. For the lambda in `ContainsTypeParameter()`, the compiler generates a closure class for `typeParameter` and an instance method on the closure class for the delegate passed to `VisitType()`. Since the `predicate` parameter in `VisitType()` is annotated with `[DoesNotEscape]`, the delegate instance created implicitly for the call and the closure class instance can be allocated on the stack.

```csharp
// Returns first component of compound 'type' that satisfies 'predicate'.
static TypeSymbol VisitType(TypeSymbol type, [DoesNotEscape] Func<TypeSymbol, bool> predicate) { ... }

// Returns true if 'type' contains 'typeParameter'.
static bool ContainsTypeParameter(TypeSymbol type, TypeParameterSymbol typeParameter)
{
    return VisitType(type, t => t == typeParameter) != null;
}
```

## Unresolved questions

The proposal covers allocation in the current scope only. The approach does not cover other allocation patterns such as calls to factory methods.

The proposed attribute `[DoesNotEscape]` does not cover relationships between parameters and return values. For instance, the annotation for `StringBuilder StringBuilder.Append(string value)` should indicate that `this` is returned from the method.

What are the exact rules used to determine when a variable escapes? For instance, use of a variable casted to an interface or class that is not in the declared interfaces or base type may be considered as escaping.

Can the format of the hints to the runtime be made compatible earlier runtimes?

## See also

Various issues have discussed allocating allocating reference types on the stack, including
[coreclr/issues/1784](https://github.com/dotnet/coreclr/issues/1784), [corefxlab/pull/2595#comment](https://github.com/dotnet/corefxlab/pull/2595#discussion_r235208262).

