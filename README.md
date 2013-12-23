TallFences
==========

Tall Fences is a task parallel library in which the workers are memory isolated by default.

> Tall Fences doesn't exist yet in any substantial sense (as of late
> 2013), but I'm writing down the idea in hope of getting to work on it
> some day.

Tall Fences will be a framework/library for adding parallelism to
applications.  The primary target audience is mainstream application
developers.  That is, not scientific and media programmers who have been
doing parallel programming for decades.

### Tall Fences in 100 words.

Tall Fences is very closely related to existing task parallel libraries
([TPL](http://msdn.microsoft.com/en-us/library/dd460717.aspx),
[PPL](http://msdn.microsoft.com/en-us/library/dd492418.aspx),
[GCD](http://en.wikipedia.org/wiki/Grand_Central_Dispatch),
[TBB](https://www.threadingbuildingblocks.org/),
[Cilk](http://software.intel.com/en-us/intel-cilk-plus), [Java
fork/join](http://coopsoft.com/ar/CalamityArticle.html), etc.).  Tall
Fences' novel feature is that the workers are processes instead of
threads.  The primary advantage of processes is that the workers' memory
is isolated by default (windows of shared memory can be created when
necessary).  Memory isolation makes avoidance of concurrency bugs much
easier.  The primary disadvantage of processes is that integrating
parallelism into an existing project requires more programmer effort.
For example, a non-trivial amount of application-specific worker
initialization and finalization logic needs to be written.

## A bit more detail

### Background: Task Parallelism

Task parallel libraries have gained substantial mindshare in recent
years as parallel computers went mainstream.  The main reason for this
has to do with the granularity problem.  One of the features of any
parallel program is the granularity of its tasks.  That is, how long
does it take to run a single ("average") task.  If the tasks are too
coarse, there won't be enough of them to distribute to the parallel
processors (and therefore lost performance opportunity).  If the tasks
are too fine, the overhead of task creation and management will
overwhelm any performance benefit from parallel execution.

In order to use parallel computers efficiently, application designers
need to put in a fair bit of work defining tasks whose granularity is in
the happy medium range.  In general developers should try to make their
tasks as fine-grained as they can without running afoul of task
management overhead problems.  More fine-grained tasks make it easier to
share work efficiently on a wide range of platforms.  The old school way
to do parallelism is to create a thread or process for each task, but
thread creation is relatively expenssive, which puts an unpleasant lower
bound on the granularity of tasks.

Worker pools are the solution to the granularity connundrum: create a
small number of threads or processes and then dispatch tasks to these
workers as needed.  Task dispatching is much cheaper than thread
creation, so using a worker pool makes it effeicient to create smaller
units of work.

Task parallel libraries create worker pools and manage the dispatching
of tasks to workers.  The application adds units of work (tasks) to a
collection that the library distributes to workers.  The library takes
almost all the responsibility for deciding how many workers to create
and which tasks to give to which workers.  The fancy libraries even do
work stealing when load imbalances crop up.

### Why processes?

The novel contribution in Tall Fences, relative to existing task
parallel frameworks, is that its workers are processes instead of
threads.  The primary reason for this is

#### Anecdotes

Interestingly, the area of threads versus processes for task parallel
libraries is one where many application developers seem to be ahead of
the research community.  Here are a few anecdotes from different
application domains where the developers chose to make their own
home-brew system for memory-isolated parallelism.

##### Financial trading

There is a tech company in the financial sector that somewhat famously
does most of its programming in OCaml.  At a technical presentation, one
of the developers was asked if it was problematic that the OCaml runtime
system is mostly single-threaded.  (The situation with multithreading in
OCaml is similar to Python with the global interpreter lock.)  The
developer responded that it really wasn't an issue for them, because
they do all their parallelism at the process level anyway, with a
separate instance of the OCaml runtime for each worker.

This is particularly

##### Databases

Michael Stonebraker.  VoltDB.

##### Static Analysis

The founder of the Tall Fences project once worked at a software
engineering tools company.  The company's flagship product is a static
analysis bug finding tool for C/C++.  Static analysis is extremely
computationally expensive, so during your humble author's tenure the
company decided to parallelize its algorithms.  The tool had been under
development for over a decade and was implemented in C that had been
heavily performance optimized over the years.  Parallelizing this code
base with threads would have almost certainly lead to a never ended
stream of subtle and nasty concurrency bugs.

The company decided to create a home-brew task parallel library that
uses processes.  Even with this architecture (which I absolutely think
is the right one), the parallelization effort what still a multiple
person-year affair.

### But what about performance?

Many programmers assume that a process-based parallel library will
inevitably be substantially lower performance that a thread-based
library, because processes have higher "overhead".  There are at least

Using processes actually has some performance benefits too!

#### Creation time

Creating processes is generally substantially more expensive than
creating threads.  The particular numbers vary quite a lot between
platforms, but there is certainly a big gap.  However, this overhead is
totally irrelevant for a well tuned task parallel library, because
worker creation (and finalization) should be very infrequent.  A few
milliseconds of overhead once every few minutes is not even worth
thinking about.

#### Memory overhead

Processes do have substantially more state associated with them than
threads.  However, the number of workers should be roughly equal to the
number of processors.  In general this should be a small number relative
to the amount of memory in the machine, so it's not a big deal.

#### Context switching time

Context switching between processes is more expensive than context
switching between threads.  However, if the library (and application)
are behaving well there should be very little context switching.  Each
worker should just stay in place on a processor for a while and run
through a bunch of tasks.  The cost of starting a new task should be
much lower than context switching (and isn't a point of difference
between threads and processes for workers anyway).

#### Application data management

There are some real issues related to application data.  With memory
isolated workers, the most natural organization of data is for each
worker to have its own copy of whatever data the application uses.
There are two distinct issues here: program logic and memory
performance.

In some cases it simply does not make sense to duplicate a piece of
data.  For example, consider a game with a data structure that
represents the state of the virtual world.  It doesn't make sense for
there to be multiple copies of the world state for a single game
instance.

For this kind of logically shared data, developers have to make a
choice.  They can create a shared memory buffer to store the data.  This
is simple to arrange, but forces the application to deal with all the
nasty shared-memory parallel programming issues that this whole project
is an attempt to avoid.  Alternatively, they can create a client/server
architecture, where workers can access the shared data through some API.
In the extreme this line of thinking ends up with keeping the shared
data in a database.

For data that can be duplicated without breaking the application there
is another issue to consider, which is the size of the data.  If it is
large, creating multiple copies might adversely affect the performance
of the application.  Fortunately, most applications have a kind of
long-tail distribution in their memory profile.  A few data structures
are responsible for the majority of the memory consumption and lots of
little ones make up the rest.  So an application can make a single
shared copy of the large structures and duplicate the rest.

#### False sharing

One of the really nasty performance gremlins in parallel programming is
false sharing.  False sharing occurs when different processors access
_distinct_ memory locations that happen to be close to each other.  If
by bad luck or bad design the locations end up in the same cache line,
then the processors will fight over which one "owns" the cache line even
though they are not accessing the same address.  False sharing can cause
significant performance problems and it is a pain to track down and fix
because the programmer needs to make source code changes that will
convince the compiler to change where data is placed in memory.

Using processes neatly avoids the vast majority of false sharing
problems.  For worker-local memory, false sharing is impossible.  For
shared memory false sharing is possible, but shouldn't happen much.

#### Unnecessary sharing

Related to false sharing, there is the problem of unnecessary sharing.
The classic example is random number generation.  If multiple workers
access the same RNG state, it can become a bottleneck.  The normal
solution is to give each worker its own RNG state.  This happens by
default with memory isolated workers.

## Implementation ideas

### Steal API

The thread-based task parallel libraries have been refining their APIs
for years.  Let's steal as much of that work as possible.

There will be a bit of a difference around initialization and shutdown.

Also we'll need to handle unexpected termination of workers reasonably
gracefully.

Also there will need to be an API for creating and using shared memory
windows.

### Fancy queue implementation

Adding and dispatching tasks should be insanely fast.  Applications
should be able to create relatively fine-grained tasks without worrying
about overhead too much.  My guess is that the task queue will need to
be implemented in shared memory with non-blocking algorithms.  Of course
applications won't have to know about this craziness at all.

### Worker pool tuning

The number of workers that should exist is a surprisingly subtle issue.
A lot of applications and libraries totally punt and just make as many
workers as there are processors.  However, there are plenty of
situations where this is not the best choice:

* There are multiple applications sharing the machine's parallel resources.
* Each worker needs to allocate a lot of memory
* Some blocking nonsense
* Contention

#### Processor sharing between applications

I think this one is the dirty little secret that multi-core cheerleaders
have been deliberately ignoring.  If multi-cores "win" and lots of
applications are adapted to use lots of TLP, how should they share the
processors?

#### Memory crap

If every worker needs lots of memory, the number of workers that you
want to create might be limited by the amount of memory in the machine.

#### Blocking

What if some workers are blocked for chunks of time?  We might actually
want more workers than processors.

#### Contention

What if the application is limited by something else.  (I guess this is
just a generalization of the memory thing.)

#### What to do about it

In general I think it will be extremely hard for a library to pick the
right number of workers all the time.  I think we need either an API for
application programmer guidance, or dynamic adaptation or both.
