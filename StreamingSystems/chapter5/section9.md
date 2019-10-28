 ## Summary

In summary, exactly-once data processing, which was once thought to be
incompatible with low-latency results, is quite possible—Dataflow does it
efficiently without sacrificing latency. This enables far richer uses for stream
processing.

Although this chapter has focused on Dataflow-specific techniques, other
streaming systems also provide exactly-once guarantees. Apache Spark
Streaming runs streaming pipelines as a series of small batch jobs, relying on
exactly-once guarantees in the Spark batch runner. Apache Flink uses a
variation on Chandy Lamport distributed snapshots to get a running
consistent state and can use these snapshots to ensure exactly-once
processing. We encourage you to learn about these other systems, as well, for
a broad understanding of how different stream-processing systems work!

In fact, no system we are aware of that provides at-least once (or better) is
able to guarantee this, including all other Beam runners.

Dataflow also provides an accurate batch runner; however, in this context
we are focused on the streaming runner.

The Dataflow optimizer groups many steps together and adds shuffles only
where they are needed.

```
Batch pipelines also need to guard against duplicates in shuffle. However
```
1

2

3

4


the problem is much easier to solve in batch, which is why historical batch
systems did do this and streaming systems did not. Streaming runtimes that
use a microbatch architecture, such as Spark Streaming, delegate duplicate
detection to a batch shuffler.

A lot of care is taken to make sure this checkpointing is efficient; for
example, schema and access pattern optimizations that are intimately tied to
the characteristics of the underlying key/value store.

This is not the custom user-supplied timestamp used for windowing. Rather
this is a deterministic processing-time timestamp that is assigned by the
sending worker.

Some care needs to be taken to ensure that this algorithm works. Each
sender must guarantee that the system timestamps it generates are strictly
increasing, and this guarantee must be maintained across worker restarts.

In theory, we could dispense with startup scans entirely by lazily building
the Bloom filter for a bucket only when a threshold number of records show
up with timestamps in that bucket.

At the time of this writing, a new, more-flexible API called SplittableDoFn
is available for Apache Beam.

We assume that nobody is maliciously modifying the bytes in the file
while we are reading it.

Again note that the SplittableDoFn API has different methods for this.
Using the requiresDedupping override.
Note that these determinism boundaries might become more explicit in the
Beam Model at some point. Other Beam runners vary in their ability to
handle nondeterministic user code.

As long as you properly handle the failure when the source file no longer
exists.

Due to the global nature of the service, BigQuery does not guarantee that
all duplicates are removed. Users can periodically run a query over their
tables to remove any duplicates that were not caught by the streaming insert

5

6

7

8

9

10

11

12

13

14

15


API. See the BigQuery documentation for more information.

Resilient Distributed Datasets; Spark’s abstraction of a distributed dataset,
similar to PCollection in Beam.

These sequence numbers are per connection and are unrelated to the
snapshot epoch number.

Only for nonidempotent sinks. Completely idempotent sinks do not need to
wait for the snapshot to complete.

Specifically, Flink assumes that the mean time to worker failure is less
than the time to snapshot; otherwise, the pipeline would be unable to make
progress.

16

17

18

19


