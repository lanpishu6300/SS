 ## Kafka


We now come to Kafka (Figure 10-23). Kafka is unique among the systems
discussed in this chapter in that it’s not a data processing framework, but
instead a transport layer. Make no mistake, however: Kafka has played one of
the most influential roles in advancing stream processing out of all the
system’s we’re discussing here.

```
Figure 10-23. Timeline: Kafka
```
If you’re not familiar with it, Kafka is essentially a persistent streaming
transport, implemented as a set of partitioned logs. It was developed
originally at LinkedIn by such industry luminaries as Neha Narkhede and Jay
Kreps, and its accolades include the following:

```
Providing a clean model of persistence that packaged that warm
fuzzy feeling of durable , replayable input sources from the batch
world in a streaming friendly interface.
```
```
Providing an elastic isolation layer between producers and
consumers.
Embodying the relationship between streams and tables that we
discussed in Chapter 6, revealing a foundational way of thinking
about data processing in general while also providing a conceptual
link to the rich and storied world of databases.
```
```
As of side of effect of all of the above, not only becoming the
```
```
9
```

```
cornerstone of a majority of stream processing installations across
the industry, but also fostering the stream-processing-as-databases
and microservices movements.
```
They must get up very early in the morning.

Of those accolades, there are two that stand out most to me. The first is the
application of durability and replayability to stream data. Prior to Kafka, most
stream processing systems used some sort of ephemeral queuing system like
Rabbit MQ or even plain-old TCP sockets to send data around. Durability
might be provided to some degree via upstream backup in the producers (i.e.,
the ability for upstream producers of data to resend if the downstream
workers crashed), but oftentimes the upstream data was stored ephemerally,
as well. And most approaches entirely ignored the idea of being able to
replay input data later in cases of backfills or for prototyping, development,
and regression testing.

Kafka changed all that. By taking the battle-hardened concept of a durable
log from the database world and applying it to the realm of stream
processing, Kafka gave us all back that sense of safety and security we’d lost
when moving from the durable input sources common in the Hadoop/batch
world to the ephemeral sources prevalent at the time in the streaming world.
With durability and replayability, stream processing took yet another step
toward being a robust, reliable replacement for the ad hoc, continuous batch
processing systems of yore that were still being applied to streaming use
cases.

As a streaming system developer, one of the more interesting visible artifacts
of the impact that Kafka’s durability and replayability features have had on
the industry is how many of the stream processing engines today have grown
to fundamentally rely on that replayability to provide end-to-end exactly-once
guarantees. Replayability is the foundation upon which end-to-end exactly-
once guarantees in Apex, Flink, Kafka Streams, Spark, and Storm are all
built. When executing in exactly-once mode, each of those systems
assumes/requires that the input data source be able to rewind and replay all of
the data up until the most recent checkpoint. When used with an input source


that does not provide such ability (even if the source can guarantee reliable
delivery via upstream backup), end-to-end exactly-once semantics fall apart.
That sort of broad reliance on replayability (and the related aspect of
durability) is a huge testament to the amount of impact those features have
had across the industry.

The second noteworthy bullet from Kafka’s resume is the popularization of
stream and table theory. We spent the entirety of Chapter 6 discussing
streams and tables as well as much of Chapters 8 and 9. And for good reason.
Streams and tables form the foundation of data processing, be it the
MapReduce family tree of systems, the enormous legacy of SQL database
systems, or what have you. Not all data processing approaches need speak
directly in terms of streams and tables but conceptually speaking, that’s how
they all operate. And as both users and developers of these systems, there’s
great value in understanding the core underlying concepts that all of our
systems build upon. We all owe a collective thanks to the folks in the Kafka
community who helped shine a broader light on the streams-and-tables way
of thinking.


```
Figure 10-24. I ❤ Logs
```
If you’d like to learn more about Kafka and the foundations it’s built on, _I_ ❤
_Logs_ by Jay Kreps (O’Reilly; Figure 10-24) is an excellent resource.
Additionally, as cited originally in Chapter 6, Kreps and Martin Kleppmann
have a pair of articles (Figure 10-25) that I highly recommend for reading up
on the origins of streams and table theory.

Kafka has made huge contributions to the world of stream processing,

```
10
```

arguably more than any other single system out there. In particular, the
application of durability and replayability to input and output streams played
a big part in helping move stream processing out of the niche realm of
approximation tools and into the big leagues of general data processing.
Additionally, the theory of streams and tables, popularized by the Kafka
community, provides deep insight into the underlying mechanics of data
processing in general.

```
Figure 10-25. Martin’s post (left) and Jay’s post (right)
```
