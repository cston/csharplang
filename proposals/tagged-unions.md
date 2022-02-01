# Discriminated unions

* [x] Proposed
* [ ] Prototype: Not Started
* [ ] Implementation: Not Started
* [ ] Specification: Not Started

## Summary
[summary]: #summary

Support _discriminated unions_ that represent _union types_.
Provide a low overhead implementation with language support for construction, deconstruction, and pattern matching.

There are several discussions of _discriminated unions_ on [dotnet/csharplang](https://github.com/dotnet/csharplang) with most discussions covering _union types_ as well as _closed type hierarchies_ (where all members have a common base type).
This proposal focuses on _union types_, with the assumption that _closed type hierarchies_ is a distinct feature that can be designed later, even though there may be overlap with this proposal.

The term _discriminated union_ in this proposal is intended to refer to _union types_ unless otherwise indicated.

## Detailed design
[design]: #detailed-design

A _discriminated union_ is declared as an `enum struct`.
A _discriminated union_ is a runtime type, derived from `System.ValueType`.
The type may be generic.

A _discriminated union_ declaration has zero or more members.
Each member is public and has a name and an optional type.
Member types are defined outside the _discriminated union_.
```csharp
enum struct Option<T> { None, T Some }

enum struct Result<T> { T Value, string Error }
```

### Construction
Instances are readonly.

Instances of untyped members are available from singletons; instances of typed members are constructed from factory methods.
```csharp
Option<string> empty = Option<string>.None;
Option<string> hello = Option<string>.Some("hello");
```

Untyped members such as `Option<T>.None` are considered compile-time constants.

_What is the value of `new Option<string>()`? Should a warning be reported for `default` or `new()`?_

_Should we allow implicit conversions from member types to discriminated union type to simplify construction when a member type is unique? `Option<string> hello = "hello";`_

### Deconstruction, pattern matching
Members are accessed through pattern matching with member names.
Untyped members use constant pattern syntax; typed members use property pattern syntax.
Member names in patterns are unqualified.
```csharp
Option<T> option = ...;

// switch expression
T t = option switch { None => default, Some: var s => s };

// conditional expression
t = option is Some: var s ? s : default;
```

Pattern matching on member type is not supported.

Direct access of typed members is supported through property syntax.
There is no equivalent for untyped members.
```csharp
string str = hello.Some; // throws if None
```

There are no implicit casts from _discriminated union_ type to member type.

_Should explicit casts and `as` be supported for types that represent a unique member?_
```csharp
str = (string)hello;   // throws if None
str = hello as string; // null if None
```

### Equality
`==`, `!=`, and `Equals()` compare tags and, for typed members, invoke `==`, `!=`, or `Equals()` on the particular typed member.

### Interfaces
A _discriminated union_ type implements `IEquatable<T>` and no other interfaces.

### Synthesized type
An `enum struct` is represented as a synthesized `struct` type, similar to the representation generated from F#.
```csharp
// enum struct Option<T> { None, Some T }
[UnionType]
struct Option<T> : IEquatable<Option<T>>
{
    public enum _Tag { None = 0, Some = 1 };

    public static Option<T> None { get; } = new Option<T>(_Tag.None, default);
    public static Option<T> CreateSome(T value) => new Option<T>(_Tag.Some, value);

    public readonly _Tag Tag;
    public readonly T Some;

    private Option(_Tag tag, T some)
    {
        Tag = tag;
        Some = some;
    }
    
    public T GetSome() => Tag == _Tag.Some ? Some : throw new InvalidOperationException();

    public static bool operator==(Option<T> a, Option<T> b) { ... }
    public static bool operator!=(Option<T> a, Option<T> b) { ... }

    public bool Equals(Option<T> other) { ... }
    public override bool Equals(object other) { ... }
}
```

_Are `public` members available for direct use? Can the `Tag` field be used directly for instance?_

_Should `enum _Tag { }` be declared outside of `Option<T>` to avoid making it generic, or should the `enum _Tag` type be replaced with `int` constants instead, as with F#?_

Fields in the synthesized type may be shared across multiple members: members that are reference types share the same field regardless of actual type; members with identical runtime types share the same field.

The synthesized type is annotated with a specific attribute in metadata.

_What is the shape of the synthesized type that the compiler expects for an `enum struct`?_

_Could a 3rd party define the synthesized type directly, to control the implementation, rather than declaring an `enum struct`?_

### Syntax
```antlr
discriminated_union
    : attribute_list* modifier* 'enum' 'struct' identifier
      type_parameter_list? type_parameter_constraint_clause*
      '{' (du_member (',' du_member)*)? ','?'}'
    ;

du_member
    : attribute_list* type? identifier

type
    : type_name
    | array_type
    | function_pointer_type
    | pointer_type
    | nullable_type
    | predefined_type
    | tuple_type
    ;
```

## Drawbacks
[drawbacks]: #drawbacks

## Alternatives
[alternatives]: #alternatives

## Unresolved questions
[unresolved]: #unresolved-questions

## Related
[related]: #related

- Issues [#113](https://github.com/dotnet/csharplang/issues/113), [#485](https://github.com/dotnet/csharplang/issues/485)
- Language Design Meetings: [2019-11-13](https://github.com/dotnet/csharplang/blob/main/meetings/2019/LDM-2019-11-13.md#initial-discriminated-union-proposal)
- Discussions [#2962](https://github.com/dotnet/csharplang/discussions/2962), [#3760](https://github.com/dotnet/csharplang/discussions/3760)

