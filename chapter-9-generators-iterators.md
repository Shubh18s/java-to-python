# Chapter 9: Generators and Iterators — Laziness as a Superpower

## The Problem

In Java, iterating over a collection means the collection exists in memory:

```java
List<Dog> dogs = db.getAllDogs();    // loads ALL dogs into memory
for (Dog dog : dogs) {
    process(dog);
}
```

If you have a million dogs, you load a million dogs. Java has `Iterator` and `Stream` to handle this — but they require explicit setup.

Python builds lazy evaluation into the language itself. The `for` loop, list comprehensions, `zip`, `map`, `filter` — all of them are lazy by default. Understanding how they work under the hood explains a lot of Python's behaviour that seems surprising at first.

---

## What Actually Happens in a `for` Loop

Python's `for` loop doesn't use indexes. It uses a protocol:

```python
for dog in dogs:
    print(dog)
```

Python internally does this:

```python
iterator = iter(dogs)         # call __iter__ on dogs — get an iterator
while True:
    try:
        dog = next(iterator)  # call __next__ — get next item
        print(dog)
    except StopIteration:     # iterator says "I'm done"
        break
```

Two separate concepts:

**Iterable** — anything with `__iter__`. Returns an iterator when asked.
**Iterator** — anything with `__next__`. Returns items one at a time, raises `StopIteration` when done.

Lists, dicts, strings, files — all iterables. The `for` loop works on anything that implements this protocol.

---

## Writing an Iterator From Scratch

```python
class CountUp:
    def __init__(self, limit):
        self.limit = limit
        self.current = 0

    def __iter__(self):
        return self              # I am my own iterator

    def __next__(self):
        if self.current >= self.limit:
            raise StopIteration
        self.current += 1
        return self.current

for n in CountUp(3):
    print(n)
# 1
# 2
# 3
```

Java equivalent — implementing `Iterable<Integer>` and `Iterator<Integer>`. Same concept, similar verbosity. Python and Java both require the protocol — Python just calls the methods `__iter__` and `__next__` instead of `iterator()` and `next()`.

The problem: managing `self.current` manually is tedious. And every iterator needs this boilerplate. Python's solution is generators.

---

## Generators — Automatic Iterators

A generator function uses `yield` instead of `return`:

```python
def count_up(limit):
    for i in range(1, limit + 1):
        yield i               # pause here, return i, resume next time

for n in count_up(3):
    print(n)
# 1
# 2
# 3
```

Python automatically implements `__iter__` and `__next__` for generator functions. The state — where you paused — is managed automatically. No manual counter, no `StopIteration` to raise explicitly (it happens when the function ends).

This is `yield`'s core behaviour: pause the function, return a value, resume from that exact point next time.

---

## `yield` Has No Java Equivalent

A normal function runs to completion and returns once:

```python
def normal():
    return 1
    return 2    # never reached — function is done after first return
```

A generator pauses at each `yield` and resumes:

```python
def generator():
    print("start")
    yield 1           # pause here — return 1
    print("after 1")
    yield 2           # pause here — return 2
    print("after 2")
    yield 3           # pause here — return 3
    print("done")

g = generator()
next(g)    # prints "start", returns 1
next(g)    # prints "after 1", returns 2
next(g)    # prints "after 2", returns 3
next(g)    # prints "done", raises StopIteration
```

The function remembers its position and all local variables between calls. This is not possible in Java without external state management.

Java 21 introduced virtual threads and Project Loom — but `yield`-style generators are still not native Java. Python has had them since 2.2 (2001).

---

## Generators Are Lazy

Calling a generator function doesn't run it:

```python
def count_up(limit):
    print("starting")
    for i in range(1, limit + 1):
        yield i

g = count_up(3)              # nothing printed — function hasn't run yet
print("generator created")   # this prints first

next(g)                      # NOW "starting" prints, returns 1
next(g)                      # returns 2
```

Output:
```
generator created
starting
```

The function body doesn't execute until you ask for values. This laziness means:

- **Memory efficient** — values produced one at a time, not all at once
- **Can represent infinite sequences** — no need to store them
- **Can represent expensive operations** — only pay the cost when you consume the value

---

## Infinite Generators

```python
def integers_from(n):
    while True:              # infinite loop — fine in a generator
        yield n
        n += 1

gen = integers_from(0)
next(gen)    # 0
next(gen)    # 1
next(gen)    # 2
# ... forever
```

You can't store an infinite list. You can have an infinite generator. Take values as needed:

```python
import itertools

# first 5 even numbers — without generating all even numbers first
evens = (n for n in integers_from(0) if n % 2 == 0)
list(itertools.islice(evens, 5))    # [0, 2, 4, 6, 8]
```

---

## Generator Expressions — Lazy List Comprehensions

Python has two syntaxes for working with sequences:

**List comprehension — eager, stores everything:**

```python
squares = [x**2 for x in range(1000000)]    # creates list of 1M items immediately
```

**Generator expression — lazy, one at a time:**

```python
squares = (x**2 for x in range(1000000))    # creates generator — nothing computed yet
```

Note: `[]` vs `()`. The generator version uses almost no memory — it computes each square only when asked.

```python
# processing a million items without loading them all
total = sum(x**2 for x in range(1000000))    # generator expression inside sum
```

---

## `yield from` — Delegating to Another Generator

```python
def first_three():
    yield 1
    yield 2
    yield 3

def combined():
    yield from first_three()    # delegate to first_three
    yield from range(4, 7)      # delegate to range

list(combined())    # [1, 2, 3, 4, 5, 6]
```

`yield from` flattens nested generators — useful for building generators out of smaller ones.

---

## Generators Underpin `asyncio`

This is the connection that matters for AI applications.

Python's `async`/`await` syntax is built on the same pause/resume mechanism as generators:

```python
# generator — pauses at yield
def fetch_data():
    data = slow_operation()
    yield data

# coroutine — pauses at await
async def fetch_data():
    data = await slow_api_call()    # pause here, let other things run
    return data
```

Both pause execution and resume later. The difference: generators pause to produce values; coroutines pause to wait for I/O. Internally, Python's event loop (the thing that runs `asyncio` code) is built on generators.

Understanding generators is prerequisite knowledge for understanding async Python — covered fully in Chapter 11.

---

## Streaming API Responses

The most immediate use case for AI applications:

```python
import anthropic

client = anthropic.Anthropic()

def stream_response(prompt):
    with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    ) as stream:
        for chunk in stream.text_stream:
            yield chunk              # yield each chunk as it arrives

# display response as it streams
for text in stream_response("explain generators"):
    print(text, end="", flush=True)
```

Without generators, you wait for the entire response before displaying anything. With generators, each chunk is yielded and displayed as it arrives. That's the streaming UI behaviour in every AI chat interface.

---

## Built-In Functions That Use the Iterator Protocol

Python's built-ins all work lazily on iterables:

```python
dogs = [Dog("Rex", 3), Dog("Max", 5), Dog("Luna", 2)]

# map — applies function, returns iterator (lazy)
names = map(lambda d: d.name, dogs)

# filter — filters items, returns iterator (lazy)
old_dogs = filter(lambda d: d.age > 3, dogs)

# zip — pairs up iterables, returns iterator (lazy)
paired = zip(dogs, range(len(dogs)))

# all lazy — nothing computed until consumed
list(names)       # ["Rex", "Max", "Luna"]
list(old_dogs)    # [Dog("Max", 5)]
```

Java's `Stream` API is the equivalent — but in Python laziness is the default for these operations, not an opt-in.

---

## `itertools` — Composable Lazy Operations

```python
import itertools

# chain — concatenate iterables without loading both
combined = itertools.chain(range(5), range(10, 15))

# islice — take first N items from any iterable
first_five = itertools.islice(some_generator, 5)

# takewhile — take items while condition is true
small = itertools.takewhile(lambda x: x < 10, integers_from(0))

# batched (Python 3.12+) — process in chunks
for batch in itertools.batched(dogs, 100):
    process_batch(batch)
```

These compose with any iterable — lists, generators, files, API responses. The same lazy protocol works throughout.

---

## Java Comparison

| Java | Python | Notes |
|---|---|---|
| `Iterable<T>` | `__iter__` | Same concept |
| `Iterator<T>` | `__next__` + `StopIteration` | Same concept |
| `Stream<T>` | Generator expression | Python default, Java opt-in |
| No equivalent | `yield` | Pause/resume function execution |
| `Stream.generate()` | Infinite generator | Python simpler |
| No equivalent | `yield from` | Generator delegation |
| `Stream.map()` | `map()` built-in | Both lazy |
| `Stream.filter()` | `filter()` built-in | Both lazy |

---

## The Mental Model

> *Python's iteration protocol separates "things that can be iterated" (iterables) from "things that do the iterating" (iterators). Generators are functions that automatically become iterators — pausing at each `yield` and resuming when asked for the next value. Laziness is the default: nothing is computed until consumed.*

`yield` is the concept with no Java equivalent. Once it clicks, generators feel natural — and async Python (which uses the same mechanism) becomes much easier to understand.

---

*Next: Chapter 10 — Dataclasses: Goodbye Boilerplate*
