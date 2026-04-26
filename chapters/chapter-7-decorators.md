# Chapter 7: Decorators — Python's Answer to AOP

## The Problem

In Java, adding cross-cutting behaviour — logging, authentication, timing, caching — to multiple methods requires:

- AOP frameworks like AspectJ with XML configuration
- Inheritance hierarchies just to share behaviour
- Manual repetition of boilerplate across every method

```java
// Java — manual repetition
public Dog createDog(String name, int age) {
    logger.info("Calling createDog");          // repeated
    long start = System.currentTimeMillis();   // repeated
    Dog result = new Dog(name, age);
    logger.info("Done in " + (System.currentTimeMillis() - start) + "ms");  // repeated
    return result;
}

public void deleteDog(int id) {
    logger.info("Calling deleteDog");          // repeated
    long start = System.currentTimeMillis();   // repeated
    db.delete(id);
    logger.info("Done in " + (System.currentTimeMillis() - start) + "ms");  // repeated
}
```

Change the logging format and you update dozens of methods.

Python solves this with decorators — a clean, reusable mechanism that wraps behaviour around functions without touching the function itself.

---

## The Foundation — Functions Are Objects

Decorators only work because functions are objects in Python (Chapter 2). You can pass them to other functions, return them, store them:

```python
def greet(name):
    print(f"Hello {name}")

# assign to variable
say_hello = greet
say_hello("Rex")         # Hello Rex

# pass to a function
def call_it(func, value):
    func(value)

call_it(greet, "Rex")    # Hello Rex
```

Once you understand this, decorators are just a natural consequence.

---

## What a Decorator Does

One sentence: **it runs code before and/or after your function.**

```python
def log(func):
    def wrapper(*args, **kwargs):
        print("BEFORE")
        result = func(*args, **kwargs)
        print("AFTER")
        return result
    return wrapper

@log
def greet(name):
    print(f"Hello {name}")

greet("Rex")
# BEFORE
# Hello Rex
# AFTER
```

`@log` is literally `greet = log(greet)`. Python passes the function object to `log`, which returns `wrapper`. From that point, `greet` is `wrapper`.

Every decorator you will ever see follows this shape:

```
your function called
    ↓
wrapper runs
    ↓
BEFORE logic
    ↓
original function runs  ← untouched
    ↓
AFTER logic
    ↓
result returned
```

The variations are only in what happens before and after. The mechanism never changes.

---

## Closures — How the Wrapper Remembers `func`

`wrapper` references `func` — but `func` is defined in `log`'s scope, not `wrapper`'s. How does `wrapper` still have access to it after `log` returns?

This is a **closure** — a function that captures variables from its enclosing scope:

```python
def outer(message):
    def inner():
        print(message)    # inner captures message from outer's scope
    return inner

fn = outer("Hello Rex")
fn()                      # Hello Rex — message still accessible
```

`inner` *closed over* `message`. Even after `outer` has returned, `message` is preserved inside `inner`.

In a decorator, `wrapper` closes over `func`. That's how the wrapper always has access to the original function.

Java achieves similar things with lambdas capturing effectively-final variables — but closures in Python are more natural and first-class.

---

## `*args` and `**kwargs` in the Wrapper

The wrapper uses `*args, **kwargs` because it doesn't know what arguments the wrapped function expects:

```python
def log(func):
    def wrapper(*args, **kwargs):    # catches everything
        print("before")
        result = func(*args, **kwargs)   # passes everything through
        return result
    return wrapper

@log
def greet(name): ...           # one argument

@log
def add(x, y): ...             # two arguments

@log
def connect(host, port=8080):  # one positional, one keyword
    ...
```

The same decorator works on all three. `*args` collects positional arguments into a tuple; `**kwargs` collects keyword arguments into a dict. Together they're a transparent pass-through for any function signature.

---

## `functools.wraps` — Preserving Identity

Without it, the wrapped function loses its identity:

```python
@log
def greet(name):
    """Greets a person by name"""
    print(f"Hello {name}")

print(greet.__name__)    # wrapper ❌ — lost
print(greet.__doc__)     # None    ❌ — lost
```

Stack traces say `wrapper`. Documentation tools show nothing. Other decorators that inspect `__name__` break silently.

Fix with `functools.wraps`:

```python
from functools import wraps

def log(func):
    @wraps(func)               # copies name, docstring, and more onto wrapper
    def wrapper(*args, **kwargs):
        print("before")
        result = func(*args, **kwargs)
        return result
    return wrapper

@log
def greet(name):
    """Greets a person by name"""
    print(f"Hello {name}")

print(greet.__name__)    # greet                    ✅
print(greet.__doc__)     # Greets a person by name  ✅
```

Always use `@wraps` in production decorators. It's a decorator decorating your wrapper — the same pattern, one level deeper.

---

## Real Use Cases

**1. Timing:**

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_query():
    time.sleep(1)

slow_query()
# slow_query took 1.0012s
```

**2. Authentication — used in every web framework:**

```python
def login_required(func):
    @wraps(func)
    def wrapper(request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("/login")
        return func(request, *args, **kwargs)
    return wrapper

@login_required
def view_profile(request):
    return render("profile.html")

@login_required
def edit_settings(request):
    return render("settings.html")
```

Without the decorator, every view function manually checks authentication. Miss one and you have a security hole.

**3. Retry logic:**

```python
def retry(times=3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed, retrying...")
        return wrapper
    return decorator

@retry(times=3)
def call_payment_api(amount):
    response = requests.post("/pay", json={"amount": amount})
    return response
```

**4. Caching — built into Python's stdlib:**

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_user(user_id):
    return db.find(user_id)    # expensive DB call

get_user(42)    # hits database
get_user(42)    # returns cached result instantly
get_user(42)    # cached again
```

`lru_cache` is just a decorator someone else wrote. The same mechanism you've learned, applied to caching.

---

## Decorators With Arguments — One Extra Layer

`@retry` (no arguments) vs `@retry(times=3)` (with arguments) — the structure changes slightly.

**Without arguments — two layers:**

```python
def decorator(func):          # receives the function
    def wrapper(*args, **kwargs):
        ...
    return wrapper
```

**With arguments — three layers:**

```python
def retry(times=3):           # receives the argument
    def decorator(func):      # receives the function
        def wrapper(*args, **kwargs):   # runs on each call
            ...
        return wrapper
    return decorator
```

The `@retry(times=3)` calls `retry(times=3)` first, gets back `decorator`, then applies `decorator` to the function. The `()` is the tell — if parentheses are present, there's an extra layer.

---

## Class Decorators

Decorators work on classes too:

```python
def singleton(cls):
    instances = {}
    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class DatabaseConnection:
    def __init__(self):
        print("Connecting to database...")

db1 = DatabaseConnection()    # Connecting to database...
db2 = DatabaseConnection()    # nothing — returns same instance
db1 is db2                    # True
```

Singleton pattern in Java is a whole ceremony. Here it's one reusable decorator applied to any class.

---

## Identifying a Decorator in the Wild

Two signals that something is a decorator:

**Signal 1 — a function that takes a function and returns a function:**

```python
def something(func):      # receives a function
    def wrapper(...):
        func(...)         # calls the original
    return wrapper        # returns a function
```

**Signal 2 — a nested function where the outer receives a callable:**

If you see a function inside a function, ask: does the outer receive a callable and return one? If yes — decorator.

If the outer just returns the inner without receiving a callable — it's a closure, not a decorator.

---

## Where You'll See This

```python
@app.route("/dogs")        # Flask — registers URL handler
@login_required            # Django — authentication
@pytest.fixture            # pytest — test setup
@dataclass                 # stdlib — generates __init__, __repr__
@property                  # stdlib — attribute control
@classmethod               # you already know this
@staticmethod              # you already know this
@lru_cache                 # stdlib — caching
@retry(times=3)            # custom — retry logic
@timer                     # custom — performance monitoring
```

All of these are the same pattern. A function (or class) that wraps another function or class. The mechanism never changes — only what happens before and after.

---

## Java Comparison

| Java approach | Python equivalent | Difference |
|---|---|---|
| AOP / AspectJ | Decorator | No framework needed — plain Python |
| Annotations + reflection | Decorator | Runtime, not compile time |
| Utility class methods called manually | Decorator | Applied once, runs automatically |
| Subclassing to add behaviour | Decorator | No class hierarchy required |

Python decorators do in 10 lines what Java AOP does with a framework, XML configuration, and a build plugin.

---

## The Mental Model

> *A decorator is a function that sandwiches behaviour around another function. Before and after, every call. Write once, apply anywhere with `@`. The original function is untouched — the wrapper is what runs.*

---

*Next: Chapter 8 — Context Managers: Guaranteed Cleanup*
