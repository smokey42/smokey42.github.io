---
title: "What happened during paipa development?"
author: Christoph
date: 2019-07-15 13:29:04 +0200
categories: python paipa history
layout: page
---
In this blog post I describe what problems **paipa** solves and what I learned during it's development.

## What is paipa?

**paipa** is a Python library which helps you write **[threaded pipelines](https://github.com/smokey42/python-paipa/blob/master/doc/introduction.rst#how-it-works-in-a-nutshell)** and **[generator pipelines](https://github.com/smokey42/python-paipa/blob/master/doc/coroutine.rst#what-is-it)**. These help the developer **stream large data-sets** and therefore **keep memory usage low**. It's therefore very suitable for batch processing.

Initially **paipa** was 2 libraries (threaded and generators) which were developed concurrently and gathered adapters to/from each other over the years. In the end I put them both into this package and released it.

### Coroutine / generator pipeline

One part of it is the so-called *coroutine pipeline* which composes a list of *Python generators* to allow the programmer to write simple loops which feed into one another. The upside of this approach is that Python's standard library **itertools** can be used to help with the data-flow. Also, the passing on of data from one step to the next is comparatively cheap as it's using the Python builtin statement *yield*. The downside is that it can only run on one CPU by design. This isn't a problem though if the program can be run multiple times concurrently with different work batches.

### Thread pipeline

The main idea of the other part of this library, the *threaded pipeline* is to allow programmers to write sequentially looking code while still having it run parallel to reduce the wall clock time of IO heavy tasks (like downloading stuff from the net). I tried to use the built in libraries in Python (futures, etc.) but these either suffered from deadlocks in some situations when used in a CSP context or were overkill to setup. So in order to minimize developer confusion I wrote this library taking inspiration from Go, Erlang, etc.

## Why threads and not asyncio?

During early development the main deployment target of **paipa** was Python 2.7, so **asyncio** wasn't very usable. You'd either had to choose **Tornado** or **Twisted**, neither of which was particularily usable in a batch context. Also during that time it was way easier to teach threading to programmers. A thread can be pretty invisible to the programmer and gives off control really nicely in Python when doing IO. If you also incentivize **no shared state** or provide some safe shared state, then the dangers of threading can be easily navigated around. Spawning thousands of threads is also relatively cheap. All in all it was the right tool for the job. Initial reluctance of fellow programmers (quote: "uh, threads!") were only short lived and the library gained wide adoption internally.

### Why not $batchFramework?

Why not **Celery**, **Hadoop** or something else?

 1. This library doesn't need any infrastructure to run. If you can run Python code you can run this library. It's also simple enough to be used by new developers on short notice.
 2. If you already use **Celery** (or **rq**) you can just use it in a *Task* and parallelize your IO without having to spawn a task for each IO operation.
 3. It has low overhead. We were able to saturate the network with concurrent downloads and uploads in **one Celery task**.

## Thread part: Challenges and lessons learned

One challenge I faced when developing it were constant lockups of my program due to some thread never getting their shutdown signal due to crashes or bugs in other parts of the code thus preventing the interpreter from being terminated. I then had to *kill -9* my interpreter and fix the bug. This happened only in the early phase and was never a problem since.

### Memory overflow

Another problem was related to processing huge amounts of data (infinite sequences, etc). The queues were pulling the data very fast and were filling all the available memory on my dev machine in no time. Linux' OOM killer then started to kill random processes on my machine and I had to reboot. I solved this problem by introducing a configurable *per step threshold* of how many entries should be in the output queue of a given step. If the threshold was reached, this particular step slept for 10 milliseconds (quite long, but we never saw any throughput problems due to this) and tried again. This implicitly lead to having a sort of *back pressure* up the pipeline. See [the relevant code section](https://github.com/smokey42/python-paipa/blob/master/paipa/threaded/steps.py#L203) for reference.

### Shutting down the system

The last major problem was getting the shutdown of all threads right, because these had to follow the correct order or some threads would never get the message. In essence the algorithm goes like this: send the shutdown message in, the thread which got the message will stop accepting new messages and send the message to it's siblings-threads of the same step until all of them have shut down. Once all siblings of said thread shut down, the next step is notified for shutdown. This leads to the orderly shutdown of all threads from the top to the bottom. See [the relevant code section](https://github.com/smokey42/python-paipa/blob/master/paipa/threaded/steps.py#L178) for reference.

After solving these problems it was used in production (image downloading and processing subsystem) and from then on only minor changes were made to the API and internal data structures. It gained a [step-local storage](https://github.com/smokey42/python-paipa/blob/master/paipa/threaded/pipeline.py#L289) where steps could share data, some more helper methods in the wrapping *Pipeline* class, a [context manager interfance](https://github.com/stylight/python-paipa/blob/master/doc/ingestion.rst#streaming-and-running-the-pipeline-in-the-background), some more ingestion methods and more documentation.

## Coroutine part: Challenges and lessons learned

This part didn't pose too many thorny challenges, only the [runtime-debugger](https://github.com/smokey42/python-paipa/blob/master/paipa/debugger.py#L90) (see an [example of it in action](https://github.com/stylight/python-paipa/blob/master/doc/coroutine.rst#debugging-pipelines) in the docs) which I added later in development was a bit of a mind-bender. To reduce the run-time overhead of the debugger I used the **Cython** project to compile it to C.

The real challenges of this library were related to how the library could be used. Due to many steps sharing the same things (db connections, etc.) I resorted to use factories with the necessary settings in them all over the place. This yielded a not very aesthetic pipeline definition. This couldn't be helped because the only other way to get the information into the steps would be to send it in. In this case the sent-through entries wouldn't be able to stand on their own anymore and the usage of **itertools.groupby** or other helpers would be prevented. In the end I accepted this trade-off and left it as it is.

Anyway, this library is very nice to work with for ETL scripts and can be adapted to file-like objects as well (not part of this lib). So a pipeline could even have a file-like stream as an input, all the data gets streamed through the generators and in the end be streamed into a new file-like object. This limits peak memory consumption dramatically and enabled massive parallelization and maxing out of all of the available resources of the batch machines.

## How to try it out

Install it in your favourite virtual environment by issuing:

{% highlight bash %}
pip install paipa
{% endhighlight %}

Afterwards it can be imported like this:

{% highlight python %}
import paipa
{% endhighlight %}

It's compatible with Python 2 and Python 3.

This [introduction page](https://github.com/smokey42/python-paipa/blob/master/doc/introduction.rst#how-it-works-in-a-nutshell) has enough documentation to help you get a little thread-pipeline running. 
## Future outlook

Now with the awesome new features of Python 3.5 and onwards (*asyncio*, *async*, *await*, etc...), I'd like to revisit some aspects of the library to make the interface of the coroutine pipelines easier and potentially also offer some **asyncio** adapters. Also, given that Python 2.7 is soon to be deprecated for good, I'd like to remove Python 2 support for future revisions.
