# Stack allocation of reference types

## Summary
Allow stack allocation of reference type instances that do not escape the allocation scope.

## Motivation
There are several common patterns where a reference type instance has a lifetime that is limited to the method in which it is allocated. Some examples: builders, `params` arrays, delegates and closure classes.
In those cases, the compiler should indicate to the runtime that the instances can be allocated on the callstack rather than the heap.

For a given application, the fraction of allocations moved to the callstack may be small, but ideally the code changes required would be small too.
For calling code that relies on approriately annotated libraries, there may be fewer heap allocations without any code changes.

## Detailed design
The compiler will use type and method annotations and escape analysis to determine if the lifetime of a reference type instance is limited to the method in which it was allocated.
If the compiler determines the lifetime is limited to the allocation scope, the compiler will emit a hint to the runtime that the instance can be allocated on the stack.

### Attributes
This proposal relies on a several attributes for the annotations.

Methods and input parameters may be annotated with `[IsStackSafe(bool safe)]`.
- A method annotated with `[IsStackSafe(true)]` indicates that execution of that method does not generate any references to `this` or to fields reachable from `this` outside of the callstack. A method annotated `[IsStackSafe(false)]` may generate references to `this` or fields reachable from `this` outside of the callstack.
- An input parameter annotated with `[IsStackSafe(true)]` indicates that execution of the containing method does not generate any references to the parameter or to fields reachable from the parameter outside of the callstack. An input parameter annotated `[IsStackSafe(false)]` may generate references to the parameter or fields reachable from the parameter outside of the callstack.
- A `class` or `struct` annotated with `[IsStackSafe(true)]` indicates that all methods are `[IsStackSafe(true)]`. The is useful for types loaded from metadata where the attribute on the type definition indicates all methods, even private interface implementations are `[IsStackSafe(true)]`.
- No attribute is the equivalent of `[IsStackSafe(false)]` but also indicates the method has not opted in to escape analysis.

If a method is annotated with `[IsStackSafe(true)]` or `[IsStackSafe(false)]`, the method return value and parameters may be annotated with `[MayReference(int parameters)]` where `parameters` is a bit array of the parameters with the 0-th bit indicating `this`.
- A method annotated with `[return: MayReference(parameters)]` indicates the return value may contain a reference to parameters in the `parameters` bit array or to fields reachable from those parameters.
- A parameter annotated with `[MayReference(parameters)]` indicates the value may contain a reference to parameters in the `parameters` bit array or to fields reachable from those parameters.
- No attribute is the equivalent of `[MayReference(0)]`.

The attributes above applied to an API are part of the contract of the API and should be declared and versioned along with the rest of the API contract.



_What if the attributes talked about lifetime instead? That is, the lifetime of the return value is tied to the lifetime of what parameters? And can the lifetime of `this` and each parameter be specified by the caller rather than the method?_


### Escape analysis
Escape analysis is used in each method to determine which references are copied to method parameters or fields reachable from method parameters, and which references are copied to globals or fields reachable from globals.




The compiler will use escape analysis in the bodies of the methods and report errors if there are references to parameters or `this` that are not captured in the `[IsStackSafe]` and `[MayReference]` attributes on the method.


_Without knowing the provenance of each parameter (not just the one we want to allocate on the stack), how do we know that each fulfills it's contract?_



Escape analysis must recognize all cases where a reference does escape but does not need to recognize all cases where a reference does not escape.

There are many cases to handle but the following are several cases where a reference must be considered escaping:
- the reference is the receiver of a method call and the method is not marked `[DoesNotEscape]`,
- the reference is an argument to a method call and the parameter is not marked `[DoesNotEscape]`,
- the reference is assigned to a local or parameter that escapes the current scope.

The compiler must also check the allocated type and any base types and interface types to verify that each base type and interface type are marked `[CanBeStackAllocated(true)]`.

The compiler must also check the allocated type and any base types and interface types to verify that each overridden method from a base type and each implemented method from a interface type has `[DoesNotEscape]` attributes that are at least as strong as the method being overridden or implemented. If not, the allocation must be considered to escape any method where the allocated instance is used, even if the corresponding parameter is marked `[DoesNotEscape]`.

### Incremental adoption
_Describe how to avoid breaking changes._

_Contracts can get stronger without being a breaking change, but weakening contracts is a breaking change._


In an annotated method, the compiler will report an error if an annotated variable may escape the containing method.

In calling code, when a reference type is allocated, the compiler will use escape analysis to determine if the instance escapes the scope in which it was allocated. If the instance does not escape the allocation scope, the compiler will emit a hint to the runtime to allow stack allocation. If the instance may escape the allocation scope, no hint is included.
Allocations may be explicit or implicit.

Escape analysis will consider calls to unannotated methods with annotated variables as arguments or receiver. 

Base classes and derived types can be annotated independently. And a derived type may have a stronger or weaker contract than the base type.
When the compiler determines if an instance `Derived d` escapes the current scope, if there is a call to virtual method `d.M1()`, the compiler checks that the implementation of `Derived.M1()` is marked `[DoesNotEscape]`. If there is a call `M2(d)` to `M2([DoesNotEscape] Base b)`, the compiler checks that all virtual methods in `Base` that are annotated `[DoesNotEscape]` are also annotated in `Derived`.

It may be necessary to mark types that are considered annotated but contain no methods with `[DoesNotEscape]` attributes, so the compiler can differentiate an annotated type with no annotations from an unannotated type.

_The contract on each method depends on the contracts on a graph of other methods. That makes the contracts fragile. And when the implementation of a particular method changes, it's not necessarily that the new implementation now legitimately allows escaping where it previously did not, it might be that the compiler does not recognize the new calling pattern even though the new implementation is still not escaping._


### Example: Boxing
In the call to `Console.WriteLine()` below, the compiler boxes the `int` argument.

_Consider using a string interpolation example instead._

```csharp
// Console.WriteLine
public static void WriteLine(string format, [DoesNotEscape] object arg);

Console.WriteLine("{0}", 42);
```

The compiler allocates an `object` instance indirectly with:
```
ldc.i4 42
box System.Int32
```

The annotation in the signature of `Console.WriteLine()` indicates to callers that the `arg` parameter does not escape the method or any method called by `WriteLine()`. That is, the only references to `arg` as a result of calling `WriteLine()` were references on the callstack within the stack frame for `WriteLine()`.

The compiler can determine there are no references to the boxed instance outside of the invocation expression locally, and the method asserts the instance does not escape the method call, so the compiler can mark the `box` instruction as a potential stack allocation.

### Example: `params` array
In the call to the overload of `Console.WriteLine()` that takes a `params object[]` argument, the compiler allocates an array.

```csharp
// Console.WriteLine
public static void WriteLine(string format, [DoesNotEscape] params object[] arg);

Console.WriteLine("{0}, {1}, {2}, {3}", a, b, c, d);
```

The compiler allocates an `object[]` instance implicitly with:
```
ldc.i4.4
newarr System.Object
```

The annotation in the signature of `Console.WriteLine()` indicates to callers that the `arg` array does not escape the method or any method called by `WriteLine()`.
That is, `WriteLine()` does not generate any references to objects in the object graph reachable from `arg` that are held after the method returns.
Not only are there no references to the array instance, there are no references to any of the elements or the fields of those elements.

_Let's call this attribute `[Borrow]`._

The compiler can determine there are no references to the array outside the invocation expression locally, and the method asserts the array instance does not escape the method call, so the compiler can mark the `newarr` instruction as a potential stack allocation.

_`object[]` implements `IEnumerable` which presumably must be marked as allowing `this` to escape, or at least allows returning an object that contains `this` which then needs to be tracked by the caller (in this case, `WriteLine()`)._

### Example: Delegate and closure class
The method `ContainsTypeParameter()` below relies on a helper method `VisitType()` which invokes `predicate` on each component of `type`.

```csharp
// Returns first component of compound 'type' that satisfies 'predicate'.
static TypeSymbol VisitType(TypeSymbol type, [DoesNotEscape] Func<TypeSymbol, bool> predicate)
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

The annotation on the `Func<TypeSymbol, bool> predicate` parameter indicates that `predicate` does not escape the method or any method called by `VisitType()`.

The compiler can determine there are no references to the delegate or to the closure class outside of `ContainsTypeParameter`, so the compiler can mark the two `newobj` instructions as potential stack allocations.
As part of analysis in the caller, the compiler must verify the lambda body does not allow the closure class instance to escape.

### Example: Builder
The method `Concat()` below uses a `System.Text.StringBuilder` instance to concatenate an array of strings.
The earlier examples described implicit allocations generated by the compiler. In this example there is an explicit allocation in source: `new StringBuilder()`.

```csharp
[CanBeStackAllocated]
public sealed class StringBuilder
{
    public StringBuilder() { ... }
    [return: ContainsThis] public StringBuilder Append(string value) { ...; return this; }
    public int Length { get; }
    public string ToString() { ... }
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

`StringBuilder` has two annotations.

The class is annotated with `[CanBeStackAllocated]` indicating that none of the methods in this class or in any base classes allow references to escape from the calling context. That is, there are no references to `this`, or to any parameters to instance methods in this class, or to any fields reachable from `this` or any parameters, other than references on the stack as a result of calling instance methods on this class.

_`static` methods must be annotated separately. That is, each `static` method is annotated independently of the containing type. That might be a significant amount of overhead - at least one attribute per `static` method._

_Each reference type that is returned or used as a by-ref parameter is potentially annotated with the relationship of that value to any of the parameters._

The `Append(string)` method is annotated with `[return: ContainsThis]` indicating that the return value may include a reference to `this`. In fact, the `Append(string)` method always returns `this` directly.

_Mention the base type `System.Object` is also annotated `[CanBeStackAllocated]`. `System.Object` is not a `sealed` type but the annotation simply indicates that this class and all base classes do not allow references to escape. It does indicate how derived types behave._

The compiler should be able to determine the `StringBuilder` instance does not escape the `Concat()` method and mark the `newobj System.Text.StringBuilder::.ctor()` instruction as a potential stack allocation.

There is some complexity in the analysis of `Concat()` though.
First, there are multiple methods invoked on the `StringBuilder` instance, each of which must be marked `[DoesNotEscape]`: `.ctor()`, `get_Length()`, `Append(string)`, and `ToString()`.
Second, `StringBuilder.Append(string)` returns `this`, so the `[DoesNotEscape]` annotation shown above is not sufficient. The annotation needs to indicate that while `this` does not silently escape, a reference to `this` is returned, and the compiler needs to verify in the caller that the return value from `Append()` is not allowed to escape the allocation scope.

_Perhaps `Append()` should be annotated as: `[DoesNotEscape, AliasThis] StringBuilder Append([Borrow] string value) { ... }`. The `[AliasThis]` attribute indicates the return value is an alias of receiver, and `[DoesNotEscape]` indicates no long-lived references._

_The compiler checks if the arguments types are known at compile time (that is, either a `sealed` type or an explicit type, and all fields are known deeply), and the call is to a method of a type known at compile-time (that is, either a `static` method or a non-`virtual` method, or a `sealed` type, or an explicit type). If the argument type and method implementation are known at compile time, then the compiler can assume the contract holds. In short, there needs to be some authority that can check the validity of the contracts in the call at compile time. That authority is the code that has sealed types. Any code that is dealing with non-sealed types can only assert that "given sealed types, the following is true"._

_There are two types of assertions: 1. Does a particular type allow instances to escape? This can only be answered with certainty with a `sealed` type, by looking at all methods on the type and base type including any interface implementations. 2. Does a block of executable code allow locals or parameters to escape? That is answered based on the expressions in the executable code including any methods it calls. To answer this second case, methods including virtual methods and interface methods need to be annotated with the escaping and aliasing behavior._


_This should be a proposal for incrementally adopting ownership annotations and the corresponding wins that would provide (such as potential stack allocation of reference types)._





The compiler is responsible for verifying the method implementation satisfies any `[DoesNotEscape]` contract for parameters or `this`.

Note that `arg` may implement `IFormattable` for instance. The annotation on `Console.WriteLine()` asserts that `arg` will not escape regardless of what methods are called on the declared type or any other type that might be implemented by `arg`.



It should be possible to opt-out from (or opt-in to) runtime stack allocation of reference types.
Perhaps there is a `[MethodImpl(MethodImplOptions.NoStackReferenceTypes)]` attribute to opt-out in the calling method for instance.

It should be possible to assert that allocations occur on the stack rather than leaving that decision to the compiler and runtime.
Perhaps `stackalloc` rather than `new` could be allowed for those cases, at least for explicit allocations.
`stackalloc` could also be used for lambda delegates: `stackalloc x => x == y`.


When compiling `Console.WriteLine()`, the compiler will report an error if references to `args` or items in `args` are copied outside the method. If `WriteLine()` only calls `args[i].ToString()`, analysis should succeed since `Object.ToString()` is annotated.

When compiling the caller, if the compiler determines an argument type satisfies the contract on `System.Object` - that is, methods overridden from `System.Object` have `[DoesNotEscape]` annotations that are at least as strong as those on `System.Object` - that argument can be boxed in an `object` instance on the stack.

## Unresolved questions

The proposal covers allocation in the current scope only. The approach does not cover other allocation patterns such as calls to factory methods.

The proposed attribute `[DoesNotEscape]` does not cover relationships between parameters and return values. For instance, the annotation for `StringBuilder StringBuilder.Append(string value)` should indicate that `this` is returned from the method.

What are the exact rules used to determine when a variable escapes? For instance, use of a variable casted to an interface or class that is not in the declared interfaces or base type may be considered as escaping.

Can the format of the hints to the runtime be made compatible earlier runtimes?

Should it be possible to compile an annotated method without verifying annotations? That might be useful for cases where escape analysis is inconclusive. Should older compilers be prevented from compiling annotated methods and silently ignoring annotations?

Will the runtime be able to use stack allocation for a `newarr` instruction where the array size is a constant previously pushed onto the stack?

The proposal requires all methods of a type to marked `[DoesNotEscape]` to allow an instance of the type to allocated on the stack. Should we also support stack allocation if some methods where `this` escapes but those methods are not called? Tracking which methods (or groups of methods or interfaces perhaps) are invoked in each method is a lot of state to track.

## See also

Various issues have discussed allocating allocating reference types on the stack, including
[coreclr/issues/1784](https://github.com/dotnet/coreclr/issues/1784), [corefxlab/pull/2595#comment](https://github.com/dotnet/corefxlab/pull/2595#discussion_r235208262).

