# Chapter 5: The Bouncer — @property and Controlled Attribute Access

## The Problem

In Java, you learn on day one: never expose fields directly.

```java
class Dog {
    public int age;    // ❌ bad Java — anyone can set this to anything
}

dog.age = -5;          // nothing stops this
dog.age = 999;         // or this
```

The Java solution — always use getters and setters:

```java
class Dog {
    private int age;

    public int getAge() { return age; }

    public void setAge(int value) {
        if (value < 0) throw new IllegalArgumentException("must be positive");
        this.age = value;
    }
}

dog.setAge(-5);    // caught
dog.setAge(3);     // fine
```

This works — but it has a cost. Every caller must use `dog.getAge()` and `dog.setAge()` forever. If you started with a public field and later decide to add validation, you have to refactor every line of calling code.

Python's original philosophy was different: just expose the field. Trust the developer. The calling code is simpler and there's less ceremony.

But then you need validation. And this is where `@property` comes in.

---

## `@property` — The Bouncer Pattern

`@property` lets you start with a plain attribute and add control later — without changing any calling code.

**Start simple:**

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age           # plain attribute — direct access

dog = Dog("Rex", 3)
dog.age = 5                      # direct assignment
print(dog.age)                   # direct access
```

**Add validation later — zero impact on callers:**

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age           # this now triggers the setter

    @property
    def age(self):               # getter
        return self._age

    @age.setter
    def age(self, value):        # setter — the bouncer
        if value < 0:
            raise ValueError("age must be positive")
        if not isinstance(value, int):
            raise TypeError("age must be an integer")
        self._age = value

dog = Dog("Rex", 3)
dog.age = 5                      # still looks identical to the caller
print(dog.age)                   # still looks identical to the caller
dog.age = -1                     # ❌ ValueError
```

The calling code never changes. The bouncer is invisible.

This is impossible in Java — switching from a public field to `getAge()`/`setAge()` breaks every caller. Python's `@property` gives you the flexibility to start simple and add control when you need it.

---

## How `@property` Works

`@property` is a decorator that returns a `property` object instead of a function. That property object holds your getter function inside it.

Step by step:

```python
class Dog:
    @property
    def age(self):
        return self._age
```

This is exactly:

```python
class Dog:
    def age(self):
        return self._age
    age = property(age)    # age is now a property object, not a function
```

`property` is a built-in class. `age` is now an instance of that class sitting on `Dog`, holding your getter function.

Then `@age.setter`:

```python
    @age.setter
    def age(self, value):
        self._age = value
```

Is exactly:

```python
    def set_age(self, value):
        self._age = value
    age = age.setter(set_age)    # creates NEW property object with getter + setter
```

`.setter()` doesn't modify the existing property — it creates a brand new property object that copies the getter and adds the setter. `age` gets reassigned to this new object.

So `age` is reassigned twice:
1. First to a getter-only property object
2. Then to a getter+setter property object

No overriding, no conflict — just reassignment.

---

## The Three Parts

```python
class Dog:
    @property
    def age(self):               # getter — dog.age
        return self._age

    @age.setter
    def age(self, value):        # setter — dog.age = 5
        self._age = value

    @age.deleter
    def age(self):               # deleter — del dog.age (rarely used)
        del self._age
```

You don't need all three. Getter-only is common for computed, read-only attributes:

```python
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def area(self):                       # computed — no setter
        return 3.14159 * self.radius ** 2

    @property
    def circumference(self):              # computed — no setter
        return 2 * 3.14159 * self.radius

c = Circle(5)
c.area                    # 78.54 — computed on access
c.circumference           # 31.42 — computed on access
c.area = 100              # ❌ AttributeError — no setter defined
```

`area` is not stored anywhere — it's computed fresh every time you access it. No setter means read-only. This is a pattern Java handles with final fields plus a getter method — Python makes it look like attribute access.

---

## What the Bouncer Can Do

The setter isn't limited to validation. Three things it can do:

**1. Validate:**

```python
@age.setter
def age(self, value):
    if value < 0:
        raise ValueError("age must be positive")
    self._age = value
```

**2. Transform — silently fix the input:**

```python
@age.setter
def age(self, value):
    self._age = int(value)    # cast to int silently — "3" becomes 3
```

**3. Trigger side effects:**

```python
@age.setter
def age(self, value):
    self._age = value
    self.updated_at = datetime.now()    # log when something changed
    self._notify_observers()            # tell other objects about the change
```

All of this hidden behind `dog.age = 3`. The caller never knows.

---

## The `_age` Convention

You'll notice `@property` examples always use `self._age` (underscore prefix) to store the actual value, while exposing `self.age` (no underscore) as the property.

This is a Python convention:

```python
self.age    # the public interface — goes through the property
self._age   # the private storage — underscore means "don't touch directly"
```

Python doesn't enforce privacy — the underscore is a social contract, not a language rule. Unlike Java's `private`, nothing prevents access to `self._age`. It just signals: *"this is an implementation detail, use the public interface."*

```python
dog._age = -5    # ✅ Python allows it — but you're breaking the contract
dog.age = -5     # ❌ ValueError — bouncer catches it
```

Java developers sometimes find this uncomfortable. The Python philosophy: trust the developer. The underscore is enough of a signal for reasonable people.

---

## Combining with `@dataclass`

`@dataclass` generates `__init__` automatically from field definitions — but plain `@dataclass` fields have no validation. Combining with `@property` adds the bouncer:

```python
from dataclasses import dataclass, field

@dataclass
class Dog:
    name: str
    _age: int = field(repr=False)    # hide raw storage from repr

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("age must be positive")
        self._age = value

dog = Dog(name="Rex", _age=3)
dog.age        # 3
dog.age = -1   # ❌ ValueError
```

The `@dataclass` handles boilerplate. `@property` adds control where needed.

---

## Java Comparison

| Java | Python |
|---|---|
| `private int age` | `self._age` (convention, not enforced) |
| `public int getAge()` | `@property` getter |
| `public void setAge(int v)` | `@age.setter` |
| Must choose upfront | Start plain, add property later |
| Breaking change to add validation | Zero impact on callers |
| `final` field — immutable | `@property` with no setter |

The critical difference: Java forces the getter/setter decision upfront. Python lets you start with a plain attribute and add `@property` when you actually need control — without touching any calling code.

---

## `@property` Is a Descriptor

Under the hood, `@property` works because the `property` class implements `__get__` and `__set__` — the descriptor protocol. When Python sees `dog.age = 5` and finds that `Dog.age` is a property object (which has `__set__`), it calls `__set__` instead of doing a normal assignment.

The full descriptor protocol is covered in Chapter 8. For now, understand that `@property` isn't magic — it's a built-in class using a mechanism that any Python class can use. You can write your own reusable validators that work the same way.

---

## The Mental Model

> *`@property` is a bouncer on your attribute. From the outside, it looks like a plain variable. On the inside, every access and assignment goes through your logic first. Start without it, add it when you need control — the caller never knows the difference.*

---

*Next: [Chapter 6 — Interfaces as Patchwork: Abstract Base Classes](chapter-6-abstract-base-classes.md)*
