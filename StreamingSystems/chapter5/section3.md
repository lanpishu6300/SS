 ## Addressing Determinism

Making this strategy work in the real world requires a lot of care, however.
One immediate wrinkle is that the Beam Model allows for user code to
produce nondeterministic output. This means that a ParDo can execute twice
on the same input record (due to a retry), yet produce different output on each
retry. The desired behavior is that only one of those outputs will commit into
the pipeline; however, the nondeterminism involved makes it difficult to
guarantee that both outputs have the same deterministic ID. Even trickier, a
ParDo can output multiple records, so each of these retries might produce a
different number of outputs!

So, why don’t we simply require that all user processing be deterministic?

```
4
```

Our experience is that in practice, many pipelines require nondeterministic
transforms And all too often, pipeline authors do not realize that the code
they wrote is nondeterministic. For example, consider a transform that looks
up supplemental data in Cloud Bigtable in order to enrich its input data. This
is a nondeterministic task, as the external value might change in between
retries of the transform. Any code that relies on current time is likewise not
deterministic. We have also seen transforms that need to rely on random
number generators. And even if the user code is purely deterministic, any
event-time aggregation that allows for late data might have nondeterministic
inputs.

Dataflow addresses this issue by using checkpointing to make
nondeterministic processing effectively deterministic. Each output from a
transform is checkpointed, together with its unique ID, to stable storage
_before_ being delivered to the next stage. Any retries in the shuffle delivery
simply replay the output that has been checkpointed—the user’s
nondeterministic code is not run again on retry. To put it another way, the
user’s code may be run multiple times but only one of those runs can “win.”
Furthermore, Dataflow uses a consistent store that allows it to prevent
duplicates from being written to stable storage.
