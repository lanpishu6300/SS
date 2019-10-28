## Definition

Consider any pipeline that ingests data and outputs results continuously. We
wish to solve the general problem of when it is safe to call an event-time
window closed, meaning that the window does not expect any more data. To
do so we would like to characterize the progress that the pipeline is making
relative to its unbounded input.

One naive approach for solving the event-time windowing problem would be
to simply base our event-time windows on the current processing time. As we
saw in Chapter 1, we quickly run into trouble—data processing and transport
is not instantaneous, so processing and event times are almost never equal.
Any hiccup or spike in our pipeline might cause us to incorrectly assign
messages to windows. Ultimately, this strategy fails because we have no
robust way to make any guarantees about such windows.

Another intuitive, but ultimately incorrect, approach would be to consider the


rate of messages processed by the pipeline. Although this is an interesting
metric, the rate may vary arbitrarily with changes in input, variability of
expected results, resources available for processing, and so on. Even more
important, rate does not help answer the fundamental questions of
completeness. Specifically, rate does not tell us when we have seen all of the
messages for a particular time interval. In a real-world system, there will be
situations in which messages are not making progress through the system.
This could be the result of transient errors (such as crashes, network failures,
machine downtime), or the result of persistent errors such as application-level
failures that require changes to the application logic or other manual
intervention to resolve. Of course, if lots of failures are occurring, a rate-of-
processing metric might be a good proxy for detecting this. However a rate
metric could never tell us that a single message is failing to make progress
through our pipeline. Even a single such message, however, can arbitrarily
affect the correctness of the output results.

We require a more robust measure of progress. To arrive there, we make one
fundamental assumption about our streaming data: _each message has an
associated logical event timestamp_. This assumption is reasonable in the
context of continuously arriving unbounded data because this implies the
continuous generation of input data. In most cases, we can take the time of
the original event’s occurrence as its logical event timestamp. With all input
messages containing an event timestamp, we can then examine the
distribution of such timestamps in any pipeline. Such a pipeline might be
distributed to process in parallel over many agents and consuming input
messages with no guarantee of ordering between individual shards. Thus, the
set of event timestamps for active in-flight messages in this pipeline will form
a distribution, as illustrated in Figure 3-1.

Messages are ingested by the pipeline, processed, and eventually marked
completed. Each message is either “in-flight,” meaning that it has been
received but not yet completed, or “completed,” meaning that no more
processing on behalf of this message is required. If we examine the
distribution of messages by event time, it will look something like Figure 3-1.
As time advances, more messages will be added to the “in-flight” distribution


on the right, and more of those messages from the “in-flight” part of the
distribution will be completed and moved into the “completed” distribution.

```
Figure 3-1. Distribution of in-flight and completed message event times within a streaming
pipeline. New messages arrive as input and remain “in-flight” until processing for them
completes. The leftmost edge of the “in-flight” distribution corresponds to the oldest
unprocessed element at any given moment.
```
There is a key point on this distribution, located at the leftmost edge of the
“in-flight” distribution, corresponding to the oldest event timestamp of any
unprocessed message of our pipeline. We use this value to define the
watermark:

```
00:00 / 00:00
```
```
1
```

```
The watermark is a monotonically increasing timestamp of the oldest
work not yet completed.
```
There are two fundamental properties that are provided by this definition that
make it useful:

Completeness

```
If the watermark has advanced past some timestamp T , we are guaranteed
by its monotonic property that no more processing will occur for on-time
(nonlate data) events at or before T. Therefore, we can correctly emit any
aggregations at or before T. In other words, the watermark allows us to
know when it is correct to close a window.
```
Visibility

```
If a message is stuck in our pipeline for any reason, the watermark cannot
advance. Furthermore, we will be able to find the source of the problem
by examining the message that is preventing the watermark from
advancing.
```

