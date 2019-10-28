  ## A General Theory of Stream and Table Relativity

Having surveyed how stream processing, batch processing, the four
_what_ / _where_ / _when_ / _how_ questions, and the Beam Model as a whole relate to
stream and table theory, let’s now attempt to articulate a more general
definition of stream and table relativity.

_A general theory of stream and table relativity_ :

```
Data processing pipelines (both batch and streaming) consist of
tables , streams , and operations upon those tables and streams.
Tables are data at rest , and act as a container for data to accumulate
and be observed over time.
```
```
Streams are data in motion , and encode a discretized view of the
evolution of a table over time.
```
```
Operations act upon a stream or table and yield a new stream or
table. They are categorized as follows:
```
```
stream → stream: Nongrouping (element-wise) operations
Applying nongrouping operations to a stream alters the data
in the stream while leaving them in motion, yielding a new
stream with possibly different cardinality.
```
```
stream → table: Grouping operations
Grouping data within a stream brings those data to rest,
yielding a table that evolves over time.
```
```
Windowing incorporates the dimension of event
time into such groupings.
Merging windows dynamically combine over time,
allowing them to reshape themselves in response to
```

```
the data observed and dictating that key remain the
unit of atomicity/parallelization, with window
being a child component of grouping within that
key.
```
```
table → stream: Ungrouping (triggering) operations
Triggering data within a table ungroups them into motion,
yielding a stream that captures a view of the table’s
evolution over time.
```
```
Watermarks provide a notion of input
completeness relative to event time, which is a
useful reference point when triggering event-
timestamped data, particularly data grouped into
event-time windows from unbounded streams.
```
```
The accumulation mode for the trigger determines
the nature of the stream, dictating whether it
contains deltas or values, and whether retractions
for previous deltas/values are provided.
```
```
table → table: (none)
There are no operations that consume a table and yield a
table, because it’s not possible for data to go from rest and
back to rest without being put into motion. As a result, all
modifications to a table are via conversion to a stream and
back again.
```
What I love about these rules is that they just make sense. They have a very
natural and intuitive feeling about them, and as a result they make it so much
easier to understand how data flow (or don’t) through a sequence of
operations. They codify the fact that data exist in one of two constitutions at
any given time (streams or tables), and they provide simple rules for
reasoning about the transitions between those states. They demystify
windowing by showing how it’s just a slight modification of a thing everyone


already innately understands: grouping. They highlight why grouping
operations in general are always such a sticking point for streaming (because
they bring data in streams to rest as tables) but also make it very clear what
sorts of operations are needed to get things unstuck (triggers; i.e., ungrouping
operations). And they underscore just how unified batch and stream
processing really are, at a conceptual level.

When I set out to write this chapter, I wasn’t entirely sure what I was going to
end up with, but the end result was much more satisfying than I’d imagined it
might be. In the chapters to come, we use this theory of stream and table
relativity again and again to help guide our analyses. And every time, its
application will bring clarity and insight that would otherwise have been
much harder to gain. Streams and tables are the best.


