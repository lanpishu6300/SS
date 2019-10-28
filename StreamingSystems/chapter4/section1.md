 ## When / Where : Processing-Time Windows

Processing-time windowing is important for two reasons:

```
For certain use cases, such as usage monitoring (e.g., web service
traffic QPS), for which you want to analyze an incoming stream of
data as it’s observed, processing-time windowing is absolutely the
appropriate approach to take.
For use cases for which the time that events happened is important
(e.g., analyzing user behavior trends, billing, scoring, etc.),
processing-time windowing is absolutely the wrong approach to
take, and being able to recognize these cases is critical.
```
As such, it’s worth gaining a solid understanding of the differences between


processing-time windowing and event-time windowing, particularly given the
prevalence of processing-time windowing in many streaming systems today.

When working within a model for which windowing as a first-class notion is
strictly event-time based, such as the one presented in this book, there are two
methods that you can use to achieve processing-time windowing:

Triggers

```
Ignore event time (i.e., use a global window spanning all of event time)
and use triggers to provide snapshots of that window in the processing-
time axis.
```
Ingress time

```
Assign ingress times as the event times for data as they arrive, and use
normal event-time windowing from there on. This is essentially what
something like Spark Streaming 1.x does.
```
Note that the two methods are more or less equivalent, although they differ
slightly in the case of multistage pipelines: in the triggers version, a
multistage pipeline will slice the processing-time “windows” independently
at each stage, so, for example, data in window _N_ for one stage might instead
end up in window _N_ –1 or _N_ +1 in the following stage; in the ingress-time
version, after a datum is incorporated into window _N_ , it will remain in
window _N_ for the duration of the pipeline due to synchronization of progress
between stages via watermarks (in the Cloud Dataflow case), microbatch
boundaries (in the Spark Streaming case), or whatever other coordinating
factor is involved at the engine level.

As I’ve noted to death, the big downside of processing-time windowing is
that the contents of the windows change when the observation order of the
inputs changes. To drive this point home in a more concrete manner, we’re
going to look at these three use cases: _event-time_ windowing, _processing-time_
windowing via triggers, and _processing-time_ windowing via ingress time.

Each will be applied to two different input sets (so six variations total). The
two inputs sets will be for the exact same events (i.e., same values, occurring
at the same event times), but with different observation orders. The first set


will be the observation order we’ve seen all along, colored white; the second
one will have all the values shifted in the processing-time axis as in Figure 4-
1 , colored purple. You can simply imagine that the purple example is another
way reality could have happened if the winds had been blowing in from the
east instead of the west (i.e., the underlying set of complex distributed
systems had played things out in a slightly different order).

```
Figure 4-1. Shifting input observation order in processing time, holding values, and event-
times constant
```
## Event-Time Windowing

To establish a baseline, let’s first compare fixed windowing in event time
with a heuristic watermark over these two observation orderings. We’ll reuse
the early/late code from Example 2-7/Figure 2-10 to get the results shown in
Figure 4-2. The lefthand side is essentially what we saw before; the righthand
side is the results over the second observation order. The important thing to
note here is that even though the overall shape of the outputs differs (due to
the different orders of observation in processing time), _the final results for the
four windows remain the same_ : 14, 18, 3, and 12.

```
00:00 / 00:00
```

_Figure 4-2. Event-time windowing over two different processing-time orderings of the same
inputs_

## Processing-Time Windowing via Triggers

Let’s now compare this to the two processing-time methods just described.
First, we’ll try the triggers method. There are three aspects to making

```
00:00 / 00:00
```

processing-time “windowing” work in this manner:

Windowing

```
We use the global event-time window because we’re essentially
emulating processing-time windows with event-time panes.
```
Triggering

```
We trigger periodically in the processing-time domain based on the
desired size of the processing-time windows.
```
Accumulation

```
We use discarding mode to keep the panes independent from one another,
thus letting each of them act like an independent processing-time
“window.”
```
The corresponding code looks something like Example 4-1; note that global
windowing is the default in Beam, hence there is no specific override of the
windowing strategy.

_Example 4-1. Processing-time windowing via repeated, discarding panes of a
global event-time window_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.triggering(Repeatedly(AlignedDelay(ONE_MINUTE)))
.discardingFiredPanes())
.apply(Sum.integersPerKey());

When executed on a streaming runner against our two different orderings of
the input data, the results look like Figure 4-3. Here are some interesting
notes about this figure:

```
Because we’re emulating processing-time windows via event-time
panes, the “windows” are delineated in the processing-time axis,
which means their effective width is measured on the y-axis instead
of the x-axis.
```
```
Because processing-time windowing is sensitive to the order that
input data are encountered, the results for each of the “windows”
differs for each of the two observation orders, even though the
```

```
events themselves technically happened at the same times in each
version. On the left we get 12, 18, 18, whereas on the right we get 7,
36, 5.
```
```
Figure 4-3. Processing-time “windowing” via triggers, over two different processing-time
orderings of the same inputs
```
## Processing-Time Windowing via Ingress Time

Lastly, let’s look at processing-time windowing achieved by mapping the
event times of input data to be their ingress times. Code-wise, there are four
aspects worth mentioning here:

Time-shifting

```
When elements arrive, their event times need to be overwritten with the
time of ingress. We can do this in Beam by providing a new DoFn that
sets the timestamp of the element to the current time via the
outputWithTimestamp method.
```
Windowing

```
Return to using standard event-time fixed windowing.
```
Triggering

```
Because ingress time affords the ability to calculate a perfect watermark,
we can use the default trigger, which in this case implicitly fires exactly
once when the watermark passes the end of the window.
```
Accumulation mode

```
Because we only ever have one output per window, the accumulation
```
```
00:00 / 00:00
```

```
mode is irrelevant.
```
The actual code might thus look something like that in Example 4-2.

_Example 4-2. Processing-time windowing via repeated, discarding panes of a
global event-time window_

PCollection<String> raw = IO.read().apply(ParDo.of(
new DoFn<String, String>() {
public void processElement(ProcessContext c) {
c.outputWithTimestmap(new Instant());
}
});
PCollection<KV<Team, Integer>> input =
raw.apply(ParDo.of(new ParseFn());
PCollection<KV<Team, Integer>> totals = input
.apply(Window.info(FixedWindows.of(TWO_MINUTES))
.apply(Sum.integersPerKey());

Execution on a streaming engine would look like Figure 4-4. As data arrive,
their event times are updated to match their ingress times (i.e., the processing
times at arrival), resulting in a rightward horizontal shift onto the ideal
watermark line. Here are some interesting notes about this figure:

```
As with the other processing-time windowing example, we get
different results when the ordering of inputs changes, even though
the values and event times for the input stay constant.
```
```
Unlike the other example, the windows are once again delineated in
the event-time domain (and thus along the x-axis). Despite this, they
aren’t bonafide event-time windows; we’ve simply mapped
processing time onto the event-time domain, erasing the original
record of occurrence for each input and replacing it with a new one
that instead represents the time the datum was first observed by the
pipeline.
```
```
Despite this, thanks to the watermark, trigger firings still happen at
exactly the same time as in the previous processing-time example.
Furthermore, the output values produced are identical to that
example, as predicted: 12, 18, 18 on the left, and 7, 36, 5 on the
```

```
right.
```
```
Because perfect watermarks are possible when using ingress time,
the actual watermark matches the ideal watermark, ascending up and
to the right with a slope of one.
```
```
Figure 4-4. Processing-time windowing via the use of ingress time, over two different
processing-time orderings of the same inputs
```
Although it’s interesting to see the different ways you can implement
processing-time windowing, the big takeaway here is the one I’ve been
harping on since the first chapter: event-time windowing is order-agnostic, at
least in the limit (actual panes along the way might differ until the input
becomes complete); processing-time windowing is not. _If you care about the
times at which your events actually happened, you must use event-time
windowing or your results will be meaningless._ I will get off my soapbox
now.


