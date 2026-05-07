# Chapter 4: OOP Reimagined — From Constructor Overloading to @classmethod

## The Problem

In Java, you create objects in multiple ways using constructor overloading:

```java
class Dog {
    String name;
    int age;

    Dog(String name, int age) {
        this.name = name;
        this.age = age;
    }

    Dog(String csv) {
        String[] parts = csv.split(",");
        this.name = parts[0];
        this.age = Integer.parseInt(parts[1]);
    }

    Dog(String name) {
        this.name = name;
        this.age = 0;
    }
}

new Dog("Rex", 3);      // constructor 1
new Dog("Rex,3");       // constructor 2
new Dog("Rex");         // constructor 3
```

Java picks the right constructor at compile time based on argument types. Clean, explicit, enforced.

Python has none of this. One `__init__`, one set of parameters. If you define two `__init__` methods, the second silently replaces the first:

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __init__(self, csv):           # silently overwrites the first
        parts = csv.split(",")
        self.name = parts[0]
        self.age = int(parts[1])

Dog("Rex", 3)      # ❌ TypeError — only the second __init__ exists
Dog("Rex,3")       # ✅ works
```

This chapter is about how Python solves this — and what it reveals about Python's design philosophy.

---

## `@staticmethod` — The Direct Equivalent

The simplest case: a method that belongs to a class but doesn't need an instance.

In Java:

```java
class MathUtils {
    static int square(int x) {
        return x * x;
    }
}

MathUtils.square(5);    // 25 — no instance needed
```

In Python:

```python
class MathUtils:
    @staticmethod
    def square(x):
        return x * x

MathUtils.square(5)    # 25 — identical behaviour
```

`@staticmethod` is the closest Python equivalent to Java's `static`. No `self`, no `cls` — just a plain function namespaced to the class. It has no reference to the class or any instance.

**When to use it:** Pure utility functions that logically belong with a class but don't need to create instances or reference the class itself.

```python
class Dog:
    @staticmethod
    def validate_age(age):
        return isinstance(age, int) and 0 < age < 30

Dog.validate_age(3)     # True
Dog.validate_age(-1)    # False
```

---

## The Problem With `@staticmethod` for Factories

If `@staticmethod` is Java's `static`, why not use it for factory methods?

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @staticmethod
    def from_string(s):
        name, age = s.split(",")
        return Dog(name, int(age))    # ❌ hardcoded — breaks in subclasses

class Labrador(Dog):
    pass

Labrador.from_string("Rex,3")    # returns Dog, not Labrador — wrong
```

The factory hardcodes `Dog`. Subclasses that inherit it still create `Dog` instances. Java doesn't have this problem because constructor overloading is tied to the specific class being constructed.

Python needs a way for the factory to know which class called it. That's `@classmethod`.

---

## `@classmethod` — Python's Named Constructor

`@classmethod` receives the class itself as the first argument, conventionally called `cls`:

```python
class Dog:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @classmethod
    def from_string(cls, s):
        name, age = s.split(",")
        return cls(name, int(age))    # cls adapts to whoever called this

    @classmethod
    def from_dict(cls, d):
        return cls(d["name"], d["age"])

    @classmethod
    def puppy(cls, name):
        return cls(name, 0)           # default age
```

Each `@classmethod` is a named constructor — an alternative way to create an instance:

```python
Dog("Rex", 3)                          # normal constructor
Dog.from_string("Rex,3")               # named constructor
Dog.from_dict({"name": "Rex", "age": 3})  # named constructor
Dog.puppy("Rex")                       # named constructor
```

**The subclass fix:**

```python
class Labrador(Dog):
    pass

Labrador.from_string("Rex,3")    # returns Labrador — cls = Labrador ✅
```

`cls` is whatever class called the method. `@staticmethod` is hardcoded; `@classmethod` is polymorphic.

---

## How `@classmethod` Maps to Java Constructors

The mapping is direct:

```java
// Java constructor overloading
Dog dog1 = new Dog("Rex", 3);
Dog dog2 = new Dog("Rex,3");
Dog dog3 = new Dog("Rex");
```

```python
# Python named constructors
dog1 = Dog("Rex", 3)
dog2 = Dog.from_string("Rex,3")
dog3 = Dog.puppy("Rex")
```

Java picks by argument types at compile time. Python picks by method name at runtime. The Java way is implicit — Python's way is explicit. In this case, Python's explicitness is arguably cleaner: `Dog.from_string` tells you exactly what's happening. `new Dog("Rex,3")` requires you to know which constructor matches that signature.

---

## The Factory Pattern — When a Separate Class Makes Sense

`@classmethod` works when you're creating one type of object. But what if you need to decide *between* types at runtime?

```python
# @classmethod doesn't work cleanly here
# Dog shouldn't know about Cat
class Dog:
    @classmethod
    def create(cls, animal_type):
        if animal_type == "cat":
            return Cat()    # ❌ Dog creating a Cat — wrong responsibility
        return cls()
```

This is where the factory class pattern makes sense:

```java
// Java — separate factory class
class AnimalFactory {
    static Animal create(String type) {
        if (type.equals("dog")) return new Dog();
        if (type.equals("cat")) return new Cat();
        throw new IllegalArgumentException("Unknown type");
    }
}
```

```python
# Python — same idea, same pattern
class AnimalFactory:
    @staticmethod
    def create(animal_type):
        types = {"dog": Dog, "cat": Cat}
        if animal_type not in types:
            raise ValueError(f"Unknown type: {animal_type}")
        return types[animal_type]()

AnimalFactory.create("dog")    # Dog instance
AnimalFactory.create("cat")    # Cat instance
```

The factory lives outside both `Dog` and `Cat` — neither class knows the other exists.

---

## How to Pick

The decision is straightforward:

| Situation | Use |
|---|---|
| One class, multiple input formats | `@classmethod` |
| Utility function, no class needed | `@staticmethod` |
| Choosing between multiple classes | Factory class |
| Creation logic is complex or shared | Factory class |

**The gut check:**

> Does this feel like a `Dog` concern? → `@classmethod` on `Dog`
> Does this feel like a decision *about* dogs and cats? → Factory class

---

## `*args` and `**kwargs` — Flexible Arguments

Python's substitute for some overloading use cases. Where Java uses overloading to handle different argument counts, Python uses default values and flexible argument syntax:

**Default values:**

```python
class Dog:
    def __init__(self, name, age=0, breed="mixed"):
        self.name = name
        self.age = age
        self.breed = breed

Dog("Rex")                    # age=0, breed="mixed"
Dog("Rex", 3)                 # breed="mixed"
Dog("Rex", 3, "Labrador")     # all specified
```

**`*args` — variable positional arguments:**

```python
def log(*messages):           # collects all positional args into a tuple
    for msg in messages:
        print(msg)

log("one")
log("one", "two", "three")
```

**`**kwargs` — variable keyword arguments:**

```python
def create_dog(**fields):     # collects all keyword args into a dict
    return Dog(**fields)

create_dog(name="Rex", age=3, breed="Labrador")
```

**Unpacking with `*` and `**`:**

```python
args = ["Rex", 3]
Dog(*args)                    # Dog("Rex", 3) — unpack list into positional args

kwargs = {"name": "Rex", "age": 3}
Dog(**kwargs)                 # Dog(name="Rex", age=3) — unpack dict into keyword args
```

This is Python's answer to some of what Java handles with overloading — not as strict, but far more flexible.

---

## The Comparison Table

| Java | Python | Notes |
|---|---|---|
| `static int square(int x)` | `@staticmethod` | No class/instance reference |
| Constructor overloading | `@classmethod` named constructors | Explicit names instead of signatures |
| Factory class | Factory class | Same pattern, less ceremony |
| `new Dog("Rex,3")` | `Dog.from_string("Rex,3")` | Python is more explicit |
| `new Dog("Rex")` | `Dog.puppy("Rex")` | Named intent |

---

## The Mental Model

> *Java constructor overloading lets one name (`Dog`) do multiple jobs, distinguished by argument types at compile time. Python replaces this with named classmethods — explicit names that tell you exactly what kind of creation is happening, resolved at runtime.*

The Python way is actually more readable. `Dog.from_string("Rex,3")` tells you more than `new Dog("Rex,3")` — you know the string is being parsed. The cost is that you write more method names. The benefit is that your code is self-documenting.

---

*Next: [Chapter 5 — The Bouncer: @property and Controlled Attribute Access](chapter-5-property.md)*
