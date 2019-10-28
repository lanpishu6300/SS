 ## Hadoop

Next in our list is Hadoop (Figure 10-6). Fair warning: this is one of those
times where I will grossly oversimplify the impact of a system for the sake of
a focused narrative. The impact Hadoop has had on our industry and the
world at large cannot be overstated, and it extends well beyond the relatively
specific scope I discuss here.

```
Figure 10-6. Timeline: Hadoop
```
Hadoop came about in 2005, when Doug Cutting and Mike Cafarella decided
that the ideas from the MapReduce paper were just the thing they needed as
they built a distributed version of their Nutch webcrawler. They had already
built their own version of Google’s distributed filesystem (originally called
NDFS for Nutch Distributed File System, later renamed to HDFS, or Hadoop
Distributed File System), so it was a natural next step to add a MapReduce


layer on top after that paper was published. They called this layer Hadoop.

The key difference between Hadoop and MapReduce was that Cutting and
Cafarella made sure the source code for Hadoop was shared with the rest of
the world by open sourcing it (along with the source for HDFS) as part of
what would eventually become the Apache Hadoop project. Yahoo’s hiring
of Cutting to help transition the Yahoo webcrawler architecture onto Hadoop
gave the project an additional boost of validity and engineering oomph, and
from there, an entire ecosystem of open source data processing tools grew.
As with MapReduce, others have told the history of Hadoop in other fora far
better than I can; one particularly good reference is Marko Bonaci’s “The
history of Hadoop,” itself originally slated for inclusion in a print book
(Figure 10-7).

```
Figure 10-7. Marko Bonaci’s “The history of Hadoop”
```
The main point I want you to take away from this section is the massive


impact the _open source ecosystem_ that flowered around Hadoop had upon the
industry as a whole. By creating an open community in which engineers
could improve and extend the ideas from those early GFS and MapReduce
papers, a thriving ecosystem was born, yielding dozens of useful tools like
Pig, Hive, HBase, Crunch, and on and on. That openness was key to
incubating the diversity of ideas that exist now across our industry, and it’s
why I’m pigeonholing Hadoop’s open source ecosystem as its single most
important contribution to the world of streaming systems as we know them
today.

