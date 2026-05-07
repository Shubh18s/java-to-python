# Chapter 14: Mixins — Multiple Inheritance Done Right

## The Problem

Java has no multiple inheritance for classes. The diamond problem — where two parent classes share a grandparent and there's ambiguity about which method to call — was considered too dangerous, so Java allows only single inheritance plus multiple interface implementation:

```java
// Java — single inheritance, multiple interfaces
class Dog extends Animal implements Serializable, Comparable<Dog> {
    // must implement all interface methods
}
```

Python allows full multiple inheritance:

```python
class Dog(Animal, Serializable, Comparable):
    pass
```

This sounds dangerous — and it can be, if misused. But Python has a disciplined pattern for using multiple inheritance safely: **mixins**.

---

## What Is a Mixin?

A mixin is a class designed to be inherited alongside other classes — not as a primary parent, but as a bundle of reusable methods that can be added to any class:

```python
class JSONMixin:
    """Adds JSON serialisation to any class."""

    def to_json(self):
        import json
        return json.dumps(self.__dict__)

    @classmethod
    def from_json(cls, json_str):
        import json
        data = json.loads(json_str)
        obj = cls.__new__(cls)
        obj.__dict__.update(data)
        return obj

class LogMixin:
    """Adds logging to any class."""

    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")

class ValidateMixin:
    """Adds validation framework to any class."""

    def validate(self):
        errors = []
        for name, value in self.__dict__.items():
            validator = getattr(self.__class__, f"validate_{name}", None)
            if validator:
                error = validator(self, value)
                if error:
                    errors.append(error)
        return errors
```

None of these mixins are meant to be instantiated directly. They exist to be mixed into other classes:

```python
class Dog(JSONMixin, LogMixin, Animal):
    def __init__(self, name, age):
        self.name = name
        self.age = age

dog = Dog("Rex", 3)
dog.to_json()              # '{"name": "Rex", "age": 3}'
dog.log("created")         # [Dog] created

Dog.from_json('{"name": "Max", "age": 5}')    # Dog instance
```

---

## The Mixin Rules

Mixins work safely when they follow these conventions:

**1. Mixins don't inherit from the primary class hierarchy:**

```python
# GOOD — mixin inherits from object (implicitly)
class JSONMixin:
    def to_json(self): ...

# BAD — mixin inherits from domain class
class JSONMixin(Animal):    # ❌ creates diamond problem
    def to_json(self): ...
```

**2. Mixins don't call `super().__init__()` with arguments:**

```python
class LogMixin:
    def __init__(self):
        super().__init__()    # ✅ pass through — don't add args
```

**3. Mixins provide methods, not state:**

```python
# GOOD — behaviour only
class JSONMixin:
    def to_json(self): ...

# BAD — mixin adding its own state
class JSONMixin:
    def __init__(self):
        self.json_version = 2    # ❌ creates init ordering problems
```

**4. Naming convention — suffix with `Mixin`:**

```python
class JSONMixin: ...      # clearly a mixin
class LogMixin: ...       # clearly a mixin
class Dog(JSONMixin, LogMixin, Animal): ...
```

---

## MRO — Method Resolution Order

Python uses C3 linearisation to determine which class's method is called when there's ambiguity. This is the algorithm that makes multiple inheritance predictable:

```python
class A:
    def method(self): print("A")

class B(A):
    def method(self): print("B")

class C(A):
    def method(self): print("C")

class D(B, C):
    pass

D().method()    # "B" — follows MRO
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

MRO order: `D → B → C → A → object`. Python searches left to right, depth-first, removing duplicates from earlier in the chain. `B` is found first, so `B.method()` runs.

**The practical rule:** put the most specific class first, mixins next, base class last:

```python
class Dog(JSONMixin, LogMixin, ValidateMixin, Animal):
    #       ←mixins→                            ←base→
    pass
```

---

## `super()` and MRO — Cooperative Inheritance

`super()` in Python doesn't mean "my parent class" — it means "the next class in the MRO":

```python
class A:
    def setup(self):
        print("A setup")

class B(A):
    def setup(self):
        print("B setup")
        super().setup()    # next in MRO — not necessarily A

class C(A):
    def setup(self):
        print("C setup")
        super().setup()    # next in MRO

class D(B, C):
    def setup(self):
        print("D setup")
        super().setup()

D().setup()
# D setup
# B setup
# C setup
# A setup
```

MRO for D: `D → B → C → A`. Each `super()` call advances to the next class in this chain. Every class in the hierarchy gets called exactly once — cooperative inheritance.

Java's `super` always means the direct parent. Python's `super()` follows MRO — more powerful, requires understanding the chain.

---

## Django's Use of Mixins

Django uses mixins extensively in its class-based views — a direct example of production Python mixin usage:

```python
# Django's view system
from django.views import View
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic.detail import SingleObjectMixin

class DogDetailView(LoginRequiredMixin, SingleObjectMixin, View):
    model = Dog
    template_name = "dog_detail.html"
    login_url = "/login/"

    def get(self, request, pk):
        dog = self.get_object()        # from SingleObjectMixin
        self.check_permissions()       # from LoginRequiredMixin
        return render(request, self.template_name, {"dog": dog})
```

`LoginRequiredMixin` adds authentication checking. `SingleObjectMixin` adds `get_object()`. `View` is the base. The view class itself is thin — behaviour comes from the mixins.

This is the Django pattern: compose views from mixins rather than deep inheritance hierarchies. New behaviour = new mixin. Add it where needed.

---

## Building a Mixin-Based System

A practical example for AI applications:

```python
import anthropic
from dataclasses import dataclass, field
from typing import Optional
import json
import time

class LogMixin:
    def log(self, message: str) -> None:
        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
        print(f"[{timestamp}] [{self.__class__.__name__}] {message}")

class TokenTrackingMixin:
    def __init__(self):
        super().__init__()
        self.total_input_tokens = 0
        self.total_output_tokens = 0

    def track_usage(self, usage) -> None:
        self.total_input_tokens += usage.input_tokens
        self.total_output_tokens += usage.output_tokens

    @property
    def total_tokens(self) -> int:
        return self.total_input_tokens + self.total_output_tokens

    def usage_summary(self) -> dict:
        return {
            "input": self.total_input_tokens,
            "output": self.total_output_tokens,
            "total": self.total_tokens
        }

class RetryMixin:
    max_retries: int = 3
    retry_delay: float = 1.0

    def with_retry(self, func, *args, **kwargs):
        for attempt in range(self.max_retries):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                time.sleep(self.retry_delay * (attempt + 1))

class ClaudeClient(LogMixin, TokenTrackingMixin, RetryMixin):
    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        super().__init__()
        self.model = model
        self.client = anthropic.Anthropic()

    def ask(self, prompt: str) -> str:
        self.log(f"Sending prompt: {prompt[:50]}...")

        def make_request():
            return self.client.messages.create(
                model=self.model,
                max_tokens=1000,
                messages=[{"role": "user", "content": prompt}]
            )

        response = self.with_retry(make_request)
        self.track_usage(response.usage)
        self.log(f"Response received. Total tokens: {self.total_tokens}")

        return response.content[0].text

client = ClaudeClient()
response = client.ask("What is a mixin?")

print(client.usage_summary())
# {"input": 15, "output": 247, "total": 262}
```

`ClaudeClient` gets logging from `LogMixin`, token tracking from `TokenTrackingMixin`, and retry logic from `RetryMixin`. Each mixin can be tested independently and reused in other classes.

---

## Mixins vs Other Patterns

| Pattern | Use when |
|---|---|
| Mixin | Reusable behaviour to add to multiple unrelated classes |
| Inheritance | True is-a relationship (`Dog` is-a `Animal`) |
| Decorator | Wrapping behaviour around a function |
| Composition | Embedding an object as an attribute |

**Prefer composition over inheritance** is the classic rule. Mixins blur this line — they use inheritance syntax but composition semantics. The test: does the mixin represent *what the class is* (inheritance) or *what the class can do* (mixin)?

```python
class Dog(Animal):          # Dog IS an Animal — inheritance ✅
class Dog(JSONMixin, Animal):   # Dog CAN serialise to JSON — mixin ✅
class Dog(Cat):             # Dog IS a Cat — wrong, don't do this ❌
```

---

## Java Comparison

| Java | Python |
|---|---|
| Single class inheritance | Multiple class inheritance |
| Multiple interface inheritance | Same — but with implementation |
| No mixin concept | Mixins — informal convention |
| `default` interface methods (Java 8+) | Mixin methods |
| No MRO — only single parent | C3 MRO — predictable resolution order |
| `super` = direct parent | `super()` = next in MRO |

Java 8's `default` interface methods are the closest Java equivalent to mixins — interfaces that provide default implementations. But they're more restricted (no state, no constructor participation) than Python mixins.

---

## The Mental Model

> *A mixin is a class that adds a bundle of related methods to whatever class inherits it — not a parent in the is-a sense, but a behaviour provider in the can-do sense. Python's MRO makes the method resolution predictable. `super()` follows MRO, not direct parents — enabling cooperative inheritance where every class in the chain participates. Use mixins to compose behaviour without deep inheritance hierarchies.*

---

*Next: [Chapter 15 — Putting It Together: Building an AI Application](chapter-15-putting-it-together.md)*
