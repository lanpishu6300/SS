 ## Use Cases

To illustrate, let’s examine some built-in sources and sinks to see how they
implement the aforementioned patterns.

## Example Source: Cloud Pub/Sub


Cloud Pub/Sub is a fully managed, scalable, reliable, and low-latency system
for delivering messages from publishers to subscribers. Publishers publish
data on named topics, and subscribers create named subscriptions to pull data
from these topics. Multiple subscriptions can be created for a single topic, in
which case each subscription receives a full copy of all data published on the
topic from the time of the subscription’s creation. Pub/Sub guarantees that
records will continue to be delivered until they are acknowledged; however, a
record might be delivered multiple times.

Pub/Sub is intended for distributed use, so many publishing processes can
publish to the same topic and many subscribing processes can pull from the
same subscription. After a record has been pulled, the subscriber must
acknowledge it within a certain amount of time, or that pull expires and
Pub/Sub will redeliver that record to another of the subscribing processes.

Although these characteristics make Pub/Sub highly scalable, they also make
it a challenging source for a system like Dataflow. It’s impossible to know
which record will be delivered to which worker, and in which order. What’s
more, in the case of failure, redelivery might send the records to different
workers in different orders!

Pub/Sub provides a stable message ID with each message, and this ID will be
the same upon redelivery. The Dataflow Pub/Sub source will default to using
this ID for removing duplicates from Pub/Sub. (The records are shuffled
based on a hash of the ID, so that repeated deliveries are always processed on
the same worker.) In some cases, however, this is not quite enough. The
user’s publishing process might retry publishes, and as a result introduce
duplicates into Pub/Sub. From that service’s perspective these are unique
records, so they will get unique record IDs. Dataflow’s Pub/Sub source
allows the user to provide their own record IDs as a custom attribute. As long
as the publisher sends the same ID when retrying, Dataflow will be able to
detect these duplicates.

Beam (and therefore Dataflow) provides a reference source implementation
for Pub/Sub. However, keep in mind that this is _not_ what Dataflow uses but
rather an implementation used only by non-Dataflow runners (such as


Apache Spark, Apache Flink, and the DirectRunner). For a variety of reasons,
Dataflow handles Pub/Sub internally and does not use the public Pub/Sub
source.

## Example Sink: Files

The streaming runner can use Beam’s file sinks (TextIO, AvroIO, and any
other sink that implements FileBasedSink) to continuously output records to
files. Example 5-3 provides an example use case.

_Example 5-3. Windowed file writes_

c.apply(Window.<..>into(FixedWindows.of(Duration.standardMinutes( 1 ))))
.apply(TextIO.writeStrings().to(new
MyNamePolicy()).withWindowedWrites());

The snippet in Example 5-3 writes 10 new files each minute, containing data
from that window. MyNamePolicy is a user-written function that determines
output filenames based on the shard and the window. You can also use
triggers, in which case each trigger pane will be output as a new file.

This process is implemented using a variant on the pattern in Example 5-3.
Files are written out to temporary locations, and these temporary filenames
are sent to a subsequent transform through a GroupByKey. After the
GroupByKey is a finalize transform that atomically moves the temporary files
into their final location. The pseudocode in Example 5-4 provides a sketch of
how a consistent streaming file sink is implemented in Beam. (For more
details, see FileBasedSink and WriteFiles in the Beam codebase.)

_Example 5-4. File sink_

c
// Tag each record with a random shard id.
.apply("AttachShard", WithKeys.of(new
RandomShardingKey(getNumShards())))
// Group all records with the same shard.
.apply("GroupByShard", GroupByKey.<..>())
// For each window, write per-shard elements to a temporary file. This
is the
// non-deterministic side effect. If this DoFn is executed multiple


times, it will
// simply write multiple temporary files; only one of these will pass
on through
// to the Finalize stage.
.apply("WriteTempFile", ParDo.of(new DoFn<..> {
@ProcessElement
public void processElement(ProcessContext c, BoundedWindow window) {
// Write the contents of c.element() to a temporary file.
// User-provided name policy used to generate a final filename.
c.output(new FileResult()).
}
}))
// Group the list of files onto a singleton key.
.apply("AttachSingletonKey", WithKeys.<..>of((Void)null))
.apply("FinalizeGroupByKey", GroupByKey.<..>create())
// Finalize the files by atomically renaming them. This operation is
idempotent.
// Once this DoFn has executed once for a given FileResult, the
temporary file
// is gone, so any further executions will have no effect.
.apply("Finalize", ParDo.of(new DoFn<..>, Void> {
@ProcessElement
public void processElement(ProcessContext c) {
for (FileResult result : c.element()) {
rename(result.getTemporaryFileName(),
result.getFinalFilename());
}
}}));

You can see how the nonidempotent work is done in WriteTempFile. After
the GroupByKey completes, the Finalize step will always see the same
bundles across retries. Because file rename is idempotent, this give us an
exactly-once sink.

## Example Sink: Google BigQuery

Google BigQuery is a fully managed, cloud-native data warehouse. Beam
provides a BigQuery sink, and BigQuery provides a streaming insert API that
supports extremely low-latency inserts. This streaming insert API allows
allows you to tag inserts with a unique ID, and BigQuery will attempt to filter

```
14
```
```
15
```

duplicate inserts with the same ID. To use this capability, the BigQuery
sink must generate statistically unique IDs for each record. It does this by
using the java.util.UUID package, which generates statistically unique
128-bit IDs.

Generating a random universally unique identifier (UUID) is a
nondeterministic operation, so we must add a Reshuffle before we insert
into BigQuery. After we do this, any retries by Dataflow will always use the
same UUID that was shuffled. Duplicate attempts to insert into BigQuery
will always have the same insert ID, so BigQuery is able to filter them. The
pseudocode shown in Example 5-5 illustrates how the BigQuery sink is
implemented.

_Example 5-5. BigQuery sink_

// Apply a unique identifier to each record
c
.apply(new DoFn<> {
@ProcessElement
public void processElement(ProcessContext context) {
String uniqueId = UUID.randomUUID().toString();
context.output(KV.of(ThreadLocalRandom.current().nextInt( 0 , 50 ),
new RecordWithId(context.element(),
uniqueId)));
}
})
// Reshuffle the data so that the applied identifiers are stable and will
not change.
.apply(Reshuffle.<Integer, RecordWithId>of())
// Stream records into BigQuery with unique ids for deduplication.
.apply(ParDo.of(new DoFn<..> {
@ProcessElement
public void processElement(ProcessContext context) {
insertIntoBigQuery(context.element().record(),
context.element.id());
}
});

Again we split the sink into a nonidempotent step (generating a random
number), followed by a step that is idempotent.

```
15
```


