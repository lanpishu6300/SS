 ## Batch Processing Versus Streams and Tables

With our proverbial knuckles now cracked, let’s start to tie up some loose
ends. To begin, we tackle the first one, regarding batch processing. At the
end, we’ll discover that the resolution to the second issue, regarding the

```
2
```

relationship of streams to bounded and unbounded data, will fall out naturally
from the answer for the first. Score one for serendipity.

## A Streams and Tables Analysis of MapReduce

To keep our analysis relatively simple, but solidly concrete, as it were, let’s
look at how a traditional MapReduce job fits into the streams/tables world.
As alluded to by its name, a MapReduce job superficially consists of two
phases: Map and Reduce. For our purposes, though, it’s useful to look a little
deeper and treat it more like six:

MapRead

```
This consumes the input data and preprocesses them a bit into a standard
key/value form for mapping.
```
Map

```
This repeatedly (and/or in parallel) consumes a single key/value pair
from the preprocessed input and outputs zero or more key/value pairs.
```
MapWrite

```
This clusters together sets of Map-phase output values having identical
keys and writes those key/value-list groups to (temporary) persistent
storage. In this way, the MapWrite phase is essentially a group-by-key-
and-checkpoint operation.
```
ReduceRead

```
This consumes the saved shuffle data and converts them into a standard
key/value-list form for reduction.
```
Reduce

```
This repeatedly (and/or in parallel) consumes a single key and its
associated value-list of records and outputs zero or more records, all of
which may optionally remain associated with that same key.
```
ReduceWrite

```
3
```

```
This writes the outputs from the Reduce phase to the output datastore.
```
Note that the MapWrite and ReduceRead phases sometimes are referred to in
aggregate as the Shuffle phase, but for our purposes, it’s better to consider
them independently. It’s perhaps also worth noting that the functions served
by the MapRead and ReduceWrite phases are more commonly referred to
these days as sources and sinks. Digressions aside, however, let’s now see
how this all relates to streams and tables.

**Map as streams/tables**

Because we start and end with static datasets, it should be clear that we
begin with a table and end with a table. But what do we have in between?
Naively, one might assume that it’s tables all the way down; after all, batch
processing is (conceptually) known to consume and produce tables. And if
you think of a batch processing job as a rough analog of executing a classic
SQL query, that feels relatively natural. But let’s look a little more closely at
what’s really happening, step by step.

First up, MapRead consumes a table and produces _something_. That
something is consumed next by the Map phase, so if we want to understand
its nature, a good place to start would be with the Map phase API, which
looks something like this in Java:

```
void map(KI key, VI value, Emit<KO, VO> emitter);
```
The map call will be repeatedly invoked for each key/value pair in the input
table. If you think this sounds suspiciously like the input table is being
consumed as a stream of records, you’d be right. We look more closely at
how the table is being converted into a stream later, but for now, suffice it to
say that the MapRead phase is iterating over the data at rest in the input table
and putting them into motion in the form of a stream that is then consumed
by the Map phase.

Next up, the Map phase consumes that stream, and then does what? Because
the map operation is an element-wise transformation, it’s not doing anything
that will halt the moving elements and put them to rest. It might change the

```
4
```

effective cardinality of the stream by either filtering some elements out or
exploding some elements into multiple elements, but those elements all
remain independent from one another after the Map phase concludes. So, it
seems safe to say that the Map phase both consumes a stream as well as
produces a stream.

After the Map phase is done, we enter the MapWrite phase. As I noted
earlier, the MapWrite groups records by key and then writes them in that
format to persistent storage. The _persistent_ part of the write actually isn’t
strictly necessary at this point as long as there’s persistence _somewhere_ (i.e.,
if the upstream inputs are saved and one can recompute the intermediate
results from them in cases of failure, similar to the approach Spark takes with
Resilient Distributed Datasets [RDDs]). What _is_ important is that the records
are grouped together into some kind of datastore, be it in memory, on disk, or
what have you. This is important because, as a result of this grouping
operation, records that were previously flying past one-by-one in the stream
are now brought to rest in a location dictated by their key, thus allowing per-
key groups to accumulate as their like-keyed brethren and sistren arrive. Note
how similar this is to the definition of stream-to-table conversion provided
earlier: _the aggregation of a stream of updates over time yields a table_. The
MapWrite phase, by virtue of grouping the stream of records by their keys,
has put those data to rest and thus converted the stream back into a table.
Cool!

We’re now halfway through the MapReduce, so, using Figure 6-1, let’s recap
what we’ve seen so far.

We’ve gone from table to stream and back again across three operations.
MapRead converted the table into a stream, which was then transformed into
a new stream by Map (via the user’s code), which was then converted back
into a table by MapWrite. We’re going to find that the next three operations
in the MapReduce look very similar, so I’ll go through them more quickly,
but I still want to point out one important detail along the way.

```
5
```

_Figure 6-1. Map phases in a MapReduce. Data in a table are converted to a stream and
back again._


**Reduce as streams/tables**

Picking up where we left off after the MapWrite phase, ReduceRead itself is
relatively uninteresting. It’s basically identical to MapRead, except that the
values being read are singleton lists of values instead of singleton values,
because the data stored by MapWrite were key/value-list pairs. But it’s still
just iterating over a snapshot of a table to convert it into a stream. Nothing
new here.

And even though it _sounds_ like it might be interesting, Reduce in this context
is really just a glorified Map phase that happens to receive a list of values for
each key instead of a single value. So it’s still just mapping single
(composite) records into zero or more new records. Nothing particularly new
here, either.

ReduceWrite is the one that’s a bit noteworthy. We know already that this
phase must convert a stream to a table, given that Reduce produces a stream
and the final output is a table. But how does that happen? If I told you it was
a direct result of key-grouping the outputs from the previous phase into
persistent storage, just like we saw with MapWrite, you might believe me,
until you remembered that I noted earlier that key-association was an
_optional_ feature of the Reduce phase. With that feature enabled, ReduceWrite
_is_ essentially identical to MapWrite. But if that feature is disabled and the
outputs from Reduce have no associated keys, what exactly is happening to
bring those data to rest?

To understand what’s going on, it’s useful to think again of the semantics of a
SQL table. Though often recommended, it’s not strictly required for a SQL
table to have a primary key uniquely identifying each row. In the case of
keyless tables, each row that is inserted is considered to be a new,
independent row (even if the data therein are identical to one or more extant
rows in the table), much as though there were an implicit
AUTO_INCREMENT field being used as the key (which incidentally, is
what’s effectively happening under the covers in most implementations, even
though the “key” in this case might just be some physical block location that
is never exposed or expected to be used as a logical identifier). This implicit
unique key assignment is precisely what’s happening in ReduceWrite with

```
6
```

unkeyed data. Conceptually, there’s still a group-by-key operation
happening; that’s what brings the data to rest. But lacking a user-supplied
key, the ReduceWrite is treating each record as though it has a new, never-
before-seen key, and effectively grouping each record with itself, resulting
again in data at rest.

Take a look at Figure 6-2, which shows the entire pipeline from the
perspective of stream/tables. You can see that it’s a sequence of TABLE →
STREAM → STREAM → TABLE → STREAM → STREAM → TABLE.
Even though we’re processing bounded data and even though we’re doing
what we traditionally think of as batch processing, it’s really just streams and
tables under the covers.

```
7
```

_Figure 6-2. Map and Reduce phases in a MapReduce, viewed from the perspective of
streams and tables_


## Reconciling with Batch Processing

So where does this leave us with respect to our first two questions?

```
1. Q: How does batch processing fit into stream/table theory?
A: Quite nicely. The basic pattern is as follows:
```
```
a. Tables are read in their entirety to become streams.
```
```
b. Streams are processed into new streams until a grouping
operation is hit.
```
```
c. Grouping turns the stream into a table.
d. Steps a through c repeat until you run out of stages in the
pipeline.
```
```
2. Q: How do streams relate to bounded/unbounded data?
A: As we can see from the MapReduce example, streams are simply
the in-motion form of data, regardless of whether they’re bounded or
unbounded.
```
Taken from this perspective, it’s easy to see that stream/table theory isn’t
remotely at odds with batch processing of bounded data. In fact, it only
further supports the idea I’ve been harping on that batch and streaming really
aren’t that different: at the end of the of day, it’s streams and tables all the
way down.

With that, we’re well on our way toward a general theory of streams and
tables. But to wrap things up cleanly, we last need to revisit the four
_what_ / _where_ / _when_ / _how_ questions within the streams/tables context, to see how
they all relate.

