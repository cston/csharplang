# Ref struct overloads in Expression trees

C#13 adds support for *`params` span*, and C# preview adds *first-class span types*.
Both of those features allow overload resolution to prefer overloads with *span type* parameters in more cases than previous language versions.

However, in `Expression` trees, choosing an overload that requires a `ref struct` instance may result in errors at compile time or runtime, and is a breaking change (see [#109757](https://github.com/dotnet/runtime/issues/109757), [#110592](https://github.com/dotnet/runtime/issues/110592)).

## Proposal

Update overload resolution to ignore candidate methods with `ref struct` parameters within `Expression` trees. 

[*12.6.4.2 Applicable function member*](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/expressions.md#12642-applicable-function-member) is updated as follows:

> - ...
> - **Within an `Expression`, if any parameters of the candidate method, other than `this`, may have a `ref struct` type, the candidate is not applicable.**

Note that the disqualifying parameter:
- May be an optional parameter
- May have a *generic type parameter* type with `allows ref struct` constraint

*What about method group conversions? Are we considering overloads with `ref struct` parameters in those cases?*

## Drawbacks

Ignoring certain overloads may be a breaking change itself, in limited scenarios where `ref struct` instances are supported in `Expression` trees currently. *Examples?*

## Alternatives

### No change / IDE fixer

Make no additional compiler changes. Instead, require callers to rewrite `Expression` instances to work around the overload resolution change.

An IDE *fixer* could be provided to rewrite calls in these cases.

### Support `ref struct` in `Expression` trees

Supporting `ref struct` instances in `Expression` trees is non-trivial.
Since the [`Expression` *interpreter*](https://learn.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression-1.compile?view=net-9.0#system-linq-expressions-expression-1-compile(system-boolean)) relies on *reflection*, this would require supporting `ref struct` in reflection or rewriting parts of the interpreter.

## Design meetings

- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-06-17.md#params-span-breaks
- https://github.com/dotnet/csharplang/blob/main/meetings/2024/LDM-2024-12-04.md#conversions-in-expression-trees
