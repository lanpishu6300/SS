 rt II. Streams and Tables


# Chapter 6. Streams and Tables

You have reached the part of the book where we talk about streams and
tables. If you recall, back in Chapter 1, we briefly discussed two important
but orthogonal dimensions of data: _cardinality_ and _constitution_. Until now,
we’ve focused strictly on the cardinality aspects (bounded versus unbounded)
and otherwise ignored the constitution aspects (stream versus table). This has
allowed us to learn about the challenges brought to the table by the
introduction of unbounded datasets, without worrying too much about the
lower-level details that really drive the way things work. We’re now going to
expand our horizons and see what the added dimension of constitution brings
to the mix.

Though it’s a bit of a stretch, one way to think about this shift in approach is
to compare the relationship of classical mechanics to quantum mechanics.
You know how in physics class they teach you a bunch of classical
mechanics stuff like Newtonian theory and so on, and then after you think
you’ve more or less mastered that, they come along and tell you it was all
bunk, and classical physics gives you only part of the picture, and there’s
actually this other thing called quantum mechanics that really explains how
things work at a lower level, but it didn’t make sense to complicate matters
up front by trying to teach you both at once, and...oh wait...we also haven’t
fully reconciled everything between the two yet, so just squint at it and trust
us that it all makes sense somehow? Well this is a lot like that, except your
brain will hurt less because physics is way harder than data processing, and
you won’t have to squint at anything and pretend it makes sense because it
actually does come together beautifully in the end, which is really cool.

So, with the stage appropriately set, the point of this chapter is twofold:

```
To try to describe the relationship between the Beam Model (as
we’ve described it in the book up to this point) and the theory of
“streams and tables” (as popularized by Martin Kleppmann and Jay
```

```
Kreps, among others, but essentially originating out of the database
world). It turns out that stream and table theory does an illuminating
job of describing the low-level concepts that underlie the Beam
Model. Additionally, a clear understanding of how they relate is
particularly informative when considering how robust stream
processing concepts might be cleanly integrated into SQL
(something we consider in Chapter 8).
To bombard you with bad physics analogies for the sheer fun of it.
Writing a book is a lot of work; you have to find little joys here and
there to keep you going.

