 ## What Is Streaming SQL?

I would argue that the answer to this question has eluded our industry for
decades. In all fairness, the database community has understood maybe 99%
of the answer for quite a while now. But I have yet to see a truly cogent and
comprehensive definition of streaming SQL that encompasses the full breadth
of robust streaming semantics. That’s what we’ll try to come up with here,
although it would be hubris to assume we’re 100% of the way there now.
Maybe 99.1%? Baby steps.

Regardless, I want to point out up front that most of what we’ll discuss in this
chapter is still purely hypothetical as of the time of writing. This chapter and
the one that follows (covering streaming joins) both describe an idealistic
vision for what streaming SQL could be. Some pieces are already
implemented in systems like Apache Calcite, Apache Flink, and Apache
Beam. Many others aren’t implemented anywhere. Along the way, I’ll try to
call out a few of the things that do exist in concrete form, but given what a
moving target that is, your best bet is to simply consult the documentation for
your specific system of interest.

On that note, it’s also worth highlighting that the vision for streaming SQL
presented here is the result of a collaborative discussion between the Calcite,
Flink, and Beam communities. Julian Hyde, the lead developer on Calcite,


has long pitched his vision for what streaming SQL might look like. In 2016,
members of the Flink community integrated Calcite SQL support into Flink
itself, and began adding streaming-specific features such as windowing
constructs to the Calcite SQL dialect. Then, in 2017, all three communities
began a discussion to try to come to agreement on what language extensions
and semantics for robust stream processing in Calcite SQL should look like.
This chapter attempts to distill the ideas from that discussion down into a
clear and cohesive narrative about integrating streaming concepts into SQL,
regardless of whether it’s Calcite or some other dialect.

## Relational Algebra

When talking about what streaming means for SQL, it’s important to keep in
mind the theoretical foundation of SQL: relational algebra. Relational algebra
is simply a mathematical way of describing relationships between data that
consist of named, typed tuples. At the heart of relational algebra is the
relation itself, which is a set of these tuples. In classic database terms, a
relation is something akin to a table, be it a physical database table, the result
of a SQL query, a view (materialized or otherwise), and so on; it’s a set of
rows containing named and typed columns of data.

One of the more critical aspects of relational algebra is its closure property:
applying any operator from the relational algebra to any valid relation
always yields another relation. In other words, relations are the common
currency of relational algebra, and all operators consume them as input and
produce them as output.

Historically, many attempts to support streaming in SQL have fallen short of
satisfying the closure property. They treat streams separately from classic
relations, providing new operators to convert between the two, and restricting
the operations that can be applied to one or the other. This significantly raises
the bar of adoption for any such streaming SQL system: would-be users must
learn the new operators and understand the places where they’re applicable,
where they aren’t, and similarly relearn the rules of applicability in this new
world for any old operators. What’s worse, most of these systems still fall

```
1
```

short of providing the full suite of streaming semantics that we would want,
such as support for robust out-of-order processing and strong temporal join
support (the latter of which we cover in Chapter 9). As a result, I would argue
that it’s basically impossible to name any existing streaming SQL
implementation that has achieved truly broad adoption. The additional
cognitive overhead and restricted capabilities of such streaming SQL systems
have ensured that they remain a niche enterprise.

To change that, to truly bring streaming SQL to the forefront, what we need
is a way for streaming to become a first-class citizen within the relational
algebra itself, such that the entire standard relational algebra can apply
naturally in both streaming and nonstreaming use cases. That isn’t to say that
streams and tables should be treated as exactly the same thing; they most
definitely are not the same, and recognizing that fact lends clarity to
understanding and power to navigating the stream/table relationship, as we’ll
see shortly. But the core algebra should apply cleanly and naturally to both
worlds, with minimal extensions beyond the standard relational algebra only
in the cases where absolutely necessary.

## Time-Varying Relations

To cut to the chase, the punchline I referred to at the beginning of the chapter
is this: the key to naturally integrating streaming into SQL is to extend
relations, the core data objects of relational algebra, to represent a set of data
_over time_ rather than a set of data at a _specific point_ in time. More succinctly,
instead of _point-in-time_ relations, we need _time-varying relations_.

But what are time-varying relations? Let’s first define them in terms of
classic relational algebra, after which we’ll also consider their relationship to
stream and table theory.

In terms of relational algebra, a time-varying relation is really just the
evolution of a classic relation over time. To understand what I mean by that,
imagine a raw dataset consisting of user events. Over time, as users generate
new events, the dataset continues to grow and evolve. If you observe that set
at a specific point in time, that’s a classic relation. But if you observe the

```
2
```

holistic evolution of the set _over time_ , that’s a time-varying relation.

Put differently, if classic relations are like two-dimensional tables consisting
of named, typed columns in the x-axis and rows of records in the y-axis,
time-varying relations are like three-dimensional tables with x- and y-axes as
before, but an additional z-axis capturing different versions of the two-
dimensional table over time. As the relation changes, new snapshots of the
relation are added in the z dimension.

Let’s look at an example. Imagine our raw dataset is users and scores; for
example, per-user scores from a mobile game as in most of the other
examples throughout the book. And suppose that our example dataset here
ultimately ends up looking like this when observed at a specific point in time,
in this case 12:07:

```
-------------------------
| Name | Score | Time |
-------------------------
| Julie | 7 | 12:01 |
| Frank | 3 | 12:03 |
| Julie | 1 | 12:03 |
| Julie | 4 | 12:07 |
-------------------------
```
In other words, it recorded the arrivals of four scores over time: Julie’s score
of 7 at 12:01, both Frank’s score of 3 and Julie’s second score of 1 at 12:03,
and, finally, Julie’s third score of 4 at 12:07 (note that the Time column here
contains processing-time timestamps representing the _arrival time_ of the
records within the system; we get into event-time timestamps a little later on).
Assuming these were the only data to ever arrive for this relation, it would
look like the preceding table any time we observed it after 12:07. But if
instead we had observed the relation at 12:01, it would have looked like the
following, because only Julie’s first score would have arrived by that point:

```
-------------------------
| Name | Score | Time |
```

```
-------------------------
| Julie | 7 | 12:01 |
-------------------------
```
If we had then observed it again at 12:03, Frank’s score and Julie’s second
score would have also arrived, so the relation would have evolved to look
like this:

```
-------------------------
| Name | Score | Time |
-------------------------
| Julie | 7 | 12:01 |
| Frank | 3 | 12:03 |
| Julie | 1 | 12:03 |
-------------------------
```
From this example we can begin to get a sense for what the _time-varying_
relation for this dataset must look like: it would capture the entire evolution
of the relation over time. Thus, if we observed the time-varying relation (or
TVR) at or after 12:07, it would thus look like the following (note the use of

a hypothetical TVR keyword to signal that we want the query to return the full
time-varying relation, not the standard point-in-time snapshot of a classic
relation):

```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Score | Time | | | Name | Score | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| | | | | | | | | | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
```

```
| | Name | Score | Time | | | Name | Score | Time | |
| ------------------------- | ------------------------- |
| | Julie | 7 | 12:01 | | | Julie | 7 | 12:01 | |
| | Frank | 3 | 12:03 | | | Frank | 3 | 12:03 | |
| | Julie | 1 | 12:03 | | | Julie | 1 | 12:03 | |
| | | | | | | Julie | 4 | 12:07 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
Because the printed/digital page remains constrained to two dimensions, I’ve
taken the liberty of flattening the third dimension into a grid of two-
dimensional relations. But you can see how the time-varying relation
essentially consists of a sequence of classic relations (ordered left to right, top
to bottom), each capturing the full state of the relation for a specific range of
time (all of which, by definition, are contiguous).

What’s important about defining time-varying relations this way is that they
really are, for all intents and purposes, just a sequence of classic relations that
each exist independently within their own disjointed (but adjacent) time
ranges, with each range capturing a period of time during which the relation
did not change. This is important, because it means that the application of a
relational operator to a time-varying relation is equivalent to individually
applying that operator to each classic relation in the corresponding sequence.
And taken one step further, the result of individually applying a relational
operator to a sequence of relations, each associated with a time interval, will
always yield a corresponding sequence of relations with the same time
intervals. In other words, the result is a corresponding time-varying relation.
This definition gives us two very important properties:

```
The full set of operators from classic relational algebra remain valid
when applied to time-varying relations, and furthermore continue to
behave exactly as you’d expect.
```
```
The closure property of relational algebra remains intact when
applied to time-varying relations.
```
Or more succinctly, _all the rules of classic relational algebra continue to
hold when applied to time-varying relations_. This is huge, because it means


that our substitution of time-varying relations for classic relations hasn’t
altered the parameters of the game in any way. Everything continues to work
the way it did back in classic relational land, just on sequences of classic
relations instead of singletons. Going back to our examples, consider two
more time-varying relations over our raw dataset, both observed at some time
after 12:07. First a simple filtering relation using a WHERE clause:

```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Score | Time | | | Name | Score | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
| | Name | Score | Time | | | Name | Score | Time | |
| ------------------------- | ------------------------- |
| | Julie | 7 | 12:01 | | | Julie | 7 | 12:01 | |
| | Julie | 1 | 12:03 | | | Julie | 1 | 12:03 | |
| | | | | | | Julie | 4 | 12:07 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
As you would expect, this relation looks a lot like the preceding one, but with
Frank’s scores filtered out. Even though the time-varying relation captures
the added dimension of time necessary to record the evolution of this dataset
over time, the query behaves exactly as you would expect, given your
understanding of SQL.

For something a little more complex, let’s consider a grouping relation in
which we’re summing up all the per-user scores to generate a total overall
score for each user:


```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | Julie | 8 | 12:03 | | | Julie | 12 | 12:07 | |
| | Frank | 3 | 12:03 | | | Frank | 3 | 12:03 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
Again, the time-varying version of this query behaves exactly as you would
expect, with each classic relation in the sequence simply containing the sum
of the scores for each user. And indeed, no matter how complicated a query
we might choose, the results are always identical to applying that query
independently to the commensurate classic relations composing the input
time-varying relation. I cannot stress enough how important this is!

All right, that’s all well and good, but time-varying relations themselves are
more of a theoretical construct than a practical, physical manifestation of
data; it’s pretty easy to see how they could grow to be quite huge and
unwieldy for large datasets that change frequently. To see how they actually
tie into real-world stream processing, let’s now explore the relationship
between time-varying relations and stream and table theory.

## Streams and Tables

For this comparison, let’s consider again our grouped time-varying relation
that we looked at earlier:


```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | Julie | 8 | 12:03 | | | Julie | 12 | 12:07 | |
| | Frank | 3 | 12:03 | | | Frank | 3 | 12:03 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
We understand that this sequence captures the full history of the relation over
time. Given our understanding of tables and streams from Chapter 6, it’s not
too difficult to understand how time-varying relations relate to stream and
table theory.

Tables are quite straightforward: because a time-varying relation is
essentially a sequence of classic relations (each capturing a snapshot of the
relation at a specific point in time), and classic relations are analogous to
tables, observing a time-varying relation as a table simply yields the point-in-
time relation snapshot for the time of observation.

For example, if we were to observe the previous grouped time-varying
relation as a table at 12:01, we’d get the following (note the use of another
hypothetical keyword, TABLE, to explicitly call out our desire for the query to
return a table):


```
-------------------------
| Name | Total | Time |
-------------------------
| Julie | 7 | 12:01 |
-------------------------
```
And observing at 12:07 would yield the expected:

```
-------------------------
| Name | Total | Time |
-------------------------
| Julie | 12 | 12:07 |
| Frank | 3 | 12:03 |
-------------------------
```
What’s particularly interesting here is that there’s actually support for the
idea of time-varying relations within SQL, even as it exists today. The SQL
2011 standard provides “temporal tables,” which store a versioned history of
the table over time (in essence, time-varying relations) as well an AS OF
SYSTEM TIME construct that allows you to explicitly query and receive a
snapshot of the temporal table/time-varying relation at whatever point in time
you specified. For example, even if we performed our query at 12:07, we
could still see what the relation looked like back at 12:03:

```
-------------------------
| Name | Total | Time |
-------------------------
| Julie | 8 | 12:03 |
| Frank | 3 | 12:03 |
-------------------------
```
So there’s some amount of precedent for time-varying relations in SQL


already. But I digress. The main point here is that tables capture a snapshot of
the time-varying relation at a specific point in time. Most real-world table
implementations simply track real time as we observe it; others maintain
some additional historical information, which in the limit is equivalent to a
full-fidelity time-varying relation capturing the entire history of a relation
over time.

Streams are slightly different beasts. We learned in Chapter 6 that they too
capture the evolution of a table over time. But they do so somewhat
differently than the time-varying relations we’ve looked at so far. Instead of
holistically capturing snapshots of the entire relation each time it changes,
they capture the _sequence of changes_ that result in those snapshots within a
time-varying relation. The subtle difference here becomes more evident with
an example.

As a refresher, recall again our baseline example TVR query:

```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | Julie | 8 | 12:03 | | | Julie | 12 | 12:07 | |
| | Frank | 3 | 12:03 | | | Frank | 3 | 12:03 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
Let’s now observe our time-varying relation as a stream as it exists at a few


distinct points in time. At each step of the way, we’ll compare the original
table rendering of the TVR at that point in time with the evolution of the
stream up to that point. To see what stream renderings of our time-varying
relation look like, we’ll need to introduce two new hypothetical keywords:

```
A STREAM keyword, similar to the TABLE keyword I’ve already
introduced, that indicates we want our query to return an event-by-
event stream capturing the evolution of the time-varying relation
over time. You can think of this as applying a per-record trigger to
the relation over time.
```
```
A special Sys.Undo column that can be referenced from a STREAM
query, for the sake of identifying rows that are retractions. More on
this in a moment.
```
Thus, starting out from 12:01, we’d have the following:

```
------------------------- -----------------------------
---
| Name | Total | Time | | Name | Total | Time |
Undo |
------------------------- -----------------------------
---
| Julie | 7 | 12:01 | | Julie | 7 | 12:01 |
|
------------------------- ........ [12:01, 12:01]
........
```
The table and stream renderings look almost identical at this point. Mod the

```
3
```

Undo column (discussed in more detail in the next example), there’s only one
difference: whereas the table version is complete as of 12:01 (signified by the
final line of dashes closing off the bottom end of the relation), the stream
version remains incomplete, as signified by the final ellipsis-like line of
periods marking both the open tail of the relation (where additional data
might be forthcoming in the future) as well as the processing-time range of
data observed so far. And indeed, if executed on a real implementation, the
STREAM query would wait indefinitely for additional data to arrive. Thus, if

we waited until 12:03, three new rows would show up for the STREAM query.
Compare that to a fresh TABLE rendering of the TVR at 12:03:

```
------------------------- -----------------------------
---
| Name | Total | Time | | Name | Total | Time |
Undo |
------------------------- -----------------------------
---
| Julie | 8 | 12:03 | | Julie | 7 | 12:01 |
|
| Frank | 3 | 12:03 | | Frank | 3 | 12:03 |
|
------------------------- | Julie | 7 | 12:03 |
undo |
| Julie | 8 | 12:03 |
|
........ [12:01, 12:03]
........
```
Here’s an interesting point worth addressing: why are there _three_ new rows in


the stream (Frank’s 3 and Julie’s undo-7 and 8) when our original dataset
contained only _two_ rows (Frank’s 3 and Julie’s 1) for that time period? The
answer lies in the fact that here we are observing the stream of changes to an
_aggregation_ of the original inputs; in particular, for the time period from
12:01 to 12:03, the stream needs to capture two important pieces of
information regarding the change in Julie’s aggregate score due to the arrival
of the new 1 value:

```
The previously reported total of 7 was incorrect.
The new total is 8.
```
That’s what the special Sys.Undo column allows us to do: distinguish
between normal rows and rows that are a _retraction_ of a previously reported
value.

A particularly nice feature of STREAM queries is that you can begin to see how
all of this relates to the world of classic Online Transaction Processing
(OLTP) tables: the STREAM rendering of this query is essentially capturing a
sequence of INSERT and DELETE operations that you could use to materialize
this relation over time in an OLTP world (and really, when you think about it,
OLTP tables themselves are essentially time-varying relations mutated over
time via a stream of INSERTs, UPDATEs, and DELETEs).

Now, if we don’t care about the retractions in the stream, it’s also perfectly
fine not to ask for them. In that case, our STREAM query would look like this:

```
-------------------------
| Name | Total | Time |
-------------------------
| Julie | 7 | 12:01 |
| Frank | 3 | 12:03 |
| Julie | 8 | 12:03 |
.... [12:01, 12:03] .....
```
```
4
```

But there’s clearly value in understanding what the full stream looks like, so
we’ll go back to including the Sys.Undo column for our final example.
Speaking of which, if we waited another four minutes until 12:07, we’d be
greeted by two additional rows in the STREAM query, whereas the TABLE query
would continue to evolve as before:

```
------------------------- -----------------------------
---
| Name | Total | Time | | Name | Total | Time |
Undo |
------------------------- -----------------------------
---
| Julie | 12 | 12:07 | | Julie | 7 | 12:01 |
|
| Frank | 3 | 12:03 | | Frank | 3 | 12:03 |
|
------------------------- | Julie | 7 | 12:03 |
undo |
| Julie | 8 | 12:03 |
|
| Julie | 8 | 12:07 |
undo |
| Julie | 12 | 12:07 |
|
........ [12:01, 12:07]
........
```
And by this time, it’s quite clear that the STREAM version of our time-varying
relation is a very different beast from the table version: the table captures a
snapshot of the entire relation _at a specific point in time_ , whereas the stream
5


captures a view of the individual changes to the relation _over time_.
Interestingly though, that means that the STREAM rendering has more in
common with our original, table-based TVR rendering:

```
---------------------------------------------------------
| [-inf, 12:01) | [12:01, 12:03) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | | | | | | Julie | 7 | 12:01 | |
| | | | | | | | | | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
| [12:03, 12:07) | [12:07, now) |
| ------------------------- | ------------------------- |
| | Name | Total | Time | | | Name | Total | Time | |
| ------------------------- | ------------------------- |
| | Julie | 8 | 12:03 | | | Julie | 12 | 12:07 | |
| | Frank | 3 | 12:03 | | | Frank | 3 | 12:03 | |
| ------------------------- | ------------------------- |
---------------------------------------------------------
```
Indeed, it’s safe to say that the STREAM query simply provides an alternate
rendering of the entire history of data that exists in the corresponding table-

based TVR query. The value of the STREAM rendering is its conciseness: it
captures only the delta of changes between each of the point-in-time relation

snapshots in the TVR. The value of the sequence-of-tables TVR rendering is the
clarity it provides: it captures the evolution of the relation over time in a
format that highlights its natural relationship to classic relations, and in doing
so provides for a simple and clear definition of relational semantics within the
context of streaming as well as the additional dimension of time that
streaming brings.

Another important aspect of the similarities between the STREAM and table-
based TVR renderings is the fact that they are essentially equivalent in the

```
5
```

overall data they encode. This gets to the core of the stream/table duality that
its proponents have long preached: streams and tables are really just two
different sides of the same coin. Or to resurrect the bad physics analogy from
Chapter 6, streams and tables are to time-varying relations as waves and
particles are to light: a complete time-varying relation is both a table and a
stream at the same time; tables and streams are simply different physical
manifestations of the same concept, depending upon the context.

Now, it’s important to keep in mind that this stream/table duality is true only
as long as both versions encode the same information; that is, when you have
full-fidelity tables or streams. In many cases, however, full fidelity is
impractical. As I alluded to earlier, encoding the full history of a time-varying
relation, no matter whether it’s in stream or table form, can be rather
expensive for a large data source. It’s quite common for stream and table
manifestations of a TVR to be lossy in some way. Tables typically encode
only the most recent version of a TVR; those that support temporal or
versioned access often compress the encoded history to specific point-in-time
snapshots, and/or garbage-collect versions that are older than some threshold.
Similarly, streams typically encode only a limited duration of the evolution of
a TVR, often a relatively recent portion of that history. Persistent streams like
Kafka afford the ability to encode the entirety of a TVR, but again this is
relatively uncommon, with data older than some threshold typically thrown
away via a garbage-collection process.

The main point here is that streams and tables are absolutely duals of one
another, each a valid way of encoding a time-varying relation. But in
practice, it’s common for the physical stream/table manifestations of a TVR
to be lossy in some way. These partial-fidelity streams and tables trade off a
decrease in total encoded information for some benefit, usually decreased
resource costs. And these types of trade-offs are important because they’re
often what allow us to build pipelines that operate over data sources of truly
massive scale. But they also complicate matters, and require a deeper
understanding to use correctly. We discuss this topic in more detail later on
when we get to SQL language extensions. But before we try to reason about
SQL extensions, it will be useful to understand a little more concretely the

```
6
```
```
7
```
biases present in both the SQL and non-SQL data processing approaches
common today.
