 ## MillWheel

Next we discuss MillWheel, a project that I first dabbled with in my 20%
time after joining Google in 2008, later joining the team full time in 2010
(Figure 10-21).

```
Figure 10-21. Timeline: MillWheel
```
MillWheel is Google’s original, general-purpose stream processing
architecture, and the project was founded by Paul Nordstrom around the time
Google’s Seattle office opened. MillWheel’s success within Google has long
centered on an ability to provide low-latency, strongly consistent processing
of unbounded, out-of-order data. Over the course of this book, we’ve looked
at most of the bits and pieces that came together in MillWheel to make this
possible:

```
Reuven discussed exactly-once guarantees in Chapter 5. Exactly-
once guarantees are essential for correctness.
```
```
In Chapter 7 we looked at persistent state , the strongly consistent
variations of which provide the foundation for maintaining that
correctness in long-running pipelines executing on unreliable
hardware.
Slava talked about watermarks in Chapter 3. Watermarks provide a
```

```
foundation for reasoning about disorder in input data.
```
```
Also in Chapter 7, we looked at persistent timers , which provide the
necessary link between watermarks and the pipeline’s business logic.
```
It’s perhaps somewhat surprising then to note that the MillWheel project was
not initially focused on correctness. Paul’s original vision more closely
targeted the niche that Storm later espoused: low-latency data processing with
weak consistency. It was the initial MillWheel customers, one building
sessions over search data and another performing anomaly detection on
search queries (the Zeitgeist example from the MillWheel paper), who drove
the project in the direction of correctness. Both had a strong need for
consistent results: sessions were used to infer user behavior, and anomaly
detection was used to infer trends in search queries; the utility of both
decreased significantly if the data they provided were not reliable. As a result,
MillWheel’s direction was steered toward one of strong consistency.

Support for out-of-order processing, which is the other core aspect of robust
streaming often attributed to MillWheel, was also motivated by customers.
The Zeitgeist pipeline, as a true streaming use case, wanted to generate an
output stream that identified anomalies in search query traffic, and only
anomalies (i.e., it was not practical for consumers of its analyses to poll all
the keys in a materialized view output table waiting for an anomaly to be
flagged; consumers needed a direct signal only when anomalies happened for
specific keys). For anomalous spikes (i.e., _increases_ in query traffic), this is
relatively straightforward: when the count for a given query exceeds the
expected value in your model for that query by some statistically significant
amount, you can signal an anomaly. But for anomalous dips (i.e., _decreases_
in query traffic), the problem is a bit trickier. It’s not enough to simply see
that the number of queries for a given search term has decreased, because for
any period of time, the observed number always starts out at zero. What you
really need to do in these cases is wait until you have reason to believe that
you’ve seen a sufficiently representative portion of the input for a given time
period, and only _then_ compare the count against your model.


### TRUE STREAMING

“True streaming use case” bears a bit of explanation. One recent trend in
streaming systems is to try to simplify the programming models to make
them more accessible by limiting the types of use cases one can address.
For example, at the time of writing, both Spark’s Structured Streaming
and Apache Kafka’s Kafka Streams systems limit themselves to what I
refer to in Chapter 8 as “materialized view semantics,” essentially
repeated updates to an eventually consistent output table. Materialized
view semantics are great when you want to consume your output as a
lookup table: any time you can just lookup a value in that table and be
okay with the latest result as of query time, materialized views are a good
fit. They are not, however, particularly well suited for use cases in which
you want to consume your output as a bonafide stream. I refer to these as
true streaming use cases, with anomaly detection being one of the better
examples.

As we’ll discuss shortly, there are certain aspects of anomaly detection
that make it unsuitable for pure materialized view semantics (i.e., record-
by-record processing only), specifically the fact that it relies on reasoning
about the completeness of the input data to accurately identify anomalies
that are the result of an absence of data (in addition to the fact that polling
an output table to see if an anomaly signal has arrived is not an approach
that scales particularly well). True streaming use cases are thus the
motivation for features like watermarks (Preferably _low_ watermarks that
pessimistically track input completeness, as described in Chapter 3, not
_high_ watermarks that track the event time of the newest record the system
is aware of, as used by Spark Structured Streaming for garbage collecting
windows, since high watermarks are more prone to incorrectly throwing
away data as event time skew varies within the pipeline) and triggers.
Systems that omit these features do so for the sake of simplicity but at the
cost of decreased ability. There can be great value in that, most certainly,
but don’t be fooled if you hear such systems claim these simplifications
yield equivalent or even greater generality; you can’t address fewer use
cases and be equally or more general.


The Zeitgeist pipeline first attempted to do this by inserting processing-time
delays before the analysis logic that looked for dips. This would work
reasonably decently when data arrived in order, but the pipeline’s authors
discovered that data could, at times, be greatly delayed and thus arrive wildly
out of order. In these cases, the processing-time delays they were using
weren’t sufficient, because the pipeline would erroneously report a flurry of
dip anomalies that didn’t actually exist. What they really needed was a way to
wait until the input became complete.

Watermarks were thus born out of this need for reasoning about input
completeness in out-of-order data. As Slava described in Chapter 3, the basic
idea was to track the known progress of the inputs being provided to the
system, using as much or as little data available for the given type of data
source, to construct a progress metric that could be used to quantify input
completeness. For simpler input sources like a statically partitioned Kafka
topic with each partition being written to in increasing event-time order (such
as by web frontends logging events in real time), you can compute a perfect
watermark. For more complex input sources like a dynamic set of input logs,
a heuristic might be the best you can do. But either way, watermarks provide
a distinct advantage over the alternative of using processing time to reason
about event-time completeness, which experience has shown serves about as
well as a map of London while trying to navigate the streets of Cairo.

So thanks to the needs of its customers, MillWheel ended up as a system with
the right set of features for supporting robust stream processing on out-of-
order data. As a result, the paper titled “MillWheel: Fault-Tolerant Stream
Processing at Internet Scale” (Figure 10-22) spends most of its time
discussing the difficulties of providing correctness in a system like this, with
consistency guarantees and watermarks being the main areas of focus. It’s
well worth your time if you’re interested in the subject.

```
8
```

```
Figure 10-22. MillWheel paper
```
Not long after the MillWheel paper was published, MillWheel was integrated
as an alternative, streaming backend for Flume, together often referred to as
Streaming Flume. Within Google today, MillWheel is in the process of being
replaced by its successor, Windmill (the execution engine that also powers
Cloud Dataflow, discussed in a moment), a ground-up rewrite that
incorporates all the best ideas from MillWheel, along with a few new ones
like better scheduling and dispatch, and a cleaner separation of user and
system code.

However, the big takeaway for MillWheel is that the four concepts listed
earlier (exactly-once, persistent state, watermarks, persistent timers) together
provided the basis for a system that was finally able to deliver on the true
promise of stream processing: robust, low-latency processing of out-of-order
data, even on unreliable commodity hardware.


