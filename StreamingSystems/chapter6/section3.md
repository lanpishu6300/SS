 ## What , Where , When , and How in a Streams and

## Tables World

In this section, we look at each of the four questions and see how they relate


to streams and tables. We’ll also answer any questions that may be lingering
from the previous section, one big one being: if grouping is the thing that
brings data to rest, what precisely is the “ungrouping” inverse that puts them
in motion? More on that later. But for now, on to transformations.

## What : Transformations

In Chapter 3, we learned that transformations tell us _what_ the pipeline is
computing; that is, whether it’s building models, counting sums, filtering
spam, and so on. We saw in the earlier MapReduce example that four of the
six stages answered _what_ questions:

```
Map and Reduce both applied the pipeline author’s element-wise
transformation on each key/value or key/value-list pair in the input
stream, respectively, yielding a new, transformed stream.
```
```
MapWrite and ReduceWrite both grouped the outputs from the
previous stage according to the key assigned by that stage (possibly
implicitly, in the optional Reduce case), and in doing so transformed
the input stream into an output table.
```
Viewed in that light, you can see that there are essentially two types of _what_
transforms from the perspective of stream/table theory:

Nongrouping

```
These operations (as we saw in Map and Reduce) simply accept a stream
of records and produce a new, transformed stream of records on the other
side. Examples of nongrouping transformations are filters (e.g., removing
spam messages), exploders (i.e., splitting apart a larger composite record
into its constituent parts), and mutators (e.g., divide by 100), and so on.
```
Grouping

```
These operations (as we saw in MapWrite and ReduceWrite) accept a
stream of records and group them together in some way, thereby
transforming the stream into a table. Examples of grouping
transformations are joins, aggregations, list/set accumulation, changelog
```

```
application, histogram creation, machine learning model training, and so
forth.
```
To get a better sense for how all of this ties together, let’s look at an updated
version of Figure 2-2, where we first began to look at transformations. To
save you jumping back there to see what we were talking about, Example 6-1
contains the code snippet we were using.

_Example 6-1. Summation pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals =
input.apply(Sum.integersPerKey());

This pipeline is simply reading in input data, parsing individual team member
scores, and then summing those scores per team. The event-time/processing-
time visualization of it looks like the diagram presented in Figure 6-3.

```
Figure 6-3. Event-time/processing-time view of classic batch processing
```
Figure 6-4 depicts a more topological view of this pipeline over time,
rendered from a streams-and-tables perspective.

```
00:00 / 00:00
```

```
Figure 6-4. Streams and tables view of classic batch processing
```
In the streams and tables version of this visualization, the passage of time is
manifested by scrolling the graph area downward in the processing-time
dimension (y-axis) as time advances. The nice thing about rendering things
this way is that it very clearly calls out the difference between nongrouping
and grouping operations. Unlike our previous diagrams, in which I elided all
initial transformations in the pipeline other than the Sum.integersByKey,
I’ve included the initial parsing operation here, as well, because the
nongrouping aspect of the parsing operation provides a nice contrast to the
grouping aspect of the summation. Viewed in this light, it’s very easy to see
the difference between the two. The nongrouping operation does nothing to
halt the motion of the elements in the stream, and as a result yields another
stream on the other side. In contrast, the grouping operation brings all the
elements in the stream to rest as it adds them together into the final sum.
Because this example was running on a batch processing engine over
bounded data, the final results are emitted only after the end of the input is
reached. As we noted in Chapter 2 this example is sufficient for bounded
data, but is too limiting in the context of unbounded data because the input
will theoretically never end. But is it really insufficient?

Looking at the new streams/tables portion of the diagram, if all we’re doing

```
00:00 / 00:00
```

is calculating sums as our final results (and not actually transforming those
sums in any additional way further downstream within the pipeline), the table
we created with our grouping operation has our answer sitting right there,
evolving over time as new data arrive. Why don’t we just read our results
from there?

This is exactly the point being made by the folks championing stream
processors as a database (primarily the Kafka and Flink crews): anywhere
you have a grouping operation in your pipeline, you’re creating a table that
includes what is effectively the output values of that portion of the stage. If
those output values happen to be the final thing your pipeline is calculating,
you don’t need to rematerialize them somewhere else if you can read them
directly out of that table. Besides providing quick and easy access to results
as they evolve over time, this approach saves on compute resources by not
requiring an additional sink stage in the pipeline to materialize the outputs,
yields disk savings by eliminating redundant data storage, and obviates the
need for any engineering work building the aforementioned sink stages. The
only major caveat is that you need to take care to ensure that only the data
processing pipeline has the ability to make modifications to the table. If the
values in the table can change out from under the pipeline due to external
modification, all bets are off regarding consistency guarantees.

A number of folks in the industry have been recommending this approach for
a while now, and it’s being put to great use in a variety of scenarios. We’ve
seen MillWheel customers within Google do the same thing by serving data
directly out of their Bigtable-based state tables, and we’re in the process of
adding first-class support for accessing state from outside of your pipeline in
the C++–based Apache Beam equivalent we use internally at Google (Google
Flume); hopefully those concepts will make their way to Apache Beam
proper someday soon, as well.

Now, reading from the state tables is great if the values therein are your final
results. But, if you have more processing to perform downstream in the
pipeline (e.g., imagine our pipeline was actually computing the top scoring
team), we still need some better way to cope with unbounded data, allowing
us to transform the table back into a stream in a more incremental fashion.

```
8
```
```
9
```

For that, we’ll want to journey back through the remaining three questions,
beginning with windowing, expanding into triggering, and finally tying it all
together with accumulation.

## Where : Windowing

As we know from Chapter 3, windowing tells us _where_ in event time
grouping occurs. Combined with our earlier experiences, we can thus also
infer it must play a role in stream-to-table conversion because grouping is
what drives table creation. There are really two aspects of windowing that
interact with stream/table theory:

Window assignment

```
This effectively just means placing a record into one or more windows.
```
Window merging

```
This is the logic that makes dynamic, data-driven types of windows, such
as sessions, possible.
```
The effect of window assignment is quite straightforward. When a record is
conceptually placed into a window, the definition of the window is
essentially combined with the user-assigned key for that record to create an
implicit composite key used at grouping time. Simple.

For completeness, let’s take another look at the original windowing example
from Chapter 3, but from a streams and tables perspective. If you recall, the
code snippet looked something like Example 6-2 (with parsing _not_ elided this
time).

_Example 6-2. Summation pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES)))
.apply(Sum.integersPerKey());

And the original visualization looked like that shown in Figure 6-5.

```
10
```

```
Figure 6-5. Event-time/processing-time view of windowed summation on a batch engine
```
And now, Figure 6-6 shows the streams and tables version.

```
Figure 6-6. Streams and tables view of windowed summation on a batch engine
```
As you might expect, this looks remarkably similar to Figure 6-4, but with
four groupings in the table (corresponding to the four windows occupied by
the data) instead of just one. But as before, we must wait until the end of our
bounded input is reached before emitting results. We look at how to address
this for unbounded data in the next section, but first let’s touch briefly on
merging windows.

**Window merging**

Moving on to merging, we’ll find that the effect of window merging is more
complicated than window assignment, but still straightforward when you
think about the logical operations that would need to happen. When grouping
a stream into windows that can merge, that grouping operation has to take
into account all of the windows that could possibly merge together.
Typically, this is limited to windows whose data all have the same key
(because we’ve already established that windowing modifies grouping to not
be just by key, but also key and window). For this reason, the system doesn’t
really treat the key/window pair as a flat composite key, but rather as a

```
00:00 / 00:00
```
```
00:00 / 00:00
```

hierarchical key, with the user-assigned key as the root, and the window a
child component of that root. When it comes time to actually group data
together, the system first groups by the root of the hierarchy (the key
assigned by the user). After the data have been grouped by key, the system
can then proceed with grouping by window within that key (using the child
components of the hierarchical composite keys). This act of grouping by
window is where window merging happens.

What’s interesting from a streams and tables perspective is how this window
merging changes the mutations that are ultimately applied to a table; that is,
how it modifies the changelog that dictates the contents of the table over
time. With nonmerging windows, each new element being grouped results in
a single mutation to the table (to add that element to the group for the
element’s key+window). With merging windows, the act of grouping a new
element can result in one or more existing windows being merged with the
new window. So, the merging operation must inspect all of the existing
windows for the current key, figure out which windows can merge with this
new window, and then atomically commit deletes for the old unmerged
windows in conjunction with an insert for the new merged window into the
table. This is why systems that support merging windows typically define the
unit of atomicity/parallelization as key, rather than key+window. Otherwise,
it would be impossible (or at least much more expensive) to provide the
strong consistency needed for correctness guarantees. When you begin to
look at it in this level of detail, you can see why it’s so nice to have the
system taking care of the nasty business of dealing with window merges. For
an even closer view of window merging semantics, I refer you to section
2.2.2 of “The Dataflow Model”.

At the end of the day, windowing is really just a minor alteration to the
semantics of grouping, which means it’s a minor alteration to the semantics
of stream-to-table conversion. For window assignment, it’s as simple as
incorporating the window into an implicit composite key used at grouping
time. When window merging becomes involved, that composite key is treated
more like a hierarchical key, allowing the system to handle the nasty business
of grouping by key, figuring out window merges within that key, and then


atomically applying all the necessary mutations to the corresponding table for
us. Hooray for layers of abstraction!

All that said, we still haven’t actually addressed the problem of converting a
table to a stream in a more incremental fashion in the case of unbounded data.
For that, we need to revisit triggers.

## When : Triggers

We learned in Chapter 3 that we use triggers to dictate _when_ the contents of a
window will be materialized (with watermarks providing a useful signal of
input completeness for certain types of triggers). After data have been
grouped together into a window, we use triggers to dictate when that data
should be sent downstream. In streams/tables terminology, we understand
that grouping means stream-to-table conversion. From there, it’s a relatively
small leap to see that triggers are the complement to grouping; in other
words, that “ungrouping” operation we were grasping for earlier. Triggers are
what drive table-to-stream conversion.

In streams/tables terminology, triggers are special procedures applied to a
table that allow for data within that table to be materialized in response to
relevant events. Stated that way, they actually sound suspiciously similar to
classic database triggers. And indeed, the choice of name here was no
coincidence; they are essentially the same thing. When you specify a trigger,
you are in effect writing code that then is evaluated for every row in the state
table as time progresses. When that trigger fires, it takes the corresponding
data that are currently at rest in the table and puts them into motion, yielding
a new stream.

Let’s return to our examples. We’ll begin with the simple per-record trigger
from Chapter 2, which simply emits a new result every time a new record
arrives. The code and event-time/processing-time visualization for that
example is shown in Example 6-3. Figure 6-7 presents the results.

_Example 6-3. Triggering repeatedly with every record_

PCollection<String>> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());


PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(Repeatedly(AfterCount(1))));
.apply(Sum.integersPerKey());

```
Figure 6-7. Streams and tables view of windowed summation on a batch engine
```
As before, new results are materialized every time a new record is
encountered. Rendered in a streams and tables type of view, this diagram
would look like Figure 6-8.

```
Figure 6-8. Streams and tables view of windowed summation with per-record triggering on
a streaming engine
```
An interesting side effect of using per-record triggers is how it somewhat
masks the effect of data being brought to rest, given that they are then
immediately put back into motion again by the trigger. Even so, the aggregate
artifact from the grouping remains at rest in the table, as the ungrouped
stream of values flows away from it.

To get a better sense of the at-rest/in-motion relationship, let’s skip forward
in our triggering examples to the basic watermark completeness streaming
example from Chapter 2, which simply emitted results when complete (due to
the watermark passing the end of the window). The code and event-
time/processing-time visualization for that example are presented in

```
00:00 / 00:00
```
```
00:00 / 00:00
```

Example 6-4 (note that I’m only showing the heuristic watermark version
here, for brevity and ease of comparison) and Figure 6-9 illustrates the
results.

_Example 6-4. Watermark completeness trigger_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark()))
.apply(Sum.integersPerKey());

```
Figure 6-9. Event-time/processing-time view of windowed summation with a heuristic
watermark on a streaming engine
```
Thanks to the trigger specified in Example 6-4, which declares that windows
should be materialized when the watermark passes them, the system is able to
emit results in a progressive fashion as the otherwise unbounded input to the
pipeline becomes more and more complete. Looking at the streams and tables
version in Figure 6-10, it looks as you might expect.

```
Figure 6-10. Streams and tables view of windowed summation with a heuristic watermark
on a streaming engine
```
In this version, you can see very clearly the ungrouping effect triggers have

```
00:00 / 00:00
```
```
00:00 / 00:00
```

on the state table. As the watermark passes the end of each window, it pulls
the result for that window out of the table and sets it in motion downstream,
separate from all the other values in the table. We of course still have the late
data issue from before, which we can solve again with the more
comprehensive trigger shown in Example 6-5.

_Example 6-5. Early, on-time, and late firings via the early/on-time/late API_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))
.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

The event-time/processing-time diagram looks like Figure 6-11.

```
Figure 6-11. Event-time/processing-time view of windowed summation on a streaming
engine with early/on-time/late trigger
```
Whereas the streams and tables version looks like that shown in Figure 6-12.

```
Figure 6-12. Streams and tables view of windowed summation on a streaming engine with
early/on-time/late trigger
```
```
00:00 / 00:00
```
```
00:00 / 00:00
```

This version makes even more clear the ungrouping effect triggers have,
rendering an evolving view of the various independent pieces of the table into
a stream, as dictated by the triggers specified in Example 6-6.

The semantics of all the concrete triggers we’ve talked about so far (event-
time, processing-time, count, composites like early/on-time/late, etc.) are just
as you would expect when viewed from the streams/tables perspective, so
they aren’t worth further discussion. However, we haven’t yet spent much
time talking about what triggers look like in a classic batch processing
scenario. Now that we understand what the underlying streams/tables
topology of a batch pipeline looks like, this is worth touching upon briefly.

At the end of the day, there’s really only one type of trigger used in classic
batch scenarios: one that fires when the input is complete. For the initial
MapRead stage of the MapReduce job we looked at earlier, that trigger would
conceptually fire for all of the data in the input table as soon as the pipeline
launched, given that the input for a batch job is assumed to be complete from
the get go. That input source table would thus be converted into a stream of
individual elements, after which the Map stage could begin processing them.

For table-to-stream conversions in the middle of the pipeline, such as the
ReduceRead stage in our example, the same type of trigger is used. In this
case, however, the trigger must actually wait for all of the data in the table to
be complete (i.e., what is more commonly referred to as all of the data being
written to the shuffle), much as our example batch pipelines in Figures 6-4
and 6-6 waited for the end of the input before emitting their final results.

Given that classic batch processing effectively always makes use of the input-
data-complete trigger, you might ask what any custom triggers specified by
the author of the pipeline might mean in a batch scenario. The answer here
really is: it depends. There are two aspects worth discussing:

Trigger guarantees (or lack thereof)

```
Most existing batch processing systems have been designed with this
lock-step read-process-group-write-repeat sequence in mind. In such
circumstances, it’s difficult to provide any sort of finer-grained trigger
abilities, because the only place they would manifest any sort of change
```
```
11
```

```
would be at the final shuffle stage of the pipeline. This doesn’t mean that
the triggers specified by the user aren’t honored, however; the semantics
of triggers are such that it’s possible to resort to lower common
denominators when appropriate.
For example, an AfterWatermark trigger is meant to trigger after the
watermark passes the end of a window. It makes no guarantees how far
beyond the end of the window the watermark may be when it fires.
Similarly, an AfterCount(N) trigger only guarantees that at least N
elements have been processed before triggering; N might very well be all
of the elements in the input set.
Note that this clever wording of trigger names wasn’t chosen simply to
accommodate classic batch systems within the model; it’s a very
necessary part of the model itself, given the natural asynchronicity and
nondeterminism of triggering. Even in a finely tuned, low-latency, true-
streaming system, it’s essentially impossible to guarantee that an
AfterWatermark trigger will fire while the watermark is precisely at the
end of any given window, except perhaps under the most extremely
limited circumstances (e.g., a single machine processing all of the data for
the pipeline with a relatively modest load). And even if you could
guarantee it, what really would be the point? Triggers provide a means of
controlling the flow of data from a table into a stream, nothing more.
```
The blending of batch and streaming

```
Given what we’ve learned in this writeup, it should be clear that the main
semantic difference between batch and streaming systems is the ability to
trigger tables incrementally. But even that isn’t really a semantic
difference, but more of a latency/throughput trade-off (because batch
systems typically give you higher throughput at the cost of higher latency
of results).
This goes back to something I said in “Batch and Streaming Efficiency
Differences”: there’s really not that much difference between batch and
streaming systems today except for an efficiency delta (in favor of batch)
and a natural ability to deal with unbounded data (in favor of streaming).
```

```
I argued then that much of that efficiency delta comes from the
combination of larger bundle sizes (an explicit compromise of latency in
favor of throughput) and more efficient shuffle implementations (i.e.,
stream → table → stream conversions). From that perspective, it should
be possible to provide a system that seamlessly integrates the best of both
worlds: one which provides the ability to handle unbounded data
naturally but can also balance the tensions between latency, throughput,
and cost across a broad spectrum of use cases by transparently tuning the
bundle sizes, shuffle implementations, and other such implementation
details under the covers.
This is precisely what Apache Beam already does at the API level. The
argument being made here is that there’s room for unification at the
execution-engine level, as well. In a world like that, batch and streaming
will no longer be a thing, and we’ll be able to say goodbye to both batch
and streaming as independent concepts once and for all. We’ll just have
general data processing systems that combine the best ideas from both
branches in the family tree to provide an optimal experience for the
specific use case at hand. Some day.
```
At this point, we can stick a fork in the trigger section. It’s done. We have
only one more brief stop on our way to having a holistic view of the
relationship between the Beam Model and streams-and-tables theory:
_accumulation_.

## How : Accumulation

In Chapter 2, we learned that the three accumulation modes (discarding,
accumulating, accumulating and retracting ) tell us how refinements of
results relate when a window is triggered multiple times over the course of its
life. Fortunately, the relationship to streams and tables here is pretty
straightforward:

```
Discarding mode requires the system to either throw away the
previous value for the window when triggering or keep around a
copy of the previous value and compute the delta the next time the
```
```
12
```
```
13
```
```
14
```

```
window triggers. (This mode might have better been called Delta
mode.)
Accumulating mode requires no additional work; the current value
for the window in the table at triggering time is what is emitted.
(This mode might have better been called Value mode.)
Accumulating and retracting mode requires keeping around copies
of all previously triggered (but not yet retracted) values for the
window. This list of previous values can grow quite large in the case
of merging windows like sessions, but is vital to cleanly reverting
the effects of those previous trigger firings in cases where the new
value cannot simply be used to overwrite a previous value. (This
mode might have better been called Value and Retractions mode.)
```
The streams-and-tables visualizations of accumulation modes add little
additional insight into their semantics, so we won’t investigate them here.

## A Holistic View of Streams and Tables in the

## Beam Model

Having addressed the four questions, we can now take a holistic view of
streams and tables in a Beam Model pipeline. Let’s take our running example
(the team scores calculation pipeline) and see what its structure looks like at
the streams-and-table level. The full code for the pipeline might look
something like Example 6-6 (repeating Example 6-4).

_Example 6-6. Our full score-parsing pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))
.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

```
14
```

Breaking that apart into stages separated by the intermediate PCollection
types (where I’ve used more semantic “type” names like Team and User
Score than real types for clarity of what is happening at each stage), you
would arrive at something like that depicted in Figure 6-13.


_Figure 6-13. Logical phases of a team score summation pipeline, with intermediate
PCollection types_


When you actually run this pipeline, it first goes through an optimizer, whose
job is to convert this logical execution plan into an optimized, physical
execution plan. Each execution engine is different, so actual physical
execution plans will vary between runners. But a believable strawperson plan
might look something like Figure 6-14.


_Figure 6-14. Theoretical physical phases of a team score summation pipeline, with
intermediate PCollection types_


There’s a lot going on here, so let’s walk through all of it. There are three
main differences between Figures 6-13 and 6-14 that we’ll be discussing:

Logical versus physical operations

```
As part of building a physical execution plan, the underlying engine must
convert the logical operations provided by the user into a sequence of
primitive operations supported by the engine. In some cases, those
physical equivalents look essentially the same (e.g., Parse), and in others,
they’re very different.
```
Physical stages and fusion

```
It’s often inefficient to execute each logical phase as a fully independent
physical stage in the pipeline (with attendant serialization, network
communication, and deserialization overhead between each). As a result,
the optimizer will typically try to fuse as many physical operations as
possible into a single physical stage.
```
Keys, values, windows, and partitioning

```
To make it more evident what each physical operation is doing, I’ve
annotated the intermediate PCollections with the type of key, value,
window, and data partitioning in effect at each point.
```
Let’s now walk through each logical operation in detail and see what it
translated to in the physical plan and how they all relate to streams and tables:

ReadFromSource

```
Other than being fused with the physical operation immediately following
it (Parse), not much interesting happens in translation for
ReadFromSource. As far as the characteristics of our data at this point,
because the read is essentially consuming raw input bytes, we basically
have raw strings with no keys, no windows, and no (or random)
partitioning. The original data source can be either a table (e.g., a
Cassandra table) or a stream (e.g., RabbitMQ) or something a little like
both (e.g., Kafka in log compaction mode). But regardless, the end result
of reading from the input source is a stream.
```

Parse

```
The logical Parse operation also translates in a relatively straightforward
manner to the physical version. Parse takes the raw strings and extracts a
key (team ID) and value (user score) from them. It’s a nongrouping
operation, and thus the stream it consumed remains a stream on the other
side.
```
Window+Trigger

```
This logical operation is spread out across a number of distinct physical
operations. The first is window assignment, in which each element is
assigned to a set of windows. That happens immediately in the
AssignWindows operation, which is a nongrouping operation that simply
annotates each element in the stream with the window(s) it now belongs
to, yielding another stream on the other side.
The second is window merging, which we learned earlier in the chapter
happens as part of the grouping operation. As such, it gets sunk down into
the GroupMergeAndCombine operation later in the pipeline. We discuss
that operation when we talk about the logical Sum operation next.
And finally, there’s triggering. Triggering happens after grouping and is
the way that we’ll convert the table created by grouping back into a
stream. As such, it gets sunk into its own operation, which follows
GroupMergeAndCombine.
```
Sum

```
Summation is really a composite operation, consisting of a couple pieces:
partitioning and aggregation. Partitioning is a nongrouping operation that
redirects the elements in the stream in such a way that elements with the
same keys end up going to the same physical machine. Another word for
partitioning is shuffling, though that term is a bit overloaded because
“Shuffle” in the MapReduce sense is often used to mean both partitioning
and grouping ( and sorting, for that matter). Regardless, partitioning
physically alters the stream in way that makes it groupable but doesn’t do
```

```
anything to actually bring the data to rest. As a result, it’s a nongrouping
operation that yields another stream on the other side.
After partitioning comes grouping. Grouping itself is a composite
operation. First comes grouping by key (enabled by the previous
partition-by-key operation). Next comes window merging and grouping
by window, as we described earlier. And finally, because summation is
implemented as a CombineFn in Beam (essentially an incremental
aggregation operation), there’s combining, where individual elements are
summed together as they arrive. The specific details are not terribly
important for our purposes here. What is important is the fact that, since
this is (obviously) a grouping operation, our chain of streams now comes
to rest in a table containing the summed team totals as they evolve over
time.
```
WriteToSink

```
Lastly, we have the write operation, which takes the stream yielded by
triggering (which was sunk below the GroupMergeAndCombine operation,
as you might recall) and writes it out to our output data sink. That data
itself can be either a table or stream. If it’s a table, WriteToSink will
need to perform some sort of grouping operation as part of writing the
data into the table. If it’s a stream, no grouping will be necessary (though
partitioning might still be desired; for example, when writing into
something like Kafka).
```
The big takeaway here is not so much the precise details of everything that’s
going on in the physical plan, but more the overall relationship of the Beam
Model to the world of streams and tables. We saw three types of operations:
nongrouping (e.g., Parse), grouping (e.g., GroupMergeAndCombine), and
ungrouping (e.g., Trigger). The nongrouping operations always consumed
streams and produced streams on the other side. The grouping operations
always consumed streams and yielded tables. And the ungrouping operations
consumed tables and yielded streams. These insights, along with everything
else we’ve learned along the way, are enough for us to formulate a more
general theory about the relationship of the Beam Model to streams and


tables.

