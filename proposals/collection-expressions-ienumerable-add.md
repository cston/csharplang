# Collection expression target types implementing `IEnumerable` and `Add(T item)`

## Summary
The requirements for collection expressions were tightened in Visual Studio 17.10 preview 1 to require target types that implement `IEnumerable` have *an instance or extension `Add` method that can be called with an argument of the *iteration type**.

That is a breaking change from Visual Studio 17.8 &mdash; there are a number of collection types that implement `IEnumerable` or `IEnumerable<T>`, and that expose a strongly-typed `Add` method but where the iteration type of the collection cannot be implicitly converted to the `Add` method parameter type. Those types are no longer valid target types for collection expressions.

The following are several proposed alternatives to address the breaking change.

## Background
This issue affects certain target types that implement `IEnumerable` (and optionally `IEnumerable<T>`) and that are populated with `Add` method calls. We sometimes refer to these types informally as *collection initializer types* because the API is also used by *collection initializer expressions*.

The spec requirements for collection expressions that target such types are split between the binding necessary for *conversion* and the binding for *construction*.

### 17.8
In 17.8, *conversion* simply requires the target type to implement non-generic `IEnumerable`, and does *not* require an `Add` method:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable`.
> * ...
> 
> The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:
> * If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.
> * If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.

An `Add` method is required for *construction* of the collection instance though, although that requirement is checked after determining *convertibility*, and the requirement is that for each element there is an applicable `Add` method that can be called with that expression:

> * The constructor that is applicable with no arguments is invoked.
> 
> * For each element in order:
>   * If the element is an *expression element*, the applicable `Add` instance or extension method is invoked with the element *expression* as the argument. (Unlike classic [*collection initializer behavior*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117154-collection-initializers), element evaluation and `Add` calls are not necessarily interleaved.)
>   * If the element is a *spread element* then ...

### 17.10p1
In 17.10p1, [*conversion*]((https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#conversions)) was updated to require the constructor and also to require an `Add` method callable with an argument of the *iteration type*. For *construction*, the requirements were unchanged from 17.8.

The updated *conversion* rule:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * **The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.**
>   * **If the collection expression has any elements, the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.**
> * ...

There were two motivations for the additional `Add` method requirement:
1. Reduce the number of `IEnumerable` implementations that are considered valid target types for collection expressions but which fail to bind successfully. This is important for overload resolution to avoid unnecessary ambiguities.
2. Align with the *params collections* preview feature which has the additional requirement to allow validating the *params* type at the method declaration rather than only at call sites.

The requirements for *params collections* preview feature:

> The *type* of a parameter collection shall be one of the following valid target types for a collection expression:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - The *type* has an instance (not an extension) method `Add` that can be invoked with a single argument of
>     the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/statements.md#1395-the-foreach-statement),
>     and the method is at least as accessible as the declaring member.
> - ...

### Breaking change
The breaking change has been reported for several collection types: some that implement `IEnumerable` only, and some that also implement `IEnumerable<T>`.

**Example 1:** [`System.Windows.Input.InputGestureCollection`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.input.inputgesturecollection?view=windowsdesktop-8.0) implements `IEnumerable` but not `IEnumerable<T>`, and has an `Add(InputGesture)` but no `Add(object)`.

```csharp
namespace System.Windows.Input
{
    public sealed class InputGestureCollection : System.Collections.IList
    {
        public InputGesture this[int index] { get; set; }
        object IList.this[int index] { get; set; }

        public int Add(InputGesture inputGesture);
        int IList.Add(object inputGesture);

        // ...
    }
}
```

**Example 2:** [`Xunit.TheoryData<T>`](https://xunit.net/) implements `IEnumerable<object[]>`, and has an `Add(T)` but no `Add(object[])`.

```csharp
namespace Xunit
{
    public abstract class TheoryData : IEnumerable<object[]>, IEnumerable
    {
        public IEnumerator<object[]> GetEnumerator();
        IEnumerator IEnumerable.GetEnumerator();
    }

    public class TheoryData<T> : TheoryData
    {
        public void Add(T p);
    }
}
```

**Example 3:** [`System.CommandLine.Command`](https://learn.microsoft.com/en-us/dotnet/standard/commandline/) implements `IEnumerable<Symbol>` and has `Add()` methods for derived types of `Symbol` but not for `Symbol`.

```csharp
namespace System.CommandLine
{
    public class Symbol { /*...*/ }

    public class Argument : Symbol { /*...*/ }

    public class Option : Symbol { /*...*/ }

    public class Command : Symbol, IEnumerable<Symbol>
    {
        public IEnumerator<Symbol> GetEnumerator();
        IEnumerator IEnumerable.GetEnumerator();

        public void Add(Argument argument);
        public void Add(Option option);
        public void Add(Command command);
    }
}
```

## Alternatives
The following are some of the alternatives that have been proposed to address the breaking change.

### Alternative 1: No changes
No changes from 17.10p1.
The affected collection types, such as the examples above, would not be useable for collection expressions or params collections.

### Alternative 2: Remove the `Add` requirement for conversion
Remove the requirement for an `Add` method that can be called with an argument of the *iteration type*, and revert to the 17.8 requirements.

For *collection expression conversions*, the updated requirement:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   * ~~If the collection expression has any elements, the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.~~
> * ...

For *construction*, no changes &mdash; an applicable `Add` method will be required for each element expression.

For *params collections*, no changes &mdash; an applicable `Add` method callable with an argument of the *iteration type* will be required.

The affected collection types would be useable for collection expressions, but not for params collections.

### Alternative 3: Remove the `Add` requirement for conversion *for non-generic enumerable implementations only*
This is a refinement of the previous alternative, but only removing the `Add` method requirement for target types that do not have a generic enumerable implementation.
The assumption is the breaking change is more likely with *non-generic* collection types such as `InputGestureCollection`.

For *collection expression conversions*, the updated requirement:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   * If the collection expression has any elements, **and if the *iteration type* is determined from an `IEnumerable<T>` implementation or from a `GetEnumerator` method that returns a type other than `IEnumerable`, then** the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.
> * ...

For *construction* and for *params collections*, no changes.

The affected collection types that implement `IEnumerable` but not `IEnumerable<T>` would be usable in collection expressions but not params collections. The affected types that implement `IEnumerable<T>` would not be usable for either collection expressions or params collections.

### Alternative 4: Require an `Add` method for conversion callable with a single argument
Relax the requirement for conversion to require an instance or extension `Add` method that can be invoked with a single argument, but without requirements on the method parameter *type*.

For *collection expression conversions*, the updated requirement:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   * If the collection expression has any elements, the *type* has an instance or extension method `Add` where:
>     * **The method can be invoked with a single argument and the corresponding parameter is either by value or `in`.**
>     * **If the method is generic, the type arguments can be inferred from the arguments.**
>     * The method is accessible at the location of the collection expression.
> * ...

For *construction*, no changes.

For *params collections*, the updated requirement:

> The *type* of a parameter collection shall be one of the following valid target types for a collection expression:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - The *type* has an instance (not an extension) method `Add` where:
>     - **The method can be invoked with a single argument and the corresponding parameter is either by value or `in`.**
>     - **If the method is generic, the type arguments can be inferred from the argument.**
>     - The method is at least as accessible as the declaring member.
> - ...

The affected collections would be usable for collection expressions and for params collections.

### Alternative 5: Determine *iteration type* from indexer type
If the *iteration type* is determined from a *non-generic* enumerable implementation and the type has an indexer, use the indexer type as the *iteration type* instead.
The assumption is that many of the *legacy non-generic* types affected that expose strongly-typed `Add` methods also expose strongly-typed indexers.

No changes are needed for *conversion*, *construction*, or *params collections*, because the requirements are all based on *iteration type*.

The affected collections that implement a single strongly-typed indexer matching the `Add` parameter type would be usable for collection expressions and for params collections.

## Meetings

- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-01-10.md#collection-expressions-conversion-vs-construction
- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-02-26.md#collection-expressions

## Issues

- https://github.com/dotnet/roslyn/issues/72098
- https://github.com/dotnet/roslyn/issues/71240

