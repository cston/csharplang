# Union struct layout

A possible field layout for `union struct` definitions.

## Proposal

The following represents one possible layout algorithm for fields in the emitted struct.
It is not the most space-efficient in all cases.
For example, the algorithm does not attempt to reorder fields or separate fields within a member, even though that might allow overlapping parts of members in additional cases.

Each union *member* has 0 or more fields. That member is represented as a `struct` definition with instance fields for field in declaration order.
The fields in the member struct definition use `[StructLayout(LayoutKind.Auto)]`.
If the user needs to control the layout of fields within the member, a custom type should be declared for the member outside the union.

It is an error to apply `[StructLayout]` to a `union` declaration.

Layout of the emitted struct for the union is determined with the following steps.

1. Emit an instance field for the tag, at offset 0, and set the next available offset.
1. For each member struct with *known size*, in order from largest to smallest:
    1. Find the first member struct already emitted with a *matching* subset of fields.
    1. If a member was found, emit an instance field for the member struct at the offset of the first matching field.
    1. Otherwise, if a member was not found, emit an instance field for the member struct at the next available offset, and advance the next available offset.
1. For the remaining member structs, with *unknown size*, if all are *unmanaged types* then:
    1. For each member struct with unknown size, emit an instance field for the member struct at the next available offset, and *do not* advance the next available offset.
1. Otherwise, if there are any member structs with *unknown size* and *managed type* then:
    1. A wrapper struct is created.
    1. For each member struct with unknown size, emit an instance field for the member struct in the wrapper struct.
    1. Emit an instance of the wrapper struct in the union struct at the next available offset.

To overlap member structs with *known size*, we depend on *matching* fields in the overlapping structs.
Matching involves comparing the fields (or subsets of fields), in order, between the two structs, and for each pair of field types, including *ref-ness*:
- If the types are *unmanaged* and the same size, the fields match.
- If the types are *reference types*, the fields match.
- Otherwise, the fields do not match.

## Examples

Example 1:
```csharp
union struct U1<T>
    where T : class
{
    A(int X, string Y),
    B(string Z),
    C(T T),
}
```

```csharp
[StructLayout(LayoutKind.Explicit)]
struct U1<T>
    where T : class
{
    struct A { int X; string Y; }
    struct B { string Z; }
    struct C { T T; }
    [FieldOffset(0)] int _tag;
    [FieldOffset(4)] A _a;
    [FieldOffset(8)] B _b; // overlap A.Y, B.Z
    [FieldOffset(8)] C _c; // overlap A.Y, B.Z, C.T
}
```

Example 2:
```csharp
union struct U2<T, U>
    where T : unmanaged
    where U : unmanaged
{
    A(T T),
    B(U U),
    C(int X, int Y),
    D(double X, double Y),
}
```

```csharp
[StructLayout(LayoutKind.Explicit)]
struct U2<T, U>
    where T : unmanaged
    where U : unmanaged
{
    struct A { T T; }
    struct B { U U; }
    struct C { int X; int Y; }
    struct D { double X; double Y; }
    [FieldOffset(0)]  int _tag;
    [FieldOffset(4)]  D _d;
    [FieldOffset(4)]  C _c; // overlap D, C
    [FieldOffset(20)] A _a;
    [FieldOffset(20)] B _b; // overlap A, B
}
```

Example 3:
```csharp
union struct U1<T, U>
{
    A(T T),
    B(U U),
}
```

```csharp
[StructLayout(LayoutKind.Explicit)]
struct U1<T, U>
{
    struct A { T T; }
    struct B { U U; }
    struct _UnknownSizes { A _a; B _b; } // no overlap
    [FieldOffset(0)] int _tag;
    [FieldOffset(4)] _UnknownSizes _unknownSizes;
}
```
