 ## Spark

Moving on, we now come to Apache Spark (Figure 10-19). This is another
section in which I’m going to greatly oversimplify the total impact that Spark
has had on the industry by focusing on a specific portion of its contributions:
those within the realm of stream processing. Apologies in advance.

```
Figure 10-19. Timeline: Spark
```
Spark got its start at the now famous AMPLab in UC Berkeley around 2009.
The thing that initially fueled Spark’s fame was its ability to oftentimes
perform the bulk of a pipeline’s calculations entirely in memory, without
touching disk until the very end. Engineers achieved this via the Resilient
Distributed Dataset (RDD) idea, which basically captured the full lineage of
data at any given point in the pipeline, allowing intermediate results to be
recalculated as needed on machine failure, under the assumptions that a) your
inputs were always replayable, and b) your computations were deterministic.
For many use cases, these preconditions were true, or at least true enough
given the massive gains in performance users were able to realize over
standard Hadoop jobs. From there, Spark gradually built up its eventual
reputation as Hadoop’s de facto successor.


A few years after Spark was created, Tathagata Das, then a graduate student
in the AMPLab, came to the realization that: hey, we’ve got this fast batch
processing engine, what if we just wired things up so we ran multiple batches
one after another, and used that to process streaming data? From that bit of
insight, Spark Streaming was born.

What was really fantastic about Spark Streaming was this: thanks to the
strongly consistent batch engine powering things under the covers, the world
now had a stream processing engine that could provide correct results all by
itself without needing the help of an additional batch job. In other words,
given the right use case, you could ditch your Lambda Architecture system
and just use Spark Streaming. All hail Spark Streaming!

The one major caveat here was the “right use case” part. The big downside to
the original version of Spark Streaming (the 1.x variants) was that it provided
support for only a specific flavor of stream processing: processing-time
windowing. So any use case that cared about event time, needed to deal with
late data, and so on, couldn’t be handled out of the box without a bunch of
extra code being written by the user to implement some form of event-time
handling on top of Spark’s processing-time windowing architecture. This
meant that Spark Streaming was best suited for in-order data or event-time-
agnostic computations. And, as I’ve reiterated throughout this book, those
conditions are not as prevalent as you would hope when dealing with the
large-scale, user-centric datasets common today.

Another interesting controversy that surrounds Spark Streaming is the age-
old “microbatch versus true streaming” debate. Because Spark Streaming is
built upon the idea of small, repeated runs of a batch processing engine,
detractors claim that Spark Streaming is not a true streaming engine in the
sense that progress in the system is gated by the global barriers of each batch.
There’s some amount of truth there. Even though true streaming engines
almost always utilize some sort of batching or bundling for the sake of
throughput, they have the flexibility to do so at much finer-grained levels,
down to individual keys. The fact that microbatch architectures process
bundles at a global level means that it’s virtually impossible to have both low


per-key latency and high overall throughput, and there are a number of
benchmarks that have shown this to be more or less true. But at the same
time, latency on the order of minutes or multiple seconds is still quite good.
And there are very few use cases that demand exact correctness and such
stringent latency capabilities. So in some sense, Spark was absolutely right to
target the audience it did originally; most people fall in that category. But that
hasn’t stopped its competitors from slamming this as a massive disadvantage
for the platform. Personally, I see it as a minor complaint at best in most
cases.

Shortcomings aside, Spark Streaming was a watershed moment for stream
processing: the first publicly available, large-scale stream processing engine
that could also provide the correctness guarantees of a batch system. And of
course, as previously noted, streaming is only a very small part of Spark’s
overall success story, with important contributions made in the space of
iterative processing and machine learning, its native SQL integration, and the
aforementioned lightning-fast in-memory performance, to name a few.

If you’re curious to learn more about the details of the original Spark 1.x
architecture, I highly recommend Matei Zaharia’s dissertation on the subject,
“An Architecture for Fast and General Data Processing on Large Clusters”
(Figure 10-20). It’s 113 pages of Sparky goodness that’s well worth the
investment.


```
Figure 10-20. Spark dissertation
```
As of today, the 2.x variants of Spark are greatly expanding upon the
semantic capabilities of Spark Streaming, incorporating many parts of the
model described in this book, while attempting to simplify some of the more
complex pieces. And Spark is even pushing a new true streaming
architecture, to try to shut down the microbatch naysayer arguments. But
when it first came on the scene, the important contribution that Spark brought
to the table was the fact that it was the _first publicly available stream
processing engine with strong consistency semantics_ , albeit only in the case
of in-order data or event-time-agnostic computation.


