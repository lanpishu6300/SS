 ## Summary

At this point, we have explored how we can use the event times of messages
to give a robust definition of progress in a stream processing system. We saw
how this notion of progress can subsequently help us answer the question of
_where_ in event time processing is taking place and _when_ in processing time
results are materialized. Specifically, we looked at how watermarks are
created at the sources, the points of data ingestion into a pipeline, and then
propagated throughout the pipeline to preserve the essential guarantees that
allow the questions of _where_ and _when_ to be answered. We also looked at the
implications of changing the output window timestamps on watermarks.
Finally, we explored some real-world system considerations when building
watermarks at scale.

Now that we have a firm footing in how watermarks work under the covers,
we can take a dive into what they can do for us as we use windowing and
triggering to answer more complex queries in Chapter 4.

Note the additional mention of monotonicity; we have not yet discussed
how to achieve this. Indeed the discussion thus far makes no mention of
monotonicity. If we considered exclusively the oldest in-flight event time, the
watermark would not always be monotonic, as we have made no assumptions
about our input. We return to this discussion later on.

To be precise, it’s not so much that the number of logs need be static as it is
that the number of logs at any given time be known a priori by the system. A
more sophisticated input source composed of a dynamically chosen number
of inputs logs, such as Pravega, could just as well be used for constructing a
perfect watermark. It’s only when the number of logs that exist in the
dynamic set at any given time is unknown (as in the example in the next
section) that one must fall back on a heuristic watermark.

1

2

3


Note that by saying “flow through the system,” I don’t necessarily imply
they flow along the same path as normal data. They might (as in Apache
Flink), but they might also be transmitted out-of-band (as in
MillWheel/Cloud Dataflow).

The _start_ of the window is not a safe choice from a watermark correctness
perspective because the first element in the window often comes _after_ the
beginning of the window itself, which means that the watermark is not
guaranteed to have been held back as far as the start of the window.

The percentile watermark triggering scheme described here is not currently
implemented by Beam; however, other systems such as MillWheel
implement this.

For more information on Flink watermarks, see the Flink documentation on
the subject.

3

4

5

6


