 # Chapter 7. The Practicalities of

# Persistent State

Why do people write books? When you factor out the joy of creativity, a
certain fondness for grammar and punctuation, and perhaps the occasional
touch of narcissism, you’re basically left with the desire to capture an
otherwise ephemeral idea so that it can be revisited in the future. At a very
high level, I’ve just motivated and explained persistent state in data
processing pipelines.

Persistent state is, quite literally, the tables we just talked about in Chapter 6,
with the additional requirement that the tables be robustly stored in a media
relatively immune to loss. Stored on local disk counts, as long as you don’t
ask your Site Reliability Engineers. Stored on a replicated set of disks is
better. Stored on a replicated set of disks in distinct physical locations is
better still. Stored in memory once definitely doesn’t count. Stored in
replicated memory across multiple machines with UPS power backup and
generators onsite maybe does. You get the picture.

In this chapter, our objective is to do the following:

```
Motivate the need for persistent state within pipelines
```
```
Look at two forms of implicit state often found within pipelines
Consider a real-world use case (advertising conversion attribution)
that lends itself poorly to implicit state, use that to motivate the
salient features of a general, explicit form of persistent state
management
Explore a concrete manifestation of one such state API, as found in
Apache Beam
```

