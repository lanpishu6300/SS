 ## Looking Forward: Toward Robust Streaming SQL

We’ve now looked at time-varying relations, the ways in which tables and
streams provide different renderings of a time-varying relation, and what the
inherent biases of the Beam and SQL models are with respect to stream and

```
12
```

table theory. So where does all of this leave us? And perhaps more to the
point, what do we need to change or add within SQL to support robust stream
processing? The surprising answer is: not much if we have good defaults.

We know that the key conceptual change is to replace classic, point-in-time
relations with time-varying relations. We saw earlier that this is a very
seamless substitution, one which applies across the full breadth of relational
operators already in existence, thanks to maintaining the critical closure
property of relational algebra. But we also saw that dealing in time-varying
relations directly is often impractical; we need the ability to operate in terms
of our two more-common physical manifestations: tables and streams. This is
where some simple extensions with good defaults come in.

We also need some tools for robustly reasoning about time, specifically event
time. This is where things like timestamps, windowing, and triggering come
into play. But again, judicious choice of defaults will be important to
minimize how often these extensions are necessary in practice.

What’s great is that we don’t really need anything more than that. So let’s
now finally spend some time looking in detail at these two categories of
extensions: _stream/table selection_ and _temporal operators_.

## Stream and Table Selection

As we worked through time-varying relation examples, we already
encountered the two key extensions related to stream and table selection.
They were those TABLE and STREAM keywords we placed after the SELECT
keyword to dictate our desired physical view of a given time-varying relation:


```
------------------------- -------------------------
| Name | Total | Time | | Name | Total | Time |
------------------------- -------------------------
| Julie | 12 | 12:07 | | Julie | 7 | 12:01 |
| Frank | 3 | 12:03 | | Frank | 3 | 12:03 |
------------------------- | Julie | 8 | 12:03 |
| Julie | 12 | 12:07 |
..... [12:01, 12:07] ....
```
These extensions are relatively straightforward and easy to use when
necessary. But the really important thing regarding stream and table selection
is the choice of good defaults for times when they aren’t explicitly provided.
Such defaults should honor the classic, table-biased behavior of SQL that
everyone is accustomed to, while also operating intuitively in a world that
includes streams. They should also be easy to remember. The goal here is to
help maintain a natural feel to the system, while also greatly decreasing the
frequency with which we must use explicit extensions. A good choice of
defaults that satisfies all of these requirements is:

```
If all of the inputs are tables , the output is a TABLE.
```
```
If any of the inputs are streams , the output is a STREAM.
```
What’s additionally important to call out here is that these physical
renderings of a time-varying relation are really only necessary when you
want to materialize the TVR in some way, either to view it directly or write it
to some output table or stream. Given a SQL system that operates under the
covers in terms of full-fidelity time-varying relations, intermediate results
(e.g., WITH AS or SELECT INTO statements) can remain as full-fidelity TVRs
in whatever format the system naturally deals in, with no need to render them
into some other, more limited concrete manifestation.

And that’s really it for stream and table selection. Beyond the ability to deal
in streams and tables directly, we also need some better tools for reasoning
about time if we want to support robust, out-of-order stream processing
within SQL. Let’s now look in more detail about what those entail.


## Temporal Operators

The foundation of robust, out-of-order processing is the event-time
timestamp: that small piece of metadata that captures the time at which an
event occurred rather than the time at which it is observed. In a SQL world,
event time is typically just another column of data for a given TVR, one
which is natively present in the source data themselves. In that sense, this
idea of materializing a record’s event time within the record itself is
something SQL already handles naturally by putting a timestamp in a regular
column.

Before we go any further, let’s look at an example. To help tie all of this SQL
stuff together with the concepts we’ve explored previously in the book, we
resurrect our running example of summing up nine scores from various
members of a team to arrive at that team’s total score. If you recall, those
scores look like Figure 8-6 when plotted on X = event-time/Y = processing-
time axes.

```
Figure 8-6. Data points in our running example
```
If we were to imagine these data as a classic SQL table, they might look

```
13
```

something like this, ordered by event time (left-to-right in Figure 8-6):

```
------------------------------------------------
| Name | Team | Score | EventTime | ProcTime |
------------------------------------------------
| Julie | TeamX | 5 | 12:00:26 | 12:05:19 |
| Frank | TeamX | 9 | 12:01:26 | 12:08:19 |
| Ed | TeamX | 7 | 12:02:26 | 12:05:39 |
| Julie | TeamX | 8 | 12:03:06 | 12:07:06 |
| Amy | TeamX | 3 | 12:03:39 | 12:06:13 |
| Fred | TeamX | 4 | 12:04:19 | 12:06:39 |
| Naomi | TeamX | 3 | 12:06:39 | 12:07:19 |
| Becky | TeamX | 8 | 12:07:26 | 12:08:39 |
| Naomi | TeamX | 1 | 12:07:46 | 12:09:00 |
------------------------------------------------
```
If you recall, we saw this table way back in Chapter 2 when I first introduced
this dataset. This rendering provides a little more detail on the data than
we’ve typically shown, explicitly highlighting the fact that the nine scores
themselves belong to seven different users, each a member of the same team.
SQL provides a nice, concise way to see the data laid out fully before we
begin diving into examples.

Another nice thing about this view of the data is that it fully captures the
event time and processing time for each record. You can imagine the event-
time column as being just another piece of the original data, and the
processing-time column as being something supplied by the system (in this

case, using a hypothetical Sys.MTime column that records the processing-
time modification timestamp of a given row; that is, the time at which that
row arrived in the source table), capturing the ingress time of the records
themselves into the system.

The fun thing about SQL is how easy it is to view your data in different ways.
For example, if we instead want to see the data in processing-time order
(bottom-to-top in Figure 8-6), we could simply update the ORDER BY clause:


```
-----------------------------------------------
| Name | Team | Score | EventTime | ProcTime |
-----------------------------------------------
| Julie | TeamX | 5 | 12:00:26 | 12:05:19 |
| Ed | TeamX | 7 | 12:02:26 | 12:05:39 |
| Amy | TeamX | 3 | 12:03:39 | 12:06:13 |
| Fred | TeamX | 4 | 12:04:19 | 12:06:39 |
| Julie | TeamX | 8 | 12:03:06 | 12:07:06 |
| Naomi | TeamX | 3 | 12:06:39 | 12:07:19 |
| Frank | TeamX | 9 | 12:01:26 | 12:08:19 |
| Becky | TeamX | 8 | 12:07:26 | 12:08:39 |
| Naomi | TeamX | 1 | 12:07:46 | 12:09:00 |
------------------------------------------------
```
As we learned earlier, these table renderings of the data are really a partial-
fidelity view of the complete underlying TVR. If we were to instead query
the full table-oriented TVR (but only for the three most important columns, for
the sake of brevity), it would expand to something like this:

```
-----------------------------------------------------------------------
| [-inf, 12:05:19) | [12:05:19, 12:05:39) |
| -------------------------------- | -------------------------------- |
| | Score | EventTime | ProcTime | | | Score | EventTime | ProcTime | |
| -------------------------------- | -------------------------------- |
| -------------------------------- | | 5 | 12:00:26 | 12:05:19 | |
| | -------------------------------- |
| | |
-----------------------------------------------------------------------
| [12:05:39, 12:06:13) | [12:06:13, 12:06:39) |
| -------------------------------- | -------------------------------- |
| | Score | EventTime | ProcTime | | | Score | EventTime | ProcTime | |
| -------------------------------- | -------------------------------- |
| | 5 | 12:00:26 | 12:05:19 | | | 5 | 12:00:26 | 12:05:19 | |
| | 7 | 12:02:26 | 12:05:39 | | | 7 | 12:02:26 | 12:05:39 | |
| -------------------------------- | | 3 | 12:03:39 | 12:06:13 | |
| | -------------------------------- |
-----------------------------------------------------------------------
```

```
| [12:06:39, 12:07:06) | [12:07:06, 12:07:19) |
| -------------------------------- | -------------------------------- |
| | Score | EventTime | ProcTime | | | Score | EventTime | ProcTime | |
| -------------------------------- | -------------------------------- |
| | 5 | 12:00:26 | 12:05:19 | | | 5 | 12:00:26 | 12:05:19 | |
| | 7 | 12:02:26 | 12:05:39 | | | 7 | 12:02:26 | 12:05:39 | |
| | 3 | 12:03:39 | 12:06:13 | | | 3 | 12:03:39 | 12:06:13 | |
| | 4 | 12:04:19 | 12:06:39 | | | 4 | 12:04:19 | 12:06:39 | |
| -------------------------------- | | 8 | 12:03:06 | 12:07:06 | |
| | -------------------------------- |
-----------------------------------------------------------------------
| [12:07:19, 12:08:19) | [12:08:19, 12:08:39) |
| -------------------------------- | -------------------------------- |
| | Score | EventTime | ProcTime | | | Score | EventTime | ProcTime | |
| -------------------------------- | -------------------------------- |
| | 5 | 12:00:26 | 12:05:19 | | | 5 | 12:00:26 | 12:05:19 | |
| | 7 | 12:02:26 | 12:05:39 | | | 7 | 12:02:26 | 12:05:39 | |
| | 3 | 12:03:39 | 12:06:13 | | | 3 | 12:03:39 | 12:06:13 | |
| | 4 | 12:04:19 | 12:06:39 | | | 4 | 12:04:19 | 12:06:39 | |
| | 8 | 12:03:06 | 12:07:06 | | | 8 | 12:03:06 | 12:07:06 | |
| | 3 | 12:06:39 | 12:07:19 | | | 3 | 12:06:39 | 12:07:19 | |
| -------------------------------- | | 9 | 12:01:26 | 12:08:19 | |
| | -------------------------------- |
| | |
-----------------------------------------------------------------------
| [12:08:39, 12:09:00) | [12:09:00, now) |
| -------------------------------- | -------------------------------- |
| | Score | EventTime | ProcTime | | | Score | EventTime | ProcTime | |
| -------------------------------- | -------------------------------- |
| | 5 | 12:00:26 | 12:05:19 | | | 5 | 12:00:26 | 12:05:19 | |
| | 7 | 12:02:26 | 12:05:39 | | | 7 | 12:02:26 | 12:05:39 | |
| | 3 | 12:03:39 | 12:06:13 | | | 3 | 12:03:39 | 12:06:13 | |
| | 4 | 12:04:19 | 12:06:39 | | | 4 | 12:04:19 | 12:06:39 | |
| | 8 | 12:03:06 | 12:07:06 | | | 8 | 12:03:06 | 12:07:06 | |
| | 3 | 12:06:39 | 12:07:19 | | | 3 | 12:06:39 | 12:07:19 | |
| | 9 | 12:01:26 | 12:08:19 | | | 9 | 12:01:26 | 12:08:19 | |
| | 8 | 12:07:26 | 12:08:39 | | | 8 | 12:07:26 | 12:08:39 | |
| -------------------------------- | | 1 | 12:07:46 | 12:09:00 | |
| | -------------------------------- |
-----------------------------------------------------------------------
```
That’s a lot of data. Alternatively, the STREAM version would render much


more compactly in this instance; thanks to there being no explicit grouping in
the relation, it looks essentially identical to the point-in-time TABLE rendering
earlier, with the addition of the trailing footer describing the range of
processing time captured in the stream so far, plus the note that the system is
still waiting for more data in the stream (assuming we’re treating the stream
as unbounded; we’ll see a bounded version of the stream shortly):

```
--------------------------------
| Score | EventTime | ProcTime |
--------------------------------
| 5 | 12:00:26 | 12:05:19 |
| 7 | 12:02:26 | 12:05:39 |
| 3 | 12:03:39 | 12:06:13 |
| 4 | 12:04:19 | 12:06:39 |
| 8 | 12:03:06 | 12:07:06 |
| 3 | 12:06:39 | 12:07:19 |
| 9 | 12:01:26 | 12:08:19 |
| 8 | 12:07:26 | 12:08:39 |
| 1 | 12:07:46 | 12:09:00 |
........ [12:00, 12:10] ........
```
But this is all just looking at the raw input records without any sort of
transformations. Much more interesting is when we start altering the
relations. When we’ve explored this example in the past, we’ve always
started with classic batch processing to sum up the scores over the entire
dataset, so let’s do the same here. The first example pipeline (previously
provided as Example 6-1) looked like Example 8-1 in Beam.

_Example 8-1. Summation pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals =
input.apply(Sum.integersPerKey());

And rendered in the streams and tables view of the world, that pipeline’s
execution looked like Figure 8-7.


```
Figure 8-7. Streams and tables view of classic batch processing
```
Given that we already have our data placed into an appropriate schema, we
won’t be doing any parsing in SQL; instead, we focus on everything in the
pipeline after the parse transformation. And because we’re going with the
classic batch model of retrieving a single answer only after all of the input
data have been processed, the TABLE and STREAM views of the summation
relation would look essentially identical (recall that we’re dealing with
bounded versions of our dataset for these initial, batch-style examples; as a
result, this STREAM query actually terminates with a line of dashes and an
END-OF-STREAM marker):

```
------------------------------------------
| Total | MAX(EventTime) | MAX(ProcTime) |
------------------------------------------
| 48 | 12:07:46 | 12:09:00 |
------------------------------------------
```
```
00:00 / 00:00
```

```
------------------------------------------
| Total | MAX(EventTime) | MAX(ProcTime) |
------------------------------------------
| 48 | 12:07:46 | 12:09:00 |
------ [12:00, 12:10] END-OF-STREAM ------
```
More interesting is when we start adding windowing into the mix. That will
give us a chance to begin looking more closely at the temporal operations that
need to be added to SQL to support robust stream processing.

**_Where_** **: windowing**

As we learned in Chapter 6, windowing is a modification of grouping by key,
in which the window becomes a secondary part of a hierarchical key. As with
classic programmatic batch processing, you can window data into more
simplistic windows quite easily within SQL as it exists now by simply

including time as part of the GROUP BY parameter. Or, if the system in
question provides it, you can use a built-in windowing operation. We look at
SQL examples of both in a moment, but first, let’s revisit the programmatic
version from Chapter 3. Thinking back to Example 6-2, the windowed Beam
pipeline looked like that shown in Example 8-2.

_Example 8-2. Summation pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES)))
.apply(Sum.integersPerKey());

And the execution of that pipeline (in streams and tables rendering from
Figure 6-5), looked like the diagrams presented in Figure 8-8.


```
Figure 8-8. Streams and tables view of windowed summation on a batch engine
```
As we saw before, the only material change from Figure 8-7 to 8-8 is that the
table created by the SUM operation is now partitioned into fixed, two-minute
windows of time, yielding four windowed answers at the end rather than the
single global sum that we had previously.

To do the same thing in SQL, we have two options: implicitly window by
including some unique feature of the window (e.g., the end timestamp) in the
GROUP BY statement, or use a built-in windowing operation. Let’s look at
both.

First, ad hoc windowing. In this case, we perform the math of calculating
windows ourselves in our SQL statement:

```
------------------------------------------------
```
```
00:00 / 00:00
```

```
| Total | Window | MAX(ProcTime) |
------------------------------------------------
| 14 | [12:00:00, 12:02:00) | 12:08:19 |
| 18 | [12:02:00, 12:04:00) | 12:07:06 |
| 4 | [12:04:00, 12:06:00) | 12:06:39 |
| 12 | [12:06:00, 12:08:00) | 12:09:00 |
------------------------------------------------
```
We can also achieve the same result using an explicit windowing statement
such as those supported by Apache Calcite:

```
------------------------------------------------
| Total | Window | MAX(ProcTime) |
------------------------------------------------
| 14 | [12:00:00, 12:02:00) | 12:08:19 |
| 18 | [12:02:00, 12:04:00) | 12:07:06 |
| 4 | [12:04:00, 12:06:00) | 12:06:39 |
| 12 | [12:06:00, 12:08:00) | 12:09:00 |
------------------------------------------------
```
This then begs the question: if we can implicitly window using existing SQL
constructs, why even bother supporting explicit windowing constructs? There
are two reasons, only the first of which is apparent in this example (we’ll see
the other one in action later on in the chapter):

```
1. Windowing takes care of the window-computation math for you. It’s
a lot easier to consistently get things right when you specify basic
parameters like width and slide directly rather than computing the
window math yourself.
2. Windowing allows the concise expression of more complex,
dynamic groupings such as sessions. Even though SQL is technically
able to express the every-element-within-some-temporal-gap-of-
```
```
14
```

```
another-element relationship that defines session windows, the
corresponding incantation is a tangled mess of analytic functions,
self joins, and array unnesting that no mere mortal could be
reasonably expected to conjure on their own.
```
Both are compelling arguments for providing first-class windowing
constructs in SQL, in addition to the ad hoc windowing capabilities that
already exist.

At this point, we’ve seen what windowing looks like from a classic
batch/classic relational perspective when consuming the data as a table. But if
we want to consume the data as a stream, we get back to that third question
from the Beam Model: when in processing time do we materialize outputs?

**_When_** **: triggers**

The answer to that question, as before, is triggers and watermarks. However,
in the context of SQL, there’s a strong argument to be made for having a
different set of defaults than those we introduced with the Beam Model in
Chapter 3: rather than defaulting to using a single watermark trigger, a more
SQL-ish default would be to take a cue from materialized views and trigger
on every element. In other words, any time a new input arrives, we produce a
corresponding new output.

_A SQL-ish default: per-record triggers_

There are two compelling benefits to using trigger-every-record as the
default:

Simplicity

```
The semantics of per-record updates are easy to understand; materialized
views have operated this way for years.
```
Fidelity

```
As in change data capture systems, per-record triggering yields a full-
fidelity stream rendering of a given time-varying relation; no information
is lost as part of the conversion.
```

The downside is primarily cost: triggers are always applied after a grouping
operation, and the nature of grouping often presents an opportunity to reduce
the cardinality of data flowing through the system, thus commensurately
reducing the cost of further processing those aggregate results downstream.
Even so, the benefits in clarity and simplicity for use cases where cost is not
prohibitive arguably outweigh the cognitive complexity of defaulting to a
non-full-fidelity trigger up front.

Thus, for our first take at consuming aggregate team scores as a stream, let’s
see what things would look like using a per-record trigger. Beam itself
doesn’t have a precise per-record trigger, so, as demonstrated in Example 8-

3 , we instead use a repeated AfterCount(1) trigger, which will fire
immediately any time a new record arrives.

_Example 8-3. Per-record trigger_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(Repeatedly(AfterCount(1)))
.apply(Sum.integersPerKey());

A streams and tables rendering of this pipeline would then look something
like that depicted in Figure 8-9.

```
Figure 8-9. Streams and tables view of windowed summation on a streaming engine with
per-record triggering
```
An interesting side effect of using per-record triggers is how it somewhat
masks the effect of data being brought to rest because they are then
immediately put back into motion again by the trigger. Even so, the aggregate

```
00:00 / 00:00
```

artifact from the grouping remains at rest in the table, as the ungrouped
stream of values flows away from it.

Moving back to SQL, we can see now what the effect of rendering the
corresponding time-value relation as a stream would be. It (unsurprisingly)
looks a lot like the stream of values in the animation in Figure 8-9:

```
------------------------------------------------
| Total | Window | MAX(ProcTime) |
------------------------------------------------
| 5 | [12:00:00, 12:02:00) | 12:05:19 |
| 7 | [12:02:00, 12:04:00) | 12:05:39 |
| 10 | [12:02:00, 12:04:00) | 12:06:13 |
| 4 | [12:04:00, 12:06:00) | 12:06:39 |
| 18 | [12:02:00, 12:04:00) | 12:07:06 |
| 3 | [12:06:00, 12:08:00) | 12:07:19 |
| 14 | [12:00:00, 12:02:00) | 12:08:19 |
| 11 | [12:06:00, 12:08:00) | 12:08:39 |
| 12 | [12:06:00, 12:08:00) | 12:09:00 |
................ [12:00, 12:10] ................
```
But even for this simple use case, it’s pretty chatty. If we’re building a
pipeline to process data for a large-scale mobile application, we might not
want to pay the cost of processing downstream updates for each and every
upstream user score. This is where custom triggers come in.

_Watermark triggers_

If we were to switch the Beam pipeline to use a watermark trigger, for
example, we could get exactly one output per window in the stream version
of the TVR, as demonstrated in Example 8-4 and shown in Figure 8-10.

_Example 8-4. Watermark trigger_

PCollection<String> raw = IO.read(...);


PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark())
.apply(Sum.integersPerKey());

```
Figure 8-10. Windowed summation with watermark triggering
```
To get the same effect in SQL, we’d need language support for specifying a

custom trigger. Something like an EMIT _<when>_ statement, such as EMIT
WHEN WATERMARK PAST _<column>_. This would signal to the system that the
table created by the aggregation should be triggered into a stream exactly
once per row, when the input watermark for the table exceeds the timestamp
value in the specified column (which in this case happens to be the end of the
window).

Let’s look at this relation rendered as a stream. From the perspective of
understanding when trigger firings occur, it’s also handy to stop relying on
the MTime values from the original inputs and instead capture the current
timestamp at which rows in the stream are emitted:

```
-------------------------------------------
| Total | Window | EmitTime |
-------------------------------------------
| 5 | [12:00:00, 12:02:00) | 12:06:00 |
```
```
00:00 / 00:00
```

```
| 18 | [12:02:00, 12:04:00) | 12:07:30 |
| 4 | [12:04:00, 12:06:00) | 12:07:41 |
| 12 | [12:06:00, 12:08:00) | 12:09:22 |
............. [12:00, 12:10] ..............
```
The main downside here is the late data problem due to the use of a heuristic
watermark, as we encountered in previous chapters. In light of late data, a
nicer option might be to also immediately output an update any time a late
record shows up, using a variation on the watermark trigger that supported
repeated late firings, as shown in Example 8-5 and Figure 8-11.

_Example 8-5. Watermark trigger with late firings_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark()
.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

```
Figure 8-11. Windowed summation with on-time/late triggering
```
We can do the same thing in SQL by allowing the specification of two
triggers:

```
A watermark trigger to give us an initial value: WHEN WATERMARK
PAST <column> , with the end of the window used as the timestamp
<column>.
```
```
A repeated delay trigger for late data: AND THEN AFTER
<duration> , with a <duration> of 0 to give us per-record
semantics.
```
```
00:00 / 00:00
```

Now that we’re getting multiple rows per window, it can also be useful to
have another two system columns available: the timing of each row/pane for
a given window relative to the watermark (Sys.EmitTiming), and the index
of the pane/row for a given window (Sys.EmitIndex, to identify the
sequence of revisions for a given row/window):

```
-----------------------------------------------------------------------
-----
| Total | Window | EmitTime | Sys.EmitTiming |
Sys.EmitIndex |
-----------------------------------------------------------------------
-----
| 5 | [12:00:00, 12:02:00) | 12:06:00 | on-time | 0
|
| 18 | [12:02:00, 12:04:00) | 12:07:30 | on-time | 0
|
| 4 | [12:04:00, 12:06:00) | 12:07:41 | on-time | 0
|
| 14 | [12:00:00, 12:02:00) | 12:08:19 | late | 1
|
| 12 | [12:06:00, 12:08:00) | 12:09:22 | on-time | 0
|
.............................. [12:00, 12:10]
..............................
```
For each pane, using this trigger, we’re able to get a single on-time answer
that is likely to be correct, thanks to our heuristic watermark. And for any
data that arrives late, we can get an updated version of the row amending our
previous results.


_Repeated delay triggers_

The other main temporal trigger use case you might want is repeated delayed
updates; that is, trigger a window one minute (in processing time) after any
new data for it arrive. Note that this is different than triggering on aligned
boundaries, as you would get with a microbatch system. As Example 8-6
shows, triggering via a delay relative to the most recent new record arriving
for the window/row helps spread triggering load out more evenly than a
bursty, aligned trigger would. It also does not require any sort of watermark
support. Figure 8-12 presents the results.

_Example 8-6. Repeated triggering with one-minute delays_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(Repeatedly(UnalignedDelay(ONE_MINUTE)))
.apply(Sum.integersPerKey());

```
Figure 8-12. Windowed summation with repeated one-minute-delay triggering
```
The effect of using such a trigger is very similar to the per-record triggering
we started out with but slightly less chatty thanks to the additional delay
introduced in triggering, which allows the system to elide some number of
the rows being produced. Tweaking the delay allows us to tune the volume of
data generated, and thus balance the tensions of cost and timeliness as
appropriate for the use case.

Rendered as a SQL stream, it would look something like this:

```
00:00 / 00:00
```

```
-----------------------------------------------------------------------
-----
| Total | Window | EmitTime | Sys.EmitTiming |
Sys.EmitIndex |
-----------------------------------------------------------------------
-----
| 5 | [12:00:00, 12:02:00) | 12:06:19 | n/a | 0
|
| 10 | [12:02:00, 12:04:00) | 12:06:39 | n/a | 0
|
| 4 | [12:04:00, 12:06:00) | 12:07:39 | n/a | 0
|
| 18 | [12:02:00, 12:04:00) | 12:08:06 | n/a | 1
|
| 3 | [12:06:00, 12:08:00) | 12:08:19 | n/a | 0
|
| 14 | [12:00:00, 12:02:00) | 12:09:19 | n/a | 1
|
| 12 | [12:06:00, 12:08:00) | 12:09:22 | n/a | 1
|
.............................. [12:00, 12:10]
..............................
```
_Data-driven triggers_

Before moving on to the final question in the Beam Model, it’s worth briefly
discussing the idea of _data-driven triggers_. Because of the dynamic way
types are handled in SQL, it might seem like data-driven triggers would be a

very natural addition to the proposed EMIT _<when>_ clause. For example,
what if we want to trigger our summation any time the total score exceeds

10? Wouldn’t something like EMIT WHEN Score > 10 work very naturally?

Well, yes and no. Yes, such a construct would fit very naturally. But when
you think about what would actually be happening with such a construct, you


essentially would be triggering on every record, and then executing the Score
> 10 predicate to decide whether the triggered row should be propagated
downstream. As you might recall, this sounds a lot like what happens with a
HAVING clause. And, indeed, you can get the exact same effect by simply
prepending HAVING Score > 10 to the end of the query. At which point, it
begs the question: is it worth adding explicit data-driven triggers? Probably
not. Even so, it’s still encouraging to see just how easy it is to get the desired
effect of data-driven triggers using standard SQL and well-chosen defaults.

**_How_** **: accumulation**

So far in this section, we’ve been ignoring the Sys.Undo column that I
introduced toward the beginning of this chapter. As a result, we’ve defaulted
to using _accumulating mode_ to answer the question of _how_ refinements for a
window/row relate to one another. In other words, any time we observed
multiple revisions of an aggregate row, the later revisions built upon the
previous revisions, accumulating new inputs together with old ones. I opted
for this approach because it matches the approach used in an earlier chapter,
and it’s a relatively straightforward translation from how things work in a
table world.

That said, accumulating mode has some major drawbacks. In fact, as we
discussed in Chapter 2, it’s plain broken for any query/pipeline with a
sequence of two or more grouping operations due to over counting. The only
sane way to allow for the consumption of multiple revisions of a row within a
system that allows for queries containing more than one serial grouping
operation is if it operates by default in _accumulating and retracting_ mode.
Otherwise, you run into issues where a given input record is included
multiple times in a single aggregation due to the blind incorporation of
multiple revisions for a single row.

So, when we come to the question of incorporating accumulation mode
semantics into a SQL world, the option that fits best with our goal of
providing an intuitive and natural experience is if the system uses retractions
by default under the covers. As noted when I introduced the Sys.Undo
column earlier, if you don’t care about the retractions (as in the examples in

```
15
```

this section up until now), you don’t need to ask for them. But if you do ask
for them, they should be there.

_Retractions in a SQL world_

To see what I mean, let’s look at another example. To motivate the problem
appropriately, let’s look at a use case that’s relatively impractical without
retractions: building session windows and writing them incrementally to a
key/value store like HBase. In this case, we’ll be producing incremental
sessions from our aggregation as they are built up. But in many cases, a given
session will simply be an evolution of one or more previous sessions. In that
case, you’d really like to delete the previous session(s) and replace it/them
with the new one. But how do you do that? The only way to tell whether a
given session replaces another one is to compare them to see whether the new
one overlaps the old one. But that means duplicating some of the session-
building logic in a separate part of your pipeline. And, more important, it
means that you no longer have idempotent output, and you’ll thus need to
jump through a bunch of extra hoops if you want to maintain end-to-end
exactly-once semantics. Far better would be for the pipeline to simply tell
you which sessions were removed and which were added in their place. This
is what retractions give you.

To see this in action (and in SQL), let’s modify our example pipeline to
compute session windows with a gap duration of one minute. For simplicity
and clarity, we go back to using the default per-record trigger. Note that I’ve
also shifted a few of the data points within processing time for these session
examples to make the diagram cleaner; event-time timestamps remain the
same. The updated dataset looks like this (with shifted processing-time
timestamps highlighted in yellow):

```
--------------------------------
| Score | EventTime | ProcTime |
--------------------------------
| 5 | 12:00:26 | 12:05:19 |
```

```
| 7 | 12:02:26 | 12:05:39 |
| 3 | 12:03:39 | 12:06:13 |
| 4 | 12:04:19 | 12:06:46 | # Originally 12:06:39
| 3 | 12:06:39 | 12:07:19 |
| 8 | 12:03:06 | 12:07:33 | # Originally 12:07:06
| 8 | 12:07:26 | 12:08:13 | # Originally 12:08:39
| 9 | 12:01:26 | 12:08:19 |
| 1 | 12:07:46 | 12:09:00 |
........ [12:00, 12:10] ........
```
To begin with, let’s look at the pipeline without retractions. After it’s clear
why that pipeline is problematic for the use case of writing incremental
sessions to a key/value store, we’ll then look at the version with retractions.

The Beam code for the nonretracting pipeline would look something like
Example 8-7. Figure 8-13 shows the results.

_Example 8-7. Session windows with per-record triggering and accumulation
but no retractions_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(Sessions.withGapDuration(ONE_MINUTE))
.triggering(Repeatedly(AfterCount(1))
.accumulatingFiredPanes())
.apply(Sum.integersPerKey());

```
Figure 8-13. Session window summation with accumulation but no retractions
```
And finally, rendered in SQL, the output stream would look like this:

```
00:00 / 00:00
```

```
-------------------------------------------
| Total | Window | EmitTime |
-------------------------------------------
| 5 | [12:00:26, 12:01:26) | 12:05:19 |
| 7 | [12:02:26, 12:03:26) | 12:05:39 |
| 3 | [12:03:39, 12:04:39) | 12:06:13 |
| 7 | [12:03:39, 12:05:19) | 12:06:46 |
| 3 | [12:06:39, 12:07:39) | 12:07:19 |
| 22 | [12:02:26, 12:05:19) | 12:07:33 |
| 11 | [12:06:39, 12:08:26) | 12:08:13 |
| 36 | [12:00:26, 12:05:19) | 12:08:19 |
| 12 | [12:06:39, 12:08:46) | 12:09:00 |
............. [12:00, 12:10] ..............
```
The important thing to notice in here (in the animation as well as the SQL
rendering) is what the stream of incremental sessions looks like. From our
holistic viewpoint, it’s pretty easy to visually identify in the animation which
later sessions supersede those that came before. But imagine receiving
elements in this stream one by one (as in the SQL listing) and needing to
write them to HBase in a way that eventually results in the HBase table
containing only the two final sessions (with values 36 and 12). How would
you do that? Well, you’d need to do a bunch of read-modify-write operations
to read all of the existing sessions for a key, compare them with the new
session, determine which ones overlap, issue deletes for the obsolete sessions,
and then finally issue a write for the new session—all at significant additional
cost, and with a loss of idempotence, which would ultimately leave you
unable to provide end-to-end, exactly-once semantics. It’s just not practical.

Contrast this then with the same pipeline, but with retractions enabled, as
demonstrated in Example 8-8 and depicted in Figure 8-14.

_Example 8-8. Session windows with per-record triggering, accumulation, and
retractions_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());


PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(Sessions.withGapDuration(ONE_MINUTE))
.triggering(Repeatedly(AfterCount(1))
.accumulatingAndRetractingFiredPanes())
.apply(Sum.integersPerKey());

```
Figure 8-14. Session window summation with accumulation and retractions
```
And, lastly, in SQL form. For the SQL version, we’re assuming that the
system is using retractions under the covers by default, and individual
retraction rows are then materialized in the stream any time we request the
special Sys.Undo column. As I described originally, the value of that
column is that it allows us to distinguish retraction rows (labeled undo in the

Sys.Undo column) from normal rows (unlabeled in the Sys.Undo column
here for clearer contrast, though they could just as easily be labeled redo,
instead):

```
--------------------------------------------------
| Total | Window | EmitTime | Undo |
--------------------------------------------------
| 5 | [12:00:26, 12:01:26) | 12:05:19 | |
| 7 | [12:02:26, 12:03:26) | 12:05:39 | |
| 3 | [12:03:39, 12:04:39) | 12:06:13 | |
| 3 | [12:03:39, 12:04:39) | 12:06:46 | undo |
| 7 | [12:03:39, 12:05:19) | 12:06:46 | |
```
```
00:00 / 00:00
```
```
16
```

```
| 3 | [12:06:39, 12:07:39) | 12:07:19 | |
| 7 | [12:02:26, 12:03:26) | 12:07:33 | undo |
| 7 | [12:03:39, 12:05:19) | 12:07:33 | undo |
| 22 | [12:02:26, 12:05:19) | 12:07:33 | |
| 3 | [12:06:39, 12:07:39) | 12:08:13 | undo |
| 11 | [12:06:39, 12:08:26) | 12:08:13 | |
| 5 | [12:00:26, 12:01:26) | 12:08:19 | undo |
| 22 | [12:02:26, 12:05:19) | 12:08:19 | undo |
| 36 | [12:00:26, 12:05:19) | 12:08:19 | |
| 11 | [12:06:39, 12:08:26) | 12:09:00 | undo |
| 12 | [12:06:39, 12:08:46) | 12:09:00 | |
................. [12:00, 12:10] .................
```
With retractions included, the sessions stream no longer just includes new
sessions, but also retractions for the old sessions that have been replaced.
With this stream, it’s trivial to properly build up the set of sessions in
HBase over time: you simply write new sessions as they arrive (unlabeled
redo rows) and delete old sessions as they’re retracted (undo rows). Much
better!

_Discarding mode, or lack thereof_

With this example, we’ve shown how simply and naturally you can
incorporate retractions into SQL to provide both _accumulating mode_ and
_accumulating and retracting mode_ semantics. But what about _discarding
mode_?

For specific use cases such as very simple pipelines that partially aggregate
high-volume input data via a single grouping operation and then write them
into a storage system, which itself supports aggregation (e.g., a database-like
system), discarding mode can be extremely valuable as a resource-saving
option. But outside of those relatively narrow use cases, discarding mode is
confusing and error-prone. As such, it’s probably not worth incorporating
directly into SQL. Systems that need it can provide it as an option outside of
the SQL language itself. Those that don’t can simply provide the more
natural default of _accumulating and retracting mode_ , with the option to
ignore retractions when they aren’t needed.

```
17
```

