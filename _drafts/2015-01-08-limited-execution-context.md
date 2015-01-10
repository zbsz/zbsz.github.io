---
layout: post
title: LimitedExecutionContext
categories : [scala, android]
tagline: "Simplified multi-threading"
---

About ExecutionContext

ExecutionContext is very simple interface, which powers Scala concurrency mechanisms. At first it looks just like basic ExecutorServiceese, but combined with Futures it can give us great control over threading.
I'm mostly programming on Android, and knowing what code is executing on which specific thread turns out to be pretty important in that environment. We can leverage this knowledge to simplify multi-threaded code and make better use of, sometimes scarce, computing resources.
Global execution context (scala.concurrent.ExecutionContext.global) does a pretty good job with clever use of ForkJoinPool which makes it a great general purpose solution. But in specific use case we could achieve more with specialized ExecutionContext implementation.


"Do not block"

There is just one rule that I follow in my code: "do not block", in practice, this means:
do not use Await.*
do not use Thread.sleep, Object.wait or any other operation that blocks a thread
use non-blocking IO when possible
use separate thread pool when you actually need to block - ie: files and database access
In my experience threads and context switching are pretty expensive on Android (especially on older devices), and following this rules make my code more robust and responsive.
Implications

This