# Struct delegates

## Summary

Support struct delegates and closures for lambda expressions to avoid heap allocations when the delegate and closure do not escape the scope where they are created.

## Detailed design

The design requires several changes:
- *Functional interfaces* for delegate definitions and implementations.
- *Partial type inference* to avoid specifying synthesized delegate types at call sites.
- *Ref analysis* updates to allow references to nested struct closures.

### Functional interfaces

*See [#3452](https://github.com/dotnet/csharplang/issues/3452) and related.*

[*10.7 Anonymous function conversions*](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/conversions.md#107-anonymous-function-conversions) is updated as follows, to allow conversion to a type parameter constrained to an interface with a single compatible method:

> An *anonymous_method_expression* or *lambda_expression* is classified as an anonymous function ([§12.19](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/expressions.md#1219-anonymous-function-expressions)). The expression does not have a type, but can be implicitly converted to a compatible delegate type **or to a compatible type parameter**. Some lambda expressions may also be implicitly converted to a compatible expression tree type.
>
> **An anonymous function `F` is compatible with a type parameter `T` provided:**
> - **`T` has a single *interface_type* constraint and no *class_type* constraint, and**
> - **The *interface_type* has a single member declared in the interface or in a base interface, and**
> - **The interface member is an instance method `M`, and**
> - **`F` is compatible with the delegate type equivalent of the signature of `M`.**

The compiler will synthesize a type for the lambda expression that implements the *interface_type* constraint.
The synthesized type is both the delegate and the closure.

The synthesized type is *not* a `System.Delegate`.

The synthesized type is created as:
- If the type parameter has a `class` constraint, the synthesized type is a `class`.
- Otherwise, if none of the captured variables may be value types, the synthesized type is a `struct`.
- Otherwise, if the type parameter has `allows ref struct` and the compiler can determine the synthesized type *instance* does not escape the calling context, the synthesized type is a `ref struct`.
- Otherwise, the synthesized type is a `class`.

### Partial type inference

*See [#7467](https://github.com/dotnet/csharplang/discussions/7467).*

Delegate types synthesized for lambda expressions will have unspeakable names that prevent those type names from being specified explicitly in source as type arguments.
However, there may be scenarios where *other* type arguments to a generic method cannot be inferred and must be specified explicitly.
For those cases, *partial type inference* is needed for the delegate types.

### Ref analysis

A `ref struct` closure will contain a `readonly ref` field to another `ref struct` when a `ref struct` is captured and mutable. (The captured `ref struct` may be a variable or another closure.)
The compiler does not support ref fields to `ref struct` instances currently, because we haven't had scenarios that justified the changes necessary for ref analysis of such cases.

We could use closures as the justification, and allow `ref` fields to `ref struct` instances *from source*, and define the ref safety rules needed.
For now though, we should just limit support to compiler-generated closures only.
For closures, the compiler will ensure that the ref field refers to an instance whose lifetime completely encloses the lifetime of the reference, so no additional ref safety rules are needed.

*Give example that shows generated code for nested closure with reference.*
