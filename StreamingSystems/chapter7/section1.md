 ## Motivation

To begin, let’s more precisely motivate persistent state. We know from
Chapter 6 that grouping is what gives us tables. And the core of what I
postulated at the beginning of this chapter was correct: the point of persisting
these tables is to capture the otherwise ephemeral data contained therein. But
why is that necessary?

## The Inevitability of Failure

The answer to that question is most clearly seen in the case of processing
unbounded input data, so we’ll start there. The main issue is that pipelines
processing unbounded data are effectively intended to run forever. But
running forever is a far more demanding Service-Level Objective than can be
achieved by the environments in which these pipelines typically execute.
Long-running pipelines will inevitably see interruptions thanks to machine
failures, planned maintenance, code changes, and the occasional
misconfigured command that takes down an entire cluster of production
pipelines. To ensure that they can resume where they left off when these
kinds of things happen, long-running pipelines need some sort of durable
recollection of where they were before the interruption. That’s where
persistent state comes in.

Let’s expand on that idea a bit beyond unbounded data. Is this only relevant
in the unbounded case? Do batch pipelines use persistent state, and why or
why not? As with nearly every other batch-versus-streaming question we’ve
come across, the answer has less to do with the nature of batch and streaming
systems themselves (perhaps unsurprising given what we learned in
Chapter 6), and more to do with the types of datasets they historically have
been used to process.

Bounded datasets by nature are finite in size. As a result, systems that process
bounded data (historically batch systems) have been tailored to that use case.
They often assume that the input can be reprocessed in its entirety upon
failure. In other words, if some piece of the processing pipeline fails and if
the input data are still available, we can simply restart the appropriate piece


of the processing pipeline and let it read the same input again. This is called
_reprocessing the input_.

They might also assume failures are infrequent and thus optimize for the
common case by persisting as little as possible, accepting the extra cost of
recomputation upon failure. For particularly expensive, multistage pipelines,
there might be some sort of per-stage global checkpointing that allows for
more efficiently resuming execution (typically as part of a shuffle), but it’s
not a strict requirement and might not be present in many systems.

Unbounded datasets, on the other hand, must be assumed to have infinite
size. As a result, systems that process unbounded data (historically streaming
systems) have been built to match. They never assume that all of the data will
be available for reprocessing, only some known subset of it. To provide at-
least-once or exactly-once semantics, any data that are no longer available for
reprocessing must be accounted for in durable checkpoints. And if at-most-
once is all you’re going for, you don’t need checkpointing.

At the end of the day, there’s nothing batch- or streaming-specific about
persistent state. State can be useful in both circumstances. It just happens to
be critical when processing unbounded data, so you’ll find that streaming
systems typically provide more sophisticated support for persistent state.

## Correctness and Efficiency

Given the inevitability of failures and the need to cope with them, persistent
state can be seen as providing two things:

```
A basis for correctness in light of ephemeral inputs. When
processing bounded data, it’s often safe to assume inputs stay around
forever; with unbounded data, this assumption typically falls short
of reality. Persistent state allows you to keep around the intermediate
bits of information necessary to allow processing to continue when
the inevitable happens, even after your input source has moved on
and forgotten about records it gave you previously.
```
```
A way to minimize work duplicated and data persisted as part of
```
```
1
```

coping with failures. Regardless of whether your inputs are
ephemeral, when your pipeline experiences a machine failure, any
work on the failed machine that wasn’t checkpointed somewhere
must be redone. Depending upon the nature of the pipeline and its
inputs, this can be costly in two dimensions: the amount of work
performed during reprocessing, and the amount of input data stored
to support reprocessing.

Minimizing duplicated work is relatively straightforward. By
checkpointing partial progress within a pipeline (both the
intermediate results computed as well as the current location within
the input as of checkpointing time), it’s possible to greatly reduce
the amount of work repeated when failures occur because none of
the operations that came before the checkpoint need to be replayed
from durable inputs. Most commonly, this involves data at rest (i.e.,
tables), which is why we typically refer to persistent state in the
context of tables and grouping. But there are persistent forms of
streams (e.g., Kafka and its relatives) that serve this function, as
well.

Minimizing the amount of data persisted is a larger discussion, one
that will consume a sizeable chunk of this chapter. For now, at least,
suffice it to say that, for many real-world use cases, rather than
remembering all of the raw inputs within a checkpoint for any given
stage in the pipeline, it’s often practical to instead remember some
partial, intermediate form of the ongoing calculation that consumes
less space than all of the original inputs (for example, when
computing a mean, the total sum and the count of values seen are
much more compact than the complete list of values contributing to
that sum and count). Not only can checkpointing these intermediate
data drastically reduce the amount of data that you need to remember
at any given point in the pipeline, it also commensurately reduces
the amount of reprocessing needed for that specific stage to recover
from a failure.

Furthermore, by intelligently garbage-collecting those bits of


```
persistent state that are no longer needed (i.e., state for records
which are known to have been processed completely by the pipeline
already), the amount of data stored in persistent state for a given
pipeline can be kept to a manageable size over time, even when the
inputs are technically infinite. This is how pipelines processing
unbounded data can continue to run effectively forever, while still
providing strong consistency guarantees but without a need for
complete recall of the original inputs to the pipeline.
```
At the end of the day, persistent state is really just a means of providing
correctness and efficient fault tolerance in data processing pipelines. The
amount of support needed in either of those dimensions depends greatly upon
the natures of the inputs to the pipeline and the operations being performed.
Unbounded inputs tend to require more correctness support than bounded
inputs. Computationally expensive operations tend to demand more
efficiency support than computationally cheap operations.

