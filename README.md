# Linq.IndexRange

![.NET Core](https://github.com/Dixin/System.Linq.IndexRange/workflows/.NET%20Core/badge.svg)

LINQ operators to enable C# 8.0 index and range new features working with LINQ queries and any type that implements `IEnumerable<T>`.

```bat
dotnet add package Linq.IndexRange
```

```cs
var element = source1.ElementAt(index: `5);
var elements = source2.ElementsIn(range: 10..^10); // or Slice(10..^10)
var query1 = EnumerableExtensions.Range(10..20).Select().Where();
var query2 = (10..20).AsEnumerable().Select().Where();
```

Proposal: https://github.com/dotnet/runtime/issues/28776

C# 8.0 introduces index and range features for array and [countable types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/ranges#adding-index-and-range-support-to-existing-library-types). It is natural and convenient to **generally support index and range for all IEnumerable<T> types and LINQ**. I searched the [API review notes](https://github.com/dotnet/apireviews), didn't find such APIs, so propose them here.

```cs
var element = source1.ElementAt(index: ^5);
var elements = source2.Slice(range: 10..^10); // or ElementIn(range: 10..^10)
var query1 = Enumerable.Range(10..20).Select().Where();
var query2 = (10..20).AsEnumerable().Select().Where();
```

# Problem

Index/range are **language level** features. Currently, they

- work with array (compiled to `RuntimeHelpers.GetSubArray`, etc.)
- work with [countable type](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/ranges#adding-index-and-range-support-to-existing-library-types)
- do not work with "uncountable" `IEnumerable<T>` types, or LINQ query.

# Rationale and usage

The goals of these LINQ APIs are:

- Use index to locate an element in sequence.
- Use a range to slice a sequence. The usage should be consistent with array, but with deferred execution.
- Use a range to start fluent LINQ query.

This enables index and range **language features** to work with any type that implements `IEnumerable<T>`, and LINQ queries.

LINQ already has `ElementAt(int)` and `ElementAtOrDefault(int)` operators. It would be natural to have a `System.Index` overload: `ElementAt(Index)` and `ElementAtOrDefault(Index)`, and a new method `ElementsIn(Range)` (or `Slice(Range)`), so that LINQ can seamlessly work with C# 8.0:

```cs
Index index = ...;
var element1 = source1.ElementAt(index);
var element2 = source2.ElementAtOrDefault(^5);
Range range = ...;
var slice1 = source3.ElementsIn(range); // or Slice(range)
var slice2 = source4.ElementsIn(2..^2); // or Slice(2..^2)
var slice2 = source5.ElementsIn(^10..); // or Slice(^10..)
```

The following `Range` overload and `AsEnumerable` overload work the same, they convert `System.Range` to a sequence, so that LINQ query can be started fluently from there:

```cs
var query1 = Enumerable.Range(10..).Select(...);
Range range = ...;
var query2 = range.AsEnumerable().Select(...);
var query3 = (10..20).AsEnumerable().Where(...);
```

With these APIs, the C# `countable[Index]` and `countable[Range]` syntax are enabled for sequences & LINQ queries as `enumerable.ElementAt(Index)` and `enumerable.Slice(Range)`.

# Proposed APIs

For LINQ to Objects:

```cs
namespace System.Linq
{
    public static partial class Enumerable
    {
        public static TSource ElementAt<TSource>(this IEnumerable<TSource> source, Index index);

        public static TSource ElementAtOrDefault<TSource>(this IEnumerable<TSource> source, Index index);

        public static IEnumerable<TSource> ElementsIn<TSource>(this IEnumerable<TSource> source, Range range);

        public static IEnumerable<TSource> Slice<TSource>(this IEnumerable<TSource> source, Range range);

        public static IEnumerable<TSource> Range<TSource>(Range range);

        public static IEnumerable<TSource> AsEnumerable<TSource>(this Range source);
    }
}
```

For remote LINQ:

```cs
namespace System.Linq
{
    public static partial class Queryable
    {
        public static TSource ElementAt<TSource>(this IQueryable<TSource> source, Index index);

        public static TSource ElementAtOrDefault<TSource>(this IQueryable<TSource> source, Index index);

        public static IQueryable<TSource> ElementsIn<TSource>(this IQueryable<TSource> source, Range range);

        public static IQueryable<TSource> Slice<TSource>(this IQueryable<TSource> source, Range range);
    }
}
```

# Implementation details (and pull request)

The [API review process](https://github.com/dotnet/corefx/blob/master/Documentation/project-docs/api-review-process.md) says PR should not be submitted before the API proposal is approved. So currently I implemented these APIs separately [Dixin/Linq.IndexRange](https://github.com/Dixin/Linq.IndexRange):

- `Enumerable`:
  - [`ElementAt(Index)` and `ElementAtOrDefault(Index)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/ElementAt.cs)
  - [`ElementsIn(Range)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/ElementsIn.cs)
  - [`Slice(Range)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/Slice.cs)
  - [`Range(Range)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/Range.cs)
  - [`AsEnumerable(Range)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/AsEnumerable.cs).
- `Queryable`:
  - [`ElementAt(Index)`, `ElementAtOrDefault(Index)`, `ElementsIn(Range)`, `Slice(Range)`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange/QueryableExtensions.cs).

Please see the [unit tests](https://github.com/Dixin/System.Linq.IndexRange/tree/master/System.Linq.IndexRange.Tests) about how they work.

These proposed APIs can be used by adding NuGet package `Linq.IndexRange`:

```cmd
dotnet add package Linq.IndexRange
```

If this proposal is doable, I can submit a PR quickly.

# Open questions

## `ElementsIn(Range)` vs `Slice(Range)` as name

Should `ElementsIn(Range)` be called `Slice`? Currently I implemented both.

- `ElementsIn(Range)` keeps the naming consistency with original `ElementAt(index)`. Might it be natural for existing LINQ users?
- `Slice` is consistent with the [countable types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/ranges#adding-index-and-range-support-to-existing-library-types), which requires a `Slice` method to support range. I prefer `Slice` for this reason.

## Out of range handling

If Index is out of the boundaries, for `ElementAt(Index)`, there are 2 options:

- **Array behavior**: throw `IndexOutOfRangeException`
- **LINQ behavior**: throw `ArgumentOutOfRangeException`. My current implementation goes this way, to keep `ElementAt(Index)` consistent with orginal `ElementAt(int)`.

If Range goes off the boundaries of source sequence, for `Slice(Range)` or `ElementsIn(Range)`, there are 2 options:

- **Array behavior**: Follow the behavior of `array[Range]`, throw `ArgumentOutOfRangeException`. I implemented `ElementsIn(Range)` following this way. See [unit tests of `ElementsIn`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange.Tests/ElementsInTests.cs).
- **LINQ behavior**: Follow the behavior of current partitioning LINQ operators like `Skip`/`Take`/`SkipLast`/`TakeLast`, do not throw any exception. I implemented `Slice` following this way. See [unit test of `Slice`](https://github.com/Dixin/System.Linq.IndexRange/blob/master/System.Linq.IndexRange.Tests/SliceTests.cs).

## `ElementAt(Index)` and `Queryable`

As @bartdesmet mentioned in the comments, LINQ providers may have issues when they see `ElementAt` having an `Index` argument, etc. Should we have a new name for the operator instead of overload? For example, `At(Index)` or `Index(Index)`?

## `Range`

For `Range(Range)` and `AsEnumerable(Range)`, the question is: what does range's start index and end index mean, when the index is from the end? For example, `10..20` can be easily converted to a sequence of 10, 11,12, ... 19, but how about `^20...^10`?

There are 2 options:

- The easiest way is to disallow, and throw exception.
- My current implementation attempts to make it flexible. Regarding `Index`'s `Value` can be from `0` to `int.MaxValue`, I assume a virtual "full range" `0..2147483648`, and any `Range` instance is a slice of that "full range". So:
  - Ranges `..` and `0..` and `..^0` and `0..^0` are converted to "full sequence" 0, 1, .. 2147483647
  - Range `100..^47` is converted to sequence 100, 101, .. 2147483600
  - Range `^48..^40` is converted to sequence 2147483600, 2147483601 .. 2147483607
  - Range `10..10` is converted to empty sequence, etc.
  
  See [unit tests of `Range(Range)`](https://github.com/Dixin/Linq.IndexRange/blob/master/Linq.IndexRange.Tests/RangeTests.cs).

## `AsEnumerable`

Should this be provided to bridge range to LINQ? For me, `(10..20).AsEnumerable().Select().Where()` is intuitive and natural.
