 ## Summary

In this chapter, we first established the basics of stream and table theory. We
first defined streams and tables relatively:

streams → tables

```
The aggregation of a stream of updates over time yields a table.
```
tables → streams

```
The observation of changes to a table over time yields a stream.
```
We next defined them independently:

```
Tables are data at rest.
Streams are data in motion.
```
We then assessed the classic MapReduce model of batch computation from a
streams and tables perspective and came to the conclusion that the following
four steps describe batch processing from that perspective:

```
1. Tables are read in their entirety to become streams.
```

```
2. Streams are processed into new streams until a grouping operation is
hit.
3. Grouping turns the stream into a table.
```
```
4. Steps 1 through 3 repeat until you run out of operations in the
pipeline.
```
From this analysis, we were able to see that streams are just as much a part of
batch processing as they are stream processing, and also that the idea of data
being a stream is an orthogonal one from whether the data in question are
bounded or unbounded.

Next, we spent a good deal of time considering the relationship between
streams and tables and the robust, out-of-order stream processing semantics
afforded by the Beam Model, ultimately arriving at the general theory of
stream and table relativity we enumerated in the previous section. In addition
to the basic definitions of streams and tables, the key insight in that theory is
that there are four (really, just three) types of operations in a data processing
pipeline:

stream → stream

```
Nongrouping (element-wise) operations
```
stream → table

```
Grouping operations
```
table → stream

```
Ungrouping (triggering) operations
```
table → table

```
(nonexistent)
```
By classifying operations in this way, it becomes trivial to understand how
data flow through (and linger within) a given pipeline over time.

Finally, and perhaps most important of all, we learned this: when you look at
things from the streams-and-tables point of view, it becomes abundantly clear


how batch and streaming really are just the same thing conceptually.
Bounded or unbounded, it doesn’t matter. It’s streams and tables from top to
bottom.

</bad-physics-jokes>

If you didn’t go to college for computer science and you’ve made it this far
in the book, you are likely either 1) my parents, 2) masochistic, or 3) very
smart (and for the record, I’m not implying these groups are necessarily
mutually exclusive; figure that one out if you can, Mom and Dad! <winky-
smiley/>).

And note that in some cases, the tables themselves can accept time as a
query parameter, allowing you to peer backward in time to snapshots of the
table as it existed in the past.

Note that no guarantees are made about the keys of two successive records
observed by a single mapper, because no key-grouping has occurred yet. The
existence of the key here is really just to allow keyed datasets to be consumed
in a natural way, and if there are no obvious keys for the input data, they’ll all
just share what is effectively a global null key.

Calling the inputs to a batch job “static” might be a bit strong. In reality, the
dataset being consumed can be constantly changing as it’s processed; that is,
if you’re reading directly from an HBase/Bigtable table within a timestamp
range in which the data aren’t guaranteed to be immutable. But in most cases,
the recommended approach is to ensure that you’re somehow processing a
static snapshot of the input data, and any deviation from that assumption is at
your own peril.

Note that grouping a stream by key is importantly distinct from simply
_partitioning_ that stream by key, which ensures that all records with the same
key end up being processed by the same machine but doesn’t do anything to
put the records to rest. They instead remain in motion and thus continue on as
a stream. A grouping operation is more like a partition-by-key followed by a
write to the appropriate group for that partition, which is what puts them to

1

2

3

4

5


rest and turns the stream into a table.

One giant difference, from an implementation perspective at least, being
that ReduceWrite, knowing that keys have already been grouped together by
MapWrite, and further knowing that Reduce is unable to alter keys for the
case in which its outputs remain keyed, can simply accumulate the outputs
generated by reducing the values for a single key in order to group them
together, which is much simpler than the full-blown shuffle implementation
required for a MapWrite phase.

Another way of looking at it is that there are two types of tables: updateable
and appendable; this is the way the Flink folks have framed it for their Table
API. But even though that’s a great intuitive way of capturing the observed
semantics of the two situations, I think it obscures the underlying nature of
what’s actually happening that causes a stream to come to rest as a table; that
is, grouping.

Though as we can clearly see from this example, it’s not just a streaming
thing; you can get the same effect with a batch system if its state tables are
world readable.

This is particularly painful if a sink for your storage system of choice
doesn’t exist yet; building proper sinks that can uphold consistency
guarantees is a surprisingly subtle and difficult task.

This also means that if you place a value into multiple windows—for
example, sliding windows—the value must conceptually be duplicated into
multiple, independent records, one per window. Even so, it’s possible in
some cases for the underlying system to be smart about how it treats certain
types of overlapping windows, thus optimize away the need for actually
duplicating the value. Spark, for example, does this for sliding windows.

Note that this high-level conceptual view of how things work in batch
pipelines belies the complexity of efficiently triggering an entire table of data
at once, particularly when that table is sizeable enough to require a plurality
of machines to process. The SplittableDoFn API recently added to Beam
provides some insight into the mechanics involved.

6

7

8

9

10

11

12


And yes, if you blend batch and streaming together you get Beam, which is
where that name came from originally. For reals.

This is why you should always use an Oxford comma.
Note that in the case of merging windows, in addition to merging the
current values for the two windows to yield a merged current value, the
previous values for those two windows would need to be merged, as well, to
allow for the later calculation of a merged delta come triggering time.

12

13

14



