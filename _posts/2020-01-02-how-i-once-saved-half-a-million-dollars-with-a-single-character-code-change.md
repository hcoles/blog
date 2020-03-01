---
layout: post
title:  "How I once saved half a million dollars with a single character code change"
author: henry
categories: [war stories, java]
image: assets/images/money.jpg
description: "Or four one liners every Java programmer should know"
featured: false
hidden: false
---

*Or four one liners every Java programmer should know*.

**I originally wrote this article in 2017 for the [NCR Edinburgh](https://ncredinburgh.com/) blog, it is reproduced here in edited form with their kind permission.** 

I have an old programmer war story to share, but first a bit of more recent
context.

When we published the first draft of ‘[Java for small
teams](http://javabook.ncredinburgh.com/)’ a commentator on Reddit took
exception to the section [‘Optimise for
Readability’](https://ncrcoe.gitbooks.io/java-for-small-
teams/content/style/10_optimise_for_readability.html).

This section contains the widely circulated advice that you should worry first
about whether your code is clear, correct and readable rather than whether it
performs well.

Not too controversial in 2016 and still not controversial in ~~2017~~ 2020.

Experience has taught us that micro-optimisation often has little or no
beneficial effect on performance, but does make code harder to understand and
maintain.

Real performance problems are often in places we don’t expect them to be. The
effective way to speed up an application is to profile it and remove the
bottlenecks you find. Most often the changes required to remove these
bottlenecks are nothing like the micro-optimizations we typically imagine in
advance.

So what was the objection? The redditor helpfully commented:

> “no no no”

and cited this [article by Randall
Hyde](http://ubiquity.acm.org/article.cfm?id=1513451) in the ACM.

I’d not seen the article before, but it makes some good points. It doesn’t
however argue that premature optimisation is something we should do.

Its main thrust is that “premature optimization is the root of all evil” has
been used as an excuse not to optimize at all, or to leave it as an item to be
addressed late in development. Like other things saved until the end it then
does not happen in any meaningful way.

I think this is entirely true. It’s a mistake I’ve made plenty of times. I’ve
worked on many projects where we didn’t profile or performance test until late
in the day. We often shipped slow, memory hungry code.

Sometimes it was easy to fix incrementally, sometimes it wasn’t. Concentrating
first on readability and performance second only works if we remember the
second bit.

While micro-optimizing each individual method is likely to result in un-
maintainable code and little performance benefit, not thinking about
performance at all can also end badly.

Over time I’ve become a little better at this. When I start up a new project I
make sure that as soon as we have a walking skeleton in place, we create some
sort of test that looks at what happens when the code is placed under load.

These tests serve a very different purpose than the performance tests that I
first encountered in my early career. Those tests tried to model realistic
loads and took place in realistic environments.

They tried to answer questions such as:

> “How much load will our production system be able to handle?”

Or

> “What hardware should we advise our clients to run our software on to deal
with their expected load?”

These remain important questions, but the tests I write don’t help answer
them.

They are still useful though. In the same way that a small unit tests answer
difference questions than a realistic end to end system test, small load tests
and micro benchmarks answer different questions to realistic performance
tests.

Small performance tests can give us early answers to questions like:

  * “Are there any obvious bugs that become visible under load?”
  * “Did that code change make things slower?”
  * “Did that code change actually make things faster?”
  * “Where are the bottlenecks in things I can easily change?”

There may well be larger problems and bottlenecks in the overall system and
the infrastructure to which it is deployed that these tests will not reveal,
but having some imperfect information early is better than having none at all.

Some care has to be taken as making these tests too small also has problems.  
Writing micro-benchmarks for every function is a great way to waste time while
achieving little of any value. The first tests I write now tend to be as big
as they can be while still being convenient to run against locally modified
code.

There is some great tooling out there that makes writing load tests easy. I’ve
only used it a handful of times, but I’m already a big fan of
[Gatling](https://github.com/gatling/gatling). For writing small micro-
benchmarks, [JMH](http://openjdk.java.net/projects/code-tools/jmh/) is the go
to tool (but beware the previous warning about making tests too small).

### My single character change

Another good point raised by the article is that sometimes micro-optimizations
do matter.

Not often, but sometimes.

Randall provides the bit of the Hoare quote that everyone forgets:

> “We should forget about small efficiencies, say about 97% of the time:
premature optimization is the root of all evil.”

Along with the full quote, we’re prone to forgetting that 3%.

Which brings me to the story from my past.

I was once part of a team that inherited a sprawling application originally
written in Java 1.3.

Our team was agile in the sense that we called our meetings stand-ups and
developers were empowered to change and improve things---as long as the
enterprise architects agreed to it and we spent a number of weeks
writing documents first.

The code was awful in ways that were entirely new to me but, despite the best
efforts of the architecture team, we managed to gradually improve things.

But there were still lots of problems, and some of them began to impact the
business plan.

The original developers of the application had not written any performance
tests and had never profiled it, but performance tests were occasionally run
by contractors using expensive proprietary tools and the results were not
good.

The main bottlenecks were in third party components that the vendors had
proven incapable of improving. In endless meetings it was argued that the
performance was “good enough”. And it had to be because we couldn't fix it.

And there were other problems.

The application required what was (for the time) a large heap of 1.5 gig.
Although the theoretical maximum size for a 32bit heap is 4 Gigabytes, in
practice the largest heap we could allocate on a Windows server (yes it ran on
Windows) was about 1.5 Gig.

So we were right up against a hard limit.

This was a huge problem, as we couldn't roll out new features that might
increase the heap size any further. After lots of meetings and discussion the
‘Enterprise Architects’ came up with a plan.

It involved some expensive re-architecturing.

It required some very expensive new servers.

It required a lot of even more expensive new licences for the third party
components.

It required delaying the features the business wanted until all this work was
done.

But it had to be done or business couldn't continue.

While the planning for this huge project was going on I fired up a profiler
and started having a play around.

As well as the profiler I was armed with an early version of the excellent
[Eclipse memory analyzer](http://www.eclipse.org/mat/).

After a bit of analysis I made a single character changed to the code base.

This reduced the peak memory consumption of the application by about 60%,
reduced GC pause times and probably also brought us slightly closer to world
peace, but I had no way to measure that.

What was the change?

I changed this:
    
```java
Set<Thing> aSet = new HashSet<Thing>();
```

To this:

```java 
Set<Thing> aSet = new HashSet<Thing>(0);
```

Most of these sets were empty for the entirety of their life, but each one was
consuming enough memory to hold the default number of entries.

Huge numbers of these empty sets were created and together they consumed over
half a gigabyte of memory. Over a third of our available heap.

My simple change set the initial capacity to 0, greatly reducing the overhead.

There will have been a small corresponding performance cost when something was
added to the sets as they would need to be re-sized, but this was dwarfed by
the reduction in GC cost.

### Good micro-optimizations

My tiny micro-optimisation had a small cost in terms of code readability
(“What’s that '0' doing there?”) but paid for this cost many many times over.

I added a comment to explain why the zero was needed, and some integration tests
that tried to measure the memory consumption of the custom data structures and
fail if they increased.

For a long time I was tempted to always add a 0 to the constructor of a Set I
thought was likely to be empty.

If you are also tempted, don’t.

Unless you know that the trick will make a different in that location, the
optimisation is premature. There is no benefit, but there is still a cost. And
it is a real cost — I had to spend time justifying the presence of that ‘0'
many times over the years as other developers questioned its purpose.

And if the optimisation was applied everywhere how would people know which
ones were important?

This particular trick also no longer works.

As I found out while writing this article, modern JDKs do not allocate any
space in a HashSet until something is added. If I had kept applying this trick
pre-emptively I would have been wasting my time.

So my story shows that micro optimizations can be useful, though usually the
results are much less dramatic.

Sometimes we can identify a key point where such as change will have large
effect, but perhaps there is a gain to be had from the cumulative effect of
lots of small optimizations. Each one on its own is irrelevant, but the net
effect might be measurable.

Or it might not.

So when should we do this?

We should do it in two situations

  1. When we know we are trading off code readability for a meaningful gain (thanks to profiling and testing)
  2. When we are not making a trade off at all

Lets talk about that 2nd one.

Even if we haven’t profiled and do not know the size of the gain, if the
(presumed) optimized version of the code is equally as readable as the non
optimized version it makes sense to use it.

Perhaps we will get no gain from it at all.

Perhaps it actually performs worse than other alternatives we considered.

But if we are not compromising any other factor by using it then it makes
sense to use our best guess at the most efficient implementation.

Performance testing and profiling should highlight if we were wrong in a way
that matters.

Considering performance last means that we **do consider it** and give it
weight **when all other things are equal**.

There is an also a much more compelling set of micro-optimisations. These are
optimizations that are both (possibly) slightly more efficient, and also
slightly cleaner compared to the code they commonly replace.

These one liners are usually utility methods in the Java libraries. Once you
know they’re there you’ll find that code that doesn’t use them looks… Well,
slightly wrong.

It is highly unlikely that the use of these one liners in any one place will
have a noticeable impact on performance, but if they are used consistently
throughout the code base they might have an overall positive effect. They
generally are slightly more memory efficient than the code they replace, and
might therefore improve garbage collection.

Or they might not, but it doesn’t matter as they also make the code slightly
cleaner.

That is reason enough to use them by itself.

We don’t really need to talk about performance at all when discussing these
one liners. They are just better ways of expressing something that is commonly
expressed more clumsily.

### Four one liners every Java programmer should know

#### java.util.Arrays.asList

For when you need to convert an array to a fixed size list.
    
    
    List<String> ss = Arrays.asList(“1”, “2”);

This is cleaner and more efficient than the commonly seen alternatives such
as:    
    
    List<String> ss = new ArrayList<>();  
    ss.add(“1”);  
    ss.sdd(“2”);

The list implementation returned by asList is specialized and may consume
slightly less memory than an ArrayList.

Be aware that the lists created by this method are non-modifiable and will
throw a `java.lang.UnsupportedOperation` if you try to modify them.

([Devin McIntyr](https://medium.com/@dantaro?source=responses---------
0-----------)e has pointed out that the returned list is not fully runtime
immutable. Although add and remove will throw, the set method does modify the
collection).

Java 9 now provides an alternative to asList.

    List.of("1", "2");

When the method is fully qualified it expresses intent more clearly than Arrays.asList, and it is likely to be infinitesimally more efficient as there are multiple overloads for different numbers of parameters.

#### java.util.Collections.empty*

Similar to `Arrays.asList` the `java.util.Collections` class contains methods
that create unmodifiable versions of the common collections classes
specialized to contain either no entries or single entries.

The most common ones are:

  * emptyMap
  * emptyList
  * emptySet
  * singletonList
  * singletonMap
  * singletonSet

But there are other more specialized methods there as well.

Again, using these results in cleaner more descriptive code compared to the  
common alternatives and the specialized data structures returned may have a
slightly smaller footprint.

#### valueOf

The form:
        
    Boolean b = Boolean.valueOf(true);

Is slightly more efficient than the subjectively uglier:
       
    Boolean b = new Boolean(true);

`valueOf` will return one of two fixed Boolean instances while calling the
constructor will always allocate a new object.

Similarly for Integers `valueOf` will return a shared instance for values
between -128 and 127.

`Long.valueOf` will similarly return objects from a cache.

The Oracle JDK does not currently look to use a cache for Floats or Doubles
when calling `valueOf`. So using `valueOf` in preference to new for floating
points might not have an advantage, but it has no disadvantage either.

Overloaded versions of the `valueOf` methods construct their desired types
from Strings. If you need to create a boxed primitive from a String, this is
the way to do it.

It is worth noting that, although it is not guaranteed by the spec, in
practice Java compilers implement autoboxing by calling `valueOf`.

So: 
    
    Boolean b = true;

and
 
    Boolean b = Boolean.valueOf(true);

Are equivalent.

(Steve McLeod has pointed out that as of Java 9 the
constructor is deprecated — so all the more reason to use valueOf.)

#### parseXXX

Similar to `valueOf` the parseXXX methods convert a string to a primitive,
only in this case the primitive is not boxed.    
    
    float f = Float.parseFloat(“1.2”);

This does exactly what the method says, and compared to the commonly seen
alternative:
        
    float f = new Float(“1.2”);

It avoids the construction and unboxing of a Float object.

Similar methods exist for Boolean, Double, Integer and the other primitive
types.

#### isEmpty

A common clumsy idiom is to check if a Collection is empty by checking its
size:
        
    if ( list.size() == 0 ) {  
      // do stuff  
    }

The clearer and more concise:

    
    
    if ( list.isEmpty() ) {  
      // do stuff  
    }

May also be more computationally efficient, depending on the data structure it
is being called on.

Often there is no performance difference, but for a large
`ConcurrentLinkedQueue` the difference can be significant.

#### What else is out there?

And that concludes the short list. I’d imagine that all of them were familiar
to a programmer that had been working with Java for over a year, but if you
are just starting out you wouldn’t know to look for them unless someone told
you they were there.

There are undoubtedly other similar useful functions scattered about the JDK
that provide the happy combination of cleanliness and infinitesimally small
performance gain over the clumsier code they replace.

If you know of any please share.

