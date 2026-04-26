# Chapter 11: Concurrency — The GIL, Threading, and asyncio

## The Problem

Java developers assume threads work the way the JVM taught them — multiple threads, multiple CPU cores, genuine parallelism:

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> processChunk(chunk1));
pool.submit(() -> processChunk(chunk2));
pool.submit(() -> processChunk(chunk3));
pool.submit(() -> processChunk(chunk4));
// all four run simultaneously on different CPU cores
```

Python threads do not work this way. And understanding why — and what to use instead — is one of the most important things a Java developer needs to know before writing production Python.

---

## The GIL — Global Interpreter Lock

CPython (the standard Python implementation) has a Global Interpreter Lock — a mutex that allows only one thread to execute Python bytecode at a time.

```python
import threading

def process(data):
    result = expensive_computation(data)
    return result

t1 = threading.Thread(target=process, args=(chunk1,))
t2 = threading.Thread(target=process, args=(chunk2,))
t1.start()
t2.start()
# despite two threads — only one runs at any moment
```

They take turns. Not parallel. The GIL defeats the purpose of threading for CPU-bound work.

**Why does the GIL exist?**

Python's memory management uses reference counting — each object tracks how many references point to it. When the count reaches zero, the object is freed. This reference counting is not thread-safe. Rather than adding fine-grained locks to every object (complex, slow, error-prone), Python's creators added one global lock. Simple, effective, limiting.

Java's JVM manages memory differently — garbage collection with thread-safe write barriers — so the JVM has no GIL. Python's simpler memory model came with this cost.

---

## IO-Bound vs CPU-Bound — The Critical Distinction

The GIL only affects CPU-bound work. For IO-bound work, threads still help:

```python
import threading
import requests

def fetch(url):
    response = requests.get(url)    # waiting for network
    print(response.status_code)     # IO wait — GIL released

# these DO run concurrently — each releases GIL while waiting for network
t1 = threading.Thread(target=fetch, args=("https://api1.com",))
t2 = threading.Thread(target=fetch, args=("https://api2.com",))
t1.start()
t2.start()
```

While t1 waits for the network response, it releases the GIL. t2 picks it up and runs. The GIL passes back and forth during IO waits. Effective concurrency — just not parallelism.

| Work type | Example | GIL impact | Solution |
|---|---|---|---|
| CPU-bound | Image processing, ML training, encryption | GIL blocks parallelism | `multiprocessing` |
| IO-bound | API calls, database queries, file reads | GIL released during wait | `threading` or `asyncio` |

AI applications are almost entirely IO-bound — calling APIs, reading files, querying databases. The GIL rarely hurts you in practice.

---

## Threading — Familiar but Limited

Python's `threading` module works like Java's threading API:

```python
import threading

# basic thread
t = threading.Thread(target=some_function, args=(arg1, arg2))
t.start()
t.join()    # wait for completion

# thread pool
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(fetch, url) for url in urls]
    results = [f.result() for f in futures]
```

Java developers will recognise `ThreadPoolExecutor` immediately.

**Locks — same concept as Java's `synchronized`:**

```java
// Java
synchronized void increment() {
    counter++;
}
```

```python
# Python
lock = threading.Lock()

def increment():
    with lock:          # context manager — lock acquired and released automatically
        counter += 1
```

**When to use threading:**
- IO-bound work with existing synchronous libraries
- Integrating with C extensions that release the GIL
- When you need OS threads for specific reasons

**When not to use threading:**
- CPU-bound work (GIL prevents real parallelism)
- New async-compatible code (use `asyncio` instead)
- When simplicity matters (threads introduce race conditions)

---

## `multiprocessing` — Bypassing the GIL

For CPU-bound work, use separate processes — each has its own Python interpreter and its own GIL:

```python
from multiprocessing import Pool

def process_image(image_path):
    # CPU-heavy work — runs in its own process, true parallelism
    img = load_image(image_path)
    return transform(img)

with Pool(processes=4) as pool:
    results = pool.map(process_image, image_paths)
    # genuinely parallel — 4 CPU cores working simultaneously
```

Java equivalent: `ProcessBuilder` or fork/join. In Python, `multiprocessing` is the standard solution for true CPU parallelism.

**The cost:** processes don't share memory. Communication between processes requires serialisation (pickling in Python). More overhead than threads, but genuine parallelism.

---

## `asyncio` — The Modern Solution for IO-Bound Work

Threading has problems even for IO-bound work:
- Threads are expensive — each takes ~8MB of memory
- Race conditions — concurrent access to shared state is hard to reason about
- Debugging is difficult — any thread can run at any time

`asyncio` solves this differently: **one thread, cooperative multitasking**.

```python
import asyncio

async def fetch(url):
    await asyncio.sleep(1)    # simulating network wait
    return f"response from {url}"

async def main():
    # run all three concurrently — on one thread
    results = await asyncio.gather(
        fetch("https://api1.com"),
        fetch("https://api2.com"),
        fetch("https://api3.com"),
    )
    print(results)

asyncio.run(main())
# completes in ~1 second — not 3
```

Three network calls, one thread, ~1 second. No GIL issues. No race conditions (single thread). No thread overhead.

---

## `async` and `await` — The Syntax

`async def` defines a coroutine — a function that can pause:

```python
async def fetch_dog(dog_id: int) -> Dog:
    response = await db.get(dog_id)    # pause here — let other coroutines run
    return Dog(**response)
```

`await` pauses the coroutine and yields control to the event loop. The event loop runs other coroutines while this one waits for its IO to complete. When the IO finishes, the event loop resumes this coroutine from where it paused.

This is the same pause/resume mechanism as generators (Chapter 9) — `await` is essentially `yield` for async code. Python's event loop is built on generators internally.

**Rules:**
- `async def` functions must be called with `await` — or scheduled on the event loop
- `await` can only be used inside `async def` functions
- Synchronous code blocks the event loop — never do blocking work without `await`

```python
async def bad():
    time.sleep(1)         # ❌ blocks the entire event loop — no other coroutines run

async def good():
    await asyncio.sleep(1)    # ✅ pauses this coroutine — others can run
```

---

## Concurrency Primitives in asyncio

```python
# gather — run multiple coroutines concurrently
results = await asyncio.gather(
    fetch("url1"),
    fetch("url2"),
    fetch("url3"),
)

# wait with timeout
try:
    result = await asyncio.wait_for(fetch("url"), timeout=5.0)
except asyncio.TimeoutError:
    print("request timed out")

# semaphore — limit concurrent operations
semaphore = asyncio.Semaphore(10)    # max 10 concurrent

async def limited_fetch(url):
    async with semaphore:
        return await fetch(url)

# queue — producer/consumer pattern
queue = asyncio.Queue()

async def producer():
    for item in items:
        await queue.put(item)

async def consumer():
    while True:
        item = await queue.get()
        await process(item)
```

Java developers will recognise `Semaphore` and producer/consumer — same concepts, async-native syntax.

---

## asyncio With the Anthropic SDK

For AI applications, concurrent API calls dramatically improve throughput:

```python
import asyncio
import anthropic

client = anthropic.AsyncAnthropic()    # async client

async def ask_claude(prompt: str) -> str:
    response = await client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

async def main():
    prompts = [
        "Explain decorators in Python",
        "Explain generators in Python",
        "Explain the GIL in Python",
        "Explain metaclasses in Python",
    ]

    # all four API calls run concurrently
    responses = await asyncio.gather(*[ask_claude(p) for p in prompts])

    for prompt, response in zip(prompts, responses):
        print(f"Q: {prompt}")
        print(f"A: {response[:100]}...")
        print()

asyncio.run(main())
# ~1 second instead of ~4 seconds sequential
```

Without `asyncio`: 4 API calls × ~1 second each = ~4 seconds.
With `asyncio`: all 4 concurrent = ~1 second.

---

## Async Streaming

Generators and asyncio combine for streaming responses:

```python
import asyncio
import anthropic

client = anthropic.AsyncAnthropic()

async def stream_response(prompt: str):
    async with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    ) as stream:
        async for text in stream.text_stream:    # async iterator
            yield text                            # async generator

async def main():
    async for chunk in stream_response("explain asyncio"):
        print(chunk, end="", flush=True)

asyncio.run(main())
```

`async for` is the async equivalent of `for` — iterates over an async iterator, awaiting each item. The same iterator protocol from Chapter 9, made async.

---

## Mixing Sync and Async — The Common Pitfall

You can't call async functions from synchronous code directly:

```python
async def fetch():
    return await api.get()

# WRONG — can't await outside async context
result = fetch()    # returns a coroutine object, not the result
```

Solutions:

```python
# Run from the top level
asyncio.run(fetch())    # ✅ — creates event loop, runs coroutine

# Run sync function in a thread from async context
async def main():
    result = await asyncio.to_thread(blocking_function, arg)    # ✅

# Run async from sync using a new event loop (rarely needed)
loop = asyncio.new_event_loop()
result = loop.run_until_complete(fetch())
```

The cleanest approach: make your entire application async from the top level down. Don't mix sync and async unnecessarily.

---

## Choosing the Right Tool

```
Is the work CPU-bound?
    Yes → multiprocessing (bypass the GIL)
    No  → Is it IO-bound?
              Yes → Is it new code?
                        Yes → asyncio (most efficient, no race conditions)
                        No  → threading (simpler for existing sync libraries)
              No  → No concurrency needed
```

For AI applications specifically:

| Task | Tool |
|---|---|
| Multiple concurrent API calls | `asyncio` + `asyncio.gather` |
| Streaming API response | Async generator + `async for` |
| CPU-heavy preprocessing | `multiprocessing` |
| Existing sync library | `threading` or `asyncio.to_thread` |

---

## Java Comparison

| Java | Python | Notes |
|---|---|---|
| `Thread` | `threading.Thread` | Both exist, Python limited by GIL |
| `ExecutorService` | `ThreadPoolExecutor` | Similar API |
| `synchronized` | `with threading.Lock():` | Context manager syntax |
| `CompletableFuture` | `asyncio.gather` | Concurrent async operations |
| `ForkJoinPool` | `multiprocessing.Pool` | True parallelism |
| No equivalent | GIL | Python-specific constraint |
| `Callable<T>` | `async def` + `await` | Different execution model |

The fundamental difference: Java's concurrency is thread-based throughout. Python's modern concurrency is event-loop-based (`asyncio`) with threads and processes as escape hatches for specific cases.

---

## The Mental Model

> *The GIL means Python threads can't run Python code in parallel — they take turns. For IO-bound work (API calls, database queries, file reads) this doesn't matter: threads release the GIL while waiting. For CPU-bound work, use `multiprocessing` for genuine parallelism. For new IO-bound code, use `asyncio` — one thread, cooperative multitasking, no race conditions, dramatically more efficient than threads for high-concurrency workloads.*

The GIL is Python's biggest surprise for Java developers. Once you understand it, the rest follows naturally.

---

*Next: Chapter 12 — Descriptors: The Engine Behind @property*
