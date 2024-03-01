# Collection expression target types implementing `IEnumerable` and `Add(T item)`

## Summary
The requirements for collection expressions were tightened in 9.0.100-preview1 to require target types that implement `IEnumerable` have *an instance or extension `Add` method that can be called with an argument of the *iteration type**.

That is a breaking change from 8.0.100 &mdash; there are a number of legacy collection types such as `InputGestureCollection` that implement `IEnumerable` and do *not* implement `IEnumerable<T>`, and that expose a strongly-typed `Add(T item)` method rather than an `Add(object item)` method. Those types are no longer valid target types for collection expressions.

The following are several proposed alternatives to address the breaking change.

## Background
This issue affects certain target types that implement `IEnumerable` and that are populated with `Add` method calls. We sometimes refer to these types informally as *collection initializer types* because the API is also used by *collection initializer expressions*.

The spec requirements for collection expressions that target such types are split between the binding necessary for *conversion* and the binding for *construction*.

### 8.0.100
In 8.0.100, *conversion* does not require an `Add` method:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable`.
> * ...
> 
> The implicit conversion exists if the type has an [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) `U` where for each *element* `Eᵢ` in the collection expression:
> * If `Eᵢ` is an *expression element*, there is an implicit conversion from `Eᵢ` to `U`.
> * If `Eᵢ` is an *spread element* `Sᵢ`, there is an implicit conversion from the *iteration type* of `Sᵢ` to `U`.

For *construction*, an applicable constructor is required for the collection instance and an applicable `Add` method is required for each element expression:

> * The constructor that is applicable with no arguments is invoked.
> 
> * For each element in order:
>   * If the element is an *expression element*, the applicable `Add` instance or extension method is invoked with the element *expression* as the argument. (Unlike classic [*collection initializer behavior*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#117154-collection-initializers), element evaluation and `Add` calls are not necessarily interleaved.)
>   * If the element is a *spread element* then ...

### 9.0.100-preview1
In 9.0.100-preview1, [*conversion*]((https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#conversions)) was updated to require the constructor and also to require an `Add` method callable with an argument of the *iteration type*:

> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * **The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.**
>   * **If the collection expression has any elements, the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.**
> * ...

For *construction*, the requirements were unchanged.

The *collection expressions conversion* requirements align with [*params collections*](https://github.com/dotnet/csharplang/blob/main/proposals/params-collections.md) which has simiar requirements:
> The *type* of a parameter collection shall be one of the following valid target types for a collection expression:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - The *type* has an instance (not an extension) method `Add` that can be invoked with a single argument of
>     the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/draft-v9/standard/statements.md#1395-the-foreach-statement),
>     and the method is at least as accessible as the declaring member.
> - ...

### Breaking change
The breaking change affects types which implement `IEnumerable` but not `IEnumerable<T>` and has a strongly-typed `Add(T)` method rather than `Add(object)`. An example is [`System.Windows.Input.InputGestureCollection`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.input.inputgesturecollection?view=windowsdesktop-8.0):
```csharp
public sealed class InputGestureCollection : System.Collections.IList
{
    public InputGesture this[int index] { get; set; }
    object IList.this[int index] { get; set; }

    public int Add(InputGesture inputGesture);
    int IList.Add(object inputGesture);

    // ...
}
```

## Alternatives
The following are some of the alternatives to the breaking change that have been proposed (see [LDM-2024-02-26](https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-02-26.md#collection-expressions)).

### Alternative 1: No changes
No changes from 9.0.100-preview1.
Collection types such as `InputGestureCollection` would not be useable as a collection expression target type.

### Alternative 2: Remove the `Add` requirement
Remove the requirement for an `Add` method that can be called with an argument of the *iteration type*.

For *collection expression conversions*, the updated requirement:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> * ...
> * A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   * The *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* constructor that can be invoked with no arguments, and the constructor is accessible at the location of the collection expression.
> * ...

For *construction*, no changes &mdash; an applicable `Add` method will be required for each element expression.

For *params collections*, no changes &mdash; an applicable `Add` method callable with an argument of the *iteration type* will be required.

### Alternative 3: Remove the `Add` requirement *for non-generic enumerable implementations only*
This is a refinement of the previous alternative, but only removing the `Add` method requirement for target types that do not have a generic enumerable implementation.
The assumption is the breaking change is more likely with *legacy non-generic* collection types such as `InputGestureCollection`.

For *collection expression conversions*, the updated requirement:
> An implicit *collection expression conversion* exists from a collection expression to the following types:
> - ...
> - A *struct* or *class type* that implements `System.Collections.IEnumerable` where:
>   - The *type* has a constructor that can be invoked with no arguments, and the constructor is at least as accessible as the declaring member.
>   - If the collection expression has any elements, **and if the *iteration type* is determined from an `IEnumerable<T>` implementation or from a `GetEnumerator` method that returns a type other than `IEnumerable`, then** the *type* has an *[applicable](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11642-applicable-function-member)* instance or extension method `Add` that can be invoked with a single argument of the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement), and the method is accessible at the location of the collection expression.
> - ...

For *construction* and for *params collections*, no changes.

### Alternative 4: Determine *iteration type* from indexer type
If the *iteration type* is determined from a *non-generic* enumerable implementation and the type has an indexer, use the indexer type as the *iteration type* instead.
The assumption is that many of the types affected that expose strongly-typed `Add` methods also expose strongly-typed indexers.

No changes are needed for *conversion*, *construction*, or *params collections*, because the requirements are all based on *iteration type*.

## Meetings

- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-01-10.md#collection-expressions-conversion-vs-construction
- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-02-26.md#collection-expressions