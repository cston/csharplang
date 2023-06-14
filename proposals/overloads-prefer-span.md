# Prefer spans over interfaces in overload resolution

* [x] Proposed
* [ ] Prototype: Not Started
* [ ] Implementation: Not Started
* [ ] Specification: Not Started

## Summary
[summary]: #summary

Overload resolution should prefer an overload where the corresponding parameter has a *span type* over an overload where the parameter has a collection *interface type*.

## Motivation
[motivation]: #motivation

For APIs that take a collection of items, it may be useful to provide overloads for *collection interfaces* and for *spans*. Supporting collection interfaces such as `IEnumerable<T>` is particularly useful for callers using LINQ; supporting spans such as `ReadOnlySpan<T>` is useful for callers focused on performance.

However, arrays are implicitly convertible to collection interfaces *and* spans, so calls with array arguments may be considered ambiguous with the current overload resolution rules.

```C#
var ia = ImmutableArray.CreateRange<int>(new[] { 1, 2, 3 }); // error: CreateRange() is ambiguous

public static class ImmutableArray
{
    public static ImmutableArray<T> CreateRange<T>(IEnumerable<T> items) { ... }
    public static ImmutableArray<T> CreateRange<T>(ReadOnlySpan<T> items) { ... }
}
```

In cases such as arrays where both overloads are applicable, overload resolution should prefer the span overload.

## Detailed design
[design]: #detailed-design

The overload resolution rules for [*11.6.4.6 better conversion target*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#11646-better-conversion-target) could be updated with an additional rule:

> Given two types `T₁` and `T₂`, `T₁` is a ***better conversion target*** than `T₂` if one of the following holds:
> 
> - ...
> - `T₁` is `ReadOnlySpan<S₁>` or `Span<S₁>`, and `T₂` is `IEnumerable<S₂>` or `ICollection<S₂>` or `IList<S₂>` or `IReadOnlyCollection<S₂>` or `IReadOnlyList<S₂>`, and an implicit conversion exists from `S₂` to `S₁`

## Drawbacks
[drawbacks]: #drawbacks

- Overloads for collection interfaces and spans are still ambiguous for arrays when compiled with *older compilers*.

  Mixing older compilers and newer TFMs is not strictly a supported scenario but it is used. Perhaps we could add a custom modifier to new overloads so those overloads are ignored by older C# and VB compilers.

## Alternatives
[alternatives]: #alternatives

- Use distinct method names rather than overloads.

- Use an extension method to overload an instance method. The extension method will only be considered when the instance method is inapplicable which avoids ambiguities when both are applicable but it also prevents the compiler from choosing the better overload.

## Unresolved questions
[unresolved]: #unresolved-questions

## Design meetings
