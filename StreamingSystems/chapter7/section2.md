 ## Implicit State

Let’s now begin to talk about the practicalities of persistent state. In most
cases, this essentially boils down to finding the right balance between always
persisting everything (good for consistency, bad for efficiency) and never
persisting anything (bad for consistency, good for efficiency). We’ll begin at
the always-persisting-everything end of the spectrum, and work our way in
the other direction, looking at ways of trading off complexity of
implementation for efficiency without compromising consistency (because
compromising consistency by never persisting anything is the easy way out
for cases in which consistency doesn’t matter, and a nonoption, otherwise).
As before, we use the Apache Beam APIs to concretely ground our
discussions, but the concepts we discuss are applicable across most systems
in existence today.

Also, because there isn’t much you can do to reduce the size of raw inputs,
short of perhaps compressing the data, our discussion centers around the
ways data are persisted within the intermediate state tables created as part of


grouping operations within a pipeline. The inherent nature of grouping
multiple records together into some sort of composite will provide us with
opportunities to eke out gains in efficiency at the cost of implementation
complexity.

## Raw Grouping

The first step in our exploration, at the always-persisting-everything end of
the spectrum, is the most straightforward implementation of grouping within
a pipeline: raw grouping of the inputs. The grouping operation in this case is
typically akin to list appending: any time a new element arrives in the group,
it’s appended to the list of elements seen for that group.

In Beam, this is exactly what you get when you apply a GroupByKey
transform to a PCollection. The stream representing that PCollection in
motion is grouped by key to yield a table at rest containing the records from
the stream, grouped together as lists of values with identical keys. This
shows up in the PTransform signature for GroupByKey, which declares the
input as a PCollection of K/V pairs, and the output as a collection of

K/Iterable<V> pairs:

```
class GroupByKey<K, V> extends PTransform<
PCollection<KV<K, V>>, PCollection<KV<K, Iterable<V>>>>>
```
Every time a trigger fires for a key+window in that table, it will emit a new
pane for that key+window, with the value being the Iterable<V> we see in
the preceding signature.

Let’s look at an example in action in Example 7-1. We’ll take the summation
pipeline from Example 6-5 (the one with fixed windowing and early/on-
time/late triggers) and convert it to use raw grouping instead of incremental
combination (which we discuss a little later in this chapter). We do this by

first applying a GroupByKey transformation to the parsed user/score
key/value pairs. The GroupByKey operation performs raw grouping, yielding
a PCollection with key/value pairs of users and Iterable<Integer>

```
2
```

groups of scores. We then sum up all of the Integers in each iterable by
using a simple MapElements lambda that converts the Iterable<Integer>
into an IntStream<Integer> and calls sum on it.

_Example 7-1. Early, on-time, and late firings via the early/on-time/late API_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> groupedScores = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))
.withLateFirings(AfterCount(1))))
.apply(GroupBy.<String, Integer>create());
PCollection<KV<Team, Integer>> totals = input
.apply(MapElements.via((KV<String, Iterable<Integer>> kv) ->
StreamSupport.intStream(
kv.getValue().spliterator(), false).sum()));

Looking at this pipeline in action, we would see something like that depicted
in Figure 7-1.


_Figure 7-1. Summation via raw grouping of inputs with windowing and early/on-time/late
triggering. The raw inputs are grouped together and stored in the table via the
GroupByKey transformation. After being triggered, the MapElements lambda sums the raw
inputs within a single pane together to yield per-team scores._

Comparing this to Figure 6-10 (which was using incremental combining,
discussed shortly), it’s clear to see this is a lot worse. First, we’re storing a lot
more data: instead of a single integer per window, we now store all the inputs
for that window. Second, if we have multiple trigger firings, we’re
duplicating effort by re-summing inputs we already added together for
previous trigger firings. And finally, if the grouping operation is the point at
which we checkpoint our state to persistent storage, upon machine failure we
again must recompute the sums for any retriggerings of the table. That’s a lot
of duplicated data and computation. Far better would be to incrementally
compute and checkpoint the actual sums, which is an example of _incremental
combining_.

## Incremental Combining

```
00:00 / 00:00
```

The first step in our journey of trading implementation complexity for
efficiency is incremental combining. This concept is manifested in the Beam
API via the CombineFn class. In a nutshell, incremental combining is a form
of automatic state built upon a user-defined associative and commutative
combining operator (if you’re not sure what I mean by these two terms, I
define them more precisely in a moment). Though not strictly necessary for
the discussion that follows, the important parts of the CombineFn API look
like Example 7-2.

_Example 7-2. Abbreviated CombineFn API from Apache Beam_

class CombineFn<InputT, AccumT, OutputT> {
// Returns an accumulator representing the empty value.
AccumT createAccumulator();

// Adds the given input value into the given accumulator
AccumT addInput(AccumT accumulator, InputT input);

// Merges the given accumulators into a new, combined accumulator
AccumT mergeAccumulators(Iterable<AccumT> accumulators);

// Returns the output value for the given accumulator
OutputT extractOutput(AccumT accumulator);
}

A CombineFn accepts inputs of type InputT, which can be combined together

into partial aggregates called _accumulators_ , of type AccumT. These
accumulators themselves can also be combined together into new
accumulators. And finally, an accumulator can be transformed into an output
value of type OutputT. For something like an average, the inputs might be

integers, the accumulators pairs of integers (i.e., Pair<sum of inputs,
count of inputs>), and the output a single floating-point value
representing the mean value of the combined inputs.

But what does all this structure buy us? Conceptually, the basic idea with
incremental combining is that many types of aggregations (sum, mean, etc.)
exhibit the following properties:

```
Incremental aggregations possess an intermediate form that captures
```

the _partial progress_ of combining a set of _N_ inputs _more compactly_
than the full list of those inputs themselves (i.e., the AccumT type in

CombineFn). As discussed earlier, for mean, this is a sum/count pair.
Basic summation is even simpler, with a single number as its
accumulator. A histogram would have a relatively complex
accumulator composed of buckets, where each bucket contains a
count for the number of values seen within some specific range. In
all three cases, however, the amount of space consumed by an
accumulator that represents the aggregation of _N_ elements remains
significantly smaller than the amount of space consumed by the
original _N_ elements themselves, particularly as the size of _N_ grows.

Incremental aggregations are _indifferent to ordering_ across two
dimensions:

```
Individual elements , meaning:
COMBINE(a, b) == COMBINE(b, a)
Groupings of elements , meaning:
COMBINE(COMBINE(a, b), c) == COMBINE(a,
COMBINE(b, c))
```
These properties are known as _commutativity_ and _associativity_ ,
respectively. In concert, they effectively mean that we are free to
combine elements and partial aggregates in any arbitrary order and
with any arbitrary subgrouping. This allows us to optimize the
aggregation in two ways:

Incrementalization

```
Because the order of individual inputs doesn’t matter, we don’t
need to buffer all of the inputs ahead of time and then process
them in some strict order (e.g., in order of event time; note,
however, that this remains independent of shuffling elements by
event time into proper event-time windows before aggregating);
we can simply combine them one-by-one as they arrive. This not
```
```
3
```

```
only greatly reduces the amount of data that must be buffered
(thanks to the first property of our operation, which stated the
intermediate form was a more compact representation of partial
aggregation than the raw inputs themselves), but also spreads the
computation load more evenly over time (versus aggregating a
burst of inputs all at once after the full input set has been
buffered).
```
Parallelization

```
Because the order in which partial subgroups of inputs are
combined doesn’t matter, we’re free to arbitrarily distribute the
computation of those subgroups. More specifically, we’re free to
spread the computation of those subgroups across a plurality of
machines. This optimization is at the heart of MapReduce’s
Combiners (the genesis of Beam’s CombineFn).
MapReduce’s Combiner optimization is essential to solving the
hot-key problem, where some sort of grouping computation is
performed on an input stream that is too large to be reasonably
processed by a single physical machine. A canonical example is
breaking down high-volume analytics data (e.g., web traffic to a
popular website) across a relatively low number of dimensions
(e.g., by web browser family: Chrome, Firefox, Safari, etc.). For
websites with a particularly high volume of traffic, it’s often
intractable to calculate stats for any single web browser family
on a single machine, even if that’s the only thing that machine is
dedicated to doing; there’s simply too much traffic to keep up
with. But with an associative and commutative operation like
summation, it’s possible to spread the initial aggregation across
multiple machines, each of which computes a partial aggregate.
The set of partial aggregates generated by those machines
(whose size is now many of orders magnitude smaller than the
original inputs) might then be further combined together on a
single machine to yield the final aggregate result.
```

```
As an aside, this ability to parallelize also yields one additional
benefit: the aggregation operation is naturally compatible with
merging windows. When two windows merge, their values must
somehow be merged, as well. With raw grouping, this means
merging the two full lists of buffered values together, which has
a cost of O(N). But with a CombineFn, it’s a simple combination
of two partial aggregates, typically an O(1) operation.
```
For the sake of completeness, consider again Example 6-5, shown in
Example 7-3, which implements a summation pipeline using incremental
combination.

_Example 7-3. Grouping and summation via incremental combination, as in
Example 6-5_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))
.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

When executed, we get what we saw Figure 6-10 (shown here in Figure 7-2).
Compared to Figure 7-1, this is clearly a big improvement, with much greater
efficiency in terms of amount of data stored and amount of computation
performed.


```
Figure 7-2. Grouping and summation via incremental combination. In this version,
incremental sums are computed and stored in the table rather than lists of inputs, which
must later be summed together independently.
```
By providing a more compact intermediate representation for a grouping
operation, and by relaxing requirements on ordering (both at the element and
subgroup levels), Beam’s CombineFn trades off a certain amount of
implementation complexity in exchange for increases in efficiency. In doing
so, it provides a clean solution for the hot-key problem and also plays nicely
with the concept of merging windows.

One shortcoming, however, is that your grouping operation must fit within a
relatively restricted structure. This is all well and good for sums, means, and
so on, but there are plenty of real-world use cases in which a more general
approach, one which allows precise control over trade-offs of complexity and
efficiency, is needed. We’ll look next at what such a general approach entails.


