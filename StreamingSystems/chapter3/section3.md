 ## Watermark Propagation

So far, we have considered only the watermark for the inputs within the
context of a single operation or stage. However, most real-world pipelines
consist of multiple stages. Understanding how watermarks propagate across
independent stages is important in understanding how they affect the pipeline
as a whole and the observed latency of its results.

### PIPELINE STAGES

```
Different stages are typically necessary every time your pipeline groups
data together by some new dimension. For example, if you had a pipeline
that consumed raw data, computed some per-user aggregates, and then
used those per-user aggregates to compute some per-team aggregates,
you’d likely end up with a three-stage pipeline:
```
```
One consuming the raw, ungrouped data
```
```
One grouping the data by user and computing per-user
aggregates
One grouping the data by team and computing per-team
aggregates
```
```
We learn more about the effects of grouping on pipeline shapes in
Chapter 6.
```
Watermarks are created at input sources, as discussed in the preceding
section. They then conceptually flow through the system as data progress
through it. You can track watermarks at varying levels of granularity. For
pipelines comprising multiple distinct stages, each stage likely tracks its own
watermark, whose value is a function of all the inputs and stages that come
before it. Therefore, stages that come later in the pipeline will have
watermarks that are further in the past (because they’ve seen less of the
overall input).

We can define watermarks at the boundaries of any single operation, or stage,

```
3
```

in the pipeline. This is useful not only in understanding the relative progress
that each stage in the pipeline is making, but for dispatching timely results
independently and as soon as possible for each individual stage. We give the
following definitions for the watermarks at the boundaries of stages:

```
An input watermark , which captures the progress of everything
upstream of that stage (i.e., how complete the input is for that stage).
For sources, the input watermark is a source-specific function
creating the watermark for the input data. For nonsource stages, the
input watermark is defined as the minimum of the output
watermarks of all shards/partitions/instances of all of its upstream
sources and stages.
An output watermark , which captures the progress of the stage itself,
and is essentially defined as the minimum of the stage’s input
watermark and the event times of all nonlate data active messages
within the stage. Exactly what “active” encompasses is somewhat
dependent upon the operations a given stage actually performs, and
the implementation of the stream processing system. It typically
includes data buffered for aggregation but not yet materialized
downstream, pending output data in flight to downstream stages, and
so on.
```
One nice feature of defining an input and output watermark for a specific
stage is that we can use these to calculate the amount of event-time latency
introduced by a stage. Subtracting the value of a stage’s output watermark
from the value of its input watermark gives the amount of event-time latency
or _lag_ introduced by the stage. This lag is the notion of how far delayed
behind real time the output of each stage will be. As an example, a stage
performing 10-second windowed aggregations will have a lag of 10 seconds
or more, meaning that the output of the stage will be at least that much
delayed behind the input and real time. Definitions of input and output
watermarks provide a recursive relationship of watermarks throughout a
pipeline. Each subsequent stage in a pipeline delays the watermark as
necessary, based on event-time lag of the stage.


Processing within each stage is also not monolithic. We can segment the
processing within one stage into a flow with several conceptual components,
each of which contributes to the output watermark. As mentioned previously,
the exact nature of these components depends on the operations the stage
performs and the implementation of the system. Conceptually, each such
component serves as a buffer where active messages can reside until some
operation has completed. For example, as data arrives, it is buffered for
processing. Processing might then write the data to state for later delayed
aggregation. Delayed aggregation, when triggered, might write the results to
an output buffer awaiting consumption from a downstream stage, as shown in
Figure 3-3.

```
Figure 3-3. Example system components of a streaming system stage, containing buffers of
in-flight data. Each will have associated watermark tracking, and the overall output
watermark of the stage will be the minimum of the watermarks across all such buffers.
```
We can track each such buffer with its own watermark. The minimum of the
watermarks across the buffers of each stage forms the output watermark of
the stage. Thus the output watermark could be the minimum of the following:

```
Per-source watermark—for each sending stage.
Per-external input watermark—for sources external to the pipeline
```
```
Per-state component watermark—for each type of state that can be
written
```

```
Per-output buffer watermark—for each receiving stage
```
Making watermarks available at this level of granularity also provides better
visibility into the behavior of the system. The watermarks track locations of
messages across various buffers in the system, allowing for easier diagnosis
of stuckness.

## Understanding Watermark Propagation

To get a better sense for the relationship between input and output
watermarks and how they affect watermark propagation, let’s look at an
example. Let’s consider gaming scores, but instead of computing sums of
team scores, we’re going to take a stab at measuring user engagement levels.
We’ll do this by first calculating per-user session lengths, under the
assumption that the amount of time a user stays engaged with the game is a
reasonable proxy for how much they’re enjoying it. After answering our four
questions once to calculate sessions lengths, we’ll then answer them a second
time to calculate average session lengths within fixed periods of time.

To make our example even more interesting, lets say that we are working
with two datasets, one for Mobile Scores and one for Console Scores. We
would like to perform identical score calculations via integer summation in
parallel over these two independant datasets. One pipeline is calculating
scores for users playing on mobile devices, whereas the other is for users
playing on home gaming consoles, perhaps due to different data collection
strategies employed for the different platforms. The important point is that
these two stages are performing the same operation but over different data,
and thus with very different output watermarks.

To begin, let’s take a look at Example 3-1 to see what the abbreviated code
for what the first section of this pipeline might be like.

_Example 3-1. Calculating session lengths_

PCollection<Double> mobileSessions = IO.read(new MobileInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())


.discardingFiredPanes())
.apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())
.discardingFiredPanes())
.apply(CalculateWindowLength());

Here, we read in each of our inputs independently, and whereas previously
we were keying our collections by team, in this example we key by user.
After that, for the first stage of each pipeline, we window into sessions and
then call a custom PTransform named CalculateWindowLength. This
PTransform simply groups by key (i.e., User) and then computes the per-
user session length by treating the size of the current window as the value for
that window. In this case, we’re fine with the default trigger (AtWatermark)
and accumulation mode (discardingFiredPanes) settings, but I’ve listed
them explicitly for completeness. The output for each pipeline for two
particular users might look something like Figure 3-4.

```
Figure 3-4. Per-user session lengths across two different input pipelines
```
Because we need to track data across multiple stages, we track everything
related to Mobile Scores in red, everything related to Console Scores in blue,
while the watermark and output for Average Session Lengths in Figure 3-5
are yellow.

We have answered the four questions of _what_ , _where_ , _when_ , and _how_ to
compute individual session lengths. Next we’ll answer them a second time to
transform those session lengths into global session-length averages within

```
00:00 / 00:00
```

fixed windows of time. This requires us to first flatten our two data sources
into one, and then re-window into fixed windows; we’ve already captured the
important essence of the session in the session-length value we computed,
and we now want to compute a global average of those sessions within
consistent windows of time over the course of the day. Example 3-2 shows
the code for this.

_Example 3-2. Calculating session lengths_

PCollection<Double> mobileSessions = IO.read(new MobileInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())
.discardingFiredPanes())
.apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())
.discardingFiredPanes())
.apply(CalculateWindowLength());

PCollection<Float> averageSessionLengths = PCollectionList
.of(mobileSessions).and(consoleSessions)
.apply(Flatten.pCollections())
.apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
.triggering(AtWatermark())
.apply(Mean.globally());

If we were to see this pipeline in action, it would look something like
Figure 3-5. As before, the two input pipelines are computing individual
session lengths for mobile and console players. Those session lengths then
feed into the second stage of the pipeline, where global session-length
averages are computed in fixed windows.


```
Figure 3-5. Average session lengths of mobile and console gaming sessions
```
Let’s walk through some of this example, given that there’s a lot going on.
The two important points here are:

```
The output watermark for each of the Mobile Sessions and Console
Sessions stages is at least as old as the corresponding input
watermark of each, and in reality a little bit older. This is because in
a real system computing answers takes time, and we don’t allow the
output watermark to advance until processing for a given input has
completed.
The input watermark for the Average Session Lengths stage is the
minimum of the output watermarks for the two stages directly
upstream.
```
The result is that the downstream input watermark is an alias for the
minimum composition of the upstream output watermarks. Note that this
matches the definitions for those two types of watermarks earlier in the
chapter. Also notice how watermarks further downstream are further in the
past, capturing the intuitive notion that upstream stages are going to be
further ahead in time than the stages that follow them.

One observation worth making here is just how cleanly we were able to ask
the questions again in Example 3-1 to substantially alter the results of the
pipeline. Whereas before we simply computed per-user session lengths, we
now compute two-minute global session-length averages. This provides a
much more insightful look into the overall behaviors of the users playing our
games and gives you a tiny glimpse of the difference between simple data
transformations and real data science.

```
00:00 / 00:00
```

Even better, now that we understand the basics of how this pipeline operates,
we can look more closely at one of the more subtle issues related to asking
the four questions over again: _output timestamps_.

## Watermark Propagation and Output Timestamps

In Figure 3-5, I glossed over some of the details of output timestamps. But if
you look closely at the second stage in the diagram, you can see that each of
the outputs from the first stage was assigned a timestamp that matched the
end of its window. Although that’s a fairly natural choice for output
timestamps, it’s not the only valid choice. As you know from earlier in this
chapter, watermarks are never allowed to move backward. Given that
restriction, you can infer that the range of valid timestamps for a given
window begins with the timestamp of the earliest nonlate record in the
window (because only nonlate records are guaranteed to hold a watermark
up) and extends all the way to positive infinity. That’s quite a lot of options.
In practice, however, there tend to be only a few choices that make sense in
most circumstances:

End of the window

```
Using the end of the window is the only safe choice if you want the
output timestamp to be representative of the window bounds. As we’ll see
in a moment, it also allows the smoothest watermark progression out of
all of the options.
```
Timestamp of first nonlate element

```
Using the timestamp of the first nonlate element is a good choice when
you want to keep your watermarks as conservative as possible. The trade-
off, however, is that watermark progress will likely be more hindered, as
we’ll also see shortly.
```
Timestamp of a specific element

```
For certain use cases, the timestamp of some other arbitrary (from the
system’s perspective) element is the right choice. Imagine a use case in
which you’re joining a stream of queries to a stream of clicks on results
```
```
4
```

```
for that query. After performing the join, some systems will find the
timestamp of the query to be more useful; others will prefer the
timestamp of the click. Any such timestamp is valid from a watermark
correctness perspective, as long as it corresponded to an element that did
not arrive late.
```
Having thought a bit about some alternate options for output timestamps,
let’s look at what effects the choice of output timestamp can have on the
overall pipeline. To make the changes as dramatic as possible, in Example 3-
3 and Figure 3-6, we’ll switch to using the earliest timestamp possible for the
window: the timestamp of the first nonlate element as the timestamp for the
window.

_Example 3-3. Average session lengths pipeline, that output timestamps for
session windows set at earliest element_

PCollection<Double> mobileSessions = IO.read(new MobileInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())
.withTimestampCombiner(EARLIEST)
.discardingFiredPanes())
.apply(CalculateWindowLength());

PCollection<Double> consoleSessions = IO.read(new ConsoleInputSource())

.apply(Window.into(Sessions.withGapDuration(Duration.standardMinutes(1)))
.triggering(AtWatermark())
.withTimestampCombiner(EARLIEST)
.discardingFiredPanes())
.apply(CalculateWindowLength());

PCollection<Float> averageSessionLengths = PCollectionList
.of(mobileSessions).and(consoleSessions)
.apply(Flatten.pCollections())
.apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
.triggering(AtWatermark())
.apply(Mean.globally());


```
Figure 3-6. Average session lengths for sessions that are output at the timestamp of the
earliest element
```
To help call out the effect of the output timestamp choice, look at the dashed
lines in the first stages showing what the output watermark for each stage is
being held to. The output watermark is delayed by our choice of timestamp,
as compared to Figures 3-7 and 3-8, in which the output timestamp was
chosen to be the end of the window. You can see from this diagram that the
input watermark of the second stage is thus subsequently also delayed.

```
Figure 3-7. Comparison of watermarks and results with different choice of window outout
```
```
00:00 / 00:00
```

```
timestamps. The watermarks in this figure correspond to output timestamps at the end of
the session windows (i.e., Figure 3-5).
```
```
Figure 3-8. In this figure, the watermarks are at the beginning of the session windows (i.e.,
Figure 3-6). We can see that the watermark line in this figure is more delayed, and the
resulting average session lengths are different.
```
As far as differences in this version compared to Figure 3-7, two are worth
noting:

Watermark delay

```
Compared to Figure 3-5, the watermark proceeds much more slowly in
Figure 3-6. This is because the output watermark for the first stage is held
back to the timestamp of the first element in every window until the input
for that window becomes complete. Only after a given window has been
materialized is the output watermark (and thus the downstream input
watermark) allowed to advance.
```
Semantic differences


```
Because the session timestamps are now assigned to match the earliest
nonlate element in the session, the individual sessions often end up in
different fixed window buckets when we then calculate the session-length
averages in the next stage. There’s nothing inherently right or wrong
about either of the two options we’ve seen so far; they’re just different.
But it’s important to understand that they will be different as well as have
an intuition for the way in which they’ll be different so that you can make
the correct choice for your specific use case when the time comes.
```
## The Tricky Case of Overlapping Windows

One additional subtle but important issue regarding output timestamps is how
to handle sliding windows. The naive approach of setting the output
timestamp to the earliest element can very easily lead to delays downstream
due to watermarks being (correctly) held back. To see why, consider an
example pipeline with two stages, each using the same type of sliding
windows. Suppose that each element ends up in three successive windows.
As the input watermark advances, the desired semantics for sliding windows
in this case would be as follows:

```
The first window completes in the first stage and is emitted
downstream.
The first window then completes in the second stage and can also be
emitted downstream.
```
```
Some time later, the second window completes in the first stage...
and so on.
```
However, if output timestamps are chosen to be the timestamp of the first
nonlate element in the pane, what actually happens is the following:

```
The first window completes in the first stage and is emitted
downstream.
The first window in the second stage remains unable to complete
```

```
because its input watermark is being held up by the output
watermark of the second and third windows upstream. Those
watermarks are rightly being held back because the earliest element
timestamp is being used as the output timestamp for those windows.
```
```
The second window completes in the first stage and is emitted
downstream.
The first and second windows in the second stage remain unable to
complete, held up by the third window upstream.
The third window completes in the first stage and is emitted
downstream.
```
```
The first, second, and third windows in the second stage are now all
able to complete, finally emitting all three in one swoop.
```
Although the results of this windowing are correct, this leads to the results
being materialized in an unnecessarily delayed way. Because of this, Beam
has special logic for overlapping windows that ensures the output timestamp
for window _N_ +1 is always greater than the end of window _N_.


