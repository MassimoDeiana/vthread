# Presentation Script

*Follow along by scrolling the page. Each section below maps to what's on screen.*

---

## [Hero — "Rethinking Threads"]

So today I want to talk about three features that came out of Project Loom — virtual threads, scoped values, and structured concurrency. These are things that are either already in our stack or about to be, so I think it's worth understanding what they actually do and why they exist.

---

## [Context — "The problem with platform threads"]

Let's start with the problem. Every thread in Java has always been a wrapper around an OS thread. Each one costs about a megabyte of memory. In theory, to keep your CPU fully busy with I/O-bound work — which is most of what we do — you'd need about a million threads. That's two terabytes of memory. Obviously not happening.

So in practice, our Spring Boot apps run with a pool of about 200 threads. And the thing is, when a thread makes a database call or an HTTP request, it just... sits there. Waiting. Doing nothing. For like a hundred milliseconds, which is an eternity in CPU time. The thread is blocked, the CPU is idle, but we can't serve more requests because all our threads are stuck waiting.

The workaround for the last ten years has been reactive programming — Mono, Flux, CompletableFuture chains. It works, the throughput is great, but the code becomes really hard to read, hard to debug, and the stack traces are basically useless. Maintenance cost goes way up.

Project Loom takes a completely different approach: instead of changing how we write code, it makes threads cheap enough that blocking is fine again.

---

## [Act 1 Header — "Virtual Threads"]

So let's start with virtual threads, which is the big one.

---

## [Old Way — Platform Thread Pool animation]

*[Let the animation play — threads block, queue builds up]*

Here's what happens today. You've got your four threads — or two hundred, doesn't matter. A request comes in, gets a thread. That thread calls the database and... it's blocked. Another request comes in, another thread, calls an external API... blocked. Pretty quickly all your threads are just sitting there waiting for I/O, and new requests start piling up in a queue. Your CPU is doing almost nothing, but you can't serve traffic.

---

## [New Way — Virtual Threads animation]

*[Let the animation play — virtual thread dots appear, float onto carriers]*

Now here's how virtual threads change this. Under the hood, the JVM still uses OS threads — it has to. These are called carrier threads, and there's a small pool of them, roughly one per CPU core.

A virtual thread runs on top of a carrier. When it hits a blocking call — say, a database query — instead of blocking the carrier, it saves its stack to heap memory and steps off. The carrier is immediately free to pick up another virtual thread. When the I/O response comes back, the virtual thread gets mounted back on any available carrier and continues where it left off.

Because they're basically just a stack on the heap — about a kilobyte each — you can create millions of them. No pooling needed. One per task, let it die when it's done.

*[Point at the counter]*

Ten thousand concurrent tasks, zero waiting. Same four CPU cores.

---

## [Code Comparison — thread pool vs virtual threads]

And the best part — look at the code. On the left, the old way: `newFixedThreadPool(200)`. On the right: `newVirtualThreadPerTaskExecutor()`. That's the only change. Same blocking code. The JVM handles the rest.

One important thing though — virtual threads help with I/O-bound work. Database calls, HTTP requests, file reads. If you're doing pure CPU-bound computation — number crunching, parallel streams — platform threads are still the right tool. There's nothing to unmount from if you're not waiting on I/O.

---

## [Act 2 Header — "Scoped Values"]

OK, next one. Scoped values. This one is less flashy but it solves a real pain point.

---

## [Old Way — ThreadLocal animation]

*[Let the animation play — copies appear, one mutates]*

So we all know ThreadLocal. You set a value on it, and that value is tied to your current thread. The problem is, when you spawn child threads, the values get copied. Each child gets its own independent copy. Now someone mutates their copy — suddenly you've got different threads seeing different values for the same thing. Who's right? Hard to tell.

And you have to remember to call `remove()` when you're done, or you leak memory. With virtual threads creating potentially millions of threads, all those copies add up fast.

---

## [New Way — ScopedValue animation]

*[Let the animation play — scope boundary appears, all threads see the same value]*

Scoped values fix all of this. You bind a value for the duration of a scope — this lambda right here. Inside that scope, any thread can read it, but nobody can change it. It's immutable. No copies, because there's nothing to copy — everyone points to the same value. When the scope ends, the value is gone. No cleanup, no leaks.

This one actually just got finalized in Java 25, so it's no longer a preview feature.

---

## [Code Comparison — ThreadLocal vs ScopedValue]

The code is cleaner too. On the left, the try-finally dance with set and remove. On the right, `ScopedValue.where().run()`. The scope handles everything.

---

## [Act 3 Header — "Structured Concurrency"]

Last one — structured concurrency. This is the one that ties everything together.

---

## [Old Way — Unstructured animation]

*[Let the animation play — Task B fails, Task C drifts away]*

So with the old ExecutorService approach, you submit tasks and they fly off into the void. Task B fails here — does the parent know? No. Does Task C stop? No, it keeps running, consuming resources, even though the result is already useless. That's a thread leak. And if you want to cancel things, you have to do it manually, and you'll probably miss some.

---

## [New Way — Structured animation]

*[Let the animation play — tree lines draw, failure propagates, Task C gets cancelled]*

Structured concurrency makes this a first-class concept. You open a scope, fork your tasks — they're all connected. When Task B fails, the scope automatically cancels Task C. The error propagates up to the parent. When the scope closes, everything is cleaned up. No leaks, no orphans.

Think of it like try-with-resources, but for concurrent tasks.

This one is still in preview in Java 25 — the API has been evolving, they changed from a subclass pattern to this Joiner-based approach. But the direction is clear.

---

## [Code Comparison — unstructured vs structured]

On the left — submit three tasks, if one fails, you have to manually figure out what to cancel. On the right — open a scope, fork tasks, join. If anything fails, everything else gets cancelled automatically. The scope handles it.

---

## [Spring Boot — "In practice"]

So that's the theory. How does this affect us concretely?

*[Scroll to the Spring Boot callout]*

One line. `spring.threads.virtual.enabled=true`. That's it. Since Spring Boot 3.2, same in 4.x. Every request handler, every `@Async` method, every scheduled task runs on virtual threads. Your existing blocking code — JPA queries, REST calls, JDBC — it all works as-is, it just scales way better now.

---

## [Code Comparison — Reactive vs Blocking]

And this is what I think is the most compelling part. On the left, what we used to write for throughput — reactive chains, flatMap, zipWith. Hard to read, impossible to debug. On the right — just... normal code. Same throughput, because the virtual threads handle the concurrency for you. Real stack traces. You can step through it in a debugger.

---

## [Conclusion — Three cards]

So to wrap up — these three features are designed to work together.

Virtual threads make threads cheap enough that you can have one per task. Scoped values let you share context across those threads safely. And structured concurrency keeps them organized — parent-child relationships, automatic cancellation.

---

## [Combined code example]

And here's what it looks like when you use all three. Scoped value binds the user for the request. Structured task scope forks three subtasks — each on its own virtual thread. They all read the user through the scoped value. If anything fails, everything else gets cancelled. Clean, readable, debuggable.

---

## [Summary table + Timeline]

*[Let the audience read the table]*

Quick summary of before and after. And here's the timeline — virtual threads finalized in Java 21, scoped values just finalized in Java 25, structured concurrency still in preview but getting close.

---

## [Closing]

The takeaway is pretty simple: write normal, blocking code, and let the JVM handle the concurrency. That's where we're heading.

Any questions?

---

*Estimated delivery time: 15-20 minutes depending on questions and animation replay.*
