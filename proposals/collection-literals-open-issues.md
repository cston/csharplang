# Open issues for collection literals

Open issues for collection literals to review at LDM (see [proposal](https://github.com/dotnet/csharplang/blob/main/proposals/collection-literals.md)).

## Support dictionaries and dictionary elements

Allow collection literals to represent dictionaries, and provide simple syntax for key-value pairs.

* The proposed syntax is `k:v`

  The proposed syntax introduces an ambiguity between conditional expression and conditional access within a collection literal:

    ```c#
    var v = a ? [b] : c; // [(a ? [b] : c)] or [(a ? [b]) : c]?
    ```

    To resolve this, we can require users to parenthesize `(a ? [b] : c)` for a conditional expression with collection literal consequence within a collection literal.

* Interfaces `I<TKey, TValue>` implemented by `Dictionary<TKey, TValue>` can be used as _target types_ for collection literals.
    ```csharp
    IReadOnlyDictionary<string, int> d = ["one":1, "two":2];
    ```

* Construction of dictionary collection literals uses the indexer `this[TKey] { get; set; }` rather than `Add(TKey, TValue)`, to provide set semantics rather than add.

* The _natural type_ of a dictionary collection literal is determined from the natural types of the keys and values independently.

## Support spread elements

Allow splatting `IEnumerable` or `IEnumerable<T>` within a collection literal.

* The proposed syntax is `..e`

  The proposed syntax introduces an ambiguity between spreads and ranges within a collection literal:

    ```c#
    Range[] ranges = [range1, ..e, range2]; // ..e: spread or range?
    ```
    To resolve this, we can:

    * Require users to parenthesize `(..e)` or include a start index `0..e` if they want a range.
    * Choose a different syntax (like `...`) for spread.  This would be unfortunate for the lack of consistency with slice patterns.

* A collection literal containing a spread element may not have a known length.

  Construction of the resulting collection may require copying or reallocation, particularly when targeting arrays or spans.

  We can optimize if the expression type has a `Length` or `Count` property, or using `TryGetNonEnumeratedLength()` at runtime. 

## Support `Construct` methods

Allow construction of custom collection types through `Construct()` methods.

* This is primarily to allow efficient construction of immutable collections. This requires BCL changes to expose `Construct()` methods on existing collections.

* A type `T` can be constructed from a collection literal through the use of a `void Construct(CollectionType)` method when:

  * the `Construct` method is found on an instance of `T` (including extension methods?), and
  * `CollectionType` is some other type known to be a [_constructible type_](constructible-collection-types).

* A type with suitable `Construct()` method can be used as a _target type_ for collection literals.
   ```csharp
   ImmutableArray<int> result = [a, b, c]; // ImmutableArray<T>.Construct(T[] values)
   ```

* Through the use of the `init` modifier, existing APIs can directly support collection literals in a manner that allows for no-overhead production of the data the final collection will store.

  * Like [`init` accessors](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-9.0/init.md#init-only-setters), an `init` method would be an instance method invocable at the point of object creation but become unavailable once object creation has completed. This facility thus prevents general use of such a marked method outside of known safe compiler scopes where the instance value being constructed cannot be observed until complete.

  * In the context of collection literals, using the `init` modifier on the [`Construct` method](#construct-methods) would allow types to trust that the collection instances passed into them cannot be mutated outside of them, and that they are being passed ownership of the collection instance.  This would negate any need to copy data that would normally be assumed to be in an untrusted location.

  * For example, if an `init void Construct(T[] values)` instance method were added to [`ImmutableArray<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable.immutablearray-1), then it would be possible for the compiler to emit the following:

    ```c#
    T[] __storage = [a, b, c];
    ImmutableArray<T> __result = new ImmutableArray<T>();
    __result.Construct(__storage);
    ```

    `ImmutableArray<T>` would then take that array directly and use it as its own backing storage.  This would be safe because the compiler (following the requirements around `init`) would ensure that no other location in the code would have access to this temporary array, and thus it would not be possible to mutate it behind the back of the `ImmutableArray<T>` instance.

    The `Construct` method can then safely update this to the final non-default state without that intermediate state being visible.
