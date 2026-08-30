# 103 — NIO, epoll, and Netty's Event Loop

## Phase: 10 — Reactive & Async at Scale
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: the transport layer underneath `orderflow`'s payment-callback ingestion. Nothing in Topics 104–108 is debuggable without knowing which thread is running your code and what happens to every other connection when that thread stops.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **A few threads multiplex many sockets.**
>
> Each **event loop** is one thread running one loop. Each iteration it calls
> `epoll_wait` (Linux), `kevent` (macOS/BSD) or the Windows equivalent, which blocks
> until at least one of the sockets registered with it becomes ready. The call returns
> a **set of ready file descriptors**. The loop then dispatches each one to the handler
> chain registered for that channel, runs any queued tasks, and goes round again.
>
> **Readiness-based** I/O (epoll, kqueue, select, poll) tells you *"this socket is
> ready — you may now call `read` without blocking"*. You still have to do the read.
>
> **Completion-based** I/O (io_uring on Linux, IOCP on Windows) tells you *"the read
> you asked for earlier has already finished, and the bytes are in the buffer you
> gave me"*. The kernel did the work; you get told afterwards.
>
> Java NIO's `Selector` and Netty's `NioEventLoop` are readiness-based. Java's
> `AsynchronousSocketChannel` presents a completion-shaped API, but on Linux it is
> emulated on top of epoll with a helper thread pool, so it is not a genuine
> completion port.

And the consequence that generates every failure in this document:

> **A channel is bound to exactly one event loop for its entire lifetime.** Every read,
> every write, every handler callback for that connection runs on that one thread. So
> if you block that thread, you do not slow down one request. You stall **every
> connection assigned to that loop**, for exactly as long as you blocked.

---

## The bridge from what you know

### This is the closest honest analogue in the entire curriculum

You already know this machine. You have debugged it. You have explained it to
juniors.

```js
// Node. You know exactly what this does and why.
const server = net.createServer((socket) => {
  socket.on('data', (chunk) => socket.write(handle(chunk)));
});
server.listen(8080);
```

libuv runs one thread. That thread calls `epoll_wait` over every registered file
descriptor. When descriptors become ready it invokes the JavaScript callbacks you
registered, one at a time, run-to-completion. If one callback takes 400 ms, **every
other connection waits 400 ms**, because there is one loop and it is busy.

Netty:

```java
// Java. Structurally the same machine.
ServerBootstrap b = new ServerBootstrap()
    .group(bossGroup, workerGroup)                 // <- the event loops
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        protected void initChannel(SocketChannel ch) {
            ch.pipeline().addLast(new PaymentCallbackHandler());
        }
    });
b.bind(8080).sync();
```

`workerGroup` is a set of threads. Each one runs a loop that calls `epoll_wait` over
the channels registered with it and dispatches readiness to the pipeline you built.
`PaymentCallbackHandler.channelRead` is your `socket.on('data', ...)`.

**Verdict: HONEST ANALOGUE.** Transfer everything. The syscall is the same syscall.
The dispatch model is the same dispatch model. The rule "never block the loop" is the
same rule, for the same reason.

| Node / libuv | Netty | Same? |
|---|---|---|
| The event loop thread | An `EventLoop` | **Yes** |
| `epoll_wait` over registered fds | `Selector.select()` → `epoll_wait` | **Yes** |
| `socket.on('data')` | `channelRead(ctx, msg)` | **Yes** |
| Run-to-completion callbacks | Run-to-completion handler invocation | **Yes** |
| "Never block the event loop" | "Never block the event loop" | **Yes** |
| `setImmediate` / microtask queue | `eventLoop.execute(Runnable)` task queue | **Yes** |
| One loop per process | **N loops per process** | **NO — this is the delta** |
| `worker_threads` for CPU work | `EventExecutorGroup` / `Schedulers.boundedElastic()` | Close enough to transfer |
| `cluster` — N processes, N heaps | N loops, **one shared heap** | **NO — this is the other delta** |

### Delta 1 — N loops, not one. The blast radius has a different shape.

Node gives you one loop per process. Block it and **100% of your traffic** stops. It
is catastrophic and it is obvious: your p50 moves.

Netty gives you N loops, typically one or two per available core. Block one and
**1/N of your connections** stop — specifically, the connections whose file
descriptors happen to be registered with that loop. The other N−1 loops keep running
perfectly.

This is worse to debug, not better:

- Your **p50 barely moves.** With 8 loops and one blocked, 7/8 of requests are fine.
- Your **p99 explodes**, because the affected connections are hit repeatedly — a
  connection is pinned to its loop for its whole lifetime, so a client unlucky enough
  to land on the sick loop is slow on *every* request it makes over that connection.
- The symptom is **bimodal latency**, and averages hide bimodality completely.

Write this down, because it is the single most useful sentence in this topic:

> In Node, blocking the loop is an outage. In Netty, blocking a loop is a
> **percentile problem that looks like a flaky network**.

### Delta 2 — one heap, so handlers are not automatically safe

In Node, "single-threaded" means you get data-race freedom for free. There is nothing
to protect.

In Netty, each channel's handlers run on one thread, so **state private to one
channel** is effectively confined and needs no synchronisation. That much transfers.

But the moment you share anything across channels — a counter, a `HashMap` cache, a
dedup set — you have N event-loop threads touching one object on one shared heap, and
every rule from Phase 9 applies: visibility, atomicity, the Java Memory Model. Topic
86 is not optional here.

The rule:

| State | Netty safety |
|---|---|
| A field on a non-`@Sharable` handler instance (one instance per channel) | **Confined to one loop. Safe.** |
| A field on a `@Sharable` handler instance (one instance for all channels) | **Touched by every loop. Must be thread-safe.** |
| Anything reached through a `static`, a Spring singleton, or a captured lambda | **Must be thread-safe.** |

### Delta 3 — you can see the syscalls

Node hides the loop. You cannot easily count `epoll_wait` calls or inspect the
interest set. In Java you can print the selector provider class, count the loop
threads by name, attach `strace`, read JFR socket-I/O events, and dump every thread's
stack. You have gained observability you never had — and, per §1.7, the obligation to
use it instead of guessing.

---

## What is this?

Three layers, bottom to top. You need all three because a symptom at the top is
almost always caused by a fact at the bottom.

### Layer 1 — the kernel: `epoll`

A **file descriptor** (fd) is a small integer the kernel gives you to refer to an open
socket, file or pipe.

`epoll` is a Linux mechanism for watching many fds at once:

```
epfd = epoll_create1(0);                  // create an epoll instance
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);  // register: "tell me when fd is readable"
n = epoll_wait(epfd, events, 1024, -1);   // block until >= 1 is ready; return how many
```

The important property: **the interest set lives in the kernel**. You register a
socket once. Each `epoll_wait` returns only the fds that are *ready*, and costs time
proportional to the number ready — not to the number watched.

Its predecessors, `select` and `poll`, take the whole watch list as an argument on
every call and scan it linearly. At 50,000 connections that is 50,000 entries copied
into the kernel and scanned, per loop iteration, to discover that 12 of them are
ready. `epoll` is the fix, and it is the reason "10,000 concurrent connections" stopped
being a research problem.

The equivalents elsewhere: **kqueue** on macOS/BSD, **wepoll**/IOCP on Windows,
**io_uring** as the newer Linux completion-based interface.

### Layer 2 — the JDK: `java.nio.channels.Selector`

Java's portable wrapper over whichever of those the platform has.

```java
Selector selector = Selector.open();
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (running) {
    selector.select();                                  // -> epoll_wait
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) { /* accept */ }
        if (key.isReadable())   { /* read    */ }
        if (key.isWritable())   { /* write   */ }
    }
    selector.selectedKeys().clear();                    // you MUST clear it
}
```

| Term | Meaning |
|---|---|
| `SelectableChannel` | A socket you can register with a selector. Must be in non-blocking mode. |
| `SelectionKey` | The registration itself — the (channel, selector) pair, plus interest ops and ready ops. |
| **interest ops** | What you want to be told about: `OP_ACCEPT`, `OP_CONNECT`, `OP_READ`, `OP_WRITE`. |
| **ready ops** | What is actually ready right now. |
| `selectedKeys()` | The ready set. **The JDK does not clear it for you** — forgetting is a classic bug. |

Which implementation you get is decided by `SelectorProvider`. On Linux it is an
epoll-backed provider; on macOS a kqueue-backed one; on Windows a wepoll-backed one
since JDK 13. You will print the actual class in the Hands-on section rather than
trust me on the names.

### Layer 3 — Netty

Raw NIO is correct and miserable: partial reads, partial writes, the `OP_WRITE`
registration dance, buffer management, the infamous JDK selector spin bug. Netty is
the library that makes it usable, and it is what everything in the Java async
ecosystem actually runs on — Spring WebFlux, gRPC-Java, the Elasticsearch and Redis
clients, Cassandra's driver, Vert.x, Zuul.

Four concepts:

| Netty concept | What it is |
|---|---|
| `EventLoop` | **One thread** running one selector loop plus a task queue. |
| `EventLoopGroup` | A set of `EventLoop`s and a policy for assigning new channels to them. |
| `Channel` | One connection. Registered to **exactly one** `EventLoop`, permanently. |
| `ChannelPipeline` | An ordered list of `ChannelHandler`s — inbound and outbound — that the loop walks for each event on that channel. |

A server has two groups by convention:

- **boss group** — accepts new connections. Usually 1 thread. Its only job is
  `OP_ACCEPT`, then handing the new channel to a worker.
- **worker group** — everything else. Typically one or two threads per core.

`ChannelPipeline` is your Express/Koa middleware chain, except it is per-connection,
it is bidirectional (inbound handlers for data arriving, outbound for data leaving),
and it runs on the loop thread.

---

## Why does it matter?

**1. It is the substrate for every other topic in this phase.**
"Which thread am I on?" is the first question in every reactive bug. Topics 104–108
all resolve to facts about event loops. Reactor's `Schedulers`, WebFlux's threading
model, `block()` detection, context propagation across thread hops — none of it makes
sense without this layer.

**2. The most damaging mistake in reactive Java lives here.**
One blocking call — a JDBC query, a `RestTemplate` call, a `.block()`, a synchronous
DNS lookup, a `synchronized` block that contends — on an event-loop thread stalls
every connection on that loop. It produces no exception, no error metric and no log
line. It produces a p99 that a week of dashboard-staring will not explain. BlockHound
exists specifically because this failure is invisible.

**3. It changes how you size a service.**
Thread-per-request sizing asks "how many threads do I need for N concurrent
requests?". Event-loop sizing asks "how many cores do I have, and is any of my work
blocking?". Those produce completely different capacity models — Topic 129's material,
and Topic 107's decision.

**4. It is where your Node intuition is worth the most and is most likely to be
mis-applied.** You will correctly recognise the loop and then incorrectly assume one
loop, no shared state, and an obvious symptom when it stalls. All three are wrong.

**5. Interviewers use it as a filter.**
"Netty is async so it's fast" versus "a few threads multiplex many sockets via epoll,
which avoids per-connection stack cost; the discipline is that no handler may block"
is a visible level difference, and there is no way to bluff the second one.

---

## Machine-level reality

### What registration actually does

When Netty registers a channel:

```
1. socket()                         -> fd 47
2. fcntl(fd, F_SETFL, O_NONBLOCK)   -> non-blocking mode. read() now returns
                                       EAGAIN instead of sleeping.
3. epoll_ctl(epfd, EPOLL_CTL_ADD, 47, {events = EPOLLIN, data = <channel ref>})
```

Two facts fall out of step 2 and step 3 that explain most of Netty's API:

1. **Non-blocking mode is what makes multiplexing possible at all.** A blocking
   `read()` would park the loop thread on one socket. In non-blocking mode `read()`
   returns immediately with either some bytes or `EAGAIN` ("nothing there"). The loop
   never sleeps inside a socket call; it sleeps only inside `epoll_wait`, which is
   watching everything at once.

2. **`epoll_ctl` stores a user pointer alongside the fd.** That is how the kernel
   hands back enough information for Netty to find the right `Channel` in O(1) when
   the fd becomes ready. There is no scan.

### One loop iteration, in order

```
loop {
    // 1. How long may I sleep? If there are queued tasks or scheduled tasks,
    //    the timeout shrinks so the loop wakes up to run them.
    timeout = computeTimeoutFromScheduledTasks();

    // 2. THE BLOCKING CALL. The thread is parked in the kernel here,
    //    consuming no CPU, watching every registered socket at once.
    n = epoll_wait(epfd, readyEvents, maxEvents, timeout);

    // 3. Process I/O: for each ready fd, look up its Channel and fire the
    //    pipeline. This is where YOUR handler code runs.
    for (i = 0; i < n; i++) processReadyChannel(readyEvents[i]);

    // 4. Run queued tasks: things other threads submitted via
    //    eventLoop.execute(...), plus writes scheduled from off-loop.
    runAllTasks(ioRatioBudget);
}
```

Step 3 and step 4 are where your code runs, and **both are single-threaded and
run-to-completion**. That is the whole safety model and the whole hazard.

Netty splits time between steps 3 and 4 using an **I/O ratio** (historically
`setIoRatio`, defaulting to spending equal time on I/O and tasks). The exact knob has
moved between Netty major versions; check `EventLoopGroup`/`IoHandler` configuration
in the version your build resolves rather than copying a blog post.

### Level-triggered vs edge-triggered — and why it matters to you

- **Level-triggered:** `epoll_wait` keeps reporting the fd as ready for as long as
  there is unread data. Forgiving. If you read only half the buffer, you get told
  again next iteration.
- **Edge-triggered:** you are told once, on the transition from "not ready" to
  "ready". If you do not drain the socket until `EAGAIN`, **you are never told
  again** and that connection hangs forever.

The JDK's `Selector` is level-triggered. Netty's **native epoll transport** defaults
to edge-triggered for throughput, and drains correctly on your behalf.

You do not implement this yourself. You need to know it because it explains a class of
bug reports — "one connection hangs, the rest are fine, restarting the client fixes
it" — that appear when a custom transport or a mis-configured `EpollChannelOption` is
in play. The option to look at is `EpollChannelOption.EPOLL_MODE`.

> **Uncertainty, flagged:** I am not going to assert Netty 4.2's exact default for
> `EPOLL_MODE`, nor the exact `IoRatio` API surface, because Netty 4.2 restructured
> the event-loop API (`MultiThreadIoEventLoopGroup` + `IoHandlerFactory`, e.g.
> `NioIoHandler.newFactory()` / `EpollIoHandler.newFactory()`) and I do not want to
> hand you a signature that does not compile. Resolve it from the Netty version your
> Boot BOM pulls in — `./mvnw dependency:tree | grep netty` — and the Netty 4.2
> migration notes.

### Why N loops, and where N comes from

One loop saturates one core. To use eight cores for I/O you need eight loops. That is
the entire reason Netty differs from libuv here — and it is also why Node needs the
`cluster` module (N processes) to use N cores while Java does not (N threads, one
heap).

The default sizes, and this is worth memorising because it is a common misconfiguration:

| Layer | Default | Property to override |
|---|---|---|
| Netty `MultithreadEventLoopGroup` | `max(1, availableProcessors * 2)` | `-Dio.netty.eventLoopThreads` |
| Netty's idea of processor count | `Runtime.getRuntime().availableProcessors()` | `-Dio.netty.availableProcessors` |
| **Reactor Netty** (what WebFlux uses) | `max(availableProcessors, 4)` | **`-Dreactor.netty.ioWorkerCount`** |
| Reactor Netty selector/boss group | small, often 1 | `-Dreactor.netty.ioSelectCount` |

So on a 2-core pod, WebFlux gives you **4** I/O worker threads, not 2, because of the
floor of 4. On a 16-core host you get 16. Verify by counting threads rather than
reasoning — the Hands-on section has the command.

### The container trap — this is Topic 82 wearing a different hat

`availableProcessors()` in a container is derived from the cgroup CPU limit, not from
the host's core count. Roughly: the JVM reads the cpu quota and period, divides, and
rounds up; it also considers `cpuset` and cpu shares.

So:

| Container setting | `availableProcessors()` | Reactor Netty I/O workers |
|---|---|---|
| `--cpus=0.5` | 1 | 4 (the floor saves you — but 4 threads on half a core just context-switch) |
| `--cpus=2` | 2 | 4 |
| `--cpus=4` | 4 | 4 |
| `--cpus=8` | 8 | 8 |
| No limit, 64-core host | 64 | 64 (64 loops, 64 selectors, and a lot of memory you did not budget) |

Two failure modes, both real:

- **Under-provisioned:** a Kubernetes CPU limit of `500m` with a large pod count means
  every replica thinks it has one core. Throughput per pod plateaus and adding replicas
  becomes the only scaling axis.
- **Over-provisioned:** no limit set, so a JVM on a 64-core node creates 64 event
  loops, 64 selectors and 64 sets of thread-local direct-buffer caches — and then gets
  throttled to 2 cores by a cgroup quota it did not read, so 64 threads fight over 2
  cores of runtime.

**Always set `reactor.netty.ioWorkerCount` explicitly in production**, and always
record it next to the Topic 65 baseline. A benchmark whose loop count you did not
record is not reproducible.

### Direct buffers — the Topic 80 connection

The kernel needs a memory address that will not move while it does the I/O. The GC
moves objects. Therefore socket I/O cannot read or write a Java heap array directly.

- If you hand the JDK a **heap** `ByteBuffer`, it copies into a temporary **direct**
  buffer first (cached per thread) and does the syscall from there. You pay a copy.
- If you hand it a **direct** buffer, the address is stable native memory and the
  syscall reads or writes it in place.

That is why **Netty defaults to pooled direct buffers** for socket I/O. Which gives
you Topic 80's entire problem set for free:

- Direct memory is **outside the heap**. Your heap dashboard will show 40% used while
  the pod gets OOMKilled with exit code 137.
- Netty runs its own accounting and its own limit, separate from the JVM's
  `-XX:MaxDirectMemorySize`. The relevant knobs are `-Dio.netty.maxDirectMemory` and
  the pooled-allocator arena settings (`io.netty.allocator.numDirectArenas`, chunk
  size, and so on).
- Netty `ByteBuf` is **reference-counted**. Forgetting `release()` leaks native
  memory. `-Dio.netty.leakDetection.level=paranoid` in staging is how you find it.

Exact default arena counts and chunk sizes have changed across Netty versions. Read
them off your running JVM rather than from memory — the Hands-on section shows how.

### The pipeline is a linked list of handlers on one thread

```
inbound  (bytes arriving)                 outbound (bytes leaving)
  |                                             ^
  v                                             |
[HttpServerCodec] -> [HttpObjectAggregator] -> [YourHandler] -> ... -> socket
```

Each `ChannelHandlerContext` is a node in a doubly-linked list. `ctx.fireChannelRead`
walks forward through inbound handlers; `ctx.writeAndFlush` walks backward through
outbound handlers. All of it runs on the channel's event loop, one event at a time.

Two consequences you will meet:

1. **A handler can be pinned to a different executor.** `pipeline.addLast(myGroup,
   handler)` runs that handler on a separate `EventExecutorGroup` instead of the loop.
   This is the sanctioned escape hatch for work that must block — the Netty equivalent
   of moving work off the loop in Node.
2. **`@Sharable` changes everything.** Without it, `ChannelInitializer` gives each
   channel its own handler instance, so instance fields are thread-confined. With it,
   one instance serves every channel on every loop, and every field is shared mutable
   state across N threads.

### Readiness vs completion, stated precisely

| | Readiness (epoll, kqueue, select) | Completion (io_uring, IOCP) |
|---|---|---|
| The kernel tells you | "you may read now" | "your read has finished" |
| Who supplies the buffer | you, at read time | you, in advance at submit time |
| Syscalls per byte-batch | at least 2 (`epoll_wait` + `read`) | can approach 0 with a polled submission queue |
| Java support | `Selector`, Netty NIO/epoll/kqueue transports | `AsynchronousSocketChannel` (API shape only), Netty's incubating io_uring transport |
| The honest state on Linux+JVM today | this is what you are running | not the default; treat as forward-looking |

**The trap in that table:** Java's `AsynchronousSocketChannel` *looks* completion-based
— you pass a `CompletionHandler` — but on Linux the JDK implements it over epoll with
a helper thread pool. You get the callback API without the syscall savings. Do not
claim "Java has true async I/O on Linux" in an interview; say "the API is
completion-shaped, the implementation is readiness-based, and io_uring is where that
changes."

---

## Example 1 — minimal

### 1a — raw NIO, so you can see the loop with nothing hiding it

```java
package com.orderflow.lab.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.channels.spi.SelectorProvider;
import java.util.Iterator;

/**
 * One thread. Many sockets. This is libuv, written out longhand.
 * Echoes bytes back, and prints which thread did it.
 */
public class BareSelectorEchoServer {

    public static void main(String[] args) throws IOException {
        System.out.println("SelectorProvider = "
                + SelectorProvider.provider().getClass().getName());

        Selector selector = Selector.open();

        ServerSocketChannel server = ServerSocketChannel.open();
        server.bind(new InetSocketAddress(8081));
        server.configureBlocking(false);                       // mandatory
        server.register(selector, SelectionKey.OP_ACCEPT);

        ByteBuffer buffer = ByteBuffer.allocateDirect(16 * 1024);

        while (true) {
            selector.select();                                  // -> epoll_wait
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();                                    // MUST remove

                if (key.isAcceptable()) {
                    SocketChannel client = server.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("accepted on thread "
                            + Thread.currentThread().getName());

                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    buffer.clear();
                    int read = client.read(buffer);
                    if (read < 0) { key.cancel(); client.close(); continue; }
                    buffer.flip();
                    client.write(buffer);
                    System.out.println("echoed " + read + " bytes on thread "
                            + Thread.currentThread().getName());
                }
            }
        }
    }
}
```

Three things to notice, and they are the whole layer:

1. **One thread name** in every printed line. One thread is serving every connection.
2. **`it.remove()`.** The selected-key set is not cleared for you. Omit it and you
   reprocess stale keys forever, burning 100% CPU. This is the most common raw-NIO
   bug and it is the first reason Netty exists.
3. **`client.write(buffer)` may write only part of the buffer.** This code ignores
   that and is therefore wrong for anything larger than a socket buffer. Handling
   partial writes correctly means registering `OP_WRITE`, queuing the remainder, and
   deregistering when drained. This is the second reason Netty exists.

### 1b — the same thing in Netty, plus the thread-name probe

```java
package com.orderflow.lab.nio;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyEchoServer {

    /** NOT @Sharable: one instance per channel, so 'seen' is confined to one loop. */
    static class EchoHandler extends ChannelInboundHandlerAdapter {
        private long seen = 0;                      // safe: single loop touches this

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            seen += ((ByteBuf) msg).readableBytes();
            System.out.println("thread=" + Thread.currentThread().getName()
                    + " channel=" + ctx.channel().id().asShortText()
                    + " totalBytes=" + seen);
            ctx.writeAndFlush(msg);                 // ownership of msg passes on
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            ctx.close();
        }
    }

    public static void main(String[] args) throws Exception {
        EventLoopGroup boss   = new NioEventLoopGroup(1);
        EventLoopGroup worker = new NioEventLoopGroup(2);   // deliberately TWO
        try {
            new ServerBootstrap()
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline().addLast(new EchoHandler());
                    }
                })
                .bind(8082).sync().channel().closeFuture().sync();
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}
```

Open six connections. **What to look for:** the printed thread names. You will see
exactly two distinct worker thread names, and **each channel id always appears with
the same thread name, forever**. That is channel-to-loop affinity, observed directly.
It is the fact that makes the failure drill work.

> `NioEventLoopGroup` is Netty 4.1's spelling. Netty 4.2 introduces
> `MultiThreadIoEventLoopGroup(n, NioIoHandler.newFactory())` and deprecates the old
> constructor. Both were still present during 4.2's transition. **Check which your BOM
> gives you before copying this**; the concept is identical either way.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline, plus a new requirement.

- Dataset: 100k products, 1M orders, 5M order lines. Recorded p50/p95/p99 per
  endpoint, committed under `/docs/java/baselines/`.
- Existing traffic mix: 70% catalogue read, 20% order read, 10% order placement.
- The MVC core runs thread-per-request on virtual threads (Topic 101).
- **New:** the payment provider will POST an asynchronous settlement callback for
  every authorisation. Volume is bursty: normally ~120/second, but the provider
  batches after its own incidents and has been observed to deliver **8,000 callbacks
  in under a minute** as a catch-up burst.
- Each callback is small (~600 bytes of JSON). Processing it means: verify an HMAC
  signature, look up the payment by provider reference, and update its status.
- The provider keeps connections alive and pipelines. Expect a few hundred long-lived
  connections, not one per callback.
- **The contract: a callback must not be lost.** Money reconciliation depends on it.

This is a textbook event-loop workload: many concurrent connections, tiny payloads,
almost no CPU per message. So the callback ingress gets its own Netty-based module.

### The version that ships and quietly ruins the p99

```java
package com.orderflow.payments.callback;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/internal/payments/callbacks")
public class PaymentCallbackController {

    private final PaymentJpaRepository payments;      // <-- blocking JDBC
    private final SignatureVerifier verifier;

    public PaymentCallbackController(PaymentJpaRepository payments,
                                     SignatureVerifier verifier) {
        this.payments = payments;
        this.verifier = verifier;
    }

    @PostMapping
    public Mono<ResponseEntity<Void>> onCallback(@RequestBody CallbackPayload body,
                                                 @RequestHeader("X-Signature") String sig) {
        verifier.verify(body, sig);

        // Looks harmless. It is a blocking JDBC round trip on an event-loop thread.
        Payment payment = payments.findByProviderRef(body.providerRef());
        payment.applyStatus(body.status());
        payments.save(payment);

        return Mono.just(ResponseEntity.accepted().build());
    }
}
```

Everything about this compiles, passes unit tests, passes an integration test at
concurrency 1, and passes a smoke test in staging.

### What actually happens under the baseline

The module runs on Reactor Netty. On a 4-CPU pod, `reactor.netty.ioWorkerCount`
resolves to 4. So there are **four** threads serving every callback connection.

`payments.findByProviderRef` is a blocking JDBC call: the thread issues a query and
parks until Postgres answers. Suppose that is 3 ms at the baseline — a perfectly
healthy number.

Now do the arithmetic that a load test would have done for you:

- 4 loops × (1 second / 3 ms) ≈ **1,330 callbacks/second, absolute ceiling**, and only
  if the loops do nothing else.
- Steady state is 120/second, so **it works fine in production for weeks.**
- The catch-up burst is 8,000 in 60 seconds ≈ 133/second average but arriving in
  clumps of hundreds. The loops saturate.
- While a loop is parked in JDBC, **every other connection registered with that loop
  is frozen.** Not queued behind a fair scheduler — frozen, because there is no
  scheduler; there is one thread and it is asleep in a socket read to Postgres.

The observable symptoms, in the order you will actually meet them:

| Symptom | Where you see it |
|---|---|
| Callback p50 barely moves | your dashboard, which is why nobody investigates |
| Callback p99 goes from 8 ms to several seconds | the percentile panel |
| The provider starts retrying (its own timeout is 10 s) | duplicate callbacks, then a duplicate-processing bug |
| Latency is **bimodal**, clustered by client connection | only visible if you plot a histogram, not a mean |
| Under a worse burst: provider connections time out and are re-established | connection-churn metric, and a fresh set of victims per loop |
| Thread dump: worker threads inside `SocketRead` under a JDBC driver frame | `jcmd <pid> Thread.print` |

And the one that hurts: **if this module shares a JVM with anything else on Netty**,
those connections are on the same loops and suffer too.

### The fix, in three layers

**Layer 1 — get the blocking call off the loop.** The minimum viable correction.

```java
package com.orderflow.payments.callback;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Scheduler;
import reactor.core.scheduler.Schedulers;

@RestController
@RequestMapping("/internal/payments/callbacks")
public class PaymentCallbackController {

    private final PaymentJpaRepository payments;
    private final SignatureVerifier verifier;

    /**
     * A BOUNDED pool, sized against the Hikari pool (Topic 109), not
     * Schedulers.boundedElastic() -- whose bound is deliberately very large and is
     * a safety net, not a capacity decision.
     */
    private final Scheduler jdbcScheduler =
            Schedulers.newBoundedElastic(16, 2_000, "callback-jdbc");

    public PaymentCallbackController(PaymentJpaRepository payments,
                                     SignatureVerifier verifier) {
        this.payments = payments;
        this.verifier = verifier;
    }

    @PostMapping
    public Mono<ResponseEntity<Void>> onCallback(@RequestBody CallbackPayload body,
                                                 @RequestHeader("X-Signature") String sig) {
        return Mono.fromCallable(() -> {
                    verifier.verify(body, sig);              // CPU-only, ~microseconds
                    Payment payment = payments.findByProviderRef(body.providerRef());
                    payment.applyStatus(body.status());
                    payments.save(payment);
                    return ResponseEntity.accepted().<Void>build();
                })
                .subscribeOn(jdbcScheduler);                  // <-- the whole fix
    }
}
```

`subscribeOn` moves the subscription — and therefore the blocking body — onto
`callback-jdbc` threads. The event loop's job shrinks back to: parse the request,
hand off, and later write the response. The loop never blocks.

Two details that matter and are usually got wrong:

- `Mono.fromCallable(...)` and not `Mono.just(doWork())`. `Mono.just` evaluates its
  argument **eagerly, at assembly time, on the calling thread** — which is the event
  loop. `subscribeOn` would then move nothing. This is Topic 104's assembly-vs-
  subscription distinction, and it is the most common way this fix is written wrongly.
- The pool is **bounded at 16**, roughly matched to the HikariCP pool. An unbounded
  offload pool does not remove the problem; it moves it from "loop stalls" to "500
  threads fighting for 16 connections", which is Topic 90's material.

**Layer 2 — bound the ingress, do not just relocate it.** Layer 1 stops the p99
disaster. It does not stop a 8,000-callback burst from queuing 8,000 tasks. Accepting
work you cannot do is Topic 105's subject, and the honest answer for this endpoint is:

```java
// Accept fast, persist durably, process asynchronously.
@PostMapping
public Mono<ResponseEntity<Void>> onCallback(@RequestBody CallbackPayload body,
                                             @RequestHeader("X-Signature") String sig) {
    return Mono.fromCallable(() -> {
                verifier.verify(body, sig);
                inbox.insertIfAbsent(body.providerRef(), body.rawJson());  // idempotent
                return ResponseEntity.accepted().<Void>build();
            })
            .subscribeOn(jdbcScheduler);
}
```

One small insert, then `202 Accepted`. A separate bounded worker drains the inbox
table. The provider's retry contract is satisfied by the insert, not by the
processing. Idempotency is Topic 116; the durable-inbox shape is Topic 115's outbox in
reverse.

**Layer 3 — record the configuration next to the baseline.** Non-negotiable:

```properties
# application-load.properties  -- committed alongside /docs/java/baselines/
reactor.netty.ioWorkerCount=4
reactor.netty.ioSelectCount=1
spring.datasource.hikari.maximum-pool-size=16
```

A benchmark run without recorded loop counts is not reproducible, and the Phase 7 gate
rule is that you must be able to re-run the baseline to within ±10%.

### The honest caveat

The Layer 1 fix removes the stall. It does **not** make this faster than the MVC path.
At the same throughput you are doing the same JDBC work on the same number of
connections, plus a thread hop per request. What you have bought is that the *ingress*
scales with connections rather than with threads, which is exactly the property you
need for a provider that keeps hundreds of sockets open and bursts. Topic 106 argues
that trade with numbers; Topic 107 decides whether you should have used virtual
threads instead.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a blocking call on an event-loop thread

**This is the single most damaging mistake in reactive Java. Everything else in this
section is a footnote to it.**

**Wrong:**

```java
public Mono<Order> lookup(long orderId) {
    Order o = orderJpaRepository.findById(orderId).orElseThrow();  // blocking JDBC
    return Mono.just(o);
}
```

Equally wrong, and all four are things people write without noticing:

```java
restTemplate.getForObject(url, String.class);      // blocking HTTP
someMono.block();                                  // blocking on a Mono
Thread.sleep(50);                                  // yes, people do this
InetAddress.getByName(host);                       // synchronous DNS. Silent killer.
```

**Exact symptom:**

- p50 essentially unchanged. p99 and p999 rise by roughly the blocking duration, and
  keep rising as concurrency grows.
- Latency is **bimodal** — a histogram shows two humps, not a long tail.
- The same client connection is slow repeatedly, while another client is fine.
  (Channel-to-loop affinity: an unlucky connection stays unlucky.)
- `jcmd <pid> Thread.print` shows threads named `reactor-http-nio-N` (or
  `nioEventLoopGroup-N-M`) with a JDBC/socket-read frame in their stack.
- Throughput plateaus at approximately `ioWorkerCount / blockingSeconds` requests per
  second, regardless of how much CPU headroom the pod has. **That formula is the
  fingerprint.** If your ceiling is suspiciously close to `4 / 0.003 ≈ 1330`, you have
  found it.
- With BlockHound installed: an immediate `BlockingOperationError`.

**Root cause:** the loop thread is parked in the kernel waiting for a socket that is
not one of the sockets it is multiplexing. `epoll_wait` is not being called. Every
channel registered with that loop is unserviced for the duration. There is no
scheduler to be fair on your behalf; there is one thread and it is asleep.

**Fix, in order of preference:**

1. Use a non-blocking client for that dependency (`WebClient` instead of
   `RestTemplate`, R2DBC instead of JDBC) — if and only if you are prepared for the
   commitment. R2DBC is a real cost; Topic 106 argues it.
2. Offload with `.subscribeOn(boundedElasticScheduler)` or, in raw Netty,
   `pipeline.addLast(blockingWorkGroup, handler)`.
3. Keep the blocking dependency on the MVC side of the service and do not route that
   path through the event loop at all. Often the right answer.

**And install BlockHound in tests**, so this can never be reintroduced silently. See
Measurement.

---

### Trap 2 — `ioWorkerCount` derived from a container limit nobody checked

**Wrong:** deploying with no explicit loop count and a Kubernetes limit of
`cpu: 500m`.

**Exact symptom:** throughput per pod plateaus far below what the CPU graph suggests
is possible. Adding memory does nothing. Adding replicas does help, which sends the
team down a horizontal-scaling path and hides the real problem. Thread dumps show a
small, suspiciously round number of I/O worker threads.

**Root cause:** `availableProcessors()` reflects the cgroup quota, not the node.
Reactor Netty derives its worker count from it (with a floor of 4). Netty derives
arena counts and thread-local buffer caches from it too. Half a core with four loops
means four threads round-robining through one CPU's worth of time slices, adding
context switches to a workload that was supposed to avoid them.

**Fix:**

```properties
reactor.netty.ioWorkerCount=4
```

set deliberately, and CPU requests/limits chosen with it. Then verify:

```bash
jcmd <pid> Thread.print | grep -c 'reactor-http-nio'
```

Record the number in the baseline file. This is Topic 82's lesson, arriving in a new
costume.

---

### Trap 3 — shared mutable state in a `@Sharable` handler

**Wrong:**

```java
@ChannelHandler.Sharable                                  // one instance, all channels
public class CallbackDedupHandler extends ChannelInboundHandlerAdapter {

    private final Map<String, Long> seenRefs = new HashMap<>();   // plain HashMap

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        CallbackPayload p = (CallbackPayload) msg;
        if (seenRefs.putIfAbsent(p.providerRef(), System.currentTimeMillis()) == null) {
            ctx.fireChannelRead(msg);
        }
    }
}
```

**Exact symptom:** three distinct failures, all intermittent, none reproducible under
low load.

- Duplicate callbacks slip through occasionally — a lost update on the map.
- Occasionally a callback is dropped that was never seen before — a torn read during a
  concurrent resize.
- Very occasionally a thread spins at 100% CPU forever inside `HashMap.get`. A
  concurrent resize of a plain `HashMap` can produce a cyclic bucket chain. The pod
  survives its liveness probe and serves nothing. (Topic 92.)

**Root cause:** `@Sharable` means one handler instance is installed into every
channel's pipeline. Channels live on N different loops. N threads are now mutating one
`HashMap` with no synchronisation. Your Node intuition — "handlers are single
threaded, so this is safe" — is correct **per channel** and false **across channels**.

**Fix:** remove `@Sharable` and give each channel its own instance (state becomes
thread-confined and genuinely safe), or keep `@Sharable` and make the state a
`ConcurrentHashMap` with an atomic operation — noting Topic 92's warning that atomic
operations do not compose, so `putIfAbsent` is right and `containsKey` followed by
`put` is not.

For a dedup set specifically, neither is production-correct: an in-memory set that
only grows is Topic 79's leak, and it does not survive a restart. Idempotency belongs
in the database (Topic 116).

---

### Trap 4 — writing faster than the socket drains

**Wrong:**

```java
for (SettlementLine line : allLinesForToday) {     // 400,000 lines
    ctx.writeAndFlush(encode(line));
}
```

**Exact symptom:** heap or direct memory climbs steeply during the export and does not
come back. Under Netty leak detection you see no leak — nothing is leaked, it is all
still reachable. Eventually `OutOfMemoryError: Direct buffer memory`, or a cgroup
OOMKill with exit code 137 and no heap dump. The connection is slow; the JVM is fine
until it is not.

**Root cause:** `writeAndFlush` does not block. If the peer is slower than you are, or
the network is congested, the bytes cannot leave. Netty queues them in the channel's
**outbound buffer** and returns immediately. You wrote 400,000 messages into memory as
fast as the CPU allowed.

This is backpressure, at the transport layer, and it is exactly why Topic 105 exists.
Netty's answer:

**Fix:**

```java
private void writeNext(ChannelHandlerContext ctx, Iterator<SettlementLine> lines) {
    while (lines.hasNext() && ctx.channel().isWritable()) {   // <-- ask before writing
        ctx.write(encode(lines.next()));
    }
    ctx.flush();
    // When the outbound buffer drains below the low watermark, Netty calls
    // channelWritabilityChanged and we resume from there.
}

@Override
public void channelWritabilityChanged(ChannelHandlerContext ctx) {
    if (ctx.channel().isWritable()) writeNext(ctx, this.pending);
}
```

with watermarks configured:

```java
.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
             new WriteBufferWaterMark(32 * 1024, 64 * 1024));
```

`isWritable()` goes false at the high watermark and true again at the low watermark.
That boolean is a **demand signal from the transport**. Hold that thought: Topic 105
generalises it into `request(n)`.

---

### Trap 5 — leaking a `ByteBuf`

**Wrong:**

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
    if (!looksLikeJson(in)) {
        return;                       // dropped the message. Never released it.
    }
    ctx.fireChannelRead(in);
}
```

**Exact symptom:** with leak detection off (the default is a light sampling level),
**nothing at all** for hours, then direct memory exhaustion:
`io.netty.util.internal.OutOfDirectMemoryError` or
`java.lang.OutOfMemoryError: Direct buffer memory`. Heap looks healthy the whole time
— Topic 80's exact signature.

With `-Dio.netty.leakDetection.level=paranoid` you get a log entry of this shape:

```
LEAK: ByteBuf.release() was not called before it's garbage-collected.
  Recent access records:
  #1: <a stack of the last places this buffer was touched>
```

*(Illustration of the format, not captured output. The real entry has full stack
frames and a `Created at:` section.)*

**Root cause:** `ByteBuf` is reference-counted, not GC-managed, because it wraps
pooled native memory that must be returned to Netty's arena. Dropping the reference
without `release()` returns the buffer to nobody. The arena keeps handing out fresh
chunks.

**Fix:** release what you consume, or extend `SimpleChannelInboundHandler<T>`, which
releases for you after `channelRead0` returns:

```java
public class CallbackHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf in) {
        if (!looksLikeJson(in)) return;         // release happens for you
        ctx.fireChannelRead(in.retain());       // retain if you pass it on
    }
}
```

The rules: whoever consumes a buffer releases it; whoever passes it on `retain()`s it
first if they also keep it. Run staging with `paranoid` leak detection for a week
after any handler change — the sampling default will not find a rare path.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM and will not print output and
call it captured. What follows is the exact source, the exact command, what to look
for, and how to read every result you might get.

### Setup

```bash
mkdir -p ~/java-lab/103 && cd ~/java-lab/103
java --version                # expect 21 or 25

curl https://start.spring.io/starter.zip \
  -d dependencies=webflux \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=loop-lab \
  -d type=maven-project -o loop-lab.zip && unzip loop-lab.zip -d loop-lab
cd loop-lab
```

Confirm what Netty version you actually have — do not assume:

```bash
./mvnw -q dependency:tree | grep -i -E 'netty|reactor'
```

**What to look for:** the `io.netty:netty-*` version and `io.projectreactor.netty:
reactor-netty-http`. Write both down. Every API question in this document is settled by
those two numbers, and I am deliberately not guessing them for you.

### Proof 1 — which selector implementation is under you

```java
public class WhichSelector {
    public static void main(String[] args) {
        System.out.println("provider = "
            + java.nio.channels.spi.SelectorProvider.provider().getClass().getName());
        System.out.println("cpus     = " + Runtime.getRuntime().availableProcessors());
    }
}
```

```bash
java WhichSelector.java
```

| What you see | What it means |
|---|---|
| A provider class name containing `EPoll` | Linux. You are on epoll. The mechanical statement applies literally. |
| A provider class name containing `KQueue` | macOS or BSD. kqueue — same readiness model, different syscall. |
| A provider class name containing `WEPoll` | Windows, JDK 13+. Readiness model again. |
| `cpus` is not what your machine has | You are in a container or have `-XX:ActiveProcessorCount` set. Everything downstream — loop counts, arenas, pools — is derived from this number. Topic 82. |

Netty's *native* transports (`EpollEventLoopGroup`, `KQueueEventLoopGroup`) bypass the
JDK selector entirely and call the syscalls through JNI. Check whether one is active:

```bash
./mvnw spring-boot:run -Dspring-boot.run.jvmArguments="-Dio.netty.noUnsafe=false" 2>&1 \
  | grep -i -E 'epoll|kqueue|native transport'
```

### Proof 2 — count your event loops and learn their names

Start the app, then:

```bash
jcmd -l                                              # find the pid
jcmd <pid> Thread.print > threads.txt

grep -o 'reactor-http-nio-[0-9]*'  threads.txt | sort -u
grep -o 'reactor-http-epoll-[0-9]*' threads.txt | sort -u
grep -c 'boundedElastic'            threads.txt
```

| What you see | What it means |
|---|---|
| `reactor-http-nio-1..4` on a 2-core box | The floor of 4 in `max(availableProcessors, 4)`. Expected, and worth knowing before you tune. |
| `reactor-http-epoll-*` instead of `-nio-` | The native epoll transport is active. Slightly better throughput, and `EpollChannelOption`s become available. |
| One loop thread only | Someone set `reactor.netty.ioWorkerCount=1`, or `availableProcessors()` is 1 **and** the floor was overridden. Your whole service is a single Node process now. |
| Dozens of loop threads | No CPU limit on a big host. You have far more loops than useful, each with its own selector and buffer caches. |
| Many `boundedElastic-*` threads | Work is being offloaded. Good — or a sign that everything is offloaded and you gained nothing over MVC. |

Now change it and watch the count change:

```bash
./mvnw spring-boot:run \
  -Dspring-boot.run.jvmArguments="-Dreactor.netty.ioWorkerCount=2"
jcmd <pid> Thread.print | grep -o 'reactor-http-nio-[0-9]*' | sort -u | wc -l
```

**What to look for:** exactly 2. If it does not change, you set the property on the
wrong JVM (Maven's, not the forked app's) — that is what
`-Dspring-boot.run.jvmArguments` is for.

### Proof 3 — print the thread name inside your handler (the primary instrument)

This is the cheapest and most valuable diagnostic in the entire phase.

```java
@RestController
public class WhichThreadController {

    @GetMapping("/whichthread")
    public Mono<String> whichThread() {
        String atAssembly = Thread.currentThread().getName();
        return Mono.fromSupplier(() -> "supplier:" + Thread.currentThread().getName())
                .map(s -> s + " | map:" + Thread.currentThread().getName())
                .doOnNext(s -> System.out.println("onNext on " + Thread.currentThread().getName()))
                .map(s -> "assembly:" + atAssembly + " | " + s);
    }
}
```

```bash
for i in 1 2 3 4 5 6 7 8; do curl -s localhost:8080/whichthread; echo; done
```

| What you see | What it means |
|---|---|
| Every operator reports the same `reactor-http-nio-N` name | Normal. No scheduler switch occurred; the whole chain ran on the I/O loop that read the request. |
| The name varies across requests but not within one | Different connections landed on different loops. Exactly the affinity you want to see. |
| `assembly:` shows a different thread from the rest | Correct and important: the chain was *assembled* on the request thread and *executed* at subscription. Topic 104. |
| Any operator reports `boundedElastic-N` | A `subscribeOn`/`publishOn` moved execution off the loop. Deliberate offload — confirm you meant it. |
| Names like `virtual-N` or `VirtualThread-...` | You are on a virtual-thread path, not an event loop. Different topic entirely (101). |

Keep this endpoint. In an incident, "which thread ran my code" is the first question,
and this answers it in one curl.

### Proof 4 — install BlockHound and catch a blocking call

BlockHound instruments the JDK so that a blocking method called from a thread it
considers non-blocking throws immediately.

```xml
<dependency>
  <groupId>io.projectreactor.tools</groupId>
  <artifactId>blockhound</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
class NoBlockingOnEventLoopTest {

    @BeforeAll
    static void installBlockHound() {
        reactor.blockhound.BlockHound.install();
    }

    @Test
    void callback_endpoint_does_not_block_the_loop() {
        webTestClient.post().uri("/internal/payments/callbacks")
            .bodyValue(samplePayload())
            .exchange()
            .expectStatus().isAccepted();
    }
}
```

**What to look for:** either the test passes, or it fails with a
`reactor.blockhound.BlockingOperationError` naming the blocking method and the thread.
The error shape is roughly:

```
reactor.blockhound.BlockingOperationError: Blocking call!
  java.net.Socket read
    at ...
```

*(Illustration of the format, not captured output.)*

| What you see | What it means |
|---|---|
| `BlockingOperationError` naming a JDBC or socket method | **You have found Trap 1.** Read the thread name in the stack: if it is `reactor-http-nio-*`, it is on a loop. |
| Test passes | No *instrumented* blocking call ran on a non-blocking thread on this path. Not a proof of absence — BlockHound knows about JDK blocking primitives, not about your CPU-heavy loop. |
| BlockHound fails to install with an instrumentation error | Java agent/module access. Add `-XX:+AllowRedefinitionToAddDeleteMethods` and check BlockHound's docs for your JDK; support for the newest JDKs sometimes lags. **Flagged uncertainty: I am not certain BlockHound installs cleanly on JDK 25 without extra flags — verify before relying on it in CI.** |
| False positives on your own code | Use `BlockHound.builder().allowBlockingCallsInside(className, methodName)` to whitelist known-safe paths, and record why in a comment. |

Run BlockHound in **tests and staging**, not production: it instruments the JDK and
has a cost.

### Proof 5 — watch `epoll_wait` with `strace` (Linux)

```bash
sudo strace -f -p <pid> -e trace=epoll_wait,epoll_ctl,read,write -c
# ... generate load for 10 seconds ...
# Ctrl-C to see the summary table
```

| What you see | What it means |
|---|---|
| `epoll_wait` count roughly tracking request count, with `read`/`write` alongside | Healthy readiness loop. Two-ish syscalls per I/O batch, exactly as the model predicts. |
| `epoll_wait` count enormously higher than request count | Busy-spinning. Either the JDK selector spin bug (Netty works around it), or a zero timeout being passed. |
| `epoll_ctl` called constantly at steady state | Interest ops are being re-registered per operation. Usually an outbound-buffer/`OP_WRITE` pattern — often a symptom of Trap 4. |
| `strace` shows a thread stuck with no syscalls at all for a long stretch | That thread is running Java code, not doing I/O. If it is a loop thread, that is a CPU-bound handler — the other way to stall a loop. |

Use `-c` for the summary. Full tracing of a busy server will drown you and will itself
change the timing.

### Proof 6 — JFR, the production-safe instrument

`strace` is a debugging tool. JFR is what you leave on.

```bash
java -XX:StartFlightRecording=duration=120s,filename=loop.jfr,settings=profile \
     -jar target/loop-lab-0.0.1-SNAPSHOT.jar

jfr summary loop.jfr
jfr print --events jdk.SocketRead,jdk.SocketWrite loop.jfr | head -60
jfr print --events jdk.ThreadPark loop.jfr | head -60
```

| What you see | What it means |
|---|---|
| `jdk.SocketRead` events whose thread is `reactor-http-nio-*` and whose duration is long | A loop thread was inside a socket read for a long time. If the remote address is your **database**, that is Trap 1, captured in production-safe form. |
| `jdk.ThreadPark` on a loop thread | A loop thread parked on a lock or a future. Almost always a blocking call in disguise — `.block()`, a `CountDownLatch`, a contended `synchronized`. |
| Long `jdk.SocketRead` on `boundedElastic-*` threads | Expected and fine. That pool exists to block. |
| `jdk.ObjectAllocationSample` dominated by `ByteBuf`-adjacent frames | Allocation pressure from buffer churn — check that pooling is enabled. Topics 68 and 80. |

**Why JFR beats a thread dump here:** a thread dump is one instant. Loop stalls are
brief and frequent. JFR records durations over a window, which is the shape of the
evidence you need.

### Proof 7 — Netty's own diagnostics

```bash
# Leak detection: run staging at 'paranoid' after any handler change.
-Dio.netty.leakDetection.level=paranoid

# Pooled allocator behaviour, printed at startup with this logger at DEBUG:
logging.level.io.netty.buffer=DEBUG
logging.level.io.netty.util.internal.PlatformDependent=DEBUG

# The allocator's own metrics, from application code:
PooledByteBufAllocator.DEFAULT.metric()      // arenas, chunk lists, allocation counts
```

| What you see | What it means |
|---|---|
| Startup DEBUG lines reporting direct-arena count, chunk size and max direct memory | Your actual allocator configuration. **Read these instead of quoting defaults** — they have changed across Netty versions. |
| `LEAK:` entries | Trap 5. The "recent access records" section names the last handler that touched the buffer. Start there. |
| `noUnsafe` or `Unsafe unavailable` messages | Netty has fallen back to slower paths. On a modern JDK with strong encapsulation this is worth investigating, since it affects allocation cost. |

---

## Failure drill

**Mandatory.** Produce the failure yourself and write down what you saw before reading
the analysis. The point is not the fact; it is the memory of watching four connections
freeze while four others did not.

### The scenario

Two event loops. Eight concurrent clients. One endpoint that blocks for 500 ms.
**Prediction to write down first:** how many of the eight clients will be slow, and by
how much?

### Setup

`application.properties`:

```properties
reactor.netty.ioWorkerCount=2
server.port=8080
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{20} - %msg%n
```

That log pattern is not decoration. `%thread` is the instrument for this whole drill.

`BlastRadiusController.java`:

```java
package com.orderflow.lab.loop;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;

@RestController
public class BlastRadiusController {

    private static final Logger log = LoggerFactory.getLogger(BlastRadiusController.class);

    /** Fast. Does nothing but report which loop served it. */
    @GetMapping("/fast")
    public Mono<String> fast() {
        String t = Thread.currentThread().getName();
        log.info("FAST served on {}", t);
        return Mono.just("fast on " + t);
    }

    /** THE POISON. A blocking sleep, on the event loop, on purpose. */
    @GetMapping("/blocking")
    public Mono<String> blocking() throws InterruptedException {
        String t = Thread.currentThread().getName();
        log.info("BLOCKING starting on {}", t);
        Thread.sleep(500);                                  // <-- the whole drill
        log.info("BLOCKING finished on {}", t);
        return Mono.just("blocked on " + t);
    }

    /** The fix, for part D. Same work, off the loop. */
    @GetMapping("/offloaded")
    public Mono<String> offloaded() {
        return Mono.fromCallable(() -> {
                    String t = Thread.currentThread().getName();
                    log.info("OFFLOADED running on {}", t);
                    Thread.sleep(500);
                    return "offloaded on " + t;
                })
                .subscribeOn(Schedulers.boundedElastic());
    }
}
```

### Commands

Terminal 1 — the server:

```bash
./mvnw spring-boot:run
```

Terminal 2 — eight persistent clients hammering `/fast`. **Persistent matters**:
each `curl` process keeps one connection, so each stays pinned to one loop.

```bash
for i in $(seq 1 8); do
  ( while true; do
      curl -s -o /dev/null -w "client$i %{time_total}\n" \
           --keepalive-time 60 localhost:8080/fast
      sleep 0.05
    done ) &
done
```

Terminal 3 — after 10 seconds of clean baseline, fire the poison:

```bash
sleep 10
for i in 1 2 3; do curl -s -o /dev/null localhost:8080/blocking; done
```

Better still, replace terminal 2 with k6 so you get a percentile histogram rather than
a scroll of numbers:

```javascript
// blastradius.js
import http from 'k6/http';
import { Trend } from 'k6/metrics';
const fastLatency = new Trend('fast_latency');

export const options = {
  scenarios: {
    steady: { executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s',
              duration: '60s', preAllocatedVUs: 20 },
  },
};

export default function () {
  const r = http.get('http://localhost:8080/fast');
  fastLatency.add(r.timings.duration);
}
```

```bash
k6 run blastradius.js &
sleep 20
for i in $(seq 1 20); do curl -s -o /dev/null localhost:8080/blocking; done
wait
```

Note the **open-model arrival rate** (`constant-arrival-rate`), not a fixed VU loop.
That is the Topic 65 rule: a closed VU loop hides tail latency through coordinated
omission, which would conceal the exact effect you are trying to see.

### What to capture

Write these down before reading on:

1. How many **distinct** thread names appear in the `FAST served on ...` lines.
2. Which thread name appears in `BLOCKING starting on ...`.
3. During the block, which `client$i` timings spike — and whether it is the *same*
   clients each time you repeat.
4. The k6 `fast_latency` percentiles: p50, p90, p99, max.
5. Repeat with `/offloaded` instead of `/blocking` and record the same five things.

### How to read it

| What you see | What it means |
|---|---|
| Exactly two distinct `reactor-http-nio-N` names on `/fast` | Confirmed: `ioWorkerCount=2` took effect. Two loops are serving eight connections. |
| Roughly half the clients spike to ~500 ms; the other half are untouched | **The drill has fired.** The blocked loop's channels froze for exactly the sleep duration. The other loop never noticed. This is the N-loop blast-radius shape. |
| The *same* clients spike every time you repeat the poison | Channel-to-loop affinity, observed. A connection is pinned to its loop for life, so an unlucky client is unlucky repeatedly. **This is why real users report "the app is slow for me and fine for my colleague".** |
| k6 p50 barely moves; p99 jumps toward 500 ms | The exact production symptom. **A mean-latency dashboard would show nothing here.** Write this down. |
| Latency histogram has two humps | Bimodality. Averages are meaningless on this distribution; percentiles are mandatory. |
| **All** clients spike | You have one loop, not two. Check the thread-name count — the property did not apply. |
| **No** client spikes | Your clients are not keeping connections alive, so each request lands on a fresh channel and can be assigned to the free loop. Add `--keepalive-time`, or use k6, which reuses connections. |
| `/offloaded` costs the caller 500 ms but no other client spikes | The fix, proven. The work still takes 500 ms — you did not make it faster — but the **blast radius is now one request instead of half the connections**. |

### The sentence to carry out of this drill

> Blocking an event loop does not make one request slow. It makes **every connection
> that happens to live on that loop** slow, repeatedly, while your p50 tells you
> everything is fine.

### Part E — argue against yourself

The offload fixed the blast radius and cost a thread hop and a thread pool. Under what
traffic shape would you *not* bother — that is, when is the blocking call so short
that leaving it on the loop is defensible? Compute the ceiling
(`ioWorkerCount / blockingSeconds`) for a 200 µs call at `ioWorkerCount=8`, compare it
to the Topic 65 arrival rate, and then state what would have to change for your answer
to flip. Write the answer down; Topic 107 will ask you for it again.

---

## Measurement

### The instrument for each claim

| Claim you want to make | Instrument | Never use |
|---|---|---|
| "This code runs on an event loop" | print `Thread.currentThread().getName()` in the operator | reading the code and assuming |
| "Nothing blocks the loop" | **BlockHound** in test + staging | code review alone |
| "We have N loops" | `jcmd <pid> Thread.print \| grep -c reactor-http-nio` | the documented default |
| "A loop stalled for X ms" | JFR `jdk.SocketRead` / `jdk.ThreadPark` durations | a thread dump (one instant) |
| "Throughput is loop-bound" | throughput plateau ≈ `loops / blockingSeconds` | CPU utilisation, which will look low |
| "This is faster than MVC" | k6 open-model run vs the recorded Topic 65 baseline | a `System.nanoTime()` loop |
| "Direct memory is stable" | `jcmd VM.native_memory summary` + Netty leak detection | heap graphs |

### Comparing against the Topic 65 baseline — the rules

The baseline is only a baseline if the comparison is honest.

1. **Match the arrival rate, not the concurrency.** Use k6's
   `constant-arrival-rate`. A closed VU loop lets the system slow the load generator
   down, which hides tail latency — coordinated omission, Topic 65's core lesson.
2. **Match the dataset.** 100k products, 1M orders, 5M order lines. A benchmark
   against 100 rows measures your cache, not your service.
3. **Match the cache state.** Warm both runs identically, or record that both are
   cold.
4. **Record the JVM and loop configuration with the numbers.** Heap, collector,
   container CPU limit, `reactor.netty.ioWorkerCount`, Hikari pool size. Topic 65's
   gate rule is ±10% reproducibility; you cannot hit that if the loop count drifted.
5. **Report p50/p95/p99/p999 and error rate, per endpoint.** Never a mean. This
   topic's entire failure mode hides inside a mean.
6. **Report the histogram shape, not just the percentiles.** Bimodality is the
   fingerprint of a loop stall, and percentiles alone can look merely "a bit worse".

### The standing rule: a naive `System.nanoTime()` loop is WRONG

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 100_000; i++) {
    webClient.get().uri("/fast").retrieve().bodyToMono(String.class).block();
}
System.out.println((System.nanoTime() - start) / 100_000 + " ns/op");
```

It is wrong for the four JIT reasons from Topic 77 — dead-code elimination, constant
folding, on-stack replacement, cold-JIT and profile pollution — and for four more that
are specific to this topic:

1. **`.block()` in the loop makes it a closed model.** You have built a coordinated-
   omission machine: when the server slows down, your loop issues fewer requests, so
   the very latency you were trying to measure never gets sampled.
2. **One connection, one loop.** A single-threaded client uses one channel, which is
   pinned to one event loop. You measured 1/N of the server.
3. **Connection setup is in the measurement** unless you explicitly reuse.
4. **The server is warming up too.** Both JITs are cold, and Netty's pooled allocator
   arenas fill during the first thousands of requests.

Use JMH for in-JVM microbenchmarks (Topic 77) and k6 with an open arrival model for
service-level numbers (Topic 65). They answer different questions and neither
substitutes for the other.

### What to graph permanently in production

- `reactor.netty.eventloop` / `reactor.netty.connection.provider` metrics via
  Micrometer (Topic 118), plus your own gauge for active connections per loop if the
  binder does not give it to you.
- **A latency histogram, not a mean**, per endpoint. Bimodality is the signal.
- Thread counts by name prefix: loops, `boundedElastic`, JDBC pool.
- Direct memory (`jcmd VM.native_memory`, or a JMX gauge) alongside heap. Topic 80.
- A **BlockHound-enabled staging deployment** running the same load profile. This is
  the only automated way to prevent Trap 1 from being reintroduced by a future PR that
  adds one innocent-looking repository call.

---

## Practice exercises

### 1 — Easy: map your loops

**Part A.** Start the WebFlux app. Without changing any configuration, determine
empirically:

1. How many I/O worker threads exist, and their exact names.
2. What `Runtime.getRuntime().availableProcessors()` returns in that JVM.
3. Which `SelectorProvider` is in use.

**Part B.** Predict the worker count from (2), then check it against (1). If they
disagree, explain why — the answer involves a floor.

**Part C.** Re-run under Docker with `--cpus=0.5`, then `--cpus=2`, then `--cpus=8`.
Produce a table: cpus → `availableProcessors()` → loop count. State the rule the table
implies, and name the one row where the rule and the observation diverge.

**Part D.** Open six connections with `curl --keepalive-time 60` and hit an endpoint
that prints its thread name. Show that each connection is served by the same loop every
time. Then explain, in one sentence, why that makes an incident report of "it's slow
for one customer" plausible rather than absurd.

### 2 — Medium: the audit (combines Topics 01–102)

This Netty handler is part of `orderflow`'s callback ingress. It contains **seven**
defects. Four are from this topic; three are from earlier topics. For each: name the
topic, state the **observable** symptom in production — what does the on-call engineer
actually see? — and write the fix.

```java
package com.orderflow.payments.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.util.CharsetUtil;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@ChannelHandler.Sharable
public class CallbackIngestHandler extends ChannelInboundHandlerAdapter {

    private static final Map<Long, Integer> processedCountByMerchant = new HashMap<>();

    private final PaymentJpaRepository payments;
    private final List<String> auditTrail = new java.util.ArrayList<>();

    public CallbackIngestHandler(PaymentJpaRepository payments) {
        this.payments = payments;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf in = (ByteBuf) msg;
        String json = in.toString(CharsetUtil.UTF_8);

        CallbackPayload p = Json.parse(json);

        Integer count = processedCountByMerchant.get(p.merchantId());
        processedCountByMerchant.put(p.merchantId(),
                count == null ? 1 : count + 1);

        Payment payment = payments.findByProviderRef(p.providerRef());
        if (payment.amountMinor() == p.amountMinor()) {
            payment.applyStatus(p.status());
            payments.save(payment);
        }

        auditTrail.add(p.providerRef());

        ctx.writeAndFlush(ctx.alloc().buffer().writeBytes("OK".getBytes()));
    }
}
```

Hints, in the order to think about them. One defect makes the service **stall every
connection on a loop** — find that one first, because its symptom will mask the
others. One is a Topic 01 wrapper-comparison bug that will pass every test you write
with small fixture values. One is a Topic 79 unbounded-growth leak. One is a Topic 92
concurrent-collection bug that can pin a CPU at 100%. Two are this topic's
buffer-lifecycle rules. One is a `@Sharable`/state-confinement error.

For each defect also answer: **would a unit test have caught it?** Be honest. Several
would not, and saying so is the point.

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — establish.** Bring up the Topic 65 stack (`orderflow` + Postgres + Redis,
100k products / 1M orders / 5M order lines). Re-run the recorded baseline and confirm
you land within ±10%. If you do not, stop and fix that first — the gate rule exists
for this exercise.

**Part B — add the ingress.** Build the WebFlux callback module from Example 2 with
the **naive blocking** version. Deploy it in the same JVM as the MVC core. Add a k6
scenario: the existing 70/20/10 mix, plus a callback stream at 120/second, plus a
**burst of 8,000 callbacks over 60 seconds** starting at t=180s.

**Part C — capture the damage.** During the burst, capture all of:

- k6 percentiles for **every** endpoint, including the catalogue reads that have
  nothing to do with payments;
- `jcmd Thread.print` at three points during the burst;
- a JFR recording covering the burst, and the `jdk.SocketRead` events for loop
  threads;
- the number of provider retries (duplicate `providerRef` values received).

Then answer: **did the catalogue-read p99 move?** Explain, mechanically, why or why
not — the answer depends on whether the MVC core shares loops with the ingress, and
you must determine which is true in your setup rather than assuming.

**Part D — fix and re-measure.** Apply the Layer 1 offload. Re-run the identical k6
script. Produce a table: endpoint → p50/p95/p99 before → after → baseline. State
plainly where the fix helped, where it did nothing, and where it made something
slightly worse (there will be somewhere — a thread hop is not free).

**Part E — find the new ceiling.** With the offload in place, increase the burst until
something breaks again. It will not be the loops this time. Identify what it is
(candidates: the bounded scheduler's queue, the Hikari pool, Postgres itself, direct
memory) and prove which with an instrument rather than an argument. Then say what
Topic 105 would do about it.

**Part F — argue against yourself.** You now have an event-loop ingress with a
bounded offload pool in front of a blocking database. Make the strongest possible case
that this whole module should have been thread-per-request on virtual threads (Topic
101) instead. Then state what would have to be true about the callback traffic for the
event-loop version to win. **Keep this answer. Topic 107 makes you defend it.**

---

## Interview questions

### Q1 — "Netty is fast. Why?"

**Mid-level answer:** "It's non-blocking and asynchronous, so it doesn't need a thread
per connection."

**Senior answer:** "The mechanism is that a small number of threads multiplex many
sockets. Each event loop is one thread calling `epoll_wait` over the file descriptors
registered with it; the kernel returns only the ready ones, and the loop dispatches
each to that channel's handler pipeline. So the cost model changes: instead of a
thread stack per connection — on the order of a megabyte of reserved stack, plus a
scheduler entry, plus context switches — you pay a few kilobytes of channel state and
one entry in the kernel's epoll interest set.

That matters at high **connection** counts, which is a different thing from high
request rates. Ten thousand mostly-idle websocket or keep-alive connections is the
canonical win. Four hundred requests per second against a database is not; there the
bottleneck is the database and the threading model is irrelevant.

And the speed is conditional on one absolute discipline: **no handler may block.** A
blocking call parks the loop thread, and every connection registered with that loop
stops until it returns. Blocking work goes on a separate `EventExecutorGroup`, or a
`boundedElastic` scheduler in the Reactor case."

**What separates them:** naming the syscall and the cost model rather than the
adjective, distinguishing connection scale from request scale, and volunteering the
discipline that makes it conditional. The last part is what an interviewer is actually
listening for.

**Follow-up:** "Where does the thread count come from?" `availableProcessors()`, with
Reactor Netty's floor of 4 — and in a container that number comes from the cgroup
quota, not the node.

---

### Q2 — "What is the difference between readiness-based and completion-based I/O?"

**Mid-level answer:** "Readiness tells you when you can read; completion tells you when
the read is done."

**Senior answer:** "That's the definition, and the consequences are where it gets
interesting. With readiness — epoll, kqueue, select — the kernel says 'this fd is
readable', and you then issue the `read` yourself. So the minimum is two syscalls per
I/O batch, and you supply the buffer at read time, which means the buffer can be
shared across connections.

With completion — IOCP on Windows, io_uring on Linux — you submit the read *in
advance*, with the buffer, and the kernel tells you afterwards that it filled it. With
io_uring's submission and completion rings you can batch many operations per syscall,
and in polled mode approach zero syscalls in steady state. The trade is that you must
commit a buffer per in-flight operation, so memory scales with concurrency in a way
readiness does not.

For Java specifically there's a trap worth stating: `AsynchronousSocketChannel` looks
completion-based because it takes a `CompletionHandler`, but on Linux the JDK
implements it over epoll with a helper thread pool. You get the callback API without
the syscall savings. Netty has an incubating io_uring transport, but epoll is what
essentially all production Java is running today. I'd want to benchmark rather than
assume io_uring is a win — it's most compelling for very high IOPS, and less so for a
service whose bottleneck is Postgres."

**What separates them:** the buffer-ownership consequence, and knowing that Java's
"async" channel API is emulated on Linux. That last fact is the one people get wrong
in both directions.

**Follow-up:** "So should we move to io_uring?" The right answer starts with "what is
the current bottleneck?" and refuses to answer until it is named.

---

### Q3 — "One handler in your Netty service calls a blocking JDBC query. What happens?"

**Mid-level answer:** "It blocks the event loop, so it hurts performance. You should
move it to another thread pool."

**Senior answer:** "It stalls that loop for the duration of the query — and the
specific shape of the damage is what makes it hard to find. A channel is bound to one
event loop for its whole lifetime, so a stall doesn't hurt 'some requests'; it hurts
**every connection assigned to that loop**, repeatedly, while the other loops are
perfectly healthy.

Concretely: with 4 loops and a 3 ms query, my throughput ceiling for that path is about
4 divided by 0.003, so roughly 1,300 per second, no matter how much CPU headroom the
pod has. Below that it looks fine. Above it, p50 barely moves because three quarters of
requests are on healthy loops, while p99 explodes. The latency histogram goes bimodal,
and users report it as 'slow for some customers' — which it literally is, because a
customer's keep-alive connection is pinned to one loop.

To find it: BlockHound in tests and staging, which throws on a blocking call from a
non-blocking thread; JFR `jdk.SocketRead` and `jdk.ThreadPark` events filtered to
threads named `reactor-http-nio-*`; and printing `Thread.currentThread().getName()`
inside the operator, which is the fastest thing to reach for in an incident. To fix it:
`subscribeOn` a bounded scheduler sized against the connection pool, or a dedicated
`EventExecutorGroup` for that handler — and I'd size the bound deliberately rather than
using the default `boundedElastic`, whose bound is a safety net, not a capacity
decision.

The design question underneath is whether this service should be on an event loop at
all. If every request ends in a blocking JDBC call, the event loop is buying me
nothing and costing me debuggability."

**What separates them:** the throughput-ceiling arithmetic, the bimodal-p99 symptom,
naming BlockHound, and finishing on the design question rather than the mechanical fix.

**Follow-up:** "Why does p50 not move?" Because N−1 of N loops are unaffected, so the
median request never touches the sick one.

---

### Q4 — "Your Node experience says one blocked callback is an outage. Is that also true in Netty?"

**Mid-level answer:** "Yes, blocking the event loop is always bad."

**Senior answer:** "Same rule, different blast-radius shape, and the difference matters
operationally.

Node has one loop per process, so blocking it stops 100% of traffic. It's catastrophic
and it's obvious — your p50 moves, your health check fails, someone pages.

Netty has N loops, typically one or two per core. Blocking one stops roughly 1/N of
connections. That's less catastrophic per event and **much harder to diagnose**,
because your p50 is fine, your CPU graph is fine, your error rate is zero, and the
symptom presents as intermittent slowness affecting some clients. I've seen that
misdiagnosed as a network problem more than once.

There's a second difference that catches people coming from Node: in Node, single-
threaded means data-race-free for free. In Netty, handler state is confined to one
loop *per channel*, so a non-`@Sharable` handler's fields are safe — but anything
shared across channels is touched by N threads on one heap, and the whole Java Memory
Model applies. A `@Sharable` handler with a plain `HashMap` field is a race, and the
Node instinct actively tells you it is safe.

Also worth saying: Node's answer for CPU work is `worker_threads` or `cluster`, which
give you separate heaps and message passing. Java's answer is another thread pool in
the same heap. That's more efficient and considerably easier to get wrong."

**What separates them:** "same rule, different shape" plus the diagnostic consequence,
and volunteering the shared-heap difference unprompted. The interviewer is testing
whether you transferred your Node model wholesale or calibrated it.

**Follow-up:** "How would you catch the `@Sharable` race in review?" Grep for
`@Sharable` and inspect every field; anything not `final` and immutable, or not a
concurrent structure, is a bug.

---

### Q5 — "How many event loop threads should this service have?"

**Mid-level answer:** "One per core, or leave the default."

**Senior answer:** "The default is `max(availableProcessors, 4)` for Reactor Netty and
`availableProcessors * 2` for raw Netty — but the important part is that
`availableProcessors()` in a container comes from the cgroup CPU quota, not the node's
core count. So on a pod limited to 500 millicores, the JVM reports 1, and Reactor
Netty's floor gives you 4 loops sharing half a core. On an unlimited pod on a 64-core
node, you get 64 loops, 64 selectors, 64 sets of thread-local buffer caches, and then
CFS throttling you didn't account for.

So my answer is: set it explicitly, record it next to the load-test baseline, and
derive it from measurement rather than a formula. The starting point is one loop per
core available to the container, assuming the loops are genuinely non-blocking — if
they aren't, the number is wrong for a different reason and adding loops is treating
the symptom.

I'd also separate the questions: loop count sizes CPU-bound I/O dispatch; the bounded
offload pool sizes blocking work and should be matched to the downstream capacity —
usually the connection pool, which is Little's Law and database capacity, not 'more is
better'. Those two numbers get tuned independently and against different constraints."

**What separates them:** container awareness, knowing both defaults including the
floor, insisting the number be recorded with the baseline, and separating loop sizing
from offload-pool sizing. Most candidates conflate the two.

**Follow-up:** "What would tell you the loop count is too low?" Throughput plateau with
loop threads at 100% CPU and no blocking in the stacks. If they are not at 100%, more
loops will not help and you have a different problem.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A channel is bound to one event loop for its lifetime. Derive from that single
   fact *both* why a loop stall produces bimodal latency **and** why a
   non-`@Sharable` handler's instance fields need no synchronisation. Then find the
   case where the second conclusion is false.

2. `epoll` keeps the interest set in the kernel; `select` passes it on every call.
   Both return ready sets. Why did that one difference change what was architecturally
   possible, rather than merely making things faster?

3. Node solves multi-core with `cluster` — N processes, N heaps. Netty solves it with
   N loops in one heap. Name one thing that is strictly better about each. Which
   would you rather debug at 3am, and does your answer change if the bug is a memory
   leak rather than a race?

4. Edge-triggered epoll tells you once; level-triggered tells you repeatedly. If you
   were designing a transport from scratch, which would you choose as the default, and
   what class of bug does each choice make more likely?

5. A `DirectByteBuffer` avoids a copy on every socket write, and costs you native
   memory the GC cannot see (Topic 80). Netty defaults to pooled direct buffers.
   Construct the strongest argument that this default is wrong for a small service, and
   then say what measurement would settle it.

6. Reactor Netty's worker-count floor is 4, even when `availableProcessors()` is 1.
   Why would anyone choose a floor rather than just using the processor count? Name a
   workload where the floor helps and one where it hurts.

7. You have proven with BlockHound that no JDK blocking primitive is called on a loop
   thread. Name two ways a loop can still be stalled that BlockHound will not catch,
   and describe how you would detect each.

---

## Quick reference card

### The layers

```
your handler          ChannelPipeline, ChannelHandler
   ^
Netty                 EventLoop (1 thread) <- EventLoopGroup
   ^
JDK                   Selector, SelectionKey, SelectableChannel
   ^
kernel                epoll_create1 / epoll_ctl / epoll_wait   (Linux)
                      kqueue / kevent                          (macOS, BSD)
                      wepoll, IOCP                             (Windows)
                      io_uring                                 (Linux, completion)
```

### Thread-name prefixes worth recognising instantly

| Prefix | What it is |
|---|---|
| `reactor-http-nio-N` | Reactor Netty I/O worker, JDK NIO transport |
| `reactor-http-epoll-N` | Reactor Netty I/O worker, native epoll transport |
| `nioEventLoopGroup-N-M` | Raw Netty event loop |
| `boundedElastic-N` | Reactor's blocking-work offload pool |
| `parallel-N` | Reactor's CPU-work scheduler |
| `http-nio-8080-exec-N` | Tomcat worker — **you are on MVC, not WebFlux** |
| `virtual-N` / `VirtualThread-...` | A virtual thread (Topic 101), not an event loop |

### Defaults (verify, do not quote)

| Setting | Default | Override |
|---|---|---|
| Reactor Netty I/O workers | `max(availableProcessors, 4)` | `-Dreactor.netty.ioWorkerCount` |
| Reactor Netty select count | small, often 1 | `-Dreactor.netty.ioSelectCount` |
| Netty event-loop threads | `max(1, availableProcessors * 2)` | `-Dio.netty.eventLoopThreads` |
| Netty's processor count | `Runtime.availableProcessors()` | `-Dio.netty.availableProcessors` |
| Netty leak detection | sampling ("simple") | `-Dio.netty.leakDetection.level=paranoid` |
| Netty direct memory limit | separate from `-XX:MaxDirectMemorySize` | `-Dio.netty.maxDirectMemory` |

### Diagnostic commands

```bash
jcmd -l                                                # find the pid
jcmd <pid> Thread.print | grep -o 'reactor-http-nio-[0-9]*' | sort -u
jcmd <pid> VM.native_memory summary                    # direct memory (needs NMT on)
jcmd <pid> VM.flags                                    # what the JVM actually decided

java -XX:StartFlightRecording=duration=120s,filename=loop.jfr,settings=profile ...
jfr print --events jdk.SocketRead,jdk.ThreadPark loop.jfr

sudo strace -f -p <pid> -e trace=epoll_wait,epoll_ctl -c

./mvnw dependency:tree | grep -i -E 'netty|reactor'    # what version am I on?
```

```java
Thread.currentThread().getName()                       // the primary instrument
java.nio.channels.spi.SelectorProvider.provider().getClass().getName()
Runtime.getRuntime().availableProcessors()
reactor.blockhound.BlockHound.install()                // in tests
PooledByteBufAllocator.DEFAULT.metric()
ctx.channel().isWritable()                             // transport-level backpressure
```

### The rules

```
NEVER   block an event-loop thread. Not JDBC, not RestTemplate, not .block(),
        not Thread.sleep, not synchronous DNS, not a contended synchronized block.
ALWAYS  offload blocking work to a BOUNDED executor sized against the downstream.
ALWAYS  set reactor.netty.ioWorkerCount explicitly in production and record it.
ALWAYS  release() a ByteBuf you consume; retain() one you pass on and keep.
ALWAYS  check isWritable() before a bulk write loop.
NEVER   put unsynchronised mutable state on a @Sharable handler.
NEVER   trust a mean latency in this phase. Percentiles, and look at the histogram.
```

---

## When would I use this at work?

**1. Diagnosing "it's slow for some customers, fine for others."**
Support escalates a report that one merchant sees multi-second latency while everyone
else is fine, and the network team finds nothing. You know that a keep-alive
connection is pinned to one event loop, so you check whether the slow merchants
cluster on one loop, print thread names on the affected path, and pull JFR
`jdk.SocketRead` durations filtered to loop threads. What looks like a network mystery
becomes "a repository call was added to a WebFlux handler in a PR three weeks ago."
This is the highest-value use of this topic and it is nearly unreachable without it.

**2. Sizing a pod and defending the number.**
Someone proposes `cpu: 500m` for a WebFlux service because "it's non-blocking so it
doesn't need CPU". You can explain that the CPU limit *is* the loop count — via
`availableProcessors()` and the cgroup quota — that four loops on half a core just
adds context switches, and that the loop count must be pinned explicitly and recorded
with the load-test baseline or the baseline is not reproducible. This is a capacity
conversation (Topic 129) that starts here.

**3. Reviewing a pull request that adds one repository call.**
A diff adds three lines to a WebFlux controller: inject a repository, call
`findById`, map the result. It looks like the smallest possible change. You know it
converts a non-blocking path into a loop-stalling one with a throughput ceiling of
`loops / queryTime`, and that no test in the suite will catch it. You ask for a
`subscribeOn` with a bounded scheduler, or for the endpoint to move to the MVC module
— and you ask for BlockHound in the test suite so the next person cannot do it
silently.

---

## Connected topics

**Prerequisites:**
- **80 — Off-heap memory:** Netty's pooled **direct** buffers are `DirectByteBuffer`s
  and native memory. Every OOMKill-with-healthy-heap story from that topic applies
  here, plus Netty's own separate accounting and leak detector.
- **82 — JVM tuning and container awareness:** `availableProcessors()` under a cgroup
  quota is what determines your loop count, arena count and buffer caches. This topic
  is where that abstract fact acquires a throughput number.
- **84 — Threads vs the Node event loop:** the model this topic makes concrete.
- **86 / 88 — The JMM and safe publication:** why `@Sharable` handler state is not
  free, and why "single-threaded per channel" does not mean "single-threaded".
- **90 — Executor pool sizing:** the offload pool you put behind the loop is an
  `ExecutorService` and every sizing rule from that topic applies.
- **92 — Concurrent collections:** the `HashMap`-in-a-`@Sharable`-handler trap, and
  why `putIfAbsent` is not the same as check-then-act.
- **93 — Bounded queues:** `isWritable()` and the write watermarks are a bounded queue
  wearing a transport-layer costume. Hold that thought for Topic 105.
- **101 — Virtual threads:** the *other* answer to the same problem. Topic 107 makes
  you choose.

**This unlocks:**
- **104 — Reactor `Mono`/`Flux`:** the programming model that runs on these loops.
  "Which thread executed this operator" is answered by this document.
- **105 — Backpressure:** `isWritable()` generalised into `request(n)` propagating
  upstream through an operator chain.
- **106 — WebFlux vs MVC:** the threading-model comparison, with the event loop on one
  side and Tomcat's `http-nio-8080-exec-*` pool on the other.
- **107 — Loom vs reactive:** the decision. The thread-economy argument for event loops
  is exactly what virtual threads attack.
- **108 — Debugging reactive:** why the stack trace shows the loop and not your code,
  and why `ThreadLocal` cannot survive a hop between these threads.
- **113–114 — Kafka:** a consumer is a push source with its own poll loop. Bridging it
  into a `Flux` without translating fetch into demand is Topic 105's OOM.
- **118 — Metrics:** which Reactor Netty gauges to export, and why a mean latency is
  the wrong metric for a system with per-loop blast radii.
- **119–120 — Tracing and MDC:** context cannot live in a `ThreadLocal` when the work
  hops between loop threads and offload pools.
- **129 — Capacity and latency budgets:** `loops / blockingSeconds` is a capacity
  model, and it belongs in the budget document.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0; Netty and
Reactor versions come from the Boot BOM and are deliberately not quoted here. Three
things in this document are hedged rather than asserted: Netty 4.2's exact event-loop
API surface (`MultiThreadIoEventLoopGroup` + `IoHandlerFactory` replaced the 4.1
constructors during the 4.2 transition), Netty's current default for
`EpollChannelOption.EPOLL_MODE`, and whether BlockHound installs on JDK 25 without
additional flags. Each has a command in the Hands-on section that settles it on your
machine in under a minute. Everything else here — the epoll model, channel-to-loop
affinity, the blast-radius shape, and the `loops / blockingSeconds` ceiling — has been
stable since Netty 4.0 and will still be true when you next debug it at 2am.*
