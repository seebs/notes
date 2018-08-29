Presenter: [Eben Freeman](https://www.gophercon.com/agenda/speakers/279043)

Liveblogger: [Seebs](https://www.seebs.net/)


Allocator Wrestling -- discussions on Go memory allocation and garbage collection

## Summary

This talk focuses on the Go allocator and garbage collector, mostly from a performance standpoint, but also just because they're really interesting. Takeaway message: It is possible to find out whether you can improve performance by changing your interactions with the allocator, and there are approachable ways to improve performance that are likely to be applicable for most applications.


## Intro

I had to go to this talk because it combines my favorite things, performance tuning and puns.

People are still filtering in. This is three ballrooms with their dividers removed, and it's pretty full, I don't think there's as many chairs as there are people. This is why a good title for a talk is important.

According to the people behind me, the speakers are playing _Despacito_. I think we got memed on by the conference staff.

A person who is not the speaker is telling us about the rules. We are expected to do a standing ovation at the end of the talk. Also we are informed that the sponsors have good swag, and raffle tickets.

Eben Freeman is a "mostly reformed" mathematician, who works at honeycomb.io. (Standing ovation, as was foretold in the prophesy.)

He's asked us to split up and go to different ends of the room, sorted by whether we're interested in Go2 or Go++. I think he is messing with us.

Slides start. [Slide deck link.](https://speakerdeck.com/emfree/allocator-wrestling) I'm not going to include bad phone pictures of the slide deck.

Memory allocation is mostly handled for you in Go, but that doesn't mean that it's free. Conventional wisdom is that it has costs, but does it? Answer: Yes. Very impressive 2x-3x speedups on some benchmarks from nothing but memory optimizations.

Memory allocation can be opaque; you have to have tools and understanding to make better choices about where to try to optimize memory performance. Also, it's just plain interesting; that's another good reason to understand it.

Overview slide: Three topics. Understanding the allocator and GC; tools for understanding it, and strategies for improving efficiency/performance.

This whole talk is from a practitioner's perspective: Authors of Go are probably smart, so a certain amount of confidence in their design is reasonable.

## SECTION 1: Understanding how the allocator works.

Go memory layout distinguishes between stacks (many, transient, "freed" automatically when functions return) and heap (things that need to be explicitly freed later). Things that might outlast a function have to be put in the heap. Stack holds stack frames in order; heap could have dragons for all we know.

### Understanding the Allocator

Allocator has three goals:

* Efficient for a given size, but avoid fragmentation.
* Avoid locking in the common case.
* Be able to reclaim freeable memory efficiently.

For efficiency and avoiding fragmentation: gather like-sized objects in blocks. For locking, use local cache. For reclaiming memory efficiently, use bitmaps and run concurrently.

Top-level organization: Heap is mapped into arenas (64MB hunks on amd64) and spans. Each arena is subdivided into "spans", and all the spans in an arena are the same size. That means that you can figure out which arena you're in, and which span you're in, with simple arithmetic:

```Go
heapArena := mheap_.arenas[ptr / arenaSize]
span := heapArena.spans[(ptr % arenaSize) / pageSize]
```

Smaller objects (<=32KB) are grouped into spans. So 64-byte objects live together, and 96-byte objects live together, and so on. Spans can vary in size; there are ~70 size classes, ranging from 8KB to 64KB.

In general, each scheduler context (usually one per CPU core) will have its own span for each size class; this means that it can grab things from that span without locking, no other scheduler context will touch that span. The allocator also handles marking which parts of the heap contain pointers.

So in the common case, there's no locking; it's only when there's no available objects in that span that it has to request a new span.

### Garbage collection

With Oscar the Grouch picture.

If a pointer escapes a function, so it needs to be on the heap, the garbage collector has to be able to free it when no references to it are left. Go's strategy is "tricolor concurrent mark/sweep". Mark phase: Find reachable objects. Sweep phase: Free unreachable objects.

Only colors are white, grey, and black, because allocator authors live in a "dour" world without any other colors. (See, there *is* more to talks than just reading the slides.) Objects start white, get marked grey when found, and when all the things an object refers to have been marked grey, the object gets marked black. When you're done marking everything, white objects can be freed; nothing referred to them. (The "free" phase is also called "sweep". You sweep everything not marked for preservation.)

So, how does this work? How does it know which parts are referents, and track the markings?

Example: A slice has a pointer, length, and capacity. So if you have a struct containing a slice, it's got one pointer. The arena has a bitmap that describes each block; two bits per pointer-sized chunk. One bit indicates that at least one following chunk for this object will contain a pointer, one bit indicates that this chunk itself is pointer. The allocator can use these to search the heap for possible-pointers very quickly.

Similarly, mark state is kept in a bitmap. Neat trick: When you're done marking everything that's referenced, the table of referenced things is precisely equal to the table of things that need to be kept allocated. Anything not-marked in GC phase can be marked as free, which means that you can just assign the bits over (possibly inverting them).

GC is supposed to be concurrent, but what if the application does things and stores, on its own stack a pointer to a white object? The GC doesn't know that the pointer is in use yet. So compiler turns writes to pointers into potential write barrier calls, which will color target objects of any object being written. There's a little more magic than that; this is just an overview.

The write barrier slows things down when it's on. Marking also just plain costs time. Usually, 25% of GOMAXPROCS is doing background marking, but a goroutine which is doing a ton of allocation can outrun that. So, goroutines have a budget; they're "charged" for allocations, based on size and also on the heap density of the heap. If the goroutine goes over budget, it has to do some marking work itself, before it can continue. (But this only happens *on allocation*.)

Local caches improve things, but the allocator has to do some bookkeeping. GC work is proportional to the scannable heap -- which is to say, the amount of *pointers*, not the amount of *data*, in the heap.


## SECTION 2: Tools for analyzing and measuring things.

Reducing allocation should improve performance, yes? Maybe! It depends on other things, like what the program is bound on -- CPU-bound or IO-bound programs may not care that much. Builtin profile can tell us where allocations happen, but it can't address the question of whether it'll matter.

Suggested tools:

* Crude experimenting.
* Sampling/profiling (pprof).
* Execution tracer (go tool trace).

How to do crude experimenting: Have you tried turning it off and on again?

We can disable the allocator or garbage collector. Disabling garbage collection (`GOGC=off`) will run you out of memory, but lets you see how much GC was costing. Disabling the allocator (`GODEBUG=sbrk=1`) gives you a naive/simple allocator that will run out of memory eventually, but until then it's cheap. In some cases, disabling the allocator can give a 30% performance improvement; that implies that you probably COULD get improvements! But possibly not; benchmark is obviously synthetic (you can't run real workloads like this), and even the persistent allocator isn't *free*.

A `pprof` CPU profile can show time spent in `mallocgc`. Eben recommends the flamegraph viewer in `pprof` web UI. Can also consider Linux `perf`, too, if you don't have `pprof` enabled in your binaries.

Profiling can show things like where malloc time is going, say, how much is spent getting spans, and how much is spent in gc assist, which would imply garbage collection being more expensive than intrinsic cost of allocating.

Remember that a program that's not CPU-bound may not be allocating on critical path, so it may not actually affect performance that much; you have to look at specific cases.

Execution tracer is the best tool. `curl .../debug/pprof` and `go tool trace trace.out`.

Output is undecipherable at first, but you can zoom in and see more. There's chances to see that mark assist is happening in other procs; that shows that allocation/GC pressure is getting ahead of the default background task.

A good measurement is "minimum mutator utilization": What was the minimum amount of resources available to mutators (goroutines doing work). [Available as CL 60790.](https://go-review.googlesource.com/c/go/+/60790) This gives you a way to see whether production is gc-bound. If you're seeing 75% utilization, that's bad -- goroutines are getting pushed into managing allocation and garbage collection. If it's close to 100%, allocation isn't slowing things down.

Put all together, this lets you see whether allocation/GC is affecting performance, which call sites are spending time on it, and how throughput changes during garbage collection passes.

### SECTION 3: What can we change?

Three things: Limit pointers, allocate batches, recycle objects. NOT an exhaustive list.

What about tuning `GOGC`? That *can* help with throughput. Doesn't always help; high `GOGC` may reduce workload, but larger heap can be more work to search. OOM risk is also significant. Tuning `GOGC` is not as controllable or directed as code optimizations.

Code optimizations:

Thing one: Avoid spurious allocations. Creating a `time.Time` object, and passing its address to a function, creates a pointer and causes heap allocation. Heap allocation implies later garbage collection.

`-gcflags="-m -m"` can give you explanations of why things are heap-allocated. There's a tool (github link) which can explain more.

But there's a more subtle point. A trivial benchmark of allocating bunch of `time.Time` objectstakes nearly 6x as long as the same number of `int64`s. Why? Because `time.Time` contains a pointer (for the timezone). Timezones are bad. But the non-obvious existence of a pointer inside the `time.Time` means that even if you didn't allocate the `time.Time` object and it's just a stack object, no garbage collection to deal with, you *still* have a pointer the garbage collector needs to check.

Would you rather have 1 horse-sized allocation or 100 duck-sized allocations? Smaller objects make better use of mcache (more objects per span), but larger allocs are faster per-byte. There's some overhead that's pretty much fixed per-allocation, so if you can do fewer allocations, you are saving (or amortizing) some of that fixed overhead.

Example: If you know you need a bunch of small buffers with similar lifetimes, you can pool them. But! Any one of those references that outlasts the others will keep the whole slab alive. So that's wasting a bunch of memory. So it's important that they *actually* have similar lifetimes. Also, this strategy isn't concurrency-safe, and if you need to do locking to allocate, it's defeating the point of the exercise. It's only useful if it lets you reduce that overhead, and locking is expensive.

Those two things are pretty easy. The other option is recycling; harder, and not as easy to be sure it'll help.

Recycling needs more discussion. Example from an existing system's architecture: system reads raw data (allocating storage for it), shoves the through channels to things that compute aggregates, which push results on to a sort/limit/whatever. But! That's a ton of allocation happening over the course of processing a whole query. So you have an explicit recycle buffer; the second phase can hand objects it's done with back to the first phase, instead of making the GC do it. First phase can just grab things from buffer channel if available, otherwise make a new one.

More sophisticated version of this: `sync.Pool`. Magical runtime support for per-CPU sharded slices of recycled objects. Lock-free get/put in the fast path. The pool itself does not keep things alive; the runtime can garbage collect `Pool` items that are not being used by anything outside the pool.

Danger: Must be careful to zero/overwrite recycled memory. (True for any recycling process.)

In Honeycomb's case, they drop all the buffers after each query, and don't reuse across queries, which simplifies recycling logic.

### SUMMARY:

Priorities, and mileage, may vary. You may care more about low latency, about predictable latency, or about throughput. These have different implications!

Review: Allocator/GC are pretty ingenious. Allocations cheap, but not free. GC can stop individual goroutines, even without STW. GC work depends on pointer density.

Toolbox: Benchmark with allocator or GC off, CPU profiler, execution tracer, escape analyzer.

And he gets another standing ovation.
