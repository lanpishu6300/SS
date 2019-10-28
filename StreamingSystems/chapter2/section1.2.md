 ## Batch Foundations: What and Where

Okay, let’s get this party started. First stop: batch processing.

## What : Transformations

The transformations applied in classic batch processing answer the question:
“ _What_ results are calculated?” Even though you are likely already familiar
with classic batch processing, we’re going to start there anyway because it’s
the foundation on top of which we add all of the other concepts.

In the rest of this chapter (and indeed, through much of the book), we look at
a single example: computing keyed integer sums over a simple dataset
consisting of nine values. Let’s imagine that we’ve written a team-based
mobile game and we want to build a pipeline that calculates team scores by
summing up the individual scores reported by users’ phones. If we were to
capture our nine example scores in a SQL table named “UserScores,” it might
look something like this:

```
------------------------------------------------
| Name | Team | Score | EventTime | ProcTime |
------------------------------------------------
| Julie | TeamX | 5   | 12:00:26 | 12:05:19 |
| Frank | TeamX | 9   | 12:01:26 | 12:08:19 |
| Ed    | TeamX | 7   | 12:02:26 | 12:05:39 |
| Julie | TeamX | 8   | 12:03:06 | 12:07:06 |
| Amy   | TeamX | 3   | 12:03:39 | 12:06:13 |
```
```
2
```

```
| Fred | TeamX | 4 | 12:04:19 | 12:06:39 |
| Naomi | TeamX | 3 | 12:06:39 | 12:07:19 |
| Becky | TeamX | 8 | 12:07:26 | 12:08:39 |
| Naomi | TeamX | 1 | 12:07:46 | 12:09:00 |
------------------------------------------------
```
Note that all the scores in this example are from users on the same team; this
is to keep the example simple, given that we have a limited number of
dimensions in our diagrams that follow. And because we’re grouping by
team, we really just care about the last three columns:

Score

```
The individual user score associated with this event
```
EventTime

```
The event time for the score; that is, the time at which the score occurred
```
ProcTime

```
The processing for the score; that is, the time at which the score was
observed by the pipeline
```
For each example pipeline, we’ll look at a time-lapse diagram that highlights
how the data evolves over time. Those diagrams plot our nine scores in the
two dimensions of time we care about: event time in the x-axis, and
processing time in the y-axis. Figure 2-1 illustrates what a static plot of the
input data looks like.


```
Figure 2-1. Nine input records, plotted in both event time and processing time
```
Subsequent time-lapse diagrams are either animations (Safari) or a sequence
of frames (print and all other digital formats), allowing you to see how the
data are processed over time (more on this shortly after we get to the first
time-lapse diagram).

Preceding each example is a short snippet of Apache Beam Java SDK
pseudocode to make the definition of the pipeline more concrete. It is
pseudocode in the sense that I sometime bend the rules to make the examples
clearer, elide details (like the use of concrete I/O sources), or simplify names
(the trigger names in Beam Java 2.x and earlier are painfully verbose; I use
simpler names for clarity). Beyond minor things like those, it’s otherwise
real-world Beam code (and real code is available on GitHub for all examples
in this chapter).

If you’re already familiar with something like Spark or Flink, you should
have a relatively easy time understanding what the Beam code is doing. But
to give you a crash course in things, there are two basic primitives in Beam:

PCollections

```
These represent datasets (possibly massive ones) across which parallel
```

```
transformations can be performed (hence the “P” at the beginning of the
name).
```
PTransforms

```
These are applied to PCollections to create new PCollections.
PTransforms may perform element-wise transformations, they may
group/aggregate multiple elements together, or they may be a composite
combination of other PTransforms, as depicted in Figure 2-2.
```
```
Figure 2-2. Types of transformations
```
For the purposes of our examples, we typically assume that we start out with
a pre-loaded PCollection<KV<Team, Integer>> named “input” (that is, a

PCollection composed of key/value pairs of Teams and Integers, where
the Teams are just something like Strings representing team names, and the
Integers are scores from any individual on the corresponding team). In a
real-world pipeline, we would’ve acquired input by reading in a
PCollection<String> of raw data (e.g., log records) from an I/O source and
then transforming it into a PCollection<KV<Team, Integer>> by parsing
the log records into appropriate key/value pairs. For the sake of clarity in this
first example, I include pseudocode for all of those steps, but in subsequent
examples, I elide the I/O and parsing.

Thus, for a pipeline that simply reads in data from an I/O source, parses
team/score pairs, and calculates per-team sums of scores, we’d have
something like that shown in Example 2-1.


_Example 2-1. Summation pipeline_

PCollection<String> raw = IO.read(...);
PCollection<KV<Team, Integer>> input = raw.apply(new ParseFn());
PCollection<KV<Team, Integer>> totals =
input.apply(Sum.integersPerKey());

Key/value data are read from an I/O source, with a Team (e.g., String of the

team name) as the key and an Integer (e.g., individual team member scores)
as the value. The values for each key are then summed together to generate
per-key sums (e.g., total team score) in the output collection.

For all the examples to come, after seeing a code snippet describing the
pipeline that we’re analyzing, we’ll then look at a time-lapse diagram
showing the execution of that pipeline over our concrete dataset for a single
key. In a real pipeline, you can imagine that similar operations would be
happening in parallel across multiple machines, but for the sake of our
examples, it will be clearer to keep things simple.

As noted previously, Safari editions present the complete execution as an
animated movie, whereas print and all other digital formats use a static
sequence of key frames that provide a sense of how the pipeline progresses
over time. In both cases, we also provide a URL to a fully animated version
on _[http://www.streamingbook.net](http://www.streamingbook.net)_.

Each diagram plots the inputs and outputs across two dimensions: event time
(on the x-axis) and processing time (on the y-axis). Thus, real time as
observed by the pipeline progresses from bottom to top, as indicated by the
thick horizontal black line that ascends in the processing-time axis as time
progresses. Inputs are circles, with the number inside the circle representing
the value of that specific record. They start out light gray, and darken as the
pipeline observes them.

As the pipeline observes values, it accumulates them in its intermediate state
and eventually materializes the aggregate results as output. State and output
are represented by rectangles (gray for state, blue for output), with the
aggregate value near the top, and with the area covered by the rectangle
representing the portions of event time and processing time accumulated into


the result. For the pipeline in Example 2-1, it would look something like that
shown in Figure 2-3 when executed on a classic batch engine.

```
Figure 2-3. Classic batch processing
```
Because this is a batch pipeline, it accumulates state until it’s seen all of the
inputs (represented by the dashed green line at the top), at which point it
produces its single output of 48. In this example, we’re calculating a sum
over all of event time because we haven’t applied any specific windowing
transformations; hence the rectangles for state and output cover the entirety
of the x-axis. If we want to process an unbounded data source, however,
classic batch processing won’t be sufficient; we can’t wait for the input to
end, because it effectively never will. One of the concepts we want is
windowing, which we introduced in Chapter 1. Thus, within the context of
our second question—“ _Where_ in event time are results calculated?”—we’ll
now briefly revisit windowing.

## Where : Windowing

As discussed in Chapter 1, windowing is the process of slicing up a data

```
00:00 / 00:00
```

source along temporal boundaries. Common windowing strategies include
fixed windows, sliding windows, and sessions windows, as demonstrated in
Figure 2-4.

```
Figure 2-4. Example windowing strategies. Each example is shown for three different keys,
highlighting the difference between aligned windows (which apply across all the data) and
unaligned windows (which apply across a subset of the data).
```
To get a better sense of what windowing looks like in practice, let’s take our
integer summation pipeline and window it into fixed, two-minute windows.

With Beam, the change is a simple addition of a Window.into transform,
which you can see highlighted in Example 2-2.

_Example 2-2. Windowed summation code_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES)))
.apply(Sum.integersPerKey());

Recall that Beam provides a unified model that works in both batch and
streaming because semantically batch is really just a subset of streaming. As
such, let’s first execute this pipeline on a batch engine; the mechanics are
more straightforward, and it will give us something to directly compare
against when we switch to a streaming engine. Figure 2-5 presents the result.


```
Figure 2-5. Windowed summation on a batch engine
```
As before, inputs are accumulated in state until they are entirely consumed,
after which output is produced. In this case, however, instead of one output,
we get four: a single output, for each of the four relevant two-minute event-
time windows.

At this point we’ve revisited the two main concepts that I introduced in
Chapter 1: the relationship between the event-time and processing-time
domains, and windowing. If we want to go any further, we’ll need to start
adding the new concepts mentioned at the beginning of this section: triggers,
watermarks, and accumulation.

