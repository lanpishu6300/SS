 ## Looking Backward: Stream and Table Biases

In many ways, the act of adding robust streaming support to SQL is largely
an exercise in attempting to merge the _where_ , _when_ , and _how_ semantics of the
Beam Model with the _what_ semantics of the classic SQL model. But to do so
cleanly, and in a way that remains true to the look and feel of classic SQL,
requires an understanding of how the two models relate to each other. Thus,
much as we explored the relationship of the Beam Model to stream and table
theory in Chapter 6, we’ll now explore the relationship of the Beam Model to
the classic SQL model, using stream and table theory as the underlying
framework for our comparison. In doing so, we’ll uncover the inherent biases
present in each model, which will provide us some insights in how to best
marry the two in a clean, natural way.

## The Beam Model: A Stream-Biased Approach

Let’s begin with the Beam Model, building upon the discussion in Chapter 6.
To begin, I want to discuss the inherent stream bias in the Beam Model as it
exists today relative to streams and tables.

If you think back to Figures 6-11 and 6-12, they showed two different views
of the same score-summation pipeline that we’ve used as an example
throughout the book: in Figure 6-11 a logical, Beam-Model view, and in
Figure 6-12 a physical, streams and tables–oriented view. Comparing the two
helped highlight the relationship of the Beam Model to streams and tables.
But by overlaying one on top of the other, as I’ve done in Figure 8-1, we can
see an additional interesting aspect of the relationship: the Beam Model’s
inherent stream bias.


_Figure 8-1. Stream bias in the Beam Model approach_


In this figure, I’ve drawn dashed red lines connecting the transforms in the
logical view to their corresponding components in the physical view. The
thing that stands out when observed this way is that all of the logical
transformations are connected by _streams_ , even the operations that involve
grouping (which we know from Chapter 6 results in a table being created
_somewhere_ ). In Beam parlance, these transformations are PTransforms, and

they are always applied to PCollections to yield new PCollections. The
important takeaway here is that PCollections in Beam are _always_ streams.
As a result, the Beam Model is an inherently stream-biased approach to data
processing: streams are the common currency in a Beam pipeline (even batch
pipelines), and tables are always treated specially, either abstracted behind
sources and sinks at the edges of the pipeline or hidden away beneath a
grouping and triggering operation somewhere in the pipeline.

Because Beam operates in terms of streams, anywhere a table is involved
(sources, sinks, and any intermediate groupings/ungroupings), some sort of
conversion is necessary to keep the underlying table hidden. Those
conversions in Beam look something like this:

```
Sources that consume tables typically hardcode the manner in which
those tables are triggered ; there is no way for a user to specify
custom triggering of the table they want to consume. The source
may be written to trigger every new update to the table as a record, it
might batch groups of updates together, or it might provide a single,
bounded snapshot of the data in the table at some point in time. It
really just depends on what’s practical for a given source, and what
use case the author of the source is trying to address.
```
```
Sinks that write tables typically hardcode the manner in which they
group their input streams. Sometimes, this is done in a way that
gives the user a certain amount of control; for example, by simply
grouping on a user-assigned key. In other cases, the grouping might
be implicitly defined; for example, by grouping on a random
physical partition number when writing input data with no natural
key to a sharded output source. As with sources, it really just
```

depends on what’s practical for the given sink and what use case the
author of the sink is trying to address.

For _grouping/ungrouping operations_ , in contrast to sources and
sinks, Beam provides users complete flexibility in how they group
data into tables and ungroup them back into streams. This is by
design. Flexibility in grouping operations is necessary because the
way data are grouped is a key ingredient of the algorithms that
define a pipeline. And flexibility in ungrouping is important so that
the application can shape the generated streams in ways that are
appropriate for the use case at hand.

However, there’s a wrinkle here. Remember from Figure 8-1 that the
Beam Model is inherently biased toward streams. As result, although
it’s possible to cleanly apply a grouping operation directly to a

stream (this is Beam’s GroupByKey operation), the model never
provides first-class table objects to which a trigger can be directly
applied. As a result, triggers must be applied somewhere else. There
are basically two options here:

Predeclaration of triggers

```
This is where triggers are specified at a point in the pipeline
before the table to which they are actually applied. In this case,
you’re essentially prespecifying behavior you’d like to see later
on in the pipeline after a grouping operation is encountered.
When declared this way, triggers are forward-propagating.
```
Post-declaration of triggers

```
This is where triggers are specified at a point in the pipeline
following the table to which they are applied. In this case, you’re
specifying the behavior you’d like to see at the point where the
trigger is declared. When declared this way, triggers are
backward-propagating.
```
Because post-declaration of triggers allows you to specify the
behavior you want at the actual place you want to observe it, it’s

```
8
```

```
much more intuitive. Unfortunately, Beam as it exists today (2.x and
earlier) uses predeclaration of triggers (similar to how windowing is
also predeclared).
```
Even though Beam provides a number of ways to cope with the fact that
tables are hidden, we’re still left with the fact that tables must always be
triggered before they can be observed, even if the contents of that table are
really the final data that you want to consume. This is a shortcoming of the
Beam Model as it exists today, one which could be addressed by moving
away from a stream-centric model and toward one that treats both streams
and tables as first-class entities.

Let’s now look at the Beam Model’s conceptual converse: classic SQL.

## The SQL Model: A Table-Biased Approach

In contrast to the Beam Model’s stream-biased approach, SQL has
historically taken a table-biased approach: queries are applied to tables, and
always result in new tables. This is similar to the batch processing model we
looked at in Chapter 6 with MapReduce, but it will be useful to consider a
concrete example like the one we just looked at for the Beam Model.

Consider the following denormalized SQL table:

```
UserScores (user, team, score, timestamp)
```
It contains user scores, each annotated with the IDs of the corresponding user
and their corresponding team. There is no primary key, so you can assume
that this is an append-only table, with each row being identified implicitly by
its unique physical offset. If we want to compute team scores from this table,
we could use a query that looks something like this:

```
SELECT team, SUM(score) as total
FROM UserScores
GROUP BY team;
```
When executed by a query engine, the optimizer will probably break this

```
9
```

query down into roughly three steps:

```
1. Scanning the input table (i.e., triggering a snapshot of it)
```
```
2. Projecting the fields in that table down to team and score
3. Grouping rows by team and summing the scores
```
If we look at this using a diagram similar to Figure 8-1, it would look like
Figure 8-2.

The SCAN operation takes the input table and triggers it into a bounded stream
that contains a snapshot of the contents of that table at query execution time.

That stream is consumed by the SELECT operation, which projects the four-
column input rows down to two-column output rows. Being a nongrouping
operation, it yields another stream. Finally, that two-column stream of teams
and user scores enters the GROUP BY and is grouped by team into a table, with

scores for the same team SUM’d together, yielding our output table of teams
and their corresponding team score totals.


```
Figure 8-2. Table bias in a simple SQL query
```
This is a relatively simple example that naturally ends in a table, so it really
isn’t sufficient to highlight the table-bias in classic SQL. But we can tease out
some more evidence by simply splitting the main pieces of this query
(projection and grouping) into two separate queries:

```
SELECT team, score
INTO TeamAndScore
FROM UserScores;
```

```
SELECT team, SUM(score) as total
INTO TeamTotals
FROM TeamAndScore
GROUP BY team;
```
In these queries, we first project the UserScores table down to just the two
columns we care about, storing the results in a temporary TeamAndScore
table. We then group that table by team, summing up the scores as we do so.
After breaking things out into a pipeline of two queries, our diagram looks
like that shown in Figure 8-3.


_Figure 8-3. Breaking the query into two to reveal more evidence of table bias_


If classic SQL exposed streams as first-class objects, you would expect the
result from the first query, TeamAndScore, to be a stream because the SELECT
operation consumes a stream and produces a stream. But because SQL’s
common currency is tables, it must first convert the projected stream into a
table. And because the user hasn’t specified any explicit key for grouping, it
must simply group keys by their identity (i.e., append semantics, typically
implemented by grouping by the physical storage offset for each row).

Because TeamAndScore is now a table, the second query must then prepend
an additional SCAN operation to scan the table back into a stream to allow the
GROUP BY to then group it back into a table again, this time with rows
grouped by team and with their individual scores summed together. Thus, we
see the two implicit conversions (from a stream and back again) that are
inserted due to the explicit materialization of the intermediate table.

That said, tables in SQL are not _always_ explicit; implicit tables can exist, as
well. For example, if we were to add a HAVING clause to the end of the query
with the GROUP BY statement, to filter out teams with scores less than a
certain threshold, the diagram would change to look something like Figure 8-
4.


_Figure 8-4. Table bias with a final HAVING clause_


With the addition of the HAVING clause, what used to be the user-visible
TeamTotals table is now an implicit, intermediate table. To filter the results
of the table according to the rules in the HAVING clause, that table must be
triggered into a stream that can be filtered and then that stream must be
implicitly grouped back into a table to yield the new output table,
LargeTeamTotals.

The important takeaway here is the clear table bias in classic SQL. Streams
are always implicit, and thus for any materialized stream a conversion
from/to a table is required. The rules for such conversions can be categorized
roughly as follows:

Input tables (i.e., sources, in Beam Model terms)

```
These are always implicitly triggered in their entirety at a specific point in
time (generally query execution time) to yield a bounded stream
containing a snapshot of the table at that time. This is identical to what
you get with classic batch processing, as well; for example, the
MapReduce case we looked at in Chapter 6.
```
Output tables (i.e., sinks, in Beam Model terms)

```
These tables are either direct manifestations of a table created by a final
grouping operation in the query, or are the result of an implicit grouping
(by some unique identifier for the row) applied to a query’s terminal
stream, for queries that do not end in a grouping operation (e.g., the
projection query in the previous examples, or a GROUP BY followed by a
HAVING clause). As with inputs, this matches the behavior seen in classic
batch processing.
```
Grouping/ungrouping operations

```
Unlike Beam, these operations provide complete flexibility in one
dimension only: grouping. Whereas classic SQL queries provide a full
suite of grouping operations (GROUP BY, JOIN, CUBE, etc.), they provide
only a single type of implicit ungrouping operation: trigger an
intermediate table in its entirety after all of the upstream data contributing
```
```
10
```

```
to it have been incorporated (again, the exact same implicit trigger
provided in MapReduce as part of the shuffle operation). As a result, SQL
offers great flexibility in shaping algorithms via grouping but essentially
zero flexibility in shaping the implicit streams that exist under the covers
during query execution.
```
**Materialized views**

Given how analogous classic SQL queries are to classic batch processing, it
might be tempting to write off SQL’s inherent table bias as nothing more than
an artifact of SQL not supporting stream processing in any way. But to do so
would be to ignore the fact that databases have supported a specific type of
stream processing for quite some time: _materialized views_. A materialized
view is a view that is physically materialized as a table and kept up to date
_over_ time by the database as the source table(s) evolve. Note how this sounds
remarkably similar to our definition of a time-varying relation. What’s
fascinating about materialized views is that they add a very useful form of
stream processing to SQL _without_ significantly altering the way it operates,
including its inherent table bias.

For example, let’s consider the queries we looked at in Figure 8-4. We can
alter those queries to instead be CREATE MATERIALIZED VIEW statements:

```
CREATE MATERIALIZED VIEW TeamAndScoreView AS
SELECT team, score
FROM UserScores;
```
```
CREATE MATERIALIZED VIEW LargeTeamTotalsView AS
SELECT team, SUM(score) as total
FROM TeamAndScoreView
GROUP BY team
HAVING SUM(score) > 100;
```
In doing so, we transform them into continuous, standing queries that process
the updates to the UserScores table continuously, in a streaming manner.
Even so, the resulting physical execution diagram for the views _looks almost
exactly the same_ as it did for the one-off queries; nowhere are streams made

```
11
```

into explicit first-class objects in order to support this idea of streaming
materialized views. The _only_ noteworthy change in the physical execution
plan is the substitution of a different trigger: SCAN-AND-STREAM instead of
SCAN, as illustrated in Figure 8-5.


_Figure 8-5. Table bias in materialized views_


What is this SCAN-AND-STREAM trigger? SCAN-AND-STREAM starts out like a
SCAN trigger, emitting the full contents of the table at a point in time into a
stream. But instead of stopping there and declaring the stream to be done
(i.e., bounded), it continues to also trigger all subsequent modifications to the
input table, yielding an unbounded stream that captures the evolution of the
table over time. In the general case, these modifications include not only
INSERTs of new values, but also DELETEs of previous values and UPDATEs to
existing values (which, practically speaking, are treated as a simultaneous
DELETE/INSERT pair, or undo/redo values as they are called in Flink).

Furthermore, if we consider the table/stream conversion rules for materialized
views, the only real difference is the trigger used:

```
Input tables are implicitly triggered via a SCAN-AND-STREAM trigger
instead of a SCAN trigger. Everything else is the same as classic batch
queries.
Output tables are treated the same as classic batch queries.
```
```
Grouping/ungrouping operations function the same as classic batch
queries, with the only difference being the use of a SCAN-AND-
STREAM trigger instead of a SNAPSHOT trigger for implicit ungrouping
operations.
```
Given this example, it’s clear to see that SQL’s inherent table bias is not just
an artifact of SQL being limited to batch processing: materialized views
lend SQL the ability to perform a specific type of stream processing without
any significant changes in approach, including the inherent bias toward
tables. Classic SQL is just a table-biased model, regardless of whether you’re
using it for batch or stream processing.
