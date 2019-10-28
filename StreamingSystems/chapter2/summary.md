 ## Summary

With this chapter complete, you now understand the basics of robust stream
processing and are ready to go forth into the world and do amazing things. Of

```
00:00 / 00:00
```

course, there are eight more chapters anxiously waiting for your attention, so
hopefully you won’t go forth like right now, this very minute. But regardless,
let’s recap what we’ve just covered, lest you forget any of it in your haste to
amble forward. First, the major concepts we touched upon:

Event time versus processing time

```
The all-important distinction between when events occurred and when
they are observed by your data processing system.
```
Windowing

```
The commonly utilized approach to managing unbounded data by slicing
it along temporal boundaries (in either processing time or event time,
though we narrow the definition of windowing in the Beam Model to
mean only within event time).
```
Triggers

```
The declarative mechanism for specifying precisely when materialization
of output makes sense for your particular use case.
```
Watermarks

```
The powerful notion of progress in event time that provides a means of
reasoning about completeness (and thus missing data) in an out-of-order
processing system operating on unbounded data.
```
Accumulation

```
The relationship between refinements of results for a single window for
cases in which it’s materialized multiple times as it evolves.
```
Second, the four questions we used to frame our exploration:

```
What results are calculated? = transformations.
```
```
Where in event time are results calculated? = windowing.
When in processing time are results materialized? = triggers plus
watermarks.
```

```
How do refinements of results relate? = accumulation.
```
Third, to drive home the flexibility afforded by this model of stream
processing (because in the end, that’s really what this is all about: balancing
competing tensions like correctness, latency, and cost), a recap of the major
variations in output we were able to achieve over the same dataset with only a
minimal amount of code change:

```
Integer summation
Example 2-1 / Figure 2-3
```
```
Integer summation
Fixed windows batch
Example 2-2 / Figure 2-5
```
```
Integer summation
Fixed windows streaming
Repeated per-record
trigger
Example 2-3 / Figure 2-6
```
```
Integer summation
Fixed windows streaming
Repeated aligned-delay
trigger
Example 2-4 / Figure 2-7
```
```
Integer summation
Fixed windows streaming
Repeated unaligned-delay
trigger
Example 2-5 / Figure 2-8
```
```
Integer summation
Fixed windows streaming
Heuristic watermark
trigger
Example 2-6 / Figure 2-10
```
```
Integer summation
Fixed windows streaming
Early/on-time/late trigger
```
```
Integer summation
Fixed windows streaming
Early/on-time/late trigger
```
```
Integer summation
Fixed windows streaming
Early/on-time/late trigger
```

```
Discarding
Example 2-9 / Figure 2-13
```
```
Accumulating
Example 2-7 / Figure 2-11
```
```
Accumulating and
Retracting
Example 2-10 / ???
```
All that said, at this point, we’ve really looked at only one type of
windowing: fixed windowing in event time. As we know, there are a number
of dimensions to windowing, and I’d like to touch upon at least two more of
those before we call it day with the Beam Model. First, however, we’re going
to take a slight detour to dive deeper into the world of watermarks, as this
knowledge will help frame future discussions (and be fascinating in and of
itself). Enter Slava, stage right...

If you’re fortunate enough to be reading the Safari version of the book, you
have full-blown time-lapse animations just like in “Streaming 102”. For print,
Kindle, and other ebook versions, there are static images with a link to
animated versions on the web.

Bear with me here. Fine-grained emotional expressions via composite
punctuation (i.e., emoticons) are strictly forbidden in O’Reilly publications <
winky-smiley/>.

And indeed, we did just that with the original triggers feature in Beam. In
retrospect, we went a bit overboard. Future iterations will be simpler and
easier to use, and in this book I focus only on the pieces that are likely to
remain in some form or another.

More accurately, the input to the function is really the state at time _P_ of
everything upstream of the point in the pipeline where the watermark is being
observed: the input source, buffered data, data actively being processed, and
so on; but conceptually it’s simpler to think of it as a mapping from
processing time to event time.

Note that I specifically chose to omit the value of 9 from the heuristic
watermark because it will help me to make some important points about late
data and watermark lag. In reality, a heuristic watermark might be just as
likely to omit some other value(s) instead, which in turn could have

1

2

3

4

5


significantly less drastic effect on the watermark. If winnowing late-arriving
data from the watermark is your goal (which is very valid in some cases, such
as abuse detection, for which you just want to see a significant majority of the
data as quickly as possible), you don’t necessarily want a heuristic watermark
rather than a perfect watermark. What you really want is a percentile
watermark, which explicitly drops some percentile of late-arriving data from
its calculations. See Chapter 3.

Which isn’t to say there aren’t use cases that care primarily about
correctness and not so much about latency; in those cases, using an accurate
watermark as the sole driver of output from a pipeline is a reasonable
approach.

And, as we know from before, this assertion is either guaranteed, in the case
of a perfect watermark being used, or an educated guess, in the case of a
heuristic watermark.

You might note that there should logically be a fourth mode: discarding and
retracting. That mode isn’t terribly useful in most cases, so I don’t discuss it
further here.

In retrospect, it probably would have been clearer to choose a different set
of names that are more oriented toward the observed nature of data in the
materialized stream (e.g., “output modes”) rather than names describing the
state management semantics that yield those data. Perhaps: discarding mode
→ delta mode, accumulating mode → value mode, accumulating and
retracting mode → value and retraction mode? However, the
discarding/accumulating/accumulating and retracting names are enshrined in
the 1.x and 2.x lineages of the Beam Model, so I don’t want to introduce
potential confusion in the book by deviating. Also, it’s very likely
accumulating modes will blend into the background more with Beam 3.0 and
the introduction of sink triggers; more on this when we discuss SQL in
Chapter 8.

6

7

8

9

