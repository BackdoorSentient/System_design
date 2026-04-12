# 68. Interface Segregation Principle (ISP)

**Section:** LLD — SOLID Principles

---

## What is it?

> **Clients should not be forced to depend on interfaces they do not use.**

Robert C. Martin.

Prefer many small, focused interfaces over one large, general-purpose interface. A *client* (the class that uses the interface) should only know about the methods it actually calls.

---

## Why does it matter?

When a client depends on a fat interface:
- It is coupled to methods it never uses
- Changes to those unused methods force the client to recompile/retest
- Implementing classes are forced to provide stub/empty implementations of methods that don't logically apply to them
- Mocking in tests is harder — you must mock 20 methods when you only need 2

ISP reduces **unnecessary coupling** between clients and the code they depend on.

---

## The Fat Interface Problem

```python
class Worker(ABC):
    @abstractmethod
    def work(self) -> None: ...

    @abstractmethod
    def eat(self) -> None: ...

    @abstractmethod
    def sleep(self) -> None: ...
```

This seems fine for a human employee. But:

```python
class Robot(Worker):
    def work(self) -> None:
        print("assembling parts")

    def eat(self) -> None:
        raise NotImplementedError("Robots don't eat")  # Forced stub

    def sleep(self) -> None:
        raise NotImplementedError("Robots don't sleep")  # Forced stub
```

`Robot` is forced to implement two methods that don't logically apply. This is an ISP violation.

**The tell**: if you're writing `raise NotImplementedError` or `pass` in an interface method, ISP is probably being violated.

---

## How to Fix It — Split into Role Interfaces

```python
from abc import ABC, abstractmethod

class Workable(ABC):
    @abstractmethod
    def work(self) -> None: ...

class Eatable(ABC):
    @abstractmethod
    def eat(self) -> None: ...

class Sleepable(ABC):
    @abstractmethod
    def sleep(self) -> None: ...

# Human needs all three
class HumanWorker(Workable, Eatable, Sleepable):
    def work(self): print("coding")
    def eat(self): print("eating lunch")
    def sleep(self): print("sleeping 8 hours")

# Robot only needs one
class Robot(Workable):
    def work(self): print("assembling parts")
```

Each class implements exactly the interfaces it logically satisfies. No stubs, no `NotImplementedError`.

---

## Role Interfaces vs Header Interfaces

| | Role Interface (ISP) | Header Interface |
|---|---|---|
| Definition | Small, focused on one use case | Exposes everything a class does |
| Size | 1–3 methods | 10–20+ methods |
| Coupling | Caller depends only on what it uses | Caller depends on everything |
| Testing | Mock 1–2 methods | Mock all 10–20 |
| Example | `Readable`, `Writable`, `Seekable` | `FileSystem` with all operations |

Prefer role interfaces. A class can implement multiple role interfaces.

---

## Classic Example — The Printer

```python
# FAT INTERFACE — ISP VIOLATION
class MultiFunctionDevice(ABC):
    @abstractmethod
    def print(self, document: str) -> None: ...

    @abstractmethod
    def scan(self, document: str) -> str: ...

    @abstractmethod
    def fax(self, document: str, number: str) -> None: ...

class SimplePrinter(MultiFunctionDevice):
    def print(self, document: str) -> None:
        print(document)

    def scan(self, document: str) -> str:
        raise NotImplementedError("This printer cannot scan")

    def fax(self, document: str, number: str) -> None:
        raise NotImplementedError("This printer cannot fax")
```

**ISP Fix:**

```python
class Printable(ABC):
    @abstractmethod
    def print(self, document: str) -> None: ...

class Scannable(ABC):
    @abstractmethod
    def scan(self, document: str) -> str: ...

class Faxable(ABC):
    @abstractmethod
    def fax(self, document: str, number: str) -> None: ...

class SimplePrinter(Printable):
    def print(self, document: str) -> None:
        print(document)

class OfficePrinter(Printable, Scannable, Faxable):
    def print(self, document: str) -> None: ...
    def scan(self, document: str) -> str: ...
    def fax(self, document: str, number: str) -> None: ...
```

`SimplePrinter` implements exactly what it supports. `OfficePrinter` implements all three.

---

## ISP in Python with Protocol (Structural Typing)

Python's `typing.Protocol` makes ISP natural — you don't even need explicit inheritance.

```python
from typing import Protocol

class Readable(Protocol):
    def read(self) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

class Seekable(Protocol):
    def seek(self, offset: int) -> None: ...

# Any class with a read() method satisfies Readable — no explicit declaration
class NetworkStream:
    def read(self) -> bytes:
        return self.socket.recv(4096)
    # No write() or seek() — that's fine, it satisfies Readable

def process_data(source: Readable) -> None:
    data = source.read()
    # Only depends on read() — not on the full file/stream interface
```

The function `process_data` depends only on `Readable`. It doesn't care about writing, seeking, or anything else.

---

## ISP and Coupling Reduction

Without ISP:
```
OrderService → OrderRepository (with 15 methods)
# OrderService only calls: find_by_id(), save()
# But it's compiled against all 15 methods
# Any change to the other 13 methods forces recompilation of OrderService
```

With ISP:
```
OrderService → OrderReader (find_by_id)
             → OrderWriter (save)
# Now changes to deletion, archiving, reporting methods don't affect OrderService
```

---

## ISP at API Level — Backend for Frontend (BFF)

ISP applied to REST APIs:

A single `/user` endpoint that returns 50 fields satisfies nobody perfectly:
- **Mobile client** needs 5 fields (name, avatar, badge count)
- **Admin panel** needs 30 fields
- **Analytics service** needs 10 fields

The **BFF pattern** (Backend for Frontend) is ISP for APIs:
- `/api/mobile/user` — returns 5 fields optimised for mobile
- `/api/admin/user` — returns full user object
- `/api/analytics/user` — returns analytics-relevant fields only

Each client depends only on the interface it needs.

---

## ISP and Testability

Smaller interfaces are dramatically easier to mock.

```python
# Fat interface — need to mock 8 methods to test one thing
class UserRepository(ABC):
    def find_by_id(self): ...
    def find_by_email(self): ...
    def save(self): ...
    def delete(self): ...
    def find_all(self): ...
    def count(self): ...
    def exists_by_email(self): ...
    def bulk_save(self): ...

# Test for AuthService needs to mock ALL of UserRepository
# even though AuthService only calls find_by_email()

# With ISP
class UserLookup(Protocol):
    def find_by_email(self, email: str) -> User | None: ...

class AuthService:
    def __init__(self, user_lookup: UserLookup):
        self.user_lookup = user_lookup

# Test — mock has exactly 1 method
class MockUserLookup:
    def find_by_email(self, email: str) -> User | None:
        return User("alice", "alice@example.com") if email == "alice@example.com" else None

def test_auth():
    svc = AuthService(MockUserLookup())
    assert svc.authenticate("alice@example.com", "password")
```

---

## Violation in Real Codebases

### Django Class-Based Views

Django's `UpdateView` and `DeleteView` mixins force a view to implement `get_object()`, `get_queryset()`, and model-level permissions even for views that only need to display data.

Developers often work around this with `super()` overrides that return `None` or raise `PermissionDenied` — a classic ISP violation signal.

### Java's `Closeable` vs `AutoCloseable`

Java recognised ISP early: `Closeable` (checked exception) and `AutoCloseable` (unchecked) are separate interfaces. Classes that need `close()` with checked exceptions and those without are not forced into the same contract.

---

## Trade-offs

| Pro | Con |
|---|---|
| Reduces unnecessary coupling | More interfaces to maintain |
| Stubs and NotImplementedError disappear | Can be over-applied to trivial single-implementor cases |
| Smaller mocks in tests | Requires thinking about client roles upfront |
| Changes ripple less (only affected clients recompile) | Finding the right interface granularity takes experience |

---

## ISP vs SRP

| SRP | ISP |
|---|---|
| About the class **being implemented** | About the **client** depending on the interface |
| "A class should have one reason to change" | "Clients should only depend on what they use" |
| Splits implementation classes | Splits interfaces/protocols |
| Applied to: classes, functions | Applied to: interfaces, protocols, APIs |

They complement each other: SRP produces cohesive classes; ISP produces focused interfaces for clients to depend on.

---

## Interview Q&A

**Q: What is ISP in one sentence?**
A: Clients should not be forced to depend on methods they don't use — split fat interfaces into focused role interfaces.

**Q: What's a fat interface and why is it a problem?**
A: A fat interface has too many methods. Implementing classes must provide stubs for methods that don't apply. Clients are coupled to the entire interface even if they only call one method. Changes to unused parts force unnecessary retesting.

**Q: How does Python's Protocol relate to ISP?**
A: `Protocol` enables structural typing — a class satisfies a Protocol if it has the right methods, without explicit declaration. You naturally define small, focused Protocols (Readable, Writable) rather than one large abstract base class, making ISP the default approach.

**Q: How does ISP apply to REST APIs?**
A: The BFF pattern — different endpoints (or different payload shapes) for different clients. Each client gets exactly the data it needs, not a superset. Mobile gets a lightweight response, admin gets full data.

**Q: What's the tell that ISP is being violated?**
A: `raise NotImplementedError` or `pass` in interface implementations. Also: a test that mocks 10 methods but only exercises 1.

**Q: How is ISP different from SRP?**
A: SRP is about the implementor (the class that provides behaviour). ISP is about the caller (the client that uses the interface). SRP keeps classes cohesive; ISP keeps interfaces focused on client needs.

---

## Numbers to Remember

- A well-designed role interface has 1–3 methods
- A fat interface with 15+ methods that forces 10+ stubs is a clear ISP violation
- BFF pattern reduces mobile payload size by 60–90% vs a full admin response — ISP applied to APIs
- Python Protocol: structural typing, zero inheritance needed, ideal for ISP