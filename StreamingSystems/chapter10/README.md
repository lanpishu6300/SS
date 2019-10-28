 # Chapter 10. The Evolution of

# Large-Scale Data Processing

You have now arrived at the final chapter in the book, you stoic literate, you.
Your journey will soon be complete!

To wrap things up, I’d like you to join me on a brief stroll through history,
starting back in the ancient days of large-scale data processing with
MapReduce and touching upon some of the highlights over the ensuing
decade and a half that have brought streaming systems to the point they’re at
today. It’s a relatively lightweight chapter in which I make a few
observations about important contributions from a number of well-known
systems (and a couple maybe not-so-well known), refer you to a bunch of
source material you can go read on your own should you want to learn more,
all while attempting not to offend or inflame the folks responsible for systems
whose truly impactful contributions I’m going to either oversimplify or
ignore completely for the sake of space, focus, and a cohesive narrative.
Should be a good time.

On that note, keep in mind as you read this chapter that we’re really just
talking about specific pieces of the MapReduce/Hadoop family tree of large-
scale data processing here. I’m not covering the SQL arena in any way shape
or form ; we’re not talking HPC/supercomputers, and so on. So as broad and
expansive as the title of this chapter might sound, I’m really focusing on a
specific vertical swath of the grand universe of large-scale data processing.
Caveat literatus, and all that.

Also note that I’m covering a disproportionate amount of Google
technologies here. You would be right in thinking that this might have
something to do with the fact that I’ve worked at Google for more than a
decade. But there are two other reasons for it: 1) big data has always been
important for Google, so there have been a number of worthwhile
contributions created there that merit discussing in detail, and 2) my

```
1
```

experience has been that folks outside of Google generally seem to enjoy
learning more about the things we’ve done, because we as a company have
historically been somewhat tight-lipped in that regard. So indulge me a bit
while I prattle on excessively about the stuff we’ve been working on behind
closed doors.

To ground our travels in concrete chronology, we’ll be following the timeline
in Figure 10-1, which shows rough dates of existence for the various systems
I discuss.

```
Figure 10-1. Approximate timeline of systems discussed in this chapter
```
At each stop, I give a brief history of the system as best I understand it and
frame its contributions from the perspective of shaping streaming systems as
we know them today. At the end, we recap all of the contributions to see how
they’ve summed up to create the modern stream processing ecosystem of
today.
