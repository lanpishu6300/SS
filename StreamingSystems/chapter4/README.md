 # Chapter 4. Advanced Windowing

Hello again! I hope you enjoyed Chapter 3 as much as I did. Watermarks are
a fascinating topic, and Slava knows them better than anyone on the planet.
Now that we have a deeper understanding of watermarks under our belts, I’d
like to dive into some more advanced topics related to the _what_ , _where_ , _when_ ,
and _how_ questions.

We first look at _processing-time windowing_ , which is an interesting mix of
both where and when, to understand better how it relates to event-time
windowing and get a sense for times when it’s actually the right approach to
take. We then dive into some more advanced event-time windowing
concepts, looking at _session windows_ in detail, and finally making a case for
why generalized _custom windowing_ is a useful (and surprisingly
straightforward) concept by exploring three different types of custom
windows: _unaligned_ fixed windows, _per-key_ fixed windows, and _bounded_
sessions windows.
