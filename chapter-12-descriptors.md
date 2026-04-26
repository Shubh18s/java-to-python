# Chapter 12: Descriptors — The Engine Behind @property

## The Problem

You've used `@property` to add validation to attributes. But `@property` has a limitation — it works on one attribute of one class. If you have ten classes that all need the same validation, you repeat the property ten times:

```python
class Dog:
    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        if not isinstance(value, int) or value < 0:
            raise ValueError("age must be a positive int")
        self._age = value

class Cat:
    @property
    def age(self):           # same validation — repeated
        return self._age

    @age.setter
    def age(self, value):    # same validation — repeated
        if not isinstance(value, int) or value < 0:
            raise ValueError("age must be a positive int")
        self._age = value
```

The descriptor protocol is the mechanism that makes `@property` work — and it lets you write that validation once and apply it anywhere.

---

## What Is a Descriptor?

A descriptor is any object that defines any of these methods:

```python
__get__(self, obj, objtype=None)    # intercepts: dog.age
__set__(self, obj, value)           # intercepts: dog.age = 5
__delete__(self, obj)               # intercepts: del dog.age
```

That's the entire protocol. If an object has any of those methods and is assigned as a class attribute — it becomes a descriptor, and Python calls those methods instead of doing normal attribute access.

`@property` works because `property` is a built-in class that implements all three.

---

## How Python Decides What to Do

When you write `dog.age = 5`, Python doesn't immediately assign. It checks the class first:

```python
# Python's internal logic for dog.age = 5
class_attr = type(dog).__dict__.get("age")    # look in the class

if hasattr(class_attr, "__set__"):
    class_attr.__set__(dog, 5)    # descriptor found — call __set__
else:
    dog.__dict__["age"] = 5       # no descriptor — normal assignment
```

And for `dog.age` (reading):

```python
# Python's internal logic for dog.age
class_attr = type(dog).__dict__.get("age")

if hasattr(class_attr, "__get__"):
    return class_attr.__get__(dog, type(dog))    # descriptor — call __get__
else:
    return dog.__dict__["age"]                   # no descriptor — normal lookup
```

The descriptor protocol is Python checking the class before doing anything with the instance. Every attribute access runs through this check.

---

## Writing a Descriptor

```python
class PositiveInt:
    """Descriptor that validates an attribute must be a positive integer."""

    def __set_name__(self, owner, name):
        """Called automatically when the descriptor is assigned to a class attribute."""
        self.name = name              # "age", "weight", etc.
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self               # accessed on the class, not an instance
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.name} must be an integer")
        if value < 0:
            raise ValueError(f"{self.name} must be positive")
        setattr(obj, self.private_name, value)

    def __delete__(self, obj):
        delattr(obj, self.private_name)
```

Now apply it to any class:

```python
class Dog:
    age = PositiveInt()        # descriptor on the class
    weight = PositiveInt()     # same descriptor, different attribute

class Cat:
    age = PositiveInt()        # reused on a different class
    lives = PositiveInt()

dog = Dog()
dog.age = 3        # ✅ calls PositiveInt.__set__
dog.age            # 3 — calls PositiveInt.__get__
dog.age = -1       # ❌ ValueError: age must be positive
dog.age = "three"  # ❌ TypeError: age must be an integer

cat = Cat()
cat.lives = 9      # ✅
cat.lives = 0      # ❌ ValueError: lives must be positive
```

One descriptor class, validated on any attribute of any class. This is what Django does with model fields.

---

## `__set_name__` — The Descriptor Knows Its Own Name

When Python reads this:

```python
class Dog:
    age = PositiveInt()
```

It automatically calls `PositiveInt.__set_name__(Dog, "age")`. The descriptor knows it was assigned to an attribute called `"age"` on the class `Dog`. Without `__set_name__`, you'd have to hardcode the name:

```python
# Without __set_name__ — fragile
class PositiveInt:
    def __init__(self, name):
        self.name = name              # must pass it manually

class Dog:
    age = PositiveInt("age")         # redundant — "age" appears twice
    weight = PositiveInt("weight")
```

`__set_name__` was added in Python 3.6 specifically to avoid this redundancy.

---

## `obj is None` — Accessing the Descriptor on the Class

Why does `__get__` check `if obj is None`?

```python
dog = Dog()
dog.age        # obj = dog — accessing on instance
Dog.age        # obj = None — accessing on the class itself
```

When you access the descriptor via the class (`Dog.age`), `obj` is `None`. If you return the value normally, it breaks — there's no instance to get the value from. Convention: return the descriptor itself:

```python
def __get__(self, obj, objtype=None):
    if obj is None:
        return self    # Dog.age returns the descriptor object
    return getattr(obj, self.private_name)
```

This allows inspection of the descriptor:

```python
Dog.age                  # <__main__.PositiveInt object>
Dog.age.__class__        # PositiveInt
```

---

## Data Descriptors vs Non-Data Descriptors

Two types, with different priority:

**Data descriptor** — implements `__set__` (and usually `__get__`):

```python
class DataDesc:
    def __get__(self, obj, objtype=None): ...
    def __set__(self, obj, value): ...    # has __set__ → data descriptor
```

**Non-data descriptor** — implements only `__get__`:

```python
class NonDataDesc:
    def __get__(self, obj, objtype=None): ...    # no __set__ → non-data descriptor
```

Priority for attribute lookup:

```
Data descriptor    > instance __dict__  > non-data descriptor
```

Data descriptors (like `@property`) take priority over instance attributes — the descriptor always wins. Non-data descriptors (like functions — yes, functions are descriptors) can be overridden by instance attributes.

This is why you can shadow a method with an instance attribute:

```python
class Dog:
    def speak(self):    # non-data descriptor (function)
        print("Woof")

dog = Dog()
dog.speak = lambda: print("custom")    # instance attribute overrides
dog.speak()    # "custom" — instance attribute wins over non-data descriptor
```

But you can't shadow a `@property`:

```python
class Dog:
    @property
    def age(self):
        return self._age

dog = Dog()
dog.age = 5    # calls __set__ — data descriptor wins over instance dict
```

---

## Functions Are Descriptors

This explains one of Python's most surprising facts: functions implement `__get__`.

```python
class Dog:
    def speak(self):
        print("Woof")

Dog.speak             # <function Dog.speak at 0x...> — unbound
Dog.speak(dog)        # explicit instance

dog.speak             # <bound method Dog.speak of <Dog...>> — bound to dog
dog.speak()           # no need to pass self — it's already bound
```

When you access `dog.speak`, Python calls `speak.__get__(dog, Dog)`. The function's `__get__` returns a *bound method* — the same function with `dog` pre-loaded as the first argument. That's why `self` works automatically.

Java developers take this for granted — `this` is injected by the JVM. In Python, `self` is passed through the descriptor protocol on every method call. Same result, transparent mechanism.

---

## `@classmethod` and `@staticmethod` Are Descriptors

Both are built-in descriptor classes:

```python
class Dog:
    @classmethod
    def from_string(cls, s): ...

    @staticmethod
    def validate(age): ...
```

- `classmethod.__get__(dog, Dog)` returns a bound method with `Dog` (not `dog`) as first arg — that's where `cls` comes from
- `staticmethod.__get__(dog, Dog)` returns the plain function — no binding

The same descriptor protocol that powers `@property` also powers `@classmethod` and `@staticmethod`. They're all just different `__get__` implementations.

---

## Django Model Fields — Descriptors in the Wild

Django's ORM is the most prominent real-world use of descriptors:

```python
class User(models.Model):
    name = models.CharField(max_length=100)    # CharField is a descriptor
    age = models.IntegerField()                # IntegerField is a descriptor
    email = models.EmailField()                # EmailField is a descriptor
```

When you do `user.name = "Rex"`, Django's descriptor:
1. Validates the value (type, max_length)
2. Tracks that the field has been modified
3. Stores the value appropriately for the database

When you do `user.name`, Django's descriptor:
1. Returns the current value
2. Handles lazy loading from the database if needed

All of that happens transparently behind normal attribute access. Django wrote the descriptor once — you get validation, change tracking, and database mapping on every model field automatically.

---

## A Practical Descriptor — Typed Field

A more complete example for real use:

```python
from typing import Type, Any, Optional

class TypedField:
    def __init__(self, expected_type: Type, nullable: bool = False):
        self.expected_type = expected_type
        self.nullable = nullable

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if value is None:
            if not self.nullable:
                raise ValueError(f"{self.name} cannot be None")
            setattr(obj, self.private_name, None)
            return

        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        setattr(obj, self.private_name, value)

class Dog:
    name = TypedField(str)
    age = TypedField(int)
    owner = TypedField(str, nullable=True)

dog = Dog()
dog.name = "Rex"      # ✅
dog.age = 3           # ✅
dog.owner = None      # ✅ — nullable=True
dog.age = "three"     # ❌ TypeError: age must be int, got str
dog.name = None       # ❌ ValueError: name cannot be None
```

---

## Everything Is a Descriptor

The full picture:

| Feature | Descriptor type | `__get__` | `__set__` |
|---|---|---|---|
| Regular function | Non-data | ✅ (binds self) | ❌ |
| `@classmethod` | Non-data | ✅ (binds cls) | ❌ |
| `@staticmethod` | Non-data | ✅ (no binding) | ❌ |
| `@property` | Data | ✅ | ✅ |
| Custom descriptor | Data or non-data | ✅ | Optional |
| Django field | Data | ✅ | ✅ |

Every attribute access in Python runs through the descriptor protocol. Most of the time this is invisible — but every method call, every `@property` access, every `@classmethod` invocation is going through `__get__`.

---

## Java Comparison

Java has no direct equivalent. The closest approaches:

| Python descriptor | Java equivalent |
|---|---|
| `__get__` / `__set__` | Reflection API (`Field.get()`, `Field.set()`) |
| `@property` | Getter/setter methods |
| Custom validator descriptor | Custom annotation + annotation processor |
| Django field | JPA `@Column` annotation + ORM framework |

Java's approach is external — annotations processed by frameworks, reflection handled by libraries. Python's descriptor protocol is built into the language itself — any class can intercept attribute access without a framework.

---

## The Mental Model

> *A descriptor is any object assigned as a class attribute that implements `__get__`, `__set__`, or `__delete__`. Python calls these methods instead of doing normal attribute access. `@property`, `@classmethod`, `@staticmethod`, and regular functions are all descriptors. Write your own to create reusable attribute validators — what `@property` does for one attribute, a descriptor does for any attribute on any class.*

---

*Next: Chapter 13 — Functors and Callable Objects*
