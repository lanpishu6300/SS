 ## Flink

Flink (Figure 10-28) burst onto the scene in 2015, rapidly transforming itself
from a system that almost no one had heard of into one of the powerhouses of
the streaming world, seemingly overnight.

```
Figure 10-28. Timeline: Flink
```
There were two main reasons for Flink’s rise to prominence:

```
Its rapid adoption of the Dataflow/Beam programming model , which
put it in the position of being the most semantically capable fully
open source streaming system on the planet at the time.
Followed shortly thereafter by its highly efficient snapshotting
implementation (derived from research in Chandy and Lamport’s
original paper “Distributed Snapshots: Determining Global States of
Distributed Systems” [Figure 10-29]), which gave it the strong
consistency guarantees needed for correctness.
```

```
Figure 10-29. Chandy-Lamport snapshots
```
Reuven covered Flink’s consistency mechanism briefly in Chapter 5, but to
reiterate, the basic idea is that periodic barriers are propagated along the
communication paths between workers in the system. The barriers act as an
alignment mechanism between the various distributed workers producing
data upstream from a consumer. When the consumer receives a given barrier
on all of its input channels (i.e., from all of its upstream producers), it
checkpoints its current progress for all active keys, at which point it is then
safe to acknowledge processing of all data that came before the barrier. By
tuning how frequently barriers are sent through the system, it’s possible to
tune the frequency of checkpointing and thus trade off increased latency (due
to the need for side effects to be materialized only at checkpoint times) in
exchange for higher throughput.


```
Figure 10-30. “Extending the Yahoo! Streaming Benchmark”
```
The simple fact that Flink now had the capability to provide exactly-once
semantics along with native support for event-time processing was huge at
the time. But it wasn’t until Jamie Grier published his article titled
“Extending the Yahoo! Streaming Benchmark” (Figure 10-30) that it became
clear just how performant Flink was. In that article, Jamie described two
impressive achievements:


```
1. Building a prototype Flink pipeline that achieved greater accuracy
than one of Twitter’s existing Storm pipelines (thanks to Flink’s
exactly-once semantics) at 1% of the cost of the original.
2. Updating the Yahoo! Streaming Benchmark to show Flink (with
exactly-once) achieving 7.5 times the throughput of Storm (without
exactly-once). Furthermore, Flink’s performance was shown to be
limited due to network saturation; removing the network bottleneck
allowed Flink to achieve almost 40 times the throughput of Storm.
```
Since then, numerous other projects (notably, Storm and Apex) have all
adopted the same type of consistency mechanism.


```
Figure 10-31. “Savepoints: Turning Back Time”
```
With the addition of a snapshotting mechanism, Flink gained the strong
consistency needed for end-to-end exactly-once. But to its credit, Flink went
one step further, and used the global nature of its snapshots to provide the
ability to restart an entire pipeline from any point in the past, a feature known
as savepoints (described in the “Savepoints: Turning Back Time” post by
Fabian Hueske and Michael Winters [Figure 10-31]). The savepoints feature
took the warm fuzziness of durable replay that Kafka had applied to the


streaming transport layer and extended it to cover the breadth of an entire
pipeline. Graceful evolution of a long-running streaming pipeline over time
remains an important open problem in the field, with lots of room for
improvement. But Flink’s savepoints feature stands as one of the first huge
steps in the right direction, and one that remains unique across the industry as
of this writing.

If you’re interested in learning more about the system constructs underlying
Flink’s snapshots and savepoints, the paper “State Management in Apache
Flink” (Figure 10-32) discusses the implementation in good detail.

```
Figure 10-32. “State Management in Apache Flink”
```
Beyond savepoints, the Flink community has continued to innovate,


including bringing the first practical streaming SQL API to market for a
large-scale, distributed stream processing engine, as we discussed in
Chapter 8.

In summary, Flink’s rapid rise to stream processing juggernaut can be
attributed primarily to three characteristics of its approach: 1) incorporating
the _best existing ideas_ from across the industry (e.g., being the first open
source adopter of the Dataflow/Beam Model), 2) _bringing its own
innovations_ to the table to push forward the state of the art (e.g., strong
consistency via snapshots and savepoints, streaming SQL), and 3) doing both
of those things _quickly_ and _repeatedly_. Add in the fact that all of this is done
in _open source_ , and you can see why Flink has consistently continued to raise
the bar for streaming processing across the industry.

## Beam

The last system we talk about is Apache Beam (Figure 10-33). Beam differs
from most of the other systems in this chapter in that it’s primarily a
programming model, API, and portability layer, not a full stack with an
execution engine underneath. But that’s exactly the point: just as SQL acts as
a lingua franca for declarative data processing, Beam aims to be the lingua
franca for programmatic data processing. Let’s explore how.

```
Figure 10-33. Timeline: Beam
```

Concretely, Beam is composed a number of components:

```
A unified batch plus streaming programming model , inherited from
Cloud Dataflow where it originated, and the finer points of which
we’ve spent the majority of this book discussing. The model is
independent of any language implementations or runtime systems.
You can think of this as Beam’s equivalent to SQL’s relational
algebra.
A set of SDKs (software development kits) that implement that
model, allowing pipelines to be expressed in terms of the model in
idiomatic ways for a given language. Beam currently provides SDKs
in Java, Python, and Go. You can think of these as Beam’s
programmatic equivalents to the SQL language itself.
```
```
A set of DSLs (domain specific languages) that build upon the
SDKs, providing specialized interfaces that capture pieces of the
model in unique ways. Whereas SDKs are required to surface all
aspects of the model, DSLs can expose only those pieces that make
sense for the specific domain a DSL is targeting. Beam currently
provides a Scala DSL called Scio and an SQL DSL, both of which
layer on top of the existing Java SDK.
```
```
A set of runners that can execute Beam pipelines. Runners take the
logical pipeline described in Beam SDK terms, and translate them as
efficiently as possible into a physical pipeline that they can then
execute. Beam runners exist currently for Apex, Flink, Spark, and
Google Cloud Dataflow. In SQL terms, you can think of these
runners as Beam’s equivalent to the various SQL database
implementations, such as Postgres, MySQL, Oracle, and so on.
```
The core vision for Beam is built around its value as a portability layer, and
one of the more compelling features in that realm is its planned support for
full cross-language portability. Though not yet fully complete (but landing
imminently), the plan is for Beam to provide sufficiently performant
abstraction layers between SDKs and runners that will allow for a full cross-


product of SDK × runner matchups. In such a world, a pipeline written in a
JavaScript SDK could seamlessly execute on a runner written in Haskell,
even if the Haskell runner itself had no native ability to execute JavaScript
code.

As an abstraction layer, the way that Beam positions itself relative to its
runners is critical to ensure that Beam actually brings value to the
community, rather than introducing just an unnecessary layer of abstraction.
The key point here is that Beam aims to never be just the intersection (lowest
common denominator) or union (kitchen sink) of the features found in its
runners. Instead, it aims to include only the best ideas across the data
processing community at large. This allows for innovation in two
dimensions:

Innovation in Beam

```
Figure 10-34. Powerful and modular I/O
```
```
Beam might include API support for runtime features that not all runners
initially support. This is okay. Over time, we expect many runners will
```

```
incorporate such features into future versions; those that don’t will be a
less-attractive runner choice for use cases that need such features.
An example here is Beam’s SplittableDoFn API for writing composable,
scalable sources (described by Eugene Kirpichov in his post “Powerful
and modular I/O connectors with Splittable DoFn in Apache Beam”
[Figure 10-34]). It’s both unique and extremely powerful but also does
not yet see broad support across all runners for some of the more
innovative parts like dynamic work rebalancing. Given the value such
features bring, however, we expect that will change over time.
```
Innovation in runners

```
Runners might introduce runtime features for which Beam does not
initially provide API support. This is okay. Over time, runtime features
that have proven their usefulness will have API support incorporated into
Beam.
An example here is the state snapshotting mechanism in Flink, or
savepoints, which we discussed earlier. Flink is still the only publicly
available streaming system to support snapshots in this way, but there’s a
proposal in Beam to provide an API around snapshots because we believe
graceful evolution of pipelines over time is an important feature that will
be valuable across the industry. If we were to magically push out such an
API today, Flink would be the only runtime system to support it. But
again, that’s okay. The point here is that the industry as a whole will
begin to catch up over time as the value of these features becomes clear.
And that’s better for everyone.
```
By encouraging innovation within both Beam itself as well as runners, we
hope to push forward the capabilities of the entire industry at a greater pace
over time, without accepting compromises along the way. And by delivering
on the promise of portability across runtime execution engines, we hope to
establish Beam as the common language for expressing programmatic data
processing pipelines, similar to how SQL exists today as the common
currency of declarative data processing. It’s an ambitious goal, and as of
writing, we’re still a ways off from seeing it fully realized, but we’ve also

```
12
```

come a long way so far.

