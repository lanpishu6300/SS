 ## Exactly Once in Sources

Beam provides a source API for reading data into a Dataflow pipeline.
Dataflow might retry reads from a source if processing fails and needs to
ensure that every unique record produced by a source is processed exactly
once.

For most sources Dataflow handles this process transparently; such sources
are _deterministic_. For example, consider a source that reads data out of files.
The records in a file will always be in a deterministic order and at
deterministic byte locations, no matter how many times the file is read. The
filename and byte location uniquely identify each record, so the service can
automatically generate unique IDs for each record. Another source that
provides similar determinism guarantees is Apache Kafka; each Kafka topic
is divided into a static set of partitions, and records in a partition always have
a deterministic order. Such deterministic sources will work seamlessly in
Dataflow with no duplicates.

However, not all sources are so simple. For example, one common source for
Dataflow pipelines is Google Cloud Pub/Sub. Pub/Sub is a _nondeterministic_
source: multiple subscribers can pull from a Pub/Sub topic, but which
subscribers receive a given message is unpredictable. If processing fails
Pub/Sub will redeliver messages but the messages might be delivered to
different workers than those that processed them originally, and in a different
order. This nondeterministic behavior means that Dataflow needs assistance
for detecting duplicates because there is no way for the service to
deterministically assign record IDs that will be stable upon retry. (We dive
into a more detailed case study of Pub/Sub later in this chapter.)

Because Dataflow cannot automatically assign record IDs, nondeterministic
sources are required to inform the system what the record IDs should be.
Beam’s Source API provides the UnboundedReader.getCurrentRecordId
method. If a source provides unique IDs per record and notifies Dataflow that
it requires deduplication, records with the same ID will be filtered out.

```
9
```
```
10
```
```
11
```
```
12
```
