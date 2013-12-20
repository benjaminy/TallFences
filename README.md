TallFences
==========

Tall Fences is a task parallel library in which the workers are memory isolated by default

> Tall Fences doesn't exist yet in any substantial sense (as of late
> 2013), but I'm writing down the idea in hope of getting to work on it
> some day.

Tall Fences will be a framework/library for adding parallelism to
applications.  The primary target audience is mainstream application
developers.  That is, not scientific and media programmers who have been
doing parallel programming for decades.

### Tall Fences in 100 words.

Tall Fences is very closely related to existing task parallel libraries
(TPL, PPL, GCD, TBB, Cilk, Java fork/join, etc.).  Tall Fences' novel
feature is that the workers are processes instead of threads.  The
primary advantage of processes is that the workers' memory is isolated
by default (windows of shared memory can be created when necessary).
Memory isolation makes avoidance of concurrency bugs much easier.  The
primary disadvantage of processes is that integrating parallelism into
an existing project requires more programmer effort.  For example, a
non-trivial amount of application-specific worker initialization and
finalization logic needs to be written.

## A bit more detail

### Background: Task Parallelism

Task parallel libraries have gained substantial mindshare in recent
years as parallel computers went mainstream.  The main reason for this
has to do with the granularity problem.  One of the features of any
parallel program is the granularity of its tasks.  That is, how long
does it take (typically) to run a single task.  If the tasks are too
coarse, there won't be enough of them to distribute to the parallel
processors (and therefore lost performance opportunity).  If the tasks
are too fine, the overhead of task creation and management will
overwhelm any performance benefit from parallel execution.

In order to use parallel computers efficiently, application designers
need to put in a fair bit of work defining tasks that hit the
granularity sweet spot.  This design work is hard enough, but
conventional low-level parallel programming frameworks force programmers
to do the additional work of managing the threads or processes that
these tasks run on.  It is common for programmers to mix together task
definition logic with low-level thread management, which leads to messy
hard-to-maintain code.

Task parallel libraries take the low-level worker management problem off
the application programmers' hands.  The application adds units of work
(tasks) to a collection that the library distributes to workers.  The
library takes almost all the responsibility for deciding how many
workers to create and which tasks to give to which workers.  The fancy
libraries even do work stealing when load imbalances crop up.

### Why processes?

The only significant innovation in Tall Fences, relative to existing
task parallel frameworks, is that it uses processes instead of threads
for workers.

### But what about performance?

Many programmers assume that a process-based parallel library will
inevitably be substantially lower performance that a thread-based
library, because processes have higher "overhead".  There are at least

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

This one actually is a legitimate potential problem.  Some algorithms
need shared data of some sort (at least logically, if not physically).

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
