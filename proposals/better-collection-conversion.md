# Better conversion from collection expression with `ReadOnlySpan<T>` overloads

## Summary

.NET 9 includes overloads for existing methods with parameters that diff by `ReadOnlySpan<string>` and `ReadOnlySpan<object>` only. However, the betterness rules for *collection expressions* do not currently prefer one of those types over the other, so calls to such methods with collection expressions of strings are ambiguous. (The cases are ambiguous even with the [*first-class spans*](https://github.com/dotnet/csharplang/blob/main/proposals/first-class-span-types.md) feature.) See [dotnet/roslyn/issues/73857](https://github.com/dotnet/roslyn/issues/73857), [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUBQMoLKwDmAdAMID2MAxhAgBQDaAIggC4AAgHAR4jgIC6ASgDcFCkVE06MehQDeFUQdH4AzEeIA2IwBZRbTtx4AlAKYQMAeRgAbAJ5UADhAwADyswABWzhwIAPwAfKIAbhBeqM4AzvKiOqIAvvqGJmaW+DZ2XLwubp6+AUHB+MQADPFJKWmZ2XkUuUA==).

```csharp
string.Concat(["a", "b", "c"]); // error: call is ambiguous

public sealed class String
{
    // ...
    public static void Concat(ReadOnlySpan<object?> values) { }
    public static void Concat(ReadOnlySpan<string?> values) { }
}
```

Overload resolution for *collection expressions* should be updated to resolve cases with `ReadOnlySpan<T>`.

## Proposal

The **following changes** are proposed for [*better conversion from expression*](https://github.com/dotnet/csharplang/blob/main/proposals/first-class-span-types.md#better-conversion-from-expression):

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression* and one of the following holds:
>   - **`T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.ReadOnlySpan<E₂>`, and an implicit non-numeric conversion exists from `E₁` to `E₂`, and no implicit conversion from `E₂` to `E₁` exists**.
>   - `T₁` is `System.ReadOnlySpan<E₁>`, and `T₂` is `System.Span<E₂>`, and an implicit **non-numeric** conversion exists from `E₁` to `E₂`.
>   - `T₁` is `System.ReadOnlySpan<E₁>` or `System.Span<E₁>`, and `T₂` is an *array_or_array_interface* with *element type* `E₂`, and an implicit **non-numeric** conversion exists from `E₁` to `E₂`.
>   - `T₁` is not a *span_type*, and `T₂` is not a *span_type*, and an implicit conversion exists from `T₁` to `T₂`.
> - `E` is not a *collection expression* and one of the following holds:
>   - ...

The new rules apply when compiling with `-langversion:13` and higher; the existing rules apply with `-langversion:12` and lower.

The proposal represents two changes to the existing rules.

The first change is a rule preferring `ReadOnlySpan<T>` over `ReadOnlySpan<U>`. The existing rules prefer `ReadOnlySpan<T>` over `Span<U>`, `U[]` or an interface implemented by `U[]`, but no preference when both are the same span type.

The new rule requires an implicit conversion between element types that is *not a numeric conversion*. The reason to disallow numeric conversions is to avoid binding differently from similar cases such as `params`. See `F1([1, 2, 3])` below which would bind to *`F1(ReadOnlySpan<byte>)`* if numeric conversions were allowed. Instead, the call remains ambiguous. (To bind this case successfully, see [alternative](#better-conversion-from-element) below.)

```csharp
F1([1, 2, 3]); // C#12: ambiguous, C#13: ambiguous
F1(1, 2, 3);   // F1(params ReadOnlySpan<int>)

static void F1(params ReadOnlySpan<byte> args) { }
static void F1(params ReadOnlySpan<int> args) { }
```

The second change in the proposal disallows implicit *numeric conversions* between element types in the *existing rules*. The reason is the same, to avoid binding differently from similar cases such as `params`, and also for consistency with the new rule. See `F2()` and `F3()` below.

```csharp
F2([1, 2, 3]); // C#12: F2(ReadOnlySpan<byte>), C#13: ambiguous
F2(1, 2, 3);   // F2(params int[])

static void F2(params ReadOnlySpan<byte> args) { }
static void F2(params int[] args) { }

F3([1, 2, 3]); // C#12: ambiguous, C#13: ambiguous
F3(1, 2, 3);   // F3(params ReadOnlySpan<int>)

static void F3(params ReadOnlySpan<int> args) { }
static void F3(params byte[] args) { }
```

The second change, disallowing numeric conversions between element types in the existing rules, is a breaking change. But the break only affects pairs of overloads where the containing collection types are distinct and a combination of `ReadOnlySpan<>` and `Span<>`, or a span type and an array or array interface, and where the element types are distinct numeric types.

To work around the breaking change, cast the collection expression argument, or add `[OverloadResolutionPriority]` to the preferred method.

*Are there any APIs in the BCL affected by the numeric conversion breaking change?*

## Alternatives

### Avoid breaking change, at least for now

We could skip the second part of the proposal, disallowing numeric conversions in existing rules. That would avoid the breaking change. 

However, we will likely want to improve the betterness rules in the future to handle more cases, particularly cases that bind successfully with `params` currently (see [alternative](#better-conversion-from-element) below). When we do that, we'll hit the same breaking changes.

### Extend same collection type rule to other collection types

The new rule targets `ReadOnlySpan<T>` only.

Instead, we could generalize the rule to apply to more collection types:

> - `E` is a *collection expression* and one of the following holds:
>   - **`T₁` and `T₂` are substituted from the same collection type with element types `E₁` and `E₂`, and an implicit non-numeric conversion exists from `E₁` to `E₂`, and no implicit conversion from `E₂` to `E₁` exists**.
>   - ...

Some possible options for the collection types supported:
- `ReadOnlySpan<T>` only *as proposed*
- `ReadOnlySpan<T>`, `Span<T>`
- `ReadOnlySpan<T>`, `Span<T>`, `T[]`
- *Any collection type*

### Better conversion from *element*

In the earlier examples, overload resolution succeeded with `params` in cases where collection expressions failed.

```csharp
F1([1, 2, 3]); // error: ambiguous
F1(1, 2, 3);   // F1(params ReadOnlySpan<int>)

static void F1(params ReadOnlySpan<byte> args) { }
static void F1(params ReadOnlySpan<int> args) { }

...
```

The same is true for overloads where there is no relationship between the element types (see https://github.com/dotnet/csharplang/issues/7651, [sharplab.io](https://sharplab.io/#v2:EYLgtghglgdgNAFxAJwK7wCYgNQB8ACATAIwCwAUPgAwAE+xALANwUVF3EDsFA3hTQI4A2OgxoBZaDAAUASn6C+5QSokBPcQFMEACwD2GaQG0AJACI9MTWbg1zCAO56zAXVlMaAek81NyZHrIIDQQYMBQAOaoeqgAzgqq6lq6BtLmlta29k5m7oLeSdr6hgAOEMihsRwAzAA89FQAfPLKggC+rK0C9CL4YuIaRallFWBV9HUNjTQA+rI0PDQdXcKihSml5ZU1tQCSAGKBkAgIEMAANprTcwtLFG1AA==)):

```csharp
MyMethod([$"one", $"two"]); // error: ambiguous
MyMethod($"one", $"two");   // MyMethod(params IEnumerable<string>)

static void MyMethod(params IEnumerable<string> _) { }
static void MyMethod(params IEnumerable<IFormattable> _) { }
```

The reason the `params` cases succeed is that better conversion from expression is applied to the *elements* rather than the collection.

We could do something similar for collection expressions if *better conversion from expression* recurses into the collections.

For instance, assuming some rule for deciding one *collection type definition* is preferred over another, the rules might be:

> Given an implicit conversion `C₁` that converts from an expression `E` to a type `T₁`, and an implicit conversion `C₂` that converts from an expression `E` to a type `T₂`, `C₁` is a *better conversion* than `C₂` if one of the following holds:
>
> - `E` is a *collection expression*, **and `T₁` has *element type* `S₁`, and `T₂` has *element type* `S₂`, and `T₁` is *no worse a collection type* than `T₂`**, and the following holds:
>   - **For each element `Eᵢ` in `E`:**
>     - **If `Eᵢ` is an expression element, the conversion from `Eᵢ` to `S₂` is not better than the conversion from `Eᵢ` to `S₁`**
>     - **If `Eᵢ` is a spread element with iteration type `Sᵢ`, the conversion from `Sᵢ` to `S₂` is not better than the conversion from `Sᵢ` to `S₁`**
>   - **For at least one element `Eᵢ` in `E`:**
>     - **If `Eᵢ` is an expression element, the conversion from `Eᵢ` to `S₁` is better than the conversion from `Eᵢ` to `S₂`**
>     - **If `Eᵢ` is a spread element with iteration type `Sᵢ`, the conversion from `Sᵢ` to `S₁` is better than the conversion from `Sᵢ` to `S₂`**
> - `E` is not a *collection expression* and one of the following holds:
>   - ...

With the rules above, and assuming we prefer `ReadOnlySpan<T>` over `T[]`, the examples would bind as:

```csharp
F1([1, 2, 3]); // F1(params ReadOnlySpan<int>)
F1(1, 2, 3);   // F1(params ReadOnlySpan<int>)

static void F1(params ReadOnlySpan<byte> args) { }
static void F1(params ReadOnlySpan<int> args) { }

F2([1, 2, 3]); // error: ambiguous
F2(1, 2, 3);   // F2(params int[])

static void F2(params ReadOnlySpan<byte> args) { }
static void F2(params int[] args) { }

F3([1, 2, 3]); // F3(params ReadOnlySpan<int>)
F3(1, 2, 3);   // F3(params ReadOnlySpan<int>)

static void F3(params ReadOnlySpan<int> args) { }
static void F3(params byte[] args) { }

MyMethod([$"one", $"two"]); // MyMethod(params IEnumerable<string>)
MyMethod($"one", $"two");   // MyMethod(params IEnumerable<string>)

static void MyMethod(params IEnumerable<string> _) { }
static void MyMethod(params IEnumerable<IFormattable> _) { }
```

This alternative is a more significant change than the original proposal, which brings higher risk. The expectation is we could start with the original proposal and move to this alternative in a future release.

## Issues

- https://github.com/dotnet/roslyn/issues/73857
- https://github.com/dotnet/csharplang/issues/7651#issuecomment-1792778421
