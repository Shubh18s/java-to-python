# Chapter 3: Static vs Dynamic Typing — What Python Is Trading and Why

## The Problem

In Java, this is caught before your code ever runs:

```java
int age = "three";    // ❌ compile error — incompatible types
```

In Python, this runs fine:

```python
age = "three"         # ✅ no error — until something tries to use it as a number
age + 1               # ❌ TypeError — but only at runtime
```

This isn't an oversight. It's a deliberate trade. Understanding what Python gains and loses with dynamic typing is essential — because it explains why half of Python's features exist and why the other half behave the way they do.

---

## What Static Typing Actually Gives You

Java's type system does three things at compile time:

**1. Catches type errors before runtime:**

```java
String name = 42;          // caught immediately
int age = getName();       // caught immediately — getName() returns String
```

**2. Enables method overloading:**

```java
void greet(String name) { ... }
void greet(int id) { ... }   // Java picks the right one at compile time
```

**3. Documents intent:**

```java
public Dog createDog(String name, int age) { ... }
```

The signature tells you exactly what goes in and what comes out. No guessing.

---

## What Dynamic Typing Gives You

Python's variables don't have types. Objects do.

```python
x = 42          # x points at an int object
x = "hello"     # x now points at a str object — perfectly valid
x = [1, 2, 3]   # x now points at a list object — still valid
```

The variable is just a label. The label can point at anything.

**1. Functions that work on any compatible type:**

```python
def double(x):
    return x * 2

double(3)         # 6    — works on int
double("Rex")     # RexRex — works on str
double([1, 2])    # [1, 2, 1, 2] — works on list
```

In Java you'd need three overloaded methods or generics. In Python one function handles all three because it doesn't care about the type — only whether `*` is supported.

**2. Faster prototyping:**

No type declarations means less ceremony. You can sketch an idea in Python in half the code Java would require.

**3. Duck typing — the real power:**

```python
def make_it_speak(thing):
    thing.speak()    # don't care what it is — just needs speak()

class Dog:
    def speak(self): print("Woof")

class Robot:
    def speak(self): print("Beep boop")

make_it_speak(Dog())    # ✅
make_it_speak(Robot())  # ✅ — Robot doesn't extend Dog, doesn't matter
```

In Java, `make_it_speak` would require `thing` to implement an `Animal` interface. In Python, if it has `speak()`, it speaks. No interface required.

This is the origin of the phrase: *"If it walks like a duck and quacks like a duck, it's a duck."*

---

## What Dynamic Typing Costs You

Nothing comes free. Here's what Python trades away:

**1. Errors hide until runtime:**

```python
def process_age(age):
    return age * 7

process_age("three")    # runs fine — until * 7 is hit
                        # TypeError: can't multiply sequence by non-int
```

In a large codebase, this bug might not surface until a specific code path is triggered in production.

**2. No overloading:**

Because Python doesn't know types at definition time, it can't pick between two functions with the same name:

```python
def greet(name):
    print(f"Hello {name}")

def greet(id):           # ❌ just overwrites the first one
    print(f"User {id}")

greet("Rex")             # calls second greet — first is gone
```

This is why Python needs `@classmethod` for named constructors, and factory classes for choosing between types. Dynamic typing created the problem; those features solve it.

**3. IDE support is harder:**

Static analysis tools can't always infer what type a variable holds, so autocomplete and refactoring tools are less reliable than in Java.

---

## Python's Answer — Optional Type Hints

Python 3.5 added the `typing` module — not to enforce types, but to annotate them:

```python
def process_age(age: int) -> int:
    return age * 7

def create_dog(name: str, age: int) -> Dog:
    return Dog(name, age)
```

These hints do nothing at runtime:

```python
process_age("three")    # still runs — hints are not enforced
```

But they give you three things:

**1. Documentation** — the signature now tells you what's expected, like Java.

**2. Tool support** — IDEs use hints for autocomplete and error highlighting.

**3. Static analysis** — tools like `mypy` check your hints without running the code:

```bash
mypy my_file.py
# error: Argument 1 to "process_age" has incompatible type "str"; expected "int"
```

This is Python optionally becoming more like Java — when you want the safety, you add the hints and run mypy. When you want the flexibility, you skip them.

---

## Common Type Hint Patterns

```python
from typing import Optional, Union, List, Dict, Tuple

# optional — can be None
def find_dog(id: int) -> Optional[Dog]:
    return db.get(id)    # might return None

# union — one of several types
def process(value: Union[int, str]) -> str:
    return str(value)

# collections
def get_names(dogs: List[Dog]) -> List[str]:
    return [dog.name for dog in dogs]

def get_registry(dogs: Dict[str, Dog]) -> Dict[str, int]:
    return {name: dog.age for name, dog in dogs.items()}

# Python 3.10+ — cleaner union syntax
def process(value: int | str) -> str:
    return str(value)

# Python 3.9+ — built-in collections work directly
def get_names(dogs: list[Dog]) -> list[str]:
    return [dog.name for dog in dogs]
```

---

## The `type` Builtin vs the `typing` Module

Easy to confuse — completely different things:

```python
type(Dog)         # built-in — tells you the type of an object at runtime
                  # <class 'type'>

from typing import List
List[str]         # typing module — annotates expected types, not enforced
```

| | `type()` builtin | `typing` module |
|---|---|---|
| Purpose | Runtime introspection | Code annotation |
| Enforced? | ✅ Yes — core to Python | ❌ No — documentation only |
| Origin | Always existed | Added Python 3.5 |
| Used for | Metaclasses, checking types | Type hints, IDE support |

The irony: `typing` was added to make Python *feel* more like Java — but Python still doesn't enforce it. You need an external tool (`mypy`) to get Java-style type checking, and even then it's optional.

---

## Dynamic Typing and the GIL

One consequence of dynamic typing that Java developers hit hard — Python's Global Interpreter Lock (GIL).

Python's memory management is not thread safe. Because types are resolved at runtime and objects can change type, managing object references safely across threads requires a lock. The GIL is that lock — it ensures only one thread executes Python code at a time.

Java's static typing means the JVM can manage memory safely across threads without a global lock. Python's dynamic typing made the GIL the simplest solution.

The full implications of the GIL — and how `asyncio` works around it — are covered in Chapter 11. For now, know that dynamic typing has runtime costs beyond just catching errors late.

---

## The Mental Model

> *Java knows what everything is at compile time and uses that knowledge to catch errors early, enable overloading, and optimise execution. Python finds out what everything is at runtime — which costs safety but buys flexibility, duck typing, and less ceremony.*

Type hints are Python's opt-in way of getting some of that Java safety back — without making it mandatory.

Your Java background gives you an advantage here: you already understand what type safety buys you. Python developers who started with Python often undervalue type hints because they've never felt the pain of large untyped codebases. You have. Use that knowledge — add type hints to Python code that matters.

---

*Next: Chapter 4 — OOP Reimagined: From Constructor Overloading to @classmethod*
