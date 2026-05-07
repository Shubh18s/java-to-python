# Chapter 8: Context Managers — Guaranteed Cleanup

## The Problem

Resource management is one of the oldest problems in programming. Open a file, read it, close it. Open a database connection, run a query, close it. Acquire a lock, do work, release it.

The problem: what if something crashes in the middle?

```python
f = open("data.txt")
data = f.read()         # what if this raises an exception?
f.close()               # never runs — file stays open forever
```

Java solved this with try-finally, then improved it with try-with-resources:

```java
// Java try-with-resources
try (Connection conn = getConnection()) {
    conn.execute("SELECT * FROM dogs");
}  // conn.close() called automatically — even if exception thrown
```

Python has the same solution — the `with` statement — built on a protocol that any object can implement.

---

## The `with` Statement

```python
with open("data.txt") as f:
    data = f.read()
# f.close() called automatically here — even if read() raised an exception
```

Direct equivalent of Java's try-with-resources. `open()` already implements the context manager protocol — you get cleanup for free.

The `with` block guarantees:
- Setup runs before the block
- Cleanup runs after the block — always, even if an exception occurs

---

## The Protocol — `__enter__` and `__exit__`

Any object can be a context manager by implementing two methods:

```python
class ManagedFile:
    def __init__(self, filename):
        self.filename = filename

    def __enter__(self):                              # runs at start of with block
        self.f = open(self.filename)
        return self.f                                 # what "as f" receives

    def __exit__(self, exc_type, exc_val, exc_tb):   # runs at end — always
        self.f.close()
        return False                                  # don't suppress exceptions

with ManagedFile("data.txt") as f:
    data = f.read()
```

`__enter__` = setup. Returns the resource.
`__exit__` = cleanup. Always runs. Receives exception info if something went wrong.

---

## `__exit__` and Exception Handling

The three parameters in `__exit__` tell you what happened inside the `with` block:

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    self.conn.close()               # cleanup always happens first

    if exc_type is None:
        # no exception — everything worked
        pass
    else:
        # exc_type: the exception class (e.g. ValueError)
        # exc_val: the exception instance
        # exc_tb: the traceback
        print(f"Something went wrong: {exc_val}")

    return False    # False = let the exception propagate (almost always right)
                    # True  = suppress the exception (use with extreme caution)
```

Returning `False` (or `None`) lets the exception continue bubbling up — you clean up but don't hide the error. Returning `True` swallows the exception silently — only appropriate when you're explicitly handling it.

---

## `@contextmanager` — Writing Context Managers Without Classes

Writing a full class just for `__enter__` and `__exit__` is verbose for simple cases. `contextlib.contextmanager` lets you write context managers as generator functions:

```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename):
    f = open(filename)
    try:
        yield f              # everything before yield = __enter__
                             # f is what "as f" receives
    finally:
        f.close()            # everything after yield = __exit__

with managed_file("data.txt") as f:
    data = f.read()
```

`yield` is the split point. Before it: setup. After it: cleanup. The `finally` ensures cleanup runs even if an exception occurs inside the `with` block.

This uses Python's generator mechanism (Chapter 9) — `yield` pauses the function, the `with` block runs, then the function resumes after `yield`. Same pause/resume mechanism, applied to resource management.

---

## Beyond Cleanup — All the Use Cases

Context managers are not just about closing resources. Any time you have *"do X, then always undo X"* — that's a context manager.

**Temporarily changing state:**

```python
@contextmanager
def temp_directory(path):
    original = os.getcwd()
    os.chdir(path)              # change directory
    try:
        yield
    finally:
        os.chdir(original)      # always restore

with temp_directory("/tmp"):
    process_files()
# back to original directory — even if process_files() crashed
```

**Timing a block of code:**

```python
@contextmanager
def timer(label):
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f"{label}: {elapsed:.3f}s")

with timer("database query"):
    results = db.execute("SELECT * FROM dogs")
# database query: 0.043s
```

**Thread locks:**

```python
import threading
lock = threading.Lock()

with lock:
    shared_counter += 1    # only one thread at a time
# lock released automatically
```

Java equivalent: `synchronized` blocks. Python's `with` is the same idea, more explicit.

**Database transactions:**

```python
@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()           # success — save everything
    except Exception:
        conn.rollback()         # failure — undo everything
        raise

with transaction(db) as conn:
    conn.execute("INSERT INTO orders ...")
    conn.execute("UPDATE inventory ...")
    conn.execute("INSERT INTO payments ...")
# either ALL three happen or NONE do
```

**Mocking in tests:**

```python
from unittest.mock import patch

def test_payment():
    with patch("myapp.stripe.charge") as mock_charge:
        mock_charge.return_value = {"status": "success"}
        result = process_payment(100)
        assert result == "success"
# real stripe.charge restored after the test — no actual charges made
```

---

## `contextlib` — What You'll Actually Use

Writing full classes or even `@contextmanager` functions for every case gets verbose. `contextlib` provides ready-made tools:

**`contextlib.suppress` — silence specific exceptions:**

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("temp.tmp")    # if file doesn't exist, silently continue
```

Use when the failure is genuinely expected and harmless. Never use to hide errors that matter.

**`contextlib.nullcontext` — optional context manager:**

```python
from contextlib import nullcontext

def process(data, conn=None):
    with conn or nullcontext():    # use conn if provided, otherwise no-op
        do_work(data)
```

**`contextlib.contextmanager` — the workhorse:**

Covered above. Use for 80% of custom context managers.

---

## When to Use a Full Class vs `@contextmanager`

| Situation | Use |
|---|---|
| Simple setup/teardown | `@contextmanager` |
| Suppress known exceptions | `contextlib.suppress` |
| Context manager needs its own methods | Full class |
| Complex exception handling (multiple types) | Full class |
| Everything else | `@contextmanager` |

Full class example — the context manager needs extra methods:

```python
class DatabasePool:
    def __enter__(self):
        self.connections = [connect() for _ in range(self.size)]
        return self

    def get_connection(self):         # extra method — not possible with @contextmanager
        return self.connections.pop()

    def __exit__(self, *args):
        for conn in self.connections:
            conn.close()
```

---

## Context Managers vs Decorators

Both wrap behaviour around something. The difference is the target:

| | Decorator | Context Manager |
|---|---|---|
| Wraps | A whole function | A specific block of code |
| Granularity | All or nothing | Surgical |
| Best for | Reusable behaviour on functions | Guaranteed paired actions on blocks |

The granularity difference matters:

```python
def process_checkout(cart, payment, db):
    validate_cart(cart)           # not part of transaction
    charge_card(payment)          # not part of transaction

    with transaction(db):         # only these need to be transactional
        db.execute("INSERT INTO orders ...")
        db.execute("UPDATE inventory ...")

    send_confirmation_email()     # not part of transaction
```

A decorator on `process_checkout` would wrap everything. The context manager wraps only the database writes. That surgical precision is why context managers exist alongside decorators — not instead of them.

---

## Tracking Token Usage — A Real AI App Example

```python
from contextlib import contextmanager
import anthropic

@contextmanager
def track_tokens(client):
    usage = {"input": 0, "output": 0}

    def call(prompt):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}]
        )
        usage["input"] += response.usage.input_tokens
        usage["output"] += response.usage.output_tokens
        return response.content[0].text

    yield call

    total = usage["input"] + usage["output"]
    print(f"\n--- Token Summary ---")
    print(f"Input:  {usage['input']}")
    print(f"Output: {usage['output']}")
    print(f"Total:  {total}")

client = anthropic.Anthropic()

with track_tokens(client) as call:
    response1 = call("What is a metaclass?")
    response2 = call("What is a decorator?")
    response3 = call("What is a context manager?")
# --- Token Summary ---
# Input:  847
# Output: 1203
# Total:  2050
```

The context manager handles the token tracking setup and summary automatically. The caller just makes API calls. Cleanup — printing the summary — is guaranteed regardless of whether any call failed.

---

## Java Comparison

| Java | Python |
|---|---|
| `try (Resource r = ...) {}` | `with Resource() as r:` |
| `try { } finally { }` | `__exit__` always called |
| `AutoCloseable` interface | Context manager protocol (`__enter__`/`__exit__`) |
| `synchronized` block | `with threading.Lock():` |
| No direct equivalent | `@contextmanager` for custom ones |
| No direct equivalent | `contextlib.suppress` |

Python's context manager protocol is more flexible than Java's `AutoCloseable` — it handles not just closing but any paired setup/teardown, including exception-dependent behaviour like commit/rollback.

---

## The Mental Model

> *A context manager guarantees that paired actions happen — setup when you enter a block, cleanup when you leave, always, even if something crashes. Decorators wrap functions; context managers wrap blocks of code. Both are the same sandwich pattern applied to different targets.*

---

*Next: [Chapter 9 — Generators and Iterators: Laziness as a Superpower](chapter-9-generators-iterators.md)*
