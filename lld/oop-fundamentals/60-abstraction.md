# 60. Abstraction
**Section:** LLD — OOP Fundamentals

---

## What is Abstraction?

**Abstraction** means hiding implementation complexity behind a simple, consistent interface — exposing only what callers need to use the component, not *how* it works internally.

The goal: callers interact with **what** a thing does, not **how** it does it.

```python
# Caller just calls .read() — doesn't know or care if this is disk, network, or memory
with open("data.csv") as f:
    contents = f.read()
```

---

## Q: How is abstraction different from encapsulation?

**A:** They're related but answer different questions:

| | Encapsulation | Abstraction |
|---|---|---|
| Question | *How* do we hide data and bind it to behaviour? | *What* should callers see vs. what should be hidden? |
| Focus | Data hiding, access control | Interface simplification |
| Mechanism | Access modifiers, getters/setters | Abstract classes, interfaces |
| Example | `_balance` is private, changed only via `withdraw()` | Caller calls `withdraw()` — doesn't know if it's SQL, Redis, or in-memory |

**Analogy:** A TV remote is abstraction — you press "volume up" without knowing the IR signal protocol. The battery compartment door being closed is encapsulation — the internals are physically bundled and hidden.

Both work together: encapsulation hides the data, abstraction simplifies the interface.

---

## Q: What is an abstract class? How do you use it in Python?

**A:** An **abstract class** is a class that:
- Cannot be instantiated directly
- Defines a **contract** for subclasses via abstract methods
- Can have concrete methods too (partial implementation, shared logic)

```python
from abc import ABC, abstractmethod
from math import pi

class Shape(ABC):
    def __init__(self, colour: str = "white"):
        self.colour = colour           # concrete attribute

    @abstractmethod
    def area(self) -> float:           # contract — subclasses must implement
        ...

    @abstractmethod
    def perimeter(self) -> float:      # contract — subclasses must implement
        ...

    def describe(self) -> str:         # concrete method — shared implementation
        return (f"{self.colour} {self.__class__.__name__}: "
                f"area={self.area():.2f}, perimeter={self.perimeter():.2f}")


class Circle(Shape):
    def __init__(self, radius: float, colour: str = "white"):
        super().__init__(colour)
        self.radius = radius

    def area(self) -> float:
        return pi * self.radius ** 2

    def perimeter(self) -> float:
        return 2 * pi * self.radius


class Rectangle(Shape):
    def __init__(self, width: float, height: float, colour: str = "white"):
        super().__init__(colour)
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)


# Shape()        → TypeError: Can't instantiate abstract class Shape
# Circle(5.0)    → fine

shapes: list[Shape] = [Circle(5.0), Rectangle(4.0, 6.0)]
for s in shapes:
    print(s.describe())
```

**What happens if a subclass doesn't implement all abstract methods?**
```python
class Hexagon(Shape):
    pass  # doesn't implement area() or perimeter()

h = Hexagon()  # TypeError: Can't instantiate abstract class Hexagon
               # with abstract methods area, perimeter
```

---

## Q: What is an Interface? How does Python implement it?

**A:** An **interface** is a pure contract — only method signatures, no implementation, no state.

**Java** has the `interface` keyword:
```java
interface Serializable {
    byte[] serialize();
    void deserialize(byte[] data);
}
```

**Python has two approaches:**

### 1. ABC with only abstract methods (traditional)
```python
from abc import ABC, abstractmethod

class Serializable(ABC):
    @abstractmethod
    def serialize(self) -> bytes: ...

    @abstractmethod
    def deserialize(self, data: bytes) -> None: ...
```

### 2. `Protocol` — structural subtyping (PEP 544, Python 3.8+)
```python
from typing import Protocol

class Flyable(Protocol):
    def fly(self, altitude: float) -> None: ...
    def land(self) -> None: ...
```

**Key difference:** With `ABC`, classes must explicitly inherit. With `Protocol`, any class that has matching methods satisfies the Protocol — no inheritance needed (duck typing with type hints).

```python
class Bird:
    def fly(self, altitude: float) -> None:
        print(f"Bird flying at {altitude}m")

    def land(self) -> None:
        print("Bird landing")

class Airplane:
    def fly(self, altitude: float) -> None:
        print(f"Plane at {altitude}ft")

    def land(self) -> None:
        print("Plane landing")

def launch(vehicle: Flyable) -> None:   # accepts anything that satisfies Flyable
    vehicle.fly(1000)

launch(Bird())      # works — Bird satisfies Flyable Protocol
launch(Airplane())  # works — no inheritance needed
```

---

## Q: When to use abstract class vs interface?

**A:**

| Use Abstract Class When | Use Interface / Protocol When |
|---|---|
| Subclasses share common implementation | Unrelated classes share a common contract |
| You want to provide a default implementation | You want pure behaviour contract, no shared state |
| Classes form a genuine is-a hierarchy | You want multiple inheritance of behaviour safely |
| You need shared state (attributes) | The contract is "can-do", not "is-a" |

**Concrete example:**

```python
# Abstract class: all HTTP handlers share common parsing logic
class BaseHTTPHandler(ABC):
    def handle(self, request: Request) -> Response:
        parsed = self._parse(request)          # shared concrete method
        return self._process(parsed)           # delegates to subclass

    def _parse(self, request: Request) -> dict:
        # common parsing logic
        ...

    @abstractmethod
    def _process(self, parsed: dict) -> Response:  # subclass fills this in
        ...

# Interface: unrelated classes all need to be loggable
class Loggable(Protocol):
    def to_log_entry(self) -> dict: ...

# User, Order, Payment are unrelated — but all can implement Loggable
# without being forced into a hierarchy
```

---

## Q: What are levels of abstraction? Why do they matter?

**A:** Code should operate at a **single level of abstraction**. Mixing high-level and low-level operations in one function is a design smell.

```python
# BAD — mixes levels of abstraction
def process_order(order_id: str) -> None:
    # Low-level database query
    conn = psycopg2.connect("postgres://...")
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
    row = cursor.fetchone()

    # High-level business logic
    if row["status"] == "pending":
        send_confirmation_email(row["customer_email"])
        apply_discount(row)
```

```python
# GOOD — each function operates at one level
def process_order(order_id: str) -> None:
    order = order_repository.get_by_id(order_id)    # high-level
    if order.is_pending():
        notification_service.send_confirmation(order)  # high-level
        discount_service.apply(order)                  # high-level

def get_by_id(self, order_id: str) -> Order:       # low-level lives here
    row = self._db.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
    return Order.from_row(row)
```

---

## Q: What is a leaky abstraction?

**A:** A leaky abstraction is when **implementation details escape through the interface** — the caller must know things they shouldn't need to know.

**Example:** An ORM (Object-Relational Mapper) that forces callers to think about SQL:
```python
# Leaky — caller must think about SQL transaction semantics
users = db.query("SELECT * FROM users WHERE ROWNUM <= 10")  # Oracle-specific

# Good abstraction — database engine is irrelevant
users = user_repo.get_paginated(page=1, page_size=10)
```

**Law of Leaky Abstractions (Joel Spolsky):** All non-trivial abstractions leak to some degree. The more complex the abstracted system, the more it will leak. TCP abstracts reliable delivery over an unreliable network — but latency, packet loss, and retransmission still affect performance, so you can't ignore the underlying network entirely.

**Implication for design:** When an abstraction leaks, add adapter/wrapper layers rather than exposing internals. If the leak is unavoidable, document it explicitly.

---

## Q: What is the Template Method pattern? (Abstraction in action)

**A:** Abstract class defines an algorithm skeleton in a concrete method. Abstract methods are the "holes" that subclasses fill in. Callers use the abstract class — never the concrete subclass directly.

```python
class DataProcessor(ABC):
    def run(self) -> None:           # template method — algorithm skeleton
        data = self.extract()
        transformed = self.transform(data)
        self.load(transformed)

    @abstractmethod
    def extract(self) -> list: ...

    @abstractmethod
    def transform(self, data: list) -> list: ...

    @abstractmethod
    def load(self, data: list) -> None: ...


class CSVProcessor(DataProcessor):
    def extract(self) -> list:
        return read_csv("input.csv")

    def transform(self, data: list) -> list:
        return [row.strip() for row in data]

    def load(self, data: list) -> None:
        write_csv("output.csv", data)
```

The `run()` method is fixed — the algorithm doesn't change. Only the steps change.

---

## Real-World Examples

| Abstraction | Hidden Complexity | What you see |
|---|---|---|
| `file.read()` | Disk seek, OS buffer, network if NFS | Just bytes |
| `requests.get(url)` | TCP handshake, TLS, DNS, HTTP parsing | Response object |
| `pandas.read_csv()` | File I/O, CSV parsing, dtype inference, memory allocation | DataFrame |
| `SQLAlchemy session.query()` | SQL generation, connection pooling, result mapping | Python objects |
| AWS S3 `put_object()` | Multi-AZ replication, storage classes, checksums | Success/error |

---

## Python's `collections.abc`

Python defines abstract base classes for all its built-in protocols:

| ABC | Abstract Methods | What it means |
|---|---|---|
| `Iterable` | `__iter__` | Can be used in `for` loops |
| `Sequence` | `__getitem__`, `__len__` | Ordered, indexed container |
| `Mapping` | `__getitem__`, `__len__`, `__iter__` | Key-value container like dict |
| `Callable` | `__call__` | Can be called like a function |
| `ContextManager` | `__enter__`, `__exit__` | Can be used with `with` |

```python
from collections.abc import Iterable

def process(items: Iterable) -> None:
    for item in items:
        ...

process([1, 2, 3])      # list — Iterable
process((1, 2, 3))      # tuple — Iterable
process({1, 2, 3})      # set — Iterable
process(range(10))      # range — Iterable
```

---

## Interview Q&A

**Q: What's the difference between abstraction and encapsulation?**
> Encapsulation is about binding data and methods together and controlling access (the mechanism). Abstraction is about simplifying interfaces and hiding complexity from callers (the principle). Encapsulation achieves abstraction, but abstraction is the higher-level goal.

**Q: What's the difference between an abstract class and an interface in Python?**
> An abstract class (via `ABC`) can have state, concrete methods, and constructors. It establishes an is-a relationship. An interface (via `Protocol`) is a pure structural contract — any class that has matching methods satisfies it without explicit inheritance. Use ABC for related classes sharing implementation; use Protocol for unrelated classes sharing a contract.

**Q: What does `TypeError: Can't instantiate abstract class X` mean?**
> A subclass of an abstract class didn't implement one or more abstract methods. Python checks at instantiation time. The fix: either implement all abstract methods in the concrete subclass, or make the subclass abstract too.

**Q: Give a real-world example of a leaky abstraction.**
> N+1 query problem in ORMs. An ORM abstracts SQL as Python objects — but if you iterate over a collection and access a related object on each iteration, the ORM fires N+1 queries instead of 1 JOIN. The caller must be aware of how SQL works to fix this (eager loading). The abstraction leaked.