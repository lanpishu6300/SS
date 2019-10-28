## Unwindowed Joins

It’s a popular myth that streaming joins over unbounded data always require
windowing. But by applying the concepts we learned in Chapter 6, we can
see that’s simply not true. Joins (both windowed and unwindowed) are
simply another type of grouping operation, and grouping operations yield
tables. Thus, if we want to consume the table created by an unwindowed join
(or, equivalently, joins within a single global window covering all of time) as
a stream, we need only apply an ungrouping (or trigger) operation that isn’t
of the “wait until we’ve seen all the input” variety. Windowing the join into a
nonglobal window and using a watermark trigger (i.e., a “wait until we’ve
seen all the input in a finite temporal chunk of the stream” trigger) is indeed
one option, but so is triggering on every record (i.e., materialized view
semantics) or periodically as processing time advances, regardless of whether
the join is windowed or not. Because it makes the examples easy to follow,
we assume the use of an implicit default per-record trigger in all of the
following unwindowed join examples that observe the join results as a
stream.

Now, onto joins themselves. ANSI SQL defines five types of joins: FULL
OUTER, LEFT OUTER, RIGHT OUTER, INNER, and CROSS. We look at the first
four in depth, and discuss the last only briefly in the next paragraph. We also
touch on two other interesting, but less-often encountered (and less well
supported, at least using standard syntax) variations: ANTI and SEMI joins.

On the surface, it sounds like a lot of variations. But as you’ll see, there’s
really only one type of join at the core: the FULL OUTER join. A CROSS join is

just a FULL OUTER join with a vacuously true join predicate; that is, it returns
every possible pairing of a row from the left table with a row from the right
table. All of the other join variations simply reduce down to some logical
1


subset of the FULL OUTER join. As a result, after you understand the
commonality between all the different join types, it becomes a lot easier to
keep them all in your head. It also makes reasoning about them in the context
of streaming all that much simpler.

One last note here before we get started: we’ll be primarily considering equi
joins with at most 1:1 cardinality, by which I mean joins in which the join
predicate is an equality statement and there is at most one matching row on
each side of the join. This keeps the examples simple and concise. When we

get to SEMI joins, we’ll expand our example to consider joins with arbitrary
N:M cardinality, which will let us observe the behavior of more arbitrary
predicate joins.

## FULL OUTER

Because they form the conceptual foundation for each of the other variations,
we first look at FULL OUTER joins. Outer joins embody a rather liberal and

optimistic interpretation of the word “join”: the result of FULL OUTER–joining
two datasets is essentially the full list of rows in both datasets, with rows in
the two datasets that share the same join key combined together, but
unmatched rows for either side included unjoined.

For example, if we FULL OUTER–join our two example datasets into a new
relation containing only the joined IDs, the result would look something like
this:

```
12:10> SELECT TABLE
Left.Id as L,
Right.Id as R,
FROM Left FULL OUTER JOIN Right
ON L.Num = R.Num;
---------------
| L | R |
---------------
| L1 | null |
| L2 | R2 |
| L3 | R3 |
```
```
1
```
```
2
```

```
| null | R4 |
---------------
```
We can see that the FULL OUTER join includes both rows that satisfied the
join predicate (e.g., “L2, R2” and “L3, R3”), but it also includes partial rows

that failed the predicate (e.g., “L1, null” and “null, R4”, where the null is
signaling the unjoined portion of the data).

Of course, that’s just a point-in-time snapshot of this FULL OUTER–join
relation, taken after all of the data have arrived in the system. We’re here to
learn about streaming joins, and streaming joins by definition involve the
added dimension of time. As we know from Chapter 8, if we want to
understand how a given dataset/relation changes over time, we want to speak
in terms of time-varying relations (TVRs). So to best understand how the join
evolves over time, let’s look now at the full TVR for this join (with changes
between each snapshot relation highlighted in yellow):

```
12:10> SELECT TVR
Left.Id as L,
Right.Id as R,
FROM Left FULL OUTER JOIN Right
ON L.Num = R.Num;
-----------------------------------------------------------------------
--
| [-inf, 12:01) | [12:01, 12:02) | [12:02, 12:03) | [12:03, 12:04)
|
| --------------- | --------------- | --------------- | ---------------
|
| | L | R | | | L | R | | | L | R | | | L | R |
|
| --------------- | --------------- | --------------- | ---------------
|
| --------------- | | null | R2 | | | L1 | null | | | L1 | null |
|
| | --------------- | | null | R2 | | | null | R2 |
|
| | | --------------- | | L3 | null |
|
| | | | ---------------
```

```
|
-----------------------------------------------------------------------
--
| [12:04, 12:05) | [12:05, 12:06) | [12:06, 12:07) |
| --------------- | --------------- | --------------- |
| | L | R | | | L | R | | | L | R | |
| --------------- | --------------- | --------------- |
| | L1 | null | | | L1 | null | | | L1 | null | |
| | null | L2 | | | null | L2 | | | L2 | L2 | |
| | L3 | L3 | | | L3 | L3 | | | L3 | L3 | |
| --------------- | | null | L4 | | | null | L4 | |
| | --------------- | --------------- |
-------------------------------------------------------
```
And, as you might then expect, the stream rendering of this TVR would
capture the specific deltas between each of those snapshots:

```
12:00> SELECT STREAM
Left.Id as L,
Right.Id as R,
CURRENT_TIMESTAMP as Time,
Sys.Undo as Undo
FROM Left FULL OUTER JOIN Right
ON L.Num = R.Num;
------------------------------
| L | R | Time | Undo |
------------------------------
| null | R2 | 12:01 | |
| L1 | null | 12:02 | |
| L3 | null | 12:03 | |
| L3 | null | 12:04 | undo |
| L3 | R3 | 12:04 | |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......
```
Note the inclusion of the Time and Undo columns, to highlight the times when
given rows materialize in the stream, and also call out instances when an
update to a given row first results in a retraction of the previous version of


that row. The undo/retraction rows are critical if this stream is to capture a
full-fidelity view of the TVR over time.

So, although each of these three renderings of the join (table, TVR, stream)
are distinct from one another, it’s also pretty clear how they’re all just
different views on the same data: the table snapshot shows us the overall
dataset as it exists after all the data have arrived, and the TVR and stream
versions capture (in their own ways) the evolution of the entire relation over
the course of its existence.

With that basic familiarity of FULL OUTER joins in place, we now understand
all of the core concepts of joins in a streaming context. No windowing
needed, no custom triggers, nothing particularly painful or unintuitive. Just a
per-record evolution of the join over time, as you would expect. Even better,
all of the other types of joins are just variations on this theme (conceptually,
at least), essentially just an additional filtering operation performed on the

per-record stream of the FULL OUTER join. Let’s now look at each of them in
more detail.

## LEFT OUTER

LEFT OUTER joins are a just a FULL OUTER join with any unjoined rows from
the right dataset removed. This is most clearly seen by taking the original
FULL OUTER join and graying out the rows that would be filtered. For a LEFT
OUTER join, that would look like the following, where every row with an
unjoined left side is filtered out of the original FULL OUTER join:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left LEFT OUTER JOIN Right FROM Left
```

```
LEFT OUTER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | null | R2 | 12:01 | |
| L2 | R2 | | L1 | null | 12:02 | |
| L3 | R3 | | L3 | null | 12:03 | |
| null | R4 | | L3 | null | 12:04 | undo |
--------------- | L3 | R3 | 12:04 | |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......
```
To see what the table and stream would actually look like in practice, let’s
look at the same queries again, but this time with the grayed-out rows omitted
entirely:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left LEFT OUTER JOIN Right FROM Left
LEFT OUTER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | L1 | null | 12:02 | |
| L2 | R2 | | L3 | null | 12:03 | |
| L3 | R3 | | L3 | null | 12:04 | undo |
--------------- | L3 | R3 | 12:04 | |
| L2 | R2 | 12:06 | |
```

```
....... [12:00, 12:10] .......
```
## RIGHT OUTER

RIGHT OUTER joins are the converse of a left join: all unjoined rows from the
left dataset in the full outer join are right out, *cough*, removed:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left RIGHT OUTER JOIN Right FROM Left
RIGHT OUTER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | null | R2 | 12:01 | |
| L2 | R2 | | L1 | null | 12:02 | |
| L3 | R3 | | L3 | null | 12:03 | |
| null | R4 | | L3 | null | 12:04 | undo |
--------------- | L3 | R3 | 12:04 | |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......
```
And here we see how the queries rendered as the actual RIGHT OUTER join
would appear:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
```

```
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left RIGHT OUTER JOIN Right FROM Left
RIGHT OUTER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L2 | R2 | | null | R2 | 12:01 | |
| L3 | R3 | | L3 | R3 | 12:04 | |
| null | R4 | | null | R4 | 12:05 | |
--------------- | null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......
```
## INNER

INNER joins are essentially the intersection of the LEFT OUTER and RIGHT
OUTER joins. Or, to think of it subtractively, the rows removed from the
original FULL OUTER join to create an INNER join are the union of the rows

removed from the LEFT OUTER and RIGHT OUTER joins. As a result, all rows
that remain unjoined on either side are absent from the INNER join:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left INNER JOIN Right FROM Left
INNER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
```

```
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | null | R2 | 12:01 | |
| L2 | R2 | | L1 | null | 12:02 | |
| L3 | R3 | | L3 | null | 12:03 | |
| null | R4 | | L3 | null | 12:04 | undo |
--------------- | L3 | R3 | 12:04 | |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......
```
And again, more succinctly rendered as the INNER join would look in reality:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left INNER JOIN Right FROM Left
INNER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L2 | R2 | | L3 | R3 | 12:04 | |
| L3 | R3 | | L2 | R2 | 12:06 | |
--------------- ....... [12:00, 12:10] .......
```
Given this example, you might be inclined to think retractions never play a
part in INNER join streams because they were all filtered out in this example.
But imagine if the value in the Left table for the row with a Num of 3 were

updated from “L3” to “L3v2” at 12:07. In addition to resulting in a different
value on the left side for our final TABLE query (again performed at 12:10,
which is after the update to row 3 on the Left arrived), it would also result in


a STREAM that captures both the removal of the old value via a retraction and
the addition of the new value:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM LeftV2 INNER JOIN Right FROM LeftV2
INNER JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- -----------------------------
```
-
| L | R | | L | R | Time | Undo
|
--------------- -----------------------------
-
| L2 | R2 | | L3 | R3 | 12:04 |
|
| L3v2 | R3 | | L2 | R2 | 12:06 |
|
--------------- | L3 | R3 | 12:07 | undo
|
| L3v2 | R3 | 12:07 |
|
....... [12:00, 12:10]
.......

## ANTI

ANTI joins are the obverse of the INNER join: they contain all of the _unjoined_
rows. Not all SQL systems support a clean ANTI join syntax, but I’ll use the
most straightforward one here for clarity:

```
12:00> SELECT
```

```
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left ANTI JOIN Right FROM Left
ANTI JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
```
-
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | null | R2 | 12:01 | |
| L2 | R2 | | L1 | null | 12:02 | |
| L3 | R3 | | L3 | null | 12:03 | |
| null | R4 | | L3 | null | 12:04 | undo |
--------------- | L3 | R3 | 12:04 | |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
| L2 | R2 | 12:06 | |
....... [12:00, 12:10] .......

What’s slightly interesting about the stream rendering of the ANTI join is that
it ends up containing a bunch of false-starts and retractions for rows which

eventually do end up joining; in fact, the ANTI join is as heavy on retractions
as the INNER join is light. The more concise versions would look like this:

```
12:00> SELECT
STREAM Left.Id as L,
12:10> SELECT TABLE Right.Id
as R,
Left.Id as L,
Sys.EmitTime as Time,
Right.Id as R Sys.Undo
as Undo
FROM Left ANTI JOIN Right FROM Left
```

```
ANTI JOIN Right
ON L.Num = R.Num; ON L.Num =
R.Num;
--------------- ------------------------------
| L | R | | L | R | Time | Undo |
--------------- ------------------------------
| L1 | null | | null | R2 | 12:01 | |
| null | R4 | | L1 | null | 12:02 | |
--------------- | L3 | null | 12:03 | |
| L3 | null | 12:04 | undo |
| null | R4 | 12:05 | |
| null | R2 | 12:06 | undo |
....... [12:00, 12:10] .......
```
## SEMI

We now come to SEMI joins, and SEMI joins are kind of weird. At first
glance, they basically look like inner joins with one side of the joined values
being dropped. And, indeed, in cases for which the cardinality relationship of
<side-being-kept>:<side-being-dropped> is N:M with M ≤ 1, this
works (note that we’ll be using kept=Left, dropped=Right for all the
examples that follow). For example, on the Left and Right datasets we’ve
used so far (which had cardinalities of 0:1, 1:0, and 1:1 for the joined data),
the INNER and SEMI join variations look identical:

```
12:10> SELECT TABLE 12:10> SELECT TABLE
Left.Id as L Left.Id as L
FROM Left INNER JOIN FROM Left SEMI JOIN
Right ON L.Num = R.Num; Right ON L.Num = R.Num;
--------------- ---------------
| L | R | | L | R |
--------------- ---------------
| L1 | null | | L1 | null |
| L2 | R2 | | L2 | R2 |
| L3 | R3 | | L3 | R3 |
| null | R4 | | null | R4 |
--------------- ---------------
```

However, there’s an additional subtlety to SEMI joins in the case of N:M
cardinality with M > 1: because the _values_ on the M side are not being

returned, the SEMI join simply predicates the join condition on there being
_any_ matching row on the right, rather than repeatedly yielding a new result
for _every_ matching row.

To see this clearly, let’s switch to a slightly more complicated pair of input
relations that highlight the N:M join cardinality of the rows contained therein.
In these relations, the N_M column states what the cardinality relationship of

rows is between the left and right sides, and the Id column (as before)
provides an identifier that is unique for each row in each of the input
relations:

```
12:15> SELECT TABLE * FROM LeftNM; 12:15> SELECT TABLE *
FROM RightNM;
--------------------- ---------------------
| N_M | Id | Time | | N_M | Id | Time |
--------------------- ---------------------
| 1:0 | L2 | 12:07 | | 0:1 | R1 | 12:02 |
| 1:1 | L3 | 12:01 | | 1:1 | R3 | 12:14 |
| 1:2 | L4 | 12:05 | | 1:2 | R4A | 12:03 |
| 2:1 | L5A | 12:09 | | 1:2 | R4B | 12:04 |
| 2:1 | L5B | 12:08 | | 2:1 | R5 | 12:06 |
| 2:2 | L6A | 12:12 | | 2:2 | R6A | 12:11 |
| 2:2 | L6B | 12:10 | | 2:2 | R6B | 12:13 |
--------------------- ---------------------
```
With these inputs, the FULL OUTER join expands to look like these:

```
12:00> SELECT STREAM
```
```
COALESCE(LeftNM.N_M,
12:15> SELECT TABLE
RightNM.N_M) as N_M,
COALESCE(LeftNM.N_M, LeftNM.Id
as L,
RightNM.N_M) as N_M,
RightNM.Id as R,
```

**_LeftNM.Id as L,
Sys.EmitTime as Time,
RightNM.Id as R, Sys.Undo
as Undo
FROM LeftNM FROM LeftNM
FULL OUTER JOIN RightNM FULL OUTER
JOIN RightNM
ON LeftNM.N_M = RightNM.N_M; ON
LeftNM.N_M = RightNM.N_M;_**
--------------------- --------------------------------
----
| N_M | L | R | | N_M | L | R | Time |
Undo |
--------------------- --------------------------------
----
| 0:1 | null | R1 | | 1:1 | L3 | null | 12:01 |
|
| 1:0 | L2 | null | | 0:1 | null | R1 | 12:02 |
|
| 1:1 | L3 | R3 | | 1:2 | null | R4A | 12:03 |
|
| 1:2 | L4 | R4A | | 1:2 | null | R4B | 12:04 |
|
| 1:2 | L4 | R4B | | 1:2 | null | R4A | 12:05 |
undo |
| 2:1 | L5A | R5 | | 1:2 | null | R4B | 12:05 |
undo |
| 2:1 | L5B | R5 | | 1:2 | L4 | R4A | 12:05 |
|
| 2:2 | L6A | R6A | | 1:2 | L4 | R4B | 12:05 |
|
| 2:2 | L6A | R6B | | 2:1 | null | R5 | 12:06 |
|
| 2:2 | L6B | R6A | | 1:0 | L2 | null | 12:07 |
|
| 2:2 | L6B | R6B | | 2:1 | null | R5 | 12:08 |
undo |
--------------------- | 2:1 | L5B | R5 | 12:08 |
|
| 2:1 | L5A | R5 | 12:09 |
|
| 2:2 | L6B | null | 12:10 |


```
|
| 2:2 | L6B | null | 12:11 |
undo |
| 2:2 | L6B | R6A | 12:11 |
|
| 2:2 | L6A | R6A | 12:12 |
|
| 2:2 | L6A | R6B | 12:13 |
|
| 2:2 | L6B | R6B | 12:13 |
|
| 1:1 | L3 | null | 12:14 |
undo |
| 1:1 | L3 | R3 | 12:14 |
|
.......... [12:00, 12:15]
..........
```
As a side note, one additional benefit of these more complicated datasets is
that the multiplicative nature of joins when there are multiple rows on each
side matching the same predicate begins to become more clear (e.g., the “2:2”
rows, which expand from two rows in each the inputs to four rows in the
output; if the dataset had a set of “3:3” rows, they’d expand from three rows
in each of the inputs to nine rows in the output, and so on).

But back to the subtleties of SEMI joins. With these datasets, it becomes much
clearer what the difference between the filtered INNER join and the SEMI join

is: the INNER join yields duplicate values for any of the rows where the N:M
cardinality has M > 1, whereas the SEMI join doesn’t (note that I’ve
highlighted the duplicate rows in the INNER join version in red, and included
in gray the portions of the full outer join that are omitted in the respective
INNER and SEMI versions):

```
12:15> SELECT TABLE 12:15> SELECT
TABLE
COALESCE(LeftNM.N_M,
COALESCE(LeftNM.N_M,
RightNM.N_M) as N_M,
```

```
RightNM.N_M) as N_M,
LeftNM.Id as L
LeftNM.Id as L
FROM LeftNM INNER JOIN RightNM FROM
LeftNM SEMI JOIN RightNM
ON LeftNM.N_M = RightNM.N_M; ON
LeftNM.N_M = RightNM.N_M;
--------------------- ---------------------
| N_M | L | R | | N_M | L | R |
--------------------- ---------------------
| 0:1 | null | R1 | | 0:1 | null | R1 |
| 1:0 | L2 | null | | 1:0 | L2 | null |
| 1:1 | L3 | R3 | | 1:1 | L3 | R3 |
| 1:2 | L4 | R5A | | 1:2 | L4 | R5A |
| 1:2 | L4 | R5B | | 1:2 | L4 | R5B |
| 2:1 | L5A | R5 | | 2:1 | L5A | R5 |
| 2:1 | L5B | R5 | | 2:1 | L5B | R5 |
| 2:2 | L6A | R6A | | 2:2 | L6A | R6A |
| 2:2 | L6A | R6B | | 2:2 | L6A | R6B |
| 2:2 | L6B | R6A | | 2:2 | L6B | R6A |
| 2:2 | L6B | R6B | | 2:2 | L6B | R6B |
--------------------- ---------------------
```
Or, rendered more succinctly:

```
12:15> SELECT TABLE 12:15> SELECT
TABLE
COALESCE(LeftNM.N_M,
COALESCE(LeftNM.N_M,
RightNM.N_M) as N_M,
RightNM.N_M) as N_M,
LeftNM.Id as L
LeftNM.Id as L
FROM LeftNM INNER JOIN RightNM FROM
LeftNM SEMI JOIN RightNM
ON LeftNM.N_M = RightNM.N_M; ON
LeftNM.N_M = RightNM.N_M;
------------- -------------
| N_M | L | | N_M | L |
------------- -------------
```

```
| 1:1 | L3 | | 1:1 | L3 |
| 1:2 | L4 | | 1:2 | L4 |
| 1:2 | L4 | | 2:1 | L5A |
| 2:1 | L5A | | 2:1 | L5B |
| 2:1 | L5B | | 2:2 | L6A |
| 2:2 | L6A | | 2:2 | L6B |
| 2:2 | L6A | -------------
| 2:2 | L6B |
| 2:2 | L6B |
-------------
```
The STREAM renderings then provide a bit of context as to which rows are
filtered out—they are simply the later-arriving duplicate rows (from the
perspective of the columns being projected):

```
12:00> SELECT STREAM 12:00> SELECT
STREAM
COALESCE(LeftNM.N_M,
COALESCE(LeftNM.N_M,
RightNM.N_M) as N_M,
RightNM.N_M) as N_M,
LeftNM.Id as L
LeftNM.Id as L
Sys.EmitTime as Time,
Sys.EmitTime as Time,
Sys.Undo as Undo,
Sys.Undo as Undo,
FROM LeftNM INNER JOIN RightNM FROM
LeftNM SEMI JOIN RightNM
ON LeftNM.N_M = RightNM.N_M; ON
LeftNM.N_M = RightNM.N_M;
------------------------------------ ---------------------------
---------
| N_M | L | R | Time | Undo | | N_M | L | R | Time
| Undo |
------------------------------------ ---------------------------
---------
| 1:1 | L3 | null | 12:01 | | | 1:1 | L3 | null | 12:01
| |
| 0:1 | null | R1 | 12:02 | | | 0:1 | null | R1 | 12:02
```

| |
| 1:2 | null | R4A | 12:03 | | | 1:2 | null | R4A | 12:03
| |
| 1:2 | null | R4B | 12:04 | | | 1:2 | null | R4B | 12:04
| |
| 1:2 | null | R4A | 12:05 | undo | | 1:2 | null | R4A | 12:05
| undo |
| 1:2 | null | R4B | 12:05 | undo | | 1:2 | null | R4B | 12:05
| undo |
| 1:2 | L4 | R4A | 12:05 | | | 1:2 | L4 | R4A | 12:05
| |
| 1:2 | L4 | R4B | 12:05 | | | 1:2 | L4 | R4B | 12:05
| |
| 2:1 | null | R5 | 12:06 | | | 2:1 | null | R5 | 12:06
| |
| 1:0 | L2 | null | 12:07 | | | 1:0 | L2 | null | 12:07
| |
| 2:1 | null | R5 | 12:08 | undo | | 2:1 | null | R5 | 12:08
| undo |
| 2:1 | L5B | R5 | 12:08 | | | 2:1 | L5B | R5 | 12:08
| |
| 2:1 | L5A | R5 | 12:09 | | | 2:1 | L5A | R5 | 12:09
| |
| 2:2 | L6B | null | 12:10 | | | 2:2 | L6B | null | 12:10
| |
| 2:2 | L6B | null | 12:10 | undo | | 2:2 | L6B | null | 12:10
| undo |
| 2:2 | L6B | R6A | 12:11 | | | 2:2 | L6B | R6A | 12:11
| |
| 2:2 | L6A | R6A | 12:12 | | | 2:2 | L6A | R6A | 12:12
| |
| 2:2 | L6A | R6B | 12:13 | | | 2:2 | L6A | R6B | 12:13
| |
| 2:2 | L6B | R6B | 12:13 | | | 2:2 | L6B | R6B | 12:13
| |
| 1:1 | L3 | null | 12:14 | undo | | 1:1 | L3 | null | 12:14
| undo |
| 1:1 | L3 | R3 | 12:14 | | | 1:1 | L3 | R3 | 12:14
| |
.......... [12:00, 12:15] .......... .......... [12:00, 12:15]
..........


And again, rendered succinctly:

```
12:00> SELECT STREAM 12:00> SELECT
STREAM
COALESCE(LeftNM.N_M,
COALESCE(LeftNM.N_M,
RightNM.N_M) as N_M,
RightNM.N_M) as N_M,
LeftNM.Id as L
LeftNM.Id as L
Sys.EmitTime as Time,
Sys.EmitTime as Time,
Sys.Undo as Undo,
Sys.Undo as Undo,
FROM LeftNM INNER JOIN RightNM FROM
LeftNM SEMI JOIN RightNM
ON LeftNM.N_M = RightNM.N_M; ON
LeftNM.N_M = RightNM.N_M;
---------------------------- ---------------------------
```
-
| N_M | L | Time | Undo | | N_M | L | Time | Undo
|
---------------------------- ---------------------------
-
| 1:2 | L4 | 12:05 | | | 1:2 | L4 | 12:05 |
|
| 1:2 | L4 | 12:05 | | | 2:1 | L5B | 12:08 |
|
| 2:1 | L5B | 12:08 | | | 2:1 | L5A | 12:09 |
|
| 2:1 | L5A | 12:09 | | | 2:2 | L6B | 12:11 |
|
| 2:2 | L6B | 12:11 | | | 2:2 | L6A | 12:12 |
|
| 2:2 | L6A | 12:12 | | | 1:1 | L3 | 12:14 |
|
| 2:2 | L6A | 12:13 | | ...... [12:00, 12:15]
......
| 2:2 | L6B | 12:13 | |
| 1:1 | L3 | 12:14 | |
...... [12:00, 12:15] ......


As we’ve seen over the course of a number of examples, there’s really
nothing special about streaming joins. They function exactly as we might
expect given our knowledge of streams and tables, with join streams
capturing the history of the join over time as it evolves. This is in contrast to
join tables, which simply capture a snapshot of the entire join as it exists at a
specific point in time, as we’re perhaps more accustomed.

But, even more important, viewing joins through the lens of stream-table
theory has lent some additional clarity. The core underlying join primitive is
the FULL OUTER join, which is a stream → table grouping operation that
collects together all the joined and unjoined rows in a relation. All of the
other variants we looked at in detail (LEFT OUTER, RIGHT OUTER, INNER,
ANTI, and SEMI) simply add an additional layer of filtering on the joined

stream following the FULL OUTER join.

