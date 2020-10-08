# Discriminated unions

## Syntax
Discriminated unions are declared as a set of zero or more members in an `enum class` or `enum struct`.
Each member has a name and an optional type.

```antlr
discriminated_union
    : attribute_list* modifier* 'enum' ('class' | 'struct')? identifier
      type_parameter_list? type_parameter_constraint_clause*
      '{' (discriminated_union_member (',' discriminated_union_member)*)? ','?'}'
    ;

discriminated_union_member
    : attribute_list* type? identifier

modifier
    : 'public'
    | 'internal'
    | 'protected'
    | 'private'
    | 'unsafe'
    ;

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

Examples:
```C#
record Rectangle(float Width, float Length);
record Circle(float Radius);
enum class Shape
{
    Rectangle Rect,
    Circle Circ,
}

enum struct Option<T> where T : notnull
{
    None,
    T Some,
}
```

All members are `public`.

An `enum class` defines an `abstract class` with derived types for typed members.
An `enum struct` defines a `struct` with properties for typed members.

## Nested member type definitions
The syntax could be extended to allow defining member types directly within the `enum class` or `enum struct`.
```C#
enum class Shape
{
    Rectangle(float Width, float Length) Rect,
    Circle(float Radius) Circ,
}
```

The type definitions above are derived classes of the generated base type.
```C#
abstract class Shape
{
    public record Rectangle(float Width, float Length) : Shape;
    public record Circle(float Radius) : Shape;
}
```

There is a distinct difference with `enum class` between a member of an external type and a member of a nested type. In both cases, members are represented as instances of a nested type, but when the member type is defined externally, the nested type results in an additional wrapper instance.

## Construction
For `enum class`, each member is represented with a unique type derived from the `abstract` base class.
Untyped members are singleton instances of derived types.
```C#
Shape shape;
shape = new Shape.Rectangle(width, length);   // Rectangle type defined inline
shape = new Shape.Circle(new Circle(radius)); // Circle type defined externally
```

For `enum struct`, each member is represented as a property on the `struct` class.
Constructor overloads are generated for reference types, and for each unique value type. The overloads take a `int tag` parameter to indicate the member.
If the member can be unambiguously determined from the type, a constructor overload without `int tag` is also generated.
Untyped members are `static readonly` fields.
```C#
Some<int> s;
s = Some<int>.None;
s = new Some<int>(42);
s = new Some<int>(Some<int>.Tags.Some, 42);
```

## Conversions
For `enum class` and `enum struct`, there is an implicit conversion operator for each unique member type to the DU type, and an explicit conversion operator from the DU type to each unique member type.
If the member type is not unique, or the member type is a base or interface of the DU type, no conversion operators are provided.
```C#
Circle circle;
Shape shape = circle;   // implicit conversion
circle = (Circle)shape; // explicit conversion
```

## Pattern matching
For `switch` statements and expressions, and `is` patterns, when the expression is an instance of a DU, pattern matching is considered to apply to members of the DU rather than the DU instance.
Pattern matching supports matching members by name in addition to existing patterns.
If the set of patterns covers all members of the enum, then the switch will be considered exhaustive.
```C#
Shape shape = ...;
float width = shape switch
{
    Rectangle r => r.Width, // match member by type
    Circ: c => c.Diameter,  // match member by name
};
```

The equivalent expressed with C#9.
```C#
float width = shape.Tag switch
{
    Shape.Tags.Rectangle => ((Rectangle)shape).Width,
    Shape.Tags.Circle => ((Circle)shape).Diameter,
    _ => throw new Exception()
};
```

If the member is untyped, the member name can be used in a pattern without `:`.
```C#
Some<int> s = ...;
int? value = s switch
{
    None => null,
    var i => i
};
```

The pattern `null` matches to the DU instance rather than any member.
```C#
Shape? shape = ...;
float? width = shape switch
{
    Rect: r => r.Width,
    Circ: c => c.Diameter,
    null => null, // shape is null
};
```

## Synthesized type
The synthesized types follow closely those generated for discriminated unions in F#.

An `enum class` is an `abstract class` with members as derived types.
Consider `enum class Shape { None, Rectangle Rect, Circle Circ }`.
```C#
abstract class Shape
{
    public sealed class Tags
    {
        public const int None = 0;
        public const int Rect = 1;
        public const int Circ = 2;
    }

    public abstract int Tag { get; }

    public static implicit operator Shape(Rectangle rectangle) => new Rect(rectangle);
    public static implicit operator Shape(Circle circle) => new Circ(circle);

    public static explicit operator Rectangle(Shape shape) => ((Rect)shape).Rectangle;
    public static explicit operator Circle(Shape shape) => ((Circ)shape).Circle;

    public sealed class None : Shape
    {
        public static readonly None Instance = new None();
        public override int Tag => Tags.None;
    }

    public sealed class Rect : Shape
    {
        public Rect(Rectangle rectangle) { Rectangle = rectangle; }
        public Rectangle Rectangle { get; }
        public override int Tag => Tags.Rect;
    }

    public sealed class Circ : Shape
    {
        public Circ(Circle circle) { Circle = circle; }
        public Circle Circle { get; }
        public override int Tag => Tags.Circ;
    }
}
```

An `enum struct` is a `struct` with members as properties.
Consider `enum struct Shape { ... }`.
```C#
struct Shape
{
    public sealed class Tags
    {
        public const int None = 0;
        public const int Rect = 1;
        public const int Circ = 2;
    }

    public static readonly Shape None = new Shape(Tags.None, null);

    public Shape(int tag, object member)
    {
        Tag = tag;
        _member = member;
    }

    public Shape(Rectangle rectangle) : this(Tags.Rect, rectangle) { }
    public Shape(Circle circle) : this(Tags.Circ, circle) { }

    public int Tag { get; }

    private readonly object _member;

    public Rectangle Rect => (Rectangle)_member;
    public Circle Circ => (Circle)_member;

    public static implicit operator Shape(Rectangle rectangle) => new Shape(rectangle);
    public static implicit operator Shape(Circle circle) => new Shape(circle);

    public static explicit operator Rectangle(Shape shape) => shape.Rect;
    public static explicit operator Circle(Shape shape) => shape.Circ;
}
```

_How do we recognize a DU from metadata?_

## Design meetings

- https://github.com/dotnet/csharplang/blob/master/meetings/2019/LDM-2019-11-13.md#initial-discriminated-union-proposal
