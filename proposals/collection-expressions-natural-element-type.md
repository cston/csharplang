# Collection expressions: natural *element type*

## Summary
[summary]: #summary

Infer a natural *element type* for collection expressions to allow collection expressions to be used in locations where the collection type and collection instance are not observable.

## Motivation
[motivation]: #motivation

Inferring an element type would allow collection expressions to be used in certain scenarios where the choice of how or whether to instantiate the collection is not observable and can be left to the compiler.

Collection expressions with inferred element type could be used within spreads to allowing adding elements *conditionally* to the containing collection:
```csharp
int[] items = [x, y, .. b ? [z] : []];
```

Collection expressions with inferred element type could be used in `foreach`:
```csharp
foreach (var b in [false, true]) { }
```

## Detailed design
[design]: #detailed-design

A *collection_type* is introduced to represent an iterator with a known element type and an unspecified collection type.

The element type of a *collection_type* is the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement).

*Collection_types* exist at compile time only; *collection_types* do not appear in source or metadata.

*Collection_types* are used in a few specific contexts only:
- collection expression natural type
- spreads
- type inference and best common type
- `foreach`

### Conversions

*Collection_types* are *co-variant*: there is an implicit *collection_type conversion* from a *collection_type* with element type `A` to a *collection_type* with element type `B` if there is an identity conversion or implicit reference conversion from `A` to `B`.

We could potentially allow *any* implicit conversion from `A` to `B`, but restricting conversions to identity conversions and implicit *reference* conversions matches *co-variance* elsewhere in the language.

### Collection expression natural type

The natural type of a *collection expression* is a *collection_type* where the element type is the [*best common type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#116315-finding-the-best-common-type-of-a-set-of-expressions) from the elements.
If there is no *best common type* from the elements, the collection expression has no natural type.

```csharp
[]              // no type
[1, null]       // no type
[1, (int?)null] // collection<int?>
```

### Spreads

*Describe construction of the containing collection.*
*How do we flatten the arbitrarily deep collection expression with *collection_type*?*

### Type inference and best common type

The type inference [*fixing*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#116312-fixing) rules are updated as follows:

> An *unfixed* type variable `Xi` with a set of bounds is *fixed* as follows:
> 
> *  The set of *candidate types* `Uj` starts out as the set of all types in the set of bounds for `Xi` where *function_types* are ignored in lower bounds if there any types that are not *function_types* **and where *collection_types* are ignored in lower bounds if there any types that are not *collection_types***.
> *  ...

The [*best common type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#116315-finding-the-best-common-type-of-a-set-of-expressions) rules handle *collection_type* accordingly since those rules are based on *type inference*.

The *best common type* rules mean that *collection_types* can be inferred from collection expressions to the expressions that contain them. For instance:
- The type of a *conditional expression* `b ? [x] : [y]` will be the *best common type* of the *collection_types* from `[x]` and `[y]`.
- The type of a *switch expression* where arms contain collection expressions only will be the *best common type* of the *collection_types* of the arms.

```csharp
_ = b ? [1] : [2s];        // error: no common type for collection<int>, collection<short>
Conditional(b, [1], [2s]); // error: no common type for collection<int>, collection<short>
CondArray(b, [1], [2s]);   // ok: int[]

static T Conditional<T>(bool b, T x, T y) => b ? x : y;
static T[] CondArray<T>(bool b, T[] x, T[] y) => b ? x : y;
```

*What is the return type of the following?*
```csharp
var array = MakeArray([1]); // collection<int>[] !?!

static T[] MakeArray(T x) => [x];
```

### foreach

For an expression with a *collection_type* used as the collection in a `foreach` statement, the compiler may use any conforming representation for the collection instance, including skipping construction of the instance entirely and simply referencing the elements directly.

*Describe rewriting of `foreach` loop.*
*How do we flatten the arbitrarily deep collection expression with *collection_type*?*

## Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

## Alternatives
[alternatives]: #alternatives

<!-- What other designs have been considered? What is the impact of not doing this? -->

## Unresolved questions
[unresolved]: #unresolved-questions

## Design meetings

<!-- Link to design notes that affect this proposal, and describe in one sentence for each what changes they led to. -->