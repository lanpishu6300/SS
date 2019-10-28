 ## Summary

This has been a long journey but a fascinating one. We’ve covered a ton of
information in this chapter, so let’s take a moment to reflect on it all.

First, we reasoned that the key difference between streaming and
nonstreaming data processing is the _added dimension of time_. We observed
that relations (the foundational data object from relational algebra, which
itself is the basis for SQL) themselves evolve over time, and from that
derived the notion of a _TVR_ , which captures the evolution of a relation as a
sequence of classic snapshot relations over time. From that definition, we
were able to see that the _closure property_ of relational algebra _remains intact_
in a world of TVRs, which means that the entire suite of relational operators
(and thus SQL constructs) continues to function as one would expect as we
move from a world of point-in-time snapshot relations into a streaming-
compatible world of TVRs.

Second, we explored the biases inherent in both the Beam Model and the
classic SQL model as they exist today, coming to the conclusion that Beam
has a stream-oriented approach, whereas SQL takes a table-oriented
approach.

And finally, we looked at the hypothetical language extensions needed to add
support for robust stream processing to SQL, as well as some carefully
chosen defaults that can greatly decrease the need for those extensions to be
used:

Table/stream selection

```
Given that any time-varying relation can be rendered in two different
ways (table or stream), we need the ability to choose which rendering we
want when materializing the results of a query. We introduced the TABLE,
STREAM, and TVR keywords to provide a nice explicit way to choose the
desired rendering.
Even better is not needing to explicitly specify a choice, and that’s where
good defaults come in. If all the inputs are tables, a good default is for the
output to be a table, as well; this gives you the classic relational query
```
```
18
```

```
behavior everyone is accustomed to. Conversely, if any of the inputs are
streams, a reasonable default is for the output to be a stream, as well.
```
Windowing

```
Though you can declare some types of simple windows declaratively
using existing SQL constructs, there is still value in having explicit
windowing operators:
```
```
Windowing operators encapsulate the window-computation math.
Windowing allows the concise expression of complex, dynamic
groupings like sessions.
```
```
Thus, the addition of simple windowing constructs for use in grouping
can help make queries less error prone while also providing capabilities
(like sessions) that are impractical to express in declarative SQL as it
exists today.
```
Watermarks

```
This isn’t so much a SQL extension as it is a system-level feature. If the
system in question integrates watermarks under the covers, they can be
used in conjunction with triggers to generate streams containing a single,
authoritative version of a row only after the input for that row is believed
to be complete. This is critical for use cases in which it’s impractical to
poll a materialized view table for results, and instead the output of the
pipeline must be consumed directly as a stream. Examples are
notifications and anomaly detection.
```
Triggers

```
Triggers define the shape of a stream as it is created from a TVR. If
unspecified, the default should be per-record triggering, which provides
straightforward and natural semantics matching those of materialized
views. Beyond the default, there are essentially two main types of useful
triggers:
```
```
Watermark triggers , for yielding a single output per window when
```

```
the inputs to that window are believed to be complete.
```
```
Repeated delay triggers , for providing periodic updates.
```
```
Combinations of those two can also be useful, especially in the case of
heuristic watermarks, to provide the early/on-time/late pattern we saw
earlier.
```
Special system columns

```
When consuming a TVR as a stream, there are some interesting metadata
that can be useful and which are most easily exposed as system-level
columns. We looked at four:
Sys.MTime
The processing time at which a given row was last modified in a
TVR.
```
```
Sys.EmitTiming
The timing of the row emit relative to the watermark (early, on-time,
late).
```
```
Sys.EmitIndex
The zero-based index of the emit version for this row.
```
```
Sys.Undo
Whether the row is a normal row or a retraction (undo). By default,
the system should operate with retractions under the covers, as is
necessary any time a series of more than one grouping operation
might exist. If the Sys.Undo column is not projected when rendering
a TVR as a stream, only normal rows will be returned, providing a
simple way to toggle between accumulating and accumulating and
retracting modes.
```
Stream processing with SQL doesn’t need to be difficult. In fact, stream
processing in SQL is quite common already in the form of materialized
views. The important pieces really boil down to capturing the evolution of

```
19
```

datasets/relations over time (via time-varying relations), providing the means
of choosing between physical table or stream representations of those time-
varying relations, and providing the tools for reasoning about time
(windowing, watermarks, and triggers) that we’ve been talking about
throughout this book. And, critically, you need good defaults to minimize
how often these extensions need to be used in practice.

What I mean by “valid relation” here is simply a relation for which the
application of a given operator is well formed. For example, for the SQL
query SELECT x FROM y, a valid relation y would be any relation containing
an attribute/column named x. Any relation not containing a such-named
attribute would be invalid and, in the case of a real database system, would
yield a query execution error.

Much credit to Julian Hyde for this name and succinct rendering of the
concept.

Note that the Sys.Undo name used here is riffing off the concise undo/redo
nomenclature from Apache Flink, which I think is a very clean way to
capture the ideas of retraction and nonretraction rows.

Now, in this example, it’s not too difficult to figure out that the new value
of 8 should replace the old value of 7, given that the mapping is 1:1. But
we’ll see a more complicated example later on when we talk about sessions
that is much more difficult to handle without having retractions as a guide.

And indeed, this is a key point to remember. There are some systems that
advocate treating streams and tables as identical, claiming that we can simply
treat streams like never-ending tables. That statement is accurate inasmuch as
the true underlying primitive is the time-varying relation, and all relational
operations may be applied equally to any time-varying relation, regardless of
whether the actual physical manifestation is a stream or a table. But that sort
of approach conflates the two very different types of views that tables and
streams provide for a given time-varying relation. Pretending that two very
different things are the same might seem simple on the surface, but it’s not a
road toward understanding, clarity, and correctness.

1 2 3 4 5 6


Here referring to tables in the sense of tables that can vary over time; that is,
the table-based TVRs we’ve been looking at.

This one courtesy Julian Hyde.
Though there are a number of efforts in flight across various projects that
are trying to simplify the specification of triggering/ungrouping semantics.
The most compelling proposal, made independently within both the Flink and
Beam communities, is that triggers should simply be specified at the outputs
of a pipeline and automatically propagated up throughout the pipeline. In this
way, one would describe only the desired shape of the streams that actually
create materialized output; the shape of all other streams in the pipeline
would be implicitly derived from there.

Though, of course, a single SQL query has vastly more expressive power
than a single MapReduce, given the far less-confining set of operations and
composition options available.

Note that we’re speaking conceptually here; there are of course a multitude
of optimizations that can be applied in actual execution; for example, looking
up specific rows via an index rather than scanning the entire table.

It’s been brought to my attention multiple times that the “MATERIALIZED”
aspect of these queries is just an optimization: semantically speaking, these
queries could just as easily be replaced with generic CREATE VIEW
statements, in which case the database might instead simply rematerialize the
entire view each time it is referenced. This is true. The reason I use the

MATERIALIZED variant here is that the semantics of a materialized view are to
incrementally update the view table in response to a stream of changes, which
is indicative of the streaming nature behind them. That said, the fact that you
can instead provide a similar experience by re-executing a bounded query
each time a view is accessed provides a nice link between streams and tables
as well as a link between streaming systems and the way batch systems have
been historically used for processing data that evolves over time. You can
either incrementally process changes as they occur or you can reprocess the
entire input dataset from time to time. Both are valid ways of processing an
evolving table of data.

6

7

8

9

10

11

12


Though it’s probably fair to say that SQL’s table bias is likely an artifact of
SQL’s _roots_ in batch processing.

For some use cases, capturing and using the current processing time for a
given record as its event time going forward can be useful (for example,
when logging events directly into a TVR, where the time of ingress is the
natural event time for that record).

Maths are easy to get wrong.
It’s sufficient for retractions to be used by default and not simply always
because the system only needs the _option_ to use retractions. There are
specific use cases; for example, queries with a single grouping operation
whose results are being written into an external storage system that supports
per-key updates, where the system can detect retractions are not needed and
disable them as an optimization.

Note that it’s a little odd for the simple addition of a new column in the
SELECT statement to result in a new rows appearing in a query. A fine
alternative approach would be to require Sys.Undo rows to be filtered out via

a WHERE clause when not needed.

Not that this triviality applies only in cases for which eventual consistency
is sufficient. If you need to always have a globally coherent view of all
sessions at any given time, you must 1) be sure to write/delete (via
tombstones) each session at its emit time, and 2) only ever read from the
HBase table at a timestamp that is less than the output watermark from your
pipeline (to synchronize reads against the multiple, independent
writes/deletes that happen when sessions merge). Or better yet, cut out the
middle person and serve the sessions from your state tables directly.

To be clear, they’re not all hypothetical. Calcite has support for the
windowing constructs described in this chapter.

Note that the definition of “index” becomes complicated in the case of
merging windows like sessions. A reasonable approach is to take the
maximum of all of the previous sessions being merged together and
increment by one.

12

13

14

15

16

17

18

19

