 ## Flume

We now return to Google territory to talk about the official successor to
MapReduce within Google: Flume ([Figure 10-8] sometimes also called
FlumeJava in reference to the original Java version of the system, and not to
be confused with Apache Flume, which is an entirely different beast that just
so happens to share the same name).

```
Figure 10-8. Timeline: Flume
```
The Flume project was founded by Craig Chambers when the Google Seattle
office opened in 2007. It was motivated by a desire to solve some of the
inherent shortcomings of MapReduce, which had become apparent over the
first few years of its success. Many of these shortcomings revolved around


MapReduce’s rigid Map → Shuffle → Reduce structure; though refreshingly
simple, it carried with it some downsides:

```
Because many use cases cannot be served by the application of a
single MapReduce, a number of bespoke orchestration systems
began popping up across Google for coordinating sequences of
MapReduce jobs. These systems all served essentially the same
purpose (gluing together multiple MapReduce jobs to create a
coherent pipeline solving a complex problem). However, having
been developed independently, they were naturally incompatible and
a textbook example of unnecessary duplication of effort.
```
```
What’s worse, there were numerous cases in which a clearly written
sequence of MapReduce jobs would introduce inefficiencies thanks
to the rigid structure of the API. For example, one team might write
a MapReduce that simply filtered out some number of elements; that
is, a map-only job with an empty reducer. It might be followed up by
another team’s map-only job doing some element-wise enrichment
(with yet another empty reducer). The output from the second job
might then finally be consumed by a final team’s MapReduce
performing some grouping aggregation over the data. This pipeline,
consisting of essentially a single chain of Map phases followed by a
single Reduce phase, would require the orchestration of three
completely independent jobs, each chained together by shuffle and
output phases materializing the data. But that’s assuming you
wanted to keep the codebase logical and clean, which leads to the
final downside...
```
```
In an effort to optimize away these inefficiencies in their
MapReductions, engineers began introducing manual optimizations
that would obfuscate the simple logic of the pipeline, increasing
maintenance and debugging costs.
```
Flume addressed these issues by providing a composable, high-level API for
describing data processing pipelines, essentially based around the same
PCollection and PTransform concepts found in Beam, as illustrated in


Figure 10-9.

```
Figure 10-9. High-level pipelines in Flume (image credit: Frances Perry)
```
These pipelines, when launched, would be fed through an optimizer to
generate a plan for an optimally efficient sequence of MapReduce jobs, the
execution of which was then orchestrated by the framework, which you can
see illustrated in Figure 10-10.

```
Figure 10-10. Optimization from a logical pipeline to a physical execution plan
```
Perhaps the most important example of an automatic optimization that Flume

```
5
```

can perform is fusion (which Reuven discussed a bit back in Chapter 5), in
which two logically independent stages can be run in the same job either
sequentially (consumer-producer fusion) or in parallel (sibling fusion), as
depicted in Figure 10-11.

```
Figure 10-11. Fusion optimizations combine successive or parallel operations together
into the same physical operation
```
Fusing two stages together eliminates serialization/deserialization and
network costs, which can be significant in pipelines processing large amounts
of data.

Another type of automatic optimization is _combiner lifting_ (see Figure 10-
12 ), the mechanics of which we already touched upon in Chapter 7 when we
talked about incremental combining. Combiner lifting is simply the automatic
application of multilevel combine logic that we discussed in that chapter: a
combining operation (e.g., summation) that logically happens after a
grouping operation is partially lifted into the stage preceding the group-by-
key (which by definition requires a trip across the network to shuffle the data)
so that it can perform partial combining before the grouping happens. In
cases of very hot keys, this can greatly reduce the amount of data shuffled
over the network, and also spread the load of computing the final aggregate


more smoothly across multiple machines.

```
Figure 10-12. Combiner lifting applies partial aggregation on the sender side of a group-
by-key operation before completing aggregation on the consumer side
```
As a result of its cleaner API and automatic optimizations, Flume Java was
an instant hit upon its introduction at Google in early 2009. Following on the
heels of that success, the team published the paper titled “Flume Java: Easy,
Efficient Data-Parallel Pipelines” (see Figure 10-13), itself an excellent
resource for learning more about the system as it originally existed.


```
Figure 10-13. FlumeJava paper
```
Flume C++ followed not too much later in 2011, and in early 2012 Flume
was introduced into Noogler training provided to all new engineers at
Google. That was the beginning of the end for MapReduce.

Since then, Flume has been migrated to no longer use MapReduce as its
execution engine; instead, it uses a custom execution engine, called Dax,
built directly into the framework itself. By freeing Flume itself from the
confines of the previously underlying Map → Shuffle → Reduce structure of
MapReduce, Dax enabled new optimizations, such as the dynamic work
rebalancing feature described in Eugene Kirpichov and Malo Denielou’s “No
shard left behind” blog post (Figure 10-14).

```
6
```

```
Figure 10-14. “No shard left behind” post
```
Though discussed in that post in the context of Cloud Dataflow, dynamic
work rebalancing (or liquid sharding, as it’s colloquially known at Google)
automatically rebalances extra work from straggler shards to other idle
workers in the system as they complete their work early. By dynamically
rebalancing the work distribution over time, it’s possible to come much
closer to an optimal work distribution than even the best educated initial
splits could ever achieve. It also allows for adapting to variations across the
pool of workers, where a slow machine that might have otherwise held up the
completion of a job is simply compensated for by moving most of its tasks to
other workers. When liquid sharding was rolled out at Google, it recouped
significant amounts of resources across the fleet.

One last point on Flume is that it was also later extended to support streaming
semantics. In addition to the batch Dax backend, Flume was extended to be
able to execute pipelines on the MillWheel stream processing system
(discussed in a moment). Most of the high-level streaming semantics


concepts we’ve discussed in this book were first incorporated into Flume
before later finding their way into Cloud Dataflow and eventually Apache
Beam.

All that said, the primary thing to take away from Flume in this section is the
introduction of a notion of _high-level pipelines_ , which enabled the _automatic
optimization_ of clearly written, logical pipelines. This enabled the creation of
much larger and complex pipelines, without the need for manual
orchestration or optimization, and all while keeping the code for those
pipelines logical and clear.

