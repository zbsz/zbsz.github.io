---
layout: post
title: LimitedExecutionContext
tags : [scala, android]
tagline: "For simpler multi-threading"
---

Multiple running threads can easily become very expensive, especially on slower Android devices. 
Once non-blocking software threads outnumber cpu cores, context switching starts consuming huge part of total run time.
Common solution to that problem is using some type of fixed thread pool. I propose going one step further, to achieve even more, 
through the use of specialized [`ExecutionContext`](http://www.scala-lang.org/files/archive/nightly/docs/library/index.html#scala.concurrent.ExecutionContext).

### About ExecutionContext

`ExecutionContext` is very simple interface, powering Scala concurrency mechanisms.
At first it looks just like basic [`Executor`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html), 
but combined with Futures it can give us great control over threading.
Default (`global`) execution context is a great general purpose solution,
due to clever use of [`ForkJoinPool`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ForkJoinPool.html).
But by following some additional rules when coding, we can achieve more with specialized implementation.

### Do not block

This is the main rule that I follow in my code, in practice, this means:

1. do not use Await, Thread.sleep, Object.wait (or any other operation that blocks a thread)
2. use non-blocking IO when possible
3. use separate thread pool when you actually need to block - ie: files and database access

There are couple tools that make following this rule easier, the main one is my [`LimitedExecutionContext`](https://github.com/zbsz/geteit-utils/blob/master/src/main/scala/com/geteit/concurrent/LimitedExecutionContext.scala).

### LimitedExecutionContext

In short, this is just an `ExecutionContext` with hard limit of concurrently executing tasks. 
Additionally, actual execution is performed on given parent `ExecutionContext`, this class does not create any threads on its own.
This lets us chain execution contexts, which is very useful in some situations.

```scala
class LimitedExecutionContext(concurrencyLimit: Int, parent: ExecutionContext) 
    extends ExecutionContext {
  ...
}
```

#### Tasks queue

Each `LimitedExecutionContext` maintains its own queue of tasks to run, and counter of currently running tasks.

```scala
  val queue = new ConcurrentLinkedQueue[Runnable]
  val runningCount = new AtomicInteger(0)
```

Current implementation uses `ConcurrentLinkedQueue`, this could be easily replaced by some other implementation, 
specifically in some cases it could be desired to use bounded queue (unbounded is easier to use generally).

#### Internal executor

`LimitedExecutionContext` doesn't schedule tasks on parent context directly, it uses single internal executor instead.
More importantly, this executor, once started will execute *several* tasks from a queue. 
Executing tasks in batches improves latency, but affects fairness, batch size is currently hard-coded,
but could be made configurable if needed.

```scala
private object Executor extends Runnable {

  override def run(): Unit = {
    // executes scheduled tasks in batch
    def executeBatch(counter: Int = 0): Unit = queue.poll() match {
      case null => // done
      case runnable =>
        runnable.run()
        if (counter < MaxBatchSize) 
          runNext(counter + 1)
    }
    
    executeBatch()
    
    ...
  }
}
```

`LimitedExecutionContext` makes sure that its executor is dispatched on parent `ExecutionContext` whenever some task is scheduled on its queue,
and that `concurrencyLimit` is obeyed (executor is only dispatched if `runningCount` is smaller then current limit). 

#### Full code

Complete code for [`LimitedExecutionContext`](https://github.com/zbsz/geteit-utils/blob/master/src/main/scala/com/geteit/concurrent/LimitedExecutionContext.scala) 
is available in [utils project](https://github.com/zbsz/geteit-utils/) on GitHub, together with tests for it.

### Usage

It may be not obvious how this class can be used, and what we gain from it.
I'm going to use this execution context in my next posts, so it should explain more.
You can also take a look at this [`LimitedExecutionContextSpec`](https://github.com/zbsz/geteit-utils/blob/master/src/test/scala/com/geteit/concurrent/LimitedExecutionContextSpec.scala) to get a feeling what it does.

#### Lightweight actor-like model

One very useful feature of this execution context is ability to serialize tasks execution, which can be used to simulate an Actor.
Let's consider simple counter class:

```scala
class Counter {
  private implicit val ec = new LimitedExecutionContext(1)
    
  private var count = 0
   
  def increment(): Unit = Future {
    count += 1
  }
}
```

In this example, access to variable `count` is not synchronized, nor is this var `volatile`.
But it's still guaranteed to execute properly, as if executed on single thread.
This is achieved by using `LimitedExecutionContext` with concurrency limit of `1`, I call it `serialized execution context`,
and every access to internal state is performed on this context through a `Future`.

This is exactly the style of programming that I use for my applications, thanks to specialized execution context and scala futures, 
multi-threaded data access is much simpler, and still performs pretty well.
