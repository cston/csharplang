# Better conversion from collection expression based on element type

## Summary

There are overload resolution cases with collection expressions that fail with candidates that differ by collection element type when there is an obvious better candidate, and where similar overload resolution cases that use single values, tuples, `params`, or *first-class spans* might succeed.

Example 1: `string.Concat()` with `ReadOnlySpan<string>` and `ReadOnlySpan<object>` overloads added in .NET 9 (see https://github.com/dotnet/roslyn/issues/73857, [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBUQAT7EDsFA3hdW7cQGy0As1AstBgAKAJSt2LcuxnUA9HOoBTZMgD2yENQhhgUAOao1qAM4cADADoAwmpgBjCAmEAlJRAwB5GABsAngDKAA4QMAA8asAAVkr2CAD8AHyicBY2do7Obh7e/sGhYXTmSeLSsmy2Dk7CANoARBB1qXXATdR19nUAuqIA3BJsAL4UAxzc+HyVma7uXr6BIeGRMXFJ2sj6JqLUTNTDZWx045MZ1dlzeYuFxMWJ65vbu/uDQA=)).

```csharp
// error: ambiguous string.Concat(ReadOnlySpan<object?>), string.Concat(ReadOnlySpan<string?>)
string.Concat(["a", "b", "c"]);

static void Concat(ReadOnlySpan<object?> args) { }
static void Concat(ReadOnlySpan<string?> args) { }
```

Example 2: `IEnumerable<string>` and `IEnumerable<IFormattable>` with interpolated strings (see https://github.com/dotnet/csharplang/issues/7651, [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUPgAwAE+xALANwUVF3EDsFA3hTQI4A2OgxoBZaDAAUASn6C+5QSpoB6NTQCmyZAHtkIGhDDAoAc1R7UAZwkBPcVoQALPRmn0AzAB56VAD5ZOAcnV3dPYl8ASQAxA0gEBAhgABstIIVVUOc3DwBtABIAIj0YLWKQkoQAdz1igF1ZFmVBAF9WVoF6EXwxcUdciO8/YkCaAH1ZGh4aDq7hURzwjxG4hIgklPSAyenZ+bagA=)).

```csharp
// error: ambiguous MyMethod(IEnumerable<string>), MyMethod(IEnumerable<IFormattable>)
MyMethod([$"one", $"two"]);

static void MyMethod(IEnumerable<string> _) { }
static void MyMethod(IEnumerable<IFormattable> _) { }
```

Example 3: Implicit numeric conversions with span and array or interface (see [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBUQAT7EDsFA3hdW7cQGy0As1AstBgAKAJSt2LcuxnUAYsWEBtYnGqE1AZgC6ogNzUA9IeoBTZMgD2yENQhhgUAOapLqAM7zFAJVMQMAPIwADYAngDKAA4QMAA8sAgAfKJqCsLAoQimSroSsvKEyqrqWroGxtSWANa2kcimAGbmnnKFvv5BYVExsRlZyXnUAL4Ug3Tc+Hxp7YEhEdFxCYl2yE7uotRMw2NcvF7pmdnaK2sbWyOU0uzje63CM53zPX2myxCr65vbV2w3kwXCBI5E6fc4UIZAA=)).

```csharp
F1([1, 2, 3]); // error: ambiguous F1(ReadOnlySpan<int>), F1(byte[])
F2([1, 2, 3]); // ok: prefers F2(ReadOnlySpan<byte>)

static void F1(ReadOnlySpan<int> args) { }
static void F1(byte[] args) { }

static void F2(ReadOnlySpan<byte> args) { }
static void F2(int[] args) { }
```

## Background

### Better conversion from expression

[*Better conversion from expression*](https://github.com/dotnet/csharplang/blob/main/proposals/first-class-span-types.md#overload-resolution) is currently defined as follows:

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression* and one of the following holds:
>   - `T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.Span<E₂>`, and an implicit conversion exists from `E₁` to `E₂`.
>   - `T₁` is `System.ReadOnlySpan<E₁>` or `System.Span<E₁>`, and `T₂` is an *array_or_array_interface* with *element type* `E₂`, and an implicit conversion exists from `E₁` to `E₂`.
>   - `T₁` is not a *span_type*, and `T₂` is not a *span_type*, and an implicit conversion exists from `T₁` to `T₂`.
> - `E` is not a *collection expression* and one of the following holds:
>   - `E` exactly matches `T₁` and `E` does not exactly match `T₂`
>   - `E` exactly matches neither of `T₁` and `T₂`,
>     and `C₁` is an implicit span conversion and `C₂` is not an implicit span conversion
>   - `E` exactly matches both or neither of `T₁` and `T₂`,
>     both or neither of `C₁` and `C₂` are an implicit span conversion,
>     and `T₁` is a better conversion target than `T₂`
> - `E` is a method group, `T₁` is compatible with the single best method from the method group for conversion `C₁`, and `T₂` is not compatible with the single best method from the method group for conversion `C₂`

The current rules have several issues:
1. There is no preference for `ReadOnlySpan<E₁>` over `ReadOnlySpan<E₂>`, or `Span<E₁>` over `Span<E₂>`.
1. The preference for `E₁[]` over `E₂[]`, and `IEnumerable<E₁>` over `IEnumerable<E₂>` etc., only apply when there is an implicit conversion from `T₁` to `T₂`.
1. The preference for `ReadOnlySpan<E₁>` over `Span<E₂>`, and for `ReadOnlySpan<E₁>` or `Span<E₁>` over `E₂[]` or an array interface with element  type `E₂` applies for any implicit conversion from `E₁` to `E₂`. In particular, this includes *implicit numeric conversions* which seems questionable.
1. The [*first-class span*](https://github.com/dotnet/csharplang/blob/main/proposals/first-class-span-types.md) changes only apply when `E` is not a *collection expression*.
1. There are inconsistencies with the betterness rules for `params`.

### Comparison without collection expressions

If the examples above are rewritten to use single values, tuples, `params`, or *first-class spans*, overload resolution succeeds, with the expected result.

Example 1: [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBUQAT7EDsFA3hdW7cQGy0As1AstBgAKAJSt2LcuxnUAwgHsYAYwgIAyrADmAGwCmwgEQRDogNyzLAeivUFAaxDUADsj0AzPcgDO8pao1tfWE6AAYAfnFpSz8VNQAVVGdgoxM4akNgU3NZGztHFzdPH1iAxOSDEOII9LDIqJjStQAFCGQIMG9Uw3TMnozlUws8hydXDy9fRTiEVvbO4Wc2jt86gG0AXQb2AF8KCTY6bnw+aYDNGF0DBWAAKz1lBHDqNq1RaiZqPejDrl4mwKXYJ1F7IN4fL77H4cY6nfwJJIpG73R7hdLIh5Pd6vd6fb4yI7/M4IirCKo1DgRbFg3GQyjQwknAFzFaLZadOx3THhTagrTeWn49iMuEzFkLJbzVbVHkbPkCiHfHZAA)
```csharp
ConcatSingle("a");                   // ok: prefers ConcatSingle(string?)
ConcatTuple(("a", "b"));             // ok: prefers ConcatTuple((string?, string?))
ConcatParams("a", "b", "c");         // ok: prefers ConcatParams(params string?[])
ConcatSpan(new[] { "a", "b", "c" }); // ok: prefers ConcatSpan(ReadOnlySpan<string?>)

static void ConcatSingle(object? arg) { }
static void ConcatSingle(string? arg) { }

static void ConcatTuple((object?, object?) arg) { }
static void ConcatTuple((string?, string?) arg) { }

static void ConcatParams(params object?[] args) { }
static void ConcatParams(params string?[] args) { }

static void ConcatSpan(ReadOnlySpan<object?> args) { }
static void ConcatSpan(ReadOnlySpan<string?> args) { }
```

Example 2: [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBUQAT7EDsFA3hdW7cQGy0As1AstBgAKAJSt2LcuxkCAnvwCmCABYB7DAGVYAcwA2i4QBIARGpiKTogNyzZAenvU1AaxDUADskUAzRcgBneSVVDW0YfUM6AAZxaTtg5XUMABVUDwNhYzMLEzhqUwQAdzUrG2pHZzdPbz9AxNDU9MzhGPyY0TiEhuSABQhkCDAA7PNLfMKSq1sKp1d3L19/IP4FJI1+weHhDwGhoJiAbQBdLvYAXwoJNjpufD5VkOTwyNbiaOoBnVFqJmpL+I3Li8HphXSZACSADE1MhIAgEBBgAZPshvr9/ldARw7g81o00hlDG9ou13j8vj8/gCZLcQY91k0iVlobD4YjkYp8qy4RAEUiDBS0VTMZRsXT7qCMJt9js9sMONETqidAERTT2BK8U8NvKRrstkEeeyBYplV81RiAecgA===)
```csharp
MyMethodSingle($"one");          // ok: prefers MyMethodSingle(string)
MyMethodTuple(($"one", $"two")); // ok: prefers MyMethodTuple((string, string))
MyMethodParams($"one", $"two");  // ok: prefers MyMethodParams(params string[])

static void MyMethodSingle(string arg) { }
static void MyMethodSingle(IFormattable arg) { }

static void MyMethodTuple((string, string) arg) { }
static void MyMethodTuple((IFormattable, IFormattable) arg) { }

static void MyMethodParams(params string[] args) { }
static void MyMethodParams(params IFormattable[] args) { }
```

Example 3: [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBUQAT7EDsFA3hdW7cQGy0As1AstBgAKAJSt2LcuxnUAYsQDKsAOYAbAKbDiogNyy2AekPUA9gGsQ1AA7INAMw3IAzvKWrNw2AnHSDbgBVUa09tOGpCUT12YzNLGztHF0Dg0O9w7yiJAwUABQhkCDBnMIjwgGZo2IsrWwcnVzyCopLrZuLqbwBtAF1fGQBfCmyObnw+BWUYdS1vagKVUWomaiG/Njoxifdpz2AATwQNeeRF5dXh9dHeFJCtLxgEDMelhaWVtZlNm4Ugu+FhAcjuEgRpXqd3hdKFdvuM3PlCsVhG1Ea5uj0TipnJDPuxYdsES1ke1XKDepjsec1gMgA===)
```csharp
F1Single(1);       // ok: prefers F1Single(int)
F1Tuple((1, 2));   // ok: prefers F1Tuple((int, int))
F1Params(1, 2, 3); // ok: prefers F1Params(params int[])

static void F1Single(int arg) { }
static void F1Single(byte arg) { }

static void F1Tuple((int, int) arg) { }
static void F1Tuple((byte, byte) arg) { }

static void F1Params(params int[] args) { }
static void F1Params(params byte[] args) { }
```

The modified examples work because the cases shown rely on an *identity conversion* from the *natural type* of the arguments to the element type of the preferred overload. By contrast, for *collection expressions*, there is no natural type so the conversion from the collection expression to the target type of the overload is a *collection expression conversion* in all cases.

Note however that overload resolution *will fail or behave differently* for single values, tuples, etc. when the argument does not have a *natural type*. [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAExAagD4AEBMBGAsAKAKwAJ1sB2AgbwOLtOwDZSAWYgWWhgAoBKW+jXz0RHAJ7sApgBcAFgHsEAFQCuABwA2k7twAkAInkxJ+uMQPSA7vP29eAbmIB6J8XkBrEMTUAnSQDNJHwBncSk5RVVNbW4yAAYzeLsBUTCZBWV1LR0YFQ0NMwtrWwd6F2Ign3kfLwgwYCgAcxV5FWCU0Q6RADFsKOzubDNMO0dU8o8vXwCg0N7+mNhpMyXk4VTieayYoeIEAIg86VHnV0nvP0CQzb7tnWAxaUkzB6e1kQBfAg6yZnQ2dgSdKRO6xbAJBhxXjECA+RrQqjEL7rOi/VhpCKZaI6ACS3WqkGk0ggwC0ZjxBIgRJJWmhsPhxERyJ+THRW2x3CWKxgxxhcIRSJZfzY7IGr2exHFdP5jMF+A+QA=)
```csharp
MyMethodTuple(($"one", $"two")); // ok: prefers MyMethodTuple((string, string))
MyMethodTuple((null, $"two"));   // error: ambiguous

F1Tuple((1, 2));       // ok: prefers F1Tuple((int, int))
F1Tuple((1, default)); // ok: prefers F1Tuple((byte, byte))

static void MyMethodTuple((string, string) arg) { }
static void MyMethodTuple((IFormattable, IFormattable) arg) { }

static void F1Tuple((int, int) arg) { }
static void F1Tuple((byte, byte) arg) { }
```

## Proposal

There are two proposed changes to address overload resolution with collection expressions (although the first proposed changes comes in two varieties).

The proposed changes are independent: we can take one or both.

### Proposal A: prefer `ReadOnlySpan<T>` over `ReadOnlySpan<U>`

**A1:** Extend the rule in [*better conversion*](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#overload-resolution) that prefers `ReadOnlySpan<E₁>` over `Span<E₂>` to also prefer `ReadOnlySpan<E₁>` over `ReadOnlySpan<E₂>`.

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression* and one of the following holds:
>   - `T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.Span<E₂>` ***or `System.ReadOnlySpan<E₂>`***, and an implicit conversion exists from `E₁` to `E₂`
>   - `T₁` is `System.ReadOnlySpan<E₁>` or `System.Span<E₁>`, and `T₂` is an *array_or_array_interface* with *element type* `E₂`, and an implicit conversion exists from `E₁` to `E₂`
>   - `T₁` is not a *span_type*, and `T₂` is not a *span_type*, and an implicit conversion exists from `T₁` to `T₂`
> - `E` is not a *collection expression* and one of the following holds:
>   - `E` exactly matches `T₁` and `E` does not exactly match `T₂`
>   - ...

**A2:** Remove the rules in [*better conversion*](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#overload-resolution) that directly target *collection expression conversions* and rely on *first-class spans* instead.

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` exactly matches `T₁` and `E` does not exactly match `T₂`
> - `E` exactly matches neither of `T₁` and `T₂`,
>   and `C₁` is an implicit span conversion and `C₂` is not an implicit span conversion
> - `E` exactly matches both or neither of `T₁` and `T₂`,
>   both or neither of `C₁` and `C₂` are an implicit span conversion,
>   and `T₁` is a better conversion target than `T₂`
> - `E` is a method group, `T₁` is compatible with the single best method from the method group for conversion `C₁`, and `T₂` is not compatible with the single best method from the method group for conversion `C₂`

Proposal A2 simplifies *better conversion* but it means that the *first-class span* rule that prefers `Span<T>` over `ReadOnlySpan<U>` also applies to *collection expressions*. That is the reverse of the C#12 rule which prefers `ReadOnlySpan<T>` over `Span<U>` since mutations of the collection expression are unobservable. While this seems like an undesirable change, the number of cases where there are overloads for `Span<T>` and `ReadOnlySpan<U>` may be small.

Proposal A1 and A2 address example 1, preferring `ReadOnlySpan<string>` over `ReadOnlySpan<object>`; the behavior for examples 2 and 3 is unchanged.

### Proposal B: consider collection expression *natural element type*

For rules in [*better conversion*](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-12.0/collection-expressions.md#overload-resolution) that apply to *collection expression conversion*, prefer the type with an *element type* that exactly matches the *natural element type* of the collection expression.

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression* and one of the following holds:
>   - `T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.Span<E₂>`, and **an identity conversion exists from `E₁` to `E₂` or `E₁` is a *better element type* for `E` than `E₂`**
>   - `T₁` is `System.ReadOnlySpan<E₁>` or `System.Span<E₁>`, and `T₂` is an *array_or_array_interface* with *element type* `E₂`, and **an identity conversion exists from `E₁` to `E₂` or `E₁` is a *better element type* for `E` than `E₂`**
>   - `T₁` is not a *span_type*, and `T₂` is not a *span_type*, and an implicit conversion exists from `T₁` to `T₂`
> - `E` is not a *collection expression* and one of the following holds:
>   - `E` exactly matches `T₁` and `E` does not exactly match `T₂`
>   - ...

A *better element type* is defined as:

> Given a *collection expression* `E`, a type `E₁` is a *better element type* than a type `E₂` if one of the following holds:
> - `E` has a *natural element type* `Tₑ` and an identity conversion exists from `Tₑ` to `E₁` and no identity conversion exists from `Tₑ` to `E₂`
> - `E` has a *natural element type* `Tₑ` and no identity conversion exists from `Tₑ` to `E₁` or from `Tₑ` to `E₂`, and an implicit conversion exists from `E₁` to `E₂`
> - `E` has no *natural element type*, and an implicit conversion exists from `E₁` to `E₂`

The *natural element type* of the collection expression is the [*best common type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/expressions.md#116315-finding-the-best-common-type-of-a-set-of-expressions) of the elements.
For each element `Eᵢ` in the collection expression, the type contributed to the *best common type* is the following:
* If `Eᵢ` is an *expression element*, the contribution is the *type* of `Eᵢ`. If `Eᵢ` does not have a type, there is no contribution.
* If `Eᵢ` is a *spread element* `..Sᵢ`, the contribution is the [*iteration type*](https://github.com/dotnet/csharpstandard/blob/standard-v6/standard/statements.md#1295-the-foreach-statement) of `Sᵢ`.

If there is no *best common type* of the elements, the collection expression has no natural type.

Proposal B addresses examples 2 and 3; the behavior for example 1 is unchanged.

### Proposal C: better conversion from element

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression* and `T₁` has *element type* `S₁` and `T₂` has *element type* `S₂`, and `T₁` is *no worse a collection type* than `T₂`, and the following holds:
>   - For each element `Eᵢ` in `E`:
>     - If `Eᵢ` is an expression element, the conversion from `Eᵢ` to `S₂` is not better than the conversion from `Eᵢ` to `S₁`
>     - If `Eᵢ` is a spread element with iteration type `Sᵢ`, the conversion from `Sᵢ` to `S₂` is not better than the conversion from `Sᵢ` to `S₁`
>   - For at least one element `Eᵢ` in `E`:
>     - If `Eᵢ` is an expression element, the conversion from `Eᵢ` to `S₁` is better than the conversion from `Eᵢ` to `S₂`
>     - If `Eᵢ` is a spread element with iteration type `Sᵢ`, the conversion from `Sᵢ` to `S₁` is better than the conversion from `Sᵢ` to `S₂`
> - `E` is not a *collection expression* and one of the following holds:
>   - ...

> `T₁` is a *no worse a collection type* than `T₂` if one of the following holds:
> - `T₁` is a type `S<A1, ..., An>`, and `T₂` is type `S<B1, ..., Bn>`
> - `T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.Span<E₂>`
> - `T₁` is `System.ReadOnlySpan<E₁>` or `System.Span<E₁>` or `E₁[]`, and `T₂` is an *array_or_array_interface*
> - `T₁` is not a *span_type*, and `T₂` is not a *span_type*, and an implicit conversion exists from `T₁` to `T₂`


## Open questions
Should we allow preferring `Span<T>` over `Span<U>` when `T` is a better element type than `U`?

Should we allow preferring `T[]` over `U[]`, or `IEnumerable<T>` over `IEnumerable<U>`, when `T` is a better element type than `U`? We currently only allow that when there is an implicit reference conversion from `T` to `U`.

Should the proposed changes be tied to `-langversion:13` and higher?

## Issues

- https://github.com/dotnet/roslyn/issues/73857
- https://github.com/dotnet/csharplang/issues/7651#issuecomment-1792778421
