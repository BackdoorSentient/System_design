# 69. Dependency Inversion Principle (DIP)

**Section:** LLD — SOLID Principles

---

## What is it?

> **A. High-level modules should not depend on low-level modules. Both should depend on abstractions.**
> **B. Abstractions should not depend on details. Details should depend on abstractions.**

Robert C. Martin.

The "inversion" refers to reversing the traditional dependency direction:

**Traditional (bad):**
```
OrderService (high-level) → MySQLOrderRepository (low-level, concrete)
```

**Inverted (good):**
```
OrderService (high-level) → OrderRepository (abstraction) ← MySQLOrderRepository (low-level)
```

Now both depend on the abstraction. Neither depends on the other directly.

---

## Why does it matter?

When a high-level module directly depends on a low-level module:
- **Changing the low-level detail** (e.g., switching from MySQL to PostgreSQL) forces changes in the high-level module
- **Testing the high-level module** requires the real low-level dependency (real DB, real SMTP server, real network)
- **Coupling is tight** — the high-level module cannot be reused without dragging in its low-level dependency
- **Parallel development is blocked** — you can't build `OrderService` until `MySQLOrderRepository` exists

With DIP:
- Swap MySQL for PostgreSQL by writing a new repository class — `OrderService` never changes
- Test `OrderService` with a mock repository — no real DB needed
- `OrderService` can be developed and tested before any database exists

---

## Traditional Dependency Direction (Violation)

```python
# LOW-LEVEL MODULE
class MySQLOrderRepository:
    def save(self, order) -> None:
        mysql.execute("INSERT INTO orders ...", order)

    def find_by_id(self, order_id: int):
        return mysql.query("SELECT * FROM orders WHERE id = ?", order_id)


# HIGH-LEVEL MODULE — depends directly on the concrete low-level class
class OrderService:
    def __init__(self):
        self.repo = MySQLOrderRepository()  # ← hardcoded dependency

    def place_order(self, order) -> None:
        # business logic...
        self.repo.save(order)
```

Problems:
- `OrderService` is tightly coupled to MySQL
- To test `OrderService`, you need a running MySQL instance
- Switching to PostgreSQL means rewriting `OrderService`

---

## Inverted Dependency

```python
from abc import ABC, abstractmethod

# ABSTRACTION — depends on nothing
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order) -> None: ...

    @abstractmethod
    def find_by_id(self, order_id: int): ...


# LOW-LEVEL MODULE — depends on the abstraction
class MySQLOrderRepository(OrderRepository):
    def save(self, order) -> None:
        mysql.execute("INSERT INTO orders ...", order)

    def find_by_id(self, order_id: int):
        return mysql.query("SELECT ...", order_id)


class PostgreSQLOrderRepository(OrderRepository):
    def save(self, order) -> None:
        pg.execute("INSERT INTO orders ...", order)

    def find_by_id(self, order_id: int):
        return pg.query("SELECT ...", order_id)


# HIGH-LEVEL MODULE — depends on the abstraction, not the concrete class
class OrderService:
    def __init__(self, repo: OrderRepository):  # ← depends on abstraction
        self.repo = repo

    def place_order(self, order) -> None:
        # business logic...
        self.repo.save(order)
```

`OrderService` doesn't know about MySQL or PostgreSQL. It only knows about `OrderRepository`.

---

## Dependency Injection (DI)

DIP is the *principle*. **Dependency Injection** is the *mechanism* to achieve it.

Instead of creating dependencies inside a class, you **inject** them from outside.

### Constructor Injection (preferred)

```python
class OrderService:
    def __init__(self, repo: OrderRepository, notifier: Notifier):
        self.repo = repo
        self.notifier = notifier
```

The caller provides the concrete implementations. The service declares what it needs via types.

### Method Injection

```python
def process(self, order, repo: OrderRepository) -> None:
    repo.save(order)
```

Useful when the dependency varies per call.

### Property Injection (avoid where possible)

```python
service = OrderService()
service.repo = MySQLOrderRepository()  # Mutable, no compile-time guarantee
```

Weakest form — no guarantee the dependency is set before use.

---

## DI Frameworks and IoC Containers

In large applications, manually wiring dependencies becomes verbose. DI frameworks automate the wiring.

### IoC Container (Inversion of Control)

You register what you need; the framework provides it.

```python
# Spring (Java) — the classic IoC container
@Service
class OrderService:
    @Autowired
    private OrderRepository repo;  # Spring injects this
```

### FastAPI's `Depends()` System

```python
from fastapi import Depends
from sqlalchemy.orm import Session
from database import get_db

def get_order_repo(db: Session = Depends(get_db)) -> OrderRepository:
    return SQLAlchemyOrderRepository(db)

@router.post("/orders")
def create_order(
    order: OrderRequest,
    repo: OrderRepository = Depends(get_order_repo)  # ← injected
):
    service = OrderService(repo)
    return service.place_order(order)
```

In tests, override the dependency:
```python
app.dependency_overrides[get_order_repo] = lambda: MockOrderRepository()
```

### Python `dependency-injector` library

```python
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    db = providers.Singleton(Database, url="postgresql://...")
    order_repo = providers.Factory(PostgreSQLOrderRepository, db=db)
    order_service = providers.Factory(OrderService, repo=order_repo)

# Wire everything up
container = Container()
service = container.order_service()
```

---

## DIP and Testability

DIP makes unit testing trivial by allowing real dependencies to be swapped with mocks.

```python
class MockOrderRepository(OrderRepository):
    def __init__(self):
        self.saved_orders = []

    def save(self, order) -> None:
        self.saved_orders.append(order)

    def find_by_id(self, order_id: int):
        return next((o for o in self.saved_orders if o.id == order_id), None)

def test_place_order():
    repo = MockOrderRepository()
    service = OrderService(repo=repo)

    order = Order(id=1, items=["book", "pen"])
    service.place_order(order)

    assert len(repo.saved_orders) == 1
    assert repo.saved_orders[0].id == 1
```

No real database. No network. Test runs in milliseconds.

---

## Violation in Real Code

### `requests.get()` called directly

```python
# VIOLATION — impossible to test without real network
def fetch_user_data(user_id: int) -> dict:
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()
```

You can't test this without hitting the real API.

**Fix: inject an HttpClient abstraction**

```python
from typing import Protocol

class HttpClient(Protocol):
    def get(self, url: str) -> dict: ...

def fetch_user_data(user_id: int, client: HttpClient) -> dict:
    return client.get(f"https://api.example.com/users/{user_id}")

# In production
fetch_user_data(1, client=RequestsHttpClient())

# In tests
class MockHttpClient:
    def get(self, url: str) -> dict:
        return {"id": 1, "name": "Alice"}

fetch_user_data(1, client=MockHttpClient())
```

---

## DIP and OCP Together

DIP enables OCP.

```
Without DIP:
OrderService depends on MySQLOrderRepository
→ Adding PostgreSQL support requires modifying OrderService (OCP violated)

With DIP:
OrderService depends on OrderRepository interface
→ Adding PostgreSQL support = write PostgreSQLOrderRepository (OCP satisfied)
→ OrderService never changes
```

They work in tandem: OCP says "don't modify"; DIP provides the mechanism to achieve that.

---

## Full Example — NotificationService

### Before (DIP Violation)

```python
import smtplib

class NotificationService:
    def __init__(self):
        # Hardcoded dependency on SMTP
        self.smtp = smtplib.SMTP("smtp.gmail.com", 587)

    def notify(self, user_email: str, message: str) -> None:
        self.smtp.sendmail("noreply@app.com", user_email, message)
```

Can't test without a real SMTP server. Can't switch to Slack or SMS without rewriting the service.

### After (DIP Applied)

```python
from abc import ABC, abstractmethod

# ABSTRACTION
class MessageSender(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...


# DETAILS — depend on the abstraction
class SmtpSender(MessageSender):
    def send(self, recipient: str, message: str) -> None:
        smtp = smtplib.SMTP("smtp.gmail.com", 587)
        smtp.sendmail("noreply@app.com", recipient, message)

class SlackSender(MessageSender):
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    def send(self, recipient: str, message: str) -> None:
        requests.post(self.webhook_url, json={"channel": recipient, "text": message})

class MockSender(MessageSender):
    def __init__(self):
        self.sent = []

    def send(self, recipient: str, message: str) -> None:
        self.sent.append({"to": recipient, "msg": message})


# HIGH-LEVEL MODULE — depends on abstraction only
class NotificationService:
    def __init__(self, sender: MessageSender):
        self.sender = sender

    def notify(self, recipient: str, message: str) -> None:
        self.sender.send(recipient, message)


# Production
smtp_service = NotificationService(SmtpSender())
slack_service = NotificationService(SlackSender("https://hooks.slack.com/..."))

# Test
def test_notification():
    mock = MockSender()
    svc = NotificationService(mock)
    svc.notify("alice@example.com", "Your order is ready")
    assert mock.sent == [{"to": "alice@example.com", "msg": "Your order is ready"}]
```

---

## Trade-offs

| Pro | Con |
|---|---|
| High-level modules are testable in isolation | More upfront design work |
| Swapping implementations is easy (MySQL → Postgres) | Indirection can make code harder to trace for newcomers |
| DI frameworks automate wiring at scale | DI frameworks add complexity (learning curve, config) |
| High-level and low-level modules can be developed in parallel | Over-engineering for simple scripts and small apps |

---

## Real-World Examples

| System | DIP in Practice |
|---|---|
| Spring (Java) | `@Autowired` injects implementations via IoC container — services depend on interfaces, not beans |
| FastAPI | `Depends()` injects session, config, services — overrideable in tests via `dependency_overrides` |
| Django | `settings.EMAIL_BACKEND` — swap email backend without touching any service code |
| SQLAlchemy | `Session` injected into repositories — swap SQLite (tests) with PostgreSQL (prod) |
| pytest fixtures | `@pytest.fixture` is manual dependency injection — the test "depends" on the fixture, not on the concrete setup |

---

## Interview Q&A

**Q: What is DIP in one sentence?**
A: High-level modules should depend on abstractions, not concrete implementations — and those abstractions should be injected rather than created internally.

**Q: What's the difference between DIP and Dependency Injection?**
A: DIP is the principle (depend on abstractions). Dependency Injection is the technique (pass dependencies from outside rather than creating them inside). DI is one way to achieve DIP.

**Q: What is an IoC container?**
A: A framework that owns object creation and dependency wiring. You declare what you need (via type hints, annotations, or config), and the container provides concrete instances at runtime. Spring, FastAPI's `Depends()`, and Python's `dependency-injector` are examples.

**Q: How does DIP enable testability?**
A: By depending on abstractions, you can inject a mock/stub/fake implementation in tests instead of the real dependency. Tests become fast, isolated, and deterministic — no real DB, no real SMTP, no real network needed.

**Q: How does DIP relate to OCP?**
A: DIP provides the mechanism for OCP. By depending on abstractions, you can extend a system with new implementations without modifying the high-level module. OCP says "don't modify"; DIP gives you the tool to achieve that.

**Q: Show how FastAPI uses DIP.**
A: `Depends()` injects a repository, session, or service into a route handler. In tests, `app.dependency_overrides` replaces the real implementation with a mock — no test database required.

---

## Numbers to Remember

- `requests.get()` called directly = untestable without a real network (avoid in any non-trivial function)
- FastAPI `dependency_overrides` = swap real DB with mock in 2 lines of test setup
- Spring IoC container manages thousands of beans — DIP at scale
- Rule of thumb: if you call a constructor (`MySQLRepo()`) inside a high-level class, DIP is likely violated