---
layout: post
title: CancellableFuture
tags : [scala]
---

There is a reason why standard [`Future`](http://www.scala-lang.org/files/archive/nightly/docs/library/index.html#scala.concurrent.Future),
as well as many other implementations, don't provide `cancel()` method. It's very hard, or impossible, to create general purpose solution,
providing reliable cancellation handling for everyone. But we can create 
[`CancellableFuture`](https://github.com/zbsz/geteit-utils/blob/master/src/main/scala/com/geteit/concurrent/CancellableFuture.scala) 
for code following some specific constraints (see [previous post]({% post_url 2015-01-11-limited-execution-context %})).

### Why CancellableFuture

Being able to cancel some execution is very important for interactive applications. It's especially important on Android, we expect the app
to be responsive, and at the same time don't have much resources to spare. There are several places in typical application, where we want to 
stop some processing. In general this happens every time user goes to another screen or just scrolls away from loading content. 
Most of the time this will be image loading (or downloading), but in some cases also other computations. `CancellableFuture` let's easily cancel our
asynchronous, non-blocking application.
 
### Rules / Spec

1. Cancelled future returns right away, even if some computation is still running (future result will be ignored), 
if it wasn't already completed then it will report special `CancelException` failure result.
2. Cancelling already completed future has no effect.
3. If some future is cancelled then all dependant futures (created through `map`, `flatMap`) will also be cancelled.
4. Cancelling of future created by `map` or `flatMap` cancels any base future that was not yet completed.
5. Cancellation is non-deterministic, future may return successfully if computation completes before `cancel` logic is executed.
6. No thread interrupts, we assume no blocking operations are used, so interrupts are not needed
7. Compatibility with regular `Future`. Due to API limitations (signature of `map`, `flatMap`, etc.), we are not able to extend scala `Future`, but we provide
easy conversions between `Future` and `CancellableFuture`.

### Chaining (`map` / `flatMap`)

Cancelling of chained futures should behave in predictable way, let's see couple examples.

#### Basic case

```scala
val f = CancellableFuture { /* comp_1 */ }
...
f.cancel()
```

In this simple case future `f` will be cancelled (unless it completes before `cancel` is called) and computation `comp_1` 
will either never happen or will be completed but its result will be discarded (depending on a time when cancellation is requested).

#### Using `map`

```scala
val f = CancellableFuture { /* comp_1 */ } map { _ => /* comp_2 */ }
...
f.cancel()
```

Again, f is cancelled if it didn't yet complete, and depending of the timing computations could either be both executed, or only `comp_1` or none of them.

#### Using `flatMap` with already started futures

```scala
val f1 = CancellableFuture { ... }
val f2 = CancellableFuture { ... }
val f3 = f1.flatMap(_ => f2)
...
f3.cancel()
```

This is more interesting example, both futures (`f1` and `f2`) start executing at the same time, but what happens when we call `f3.cancel()`?

 - `f3` is cancelled (assuming it didn't yet complete)
 - `f1` is cancelled if it didn't complete before `cancel()` was called
 - `f2` is either:
    - cancelled if `f1` was completed before `cancel` was called
    - completed if `cancel()` was cancelled after `f2` completed
    - completed if `f1` was cancelled
    
The interesting part is that `f2` completes successfully if `f1` is cancelled, reasoning here is that we only cancel as much as we need to cancel `f3`, 
if `f1` was cancelled (returns `CancelException`) then `f3` is cancelled regardless of the result of `f2`.
Actually `f3` doesn't event know anything about `f2` at all, since its execution never even executed the body of `flatMap` our execution does
    
#### Recursive (maybe infinite) execution

```scala
def recurse(): CancellableFuture[_] =
    CancellableFuture { 
        ... 
    } flatMap { _ => 
        recurse()
    }

val f = recurse()
...
f.cancel()
```

We could chain futures in possibly infinite chain with recursion. Cancellation should follow the same pattern as with regular `flatMap` in previous example,
that means that currently executing future should be cancelled and thus recursion breaks with `CancelException`.

### Implementation

Full [`CancellableFuture`](https://github.com/zbsz/geteit-utils/blob/master/src/main/scala/com/geteit/concurrent/CancellableFuture.scala) implementation
is available on github in my [utils project](https://github.com/zbsz/geteit-utils/). It currently contains only basic methods, doesn't provide complete `Future`
API, but will be extended when needed.

The main implementation complexity arises from the last (recursive) use case example. `CancellableFuture` created by `flatMap` needs to keep references to
its base futures, in recursive case that means we need to keep whole chain of references to all base futures, and it's growing all the time. 
Cancel request has to be propagated through this chain, and it can easily lead to stack overflow, so we need to use *trampolining* to prevent that.
Another issue is that we should not keep any references to completed futures and its results, as this would easily cause memory leaks.
All this concerns, together with use cases can be seen in created tests:
[`CancellableFutureSpec`](https://github.com/zbsz/geteit-utils/blob/master/src/test/scala/com/geteit/concurrent/CancellableFutureSpec.scala).

### Conclusion

Aside from couple implementation issues this is quite simple class, and together with [`LimitedExecutionContext`]({% post_url 2015-01-11-limited-execution-context %}),
provides basic building blocks for responsive application development. It gives us great possibilities, and control of what is actually executing. 
But it also requires us to strictly follow the rules described here and in previous post. 

It's important to remember that cancellation is non-deterministic 
and it's almost impossible to use it in any kind of transactions. Whenever we use `CancellableFuture` we need to remember that execution can be cancelled 
at any step, and we need to make sure that it doesn't lead to inconsistent data.