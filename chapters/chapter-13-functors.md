# Chapter 13: Functors and Callable Objects

## The Problem

In Java, passing behaviour around requires interfaces:

```java
// Java 7 — verbose
Comparator<Dog> byAge = new Comparator<Dog>() {
    @Override
    public int compare(Dog a, Dog b) {
        return Integer.compare(a.getAge(), b.getAge());
    }
};

// Java 8 — lambda syntax
Comparator<Dog> byAge = (a, b) -> Integer.compare(a.getAge(), b.getAge());

// Java functional interfaces
Runnable r = () -> System.out.println("running");
Callable<String> c = () -> "result";
Predicate<Dog> isOld = dog -> dog.getAge() > 5;
```

Java 8's lambdas are a step forward — but they're still tied to specific functional interfaces (`Runnable`, `Callable`, `Predicate`, `Function`, etc.). The type system enforces which interface each lambda implements.

Python has no concept of functional interfaces. Any object that implements `__call__` is callable — and callable objects can carry state, configuration, and behaviour together in a way that pure lambdas can't.

---

## `__call__` — Making Objects Callable

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor    # state

    def __call__(self, x):      # behaviour
        return x * self.factor

double = Multiplier(2)
triple = Multiplier(3)

double(5)    # 10 — called like a function
triple(5)    # 15

# it's still an object
double.factor    # 2
```

`double` looks and behaves like a function — but it's an object with state. This is a functor: a callable object that carries state with it.

Java's closest equivalent: a lambda capturing variables, or a class implementing a `@FunctionalInterface`. Python makes this a natural feature of any class.

---

## Why Functors Over Plain Functions?

Plain functions can't maintain state between calls (without globals or closures):

```python
# function — stateless
def multiply_by_2(x):
    return x * 2

# closure — stateful but limited
def multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply

double = multiplier(2)
double(5)    # 10
```

The closure works — but it can't be inspected, serialised, or extended. A callable class can:

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, x):
        return x * self.factor

    def __repr__(self):
        return f"Multiplier(factor={self.factor})"

    def compose(self, other):
        """Create a new multiplier that applies both."""
        return Multiplier(self.factor * other.factor)

double = Multiplier(2)
triple = Multiplier(3)
sextuple = double.compose(triple)    # 2 * 3 = 6

sextuple(5)    # 30
repr(double)   # "Multiplier(factor=2)" — inspectable
```

---

## Real Use Cases

**1. Configurable predicates:**

```python
class AgeFilter:
    def __init__(self, min_age, max_age):
        self.min_age = min_age
        self.max_age = max_age

    def __call__(self, dog):
        return self.min_age <= dog.age <= self.max_age

puppies = AgeFilter(0, 1)
adults = AgeFilter(2, 8)
seniors = AgeFilter(9, 20)

dogs = [Dog("Rex", 3), Dog("Max", 10), Dog("Luna", 0)]

list(filter(puppies, dogs))   # [Dog("Luna", 0)]
list(filter(adults, dogs))    # [Dog("Rex", 3)]
list(filter(seniors, dogs))   # [Dog("Max", 10)]
```

Java equivalent: a class implementing `Predicate<Dog>`. Python's is simpler — no interface declaration needed.

**2. Stateful transformers:**

```python
class TokenCounter:
    def __init__(self):
        self.total = 0

    def __call__(self, text):
        tokens = len(text.split())
        self.total += tokens
        return tokens

count = TokenCounter()

count("hello world")       # 2
count("this is python")    # 3
count.total                # 5 — accumulated state
```

**3. Decorators implemented as classes:**

Callable objects make excellent decorators when the decorator itself needs state:

```python
class retry:
    def __init__(self, times=3, exceptions=(Exception,)):
        self.times = times
        self.exceptions = exceptions

    def __call__(self, func):
        from functools import wraps

        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(self.times):
                try:
                    return func(*args, **kwargs)
                except self.exceptions as e:
                    if attempt == self.times - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}")
        return wrapper

@retry(times=3, exceptions=(ConnectionError, TimeoutError))
def call_api(url):
    return requests.get(url)
```

The decorator itself is a callable object — `retry(times=3)` creates an instance, then calling that instance on `call_api` applies the decoration.

**4. Pipeline stages:**

```python
class Pipeline:
    def __init__(self):
        self.stages = []

    def add(self, stage):
        self.stages.append(stage)
        return self    # fluent interface

    def __call__(self, data):
        for stage in self.stages:
            data = stage(data)
        return data

class Normalise:
    def __call__(self, text):
        return text.lower().strip()

class Tokenise:
    def __call__(self, text):
        return text.split()

class RemoveStopWords:
    STOP_WORDS = {"the", "a", "an", "is", "are"}

    def __call__(self, tokens):
        return [t for t in tokens if t not in self.STOP_WORDS]

pipeline = Pipeline()
pipeline.add(Normalise()).add(Tokenise()).add(RemoveStopWords())

pipeline("The Dogs ARE Running")
# ["dogs", "running"]
```

Each stage is a callable object. The pipeline chains them. Stages can be swapped, reordered, configured independently.

---

## `callable()` — Checking If Something Is Callable

```python
callable(print)              # True — built-in function
callable(Dog)                # True — calling a class creates an instance
callable(Multiplier(2))      # True — has __call__
callable(dog)                # False — Dog instance without __call__
callable(42)                 # False — int
callable(lambda x: x)       # True — lambda
```

Everything callable in Python — functions, classes, lambdas, objects with `__call__` — can be passed wherever a callable is expected. No interface, no type declaration. Just check `callable()` or duck-type it.

---

## Partial Application — `functools.partial`

Creating specialised versions of functions without writing new classes:

```python
from functools import partial

def multiply(x, factor):
    return x * factor

double = partial(multiply, factor=2)    # specialised version
triple = partial(multiply, factor=3)

double(5)    # 10
triple(5)    # 15

# same as Multiplier functor — but for simple cases, partial is lighter
```

`partial` creates a callable that wraps another callable with some arguments pre-filled. Java equivalent: method references with argument binding, or a lambda wrapping a call.

For simple cases, prefer `partial`. For complex stateful cases with multiple methods, prefer a callable class.

---

## Lambda — Python's Lightweight Anonymous Function

Python's `lambda` is deliberately limited:

```python
# lambda — single expression only
square = lambda x: x ** 2
add = lambda x, y: x + y

# equivalent function
def square(x):
    return x ** 2
```

Java's lambdas can be multi-line and contain statements. Python's `lambda` is intentionally restricted to a single expression — if you need more, use a `def`.

The restriction is a design choice: Guido van Rossum (Python's creator) wanted to discourage complex anonymous functions. If it's complex enough to need multiple lines, it deserves a name.

```python
# Python — complex logic gets a name
def transform(dog):
    if dog.age > 5:
        return dog.name.upper()
    return dog.name.lower()

# not this
transform = lambda dog: dog.name.upper() if dog.age > 5 else dog.name.lower()
# technically works — but unreadable, fights the language
```

---

## Comparison: Java Functional Interfaces vs Python Callables

| Java | Python |
|---|---|
| `Runnable` | Any `callable` with no args |
| `Callable<T>` | Any `callable` returning a value |
| `Predicate<T>` | Any `callable` returning `bool` |
| `Function<T,R>` | Any `callable` taking one arg |
| `Consumer<T>` | Any `callable` used for side effects |
| `Comparator<T>` | Any `callable` used as `key=` |
| `@FunctionalInterface` | `callable()` check or duck typing |

Python has no functional interface taxonomy. Any callable works anywhere a callable is expected. Less structure — more flexibility.

---

## Functors in AI Applications

```python
class PromptTemplate:
    def __init__(self, template: str, model: str = "claude-sonnet-4-20250514"):
        self.template = template
        self.model = model
        self.call_count = 0

    def __call__(self, client, **kwargs) -> str:
        self.call_count += 1
        prompt = self.template.format(**kwargs)
        response = client.messages.create(
            model=self.model,
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text

    def __repr__(self):
        return f"PromptTemplate(calls={self.call_count})"

# create reusable prompt templates
summarise = PromptTemplate("Summarise the following in 3 bullet points:\n\n{text}")
classify = PromptTemplate("Classify this text as positive, negative, or neutral:\n\n{text}")
translate = PromptTemplate("Translate the following to {language}:\n\n{text}")

client = anthropic.Anthropic()

# use like functions
result = summarise(client, text="Python is a dynamic language...")
result = classify(client, text="I love Python!")
result = translate(client, text="Hello world", language="French")

# but with state
print(summarise)    # PromptTemplate(calls=1)
print(classify)     # PromptTemplate(calls=1)
```

Each template is callable like a function — but carries its own template string, model configuration, and call count. A plain function couldn't carry that state cleanly.

---

## The Mental Model

> *A functor is an object that behaves like a function — implement `__call__` and Python treats it as callable everywhere. Use functors when your callable needs state, configuration, or additional methods. Use plain functions for stateless behaviour. Use `functools.partial` for simple argument pre-filling. The choice is about complexity — reach for the simplest tool that does the job.*

---

*Next: [Chapter 14 — Mixins: Multiple Inheritance Done Right](chapter-14-mixins.md)*
