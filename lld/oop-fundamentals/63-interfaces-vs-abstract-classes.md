# 63. Interfaces vs Abstract Classes
**Section:** LLD — OOP Fundamentals

---

## Quick Reference

| Feature | Interface | Abstract Class |
|---|---|---|
| Can be instantiated? | No | No |
| Can have state (attributes)? | No (Java) / technically yes but avoid (Python) | Yes |
| Can have concrete methods? | No (Java 7) / Yes (Java 8+ default methods) | Yes |
| Constructor? | No | Yes |
| Multiple inheritance? | Yes — safe | Possible but risky (diamond problem) |
| Relationship type | **Can-do** (Flyable, Serializable) | **Is-a** (Animal, BaseHandler) |
| Python mechanism | `Protocol` or ABC with only `@abstractmethod` | `ABC` with concrete methods + `@abstractmethod` |

---

## What is an Interface?

A **pure contract** — a list of method signatures that a class promises to implement. No state, no implementation.

**Java:**
```java
interface Serializable {
    byte[] serialize();
}

interface Comparable<T> {
    int compareTo(T other);
}

// A class can implement multiple interfaces
class User implements Serializable, Comparable<User> {
    byte[] serialize() { ... }
    int compareTo(User other) { ... }
}
```

**Python via ABC (traditional approach):**
```python
from abc import ABC, abstractmethod

class Serializable(ABC):
    @abstractmethod
    def serialize(self) -> bytes: ...

    @abstractmethod
    def deserialize(self, data: bytes) -> None: ...
```

**Python via Protocol (structural, PEP 544):**
```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    def serialize(self) -> bytes: ...
    def deserialize(self, data: bytes) -> None: ...
```

---

## What is an Abstract Class?

An **abstract class** defines a partial implementation and a contract. It:
- Cannot be instantiated
- Has abstract methods (pure contract) + concrete methods (shared logic)
- Can hold state
- Has a constructor (called via `super().__init__()`)

```python
from abc import ABC, abstractmethod

class BaseHTTPHandler(ABC):
    def __init__(self, timeout: int = 30):
        self.timeout = timeout               # shared state
        self._headers: dict = {}

    def handle(self, request: dict) -> dict:      # concrete — template method
        parsed = self._parse(request)
        validated = self._validate(parsed)
        return self._process(validated)

    def _parse(self, request: dict) -> dict:      # concrete — shared logic
        return {k: v.strip() for k, v in request.items()}

    @abstractmethod
    def _validate(self, data: dict) -> dict: ...  # subclass must implement

    @abstractmethod
    def _process(self, data: dict) -> dict: ...   # subclass must implement


class CreateUserHandler(BaseHTTPHandler):
    def _validate(self, data: dict) -> dict:
        if "email" not in data:
            raise ValueError("email required")
        return data

    def _process(self, data: dict) -> dict:
        user = User.create(data["email"])
        return {"id": user.id, "status": "created"}
```

---

## Q: When to use Interface vs Abstract Class?

### Use Interface (Protocol / pure ABC) when:

**1. Unrelated classes need to share a contract**
```python
class Bird:
    def fly(self, altitude: float) -> None:
        print("Bird flying")

class Airplane:
    def fly(self, altitude: float) -> None:
        print("Airplane flying")

class Superman:
    def fly(self, altitude: float) -> None:
        print("Superman flying")

# Bird, Airplane, Superman are completely unrelated — no shared implementation
# A Protocol is perfect here
class Flyable(Protocol):
    def fly(self, altitude: float) -> None: ...
```

**2. You need multiple "can-do" contracts on the same class**
```python
class PaymentProcessor(Serializable, Auditable, Retryable):
    # Can implement multiple protocols
    ...
```

### Use Abstract Class when:

**1. Related classes share common implementation**
```python
class BaseRepository(ABC):
    def __init__(self, db_connection):
        self._db = db_connection     # shared state

    def find_all(self) -> list:      # shared implementation
        return self._db.execute(f"SELECT * FROM {self.table_name()}")

    @abstractmethod
    def table_name(self) -> str: ...

    @abstractmethod
    def from_row(self, row: dict): ...

class UserRepository(BaseRepository):
    def table_name(self) -> str: return "users"
    def from_row(self, row: dict) -> User: return User(**row)

class OrderRepository(BaseRepository):
    def table_name(self) -> str: return "orders"
    def from_row(self, row: dict) -> Order: return Order(**row)
```

**2. Template Method pattern — skeleton algorithm + fill-in steps**

**3. You need shared constructors or state initialisation**

---

## Q: How does Python's `ABC` module work?

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str: ...

    @abstractmethod
    def move(self) -> str: ...

    def breathe(self) -> str:          # concrete method
        return "inhaling oxygen"


class Dog(Animal):
    def speak(self) -> str:
        return "Woof"

    def move(self) -> str:
        return "running"


class Cat(Animal):
    def speak(self) -> str:
        return "Meow"
    # Forgot to implement move() — will fail at instantiation


Animal()  # TypeError: Can't instantiate abstract class Animal
Cat()     # TypeError: Can't instantiate abstract class Cat with abstract method move
Dog()     # OK — all abstract methods implemented
```

**Abstract properties:**
```python
class Shape(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...      # subclasses must define name as a property

class Circle(Shape):
    @property
    def name(self) -> str:
        return "Circle"
```

---

## Q: What is Python's `Protocol` and how is it different from ABC?

**A:** `Protocol` implements **structural subtyping** — also called "static duck typing." A class satisfies a Protocol if it has the right methods, regardless of whether it explicitly inherits from the Protocol.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...
    def resize(self, factor: float) -> None: ...


class Circle:             # does NOT inherit from Drawable
    def draw(self) -> None:
        print("Drawing circle")

    def resize(self, factor: float) -> None:
        self.radius *= factor


class Button:             # does NOT inherit from Drawable
    def draw(self) -> None:
        print("Drawing button")

    def resize(self, factor: float) -> None:
        self.width *= factor


def render(component: Drawable) -> None:   # accepts anything with draw() and resize()
    component.draw()

render(Circle())    # works — mypy verifies structural compatibility
render(Button())    # works
isinstance(Circle(), Drawable)   # True — with @runtime_checkable
```

**ABC requires explicit inheritance. Protocol does not.**

| | ABC | Protocol |
|---|---|---|
| Subclassing required? | Yes — must `class Dog(Animal)` | No — structural match is enough |
| Checked at | Instantiation (runtime) | Type checking (mypy) + optionally runtime |
| Best for | Enforcing subclass implementation | Type-safe duck typing |
| Use case | Frameworks, template method | Library code, accepting arbitrary types |

---

## Q: Interface Segregation — how does this relate?

**A:** The Interface Segregation Principle (ISP) says: don't force a class to implement methods it doesn't use. Prefer many small, focused interfaces over one fat interface.

```python
# FAT interface — violation
class Worker(ABC):
    @abstractmethod
    def work(self) -> None: ...

    @abstractmethod
    def eat(self) -> None: ...

    @abstractmethod
    def sleep(self) -> None: ...

class Robot(Worker):
    def work(self) -> None: ...
    def eat(self) -> None:    # Robot doesn't eat — forced to implement anyway
        raise NotImplementedError
    def sleep(self) -> None:  # Robot doesn't sleep — same problem
        raise NotImplementedError
```

```python
# Segregated interfaces — correct
class Workable(Protocol):
    def work(self) -> None: ...

class Eatable(Protocol):
    def eat(self) -> None: ...

class Sleepable(Protocol):
    def sleep(self) -> None: ...

class Human:
    def work(self): ...
    def eat(self): ...
    def sleep(self): ...

class Robot:
    def work(self): ...        # only implements Workable — no fake eat/sleep
```

---

## Real-World: Python's `collections.abc`

Python's standard library is the best example of well-designed abstract base classes:

| ABC | Abstract Methods | Mixin Methods Provided |
|---|---|---|
| `Container` | `__contains__` | — |
| `Iterable` | `__iter__` | — |
| `Sized` | `__len__` | — |
| `Sequence` | `__getitem__`, `__len__` | `__contains__`, `__iter__`, `__reversed__`, `index`, `count` |
| `MutableSequence` | `__getitem__`, `__setitem__`, `__delitem__`, `__len__`, `insert` | `append`, `clear`, `reverse`, `extend`, `pop`, `remove`, `__iadd__` |
| `Mapping` | `__getitem__`, `__len__`, `__iter__` | `__contains__`, `keys`, `values`, `items`, `get`, `__eq__` |

You implement 2-5 abstract methods and get 10+ concrete methods for free.

---

## Code Example: Both Together

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Protocol — pure contract, no hierarchy needed
class Flyable(Protocol):
    def fly(self, altitude: float) -> None: ...
    def land(self) -> None: ...

# Abstract class — shared implementation for related classes
class Aircraft(ABC):
    def __init__(self, model: str, max_altitude: float):
        self.model = model
        self.max_altitude = max_altitude
        self._current_altitude: float = 0.0

    def fly(self, altitude: float) -> None:       # concrete — shared logic
        if altitude > self.max_altitude:
            raise ValueError(f"Cannot exceed max altitude {self.max_altitude}m")
        self._current_altitude = altitude
        self._engage_propulsion()                 # delegates to subclass

    def land(self) -> None:
        self._current_altitude = 0.0
        self._cut_propulsion()

    @abstractmethod
    def _engage_propulsion(self) -> None: ...     # subclass fills in

    @abstractmethod
    def _cut_propulsion(self) -> None: ...


class Jet(Aircraft):
    def _engage_propulsion(self) -> None:
        print(f"{self.model} jet engines engaged at {self._current_altitude}m")

    def _cut_propulsion(self) -> None:
        print(f"{self.model} jet engines cut")

# Jet satisfies the Flyable Protocol without explicitly inheriting from it
def launch(vehicle: Flyable) -> None:
    vehicle.fly(10000)

launch(Jet("Boeing 737", 12500))   # works via Protocol
```

---

## Interview Q&A

**Q: Can an abstract class implement an interface?**
> Yes. An abstract class can declare that it implements an interface (inherits from it) while leaving some interface methods abstract for concrete subclasses to fill in. This is the "partial implementation" pattern: the abstract class provides what all subclasses will share; each concrete subclass provides the rest.

**Q: What happens if a class inherits from two ABCs that both define the same abstract method?**
> It's fine — the concrete class must implement the method once, satisfying both ABCs. Python's MRO ensures only one implementation is looked up. If both ABCs provide concrete implementations of the same method, MRO determines which one is used.

**Q: When would you use `@runtime_checkable` on a Protocol?**
> When you need `isinstance()` checks at runtime — for example, in a framework that accepts plugins. Without it, Protocol is only useful for static type checking (mypy). Note: `@runtime_checkable` only checks method existence, not signatures.

**Q: Python has no `interface` keyword. How do you enforce a pure interface?**
> Use a class that inherits from `ABC` with only `@abstractmethod` methods and no concrete methods or state — this is functionally equivalent to a Java interface. Or use `Protocol` for structural typing that doesn't require explicit inheritance.