 ## Generalized State

```
00:00 / 00:00
```

Though both of the implicit approaches we’ve looked at so far have their
merits, they each fall short in one dimension: flexibility. The raw grouping
method requires you to always buffer up the raw inputs to the grouping
operation before processing the group in whole, so there’s no way to partially
process some of the data along the way; it’s all or nothing. The incremental
combining approach specifically allows for partial processing but with the
restriction that the processing in question be commutative and associative and
happen as records arrive one-by-one.

If we want to support a more generalized approach to streaming persistent
state, we need something more flexible. Specifically, we need flexibility in
three dimensions:

```
Flexibility in data structures; that is, an ability to structure the data
we write and read in ways that are most appropriate and efficient for
the task at hand. Raw grouping essentially provides an appendable
list, and incremental combination essentially provides a single value
that is always written and read in its entirety. But there are myriad
other ways in which we might want to structure our persistent data,
each with different types of access patterns and associated costs:
maps, trees, graphs, sets, and so on. Supporting a variety of
persistent data types is critical for efficiency.
Beam supports flexibility in data types by allowing a single DoFn to
declare multiple state fields, each of a specific type. In this way,
logically independent pieces of state (e.g., visits and impressions)
can be stored separately, and semantically different types of state
(e.g., maps and lists) can be accessed in ways that are natural given
their types of access patterns.
Flexibility in write and read granularity; that is, an ability to tailor
the amount and type of data written or read at any given time for
optimal efficiency. What this boils down to is the ability to write and
read precisely the necessary amount of data at any given point of
time: no more, and no less (and in parallel as much as possible).
This goes hand in hand with the previous point, given that dedicated
```

```
data types allow for focused types of access patterns (e.g., a set-
membership operation that can use something like a Bloom filter
under the covers to greatly minimize the amount of data read in
certain circumstances). But it goes beyond it, as well; for example,
allowing multiple large reads to be dispatched in parallel (e.g., via
futures).
In Beam, flexibly granular writes and reads are enabled via datatype-
specific APIs that provide fine-grained access capabilities, combined
with an asynchronous I/O mechanism that allows for writes and
reads to be batched together for efficiency.
```
```
Flexibility in scheduling of processing; that is, an ability to bind the
time at which specific types of processing occur to the progress of
time in either of the two time domains we care about: event-time
completeness and processing time. Triggers provide a restricted set
of flexibility here, with completeness triggers providing a way to
bind processing to the watermark passing the end of the window,
and repeated update triggers providing a way to bind processing to
periodic progress in the processing-time domain. But for certain use
cases (e.g., certain types of joins, for which you don’t necessarily
care about input completeness of the entire window, just input
completeness up to the event-time of a specific record in the join),
triggers are insufficiently flexible. Hence, our need for a more
general solution.
In Beam, flexible scheduling of processing is provided via timers. A
timer is a special type of state that binds a specific point in time in
either supported time domain (event time or processing time) with a
method to be called when that point in time is reached. In this way,
specific bits of processing can be delayed until a more appropriate
time in the future.
```
The common thread among these three characteristics is _flexibility_. A specific
subset of use cases are served very well by the relatively inflexible
approaches of raw grouping or incremental combination. But when tackling

```
4
```

anything outside their relatively narrow domain of expertise, those options
often fall short. When that happens, you need the power and flexibility of a
fully general-state API to let you tailor your utilization of persistent state
optimally.

To think of it another way, raw grouping and incremental combination are
relatively high-level abstractions that enable the pithy expression of pipelines
with (in the case of combiners, at least) some good properties for automatic
optimizations. But sometimes you need to go low level to get the behavior or
performance you need. That’s what generalized state lets you do.

## Case Study: Conversion Attribution

To see this in action, let’s now look at a use case that is poorly served by both
raw grouping and incremental combination: _conversion attribution_. This is a
technique that sees widespread use across the advertising world to provide
concrete feedback on the effectiveness of advertisements. Though relatively
easy to understand, its somewhat diverse set of requirements doesn’t fit
nicely into either of the two types of implicit state we’ve considered so far.

Imagine that you have an analytics pipeline that monitors traffic to a website
in conjunction with advertisement impressions that directed traffic to that
site. The goal is to provide attribution of specific advertisements shown to a
user toward the achievement of some goal on the site itself (which often
might lie many steps beyond the initial advertisement landing page), such as
signing up for a mailing list or purchasing an item.

Figure 7-3 shows an example set of website visits, goals, and ad impressions,
with one attributed conversion highlighted in red. Building up conversion
attributions over an unbounded, out-of-order stream of data requires keeping
track of impressions, visits, and goals seen so far. That’s where persistent
state comes in.


```
Figure 7-3. Example conversion attribution
```
In this diagram, a user’s traversal of various pages on a website is represented
as a graph. Impressions are advertisements that were shown to the user and
clicked, resulting in the user visiting a page on the site. Visits represent a
single page viewed on the site. Goals are specific visited pages that have been
identified as a desired destination for users (e.g., completing a purchase, or
signing up for a mailing list). The goal of conversion attribution is to identify
ad impressions that resulted in the user achieving some goal on the site. In
this figure, there is one such conversion highlighted in red. Note that events
might arrive out of order, hence the event-time axis in the diagram and the
watermark reference point indicating the time up to which input is believed to
be correct.

A lot goes into building a robust, large-scale attribution pipeline, but there are


a few aspects worth calling out explicitly. Any such pipeline we attempt to
build must do the following:

Handle out-of-order data

```
Because the website traffic and ad impression data come from separate
systems, both of which are implemented as distributed collection services
themselves, the data might arrive wildly out of order. Thus, our pipeline
must be resilient to such disorder.
```
Handle high volumes of data

```
Not only must we assume that this pipeline will be processing data for a
large number of independent users, but depending upon the volume of a
given ad campaign and the popularity of a given website, we might need
to store a large amount of impression and/or traffic data as we attempt to
build evidence of attribution. For example, it would not be unheard of to
store 90 days worth of visit, impression, and goal tree data per user to
allow us to build up attributions that span multiple months’ worth of
activity.
```
Protect against spam

```
Given that money is involved, correctness is paramount. Not only must
we ensure that visits and impressions are accounted for exactly once
(something we’ll get more or less for free by simply using an execution
engine that supports effectively-once processing), but we must also guard
our advertisers against spam attacks that attempt to charge advertisers
unfairly. For example, a single ad that is clicked multiple times in a row
by the same user will arrive as multiple impressions, but as long as those
clicks occur within a certain amount of time of one another (e.g., within
the same day), they must be attributed only once. In other words, even if
the system guarantees we’ll see every individual impression once, we
must also perform some manual deduplication across impressions that are
technically different events but which our business logic dictates we
interpret as duplicates.
```
Optimize for performance

```
5
```

```
Above all, because of the potential scale of this pipeline, we must always
keep an eye toward optimizing the performance of our pipeline. Persistent
state, because of the inherent costs of writing to persistent storage, can
often be the performance bottleneck in such a pipeline. As such, the
flexibility characteristics we discussed earlier will be critical in ensuring
our design is as performant as possible.
```
## Conversion Attribution with Apache Beam

Now that we understand the basic problem that we’re trying to solve and
have some of the important requirements squarely in mind, let’s use Beam’s
State and Timers API to build a basic conversion attribution transformation.

We’ll write this just like we would any other DoFn in Beam, but we’ll make
use of state and timer extensions that allow us to write and read persistent
state and timer fields. Those of you that want to follow along in real code can
find the full implementation on GitHub.

Note that, as with all grouping operations in Beam, usage of the State API is
scoped to the current key and window, with window lifetimes dictated by the
specified allowed lateness parameter; in this example, we’ll be operating
within a single global window. Parallelism is linearized per key, as with most

DoFns. Also note that, for simplicity, we’ll be eliding the manual garbage
collection of visits and impressions falling outside of our 90-day horizon that
would be necessary to keep the persisted state from growing forever.

To begin, let’s define a few POJO classes for visits, impressions, a
visit/impression union (used for joining), and completed attributions, as
shown in Example 7-4.

_Example 7-4. POJO definitions of Visit, Impression, VisitOrImpression, and
Attribution objects_

@DefaultCoder(AvroCoder.class)
class Visit {
@Nullable private String url;
@Nullable private Instant timestamp;
// The referring URL. Recall that we’ve constrained the problem in
this


// example to assume every page on our website has exactly one
possible
// referring URL, to allow us to solve the problem for simple trees
// rather than more general DAGs.
@Nullable private String referer;
@Nullable private boolean isGoal;

@SuppressWarnings("unused")
public Visit() {
}

public Visit(String url, Instant timestamp, String referer,
boolean isGoal) {
this.url = url;
this.timestamp = timestamp;
this.referer = referer;
this.isGoal = isGoal;
}

public String url() { return url; }
public Instant timestamp() { return timestamp; }
public String referer() { return referer; }
public boolean isGoal() { return isGoal; }

@Override
public String toString() {
return String.format("{ %s %s from:%s%s }", url, timestamp,
referer,
isGoal? " isGoal" : "");
}
}

@DefaultCoder(AvroCoder.class)
class Impression {
@Nullable private Long id;
@Nullable private String sourceUrl;
@Nullable private String targetUrl;
@Nullable private Instant timestamp;

public static String sourceAndTarget(String source, String target) {
return source + ":" + target;
}


@SuppressWarnings("unused")
public Impression() {
}

public Impression(Long id, String sourceUrl, String targetUrl,
Instant timestamp) {
this.id = id;
this.sourceUrl = sourceUrl;
this.targetUrl = targetUrl;
this.timestamp = timestamp;
}

public Long id() { return id; }
public String sourceUrl() { return sourceUrl; }
public String targetUrl() { return targetUrl; }
public String sourceAndTarget() {
return sourceAndTarget(sourceUrl, targetUrl);
}
public Instant timestamp() { return timestamp; }

@Override
public String toString() {
return String.format("{ %s source:%s target:%s %s }",
id, sourceUrl, targetUrl, timestamp);
}
}

@DefaultCoder(AvroCoder.class)
class VisitOrImpression {
@Nullable private Visit visit;
@Nullable private Impression impression;

@SuppressWarnings("unused")
public VisitOrImpression() {
}

public VisitOrImpression(Visit visit, Impression impression) {
this.visit = visit;
this.impression = impression;
}

public Visit visit() { return visit; }
public Impression impression() { return impression; }


}

@DefaultCoder(AvroCoder.class)
class Attribution {
@Nullable private Impression impression;
@Nullable private List<Visit> trail;
@Nullable private Visit goal;

@SuppressWarnings("unused")
public Attribution() {
}

public Attribution(Impression impression, List<Visit> trail, Visit
goal) {
this.impression = impression;
this.trail = trail;
this.goal = goal;
}

public Impression impression() { return impression; }
public List<Visit> trail() { return trail; }
public Visit goal() { return goal; }

@Override
public String toString() {
StringBuilder builder = new StringBuilder();
builder.append("imp=" + impression.id() + " " +
impression.sourceUrl());
for (Visit visit : trail) {
builder.append(" → " + visit.url());
}
builder.append(" → " + goal.url());
return builder.toString();
}
}

We next define a Beam DoFn to consume a flattened collection of Visits and
Impressions, keyed by the user. In turn, it will yield a collection of
Attributions. Its signature looks like Example 7-5.

_Example 7-5. DoFn signature for our conversion attribution transformation_

class AttributionFn extends DoFn<KV<String, VisitOrImpression>,


Attribution>

Within that DoFn, we need to implement the following logic:

```
1. Store all visits in a map keyed by their URL so that we can easily
look them up when tracing visit trails backward from a goal.
```
```
2. Store all impressions in a map keyed by the URL they referred to, so
we can identify impressions that initiated a trail to a goal.
3. Any time we see a visit that happens to be a goal, set an event-time
timer for the timestamp of the goal. Associated with this timer will
be a method that performs goal attribution for the pending goal. This
will ensure that attribution only happens once the input leading up to
the goal is complete.
```
```
4. Because Beam lacks support for a dynamic set of timers (currently
all timers must be declared at pipeline definition time, though each
individual timer can be set and reset for different points in time at
runtime), we also need to keep track of the timestamps for all of the
goals we still need to attribute. This will allow us to have a single
attribution timer set for the minimum timestamp of all pending
goals. After we attribute the goal with the earliest timestamp, we set
the timer again with the timestamp of the next earliest goal.
```
Let’s now walk through the implementation in pieces. First up, we need to
declare specifications for all of our state and timer fields within the DoFn. For
state, the specification dictates the type of data structure for the field itself
(e.g., map or list) as well as the type(s) of data contained therein, and their
associated coder(s); for timers, it dictates the associated time domain. Each

specification is then assigned a unique ID string (via the @StateID/@TimerId
annotations), which will allow us to dynamically associate these
specifications with parameters and methods later on. For our use case, we’ll
define (in Example 7-6) the following:

```
Two MapState specifications for visits and impressions
```

```
A single SetState specification for goals
```
```
A ValueState specification for keeping track of the minimum
pending goal timestamp
```
```
A Timer specification for our delayed attribution logic
```
_Example 7-6. State field specifications_

class AttributionFn extends DoFn<KV<String, VisitOrImpression>,
Attribution> {
@StateId("visits")
private final StateSpec<MapState<String, Visit>> visitsSpec =
StateSpecs.map(StringUtf8Coder.of(), AvroCoder.of(Visit.class));

// Impressions are keyed by both sourceUrl (i.e., the query) and
targetUrl
// (i.e., the click), since a single query can result in multiple
impressions.
// The source and target are encoded together into a single string by
the
// Impression.sourceAndTarget method.
@StateId("impressions")
private final StateSpec<MapState<String, Impression>> impSpec =
StateSpecs.map(StringUtf8Coder.of(),
AvroCoder.of(Impression.class));

@StateId("goals")
private final StateSpec<SetState<Visit>> goalsSpec =
StateSpecs.set(AvroCoder.of(Visit.class));

@StateId("minGoal")
private final StateSpec<ValueState<Instant>> minGoalSpec =
StateSpecs.value(InstantCoder.of());

@TimerId("attribution")
private final TimerSpec timerSpec =
TimerSpecs.timer(TimeDomain.EVENT_TIME);

... continued in Example 7 - 7 below ...

Next up, we implement our core @ProcessElement method. This is the
processing logic that will run every time a new record arrives. As noted


earlier, we need to record visits and impressions to persistent state as well as
keep track of goals and manage the timer that will bind our attribution logic
to the progress of event-time completeness as tracked by the watermark.
Access to state and timers is provided via parameters passed to our

@ProcessElement method, and the Beam runtime invokes our method with
appropriate parameters indicated by @StateId and @TimerId annotations.
The logic itself is then relatively straightforward, as demonstrated in
Example 7-7.

_Example 7-7. @ProcessElement implementation_

... continued from Example 7 - 6 above ...

@ProcessElement
public void processElement(
@Element KV<String, VisitOrImpression> kv,
@StateId("visits") MapState<String, Visit> visitsState,
@StateId("impressions") MapState<String, Impression>
impressionsState,
@StateId("goals") SetState<Visit> goalsState,
@StateId("minGoal") ValueState<Instant> minGoalState,
@TimerId("attribution") Timer attributionTimer) {
Visit visit = kv.getValue().visit();
Impression impression = kv.getValue().impression();

if (visit != null) {
if (!visit.isGoal()) {
LOG.info("Adding visit: {}", visit);
visitsState.put(visit.url(), visit);
} else {
LOG.info("Adding goal (if absent): {}", visit);
goalsState.addIfAbsent(visit);
Instant minTimestamp = minGoalState.read();
if (minTimestamp == null ||
visit.timestamp().isBefore(minTimestamp)) {
LOG.info("Setting timer from {} to {}",
Utils.formatTime(minTimestamp),
Utils.formatTime(visit.timestamp()));
attributionTimer.set(visit.timestamp());
minGoalState.write(visit.timestamp());
}
LOG.info("Done with goal");


}
}
if (impression != null) {
// Dedup logical impression duplicates with the same source and
target URL.
// In this case, first one to arrive (in processing time) wins. A
more
// robust approach might be to pick the first one in event time,
but that
// would require an extra read before commit, so the processing-
time
// approach may be slightly more performant.
LOG.info("Adding impression (if absent): {} → {}",
impression.sourceAndTarget(), impression);
impressionsState.putIfAbsent(impression.sourceAndTarget(),
impression);
}
}

... continued in Example 7 - 8 below ...

Note how this ties back to our three desired capabilities in a general state
API:

Flexibility in data structures

```
We have maps, a set, a value, and a timer. They allow us to efficiently
manipulate our state in ways that are effective for our algorithm.
```
Flexibility in write and read granularity

```
Our @ProcessElement method is called for every single visit and
impression we process. As such, we need it to be as efficient as possible.
We take advantage of the ability to make fine-grained, blind writes only
to the specific fields we need. We also only ever read from state within
our @ProcessElement method in the uncommon case of encountering a
new goal. And when we do, we read only a single integer value, without
touching the (potentially much larger) maps and list.
```
Flexibility in scheduling of processing

```
Thanks to timers, we’re able to delay our complex goal attribution logic
```

```
(defined next) until we’re confident we’ve received all the necessary
input data, minimizing duplicated work and maximizing efficiency.
```
Having defined the core processing logic, let’s now look at our final piece of
code, the goal attribution method. This method is annotated with an
@TimerId annotation to identify it as the code to execute when the
corresponding attribution timer fires. The logic here is significantly more
complicated than the @ProcessElement method:

```
1. First, we need to load the entirety of our visit and impression maps,
as well as our set of goals. We need the maps to piece our way
backward through the attribution trail we’ll be building, and we need
the goals to know which goals we’re attributing as a result of the
current timer firing, as well as the next pending goal we want to
schedule for attribution in the future (if any).
2. After we’ve loaded our state, we process goals for this timer one at a
time in a loop, repeatedly:
```
```
Checking to see if any impressions referred the user to the
current visit in the trail (beginning with the goal). If so,
we’ve completed attribution of this goal and can break out
of the loop and emit the attribution trail.
```
```
Checking next to see if any visits were the referrer for the
current visit. If so, we’ve found a back pointer in our trail,
so we traverse it and start the loop over.
```
```
If no matching impressions or visits are found, we have a
goal that was reached organically, with no associated
impression. In this case, we simply break out of the loop
and move on to the next goal, if any.
```
```
3. After we’ve exhausted our list of goals ready for attribution, we set a
timer for the next pending goal in the list (if any) and reset the
corresponding ValueState tracking the minimum pending goal
timestamp.
```

To keep things concise, we first look at the core goal attribution logic, shown
in Example 7-8, which roughly corresponds to point 2 in the preceding list.

_Example 7-8. Goal attribution logic_

... continued from Example 7 - 7 above ...

private Impression attributeGoal(Visit goal,
Map<String, Visit> visits,
Map<String, Impression> impressions,
List<Visit> trail) {
Impression impression = null;
Visit visit = goal;
while (true) {
String sourceAndTarget = Impression.sourceAndTarget(
visit.referer(), visit.url());
LOG.info("attributeGoal: visit={} sourceAndTarget={}",
visit, sourceAndTarget);
if (impressions.containsKey(sourceAndTarget)) {
LOG.info("attributeGoal: impression={}", impression);
// Walked entire path back to impression. Return success.
return impressions.get(sourceAndTarget);
} else if (visits.containsKey(visit.referer())) {
// Found another visit in the path, continue searching.
visit = visits.get(visit.referer());
trail.add( 0 , visit);
} else {
LOG.info("attributeGoal: not found");
// Referer not found, trail has gone cold. Return failure.
return null;
}
}
}

... continued in Example 7 - 9 below ...

The rest of the code (eliding a few simple helper methods), which handles
initializing and fetching state, invoking the attribution logic, and handling
cleanup to schedule any remaining pending goal attribution attempts, looks
like Example 7-9.

_Example 7-9. Overall @TimerId handling logic for goal attribution_


... continued from Example 7 - 8 above ...

@OnTimer("attribution")
public void attributeGoal(
@Timestamp Instant timestamp,
@StateId("visits") MapState<String, Visit> visitsState,
@StateId("impressions") MapState<String, Impression>
impressionsState,
@StateId("goals") SetState<Visit> goalsState,
@StateId("minGoal") ValueState<Instant> minGoalState,
@TimerId("attribution") Timer attributionTimer,
OutputReceiver<Attribution> output) {
LOG.info("Processing timer: {}", Utils.formatTime(timestamp));

// Batch state reads together via futures.
ReadableState<Iterable<Map.Entry<String, Visit> > > visitsFuture
= visitsState.entries().readLater();
ReadableState<Iterable<Map.Entry<String, Impression> > >
impressionsFuture
= impressionsState.entries().readLater();
ReadableState<Iterable<Visit>> goalsFuture = goalsState.readLater();

// Accessed the fetched state.
Map<String, Visit> visits = buildMap(visitsFuture.read());
Map<String, Impression> impressions =
buildMap(impressionsFuture.read());
Iterable<Visit> goals = goalsFuture.read();

// Find the matching goal
Visit goal = findGoal(timestamp, goals);

// Attribute the goal
List<Visit> trail = new ArrayList<>();
Impression impression = attributeGoal(goal, visits, impressions,
trail);
if (impression != null) {
output.output(new Attribution(impression, trail, goal));
impressions.remove(impression.sourceAndTarget());
}
goalsState.remove(goal);

// Set the next timer, if any.
Instant minGoal = minTimestamp(goals, goal);


if (minGoal != null) {
LOG.info("Setting new timer at {}", Utils.formatTime(minGoal));
minGoalState.write(minGoal);
attributionTimer.set(minGoal);
} else {
minGoalState.clear();
}
}

This code block ties back to the three desired capabilities of a general state
API in very similar ways as the @ProcessElement method, with one
noteworthy difference:

Flexibility in write and read granularity

```
We were able to make a single, coarse-grained read up front to load all of
the data in the maps and set. This is typically much more efficient than
loading each field separately, or even worse loading each field element by
element. It also shows the importance of being able to traverse the
spectrum of access granularities, from fine-grained to coarse-grained.
```
And that’s it! We’ve implemented a basic conversion attribution pipeline, in a
way that’s efficient enough to be operated at respectable scales using a
reasonable amount of resources. And importantly, it functions properly in the
face of out-of-order data. If you look at the dataset used for the unit test in
Example 7-10, you can see it presents a number of challenges, even at this
small scale:

```
Tracking and attributing multiple distinct conversions across a
shared set of URLs.
Data arriving out of order, and in particular, goals arriving (in
processing time) before visits and impressions that lead to them, as
well as other goals which occurred earlier.
Source URLs that generate multiple distinct impressions to different
target URLs.
Physically distinct impressions (e.g., multiple clicks on the same
advertisement) that must be deduplicated to a single logical
```

```
impression.
```
_Example 7-10. Example dataset for validating conversion attribution logic_

private static TestStream<KV<String, VisitOrImpression>> createStream() {
// Impressions and visits, in event-time order, for two (logical)
attributable
// impressions and one unattributable impression.
Impression signupImpression = new Impression(
123L, "http://search.com?q=xyz",
"http://xyz.com/", Utils.parseTime("12:01:00"));
Visit signupVisit = new Visit(
"http://xyz.com/", Utils.parseTime("12:01:10"),
"http://search.com?q=xyz", false/*isGoal*/);
Visit signupGoal = new Visit(
"http://xyz.com/join-mailing-list", Utils.parseTime("12:01:30"),
"http://xyz.com/", true/*isGoal*/);

Impression shoppingImpression = new Impression(
456L, "http://search.com?q=thing",
"http://xyz.com/thing", Utils.parseTime("12:02:00"));
Impression shoppingImpressionDup = new Impression(
789L, "http://search.com?q=thing",
"http://xyz.com/thing", Utils.parseTime("12:02:10"));
Visit shoppingVisit1 = new Visit(
"http://xyz.com/thing", Utils.parseTime("12:02:30"),
"http://search.com?q=thing", false/*isGoal*/);
Visit shoppingVisit2 = new Visit(
"http://xyz.com/thing/add-to-cart", Utils.parseTime("12:03:00"),
"http://xyz.com/thing", false/*isGoal*/);
Visit shoppingVisit3 = new Visit(
"http://xyz.com/thing/purchase", Utils.parseTime("12:03:20"),
"http://xyz.com/thing/add-to-cart", false/*isGoal*/);
Visit shoppingGoal = new Visit(
"http://xyz.com/thing/receipt", Utils.parseTime("12:03:45"),
"http://xyz.com/thing/purchase", true/*isGoal*/);

Impression unattributedImpression = new Impression(
000L, "http://search.com?q=thing",
"http://xyz.com/other-thing", Utils.parseTime("12:04:00"));
Visit unattributedVisit = new Visit(
"http://xyz.com/other-thing", Utils.parseTime("12:04:20"),
"http://search.com?q=other thing", false/*isGoal*/);


// Create a stream of visits and impressions, with data arriving out
of order.
return TestStream.create(
KvCoder.of(StringUtf8Coder.of(),
AvroCoder.of(VisitOrImpression.class)))
.advanceWatermarkTo(Utils.parseTime("12:00:00"))
.addElements(visitOrImpression(shoppingVisit2, null))
.addElements(visitOrImpression(shoppingGoal, null))
.addElements(visitOrImpression(shoppingVisit3, null))
.addElements(visitOrImpression(signupGoal, null))
.advanceWatermarkTo(Utils.parseTime("12:00:30"))
.addElements(visitOrImpression(null, signupImpression))
.advanceWatermarkTo(Utils.parseTime("12:01:00"))
.addElements(visitOrImpression(null, shoppingImpression))
.addElements(visitOrImpression(signupVisit, null))
.advanceWatermarkTo(Utils.parseTime("12:01:30"))
.addElements(visitOrImpression(null, shoppingImpressionDup))
.addElements(visitOrImpression(shoppingVisit1, null))
.advanceWatermarkTo(Utils.parseTime("12:03:45"))
.addElements(visitOrImpression(null, unattributedImpression))
.advanceWatermarkTo(Utils.parseTime("12:04:00"))
.addElements(visitOrImpression(unattributedVisit, null))
.advanceWatermarkToInfinity();
}

And remember, we’re working here on a relatively constrained version of
conversion attribution. A full-blown impelementation would have additional
challenges to deal with (e.g., garbage collection, DAGs of visits instead of
trees). Regardless, this pipeline provides a nice contrast to the oftentimes
insufficiently flexible approaches provided by raw grouping an incremental
combination. By trading off some amount of implementation complexity, we
were able to find the necessary balance of efficiency, without compromising
on correctness. Additionally, this pipeline highlights the more imperative
approach towards stream processing that state and timers afford (think C or
Java), which is a nice complement to the more functional approach afforded
by windowing and triggers (think Haskell).
