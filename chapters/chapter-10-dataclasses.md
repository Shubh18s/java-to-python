# Chapter 10: Dataclasses — Goodbye Boilerplate

## The Problem

Java developers know this ceremony by heart:

```java
class Dog {
    private String name;
    private int age;
    private String breed;

    public Dog(String name, int age, String breed) {
        this.name = name;
        this.age = age;
        this.breed = breed;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public String getBreed() { return breed; }

    public String toString() {
        return "Dog{name='" + name + "', age=" + age + ", breed='" + breed + "'}";
    }

    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Dog)) return false;
        Dog dog = (Dog) o;
        return age == dog.age &&
               name.equals(dog.name) &&
               breed.equals(dog.breed);
    }

    public int hashCode() {
        return Objects.hash(name, age, breed);
    }
}
```

Fifty lines for three fields. You're not writing logic — you're writing ceremony.

Java IDEs generate this for you. Lombok eliminates it with annotations. But it still clutters your codebase.

Python's `@dataclass` generates all of it automatically from a simple field declaration.

---

## `@dataclass` — The Basic Form

```python
from dataclasses import dataclass

@dataclass
class Dog:
    name: str
    age: int
    breed: str = "mixed"    # default value
```

Three lines. Python generates:

- `__init__(self, name, age, breed="mixed")`
- `__repr__` — readable string representation
- `__eq__` — equality comparison based on all fields

```python
dog1 = Dog("Rex", 3)
dog2 = Dog("Rex", 3)
dog3 = Dog("Rex", 3, "Labrador")

print(dog1)           # Dog(name='Rex', age=3, breed='mixed')
dog1 == dog2          # True — __eq__ generated
dog1 == dog3          # False — breed differs
```

The `name: str` syntax is a type hint — it tells `@dataclass` what fields to generate, and documents the expected types. Python doesn't enforce the types at runtime, but tools like `mypy` will check them statically.

---

## Default Values

```python
@dataclass
class Dog:
    name: str              # required — no default
    age: int = 0           # optional — default 0
    breed: str = "mixed"   # optional — default "mixed"
    vaccinated: bool = False

Dog("Rex")                           # Dog(name='Rex', age=0, breed='mixed', vaccinated=False)
Dog("Rex", 3)                        # Dog(name='Rex', age=3, breed='mixed', vaccinated=False)
Dog("Rex", 3, "Labrador")            # Dog(name='Rex', age=3, breed='Labrador', vaccinated=False)
Dog("Rex", 3, "Labrador", True)      # all specified
```

Java equivalent: constructor overloading with multiple constructors. Python handles it with defaults on a single generated constructor.

**Rule:** fields with defaults must come after fields without. Same as Python function arguments.

---

## The Mutable Default Gotcha

This is one of Python's most common mistakes, amplified by dataclasses:

```python
# WRONG — all instances share the same list
@dataclass
class Dog:
    tricks: list = []    # ❌ SyntaxError — dataclass catches this

# Python actually raises an error here to protect you
# ValueError: mutable default <class 'list'> for field tricks
#             is not allowed: use default_factory
```

`@dataclass` catches this and raises an error. The fix:

```python
from dataclasses import dataclass, field

@dataclass
class Dog:
    tricks: list = field(default_factory=list)    # ✅ new list per instance

dog1 = Dog()
dog2 = Dog()
dog1.tricks.append("sit")
dog2.tricks    # [] — separate list, not shared
```

`default_factory` is called once per instance — not once at class definition. Use it for any mutable default: `list`, `dict`, `set`, or a custom factory function.

---

## `field()` — Fine-Grained Control

`field()` gives you control over individual fields:

```python
from dataclasses import dataclass, field

@dataclass
class Dog:
    name: str
    age: int
    tricks: list = field(default_factory=list)
    _id: str = field(default_factory=lambda: str(uuid.uuid4()), repr=False)
    internal_state: dict = field(default_factory=dict, repr=False, compare=False)
```

Key `field()` options:

| Option | What it does |
|---|---|
| `default` | Simple default value |
| `default_factory` | Callable that creates the default |
| `repr=False` | Exclude from `__repr__` output |
| `compare=False` | Exclude from `__eq__` comparison |
| `init=False` | Don't include in `__init__` — set in `__post_init__` |

---

## `__post_init__` — Validation and Computed Fields

`@dataclass` generates `__init__` but you can hook into it with `__post_init__`, which runs after the generated `__init__`:

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Dog:
    name: str
    age: int
    age_in_human_years: int = field(init=False)    # not in constructor

    def __post_init__(self):
        # validation
        if self.age < 0:
            raise ValueError("age must be positive")
        if not self.name:
            raise ValueError("name cannot be empty")

        # computed fields
        self.age_in_human_years = self.age * 7

dog = Dog("Rex", 3)
dog.age_in_human_years    # 21 — computed automatically

Dog("", 3)     # ValueError: name cannot be empty
Dog("Rex", -1) # ValueError: age must be positive
```

Java equivalent: constructor body validation. Python's `__post_init__` is the same idea — runs after the generated constructor sets the fields.

---

## Immutable Dataclasses — `frozen=True`

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x = 5.0              # ❌ FrozenInstanceError — cannot assign to field

# frozen dataclasses are hashable — can be used as dict keys or in sets
point_set = {Point(0, 0), Point(1, 1), Point(0, 0)}
len(point_set)    # 2 — duplicates removed
```

Java equivalent: `final` fields with no setters. Frozen dataclasses are Python's value objects — immutable data containers.

Regular dataclasses are not hashable by default (because mutable objects shouldn't be dict keys):

```python
@dataclass
class Dog:
    name: str

d = {Dog("Rex"): "value"}    # ❌ TypeError: unhashable type: 'Dog'

@dataclass(frozen=True)
class Dog:
    name: str

d = {Dog("Rex"): "value"}    # ✅ frozen — hashable
```

---

## Combining `@dataclass` With `@property`

Plain `@dataclass` fields have no validation on assignment after creation. Combining with `@property` adds the bouncer:

```python
from dataclasses import dataclass, field

@dataclass
class Dog:
    name: str
    _age: int = field(repr=False)    # hide raw storage

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("age must be positive")
        self._age = value

    def __post_init__(self):
        # trigger setter validation on construction too
        self.age = self._age
```

`@dataclass` handles boilerplate. `@property` adds validation on individual fields that need it.

---

## Inheritance With Dataclasses

```python
@dataclass
class Animal:
    name: str
    age: int

@dataclass
class Dog(Animal):
    breed: str = "mixed"
    tricks: list = field(default_factory=list)

dog = Dog("Rex", 3, "Labrador")
print(dog)    # Dog(name='Rex', age=3, breed='Labrador', tricks=[])
```

The child class inherits parent fields — parent fields come first in `__init__`. Same rules as regular inheritance.

**One rule:** if the parent has a field with a default, all child fields must also have defaults. (Same as Python function arguments — required args can't follow optional ones.)

---

## Dataclasses in AI Applications

Dataclasses are ideal for structured data flowing through AI pipelines:

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Message:
    role: str
    content: str

@dataclass
class ConversationRequest:
    messages: list[Message] = field(default_factory=list)
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 1000
    temperature: float = 1.0

    def add_message(self, role: str, content: str) -> None:
        self.messages.append(Message(role=role, content=content))

@dataclass(frozen=True)
class TokenUsage:
    input_tokens: int
    output_tokens: int

    @property
    def total(self) -> int:
        return self.input_tokens + self.output_tokens

@dataclass
class ConversationResponse:
    content: str
    usage: TokenUsage
    model: str
    stop_reason: Optional[str] = None

# usage
request = ConversationRequest()
request.add_message("user", "What is a metaclass?")
request.add_message("user", "What is a decorator?")

# process request...
usage = TokenUsage(input_tokens=150, output_tokens=320)
response = ConversationResponse(
    content="A metaclass is...",
    usage=usage,
    model="claude-sonnet-4-20250514"
)

print(response.usage.total)    # 470
```

Structured, self-documenting, with generated equality and representation. No boilerplate.

---

## Java Comparison

| Java | Python `@dataclass` |
|---|---|
| Constructor | Generated `__init__` |
| `toString()` | Generated `__repr__` |
| `equals()` | Generated `__eq__` |
| `hashCode()` | Generated `__hash__` (frozen only by default) |
| `final` fields | `frozen=True` |
| Constructor overloading | Default values + `@classmethod` |
| Validation in constructor | `__post_init__` |
| Lombok `@Data` | `@dataclass` |
| Lombok `@Value` | `@dataclass(frozen=True)` |

If you've used Lombok — `@dataclass` is exactly that, built into the standard library.

---

## How It Works Underneath

`@dataclass` is a class decorator that inspects the class's type annotations at runtime and generates the methods. It's using the metaclass machinery and Python's object model — the same things you've learned throughout this book — to generate code at class definition time.

```python
@dataclass
class Dog:
    name: str
    age: int
```

At class definition time, `dataclass` reads `Dog.__annotations__` (`{"name": str, "age": int}`), generates `__init__`, `__repr__`, `__eq__`, and adds them to the class. All at runtime, in pure Python.

Java's Lombok does the same at compile time with annotation processing. Python does it at import time with decorators.

---

## The Mental Model

> *`@dataclass` is a code generator. Declare your fields with type annotations, and Python generates the constructor, string representation, and equality comparison for you. Start with `@dataclass`, add `field()` for control, and `__post_init__` for validation. Only reach for `@property` when you need the bouncer on a specific field.*

---

*Next: [Chapter 11 — Concurrency: The GIL, Threading, and asyncio](chapter-11-concurrency.md)*
