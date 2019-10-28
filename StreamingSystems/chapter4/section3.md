 ## Where : Custom Windowing

Up until now, we’ve talked primarily about predefined types of windowing
strategies: fixed, sliding, and sessions. You can get a lot of mileage out of
standard types of windows, but there are plenty of real-world use cases for
which being able to define a custom windowing strategy can really save the
day (three of which we’re about to see now).

Most systems today don’t support custom windowing to the degree that it’s
supported in Beam, so we focus on the Beam approach. In Beam, a custom
windowing strategy consists of two things:

Window assignment

```
1
```

```
This places each element into an initial window. At the limit, this allows
every element to be placed within a unique window, which is very
powerful.
```
(Optional) window merging

```
This allows windows to merge at grouping times, which makes it possible
for windows to evolve over time, which we saw in action earlier with
session windows.
```
To give you a sense for how simple windowing strategies really are, and also
how useful custom windows support can be, we’re going to look in detail at
the stock implementations of fixed windows and sessions in Beam and then
consider a few real-world use cases that require custom variations on those
themes. In the process, we’ll see both how easy it is to create a custom
windowing strategy, and how limiting the lack of custom windowing support
can be when your use case doesn’t quite fit into the stock approaches.

## Variations on Fixed Windows

To begin, let’s look at the relatively simple strategy of fixed windows. The
stock fixed-windows implementation is as straightforward as you might
imagine, and consists of the following logic:

Assignment

```
The element is placed into the appropriate fixed-window based on its
timestamp and the window’s size and offset parameters.
```
Merging

```
None.
```
An abbreviated version of the code looks like Example 4-4.

_Example 4-4. Abbreviated FixedWindows implementation_

public class FixedWindows extends WindowFn<Object, IntervalWindow> {
private final Duration size;
private final Duration offset;
public Collection<IntervalWindow> assignWindow(AssignContext c) {


long start = c.timestamp().getMillis() - c.timestamp()
.plus(size)
.minus(offset)
.getMillis() % size.getMillis();
return Arrays.asList(IntervalWindow(new Instant(start), size));
}
}

Keep in mind that the point of showing you the code here isn’t so much to
teach you how to write windowing strategies (although it’s nice to demystify
them and call out how simple they are). It’s really to help contrast the
comparative ease and difficulty of supporting some relatively basic use cases,
both with and without custom windowing, respectively. Let’s consider two
such use cases that are variations on the fixed-windows theme now.

**Unaligned fixed windows**

One characteristic of the default fixed-windows implementation that we
alluded to previously is that windows are aligned across all of the data. In our
running example, the window from noon to 1 PM for any given team aligns
with the corresponding windows for all other teams, which also extend from
noon to 1 PM. And in use cases for which you want to compare like windows
across another dimension, such as between teams, this alignment is very
useful. However, it comes at a somewhat subtle cost. All of the active
windows from noon to 1 PM become complete at around the same time,
which means that once an hour the system is hit with a massive load of
windows to materialize.

To see what I mean, let’s look at a concrete example (Example 4-5). We’ll
begin with a score summation pipeline as we’ve used in most examples, with
fixed two-minute windows, and a single watermark trigger.

_Example 4-5. Watermark completeness trigger (same as Example 2-6)_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark()))
.apply(Sum.integersPerKey());

But in this instance, we’ll look at two different keys (see Figure 4-8) from the


same dataset in parallel. What we’ll see is that the outputs for those two keys
are all aligned, on account of the windows being aligned across all of the
keys. As a result, we end up with _N_ panes being materialized every time the
watermark passes the end of a window, where _N_ is the number of keys with
updates in that window. In this example, where _N_ is 2, that’s maybe not too
painful. But when _N_ starts to order in the thousands, millions, or more, that
synchronized burstiness can become problematic.

```
Figure 4-8. Aligned fixed windows
```
In circumstances for which comparing across windows is unnecessary, it’s
often more desirable to spread window completion load out evenly across
time. This makes system load more predictable, which can reduce the
provisioning requirements for handling peak load. In most systems, however,
unaligned fixed windows are only available if the system provides support for
them out of the box. But with custom-windowing support, it’s a relatively
trivial modification to the default fixed-windows implementation to provide
unaligned fixed-windows support. What we want to do is continue
guaranteeing that the windows for all elements being grouped together (i.e.,
the ones with the same key) have the same alignment, while relaxing the
alignment restriction across different keys. The code changes to the default
fixed-windowing strategy and looks something like Example 4-6.

_Example 4-6. Abbreviated UnalignedFixedWindows implementation_

public class UnalignedFixedWindows
extends WindowFn<KV<K, V>, IntervalWindow> {
private final Duration size;
private final Duration offset;
public Collection<IntervalWindow> assignWindow(AssignContext c) {
long perKeyShift = hash(c.element().key()) % size;

```
00:00 / 00:00
```
```
2
```

long start = perKeyShift + c.timestamp().getMillis()

- c.timestamp()
.plus(size)
.minus(offset)
return Arrays.asList(IntervalWindow(new Instant(start), size));
}
}

With this change, the windows for all elements _with the same key_ are
aligned, but the windows for elements _with different keys_ will (typically) be
unaligned, thus spreading window completion load out at the cost of also
making comparisons across keys somewhat less meaningful. We can switch
our pipeline to use our new windowing strategy, illustrated in Example 4-7.

_Example 4-7. Unaligned fixed windows with a single watermark trigger_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(UnalignedFixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark()))
.apply(Sum.integersPerKey());

And then you can see what this looks like in Figure 4-9 by comparing
different fixed-window alignments across the same dataset as before (in this
case, I’ve chosen a maximal phase shift between the two alignments to most
clearly call out the benefits, given that randomly chosen phases across a large
number of keys will result in similar effects).

```
Figure 4-9. Unaligned fixed windows
```
Note how there are no instances where we emit multiple panes for multiple
keys simultaneously. Instead, the panes arrive individually at a much more
even cadence. This is another example of being able to make trade-offs in one
dimension (ability to compare across keys) in exchange for benefits in

```
3
```
```
00:00 / 00:00
```

another dimension (reduced peak resource provisioning requirements) when
the use case allows. Such flexibility is critical when you’re trying to process
massive quantities of data as efficiently as possible.

Let’s now look at a second variation on fixed windows, one which is more
intrinsically tied to the data being processed.

**Per-element/key fixed windows**

Our second example comes courtesy of one of the early adopters of Cloud
Dataflow. This company generates analytics data for its customers, but each
customer is allowed to configure the window size over which it wants to
aggregate its metrics. In other words, each customer gets to define the
specific size of its fixed windows.

Supporting a use case like this isn’t too difficult as long the number of
available window sizes is itself fixed. For example, you could imagine
offering the option of choosing 30-minute, 60-minute, and 90-minute fixed
windows and then running a separate pipeline (or fork of the pipeline) for
each of those options. Not ideal, but not too horrible. However, that rapidly
becomes intractable as the number of options increases, and in the limit of
providing support for truly arbitrary window sizes (which is what this
customer’s use case required) is entirely impractical.

Fortunately, because each record the customer processes is already annotated
with metadata describing the desired size of window for aggregation,
supporting arbitrary, per-user fixed-window size was as simple as changing a
couple of lines from the stock fixed-windows implementation, as
demonstrated in Example 4-8.

_Example 4-8. Modified (and abbreviated) FixedWindows implementation that
supports per-element window sizes_

public class PerElementFixedWindows<T extends HasWindowSize%gt;
extends WindowFn<T, IntervalWindow> {
private final Duration offset;
public Collection<IntervalWindow> assignWindow(AssignContext c) {
long perElementSize = c.element().getWindowSize();
long start = perKeyShift + c.timestamp().getMillis()

- c.timestamp()


.plus(size)
.minus(offset)
.getMillis() % size.getMillis();
return Arrays.asList(IntervalWindow(
new Instant(start), perElementSize));
}
}

With this change, each element is assigned to a fixed window with the
appropriate size, as dictated by metadata carried around in the element itself.
Changing the pipeline code to use this new strategy is again trivial, as shown
in Example 4-9.

_Example 4-9. Per-element fixed-window sizes with a single watermark
trigger_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(PerElementFixedWindows.of(TWO_MINUTES))
.triggering(AfterWatermark()))
.apply(Sum.integersPerKey());

And then looking at an this pipeline in action (Figure 4-10), it’s easy to see
that the elements for Key A all have two minutes as their window size,
whereas the elements for Key B have one-minute window sizes.

```
Figure 4-10. Per-key custom-sized fixed windows
```
This really isn’t something you would ever reasonably expect a system to
provide to you; the nature of where window size preferences are stored is too
use-case specific for it to make sense to try to build into a standard API.
Nevertheless, as exhibited by this customer’s needs, use cases like this do
exist. That’s why the flexibility provided by custom windowing is so
powerful.

```
4
```
```
00:00 / 00:00
```

## Variations on Session Windows

To really drive home the usefulness of custom windowing, let’s look at one
final example, which is a variation on sessions. Session windowing is
understandably a bit more complex than fixed windows. Its implementation
consists of the following:

Assignment

```
Each element is initially placed into a proto-session window that begins at
the element’s timestamp and extends for the gap duration.
```
Merging

```
At grouping time, all eligible windows are sorted, after which any
overlapping windows are merged together.
```
An abbreviated version of the sessions code (hand merged together from a
number of helper classes) looks something like that shown in Example 4-10.

_Example 4-10. Abbreviated Sessions implementation_

public class Sessions extends WindowFn<Object, IntervalWindow> {
private final Duration gapDuration;
public Collection<IntervalWindow> assignWindows(AssignContext c) {
return Arrays.asList(
new IntervalWindow(c.timestamp(), gapDuration));
}
public void mergeWindows(MergeContext c) throws Exception {
List<IntervalWindow> sortedWindows = new ArrayList<>();
for (IntervalWindow window : c.windows()) {
sortedWindows.add(window);
}
Collections.sort(sortedWindows);
List<MergeCandidate> merges = new ArrayList<>();
MergeCandidate current = new MergeCandidate();
for (IntervalWindow window : sortedWindows) {
if (current.intersects(window)) {
current.add(window);
} else {
merges.add(current);
current = new MergeCandidate(window);
}


}
merges.add(current);
for (MergeCandidate merge : merges) {
merge.apply(c);
}
}
}

As before, the point of seeing the code isn’t so much to teach you how
custom windowing functions are implemented, or even what the
implementation of sessions looks like; it’s really to show the ease with which
you can support new use via custom windowing.

**Bounded sessions**

One such custom use case I’ve come across multiple times is bounded
sessions: sessions that are not allowed to grow beyond a certain size, either in
time, element count, or some other dimension. This can be for semantic
reasons, or it can simply be an exercise in spam protection. However, given
the variations in types of limits (some use cases care about total session size
in event time, some care about total element count, some care about element
density, etc.), it’s difficult to provide a clean and concise API for bounded
sessions. Much more practical is allowing users to implement their own
custom windowing logic, tailored to their specific use case. An example of
one such use case, in which session windows are time-limited, might look
something like Example 4-11 (eliding some of the builder boilerplate we’ll
utilize here).

_Example 4-11. Abbreviated Sessions implementation_

public class BoundedSessions extends WindowFn<Object, IntervalWindow> {
private final Duration gapDuration;
private final Duration maxSize;
public Collection<IntervalWindow> assignWindows(AssignContext c) {
return Arrays.asList(
new IntervalWindow(c.timestamp(), gapDuration));
}
private Duration windowSize(IntervalWindow window) {
return window == null
? new Duration( 0 )


: new Duration(window.start(), window.end());
}
public static void mergeWindows(
WindowFn<?, IntervalWindow>.MergeContext c) throws Exception {
List<IntervalWindow> sortedWindows = new ArrayList<>();
for (IntervalWindow window : c.windows()) {
sortedWindows.add(window);
}
Collections.sort(sortedWindows);
List<MergeCandidate> merges = new ArrayList<>();
MergeCandidate current = new MergeCandidate();
for (IntervalWindow window : sortedWindows) {
MergeCandidate next = new MergeCandidate(window);
if (current.intersects(window)) {
current.add(window);
if (windowSize(current.union) <= (maxSize - gapDuration))
continue;
// Current window exceeds bounds, so flush and move to next
next = new MergeCandidate();
}
merges.add(current);
current = next;
}
merges.add(current);
for (MergeCandidate merge : merges) {
merge.apply(c);
}
}
}

As always, updating our pipeline (the early/on-time/late version of it, from
Example 2-7, in this case) to use this custom windowing strategy is trivial, as
you can see in Example 4-12.

_Example 4-12. Early, on-time, and late firings via the early/on-time/late API_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(BoundedSessions
.withGapDuration(ONE_MINUTE)
.withMaxSize(THREE_MINUTES))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))


.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

And executed over our running example, it might then look something like
Figure 4-11.

```
Figure 4-11. Per-key custom-sized fixed windows
```
Note how the large session with value 36 that spanned [12:00.26, 12:05.20),
or nearly five minutes of time, in the unbounded sessions implementation
from Figure 2-7 now ends up broken apart into two shorter sessions of length
2 minutes and 2 minutes 53 seconds.

Given how few systems provide custom windowing support today, it’s worth
pointing out how much more effort would be required to implement such a
thing using a system that supported only an unbounded sessions
implementation. Your only real recourse would be to write code downstream
of the session grouping logic that looked at the generated sessions and
chopped them up if they exceed the length limit. This would require the
ability to decompose a session after the fact, which would obviate the
benefits of incremental aggregation (something we look at in more detail in
Chapter 7), increasing cost. It would also eliminate any spam protection
benefits one might hope to gain by limiting session lengths, because the
sessions would first need to grow to their full sizes before being chopped or
truncated.

## One Size Does Not Fit All

We’ve now looked at three real-world use cases, each of which was a subtle
variation on the stock types of windowing typically provided by data

```
00:00 / 00:00
```

processing systems: unaligned fixed windows, per-element fixed windows,
and bounded sessions. In all three cases, we saw how simple it was to support
those use cases via custom windowing and how much more difficult (or
expensive) it would be to support those use cases without it. Though custom
windowing doesn’t see broad support across the industry as yet, it’s a feature
that provides much needed flexibility for balancing trade-offs when building
data processing pipelines that need to handle complex, real-world use cases
over massive amounts of data as efficiently as possible.


