 ## Processing-Time Watermarks

Until now, we have been looking at watermarks as they relate to the data
flowing through our system. We have seen how looking at the watermark can
help us identify the overall delay between our oldest data and real time.

```
00:00 / 00:00
```
```
rd th th
```
```
th rd th
```
```
rd
```
```
th
```
```
th
```

However, this is not enough to distinguish between old data and a delayed
system. In other words, by only examining the event-time watermark as we
have defined it up until now, we cannot distinguish between a system that is
processing data from an hour ago quickly and without delay, and a system
that is attempting to process real-time data and has been delayed for an hour
while doing so.

To make this distinction, we need something more: processing-time
watermarks. We have already seen that there are two time domains in a
streaming system: processing time and event time. Until now, we have
defined the watermark entirely in the event-time domain, as a function of
timestamps of the data flowing through the system. This is an event-time
watermark. We will now apply the same model to the processing-time
domain to define a processing-time watermark.

Our stream processing system is constantly performing operations such as
shuffling messages between stages, reading or writing messages to persistent
state, or triggering delayed aggregations based on watermark progress. All of
these operations are performed in response to previous operations done at the
current or upstream stage of the pipeline. Thus, just as data elements “flow”
through the system, a cascade of operations involved in processing these
elements also “flows” through the system.

We define the processing-time watermark in the exact same way as we have
defined the event-time watermark, except instead of using the event-time
timestamp of oldest work not yet completed, we use the processing-time
timestamp of the oldest operation not yet completed. An example of delay to
the processing-time watermark could be a stuck message delivery from one
stage to another, a stuck I/O call to read state or external data, or an exception
while processing that prevents processing from completing.

The processing-time watermark, therefore, provides a notion of processing
delay separate from the data delay. To understand the value of this
distinction, consider the graph in Figure 3-12 where we look at the event-time
watermark delay.

We see that the data delay is monotonically increasing, but there is not


enough information to distinguish between the cases of a stuck system and
stuck data. Only by looking at the processing-time watermark, shown in
Figure 3-13, can we distinguish the cases.

```
Figure 3-12. Event-time watermark increasing. It is not possible to know from this
information whether this is due to data buffering or system processing delay.
```
```
Figure 3-13. Processing-time watermark also increasing. This indicates that the system
processing is delayed.
```
In the first case (Figure 3-12), when we examine the processing-time
watermark delay we see that it too is increasing. This tells us that an
operation in our system is stuck, and the stuckness is also causing the data
delay to fall behind. Some real-world examples of situations in which this
might occur are when there is a network issue preventing message delivery
between stages of a pipeline or if a failure has occurred and is being retried.
In general, a growing processing-time watermark indicates a problem that is
preventing operations from completing that are necessary to the system’s
function, and often involves user or administrator intervention to resolve.

In this second case, as seen in Figure 3-14, the processing-time watermark


delay is small. This tells us that there are no stuck operations. The event-time
watermark delay is still increasing, which indicates that we have some
buffered state that we are waiting to drain. This is possible, for example, if
we are buffering some state while waiting for a window boundary to emit an
aggregation, and corresponds to a normal operation of the pipeline, as in
Figure 3-15.

```
Figure 3-14. Event-time watermark delay increasing, processing-time watermark stable.
This is an indication that data are buffered in the system and waiting to be processed,
rather than an indication that a system operation is preventing data processing from
completing.
```
```
Figure 3-15. Watermark delay for fixed windows. The event-time watermark delay
increases as elements are buffered for each window, and decreases as each window’s
aggregate is emitted via an on-time trigger, whereas the processing-time watermark simply
tracks system-level delays (which remain relatively steady in a healthy pipeline).
```
Therefore, the processing-time watermark is a useful tool in distinguishing
system latency from data latency. In addition to visibility, we can use the
processing-time watermark at the system-implementation level for tasks such
as garbage collection of temporary state (Reuven talks more about an


example of this in Chapter 5).


