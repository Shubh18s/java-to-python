# Chapter 2: Everything Is an Object (Including Classes)

## The Problem

In Java, there is a clear hierarchy of things:

- **Primitives** — `int`, `boolean`, `double` — raw values, not objects
- **Objects** — instances of classes — have methods, can be passed around
- **Classes** — blueprints for objects — exist at compile time, not runtime citizens

This hierarchy feels natural. Classes are special. They exist to create objects. They are not objects themselves.

Python breaks this completely.

In Python, **everything** is an object. Primitives, functions, classes — all objects. All have types. All can be passed around, stored in variables, returned from functions.

This single idea — everything is an object — is the foundation that makes decorators, metaclasses, descriptors, and half of Python's power possible. Without understanding it, those features feel like magic. With it, they feel inevitable.

---

## Functions Are Objects

In Java, a method belongs to a class. You cannot pass it around independently:

```java
// Java — you cannot do this
void greet(String name) { System.out.println("Hello " + name); }
someMethod(greet);    // ❌ methods are not objects
```

Java 8 introduced lambdas and method references to partially address this — but it's a workaround, not the language's natural model.

In Python, functions are objects from the start:

```python
def greet(name):
    print(f"Hello {name}")

# a function is just an object — assign it to a variable
say_hello = greet        # no () — not calling it, just pointing at it
say_hello("Rex")         # Hello Rex

# pass it to another function
def call_twice(func, value):
    func(value)
    func(value)

call_twice(greet, "Rex")
# Hello Rex
# Hello Rex

# store it in a list
actions = [greet, print, len]
actions[0]("Rex")        # Hello Rex
```

`greet` is just a name pointing at a function object. You can point other names at it, pass it around, store it anywhere you'd store any other value.

This is why decorators work. A decorator is just a function that receives a function object and returns a new function object:

```python
def log(func):           # receives function object
    def wrapper(*args, **kwargs):
        print("before")
        return func(*args, **kwargs)
    return wrapper       # returns function object

@log
def greet(name):
    print(f"Hello {name}")
```

`@log` is just `greet = log(greet)`. Nothing magical — just passing a function object to another function.

---

## Classes Are Objects Too

This is where Java developers stop and stare.

In Python, a class is also just an object. It gets created at runtime, assigned to a variable, and can be passed around like anything else:

```python
class Dog:
    def speak(self):
        print("Woof")

# Dog is just a variable pointing at a class object
print(Dog)              # <class '__main__.Dog'>
print(type(Dog))        # <class 'type'>

# assign it to another variable
Animal = Dog
Animal()                # creates a Dog instance

# pass it to a function
def create(cls):
    return cls()

create(Dog)             # Dog instance
create(list)            # empty list — list is also just a class object
```

In Java, `Dog` at runtime is a `Class<Dog>` object you access through reflection — awkward, verbose, rarely used. In Python, `Dog` *is* just an object, naturally, always.

This is why `@classmethod` works. `cls` is just the class object passed as an argument:

```python
class Dog:
    @classmethod
    def from_string(cls, s):
        name, age = s.split(",")
        return cls(name, int(age))    # cls is the Dog class object
```

And why factory patterns are simpler:

```python
def animal_factory(animal_class, *args):
    return animal_class(*args)        # calling the class object creates an instance

animal_factory(Dog, "Rex", 3)
animal_factory(Cat, "Whiskers", 2)
```

---

## `type` — The Class of All Classes

If everything is an object, and every object has a type — what is the type of a class?

```python
dog = Dog()
print(type(dog))        # <class '__main__.Dog'>   — dog's type is Dog
print(type(Dog))        # <class 'type'>           — Dog's type is type
print(type(int))        # <class 'type'>           — int's type is type
print(type(str))        # <class 'type'>           — str's type is type
print(type(type))       # <class 'type'>           — type's type is itself
```

`type` is the class of all classes. Every class in Python — including built-ins like `int`, `str`, `list` — is an instance of `type`.

`type` is called a **metaclass** — a class whose instances are themselves classes.

The hierarchy:

```
type          ← metaclass — its instances are classes
  └── Dog     ← class — its instances are objects  
        └── dog_instance
```

Compare to Java:

```
Class<Dog>    ← Java's runtime class representation (accessed via reflection)
  └── Dog     ← class
        └── dogInstance
```

In Java, `Class<Dog>` is a separate reflection API you rarely touch. In Python, `type` is just how the language works — every class creation goes through it.

---

## `type` Creates Classes at Runtime

Because `type` is just a class, you can call it to create new classes dynamically:

```python
# these two are identical
class Dog:
    species = "Canis lupus"
    def speak(self):
        print("Woof")

Dog = type("Dog", (), {
    "species": "Canis lupus",
    "speak": lambda self: print("Woof")
})
```

`type(name, bases, dict)` — three arguments:
- `name` — the class name as a string
- `bases` — tuple of parent classes
- `dict` — the class attributes and methods

This is what Python does internally every time it reads a `class` statement. You're just doing it explicitly.

When would you actually use this? Mostly in frameworks:

```python
# Django-style ORM — creates model classes from configuration
def create_model(name, fields):
    attrs = {"__module__": __name__}
    attrs.update(fields)
    return type(name, (BaseModel,), attrs)

UserModel = create_model("User", {
    "name": CharField(),
    "email": EmailField(),
    "age": IntegerField()
})
```

The framework doesn't know your field names at import time — it builds the class at runtime from whatever you defined. Java does this with reflection; Python does it by treating class creation as a normal function call.

---

## Metaclasses — Customising Class Creation

Since `type` creates all classes, you can subclass `type` to customise how classes are created:

```python
class EnforceMeta(type):
    def __new__(cls, name, bases, dct):
        # intercept class creation
        if name != "Animal" and "speak" not in dct:
            raise TypeError(f"{name} must implement speak()")
        return super().__new__(cls, name, bases, dct)

class Animal(metaclass=EnforceMeta):
    pass

class Dog(metaclass=EnforceMeta):
    def speak(self):
        print("Woof")    # ✅ has speak()

class Cat(metaclass=EnforceMeta):
    pass               # ❌ TypeError at class definition time
```

This is the same idea as subclassing any other class — but because you're subclassing `type`, your class gets to control how *other* classes are built.

Java equivalent — there isn't one. The closest is annotations combined with compile-time annotation processors. Python does it in plain Python at runtime.

In practice, most Python developers never write their own metaclass. But understanding them explains why `ABC`, `dataclasses`, and Django models work the way they do — they're all metaclasses underneath.

---

## `__new__` vs `__init__` — Two Stages of Creation

Java has one constructor. Python has two:

```python
class Dog:
    def __new__(cls):
        print("1. creating the object")
        instance = super().__new__(cls)
        return instance              # must return the new instance

    def __init__(self):
        print("2. initialising the object")
        self.name = "Rex"

dog = Dog()
# 1. creating the object
# 2. initialising the object
```

`__new__` creates the object and returns it. `__init__` receives the created object and sets it up. Java's constructor does both in one step.

Why does Python split them?

Because `__new__` is also what gets called when a *class* is created — inside a metaclass:

```python
class Meta(type):
    def __new__(cls, name, bases, dct):
        # cls = Meta (the metaclass)
        # name = "Dog" (the class being created)
        # bases = () (parent classes)
        # dct = {"speak": <function>} (class contents)
        print(f"Creating class: {name}")
        return super().__new__(cls, name, bases, dct)

class Dog(metaclass=Meta):
    def speak(self):
        print("Woof")
# Creating class: Dog
```

Same `__new__` mechanism — just operating one level up. Creating a class is creating an object, because classes are objects.

---

## The Two Pillars of Python's Object Model

Two built-in things underpin everything:

**`object`** — the base class of all classes:

```python
class Dog:           # implicitly: class Dog(object)
    pass

isinstance(Dog(), object)    # True — everything is an object
```

**`type`** — the metaclass of all classes:

```python
isinstance(Dog, type)        # True — every class is an instance of type
isinstance(int, type)        # True
isinstance(type, type)       # True — type is an instance of itself
```

```
object  ← every class inherits from object (is-a relationship)
type    ← every class is an instance of type (created-by relationship)
```

Java has `Object` as the base class — but no equivalent of `type`. Classes in Java are not runtime objects you interact with directly. In Python they are, always, by design.

---

## Why This Matters for AI Applications

Everything in Python's data and ML ecosystem depends on this object model:

```python
import torch

# a neural network layer is just a class
class MyModel(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.layer = torch.nn.Linear(768, 10)

# pass the class around — not an instance
def build_model(model_class, *args):
    return model_class(*args)

model = build_model(MyModel)
```

```python
# Anthropic SDK — the client is an object, models are strings, 
# but the streaming response is an iterator — all objects
client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1000,
    messages=[{"role": "user", "content": "explain metaclasses"}]
) as stream:
    for text in stream.text_stream:    # stream is an iterator object
        print(text, end="", flush=True)
```

The `with` statement works because the stream object has `__enter__` and `__exit__`. The `for` loop works because it has `__iter__` and `__next__`. All of that works because in Python, objects define their own behaviour through methods — and everything, including the stream, is an object.

---

## The Mental Model

> *In Java, classes are blueprints that exist at compile time. In Python, classes are objects that exist at runtime — created by `type`, passed around like any other value, customisable at the moment of creation.*

Once this clicks, nothing in Python feels magical anymore. Decorators pass function objects around. `@classmethod` passes the class object as `cls`. Metaclasses subclass `type` to intercept class creation. Descriptors are objects that intercept attribute access.

All of it is just objects, all the way down.

---

*Next: Chapter 3 — Static vs Dynamic Typing: What Python Is Trading and Why*
