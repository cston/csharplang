# Collection expression target types implementing `IEnumerable` and with strongly-typed `Add`

## Summary
The conversion rules for collection expressions were tightened at [LDM-2024-01-10](https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-01-10.md#collection-expressions-conversion-vs-construction) to require target types that implement `IEnumerable`, and that do not have create method, to have:
> An accessible Add instance or extension method that can be invoked with value of iteration type as the argument.

That is a breaking change for collection types where the `Add` method has a parameter type that is *implicitly convertible but not identical* to the iteration type. Those types are no longer valid targets for collection expressions. We should relax the recent `Add` method requirement to address the breaking change.

Additionally, there are collection types where there is *no conversion* between the `Add` method parameter type and the iteration type. Those types are valid targets for classic *collection initializers* but have never been valid for *collection expressions*. We should consider supporting those types for collection expressions as well.

## Examples
There are several categories of collection types that are supported by classic *collection initializers* that are not supported in *collection expressions*.

The first two categories were supported before the recent requirement for `Add`, and these two categories represent the breaking change. The last category has not been supported previously with collection expressions but could be considered.

**Category 1:** Types that implement `IEnumerable` but not `IEnumerable<T>` and have a strongly-typed `Add(T)`.

Example: [`System.Windows.Input.InputGestureCollection`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.input.inputgesturecollection?view=windowsdesktop-8.0) implements `IEnumerable` but not `IEnumerable<T>`, and has an `Add(InputGesture)` but no `Add(object)`.

```csharp
namespace System.Windows.Input
{
    public sealed class InputGestureCollection : System.Collections.IList
    {
        public int Add(InputGesture inputGesture);
        // ...
    }
}

InputGestureCollection c = [new KeyGesture(default)]; // error: breaking change
```

**Category 2:** Types that implement `IEnumerable<T>` and have a strongly-typed `Add(U)` where `U` is implicitly convertible to `T`.
This is a generic form of category 1.

Example: [`System.CommandLine.Command`](https://learn.microsoft.com/en-us/dotnet/api/system.commandline?view=system-commandline) implements `IEnumerable<Symbol>` and has `Add()` methods for derived types of `Symbol` but not for `Symbol`.

```csharp
namespace System.CommandLine
{
    public class Symbol { /*...*/ }

    public class Argument : Symbol { /*...*/ }

    public class Option : Symbol { /*...*/ }

    public class Command : Symbol, IEnumerable<Symbol>
    {
        public IEnumerator<Symbol> GetEnumerator();
        public void Add(Argument argument);
        public void Add(Option option);
        public void Add(Command command);
        // ...
    }

    public class RootCommand : Command { /*...*/ }
}

RootCommand c = [new Argument()]; // error: breaking change
```

**Category 3:** Types that implement `IEnumerable<T>` and have a strongly-typed `Add(U)` where `U` and `T` are unrelated.

This category is distinctly different from the previous two categories since it has not been supported previously with collection expressions.

Example: [`Xunit.TheoryData<T>`](https://xunit.net/) implements `IEnumerable<object[]>`, and has an `Add(T)` but no `Add(object[])`.

```csharp
namespace Xunit
{
    public abstract class TheoryData : IEnumerable<object[]>, IEnumerable
    {
        // ...
    }

    public class TheoryData<T> : TheoryData
    {
        public void Add(T p);
    }
}

TheoryData<string> d = ["a", "b", "c"]; // error
```

## Background
This issue affects target types that implement `IEnumerable` and do not have create methods. The conversion requirements for such types were modified in [LDM-2024-01-10](https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-01-10.md#collection-expressions-conversion-vs-construction), and those changes were implemented in 17.10p1.

**In 17.8**, *conversion* simply requires the target type to implement non-generic `IEnumerable`, and does *not* require an `Add` method:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable`.
> 
> The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:
> * If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.
> * If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.

Even though an `Add` method is not required for *conversion*, an `Add` method is required for *construction* of the collection instance. That requirement is checked after determining *convertibility*, and the requirement is that for each element there is an applicable `Add` method that can be called with that expression:

> * The constructor that is applicable with no arguments is invoked.
> 
> * For each element in order:
>   * If the element is an *expression element*, the applicable `Add` instance or extension method is invoked with the element *expression* as the argument. (Unlike classic [*collection initializer behavior*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117154-collection-initializers), element evaluation and `Add` calls are not necessarily interleaved.)
>   * If the element is a *spread element* then ...

**In 17.10p1**, [*conversion*]((https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#conversions)) was updated to require the constructor and also to require an `Add` method callable with an argument of the *iteration type*. For *construction*, the requirements were unchanged from 17.8. The **updated** conversion requirement:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * **The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.**
>   * **If the collection expression has any elements, the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.**
> 
> The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:
> * If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.
> * If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.

There were two motivations for the additional `Add` method requirement:
1. Reduce the number of `IEnumerable` implementations that are considered valid target types for collection expressions but which fail to bind successfully. This is important for overload resolution to avoid unnecessary ambiguities.
2. Align with the *params collections* preview feature which has the additional requirement to allow validating the *params* type at the method declaration rather than only at call sites.

**For *params collections***, the requirements for the preview feature are:

> The *type* of a parameter collection shall be one of the following valid target types for a collection expression:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - The *type* has an instance (not an extension) method `Add` that can be invoked with a single argument of
>     the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/statements.md#1395-the-foreach-statement),
>     and the method is at least as accessible as the declaring member.

## Proposal
Relax the requirement for *conversion* to require an instance or extension `Add` method that can be invoked with a single argument, but without requirements on the method parameter *type*.
For *construction*, there are no changes.

For *collection expression conversions*, the **proposed change**:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   * If the collection expression has any elements, the *type* has an instance or extension method `Add` where:
>     * **The method can be invoked with a single argument.**
>     * **If the method is generic, the type arguments can be inferred from the collection and argument.**
>     * The method is accessible at the location of the collection expression.
> 
> The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:
> * If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.
> * If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.

The proposed change would resolve the breaking change in 17.10p1 and allow types in category 1 and 2 to be used as collection expression target types.

For *params collections*, there is a corresponding **proposed change**:

> The *type* of a parameter collection shall be one of the following valid target types for a collection expression:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - The *type* has an instance (not an extension) method `Add` where:
>     - **The method can be invoked with a single argument.**
>     - **If the method is generic, the type arguments can be inferred from the argument.**
>     - The method is at least as accessible as the declaring member.
> - ...

### Extended proposal
We could extend the proposal to remove the requirement that each element in the collection expression is implicitly convertible to the *iteration type* for these types. (We would only remove this requirement for types that implement `IEnumerable` and do not use a create method.)

For *collection expression conversions*, the **extended proposed change**:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
>   * If the collection expression has any elements, the *type* has an instance or extension method `Add` where:
>     * **The method can be invoked with a single argument.**
>     * **If the method is generic, the type arguments can be inferred from the collection and argument.**
>     * The method is accessible at the location of the collection expression.
> 
> ~~The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:~~
> * ~~If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.~~
> * ~~If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.~~

The extended proposal would allow types in category 1, 2, and 3 to be used as collection expression target types.

## Meetings

- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-01-10.md#collection-expressions-conversion-vs-construction
- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-02-26.md#collection-expressions

## Issues

- https://github.com/dotnet/roslyn/issues/72098
- https://github.com/dotnet/roslyn/issues/71240
