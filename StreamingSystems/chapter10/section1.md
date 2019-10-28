 ## MapReduce

We begin the journey with MapReduce (Figure 10-2).


```
Figure 10-2. Timeline: MapReduce
```
I think it’s safe to say that large-scale data processing as we all know it today
got its start with MapReduce way back in 2003. At the time, engineers
within Google were building all sorts of bespoke systems to tackle data
processing challenges at the scale of the World Wide Web. As they did so,
they noticed three things:

Data processing is hard

```
As the data scientists and engineers among us well know, you can build a
career out of just focusing on the best ways to extract useful insights from
raw data.
```
Scalability is hard

```
Extracting useful insights over massive-scale data is even more difficult
yet.
```
Fault-tolerance is hard

```
Extracting useful insights from massive-scale data in a fault-tolerant,
correct way on commodity hardware is brutal.
```
After solving all three of these challenges in tandem across a number of use
cases, they began to notice some similarities between the custom systems
they’d built. And they came to the conclusion that if they could build a

```
2
```
```
3
```

framework that took care of the latter two issues (scalability and fault-
tolerance), it would make focusing on the first issue a heck of a lot simpler.
Thus was born MapReduce.

The basic idea with MapReduce was to provide a simple data processing API
centered around two well-understand operations from the functional
programming realm: map and reduce (Figure 10-3). Pipelines built with that
API would then be executed on a distributed systems framework that took
care of all the nasty scalability and fault-tolerance stuff that quickens the
hearts of hardcore distributed-systems engineers and crushes the souls of the
rest of us mere mortals.

```
Figure 10-3. Visualization of a MapReduce job
```
We already discussed the semantics of MapReduce in great detail back in
Chapter 6, so we won’t dwell on them here. Simply recall that we broke
things down into six discrete phases (MapRead, Map, MapWrite,
ReduceRead, Reduce, ReduceWrite) as part of our streams and tables
analysis, and we came to the conclusion in the end that there really wasn’t all
that much different between the overall Map and Reduce phases; at a high-
level, they both do the following:

```
Convert a table to a stream
```
```
Apply a user transformation to that stream to yield another stream
```
```
4
```

```
Group that stream into a table
```
After it was placed into service within Google, MapReduce found such broad
application across a variety of tasks that the team decided it was worth
sharing its ideas with the rest of the world. The result was the MapReduce
paper, published at OSDI 2004 (see Figure 10-4).

```
Figure 10-4. The MapReduce paper, published at OSDI 2004
```
In it, the team described in detail the history of the project, design of the API
and implementation, and details about a number of different use cases to
which MapReduce had been applied. Unfortunately, they provided no actual
source code, so the best that folks outside of Google at the time could do was
say, “Yes, that sounds very nice indeed,” and go back to building their
bespoke systems.

Over the course of the decade that followed, MapReduce continued to
undergo heavy development within Google, with large amounts of time
invested in making the system scale to unprecedented levels. For a more


detailed account of some of the highlights along that journey, I recommend
the post “History of massive-scale sorting experiments at Google”
(Figure 10-5) written by our official MapReduce historian/scalability and
performance wizard, Marián Dvorský.

```
Figure 10-5. Marián Dvorský’s “History of massive-scale sorting experiments” blog post
```
But for our purposes here, suffice it to say that nothing else yet has touched
the magnitude of scale achieved by MapReduce, not even within Google.
Considering how long MapReduce has been around, that’s saying something;
14 years is an eternity in our industry.


From a streaming systems perspective, the main takeaways I want to leave
you with for MapReduce are _simplicity_ and _scalability_. MapReduce took the
first brave steps toward taming the unruly beast that is massive-scale data
processing, exposing a simple and straightforward API for crafting powerful
data processing pipelines, its austerity belying the complex distributed
systems magic happening under the covers to allow those pipelines to run at
scale on large clusters of commodity hardware.


