 ## Summary

In this chapter, we analyzed the world of joins (using the join vocabulary of
SQL) within the context of stream processing. We began with unwindowed
joins and saw how, conceptually, all joins are streaming joins as the core. We
saw how the foundation for essentially all of the other join variations is the
FULL OUTER join, and discussed the specific alterations that occur as part of
LEFT OUTER, RIGHT OUTER, INNER, ANTI, SEMI, and even CROSS joins. In
addition, we saw how all of those different join patterns interact in a world of
TVRs and streams.

We next moved on to windowed joins, and learned that windowing a join is
typically motivated by one or both of the following benefits:

```
The ability to partition the join within time for some business need
The ability to tie results from the join to the progress of a watermark
```
And, finally, we explored in depth one of the more interesting and useful
types of windows with respect to joining: temporal validity windows. We
saw how temporal validity windows very naturally carve time into regions of
validity for given values, based only on the specific points in time where

```
00:00 / 00:00
```

those values change. We learned that joins within validity windows require a
windowing framework that supports windows that can split over time, which
is something no existing streaming system today supports natively. And we
saw how concisely validity windows allowed us to solve the problem of
joining TVRs for currency conversion rates and orders together in a robust,
natural way.

Joins are often one of the more intimidating aspects of data processing,
streaming or otherwise. However, by understanding the theoretical
foundation of joins and how straightforwardly we can derive all the different
types of joins from that basic foundation, joins become a much less
frightening beast, even with the additional dimension of time that streaming
adds to the mix.

From a conceptual perspective, at least. There are many different ways to
implement each of these types of joins, some of which are likely much more
efficient than performing an actual FULL OUTER join and then filtering down
its results, especially when the rest of the query and the distribution of the
data are taken into consideration.

Again, ignoring what happens when there are duplicate join keys; more on
this when we get to SEMI joins.

From a conceptual perspective, at least. There are, of course, many different
ways to implement each of these types of joins, some of which might be

much more efficient than performing an actual FULL OUTER join and then
filtering down its results, depending on the rest of the query and the
distribution of the data.

Note that the example data and the temporal join use case motivating it are
lifted almost wholesale from Julian Hyde’s excellent “Streams, joins, and
temporal tables” document.

It’s a partial implementation because it only works if the windows exist in
isolation, as in Figure 9-1. As soon as you mix the windows with other data,
such as the joining examples below, you would need some mechanism for

1

2

3

4

5


splitting the data from the shrunken window into two separate windows,
which Beam does not currently provide.



