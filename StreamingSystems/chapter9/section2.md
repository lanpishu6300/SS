  # All Your Joins Are Belong to Streaming

What does it mean to join two datasets? We understand intuitively that joins
are just a specific type of grouping operation: by joining together data that
share some property (i.e., key), we collect together some number of
previously unrelated individual data elements into a _group_ of related
elements. And as we learned in Chapter 6, grouping operations always
consume a stream and yield a table. Knowing these two things, it’s only a
small leap to then arrive at the conclusion that forms the basis for this entire
chapter: _at their hearts, all joins are streaming joins_.

What’s great about this fact is that it actually makes the topic of streaming
joins that much more tractable. All of the tools we’ve learned for reasoning
about time within the context of streaming grouping operations (windowing,
watermarks, triggers, etc.) continue to apply in the case of streaming joins.
What’s perhaps intimidating is that adding streaming to the mix seems like it
could only serve to complicate things. But as you’ll see in the examples that
follow, there’s a certain elegant simplicity and consistency to modeling all
joins as streaming joins. Instead of feeling like there are a confounding


multitude of different join approaches, it becomes clear that nearly all types
of joins really boil down to minor variations on the same pattern. In the end,
that clarity of insight helps makes joins (streaming or otherwise) much less
intimidating.

To give us something concrete to reason about, let’s consider a number of
different types of joins as they’re applied the following datasets, conveniently

named Left and Right to match the common nomenclature:

```
12:10> SELECT TABLE * FROM Left; 12:10> SELECT TABLE
* FROM Right;
-------------------- --------------------
| Num | Id | Time | | Num | Id | Time |
-------------------- --------------------
| 1 | L1 | 12:02 | | 2 | R2 | 12:01 |
| 2 | L2 | 12:06 | | 3 | R3 | 12:04 |
| 3 | L3 | 12:03 | | 4 | R4 | 12:05 |
-------------------- --------------------
```
Each contains three columns:

Num

```
A single number.
```
Id

```
A portmanteau of the first letter in the name of the corresponding table
(“L” or “R”) and the Num, thus providing a way to uniquely identify the
source of a given cell in join results.
```
Time

```
The arrival time of the given record in the system, which becomes
important when considering streaming joins.
```
To keep things simple, note that our initial datasets will have strictly unique
join keys. When we get to SEMI joins, we’ll introduce some more
complicated datasets to highlight join behavior in the presence of duplicate
keys.


We first look at _unwindowed joins_ in a great deal of depth because
windowing often affects join semantics in only a minor way. After we
exhaust our appetite for unwindowed joins, we then touch upon some of the
more interesting points of joins in a windowed context.

