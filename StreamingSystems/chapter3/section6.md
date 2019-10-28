  ## Case Studies

Now that we’ve laid the groundwork for how watermarks ought to behave,
it’s time to take a look at some real systems to understand how different
mechanisms of the watermark are implemented. We hope that these shed
some light on the trade-offs that are possible between latency and correctness
as well as scalability and availability for watermarks in real-world systems.

## Case Study: Watermarks in Google Cloud

## Dataflow

There are many possible approaches to implementing watermarks in a stream
processing system. Here, we present a quick survey of the implementation in
Google Cloud Dataflow, a fully managed service for executing Apache Beam
pipelines. Dataflow includes SDKs for defining data processing workflows,
and a Cloud Platform managed service to run those workflows on Google
Cloud Platform resources.

Dataflow stripes (shards) each of the data processing steps in its data
processing graph across multiple physical workers by splitting the available
keyspace of each worker into key ranges and assigning each range to a

worker. Whenever a GroupByKey operation with distinct keys is encountered,
data must be shuffled to corresponding keys.

Figure 3-16 depicts a logical representation of the processing graph with a
GroupByKey.


```
Figure 3-16. A GroupByKey step consumes data from another DoFn. This means that there
is a data shuffle between the keys of the first step and the keys of the second step.
```
Whereas the physical assignment of key ranges to workers might look
Figure 3-17.


_Figure 3-17. Key ranges of both steps are assigned (striped) across the available workers._


In the watermark propagation section, we discussed that the watermark is
maintained for multiple subcomponents of each step. Dataflow keeps track of
the per-range watermarks of each of these components. Watermark
aggregation then involves computing the minimum of each watermark across
all ranges, ensuring that the following guarantees are met:

```
All ranges must be reporting a watermark. If a watermark is not
present for a range, we cannot advance the watermark, because a
range not reporting must be treated as unknown.
Ensure that the watermark is monotonically increasing. Because late
data is possible, we must not update the watermark if it would cause
the watermark to move backward.
```
Google Cloud Dataflow performs aggregation via a centralized aggregator
agent. We can shard this agent for efficiency. From a correctness standpoint,
the watermark aggregator serves as a “single source of truth” about the
watermark.

Ensuring correctness in distributed watermark aggregation poses certain
challenges. It is paramount that watermarks are not advanced prematurely
because advancing the watermark prematurely will turn on-time data into late
data. Specifically, as physical assignments are actuated to workers, the
workers maintain leases on the persistent state attached to the key ranges,
ensuring that only a single worker may mutate the persistent state for a key.
To guarantee watermark correctness, we must ensure that each watermark
update from a worker process is admitted into the aggregate only if the
worker process still maintains a lease on its persistent state; therefore, the
watermark update protocol must take state ownership lease validation into
account.

## Case Study: Watermarks in Apache Flink

Apache Flink is an open source stream processing framework for distributed,
high-performing, always-available, and accurate data streaming applications.
It is possible to run Beam programs using a Flink runner. In doing so, Beam


relies on the implementation of stream processing concepts such as
watermarks within Flink. Unlike Google Cloud Dataflow, which implements
watermark aggregation via a centralized watermark aggregator agent, Flink
performs watermark tracking and aggregation in-band.

To understand how this works, let’s look at a Flink pipeline, as shown in
Figure 3-18.

```
Figure 3-18. A Flink pipeline with two sources and event-time watermarks propagating in-
band
```
In this pipeline data is generated at two sources. These sources also both
generate watermark “checkpoints” that are sent synchronously in-band with
the data stream. This means that when a watermark checkpoint from source A
for timestamp “53” is emitted, it guarantees that no nonlate data messages
will be emitted from source A with timestamp behind “53”. The downstream
“keyBy” operators consume the input data and the watermark checkpoints.
As new watermark checkpoints are consumed, the downstream operators’
view of the watermark is advanced, and a new watermark checkpoint for
downstream operators can be emitted.

This choice to send watermark checkpoints in-band with the data stream
differs from the Cloud Dataflow approach that relies on central aggregation
and leads to a few interesting trade-offs.

```
6
```

Following are some advantages of in-band watermarks:

Reduced watermark propagation latency, and very low-latency watermarks

```
Because it is not necessary to have watermark data traverse multiple hops
and await central aggregation, it is possible to achieve very low latency
more easily with the in-band approach.
```
No single point of failure for watermark aggregation

```
Unavailability in the central watermark aggregation agent will lead to a
delay in watermarks across the entire pipeline. With the in-band
approach, unavailability of part of the pipeline cannot cause watermark
delay to the entire pipeline.
```
Inherent scalability

```
Although Cloud Dataflow scales well in practice, more complexity is
needed to achieve scalability with a centralized watermark aggregation
service versus implicit scalability with in-band watermarks.
```
Here are some advantages of out-of-band watermark aggregation:

Single source of “truth”

```
For debuggability, monitoring, and other applications such as throttling
inputs based on pipeline progress, it is advantageous to have a service that
can vend the values of watermarks rather than having watermarks implicit
in the streams, with each component of the system having its own partial
view.
```
Source watermark creation

```
Some source watermarks require global information. For example,
sources might be temprarily idle, have low data rates, or require out-of-
band information about the source or other system components to
generate the watermarks. This is easier to achieve in a central service. For
an example see the case study that follows on source watermarks for
Google Cloud Pub/Sub.
```

## Case Study: Source Watermarks for Google Cloud

## Pub/Sub

Google Cloud Pub/Sub is a fully managed real-time messaging service that
allows you to send and receive messages between independent applications.
Here, we discuss how to create a reasonable heuristic watermark for data sent
into a pipeline via Cloud Pub/Sub.

First, we need to describe a little about how Pub/Sub works. Messages are
published on Pub/Sub _topics_. A particular topic can be subscribed to by any
number of Pub/Sub _subscriptions_. The same messages are delivered on all
subscriptions subscribed to a given topic. The method of delivery is for
clients to _pull_ messages off the subscription, and to ack the receipt of
particular messages via provided IDs. Clients do not get to choose which
messages are pulled, although Pub/Sub does attempt to provide oldest
messages first, with no hard guarantees around this.

To build a heuristic, we make some assumptions about the source that is
sending data into Pub/Sub. Specifically, we assume that the timestamps of the
original data are “well behaved”; in other words, we expect a bounded
amount of out-of-order timestamps on the source data, before it is sent to
Pub/Sub. Any data that are sent with timestamps outside the allowed out-of-
order bounds will be considered late data. In our current implementation, this
bound is at least 10 seconds, meaning reordering of timestamps up to 10
seconds before sending to Pub/Sub will not create late data. We call this
value the _estimation band_. Another way to look at this is that when the
pipepline is perfectly caught up with the input, the watermark will be 10
seconds behind real time to allow for possible reorderings from the source. If
the pipeline is backlogged, all of the backlog (not just the 10-second band) is
used for estimating the watermark.

What are the challenges we face with Pub/Sub? Because Pub/Sub does not
guarantee ordering, we must have some kind of additional metadata to know
enough about the backlog. Luckily, Pub/Sub provides a measurement of
backlog in terms of the “oldest unacknowledged publish timestamp.” This is
not the same as the event timestamp of our message, because Pub/Sub is


agnostic to the application-level metadata being sent through it; instead, this
is the timestamp of when the message was ingested by Pub/Sub.

This measurement is not the same as an event-time watermark. It is in fact the
processing-time watermark for Pub/Sub message delivery. The Pub/Sub
publish timestamps are not equal to the event timestamps, and in the case that
historical (past) data are being sent, it might be arbitrarily far away. The
ordering on these timestamps might also be different because, as mentioned
earlier, we allow a limited amount of reordering.

However, we can use this as a measure of backlog to learn enough
information about the event timestamps present in the backlog so that we can
create a reasonable watermark as follows.

We create two subscriptions to the topic containing the input messages: a
_base subscription_ that the pipeline will actually use to read the data to be
processed, and a _tracking subscription_ , which is used for metadata only, to
perform the watermark estimation.

Taking a look at our base subscription in Figure 3-19, we see that messages
might arrive out of order. We label each message with its Pub/Sub publish
timestamp “pt” and its event-time timestamp “et.” Note that the two time
domains can be unrelated.

```
Figure 3-19. Processing-time and event-time timestamps of messages arriving on a
Pub/Sub subscription
```
Some messages on the base subscription are unacknowledged forming a


backlog. This might be due to them not yet being delivered or they might
have been delivered but not yet processed. Remember also that pulls from
this subscription are distributed across multiple shards. Thus, it is not
possible to say just by looking at the base subscription what our watermark
should be.

The tracking subscription, seen in Figure 3-20, is used to effectively inspect
the backlog of the base subscription and take the minimum of the event
timestamps in the backlog. By maintaining little or no backlog on the
tracking subscription, we can inspect the messages ahead of the base
subsciption’s oldest unacknowledged message.

```
Figure 3-20. An additional “tracking” subscription receiving the same messages as the
“base” subscription
```
We stay caught up on the tracking subscription by ensuring that pulling from
this subscription is computationally inexpensive. Conversely, if we fall
sufficiently behind on the tracking subscription, we will stop advancing the
watermark. To do so, we ensure that at least one of the following conditions
is met:

```
The tracking subscription is sufficiently ahead of the base
subscription. Sufficiently ahead means that the tracking subscription
is ahead by at least the estimation band. This ensures that any
bounded reorder within the estimation band is taken into account.
```

```
The tracking subscription is sufficiently close to real time. In other
words, there is no backlog on the tracking subscription.
```
We acknowledge the messages on the tracking subscription as soon as
possible, after we have durably saved metadata about the publish and event
timestamps of the messages. We store this metadata in a sparse histogram
format to minimize the amount of space used and the size of the durable
writes.

Finally, we ensure that we have enough data to make a reasonable watermark
estimate. We take a band of event timestamps we’ve read from our tracking
subscription with publish timestamps newer than the oldest unacknowledged
of the base subscription, or the width of the estimation band. This ensures
that we consider all event timestamps in the backlog, or if the backlog is
small, the most recent estimation band, to make a watermark estimate.

Finally, the watermark value is computed to be the minimum event time in
the band.

This method is correct in the sense that all timestamps within the reordering
limit of 10 seconds at the input will be accounted for by the watermark and
not appear as late data. However, it produces possibly an overly conservative
watermark, one that advances “too slowly” in the sense described in
Chapter 2. Because we consider all messages ahead of the base subscription’s
oldest unacknowledged message on the tracking subscription, we can include
event timestamps in the watermark estimate for messages that have already
been acknowledged.

Additionally, there are a few heuristics to ensure progress. This method
works well in the case of dense, frequently arriving data. In the case of sparse
or infrequent data, there might not be enough recent messages to build a
reasonable estimate. In the case that we have not seen data on the
subscription in more than two minutes (and there’s no backlog), we advance
the watermark to near real time. This ensures that the watermark and the
pipeline continue to make progress even if no more messages are
forthcoming.


All of the above ensures that as long as source data-event timestamp
reordering is within the estimation band, there will be no additional late data.
