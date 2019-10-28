 # Chapter 5. Exactly-Once and

# Side Effects

We now shift from discussing programming models and APIs to the systems
that implement them. A model and API allows users to describe what they
want to compute. Actually running the computation accurately at scale
requires a system—usually a distributed system.

In this chapter, we focus on how an implementing system can correctly
implement the Beam Model to produce accurate results. Streaming systems
often talk about _exactly-once processing_ ; that is, ensuring that every record is
processed exactly one time. We will explain what we mean by this, and how
it might be implemented.

As a motivating example, this chapter focuses on techniques used by Google
Cloud Dataflow to efficiently guarantee exactly-once processing of records.
Toward the end of the chapter, we also look at techniques used by some other
popular streaming systems to guarantee exactly once.

