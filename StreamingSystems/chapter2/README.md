Chapter 2. The What , Where ,

# When , and How of Data

# Processing

Okay party people, it’s time to get concrete!

Chapter 1 focused on three main areas: _terminology_ , defining precisely what I
mean when I use overloaded terms like “streaming”; _batch versus streaming_ ,
comparing the theoretical capabilities of the two types of systems, and
postulating that only two things are necessary to take streaming systems
beyond their batch counterparts: correctness and tools for reasoning about
time; and _data processing patterns_ , looking at the conceptual approaches
taken with both batch and streaming systems when processing bounded and
unbounded data.

In this chapter, we’re now going to focus further on the data processing
patterns from Chapter 1, but in more detail, and within the context of
concrete examples. By the time we’re finished, we’ll have covered what I
consider to be the core set of principles and concepts required for robust out-
of-order data processing; these are the tools for reasoning about time that
truly get you beyond classic batch processing.

To give you a sense of what things look like in action, I use snippets of
Apache Beam code, coupled with time-lapse diagrams to provide a visual
representation of the concepts. Apache Beam is a unified programming
model and portability layer for batch and stream processing, with a set of
concrete SDKs in various languages (e.g., Java and Python). Pipelines written
with Apache Beam can then be portably run on any of the supported
execution engines (Apache Apex, Apache Flink, Apache Spark, Cloud
Dataflow, etc.).

I use Apache Beam here for examples not because this is a Beam book (it’s

```
1
```

not), but because it most completely embodies the concepts described in this
book. Back when “Streaming 102” was originally written (back when it was
still the Dataflow Model from Google Cloud Dataflow and not the Beam
Model from Apache Beam), it was literally the only system in existence that
provided the amount of expressiveness necessary for all the examples we’ll
cover here. A year and a half later, I’m happy to say much has changed, and
most of the major systems out there have moved or are moving toward
supporting a model that looks a lot like the one described in this book. So rest
assured that the concepts we cover here, though informed through the Beam
lens, as it were, will apply equally across most other systems you’ll come
across.

