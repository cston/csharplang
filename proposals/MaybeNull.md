# MaybeNullAttribute

Allow marking references to unconstrained type parameters as nullable.

## Motivation
The syntax `T?` is only supported for type parameters that are constrained to either reference types or values.
However there are scenarios where a reference to a  type parameter without such a constraint may allow a default value.

`new List<string>().FirstOrDefault()` will return `null`.
```C#
public static class Enumerable
{
    // ...
    public static [MaybeNull]TSource FirstOrDefault<TSource>(IEnumerable<TSource> source);
}
```

`new Dictionary<string, object>().TryGetValue(key, out value)` will set `value` to `null`.
```C#
public class Dictionary<TKey, TValue>
{
    // ...
    public bool TryGetValue(TKey key, [MaybeNull][NotNullWhenTrue]out TValue value) { ... }
}
```

Implementations of `IEqualityComparer<T>` should allow `null` for arguments to `Equals()` but not for `GetHashCode()` when `T` is a reference type.
```C#
public interface IEqualityComparer<in T>
{
    bool Equals([MaybeNull]T x, [MaybeNull]T y);
    int GetHashCode(T obj);
}

public abstract class StringComparer : IEqualityComparer<string>
{
    public abstract bool Equals(string? x, string? y);
    public abstract int GetHashCode(string obj);
}
```

## Alternatives
There are two obvious alternatives: do not allow annotating unconstrained type parameters; or use `T?` syntax.

If there is no mechanism to annotate unconstrained type parameters, then affected declarations in
nullable-aware contexts will be annotated incorrectly. The only work around would be to
wrap such declarations in `#nullable disable`.

If we allow `T?` for an unconstrained type parameter, the meaning of `T?` may be surprising
when `T` is substituted with a value type, given the expected behavior.

```C#
_ = new List<string?>().FirstOrDefault(); // string?
_ = new List<string>().FirstOrDefault();  // string?
_ = new List<int?>().FirstOrDefault();    // int?
_ = new List<int>().FirstOrDefault();     // int
_ = new List<T>().FirstOrDefault();       // T
```

If the syntax `T?` is allowed regardless of the constraints on `T`, we will need syntax to
indicate no reference type or value type constraint. See `B.F<T>()` below.

```C#
abstract class A
{
    public abstract void F<T>(T? t) where T : struct;
    public abstract void F<T>(T? t);
}

abstract class B : A
{
    public override void F<T>(T? t) where T : object? { }
}
```

## Detailed design

### Attribute
The attribute is matched by namespace-qualified name.
The only constructor that is recognized is the parameter-less constructor.
Binding follows normal lookup and accessibility rules.

```C#
namespace System.Runtime.CompilerServices
{
    public class MaybeNullAttribute : System.Attribute { }
}
```

There are limitations on where an attribute can be applied which therefore limit the uses of `[MaybeNull]`.

An attribute can be applied to a top-level type only. It is not possible to mark nested type parameter references as nullable.
```C#
static IEnumerable<[MaybeNull]T> Filter(IEnumerable<T> source, Func<T, bool> predicate); // not supported
```

An attribute cannot be applied within executable code or on local function declarations.
As a result, warnings might be more difficult to avoid in generic methods.
```C#
static void M<T>(IEnumerable<T> source)
{
    T t = source.FirstOrDefault(); // warning: value maybe null
    ...
}
```

An attribute cannot be applied within a constraint.
```C#
interface I<T, U> where U : [MaybeNull]T // syntax error
{
}
```

If the attribute is applied to a type parameter that is constrained to a reference type or a value type,
or if the attribute is applied to an explicit type, a warning is reported and the attribute has no effect.

### Conversions
`T` is implicitly convertible to `[MaybeNull]U` with no warning if `T` is implicitly convertible to `U`.

`[MaybeNull]T` is explicitly convertible to `U` with no warning if `T` is explicitly convertible to `U`.

### Implementations and overrides
`[MaybeNull]` is considered when matching interface implementations with interface members and
when matching overrides with base members.

In the following, `T` is a type parameter in an interface member or base class member,
and `U`, `C`, `S` are substitutions for `T` in interface implementations or derived classes,
where `U` is an unconstrained type parameter, where `C` is a `class`, and where `S` is a `struct`.

An `out` parameter `[MaybeNull]T` in the signature of an interface or base class member and implemented or overridden by:
- `[MaybeNull]U` is valid
- `U` is a warning
- `C?` is valid
- `C` is a warning
- `S` is valid
- `S?` is an error.

An `out` parameter `T` in the signature of an interface or base class member and implemented or overridden by:
- `[MaybeNull]U` is valid
- `U` is valid
- `C?` is valid
- `C` is valid
- `S` is valid
- `S?` is an error.

An `in` parameter `[MaybeNull]T` in the signature of an interface or base class member and implemented or overridden by:
- `[MaybeNull]U` is a warning
- `U` is valid
- `C?` is a warning
- `C` is valid
- `S` is valid
- `S?` is an error.

An `in` parameter `T` in the signature of an interface or base class member and implemented or overridden by:
- `[MaybeNull]U` is valid
- `U` is valid
- `C?` is valid
- `C` is valid
- `S` is valid
- `S?` is an error.

```C#
abstract class A<T>
{
    public abstract T F1(T t) { }
    public abstract [MaybeNull]T F2([MaybeNull]T t) { }
}
class B1<T> : A<T>
{
    public override T F1([MaybeNull]T t) { } // ok
    public override T F2([MaybeNull]T t) { } // ok
}
class B2<T> : A<T> where T : class
{
    public override T? F1(T t) { } // warning: return
    public override T F2(T t) { }  // warning: argument
}
```

### Type inference
Type inference with `[MaybeNull]T` may result in unspeakable types. For that reason, the
inferred types for `var` declarations, and for best type inference and method type inference,
do not include `[MaybeNull]`.

```C#
static T Copy<T>(T t) { ... }

static List<T> MakeList<T>(T t) { ... }

static void F<T>([MaybeNull]T x, [MaybeNull]T y)
{
    var z = x;                  // T z;
    var array = new[] { x, y }; // T[] array
    var copy = Copy(x);         // T copy;
    var list = MakeList(x);     // List<T> list
}
```
