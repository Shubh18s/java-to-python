# Chapter 6: Interfaces as Patchwork — Abstract Base Classes

## The Problem

Java enforces contracts at compile time. If a class claims to implement an interface, every method in that interface must be present — or the code doesn't compile:

```java
interface Animal {
    void speak();
    void move();
}

class Cat implements Animal {
    public void speak() { System.out.println("Meow"); }
    // forgot move() — compile error immediately
}
```

Python has no compiler. Classes can claim to be things they aren't, and no one finds out until runtime — potentially deep in production:

```python
class Cat:
    def speak(self):
        print("Meow")
    # forgot move() — no error

cat = Cat()               # fine
cat.move()                # AttributeError — discovered at runtime
```

Abstract Base Classes (ABCs) are Python's solution — enforcing method contracts at instantiation time, not compile time.

---

## Python's Original Answer — Duck Typing

Before understanding ABCs, understand why Python didn't have them originally.

Python's philosophy was: *if it has the method, it works.* No formal contract needed:

```python
def make_it_move(thing):
    thing.move()           # don't care what it is

class Dog:
    def move(self): print("running")

class Car:
    def move(self): print("driving")

class River:
    def move(self): print("flowing")

make_it_move(Dog())      # ✅
make_it_move(Car())      # ✅ — Car is not an Animal, doesn't matter
make_it_move(River())    # ✅ — River is not even alive, doesn't matter
```

This is duck typing: *"If it walks like a duck and quacks like a duck, it's a duck."*

For many Python codebases, this is enough. Small teams, trusted developers, clear conventions. You don't need a formal contract if everyone knows what they're doing.

But as Python grew into large enterprise codebases — the kind Java was designed for — informal contracts broke down. ABCs were added as an opt-in safety net.

---

## The `NotImplementedError` Approach — Lightweight Contract

Before ABCs, Python developers documented intent with `NotImplementedError`:

```python
class Animal:
    def speak(self):
        raise NotImplementedError("subclasses must implement speak()")

    def move(self):
        raise NotImplementedError("subclasses must implement move()")

class Cat(Animal):
    def speak(self):
        print("Meow")
    # forgot move()

cat = Cat()          # ✅ no error yet
cat.move()           # ❌ NotImplementedError — discovered at call time
```

Better than a silent `AttributeError` — at least the error message is clear. But still discovered at runtime, and only when the specific method is called.

---

## `ABC` — Formal Contracts

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self):
        pass

    @abstractmethod
    def move(self):
        pass

class Dog(Animal):
    def speak(self): print("Woof")
    def move(self): print("running")    # ✅ both implemented

class Cat(Animal):
    def speak(self): print("Meow")
    # forgot move()

class Snake(Animal):
    pass                                # forgot everything

dog = Dog()      # ✅ fine
cat = Cat()      # ❌ TypeError: Can't instantiate abstract class Cat
                 #    with abstract method move
snake = Snake()  # ❌ TypeError: Can't instantiate abstract class Snake
                 #    with abstract methods move, speak
```

Caught at instantiation time — before any method is called. Much closer to Java's compile-time safety.

---

## ABC Is Both Interface and Abstract Class

Java distinguishes between two concepts:

```java
// Interface — pure contract, no implementation
interface Animal {
    void speak();
    void move();
}

// Abstract class — partial implementation allowed
abstract class Animal {
    abstract void speak();      // must implement

    void breathe() {            // provided — subclasses get this free
        System.out.println("breathing");
    }
}
```

Python's `ABC` covers both in one:

```python
class Animal(ABC):
    @abstractmethod
    def speak(self):            # must implement — like Java interface
        pass

    @abstractmethod
    def move(self):             # must implement — like Java interface
        pass

    def breathe(self):          # provided — like Java abstract class
        print("breathing")

class Dog(Animal):
    def speak(self): print("Woof")
    def move(self): print("running")
    # breathe() inherited for free

dog = Dog()
dog.breathe()    # breathing — no need to implement it
```

This is a deliberate simplification — Python decided one concept covers both use cases. Whether that's better or worse than Java's explicit distinction is a matter of taste.

---

## A Real World Example — Payment Processors

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):

    @abstractmethod
    def charge(self, amount: float) -> dict:
        """Charge the customer. Returns transaction dict."""
        pass

    @abstractmethod
    def refund(self, transaction_id: str) -> bool:
        """Refund a transaction. Returns success bool."""
        pass

    def validate_amount(self, amount: float) -> None:
        """Shared validation — not abstract, provided free."""
        if amount <= 0:
            raise ValueError("Amount must be positive")
        if amount > 10000:
            raise ValueError("Amount exceeds limit")

class StripeProcessor(PaymentProcessor):
    def charge(self, amount: float) -> dict:
        self.validate_amount(amount)    # free from base class
        # stripe-specific logic
        return {"status": "success", "processor": "stripe"}

    def refund(self, transaction_id: str) -> bool:
        # stripe-specific logic
        return True

class PaypalProcessor(PaymentProcessor):
    def charge(self, amount: float) -> dict:
        self.validate_amount(amount)    # same free validation
        # paypal-specific logic
        return {"status": "success", "processor": "paypal"}

    def refund(self, transaction_id: str) -> bool:
        # paypal-specific logic
        return True

# can't instantiate PaymentProcessor directly
processor = PaymentProcessor()    # ❌ TypeError

# must use a concrete implementation
stripe = StripeProcessor()
stripe.charge(100.0)              # ✅
```

The framework guarantees every processor has `charge()` and `refund()`. Shared validation lives once. New processors can't be instantiated unless they implement the contract.

---

## ABCs and `isinstance`

ABCs integrate with Python's type checking:

```python
stripe = StripeProcessor()

isinstance(stripe, StripeProcessor)    # True
isinstance(stripe, PaymentProcessor)   # True — isinstance works up the hierarchy
isinstance(stripe, ABC)                # True
```

This is the same as Java's `instanceof` — a concrete class is an instance of its abstract parent.

You can also register classes as virtual subclasses — telling Python's type system that a class satisfies a contract without actually inheriting from the ABC:

```python
class LegacyProcessor:
    def charge(self, amount): ...
    def refund(self, transaction_id): ...

PaymentProcessor.register(LegacyProcessor)

isinstance(LegacyProcessor(), PaymentProcessor)    # True — virtually registered
```

This is purely for type-checking purposes — `@abstractmethod` enforcement doesn't apply to registered classes. Useful for integrating third-party code that can't inherit from your ABC.

---

## ABCMeta — The Metaclass Underneath

ABCs work because of a metaclass called `ABCMeta`. When you write:

```python
class Animal(ABC):
    pass
```

`ABC` is defined as:

```python
class ABC(metaclass=ABCMeta):
    pass
```

`ABCMeta` intercepts class creation (remember `type.__new__` from Chapter 2?) and tracks which abstract methods each class defines and which it inherits without implementing. At instantiation time it checks — if any abstract methods remain unimplemented, it raises `TypeError`.

You don't need to know this to use ABCs. But it connects to the bigger picture: every time Python adds a "safety" feature, it's using the existing metaclass machinery to do it. ABCs aren't a special language feature — they're a library built on the same object model you already understand.

---

## Three Approaches Compared

| Approach | Error discovered | Enforcement | When to use |
|---|---|---|---|
| Duck typing | At call time | None | Small teams, clear conventions |
| `NotImplementedError` | At call time | Informal | Documenting intent lightly |
| `ABC` | At instantiation | Formal | Large teams, framework code |

---

## The Pattern Again

ABCs follow the same pattern from Chapter 1 — Python bolting on Java-style safety as an opt-in feature:

- Java enforces interface contracts at compile time — mandatory, no choice
- Python gives you duck typing by default — flexible, informal
- `ABC` is the opt-in middle ground — formal contracts, checked at instantiation

The keyword is always opt-in. You decide how much safety you need for each part of your codebase.

---

## The Mental Model

> *ABCs are Python's opt-in interfaces. Duck typing is the default — if it has the method, it works. ABC is for when your codebase is large enough that informal contracts break down and you need Java-style guarantees at instantiation time.*

Your Java background makes you the ideal judge of when to reach for `ABC`. You've worked in codebases where missing method implementations caused real problems. Use that instinct — add ABCs where the team size and complexity warrants it, and trust duck typing where it doesn't.

---

*Next: Chapter 7 — Decorators: Python's Answer to AOP*
