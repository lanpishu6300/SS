  ## Where : Session Windows

Enough with processing-time windowing. Let’s now go back to tried-and-
true event-time windowing, but now we’re going to look at one of my
favorite features: the dynamic, data-driven windows called _sessions_.

Sessions are a special type of window that captures a period of activity in the
data that is terminated by a gap of inactivity. They’re particularly useful in
data analysis because they can provide a view of the activities for a specific
user over a specific period of time during which they were engaged in some
activity. This allows for the correlation of activities within the session,
drawing inferences about levels of engagement based on the lengths of the

```
00:00 / 00:00
```

sessions, and so on.

From a windowing perspective, sessions are particularly interesting in two
ways:

```
They are an example of a data-driven window : the location and sizes
of the windows are a direct consequence of the input data
themselves, rather than being based on some predefined pattern
within time, as are fixed and sliding windows.
```
```
They are also an example of an unaligned window ; that is, a window
that does not apply uniformly across the data, but instead only to a
specific subset of the data (e.g., per user). This is in contrast to
aligned windows like fixed and sliding windows, which typically
apply uniformly across the data.
```
For some use cases, it’s possible to tag the data within a single session with a
common identifier ahead of time (e.g., a video player that emits heartbeat
pings with quality-of-service information; for any given viewing, all of the
pings can be tagged ahead of time with a single session ID). In this case,
sessions are much easier to construct because it’s basically just a form of
grouping by key.

However, in the more general case (i.e., where the actual session itself is not
known ahead of time), the sessions must be constructed from the locations of
the data within time alone. When dealing with out-of-order data, this
becomes particularly tricky.

Figure 4-5 shows an example of this, with five independent records grouped
together into session windows with a gap timeout of 60 minutes. Each record
starts out in a 60-minute window of its own (a proto-session). Merging
together overlapping proto-sessions yields the two larger session windows
containing three and two records, respectively.


```
Figure 4-5. Unmerged proto-session windows, and the resultant merged sessions
```
They key insight in providing general session support is that a complete
session window is, by definition, a composition of a set of smaller,
overlapping windows, each containing a single record, with each record in
the sequence separated from the next by a gap of inactivity no larger than a
predefined timeout. Thus, even if we observe the data in the session out of
order, we can build up the final session simply by merging together any
overlapping windows for individual data as they arrive.

To look at this another way, consider the example we’ve been using so far. If
we specify a session timeout of one minute, we would expect to identify two
sessions in the data, delineated in Figure 4-6 by the dashed black lines. Each
of those sessions captures a burst of activity from the user, with each event in
the session separate by less than one minute from at least one other event in
the session.


```
Figure 4-6. Sessions we want to compute
```
To see how the window merging works to build up these sessions over time
as events are encountered, let’s look at it in action. We’ll take the early/late
code with retractions enabled from Example 2-10 and update the windowing
to build sessions with a one-minute gap duration timeout instead. Example 4-
3 illustrates what this looks like.

_Example 4-3. Early/on-time/late firings with session windows and retractions_

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(Sessions.withGapDuration(ONE_MINUTE))
.triggering(
AfterWatermark()
.withEarlyFirings(AlignedDelay(ONE_MINUTE))
.withLateFirings(AfterCount(1))))
.apply(Sum.integersPerKey());

Executed on a streaming engine, you’d get something like that shown in
Figure 4-7 (note that I’ve left in the dashed black lines annotating the
expected final sessions for reference).


```
Figure 4-7. Early and late firings with session windows and retractions on a streaming
engine
```
There’s quite a lot going on here, so I’ll walk you through some of it:

```
When the first record with value 5 is encountered, it’s placed into a
single proto-session window that begins at that record’s event time
and spans the width of the session gap duration; for example, one
minute beyond the point at which that datum occurred. Any
windows we encounter in the future that overlap this window should
be part of the same session and will be merged into it as such.
```
```
The second record to arrive is the 7, which similarly is placed into its
own proto-session window, given that it doesn’t overlap with the
window for the 5.
```
```
In the meantime, the watermark has passed the end of the first
window, so the value of 5 is materialized as an on-time result just
before 12:06. Shortly thereafter, the second window is also
materialized as a speculative result with value 7, right as processing
time hits 12:06.
We next observe a pair of records 3 and 4, the proto-sessions for
which overlap. As a result, they are merged together, and by the time
the early trigger for 12:07 fires, a single window with value 7 is
emitted.
```
```
When the 8 arrives shortly thereafter, it overlaps with both of the
windows with value 7. All three are thus merged together, forming a
new combined session with value 22. When the watermark then
passes the end of this session, it materializes both the new session
```
```
00:00 / 00:00
```

```
with value 22 as well as retractions for the two windows of value 7
that were previously emitted, but later incorporated into it.
A similar dance occurs when the 9 arrives late, joining the proto-
session with value 5 and session with value 22 into a single larger
session of value 36. The 36 and the retractions for the 5 and 22
windows are all emitted immediately by the late data trigger.
```
This is some pretty powerful stuff. And what’s really awesome is how easy it
is to describe something like this within a model that breaks apart the
dimensions of stream processing into distinct, composable pieces. In the end,
you can focus more on the interesting business logic at hand, and less on the
minutiae of shaping the data into some usable form.

If you don’t believe me, check out this blog post describing how to manually
build up sessions on Spark Streaming 1.x (note that this is not done to point
fingers at them; the Spark folks had just done a good enough job with
everything else that someone actually bothered to go to the trouble of
documenting what it takes to build a specific variety of sessions support on
top of Spark 1.x; you can’t say the same for most other systems out there).
It’s quite involved, and they’re not even doing proper event-time sessions, or
providing speculative or late firings, or retractions.
