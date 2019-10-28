 ## Summary

Advanced windowing is a complex and varied topic. In this chapter, we
covered three advanced concepts:

Processing-time windows

```
We saw how this relates to event-time windowing, calling out the places
where it’s inherently useful and, most important, identifying those where
it’s not by specifically highlighting the stability of results that event-time
windowing affords us.
```
Session windows

```
We had our first introduction to the dynamic class of merging window
strategies and seeing just how much heavy lifting the system does for us
in providing such a powerful construct that you can simply drop into
place.
```
Custom windows

```
Here, we looked at three real-world examples of custom windows that are
difficult or impossible to achieve in systems that provide only a static set
of stock windowing strategies but relatively trivial to implement in a
system with custom-windowing support:
```
```
Unaligned fixed windows , which provide a more even distribution
of outputs over time when using a watermark trigger in conjunction
with fixed windows.
```

```
Per-element fixed windows , which provide the flexibility to
dynamically choose the size of fixed windows per element (e.g., to
provide customizable per-user or per-ad-campaign window sizes),
for greater customization of the pipeline semantics to the use case at
hand.
```
```
Bounded-session windows , which limit how large a given session
may grow; for example, to counteract spam attempts or to place a
bound on the latency for completed sessions being materialized by
the pipeline.
```
After deep diving through watermarks in Chapter 3 with Slava and taking a
broad survey of advanced windowing here, we’ve now gone well beyond the
basics of robust stream processing in multiple dimensions. With that, we
conclude our focus on the Beam Model and thus Part I of the book.

Up next is Reuven’s Chapter 5 on consistency guarantees, exactly-once
processing, and side effects, after which we begin our journey into Part II,
_Streams and Tables_ with Chapter 6.

As far as I know, Apache Flink is the only other system to support custom
windowing to the extent that Beam does. And to be fair, its support extends
even beyond that of Beam’s, thanks to the ability to provide a custom
window evictor. Head asplode.

And I’m not actually aware of any such systems at this time.
This naturally implies the use of keyed data, but because windowing is
intrinsically tied to grouping by key anyway, that restriction isn’t particularly
burdensome.

And it’s not critical that the element itself know the window size; you could
just as easily look up and cache the appropriate window size for whatever the
desired dimension is; for example, per-user.

1

2

3

4

