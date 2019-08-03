---
title: Pipes and filters architectures with Python generators (and other iterators)
date: 2013-07-17 15:00:00 +200
author: Christoph
canonical_url: https://tech.stylight.com/pipes-and-filters-architectures-with-python-generators/
layout: page
---

## Motivation

Most developers have to write programs that read and process items of stuff. Be it products, feed-items, images, you name it. I’ve certainly had to. Apprentice programmers would just write it all down sequentially, but – as many fellow programmers can testify – this approach leads to maintenance problems due to a [mixing of responsibilities](https://en.wikipedia.org/wiki/Separation_of_concerns) in one place.

The [pipes and filters](https://en.wikipedia.org/wiki/Pipeline_(software)) design pattern is a natural fit to this problem. They allow you to separate out responsibilities in distinct processing units. While this may lead to an increase in perceived complexity, it actually reduces the complexity you have to handle by confining it to some other level.

One especially brilliant example of this type of architecture is UNIX’ pipe system.

Disclaimer: For this post I will show you the outline of a simple yet extensible feed (RSS or other) parsing system. The implementation of the concrete parts is left as an excercise to the reader. 🙂

## A naive solution with lots of boilerplate

We want to separate out processing steps (to stay maintainable) so we need to agree on an interface to pass information from one step to the next. We just start with simple class with some hook-methods.

{% highlight python %}
# WARNING, Anti-Example!
# Do not use. Better code will come up below!!1!
class ProcessingStep(object):
    def __init__(self):
        self.results = []

        def add(self, items):
            for item in items:
                # This is built that way so that self.process()
                # can either drop or add items to the results.
                # (self.process() has to return the item again
                # if it should be preserved, an empty list if
                # it should be dropped)
                for result in self.process(item):
                    self.results.append(result)

        def get(self):
            return self.results

        def process(self, item):
            raise NotImplementedError
{% endhighlight %}

Creating new steps is trivially easy now. We just have to subclass `ProcessingStep`, override `process` and we’re done.

{% highlight python %}
class IdentityStep(ProcessingStep):
    def process(self, item):
        return [item]

# We need some management class which will control our pipeline
# and feed the items through it.
class Pipeline(object):
    def __init__(self, steps):
        self.steps = steps

    def feed(self, items):
        # Feed the item into one step, get the result, feed the
        # result to the next step and so on.
        for step in steps:
            step.add(items)
            items = step.get()

        return items
{% endhighlight %}

Nice! This solution looks very clean and simple. It has a reasonable small line count and no magic. It still has problems though.

## Reality catches up

When implementing the feed parser we identified our basic steps: parsing the feed, processing each article, downloading images, storing the articles in our database.

Sometimes we want code to run only once in the process though, be it before the items get passed or afterwards. Examples are caching, database access, etc. So to achieve that we’ll have to change either the `ProcessingStep` or `Pipeline` class. We would add a `before_loop` and probably an `after_loop` hook method to every step.

Regardless, our next requirement would be: Buffer 500 entries per step to enable a grouped database flush. Grouping by some arbitrary key would also be nice, wouldn’t it? Although it’s actually quite simple to extend this solution, the base-class will just grow and grow with every feature. Testing it thorougly will also take some time. Another objection I gave myself was that these patterns look so common they ought to be solved already. It just doesn’t feel right to code at this level in Python.

This was the point where I decided to look at the [itertools](https://docs.python.org/3/library/itertools.html) module again. To my delight many of my desired patterns were already implemented, present in the recipe section of the docs, or could be coded up quickly without much boilerplate.

## More flexible solution with less boilerplate

Fortunately Python has a very nice language feature: [generators](https://docs.python.org/3/glossary.html#term-generator). Generators are like a swiss-army knife of iterables. You can combine them, you can wrap them, you can even send them values. And the nice part about them is: you can write them as if they were writing regular sequential Python code. And they are a perfect fit for writing pipeline processing software.

{% highlight python %}
def processing_step(items):
    """Example of a processing step."""

    # This section of code is run before the current step
    # starts processing.

    for item in items:
        # Your processing goes here.
        yield item

    # this section of code is run after all items have
    # been processed by this step.
{% endhighlight %}

If we look at this example we really see the power of a generator. Every processing step already has support for pre-run and post-run code with no additional boilerplate code. You just put the code in your generator.

Another nice thing about using generators is: we can use the `itertools` module to help us with commonly occuring iterator patterns – like grouping, buffering, etc.

{% highlight python %}
def chunk(chunk_size=32):
    """Group chunk_size elements into lists and emit them.

    >>> chunker = chunk(32)
    >>> gen = chunker(xrange(64))
    >>> assert len(list(gen)) == 2

    """
    def chunker(gen):
        gen = iter(gen)
        steps = range(chunk_size)
        chunk = []
        try:
            while True:
                for _ in steps:
                    chunk.append(gen.next())

                yield chunk
                chunk = []
        except StopIteration:
            if chunk:
                yield chunk

    return chunker

def unchunk(gen):
    """Flatten a sequence, but only one level deep."""
    return itertools.chain.from_iterable(gen)
{% endhighlight %}

A simple filter solution would look something like this.

{% highlight python %}
def filter_test_entries(gen):
    for entry in gen:
        if entry.title.startswith("test"):
            continue
        yield entry
{% endhighlight %}

## Putting the band back together

We now have seen some individual components of the whole pipeline system, but to actually use the system we need two more (very simple) functions.

The first one actually constructs our processing-pipeline. It’s implemented in a very simple way and this actually was my first implementation of it. I may, in later blog-posts, elaborate a bit about the subsequent evolution of this particular function.

{% highlight python %}
def combine_pipeline(source, pipeline):
    """Combine source and pipeline and return a generator.

    This is basically the same as the following function:

    >>> def combine_pipeline_explicit(source, pipeline):
    ...     gen = source
    ...     for step in pipeline:
    ...         gen = step(gen)
    ...     return gen

    """
    return reduce(lambda x, y: y(x), pipeline, source)
{% endhighlight %}

The second one we use to kickstart the combined pipeline. Please note that, since we use generators here, the pipeline can only be run once. To reset the pipeline, just call `combine_pipeline` again.



{% highlight python %}
def consume(iter):
    """Consume an iterator, trowing away the results.

    You may use this if you want to trigger some code in a
    generator but don’t want to actually know the result because
    everything happens as a "side-effect" of the generator running.

    There are better ones, but this one is simple to explain.
    Read the itertools recipe section for a more performant
    implementation of this.
    """
    for _ in iter:
        pass
{% endhighlight %}

Then you can use it like that:

{% highlight python %}
pipeline_steps = [
    parse_xml,
    download_images,
    chunk(32),
    # This one combines images into
    # chunks of 32 and passes these
    # on to `save_database`. This is
    # useful to reduce the accumulation
    # of latency-costs of the db-access.
    save_to_database
]

source = open("feed.xml", "r")
pipeline = combine_pipeline(source, pipeline_steps)
consume(pipeline)
# You may also iterate yourself over the pipeline, e.g.:
#
# for entry in pipline:
#     print(entry)
#
{% endhighlight %}

And that was it. Every entry coming out of pipeline will have gone through every processing step along the way. Any additional processing steps can be simply added/removed/replaced just by putting it in the `pipeline_steps` list. Neat.

## Conclusion

I’m very happy with this system and although it changed a bit and gathered many processing steps, the core of it is exactly as presented here. The pipeline length in our system is nearing 30 distinct steps now and it is thoroughly tested (~95% coverage). The longest processing step has around 70 [SLOC](https://en.wikipedia.org/wiki/Source_lines_of_code).

At first I was wary because I knew that generators can be detrimental to performance because of the function-call overhead inherent in every iteration, yet I found that the performance of the generators were negligable in comparison to the run-time of the steps themselves.

Aside: While writing this blog-post I found it very hard to (re-)create the initial (anti-)example in the first section. Knowing a solution to a problem seems to make you blind for other ones.
