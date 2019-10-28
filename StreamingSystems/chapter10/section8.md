 ## Cloud Dataflow

Cloud Dataflow (Figure 10-26) is Google’s fully managed, cloud-based data
processing service. Dataflow launched to the world in August 2015. It was
built with the intent to take the decade-plus of experiences that had gone into
building MapReduce, Flume, and MillWheel, and package them up into a
serverless cloud experience.


```
Figure 10-26. Timeline: Cloud Dataflow
```
Although the serverless aspect of Cloud Dataflow is perhaps its most
technically challenging and distinguishing factor from a systems perspective,
the primary contribution to streaming systems that I want to discuss here is its
unified batch plus streaming programming model. That’s all the
transformations, windowing, watermarks, triggers, and accumulation
goodness we’ve spent most of the book talking about. And all of them, of
course, wrapped up the _what_ / _where_ / _when_ / _how_ way of thinking about things.

The model first arrived back in Flume, as we looked to incorporate the robust
out-of-order processing support in MillWheel into the higher-level
programming model Flume afforded. The combined batch and streaming
approach available to Googlers internally with Flume was then the basis for
the fully unified model included in Dataflow.

The key insight in the unified model—the full extent of which none of us at
the time even truly appreciated—is that under the covers, batch and streaming
are really not that different: they’re both just minor variations on the streams
and tables theme. As we learned in Chapter 6, the main difference really boils
down to the ability to incrementally trigger tables into streams; everything
else is conceptually the same. By taking advantage of the underlying
commonalities of the two approaches, it was possible to provide a single,
nearly seamless experience that applied to both worlds. This was a big step
forward in making stream processing more accessible.

```
11
```

In addition to taking advantage of the commonalities between batch and
streaming, we took a long, hard look at the variety of use cases we’d
encountered over the years at Google and used those to inform the pieces that
went into the unified model. Key aspects we targeted included the following:

```
Unaligned, event-time windows such as sessions, providing the
ability to concisely express powerful analytic constructs and apply
them to out-of-order data.
Custom windowing support , because one (or even three or four) sizes
rarely fit all.
```
```
Flexible triggering and accumulation modes , providing the ability to
shape the way data flow through the pipeline to match the
correctness, latency, and cost needs of the given use case.
```
```
The use of watermarks for reasoning about input completeness ,
which is critical for use cases like anomalous dip detection where the
analysis depends upon an absence of data.
```
```
Logical abstraction of the underlying execution environment, be it
batch, microbatch, or streaming, providing flexibility of choice in
execution engine and avoiding system-level constructs (such as
micro-batch size) from creeping into the logical API.
```
Taken together, these aspects provided the flexibility to balance the tensions
between correctness, latency, and cost, allowing the model to be applied
across a wide breadth of use cases.


```
Figure 10-27. Dataflow Model paper
```
Given that you’ve just read an entire book covering the finer points of the
Dataflow/Beam Model, there’s little point in trying to retread any those
concepts here. However, if you’re looking for a slightly more academic take
on things as well as a nice overview of some of the motivating use cases
alluded to earlier, you might find our 2015 Dataflow Model paper worthwhile
(Figure 10-27).

Though there are many other compelling aspects to Cloud Dataflow, the
important contribution from the perspective of this chapter is its _unified batch
plus streaming programming model_. It brought the world a comprehensive
approach to tackling unbounded, out-of-order datasets, and in a way that
provided the flexibility to make the trade-offs necessary to balance the


tensions between correctness, latency, and cost to match the requirements for
a given use case.
