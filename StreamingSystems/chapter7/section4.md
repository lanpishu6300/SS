 ## Summary


In this chapter, we’ve looked closely at why persistent state is important,
coming to the conclusion that it provides a basis for correctness and
efficiency in long-lived pipelines. We then looked at the two most common
types of implicit state encountered in data processing systems: raw grouping
and incremental combination. We learned that raw grouping is
straightforward but potentially inefficient and that incremental combination
greatly improves efficiency for operations that are commutative and
associative. Finally, we looked a relatively complex, but very practical use
case (and implementation via Apache Beam Java) grounded in real-world
experience, and used that to highlight the important characteristics needed in
a general state abstraction:

```
Flexibility in data structures , allowing for the use of data types
tailored to specific use cases at hand.
```
```
Flexibility in write and read granularity , allowing the amount of
data written and read at any point to be tailored to the use case,
minimizing or maximizing I/O as appropriate.
```
```
Flexibility in scheduling of processing , allowing certain portions of
processing to be delayed until a more appropriate point in time, such
as when the input is believed to be complete up to a specific point in
event time.
```
For some definition of “forever,” typically at least “until we successfully
complete execution of our batch pipeline and no longer require the inputs.”

Recall that Beam doesn’t currently expose these state tables directly; you
must trigger them back into a stream to observe their contents as a new
PCollection.

Or, as my colleague Kenn Knowles points out, if you take the definition as
being commutativity across sets, the three-parameter version of
commutativity is actually sufficient to also imply associativity: COMBINE(a,

b, c) == COMBINE(a, c, b) == COMBINE(b, a, c) == COMBINE(b, c,
a) == COMBINE(c, a, b) == COMBINE(c, b, a). Math is fun.

1

2

3

4


And indeed, timers are the underlying feature used to implement most of the
completeness and repeated updated triggers we discussed in Chapter 2 as well
as garbage collection based on allowed lateness.

Thanks to the nature of web browsing, the visit trails we’ll be analyzing are
trees of URLs linked by HTTP referrer fields. In reality, they would end up
being directed graphs, but for the sake of simplicity, we’ll assume each page
on our website has incoming links from exactly one other referring page on
the site, thus yielding a simpler tree structure. Generalizing to graphs is a
natural extension of the tree-based implementation, and only further drives
home the points being made.

4

5



