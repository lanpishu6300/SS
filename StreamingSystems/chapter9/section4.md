 ## Windowed Joins

Having looked at a variety of unwindowed joins, let’s next explore what
windowing adds to the mix. I would argue that there are two motivations for
windowing your joins:

To partition time in some meaningful way

```
An obvious case is fixed windows; for example, daily windows, for
which events that occurred in the same day should be joined together for
some business reason (e.g., daily billing tallies). Another might be
limiting the range of time within a join for performance reasons.
However, it turns out there are even more sophisticated (and useful) ways
of partitioning time in joins, including one particularly interesting use
case that no streaming system I’m aware of today supports natively:
temporal validity joins. More on this in just a bit.
```
To provide a meaningful reference point for timing out a join

```
This is useful for a number of unbounded join situations, but it is perhaps
most obviously beneficial for use cases like outer joins, for which it is
```
```
3
```

```
unknown a priori if one side of the join will ever show up. For classic
batch processing (including standard interactive SQL queries), outer joins
are timed out only when the bounded input dataset has been fully
processed. But when processing unbounded data, we can’t wait for all
data to be processed. As we discussed in Chapters 2 and 3 , watermarks
provide a progress metric for gauging the completeness of an input source
in event time. But to make use of that metric for timing out a join, we
need some reference point to compare against. Windowing a join
provides that reference by bounding the extent of the join to the end of
the window. After the watermark passes the end of the window, the
system may consider the input for the window complete. At that point,
just as in the bounded join case, it’s safe to time out any unjoined rows
and materialize their partial results.
```
That said, as we saw earlier, windowing is absolutely not a requirement for
streaming joins. It makes a lot of sense in a many cases, but by no means is it
a necessity.

In practice, most of the use cases for windowed joins (e.g., daily windows)
are relatively straightforward and easy to extrapolate from the concepts we’ve
learned up until now. To see why, we look briefly at what it means to apply
fixed windows to some of the join examples we already encountered. After
that, we spend the rest of this chapter investigating the much more interesting
(and mind-bending) topic of _temporal validity joins_ , looking first in detail at
what I mean by temporal validity windows, and then moving on to looking at
what joins mean within the context of such windows.

## Fixed Windows

Windowing a join adds the dimension of time into the join criteria
themselves. In doing so, the window serves to scope the set of rows being
joined to only those contained within the window’s time interval. This is
perhaps more clearly seen with an example, so let’s take our original Left
and Right tables and window them into five-minute fixed windows:


```
12:10> SELECT TABLE *, 12:10> SELECT
TABLE *,
TUMBLE(Time, INTERVAL '5' MINUTE)
TUMBLE(Time, INTERVAL '5' MINUTE)
as Window FROM Left; as Window
FROM Right
------------------------------------- ----------------------------
---------
| Num | Id | Time | Window | | Num | Id | Time | Window
|
------------------------------------- ----------------------------
---------
| 1 | L1 | 12:02 | [12:00, 12:05) | | 2 | R2 | 12:01 | [12:00,
12:05) |
| 2 | L2 | 12:06 | [12:05, 12:10) | | 3 | R3 | 12:04 | [12:00,
12:05) |
| 3 | L3 | 12:03 | [12:00, 12:05) | | 4 | R4 | 12:05 | [12:05,
12:06) |
------------------------------------- ----------------------------
---------
```
In our previous Left and Right examples, the join criterion was simply
Left.Num = Right.Num. To turn this into a windowed join, we would

expand the join criteria to include window equality, as well: Left.Num =
Right.Num AND Left.Window = Right.Window. Knowing that, we can
already infer from the preceding windowed tables how our join is going to
change (highlighted for clarity): because the L2 and R2 rows do not fall
within the same five-minute fixed window, they will not be joined together in
the windowed variant of our join.

And indeed, if we compare the unwindowed and windowed variants side-by-
side as tables, we can see this clearly (with the corresponding L2 and R2 rows
highlighted on each side of the join):

```
12:10> SELECT TABLE
Left.Id as L,
Right.Id as R,
COALESCE(
```

```
TUMBLE(Left.Time, INTERVAL '5' MINUTE),
```
```
TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE ) AS Window
Left.Id as L, FROM Left
Right.Id as R, FULL OUTER JOIN
Right
FROM Left ON L.Num = R.Num
AND
FULL OUTER JOIN Right
TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
ON L.Num = R.Num;
TUMBLE(Right.Time, INTERVAL '5' MINUTE);
--------------- --------------------------------
| L | R | | L | R | Window |
--------------- --------------------------------
| L1 | null | | L1 | null | [12:00, 12:05) |
| L2 | R2 | | null | R2 | [12:00, 12:05) |
| L3 | R3 | | L3 | R3 | [12:00, 12:05) |
| null | R4 | | L2 | null | [12:05, 12:10) |
--------------- | null | R4 | [12:05, 12:10) |
--------------------------------
```
The difference is also readily apparent when comparing the unwindowed and
windowed joins as streams. As I’ve highlighted in the example that follows,
they differ primarily in their final rows. The unwindowed side completes the

join for Num = 2, yielding a retraction for the unjoined R2 row in addition to
a new row for the completed L2, R2 join. The windowed side, on the other
hand, simply yields an unjoined L2 row because L2 and R2 fall within
different five-minute windows:

```
12:10> SELECT STREAM
Left.Id as L,
Right.Id as R,
Sys.EmitTime as
Time,
COALESCE(
```

```
TUMBLE(Left.Time, INTERVAL '5' MINUTE),
12:10> SELECT STREAM
TUMBLE(Right.Time, INTERVAL '5' MINUTE)
Left.Id as L, ) AS Window,
Right.Id as R, Sys.Undo as Undo
Sys.EmitTime as Time, FROM Left
Sys.Undo as Undo FULL OUTER JOIN
Right
FROM Left ON L.Num = R.Num
AND
FULL OUTER JOIN Right
TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
ON L.Num = R.Num;
TUMBLE(Right.Time, INTERVAL '5' MINUTE);
------------------------------ --------------------------------------
---------
| L | R | Time | Undo | | L | R | Time | Window
| Undo |
------------------------------ --------------------------------------
---------
| null | R2 | 12:01 | | | null | R2 | 12:01 | [12:00, 12:05)
| |
| L1 | null | 12:02 | | | L1 | null | 12:02 | [12:00, 12:05)
| |
| L3 | null | 12:03 | | | L3 | null | 12:03 | [12:00, 12:05)
| |
| L3 | null | 12:04 | undo | | L3 | null | 12:04 | [12:00, 12:05)
| undo |
| L3 | R3 | 12:04 | | | L3 | R3 | 12:04 | [12:00, 12:05)
| |
| null | R4 | 12:05 | | | null | R4 | 12:05 | [12:05, 12:10)
| |
| null | R2 | 12:06 | undo | | L2 | null | 12:06 | [12:05, 12:10)
| |
| L2 | R2 | 12:06 | | ............... [12:00, 12:10]
................
....... [12:00, 12:10] .......
```
And with that, we now understand the effects of windowing on a FULL OUTER
join. By applying the rules we learned in the first half of the chapter, it’s then


easy to derive the windowed variants of LEFT OUTER, RIGHT OUTER, INNER,
ANTI, and SEMI joins, as well. I will leave most of these derivations as an
exercise for you to complete, but to give a single example, LEFT OUTER join,
as we learned, is just the FULL OUTER join with null columns on the left side

of the join removed (again, with L2 and R2 rows highlighted to compare the
differences):

```
12:10> SELECT TABLE
Left.Id as L,
Right.Id as R,
COALESCE(
```
```
TUMBLE(Left.Time, INTERVAL '5' MINUTE),
```
```
TUMBLE(Right.Time, INTERVAL '5' MINUTE)
12:10> SELECT TABLE ) AS Window
Left.Id as L, FROM Left
Right.Id as R, LEFT OUTER JOIN
Right
FROM Left ON L.Num = R.Num
AND
LEFT OUTER JOIN Right
TUMBLE(Left.Time, INTERVAL '5' MINUTE) =
ON L.Num = R.Num;
TUMBLE(Right.Time, INTERVAL '5' MINUTE);
--------------- --------------------------------
| L | R | | L | R | Window |
--------------- --------------------------------
| L1 | null | | L1 | null | [12:00, 12:05) |
| L2 | R2 | | L2 | null | [12:05, 12:10) |
| L3 | R3 | | L3 | R3 | [12:00, 12:05) |
--------------- --------------------------------
```
By scoping the region of time for the join into fixed five-minute intervals, we
chopped our datasets into two distinct windows of time: [12:00, 12:05)
and [12:05, 12:10). The exact same join logic we observed earlier was
then applied within those regions, yielding a slightly different outcome for


the case in which the L2 and R2 rows fell into separate regions. And at a basic
level, that’s really all there is to windowed joins.

## Temporal Validity

Having looked at the basics of windowed joins, we now spend the rest of the
chapter looking at a somewhat more advanced approach: temporal validity
windowing.

**Temporal validity windows**

Temporal validity windows apply in situations in which the rows in a relation
effectively slice time into regions wherein a given value is valid. More
concretely, imagine a financial system for performing currency conversions.
Such a system might contain a time-varying relation that captured the current
conversion rates for various types of currency. For example, there might be a
relation for converting from different currencies to Yen, like this:

```
12:10> SELECT TABLE * FROM YenRates;
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| USD | 102 | 12:00:00 | 12:04:13 |
| Euro | 114 | 12:00:30 | 12:06:23 |
| Yen | 1 | 12:01:00 | 12:05:18 |
| Euro | 116 | 12:03:00 | 12:09:07 |
| Euro | 119 | 12:06:00 | 12:07:33 |
--------------------------------------
```
To highlight what I mean by saying that temporal validity windows
“effectively slice time into regions wherein a given value is valid,” consider
only the Euro-to-Yen conversion rates in that relation:

```
12:10> SELECT TABLE * FROM YenRates WHERE Curr = "Euro";
--------------------------------------
| Curr | Rate | EventTime | ProcTime |
--------------------------------------
| Euro | 114 | 12:00:30 | 12:06:23 |
```
```
4
```

```
| Euro | 116 | 12:03:00 | 12:09:07 |
| Euro | 119 | 12:06:00 | 12:07:33 |
--------------------------------------
```
From a database engineering perspective, we understand that these values
don’t mean that the rate for converting Euros to Yen is 114 ¥/€ at precisely
12:00, 116 ¥/€ at 12:03, 119 ¥/€ at 12:06, and undefined at all other times.
Instead, we know that the intent of this table is to capture the fact that the
conversion rate for Euros to Yen is undefined until 12:00, 114 ¥/€ from 12:00
to 12:03, 116 ¥/€ from 12:03 to 12:06, and 119 ¥/€ from then on. Or drawn
out in a timeline:

```
Undefined 114 ¥/€ 116 ¥/€
119 ¥/€
|----[-inf, 12:00)----|----[12:00, 12:03)----|----[12:03, 12:06)----|--
--[12:06, now)----→
```
Now, if we knew all of the rates ahead of time, we could capture these
regions explicitly in the row data themselves. But if we instead need to build
up these regions incrementally, based only upon the start times at which a
given rate becomes valid, we have a problem: the region for a given row will
change over time depending on the rows that come after it. This is a problem
even if the data arrive in order (because every time a new rate arrives, the
previous rate changes from being valid forever to being valid until the arrival
time of the new rate), but is further compounded if they can arrive _out of
order_. For example, using the processing-time ordering in the preceding

YenRates table, the sequence of timelines our table would effectively
represent over time would be as follows:

```
Range of processing time | Event-time validity timeline during that
range of processing-time
=========================|=============================================
=================================
|
| Undefined
[-inf, 12:06:23) | |--[-inf, +inf)-----------------------------
----------------------------→
```

```
|
| Undefined 114 ¥/€
[12:06:23, 12:07:33) | |--[-inf, 12:00)--|--[12:00, +inf)----------
----------------------------→
|
| Undefined 114 ¥/€
119 ¥/€
[12:07:33, 12:09:07) | |--[-inf, 12:00)--|--[12:00, 12:06)---------
------------|--[12:06, +inf)→
|
| Undefined 114 ¥/€
116 ¥/€ 119 ¥/€
[12:09:07, now) | |--[-inf, 12:00)--|--[12:00, 12:03)--|--
[12:03, 12:06)--|--[12:06, +inf)→
```
Or, if we wanted to render this as a time-varying relation (with changes
between each snapshot relation highlighted in yellow):

```
12:10> SELECT TVR * FROM YenRatesWithRegion ORDER BY
EventTime;
-----------------------------------------------------------------------
----------------------
| [-inf, 12:06:23) | [12:06:23,
12:07:33) |
| ------------------------------------------- | -----------------------
-------------------- |
| | Curr | Rate | Region | ProcTime | | | Curr | Rate | Region
| ProcTime | |
| ------------------------------------------- | -----------------------
-------------------- |
| ------------------------------------------- | | Euro | 114 | [12:00,
+inf) | 12:06:23 | |
| | -----------------------
-------------------- |
-----------------------------------------------------------------------
----------------------
| [12:07:33, 12:09:07) | [12:09:07,
+inf) |
| ------------------------------------------- | -----------------------
-------------------- |
| | Curr | Rate | Region | ProcTime | | | Curr | Rate | Region
| ProcTime | |
```

```
| ------------------------------------------- | -----------------------
-------------------- |
| | Euro | 114 | [12:00, 12:06) | 12:06:23 | | | Euro | 114 | [12:00,
12:03) | 12:06:23 | |
| | Euro | 119 | [12:06, +inf) | 12:07:33 | | | Euro | 116 | [12:03,
12:06) | 12:09:07 | |
| ------------------------------------------- | | Euro | 119 | [12:06,
+inf) | 12:07:33 | |
| | -----------------------
-------------------- |
-----------------------------------------------------------------------
----------------------
```
What’s important to note here is that half of the changes involve updates to
multiple rows. That maybe doesn’t sound so bad, until you recall that the
difference between each of these snapshots is the arrival of exactly one new
row. In other words, the arrival of a single new input row results in
transactional modifications to multiple output rows. That sounds less good.
On the other hand, it also sounds a lot like the multirow transactions involved
in building up session windows. And indeed, this is yet another example of
windowing providing benefits beyond simple partitioning of time: it also
affords the ability to do so in ways that involve complex, multirow
transactions.

To see this in action, let’s look at an animation. If this were a Beam pipeline,
it would probably look something like the following:

```
PCollection<Currency, Decimal> yenRates = ...;
PCollection<Decimal> validYenRates = yenRates
.apply(Window.into(new ValidityWindows())
.apply(GroupByKey.<Currency, Decimal>create());
```
Rendered in a streams/tables animation, that pipeline would look like that
shown in Figure 9-1.


```
Figure 9-1. Temporal validity windowing over time
```
This animation highlights a critical aspect of temporal validity: shrinking
windows. Validity windows must be able to shrink over time, thereby
diminishing the reach of their validity and splitting any data contained therein
across the two new windows. See the code snippets on GitHub for an
example partial implementation.

In SQL terms, the creation of these validity windows would look something
like the following (making using of a hypothetical VALIDITY_WINDOW
construct), viewed as a table:

```
12:10> SELECT TABLE
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
```
```
00:00 / 00:00
```
```
5
```

```
VALIDITY_WINDOW(EventTime)
HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window |
--------------------------------
| Euro | 114 | [12:00, 12:03) |
| Euro | 116 | [12:03, 12:06) |
| Euro | 119 | [12:06, +inf) |
--------------------------------
```
### VALIDITY WINDOWS IN STANDARD SQL

```
Note that it’s possible to describe validity windows in standard SQL
using a three-way self-join:
```
```
SELECT
r1.Curr,
MAX(r1.Rate) AS Rate,
r1.EventTime AS WindowStart,
r2.EventTime AS WIndowEnd
FROM YenRates r1
LEFT JOIN YenRates r2
ON r1.Curr = r2.Curr
AND r1.EventTime < r2.EventTime
LEFT JOIN YenRates r3
ON r1.Curr = r3.Curr
AND r1.EventTime < r3.EventTime
AND r3.EventTime < r2.EventTime
WHERE r3.EventTime IS NULL
GROUP BY r1.Curr, WindowStart, WindowEnd
HAVING r1.Curr = 'Euro';
```
```
Thanks to Martin Kleppmann for pointing this out.
```
Or, perhaps more interestingly, viewed as a stream:

```
12:00> SELECT STREAM
Curr,
MAX(Rate) as Rate,
```

```
VALIDITY_WINDOW(EventTime) as Window,
Sys.EmitTime as Time,
Sys.Undo as Undo,
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime)
HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window | Time | Undo |
--------------------------------------------------
| Euro | 114 | [12:00, +inf) | 12:06:23 | |
| Euro | 114 | [12:00, +inf) | 12:07:33 | undo |
| Euro | 114 | [12:00, 12:06) | 12:07:33 | |
| Euro | 119 | [12:06, +inf) | 12:07:33 | |
| Euro | 114 | [12:00, 12:06) | 12:09:07 | undo |
| Euro | 114 | [12:00, 12:03) | 12:09:07 | |
| Euro | 116 | [12:03, 12:06) | 12:09:07 | |
................. [12:00, 12:10] .................
```
Great, we have an understanding of how to use point-in-time values to
effectively slice up time into ranges within which those values are valid. But
the real power of these temporal validity windows is when they are applied in
the context of joining them with other data. That’s where temporal validity
joins come in.

**Temporal validity joins**

To explore the semantics of temporal validity joins, suppose that our
financial application contains another time-varying relation, one that tracks
currency-conversion orders from various currencies to Yen:

```
12:10> SELECT TABLE * FROM YenOrders;
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2 | 12:02:00 | 12:05:07 |
| USD | 1 | 12:03:00 | 12:03:44 |
| Euro | 5 | 12:05:00 | 12:08:00 |
```

```
| Yen | 50 | 12:07:00 | 12:10:11 |
| Euro | 3 | 12:08:00 | 12:09:33 |
| USD | 5 | 12:10:00 | 12:10:59 |
----------------------------------------
```
And for simplicity, as before, let’s focus on the Euro conversions:

```
12:10> SELECT TABLE * FROM YenOrders WHERE Curr = "Euro";
----------------------------------------
| Curr | Amount | EventTime | ProcTime |
----------------------------------------
| Euro | 2 | 12:02:00 | 12:05:07 |
| Euro | 5 | 12:05:00 | 12:08:00 |
| Euro | 3 | 12:08:00 | 12:09:33 |
----------------------------------------
```
We’d like to robustly join these orders to the YenRates relation, treating the
rows in YenRates as defining validity windows. As such, we’ll actually want
to join to the validity-windowed version of the YenRates relation we
constructed at the end of the last section:

```
12:10> SELECT TABLE
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime)
HAVING Curr = "Euro";
--------------------------------
| Curr | Rate | Window |
--------------------------------
| Euro | 114 | [12:00, 12:03) |
| Euro | 116 | [12:03, 12:06) |
| Euro | 119 | [12:06, +inf) |
--------------------------------
```

Fortunately, after we have our conversion rates placed into validity windows,
a windowed join between those rates and the YenOrders relation gives us
exactly what we want:

```
12:10> WITH ValidRates AS
(SELECT
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime))
SELECT TABLE
YenOrders.Amount as "E",
ValidRates.Rate as "Y/E",
YenOrders.Amount * ValidRates.Rate as "Y",
YenOrders.EventTime as Order,
ValidRates.Window as "Rate Window"
FROM YenOrders FULL OUTER JOIN ValidRates
ON YenOrders.Curr = ValidRates.Curr
AND WINDOW_START(ValidRates.Window) <=
YenOrders.EventTime
AND YenOrders.EventTime <
WINDOW_END(ValidRates.Window)
HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y | Order | Rate Window |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | [12:00, 12:03) |
| 5 | 116 | 580 | 12:05 | [12:03, 12:06) |
| 3 | 119 | 357 | 12:08 | [12:06, +inf) |
-------------------------------------------
```
Thinking back to our original YenRates and YenOrders relations, this joined
relation indeed looks correct: each of the three conversions ended up with the
(eventually) appropriate rate for the given window of event time within


which their corresponding order fell. So we have a decent sense that this join
is doing what we want in terms of providing us the eventual correctness we
want.

That said, this simple snapshot view of the relation, taken after all the values
have arrived and the dust has settled, belies the complexity of this join. To
really understand what’s going on here, we need to look at the full TVR.
First, recall that the validity-windowed conversion rate relation was actually
much more complex than the previous simple table snapshot view might lead
you to believe. For reference, here’s the STREAM version of the validity
windows relation, which better highlights the evolution of those conversion
rates over time:

```
12:00> SELECT STREAM
Curr,
MAX(Rate) as Rate,
VALIDITY(EventTime) as Window,
Sys.EmitTime as Time,
Sys.Undo as Undo,
FROM YenRates
GROUP BY
Curr,
VALIDITY(EventTime)
HAVING Curr = "Euro";
--------------------------------------------------
| Curr | Rate | Window | Time | Undo |
--------------------------------------------------
| Euro | 114 | [12:00, +inf) | 12:06:23 | |
| Euro | 114 | [12:00, +inf) | 12:07:33 | undo |
| Euro | 114 | [12:00, 12:06) | 12:07:33 | |
| Euro | 119 | [12:06, +inf) | 12:07:33 | |
| Euro | 114 | [12:00, 12:06) | 12:09:07 | undo |
| Euro | 114 | [12:00, 12:03) | 12:09:07 | |
| Euro | 116 | [12:03, 12:06) | 12:09:07 | |
................. [12:00, 12:10] .................
```
As a result, if we look at the full TVR for our validity-windowed join, you
can see that the corresponding evolution of this join over time is much more


complicated, due to the out-of-order arrival of values on both sides of the
join:

```
12:10> WITH ValidRates AS
(SELECT
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime))
SELECT TVR
YenOrders.Amount as "E",
ValidRates.Rate as "Y/E",
YenOrders.Amount * ValidRates.Rate as "Y",
YenOrders.EventTime as Order,
ValidRates.Window as "Rate Window"
FROM YenOrders FULL OUTER JOIN ValidRates
ON YenOrders.Curr = ValidRates.Curr
AND WINDOW_START(ValidRates.Window) <=
YenOrders.EventTime
AND YenOrders.EventTime <
WINDOW_END(ValidRates.Window)
HAVING Curr = "Euro";
-----------------------------------------------------------------------
--------------------
| [-inf, 12:05:07) | [12:05:07,
12:06:23) |
| ------------------------------------------ | ------------------------
------------------ |
| | E | Y/E | Y | Order | Rate Window | | | E | Y/E | Y | Order
| Rate Window | |
| ------------------------------------------ | ------------------------
------------------ |
| ------------------------------------------ | | 2 | | | 12:02
| | |
| | ------------------------
------------------ |
```

-----------------------------------------------------------------------
--------------------
| [12:06:23, 12:07:33) | [12:07:33,
12:08:00) |
| ------------------------------------------ | ------------------------
------------------ |
| | E | Y/E | Y | Order | Rate Window | | | E | Y/E | Y | Order
| Rate Window | |
| ------------------------------------------ | ------------------------
------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, +inf) | | | 2 | 114 | 228 | 12:02
| [12:00, 12:06) | |
| ------------------------------------------ | | | 119 | |
| [12:06, +inf) | |
| | ------------------------
------------------ |
-----------------------------------------------------------------------
--------------------
| [12:08:00, 12:09:07) | [12:09:07,
12:09:33) |
| ------------------------------------------ | ------------------------
------------------ |
| | E | Y/E | Y | Order | Rate Window | | | E | Y/E | Y | Order
| Rate Window | |
| ------------------------------------------ | ------------------------
------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, 12:06) | | | 2 | 114 | 228 | 12:02
| [12:00, 12:03) | |
| | 5 | 114 | 570 | 12:05 | [12:03, 12:06) | | | 5 | 116 | 580 | 12:05
| [12:03, 12:06) | |
| | | 119 | | | [12:06, +inf) | | | | 119 | | 12:08
| [12:06, +inf) | |
| ------------------------------------------ | ------------------------
------------------ |
-----------------------------------------------------------------------
--------------------
| [12:09:33, now) |
| ------------------------------------------ |
| | E | Y/E | Y | Order | Rate Window | |
| ------------------------------------------ |
| | 2 | 114 | 228 | 12:02 | [12:00, 12:03) | |
| | 5 | 116 | 580 | 12:05 | [12:03, 12:06) | |
| | 3 | 119 | 357 | 12:08 | [12:06, +inf) | |


```
| ------------------------------------------ |
----------------------------------------------
```
In particular, the result for the 5 € order is originally quoted at 570 ¥ because
that order (which happened at 12:05) originally falls into the validity window
for the 114 ¥/€ rate. But when the 116 ¥/€ rate for event time 12:03 arrives
out of order, the result for the 5 € order must be updated from 570 ¥ to 580 ¥.
This is also evident if you observe the results of the join as a stream (here
I’ve highlighted the incorrect 570 ¥ in red, and the retraction for 570 ¥ and
subsequent corrected value of 580 ¥ in blue):

```
12:00> WITH ValidRates AS
(SELECT
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime))
SELECT STREAM
YenOrders.Amount as "E",
ValidRates.Rate as "Y/E",
YenOrders.Amount * ValidRates.Rate as "Y",
YenOrders.EventTime as Order,
ValidRates.Window as "Rate Window",
Sys.EmitTime as Time,
Sys.Undo as Undo
FROM YenOrders FULL OUTER JOIN ValidRates
ON YenOrders.Curr = ValidRates.Curr
AND WINDOW_START(ValidRates.Window) <=
YenOrders.EventTime
AND YenOrders.EventTime <
WINDOW_END(ValidRates.Window)
HAVING Curr = “Euro”;
------------------------------------------------------------
| E | Y/E | Y | Order | Rate Window | Time | Undo |
```

```
------------------------------------------------------------
| 2 | | | 12:02 | | 12:05:07 | |
| 2 | | | 12:02 | | 12:06:23 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, +inf) | 12:06:23 | |
| 2 | 114 | 228 | 12:02 | [12:00, +inf) | 12:07:33 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:07:33 | |
| | 119 | | | [12:06, +inf) | 12:07:33 | |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:08:00 | |
| 2 | 114 | 228 | 12:02 | [12:00, 12:06) | 12:09:07 | undo |
| 5 | 114 | 570 | 12:05 | [12:00, 12:06) | 12:09:07 | undo |
| 2 | 114 | 228 | 12:02 | [12:00, 12:03) | 12:09:07 | |
| 5 | 116 | 580 | 12:05 | [12:03, 12:06) | 12:09:07 | |
| | 119 | | | [12:06, +inf) | 12:09:33 | undo |
| 3 | 119 | 357 | 12:08 | [12:06, +inf) | 12:09:33 | |
...................... [12:00, 12:10] ......................
```
It’s worth calling out that this is a fairly messy stream due to the use of a
FULL OUTER join. In reality, when consuming conversion orders as a stream,
you probably don’t care about unjoined rows; switching to an INNER join
helps eliminate those rows. You probably also don’t care about cases for
which the rate window changes, but the actual conversion value isn’t
affected. By removing the rate window from the stream, we can further
decrease its chattiness:

```
12:00> WITH ValidRates AS
(SELECT
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime))
SELECT STREAM
YenOrders.Amount as "E",
ValidRates.Rate as "Y/E",
YenOrders.Amount * ValidRates.Rate as "Y",
YenOrders.EventTime as Order,
```

```
ValidRates.Window as "Rate Window",
Sys.EmitTime as Time,
Sys.Undo as Undo
FROM YenOrders INNER JOIN ValidRates
ON YenOrders.Curr = ValidRates.Curr
AND WINDOW_START(ValidRates.Window) <=
YenOrders.EventTime
AND YenOrders.EventTime <
WINDOW_END(ValidRates.Window)
HAVING Curr = "Euro";
-------------------------------------------
| E | Y/E | Y | Order | Time | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:06:23 | |
| 5 | 114 | 570 | 12:05 | 12:08:00 | |
| 5 | 114 | 570 | 12:05 | 12:09:07 | undo |
| 5 | 116 | 580 | 12:05 | 12:09:07 | |
| 3 | 119 | 357 | 12:08 | 12:09:33 | |
............. [12:00, 12:10] ..............
```
Much nicer. We can now see that this query very succinctly does what we
originally set out to do: join two TVRs for currency conversion rates and
orders in a robust way that is tolerant of data arriving out of order. Figure 9-2
visualizes this query as an animated diagram. In it, you can also very clearly
see the way the overall structure of things change as they evolve over time.


```
Figure 9-2. Temporal validity join, converting Euros to Yen with per-record triggering
```
_Watermarks and temporal validity joins_

With this example, we’ve highlighted the first benefit of windowed joins
called out at the beginning of this section: windowing a join allows you to
partition that join within time for some practical business need. In this case,
the business need was slicing time into regions of validity for our currency
conversion rates.

Before we call it a day, however, it turns out that this example also provides
an opportunity to highlight the second point I called out: the fact that
windowing a join can provide a meaningful reference point for watermarks.
To see how that’s useful, imagine changing the previous query to replace the
implicit default per-record trigger with an explicit watermark trigger that
would fire only once when the watermark passed the end of the validity
window in the join (assuming that we have a watermark available for both of
our input TVRs that accurately tracks the completeness of those relations in
event time as well as an execution engine that knows how to take those
watermarks into consideration). Now, instead of our stream containing
multiple outputs and retractions for rates arriving out of order, we could

```
00:00 / 00:00
```

instead end up with a stream containing a single, correct converted result per
order, which is clearly even more ideal than before:

```
12:00> WITH ValidRates AS
(SELECT
Curr,
MAX(Rate) as Rate,
VALIDITY_WINDOW(EventTime) as Window
FROM YenRates
GROUP BY
Curr,
VALIDITY_WINDOW(EventTime))
SELECT STREAM
YenOrders.Amount as "E",
ValidRates.Rate as "Y/E",
YenOrders.Amount * ValidRates.Rate as "Y",
YenOrders.EventTime as Order,
Sys.EmitTime as Time,
Sys.Undo as Undo
FROM YenOrders INNER JOIN ValidRates
ON YenOrders.Curr = ValidRates.Curr
AND WINDOW_START(ValidRates.Window) <=
YenOrders.EventTime
AND YenOrders.EventTime <
WINDOW_END(ValidRates.Window)
HAVING Curr = "Euro"
EMIT WHEN WATERMARK PAST
WINDOW_END(ValidRates.Window);
-------------------------------------------
| E | Y/E | Y | Order | Time | Undo |
-------------------------------------------
| 2 | 114 | 228 | 12:02 | 12:08:52 | |
| 5 | 116 | 580 | 12:05 | 12:10:04 | |
| 3 | 119 | 357 | 12:08 | 12:10:13 | |
............. [12:00, 12:11] ..............
```
Or, rendered as an animation, which clearly shows how joined results are not
emitted into the output stream until the watermark moves beyond them, as


demonstrated in Figure 9-3.

```
Figure 9-3. Temporal validity join, converting Euros to Yen with watermark triggering
```
Either way, it’s impressive to see how this query encapsulates such a
complex set of interactions into a clean and concise rendering of the desired
results.
