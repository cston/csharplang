# Open issues for collection literals

Open issues for collection literals to review at LDM (see [proposal](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md)).

## Support spread operator

Support a _spread_ operator to inline an `IEnumerable` within a collection literal. The proposed syntax is `..e`

* Possible translation of `List<int> list = [x, ..e, y];`

    ```c#
    // List<int> list = [x, ..e, y];
    List<int> __result = new List<int>();
    __result.Add(x);
    foreach (var __i in e)
        __result.Add(__i);
    __result.Add(y);
    ```

* Avoid extra allocations when the enumerable type has a `Length` or `Count` property, or when `TryGetNonEnumeratedCount<T>(this IEnumerable<T>, out int)` returns true.

    ```c#
    List<int> __result = e.TryGetNonEnumeratedCount(out int n)
        ? new List<int>(capacity: 2 + n)
        : new List<int>();
    // add items
    ```

* Avoid intermediate collections when not observable.

    For instance, avoid generating collections for the conditional expression `b ? [1, 2, 3] : []` below.
    ```csharp
    bool b = ...;
    var v = [x, y, .. b ? [1, 2, 3] : [ ]];
    ```

* Syntax ambiguity between spreads and ranges in a collection literal. (Since the ambiguity is within a collection literal, this is not a compatibility issue.)

    ```c#
    Range[] ranges = [range1, ..e, range2]; // ..e: spread or range?
    ```

    * Require parentheses `(..e)` or include a start index `0..e` for a range, or
    * Choose a different syntax (like `...`) for spread. (Lack of consistency with slice patterns.)

## Support dictionaries and dictionary elements
Support collection literals that represent dictionaries, with a simple syntax for key-value pairs. The proposed syntax for key-value pairs is `k:v`

* Possible translation of `var d = [k1:v1, k2:v2];`
    ```c#
    // var d = [k1:v1, k2:v2];
    var __result = new Dictionary<TKey, TValue>();
    __result[k1] = v1;
    __result[k2] = v2;
    ```

* Interfaces `I<TKey, TValue>` implemented by `Dictionary<TKey, TValue>` can be used as _target types_ for collection literals.
    ```csharp
    IReadOnlyDictionary<string, int> d = [x:y, ..e];
    ```

* The _natural type_ of a collection literal is `Dictionary<TKey, TValue>` when the _best common type_ of the elements is `KeyValuePair<TKey, TValue>`.

* Construction of dictionary collection literals uses the indexer `this[TKey] { get; set; }` rather than `Add(TKey, TValue)`, to provide consistent set semantics rather than add.
    ```c#
    // IReadOnlyDictionary<string, int> d = [x:y, ..e];
    var __result = new Dictionary<string, int>();
    __result[x] = y;
    foreach (var (__k, __v) in e)
        __result[__k] = __v;
    IReadOnlyDictionary<string, int> d = __result;
    ```

* Syntax ambiguity in a collection literal between a conditional expression and k:v with conditional access.  (Since the ambiguity is within a collection literal, this is not a compatibility issue.)

    ```c#
    var v = [a ? [b] : c]; // [a ? ([b]) : c)] or [(a ? [b]) : c]?
    ```

    Could bind to conditional access, based on precedence, and require parentheses for `a ? ([b]) : c`.

## Support `Construct` methods

Allow collection types with custom `Construct` methods (see [proposal](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md#construct-methods)).

* This is primarily to allow efficient construction of immutable collections, and requires BCL changes to expose `Construct` methods on existing immutable collection types.

* A type `T` can be constructed from a collection literal through the use of a `void Construct(CollectionType)` method when:

  * the `Construct` method is found on an instance of `T` (including extension methods?), and
  * `CollectionType` is some other type known to be a _constructible type_.

* A type with suitable `Construct` method can be used as a _target type_ for collection literals.
   ```csharp
   ImmutableArray<T> result = [x, ..e, y]; // ImmutableArray<T>.Construct(T[] values)
   ```

* May require an `init Construct()` method to ensure collection instance is not mutated after construction.

* To implement: `ImmutableArray<T> result = [x, ..e, y];`

    Implementing explicitly, using a builder:
    ```c#
    var __builder = ImmutableArray.CreateBuilder<int>(initialCapacity: 2 + n);

    __builder.Add(x);
    __builder.AddRange(e);
    __builder.Add(y);

    // Create final result. __builder is now garbage.
    ImmutableArray<int> __result = __builder.MoveToImmutable();
    ```

    Possible translation from collection literal using `init void ImmutableArray<T>.Construct(T[] values)` method:
    ```c#
    T[] __storage = new T[2 + n];
    int __index = 0;

    __storage[__index++] = x;
    foreach (var __t in e)
        __storage[__index++] = __t;
    __storage[__index++] = y;
    
    // Create final result. __storage owned by __result.
    var __result = new ImmutableArray<T>();
    __result.Construct(__storage); // init void ImmutableArray<T>.Construct(T[] values)
    ```

* _Could we use constructors or factory methods instead?_

    For instance, with support for `params ReadOnlySpan<T>`:
    ```c#
    public struct ImmutableArray<T>
    {
        public ImmutableArray(params ReadOnlySpan<T> values) { ... }
    }
