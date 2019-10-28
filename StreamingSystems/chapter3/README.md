  # Chapter 3. Watermarks

So far, we have been looking at stream processing from the perspective of the
pipeline author or data scientist. Chapter 2 introduced watermarks as part of
the answer to the fundamental questions of _where_ in event-time processing is
taking place and _when_ in processing time results are materialized. In this
chapter, we approach the same questions, but instead from the perspective of
the underlying mechanics of the stream processing system. Looking at these
mechanics will help us motivate, understand, and apply the concepts around
watermarks. We discuss how watermarks are created at the point of data
ingress, how they propagate through a data processing pipeline, and how they
affect output timestamps. We also demonstrate how watermarks preserve the
guarantees that are necessary for answering the questions of _where_ in event-
time data are processed and _when_ it is materialized, while dealing with
unbounded data.

