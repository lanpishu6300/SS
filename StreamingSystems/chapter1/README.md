
# Chapter 1. Streaming 101
Streaming data processing is a big deal in big data these days, and for good
reasons; among them are the following:

```
Businesses crave ever-more timely insights into their data, and
switching to streaming is a good way to achieve lower latency
```
```
The massive, unbounded datasets that are increasingly common in
modern business are more easily tamed using a system designed for
such never-ending volumes of data.
```
```
Processing data as they arrive spreads workloads out more evenly
over time, yielding more consistent and predictable consumption of
resources.
```
Despite this business-driven surge of interest in streaming, streaming systems
long remained relatively immature compared to their batch brethren. It’s only
recently that the tide has swung conclusively in the other direction. In my
more bumptious moments, I hope that might be in small part due to the solid
dose of goading I originally served up in my “Streaming 101” and
“Streaming 102” blog posts (on which the first few chapters of this book are
rather obviously based). But in reality, there’s also just a lot of industry
interest in seeing streaming systems mature and a lot of smart and active
folks out there who enjoy building them.

Even though the battle for general streaming advocacy has been, in my
opinion, effectively won, I’m still going to present my original arguments
from “Streaming 101” more or less unaltered. For one, they’re still very
applicable today, even if much of industry has begun to heed the battle cry.
And for two, there are a lot of folks out there who still haven’t gotten the
memo; this book is an extended attempt at getting these points across.

To begin, I cover some important background information that will help
frame the rest of the topics I want to discuss. I do this in three specific


sections:

Terminology

```
To talk precisely about complex topics requires precise definitions of
terms. For some terms that have overloaded interpretations in current use,
I’ll try to nail down exactly what I mean when I say them.
```
Capabilities

```
I remark on the oft-perceived shortcomings of streaming systems. I also
propose the frame of mind that I believe data processing system builders
need to adopt in order to address the needs of modern data consumers
going forward.
```
Time domains

```
I introduce the two primary domains of time that are relevant in data
processing, show how they relate, and point out some of the difficulties
these two domains impose.
