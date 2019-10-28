
 ## Stream-and-Table Basics Or: a Special Theory of

## Stream and Table Relativity

The basic idea of streams and tables derives from the database world. Anyone
familiar with SQL is likely familiar with tables and their core properties,
roughly summarized as: tables contain rows and columns of data, and each
row is uniquely identified by some sort of key, either explicit or implicit.

If you think back to your database systems class in college, you’ll probably
recall the data structure underlying most databases is an _append-only log_. As
transactions are applied to a table in the database, those transactions are
recorded in a log, the contents of which are then serially applied to the table
to materialize those updates. In streams and tables nomenclature, that log is
effectively the stream.

From that perspective, we now understand how to create a table from a
stream: the table is just the result of applying the transaction log of updates
found in the stream. But how to do we create a stream from a table? It’s
essentially the inverse: a stream is a changelog for a table. The motivating
example typically used for table-to-stream conversion is _materialized views_.
Materialized views in SQL let you specify a query on a table, which itself is
then manifested by the database system as another first-class table. This
materialized view is essentially a cached version of that query, which the

```
1
```

database system ensures is always up to date as the contents of the source
table evolve over time. Perhaps unsurprisingly, materialized views are
implemented via the changelog for the original table; any time the source
table changes, that change is logged. The database then evaluates that change
within the context of the materialized view’s query and applies any resulting
change to the destination materialized view table.

Combining these two points together and employing yet another questionable
physics analogy, we arrive at what one might call the Special Theory of
Stream and Table Relativity:

Streams → tables

```
The aggregation of a stream of updates over time yields a table.
```
Tables → streams

```
The observation of changes to a table over time yields a stream.
```
This is a very powerful pair of concepts, and their careful application to the
world of stream processing is a big reason for the massive success of Apache
Kafka, the ecosystem that is built around these underlying principles.
However, those statements themselves are not quite general enough to allow
us to tie streams and tables to all of the concepts in the Beam Model. For that,
we must go a little bit deeper.

## Toward a General Theory of Stream and Table

## Relativity

If we want to reconcile stream/table theory with everything we know of the
Beam Model, we’ll need to tie up some loose ends, specifically:

```
How does batch processing fit into all of this?
What is the relationship of streams to bounded and unbounded
datasets?
```
```
How do the four what , where , when , how questions map onto a
streams/tables world?
```

As we attempt to do so, it will be helpful to have the right mindset about
streams and tables. In addition to understanding them in relation to each
other, as captured by the previous definition, it can be illuminating to define
them independent of each other. Here’s a simple way of looking at it that will
underscore some of our future analyses:

```
Tables are data at rest.
This isn’t to say tables are static in any way; nearly all useful tables
are continuously changing over time in some way. But at any given
time, a snapshot of the table provides some sort of picture of the
dataset contained together as a whole. In that way, tables act as a
conceptual resting place for data to accumulate and be observed over
time. Hence, data at rest.
```
```
Streams are data in motion.
Whereas tables capture a view of the dataset as a whole at a specific
point in time , streams capture the evolution of that data over time.
Julian Hyde is fond of saying streams are like the derivatives of
tables, and tables the integrals of streams, which is a nice way of
thinking about it for you math-minded individuals out there.
Regardless, the important feature of streams is that they capture the
inherent movement of data within a table as it changes. Hence, data
in motion.
```
Though tables and streams are intimately related, it’s important to keep in
mind that they are very much _not_ the same thing, even if there are many cases
in which one might be fully derived from the other. The differences are subtle
but important, as we’ll see.


