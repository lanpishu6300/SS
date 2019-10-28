 ## Other Systems

Now that we have explained Dataflow’s exactly once in detail, let us contrast
this with some brief overviews of other popular streaming systems. Each
implements exactly-once guarantees in a different way and makes different
trade-offs as a result.

## Apache Spark Streaming

Spark Streaming uses a microbatch architecture for continuous data
processing. Users logically deal with a stream object; however, under the
covers, Spark represents this stream as a continuous series of RDDs. Each
RDD is processed as a batch, and Spark relies on the exactly-once nature of
batch processing to ensure correctness; as mentioned previously, techniques
for correct batch shuffles have been known for some time. This approach can
cause increased latency to output—especially for deep pipelines and high
input volumes—and often careful tuning is required to achieve desired
latency.

Spark does assume that operations are all idempotent and might replay the
chain of operations up the current point in the graph. A checkpoint primitive
is provided, however, that causes an RDD to be materialized, guaranteeing
that history prior to that RDD will not be replayed. This checkpoint feature is
intended for performance reasons (e.g., to prevent replaying an expensive
operation); however, you can also use it to implement nonidempotent side
effects.

## Apache Flink

Apache Flink also provides exactly-once processing for streaming pipelines
but does so in a manner different than either Dataflow or Spark. Flink
streaming pipelines periodically compute consistent snapshots, each
representing the consistent point-in-time state of an entire pipeline. Flink
snapshots are computed progressively, so there is no need to halt all
processing while computing a snapshot. This allows records to continue

```
16
```

flowing through the system while taking a snapshot, alleviating some of the
latency issues with the Spark Streaming approach.

Flink implements these snapshots by inserting special numbered snapshot
markers into the data streams flowing from sources. As each operator
receives a snapshot marker, it executes a specific algorithm allowing it to
copy its state to an external location and propagate the snapshot marker to
downstream operators. After all operators have executed this snapshot
algorithm, a complete snapshot is made available. Any worker failures will
cause the entire pipeline to roll back its state from the last complete snapshot.
In-flight messages do not need to be included in the snapshot. All message
delivery in Flink is done via an ordered TCP-based channel. Any connection
failures can be handled by resuming the connection from the last good
sequence number; unlike Dataflow, Flink tasks are statically allocated to
workers, so it can assume that the connection will resume from the same
sender and replay the same payloads.

Because Flink might roll back to the previous snapshot at any time, any state
modifications not yet in a snapshot must be considered tentative. A sink that
sends data to the world outside the Flink pipeline must wait until a snapshot
has completed, and then send only the data that is included in that snapshot.
Flink provides a notifySnapshotComplete callback that allows sinks to
know when each snapshot is completed, and send the data onward. Even
though this does affect the output latency of Flink pipelines, this latency is
introduced only at sinks. In practice, this allows Flink to have lower end-to-
end latency than Spark for deep pipelines because Spark introduces batch
latency at each stage in the pipeline.

Flink’s distributed snapshots are an elegant way of dealing with consistency
in a streaming pipeline; however, a number of assumptions are made about
the pipeline. Failures are assumed to be rare, as the impact of a failure
(rolling back to the previous snapshot) is substantial. To maintain low-latency
output, it is also assumed that snapshots can complete quickly. It remains to
be seen whether this causes issues on very large clusters where the failure
rate will likely increase, as will the time needed to complete a snapshot.

```
17
```
```
18
```
```
19
```

Implementation is also simplified by assuming that tasks are statically
allocated to workers (at least within a single snapshot epoch). This
assumption allows Flink to provide a simple exactly-once transport between
workers because it knows that if a connection fails, the same data can be
pulled in order from the same worker. In contrast, tasks in Dataflow are
constantly load balanced between workers (and the set of workers is
constantly growing and shrinking), so Dataflow is unable to make this
assumption. This forces Dataflow to implement a much more complex
transport layer in order to provide exactly-once processing.


