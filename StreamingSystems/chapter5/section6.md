 ## Exactly Once in Sinks

At some point, every pipeline needs to output data to the outside world, and a
sink is simply a transform that does exactly that. Keep in mind that delivering
data externally is a side effect, and we have already mentioned that Dataflow
does not guarantee exactly-once application of side effects. So, how can a
sink guarantee that outputs are delivered exactly once?

The simplest answer is that a number of built-in sinks are provided as part of
the Beam SDK. These sinks are carefully designed to ensure that they do not
produce duplicates, even if executed multiple times. Whenever possible,
pipeline authors are encouraged to use one of these built-in sinks.

However, sometimes the built-ins are insufficient and you need to write your
own. The best approach is to ensure that your side-effect operation is
idempotent and therefore robust in the face of replay. However, often some
component of a side-effect DoFn is nondeterministic and thus might change
on replay. For example, in a windowed aggregation, the set of records in the
window can also be nondeterministic!

Specifically, the window might attempt to fire with elements e0, e1, e2, but
the worker crashes before committing the window processing (but not before
those elements are sent as a side effect). When the worker restarts, the
window will fire again, but now a late element e3 shows up. Because this
element shows up before the window is committed, it’s not counted as late
data, so the DoFn is called again with elements e0, e1, e2, e3. These are then
sent to the side-effect operation. Idempotency does not help here, because
different logical record sets were sent each time.

There are other ways nondeterminism can be introduced. The standard way to
address this risk is to rely on the fact that Dataflow currently guarantees that
only one version of a DoFn’s output can make it past a shuffle boundary.

A simple way of using this guarantee is via the built-in Reshuffle transform.
The pattern presented in Example 5-2 ensures that the side-effect operation
always receives a deterministic record to output.

_Example 5-2. Reshuffle example_

```
13
```

c.apply(Window.<..>into(FixedWindows.of(Duration.standardMinutes( 1 ))))
.apply(GroupByKey.<..>.create())
.apply(new PrepareOutputData())
.apply(Reshuffle.<..>of())
.apply(WriteToSideEffect());

The preceding pipeline splits the sink into two steps: PrepareOutputData
and WriteToSideEffect. PrepareOutputData outputs records
corresponding to idempotent writes. If we simply ran one after the other, the
entire process might be replayed on failure, PrepareOutputData might
produce a different result, and both would be written as side effects. When
we add the Reshuffle in between the two, Dataflow guarantees this can’t
happen.

Of course, Dataflow might still run the WriteToSideEffect operation
multiple times. The side effects themselves still need to be idempotent, or the
sink will receive duplicates. For example, an operation that sets or overwrites
a value in a data store is idempotent, and will generate correct output even if
it’s run several times. An operation that appends to a list is not idempotent; if
the operation is run multiple times, the same value will be appended each
time.

While Reshuffle provides a simple way of achieving stable input to a DoFn,
a GroupByKey works just as well. However, there is currently a proposal that
removes the need to add a GroupByKey to achieve stable input into a DoFn.
Instead, the user could annotate WriteToSideEffect with a special

annotation, @RequiresStableInput, and the system would then ensure
stable input to that transform.

