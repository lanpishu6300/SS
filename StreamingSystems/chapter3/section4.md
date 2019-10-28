 ## Percentile Watermarks

So far, we have concerned ourselves with watermarks as measured by the
minimum event time of active messages in a stage. Tracking the minimum
allows the system to know when all earlier timestamps have been accounted
for. On the other hand, we could consider the entire distribution of event
timestamps for active messages and make use of it to create finer-grained
triggering conditions.

Instead of considering the minimum point of the distribution, we could take
any percentile of the distribution and say that we are guaranteed to have
processed this percentage of all events with earlier timestamps.

What is the advantage of this scheme? If for the business logic “mostly”
correct is sufficient, percentile watermarks provide a mechanism by which

```
5
```

the watermark can advance more quickly and more smoothly than if we were
tracking the minimum event time by discarding outliers in the long tail of the
distribution from the watermark. Figure 3-9 shows a compact distribution of
event times where the 90 percentile watermark is close to the 100
percentile. Figure 3-10 demonstrates a case where the outlier is further
behind, so the 90 percentile watermark is significantly ahead of the 100
percentile. By discarding the outlier data from the watermark, the percentile
watermark can still keep track of the bulk of the distribution without being
delayed by the outliers.

```
Figure 3-9. Normal-looking watermark histogram
```
```
Figure 3-10. Watermark histogram with outliers
```
```
th th
```
```
th th
```

Figure 3-11 shows an example of percentile watermarks used to draw
window boundaries for two-minute fixed windows. We can draw early
boundaries based on the percentile of timestamps of arrived data as tracked
by the percentile watermark.

```
Figure 3-11. Effects of varying watermark percentiles. As the percentile increases, more
events are included in the window: however, the processing time delay to materialize the
window also increases.
```
Figure 3-11 shows the 33 percentile, 66 percentile, and 100 percentile
(full) watermark, tracking the respective timestamp percentiles in the data
distribution. As expected, these allow boundaries to be drawn earlier than
tracking the full 100 percentile watermark. Notice that the 33 and 66
percentile watermarks each allow earlier triggering of windows but with the
trade-off of marking more data as late. For example, for the first window,
[12:00, 12:02), a window closed based on the 33 percentile watermark
would include only four events and materialize the result at 12:06 processing
time. If we use the 66 percentile watermark, the same event-time window
would include seven events, and materialize at 12:07 processing time. Using
the 100 percentile watermark includes all ten events and delays
materializing the results until 12:08 processing time. Thus, percentile
watermarks provide a way to tune the trade-off between latency of
materializing results and precision of the results.


