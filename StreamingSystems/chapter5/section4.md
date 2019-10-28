 ## Performance

To implement exactly-once shuffle delivery, a catalog of record IDs is stored
in each receiver key. For every record that arrives, Dataflow looks up the
catalog of IDs already seen to determine whether this record is a duplicate.
Every output from step to step is checkpointed to storage to ensure that the
generated record IDs are stable.

However, unless implemented carefully, this process would significantly
degrade pipeline performance for customers by creating a huge increase in
reads and writes. Thus, for exactly-once processing to be viable for Dataflow
users, that I/O has to be reduced, in particular by preventing I/O on every
record.

Dataflow achieves this goal via two key techniques: _graph optimization_ and

```
5
```

_Bloom filters_.

## Graph Optimization

The Dataflow service runs a series of optimizations on the pipeline graph
before executing it. One such optimization is _fusion_ , in which the service
fuses many logical steps into a single execution stage. Figure 5-3 shows some
simple examples.

```
Figure 5-3. Example optimizations: fusion
```
All fused steps are run as an in-process unit, so there’s no need to store
exactly-once data for each of them. In many cases, fusion reduces the entire
graph down to a few physical steps, greatly reducing the amount of data
transfer needed (and saving on state usage, as well).

Dataflow also optimizes associative and commutative Combine operations
(such as Count and Sum) by performing partial combining locally before
sending the data to the main grouping operation, as illustrated in Figure 5-4.
This approach can greatly reduce the number of messages for delivery,
consequently also reducing the number of reads and writes.


```
Figure 5-4. Example optimizations: combiner lifting
```
## Bloom Filters

The aforementioned optimizations are general techniques that improve
exactly-once performance as a byproduct. For an optimization aimed strictly
at improving exactly-once processing, we turn to _Bloom filters_.

In a healthy pipeline, most arriving records will not be duplicates. We can use
that fact to greatly improve performance via Bloom filters, which are
compact data structures that allow for quick set-membership checks. Bloom
filters have a very interesting property: they can return false positives but
never false negatives. If the filter says “Yes, the element is in the set,” we
know that the element is _probably_ in the set (with a probability that can be
calculated). However, if the filter says an element is _not_ in the set, it
definitely isn’t. This function is a perfect fit for the task at hand.

The implementation in Dataflow works like this: each worker keeps a Bloom
filter of every ID it has seen. Whenever a new record ID shows up, it looks it
up in the filter. If the filter returns false, this record is not a duplicate and the
worker can skip the more expensive lookup from stable storage. It needs to
do that second lookup only if the Bloom filter returns true, but as long as the
filter’s false-positive rate is low, that step is rarely needed.

Bloom filters tend to fill up over time, however, and as that happens, the
false-positive rate increases. We also need to construct this Bloom filter anew


any time a worker restarts by scanning the ID catalog stored in state.
Helpfully, Dataflow attaches a system timestamp to each record. Thus,
instead of creating a single Bloom filter, the service creates a separate one for
every 10-minute range. When a record arrives, Dataflow queries the
appropriate filter based on the system timestamp. This step prevents the
Bloom filters from saturating because filters are garbage-collected over time,
and it also bounds the amount of data that needs to be scanned at startup.

Figure 5-5 illustrates this process: records arrive in the system and are
delegated to a Bloom filter based on their arrival time. None of the records
hitting the first filter are duplicates, and all of their catalog lookups are

filtered. Record r1 is delivered a second time, so a catalog lookup is needed
to verify that it is indeed a duplicate; the same is true for records r4 and r6.
Record r8 is not a duplicate; however, due to a false positive in its Bloom
filter, a catalog lookup is generated (which will determine that r8 is not a
duplicate and should be processed).

```
Figure 5-5. Exactly-once Bloom filters
```
```
6
```
```
7
```
```
8
```

## Garbage Collection

Every Dataflow worker persistently stores a catalog of unique record IDs it
has seen. As Dataflow’s state and consistency model is per-key, in reality
each key stores a catalog of records that have been delivered to that key. We
can’t store these identifiers forever, or all available storage will eventually fill
up. To avoid that issue, you need garbage collection of acknowledged record
IDs.

One strategy for accomplishing this goal would be for senders to tag each
record with a strictly increasing sequence number in order to track the earliest
sequence number still in flight (corresponding to an unacknowledged record
delivery). Any identifier in the catalog with an earlier sequence number could
then be garbage-collected because all earlier records have already been
acknowledged.

There is a better alternative, however. As previously mentioned, Dataflow
already tags each record with a system timestamp that is used for bucketing
exactly-once Bloom filters. Consequently, instead of using sequence numbers
to garbage-collect the exactly-once catalog, Dataflow calculates a garbage-
collection watermark based on these system timestamps (this is the
processing-time watermark discussed in Chapter 3). A nice side benefit of
this approach is that because this watermark is based on the amount of
physical time spent waiting in a given stage (unlike the data watermark,
which is based on custom event times), it provides intuition on what parts of
the pipeline are slow. This metadata is the basis for the System Lag metric
shown in the Dataflow WebUI.

What happens if a record arrives with an old timestamp and we’ve already
garbage-collected identifiers for this point in time? This can happen due to an
effect we call _network remnants_ , in which an old message becomes stuck for
an indefinite period of time inside the network and then suddenly shows up.
Well, the low watermark that triggers garbage collection won’t advance until
record deliveries have been acknowledged, so we know that this record has
already been successfully processed. Such network remnants are clearly


duplicates and are ignored.

