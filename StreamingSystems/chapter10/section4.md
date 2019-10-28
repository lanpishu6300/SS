 ## Storm

Next up is Apache Storm (Figure 10-15), the first real streaming system we
cover. Storm most certainly wasn’t the first streaming system in existence,
but I would argue it was the first streaming system to see truly broad
adoption across the industry, and for that reason we give it a closer look here.

```
Figure 10-15. Timeline: Storm
```

```
Figure 10-16. “History of Apache Storm and lessons learned”
```
Storm was the brainchild of Nathan Marz, who later chronicled the history of
its creation in a blog post titled “History of Apache Storm and lessons
learned” (Figure 10-16). The TL;DR version of it is that Nathan’s team at the
startup employing him then, BackType, had been attempting to process the
Twitter firehose using a custom system of queues and workers. He came to
essentially the same realization that the MapReduce folks had nearly a decade
earlier: the actual data processing portion of their code was only a tiny
amount of the system, and building those real-time data processing pipelines
would be a lot easier if there were a framework doing all the distributed
system’s dirty work under the covers. Out of that was born Storm.

The interesting thing about Storm, in comparison to the rest of the systems
we’ve talked about so far, is that the team chose to loosen the strong
consistency guarantees found in all of the other systems we’ve talked about
so far as a way of providing lower latency. By combining at-most once or at-
least once semantics with per-record processing and no integrated (i.e., no
consistent) notion of persistent state, Storm was able provide much lower


latency in providing results than systems that executed over batches of data
and guaranteed exactly-once correctness. And for a certain type of use cases,
this was a very reasonable trade-off to make.

Unfortunately, it quickly became clear that people really wanted to have their
cake and eat it, too. They didn’t just want to get their answers quickly, they
wanted to have both low-latency results _and_ eventual correctness. But such a
thing was impossible with Storm alone. Enter the Lambda Architecture.

Given the limitations of Storm, shrewd engineers began running a weakly
consistent Storm streaming pipeline alongside a strongly consistent Hadoop
batch pipeline. The former produced low-latency, inexact results, whereas the
latter produced high-latency, exact results, both of which would then be
somehow merged together in the end to provide a single low-latency,
eventually consistent view of the outputs. We learned back in Chapter 1 that
the Lambda Architecture was Marz’s other brainchild, as detailed in his post
titled “How to beat the CAP theorem” (Figure 10-17).^7


```
Figure 10-17. “How to beat the CAP theorem”
```
I’ve already spent a fair amount of time harping on the shortcomings of the
Lambda Architecture, so I won’t belabor those points here. But I will reiterate
this: the Lambda Architecture became quite popular, despite the costs and
headaches associated with it, simply because it met a critical need that a great
many businesses were otherwise having a difficult time fulfilling: that of
getting low-latency, but eventually correct results out of their data processing
pipelines.

From the perspective of the evolution of streaming systems, I argue that
Storm was responsible for first bringing low-latency data processing to the
masses. However, it did so at the cost of weak consistency, which in turn
brought about the rise of the Lambda Architecture, and the years of dual-
pipeline darkness that followed.


```
Figure 10-18. Heron paper
```
But hyperbolic dramaticism aside, Storm was the system that gave the
industry its first taste of low-latency data processing, and the impact of that is
reflected in the broad interest in and adoption of streaming systems today.

Before moving on, it’s also worth giving a shout out to Heron. In 2015,
Twitter (the largest known user of Storm in the world, and the company that
originally fostered the Storm project) surprised the industry by announcing it
was abandoning the Storm execution engine in favor of a new system it had
developed in house, called Heron. Heron aimed to address a number of
performance and maintainability issues that had plagued Storm, while
remaining API compatible, as detailed in the company’s paper titled “Twitter
Heron: Stream Processing at Scale” (Figure 10-18). Heron itself was
subsequently open sourced (with governance moved to its own independent
foundation, not an existing one like Apache). Given the continued
development on Storm, there are now two competing variants of the Storm


lineage. Where things will end up is anyone’s guess, but it will be exciting to
watch.

