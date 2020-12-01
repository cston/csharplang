# Discriminated unions

## Summary

Discriminated unions are types with named members with optional types where each instance of a discriminated union is represented by exactly one of the members.

## Design
Discriminated unions are declared as a set of zero or more members in an `enum class` or `enum struct`.
Each member is public and has a name and an optional type.
```C#
enum struct Option<T>
{
    None,
    T Some,
}
```

An `enum class` or an `enum struct` is represented as a synthesized `class` or `struct` type.
The synthesized type includes `static` properties for each untyped member and `static` factory methods for typed members.
Instances of the discriminated union are created using the `static` members.
```C#
// synthesized type
struct Option<T>
{
    public static Option<T> None { get; }
    public static Option<T> Some(T value);
}

Option<T> x = Option<T>.None;
Option<T> y = Option<T>.Some(default(T));
```

_What is the value of `default(Option<T>)` or `new Option<T>()`?_

The synthesized type includes an `enum Tag` type to identify the member of each instance, and instance accessor methods for typed members.
```C#
// synthesized type
struct Option<T>
{
    public enum _Tag { None = 0, Some = 1 };
    public _Tag Tag { get; }
    public T GetSome();
}
```
_Should `enum _Tag { }` be declared outside of `Option<T>` to avoid making it generic?_

Member names can be used in pattern matching. Untyped members use the syntax of constant patterns and typed members use the syntax of property patterns.
```C#
static T GetValue<T>(Option<T> option) =>
    option switch
    {
        None => default,
        Some: var t => t
    };

static T GetValue<T>(object obj) =>
    obj switch
    {
        Option<T> { None } => default,
        Option<T> { Some: var t } => t,
        _ => throw new Exception()
    };
```

The `Tag` property and accessor methods are used by the compiler for pattern matching. For instance, the condition `if (option is Some: var t) { ... }` is lowered as:
```C#
// if (option is Some: var t) { ... }
if (option.Tag == Option<T>._Tag.Some)
{
    T t = option.GetSome();
    ...
}
```

The discriminated union type does not define any implicit or explicit conversions between typed members and the DU type. For instance, `Option<T>` does not define any conversions between `T` and `Option<T>`. 

## Syntax
```antlr
discriminated_union
    : attribute_list* modifier* 'enum' ('class' | 'struct')? identifier
      type_parameter_list? type_parameter_constraint_clause*
      '{' (discriminated_union_member (',' discriminated_union_member)*)? ','?'}'
    ;

discriminated_union_member
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

## Synthesized type
The synthesized types are similar to the types synthesized for discriminated unions in F#.

An `enum class` defines an `abstract class` with derived types for members. Consider `enum class Option<T> { ... }`:
```C#
// synthesized type
abstract class Option<T>
{
    public enum _Tag { None = 0, Some = 1 };

    public static Option<T> None { get; } = new _None();
    public static Option<T> Some(T value) => new _Some(value);

    public abstract _Tag Tag { get; }
    public T GetSome() => ((_Some)this).Value;

    private sealed class _None : Option<T>
    {
        public override _Tag Tag => _Tag.None;
    }

    private sealed class _Some : Option<T>
    {
        public override _Tag Tag => _Tag.Some;
        public readonly T Value;
        public _Some(T value)
        {
            Value = value;
        }
    }
}
```

An `enum struct` defines a synthesized `struct` with properties for members. Consider `enum struct Option<T> { ... }`:
```C#
// synthesized type
struct Option<T>
{
    public enum _Tag { None = 0, Some = 1 };

    public static Option<T> None { get; } = new Option<T>(_Tag.None, default);
    public static Option<T> Some(T value) => new Option<T>(_Tag.Some, value);

    public _Tag Tag { get; }
    public T GetSome() => Tag == _Tag.Some ? _some : throw new Exception();

    private Option(_Tag tag, T some)
    {
        Tag = tag;
        _some = some;
    }
    private readonly T _some;
}
```

_Describe sharing of `struct` fields between discriminated union members._

_How do we recognize a DU from metadata?_

## Design meetings

- https://github.com/dotnet/csharplang/blob/master/meetings/2019/LDM-2019-11-13.md#initial-discriminated-union-proposal
