# Linq.IndexRange

LINQ operators to enable C# 8.0 index and range new features working with LINQ queries and any type that implements `IEnumerable<T>`.

```bat
dotnet add package Linq.IndexRange
```

```cs
var element = source1.ElementAt(index: `5);
var elements = source2.ElementsIn(range: 10..^10); // or Slice(10..^10)
var query1 = Enumerable.Range(10..20).Select().Where();
var query2 = (10..20).AsEnumerable().Select().Where();
```

If adding NuGet package by command fails, please manually add it to your project file:

```xml
<ItemGroup>
  <PackageReference Include="Linq.IndexRange" Version="1.0.0-preview2" />
</ItemGroup>
```

Proposed these APIs to .NET Core [dotnet/corefx/#35552](https://github.com/dotnet/corefx/issues/35552).

## Problem

Index/range are **language level** features. However, currently (v3.0.0-preview2/SDK 3.0.100-preview-010184), they

- only work with array, not other types like `List<T>`
- are compiled to array copy, apparently no deferred execution.

## Rationale and usage

The goals of these LINQ APIs are:

- Use index to locate an element in sequence. 
- Use range to slice a sequence. The usage should be consistent with array, but with deferred execution.
- Use range to start fluent LINQ query.

This enables index and range **langaguge features** to work with any type that implements `IEnumerable<T>`.

LINQ already has `ElementAt(int index)` and `ElementAtOrDefault(int index)` query operator. It would be natural to have a overload for `System.Index`: `ElementAt(Index index)` and `ElementAtOrDefault(Index index)`, and a new method `ElementsIn(Range range)` (or `Slice(Range)`), so that LINQ can seamlessly work with C# 8.0:

```cs
Index index = ...;
var element1 = source1.ElementAt(index);
var element2 = source2.ElementAtOrDefault(^5);
Range range = ...;
var slice1 = source3.ElementsIn(range); // or Slice(range)
var slice2 = source4.ElementsIn(2..^2) // or Slice(2..^2)
var slice2 = source5.ElementsIn(^10..); // or Slice(^10..)
```

The following `Range` overload and `AsEnumerable` overload for `System.Range` convert it to a sequence, so that LINQ query can be started fluently from c# range:

```cs
var query1 = Enumerable.Range(10..).Select(...);
Range range = ...;
var query1 = range.AsEnumerable().Select(...);
var query2 = (10..20).AsEnumerable().Where(...);
```

## Proposed APIs

For LINQ to Objects:

```cs
namespace System.Linq
{
    public static partial class Queryable
    {
        public static TSource ElementAt<TSource>(this IEnumerable<TSource> source, Index index) { throw null; }

        public static TSource ElementAtOrDefault<TSource>(this IEnumerable<TSource> source, Index index) { throw null; }

        public static IEnumerable<TSource> ElementsIn<TSource>(this IEnumerable<TSource> source, Range range) { throw null; }

        public static IEnumerable<TSource> Range<TSource>(Range range) { throw null; }

        public static IEnumerable<TSource> AsEnumerable<TSource>(this Range source) { throw null; }
}
```

For remote LINQ:

```cs
namespace System.Linq
{
    public static partial class Queryable
    {
        public static TSource ElementAt<TSource>(this IQueryable<TSource> source, Index index) { throw null; }

        public static TSource ElementAtOrDefault<TSource>(this IQueryable<TSource> source, Index index) { throw null; }

        public static IQueryable<TSource> ElementsIn<TSource>(this IQueryable<TSource> source, Range range) { throw null; }
}
```

## Implementation details (and pull request)

The [API review process](https://github.com/dotnet/corefx/blob/master/Documentation/project-docs/api-review-process.md) says PR should not be submitted before the API proposal is approved. So currently I implemented these APIs separately https://github.com/Dixin/Linq.IndexRange :

- `Enumerable`: [`ElementAt(Index)`, `ElementAtOrDefault(Index)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/ElementAt.cs)
- `Enumerable`: [`ElementsIn(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/ElementsIn.cs) and [`Slice(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/Slice.cs)
- `Enumerable`: [`Range(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/Range.cs) and [`AsEnumerable(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/AsEnumerable.cs). They are the same conversion.
- `Queryable`: [`ElementAt(Index)`, `ElementAtOrDefault(Index)`, `ElementsIn(Range)`, `Slice(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange/QueryableExtensions.cs).

So that anyone can start to use these APIs by adding a NuGet package:

```bat
dotnet add package Linq.IndexRange
```

If this proposal is doable, I can submit a PR quickly.

The implementation of `ElementAt(Index)`, `ElementAtOrDefault(Index)` and `ElementsIn(Range)` for `IQueryable<T>` is straightforward. They just create an expression tree. `ElementAt(Index)` and `ElementAtOrDefault(Index)` for `IEnumerable<T>` are straightforward as well.

## Open questions

### `ElementsIn(Range)`

Should it be called `Slice`? Currently I implement it as `ElementsIn(Range range)` to maintain the consistency with original `ElementAt(int index)`, which could be natural for existing LINQ users.

What should we do when the range's start index and/or end index go off the boundaries of source sequence? There are 2 options:

- Follow the behavior of range with array, and throw `OverflowException`/`ArgumentEception`/`ArgumentOutOfRangeException` accordingly.
- Follow the behavior of current partitioning LINQ operators like `Skip`/`Take`/`SkipLast`/`TakeLast`, do not throw exception.

I implemented `ElementsIn(Range)` following the array behavior. See [unit tests of `ElementsIn`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange.Tests/ElementsInTests.cs). And I implemented `Slice` following the LINQ behavior. See [unit test of `Slice`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange.Tests/SliceTests.cs).

### `ElementAt(Index)` and `Queryable`

As @bartdesmet mentioned in the comments, LINQ providers may have issues when they see `ElementAt` having an `Index` argument, etc. Should we have a new name for the operator instead of overload? For example, `At(Index)` and `Slice(Range)`?

### `Range`

For `Range(Range)` and `AsEnumerable(Range)`, the question is: what does range's start index and end index mean, when the index is from the end? For example, `10..20` can be easily converted to a sequence of 10, 11,12, ... 19, but how about `^20...^10`? 

The easiest way is to disallow, and throw exception.

My current implementation attempts to make it flexible. Regarding `Index`'s `Value` can be from `0` to `int.MaxValue`, I assume a virtual "full range" `0..2147483648`, and any `Range` instance is a slice of that "full range". So:

- Ranges `..` and `0..` and `..^0` and `0..^0` are converted to "full sequence" 0, 1, .. 2147483647
- Range `100..^47` is converted to sequence 100, 101, .. 2147483600
- Range `^48..^40` is converted to sequence 2147483600, 2147483601 .. 2147483607
- Range `10..10` is converted to empty sequence

etc. See [unit tests of `Range(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange.Tests/RangeTests.cs).

### `AsEnumerable`

Should this be provided to bridge range to LINQ? For me, at least `(10..20).AsEnumerable().Select().Where()` is intuitive and natural.
