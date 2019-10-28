## Summary

We just took a whirlwind tour through a decade and a half of advances in
data processing technology, with a focus on the contributions that made
streaming systems what they are today. To summarize one last time, the main
takeaways for each system were:

MapReduce—scalability and simplicity

```
By providing a simple set of abstractions for data processing on top of a
robust and scalable execution engine, MapReduce allowed data engineers
to focus on the business logic of their data processing needs rather than
the gnarly details of building distributed systems resilient to the failure
modes of commodity hardware.
```
Hadoop—open source ecosystem

```
By building an open source platform on the ideas of MapReduce, Hadoop
created a thriving ecosystem that expanded well beyond the scope of its
progenitor and allowed a multitude of new ideas to flourish.
```
Flume—pipelines, optimization

```
By coupling a high-level notion of logical pipeline operations with an
intelligent optimizer, Flume made it possible to write clean and
maintainable pipelines whose capabilities extended beyond the Map →
Shuffle → Reduce confines of MapReduce, without sacrificing any of the
performance theretofore gained by contorting the logical pipeline via
hand-tuned manual optimizations.
```
Storm—low latency with weak consistency

```
By sacrificing correctness of results in favor of decreased latency, Storm
brought stream processing to the masses and also ushered in the era of the
Lambda Architecture, where weakly consistent stream processing engines
were run alongside strongly consistent batch systems to realize the true
```

```
business goal of low-latency, eventually consistent results.
```
Spark—strong consistency

```
By utilizing repeated runs of a strongly consistent batch engine to provide
continuous processing of unbounded datasets, Spark Streaming proved it
possible to have both correctness and low-latency results, at least for in-
order datasets.
```
MillWheel—out-of-order processing

```
By coupling strong consistency and exactly-once processing with tools
for reasoning about time like watermarks and timers, MillWheel
conquered the challenge of robust stream processing over out-of-order
data.
```
Kafka—durable streams, streams and tables

```
By applying the concept of a durable log to the problem of streaming
transports, Kafka brought back the warm, fuzzy feeling of replayability
that had been lost by ephemeral streaming transports like RabbitMQ and
TCP sockets. And by popularizing the ideas of stream and table theory, it
helped shed light on the conceptual underpinnings of data processing in
general.
```
Cloud Dataflow—unified batch plus streaming

```
By melding the out-of-order stream processing concepts from MillWheel
with the logical, automatically optimizable pipelines of Flume, Cloud
Dataflow provided a unified model for batch plus streaming data
processing that provided the flexibility to balance the tensions between
correctness, latency, and cost to match any given use case.
```
Flink—open source stream processing innovator

```
By rapidly bringing the power of out-of-order processing to the world of
open source and combining it with innovations of their own like
distributed snapshots and its related savepoints features, Flink raised the
bar for open source stream processing and helped lead the current charge
```

```
of stream processing innovation across the industry.
```
Beam—portability

```
By providing a robust abstraction layer that incorporates the best ideas
from across the industry, Beam provides a portability layer positioned as
the programmatic equivalent to the declarative lingua franca provided by
SQL, while also encouraging the adoption of innovative new ideas
throughout the industry.
```
To be certain, these 10 projects and the sampling of their achievements that
I’ve highlighted here do not remotely encompass the full breadth of the
history that has led the industry to where it exists today. But they stand out to
me as important and noteworthy milestones along the way, which taken
together paint an informative picture of the evolution of stream processing
over the past decade and a half. We’ve come a long way since the early days
of MapReduce, with a number of ups, downs, twists, and turns along the way.
Even so, there remains a long road of open problems ahead of us in the realm
of streaming systems. I’m excited to see what the future holds.

Which means I’m skipping a ton of the academic literature around stream
processing, because that’s where much of it started. If you’re really into
hardcore academic papers on the topic, start from the references in “The
Dataflow Model” paper and work backward. You should be able to find your
way pretty easily.

Certainly, MapReduce itself was built upon many ideas that had been well
known before, as is even explicitly stated in the MapReduce paper. That
doesn’t change the fact that MapReduce was the system that tied those ideas
together (along with some of its own) to create something practical that
solved an important and emerging problem better than anyone else before
ever had, and in a way that inspired generations of data-processing systems
that followed.

To be clear, Google was most certainly not the only company tackling data
processing problems at this scale at the time. Google was just one among a

1

2

3


number of companies involved in that first generation of attempts at taming
massive-scale data processing.

And to be clear, MapReduce actually built upon the Google File System,
GFS, which itself solved the scalability and fault-tolerance issues for a
specific subset of the overall problem.

Not unlike the query optimizers long used in the database world.
Noogler == New + Googler == New hires at Google
As an aside, I also highly recommend reading Martin Kleppmann’s “A
Critique of the CAP Theorem” for very nice analysis of the shortcomings of
the CAP theorem itself, as well as a more principled alternative way of
looking at the same problem.

For the record, written primarily by Sam McVeety with help from Reuven
and bits of input from the rest of us on the author list; we shouldn’t have
alphabetized that author list, because everyone always assumes I’m the
primary author on it, even though I wasn’t.

Kafka Streams and now KSQL are of course changing that, but those are
relatively recent developments, and I’ll be focusing primarily on the Kafka of
yore.

While I recommend the book as the most comprehensive and cohesive
resource, you can find much of the content from it scattered across O’Reilly’s
website if you just search around for Kreps’ articles. Sorry, Jay...

As with many broad generalizations, this one is true in a specific context,
but belies the underlying complexity of reality. As I alluded to in Chapter 1,
batch systems go to great lengths to optimize the cost and runtime of data
processing pipelines over bounded datasets in ways that stream processing
engines have yet to attempt to duplicate. To imply that modern batch and
streaming systems only differ in one small way is a sizeable
oversimplification in any realm beyond the purely conceptual.

There’s an additional subtlety here that’s worth calling out: even as runners
adopt new semantics and tick off feature checkboxes, it’s not the case that

4 5 6 7 8 9

10

11

12


you can blindly choose any runner and have an identical experience. This is
because the runners themselves can still vary greatly in their runtime and
operational characteristics. Even for cases in which two given runners
implement the same set of semantic features within the Beam Model, the way
they go about executing those features at runtime is typically very different.
As a result, when building a Beam pipeline, it’s important to do your
homework regarding various runners, to ensure that you choose a runtime
platform that serves your use case best.


