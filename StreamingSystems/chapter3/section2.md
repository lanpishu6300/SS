## Source Watermark Creation

Where do these watermarks come from? To establish a watermark for a data
source, we must assign a logical event timestamp to every message entering
the pipeline from that source. As Chapter 2 informs us, all watermark
creation falls into one of two broad categories: _perfect_ or _heuristic_. To remind
ourselves about the difference between perfect and heuristic watermarks, let’s
look at Figure 3-2, which presents the windowed summation example from
Chapter 2.

```
1
```

_Figure 3-2. Windowed summation with perfect (left) and heuristic (right) watermarks_

```
00:00 / 00:00
```

Notice that the distinguishing feature is that perfect watermarks ensure that
the watermark accounts for _all_ data, whereas heuristic watermarks admit
some late-data elements.

After the watermark is created as either perfect or heuristic, watermarks
remain so throughout the rest of the pipeline. As to what makes watermark
creation perfect or heuristic, it depends a great deal on the nature of the
source that’s being consumed. To see why, let’s look at a few examples of
each type of watermark creation.

## Perfect Watermark Creation

Perfect watermark creation assigns timestamps to incoming messages in such
a way that the resulting watermark is a _strict guarantee_ that no data with
event times less than the watermark will ever be seen again from this source.
Pipelines using perfect watermark creation never have to deal with late data;
that is, data that arrive after the watermark has advanced past the event times
of newly arriving messages. However, perfect watermark creation requires
perfect knowledge of the input, and thus is impractical for many real-world
distributed input sources. Here are a couple of examples of use cases that can
create perfect watermarks:

Ingress timestamping

```
A source that assigns ingress times as the event times for data entering the
system can create a perfect watermark. In this case, the source watermark
simply tracks the current processing time as observed by the pipeline.
This is essentially the method that nearly all streaming systems
supporting windowing prior to 2016 used.
Because event times are assigned from a single, monotonically increasing
source (actual processing time), the system thus has perfect knowledge
about which timestamps will come next in the stream of data. As a result,
event-time progress and windowing semantics become vastly easier to
reason about. The downside, of course, is that the watermark has no
correlation to the event times of the data themselves; those event times
were effectively discarded, and the watermark instead merely tracks the
```

```
progress of data relative to its arrival in the system.
```
Static sets of time-ordered logs

```
A statically sized input source of time-ordered logs (e.g., an Apache
Kafka topic with a static set of partitions, where each partition of the
source contains monotonically increasing event times) would be
relatively straightforward source atop which to create a perfect
watermark. To do so, the source would simply track the minimum event
time of unprocessed data across the known and static set of source
partitions (i.e., the minimum of the event times of the most recently read
record in each of the partitions).
Similar to the aforementioned ingress timestamps, the system has perfect
knowledge about which timestamps will come next, thanks to the fact that
event times across the static set of partitions are known to increase
monotonically. This is effectively a form of bounded out-of-order
processing; the amount of disorder across the known set of partitions is
bounded by the minimum observed event time among those partitions.
Typically, the only way you can guarantee monotonically increasing
timestamps within partitions is if the timestamps within those partitions
are assigned as data are written to it; for example, by web frontends
logging events directly into Kafka. Though still a limited use case, this is
definitely a much more useful one than ingress timestamping upon arrival
at the data processing system because the watermark tracks meaningful
event times of the underlying data.
```
## Heuristic Watermark Creation

Heuristic watermark creation, on the other hand, creates a watermark that is
merely an _estimate_ that no data with event times less than the watermark will
ever be seen again. Pipelines using heuristic watermark creation might need
to deal with some amount of _late data_. Late data is any data that arrives after
the watermark has advanced past the event time of this data. Late data is only
possible with heuristic watermark creation. If the heuristic is a reasonably

```
2
```

good one, the amount of late data might be very small, and the watermark
remains useful as a completion estimate. The system still needs to provide a
way for the user to cope with late data if it’s to support use cases requiring
correctness (e.g., things like billing).

For many real-world, distributed input sources, it’s computationally or
operationally impractical to construct a perfect watermark, but still possible
to build a highly accurate heuristic watermark by taking advantage of
structural features of the input data source. Following are two example for
which heuristic watermarks (of varying quality) are possible:

Dynamic sets of time-ordered logs

```
Consider a dynamic set of structured log files (each individual file
containing records with monotonically increasing event times relative to
other records in the same file but with no fixed relationship of event times
between files), where the full set of expected log files (i.e., partitions, in
Kafka parlance) is not known at runtime. Such inputs are often found in
global-scale services constructed and managed by a number of
independent teams. In such a use case, creating a perfect watermark over
the input is intractable, but creating an accurate heuristic watermark is
quite possible.
By tracking the minimum event times of unprocessed data in the existing
set of log files, monitoring growth rates, and utilizing external
information like network topology and bandwidth availability, you can
create a remarkably accurate watermark, even given the lack of perfect
knowledge of all the inputs. This type of input source is one of the most
common types of unbounded datasets found at Google, so we have
extensive experience with creating and analyzing watermark quality for
such scenarios and have seen them used to good effect across a number of
use cases.
```
Google Cloud Pub/Sub

```
Cloud Pub/Sub is an interesting use case. Pub/Sub currently makes no
guarantees on in-order delivery; even if a single publisher publishes two
messages in order, there’s a chance (usually small) that they might be
```

```
delivered out of order (this is due to the dynamic nature of the underlying
architecture, which allows for transparent scaling up to very high levels
of throughput with zero user intervention). As a result, there’s no way to
guarantee a perfect watermark for Cloud Pub/Sub. The Cloud Dataflow
team has, however, built a reasonably accurate heuristic watermark by
taking advantage of what knowledge is available about the data in Cloud
Pub/Sub. The implementation of this heuristic is discussed at length as a
case study later in this chapter.
```
Consider an example where users play a mobile game, and their scores are
sent to our pipeline for processing: you can generally assume that for any
source utilizing mobile devices for input it will be generally impossible to
provide a perfect watermark. Due to the problem of devices that go offline for
extended periods of time, there’s just no way to provide any sort of
reasonable estimate of absolute completeness for such a data source. You
can, however, imagine building a watermark that accurately tracks input
completeness for devices that are currently online, similar to the Google
Pub/Sub watermark described a moment ago. Users who are actively online
are likely the most relevant subset of users from the perspective of providing
low-latency results anyway, so this often isn’t as much of a shortcoming as
you might initially think.

With heuristic watermark creation, broadly speaking, the more that is known
about the source, the better the heuristic, and the fewer late data items will be
seen. There is no one-size-fits-all solution, given that the types of sources,
distributions of events, and usage patterns will vary greatly. But in either case
(perfect or heuristic), after a watermark is created at the input source, the
system can propagate the watermark through the pipeline perfectly. This
means perfect watermarks will remain perfect downstream, and heuristic
watermarks will remain strictly as heuristic as they were when established.
This is the benefit of the watermark approach: you can reduce the complexity
of tracking completeness in a pipeline entirely to the problem of creating a
watermark at the source.

