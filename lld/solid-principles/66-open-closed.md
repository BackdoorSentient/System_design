# 66. Open/Closed Principle (OCP)

**Section:** LLD — SOLID Principles

---

## What is it?

> **Software entities (classes, modules, functions) should be open for extension but closed for modification.**

Originally stated by Bertrand Meyer (1988), popularised by Robert C. Martin.

- **Open for extension** — you can add new behaviour.
- **Closed for modification** — you do so *without changing existing, tested code*.

The goal: add features by writing new code, not by editing old code.

---

## Why does it matter?

Modifying existing code is risky:
- It can introduce **regressions** — breaking behaviour that was working fine.
- It requires **re-testing** all affected paths, not just the new feature.
- In a deployed system, it means **a new release** even for an unrelated extension.
- If many teams share the class, one team's change impacts everyone.

OCP trades short-term convenience (just add an `elif`) for long-term stability.

---

## How to Achieve OCP

The mechanism is **abstraction** — define a stable interface or abstract class, and write new behaviour as new implementations.

```
Without OCP:
  Caller → ConcreteClassA (edit this when adding ConcreteClassB)

With OCP:
  Caller → AbstractInterface ← ConcreteClassA
                             ← ConcreteClassB (new, no existing code changed)
```

The **caller** is closed for modification. New implementations are extensions.

---

## Classic Violation — The if/elif Chain

```python
def calculate_area(shape: dict) -> float:
    if shape["type"] == "circle":
        return 3.14159 * shape["radius"] ** 2
    elif shape["type"] == "rectangle":
        return shape["width"] * shape["height"]
    elif shape["type"] == "triangle":       # ← modification to add triangle
        return 0.5 * shape["base"] * shape["height"]
    # Every new shape requires modifying this function
```

Every time a new shape is added, this function must be edited. Violates OCP.

---

## OCP Fix — Polymorphism via Abstract Class

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        ...

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        return 3.14159 * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

# Adding Triangle: ZERO modification to existing code
class Triangle(Shape):
    def __init__(self, base: float, height: float):
        self.base = base
        self.height = height

    def area(self) -> float:
        return 0.5 * self.base * self.height

# Caller is closed for modification
def total_area(shapes: list[Shape]) -> float:
    return sum(s.area() for s in shapes)
```

Adding `Triangle` required only a new file, zero changes to existing code.

---

## OCP and the Strategy Pattern

The Strategy pattern is the canonical OCP implementation. The *context* class is closed for modification; new strategies are extensions.

```python
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, price: float) -> float:
        ...

class SeasonalDiscount(DiscountStrategy):
    def apply(self, price: float) -> float:
        return price * 0.80  # 20% off

class LoyaltyDiscount(DiscountStrategy):
    def apply(self, price: float) -> float:
        return price * 0.90  # 10% off

class BulkDiscount(DiscountStrategy):
    def __init__(self, threshold: int):
        self.threshold = threshold

    def apply(self, price: float) -> float:
        return price * 0.75  # 25% off for bulk

# DiscountCalculator is closed for modification
class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy):
        self.strategy = strategy

    def calculate(self, price: float) -> float:
        return self.strategy.apply(price)

# Adding a new discount type: write a new class, touch nothing else
class FlashSaleDiscount(DiscountStrategy):
    def apply(self, price: float) -> float:
        return price * 0.50
```

The `DiscountCalculator` has never needed to change, even as 10 discount types were added.

---

## OCP at System Level — Plugin Architecture

OCP isn't just for classes. At the system level, plugin architectures embody OCP:

- **Payment processors**: Stripe, PayPal, CryptoGateway all implement `PaymentProcessor` interface. The checkout service never changes.
- **Export formats**: CSV, PDF, XLSX all implement `Exporter`. The report service never changes.
- **Storage backends**: S3, GCS, local disk all implement `BlobStore`. The upload handler never changes.

```
Core System (closed for modification)
    ↓
PaymentProcessor interface
    ↑                   ↑                  ↑
StripeGateway    PayPalGateway    CryptoGateway (new, no core changes)
```

---

## OCP Violation in Real Code — Logging Example

```python
class Logger:
    def log(self, message: str, destination: str) -> None:
        if destination == "file":
            with open("app.log", "a") as f:
                f.write(message + "\n")
        elif destination == "database":
            db.insert("logs", {"message": message})
        elif destination == "http":               # ← new requirement, modification
            requests.post("https://log.service/", json={"msg": message})
```

Every new log destination requires modifying the `Logger` class.

**OCP Fix:**
```python
class LogHandler(ABC):
    @abstractmethod
    def emit(self, message: str) -> None: ...

class FileHandler(LogHandler):
    def emit(self, message: str) -> None:
        with open("app.log", "a") as f:
            f.write(message + "\n")

class DatabaseHandler(LogHandler):
    def emit(self, message: str) -> None:
        db.insert("logs", {"message": message})

class HttpHandler(LogHandler):
    def __init__(self, endpoint: str):
        self.endpoint = endpoint

    def emit(self, message: str) -> None:
        requests.post(self.endpoint, json={"msg": message})

class Logger:
    def __init__(self, handlers: list[LogHandler]):
        self.handlers = handlers

    def log(self, message: str) -> None:
        for handler in self.handlers:
            handler.emit(message)
```

Adding a Slack handler: write one new class. The `Logger` never changes.

---

## The Dependency Inversion Link

OCP requires you to depend on **abstractions**, not concretions.

```python
# Violates both OCP and DIP
class NotificationService:
    def __init__(self):
        self.sender = SmtpEmailSender()  # concrete, hardcoded

# Satisfies both OCP and DIP
class NotificationService:
    def __init__(self, sender: MessageSender):  # abstraction
        self.sender = sender
```

By depending on `MessageSender`, adding `SlackSender` or `SMSSender` extends the system with zero modifications to `NotificationService`.

---

## Pragmatic OCP — You Can't Anticipate Everything

> "Apply OCP to the most volatile parts of the system."

You cannot apply OCP everywhere without making the system over-engineered. The principle is most valuable where change is likely:
- **Business rules** (discount types, pricing models) → volatile, apply OCP
- **Integration points** (payment providers, notification channels) → volatile, apply OCP
- **Core algorithms** that never change → pragmatic to leave as-is

Balance OCP with **YAGNI**: don't add an abstraction until you have two implementations, or until a second implementation is likely.

The "Rule of Three": refactor to an OCP-compliant abstraction on the *third* variation, not the first.

---

## Before / After — Full Example

### Before

```python
class DiscountCalculator:
    def calculate(self, price: float, discount_type: str) -> float:
        if discount_type == "seasonal":
            return price * 0.80
        elif discount_type == "loyalty":
            return price * 0.90
        elif discount_type == "bulk":
            return price * 0.75
        # Next sprint: add "flash_sale" → must modify this class
        return price
```

### After

```python
from abc import ABC, abstractmethod

class Discount(ABC):
    @abstractmethod
    def apply(self, price: float) -> float: ...

class SeasonalDiscount(Discount):
    def apply(self, price: float) -> float:
        return price * 0.80

class LoyaltyDiscount(Discount):
    def apply(self, price: float) -> float:
        return price * 0.90

class BulkDiscount(Discount):
    def apply(self, price: float) -> float:
        return price * 0.75

# New requirement — zero modification to existing code
class FlashSaleDiscount(Discount):
    def apply(self, price: float) -> float:
        return price * 0.50

class DiscountCalculator:
    def calculate(self, price: float, discount: Discount) -> float:
        return discount.apply(price)
```

---

## Trade-offs

| Pro | Con |
|---|---|
| New features don't risk regressions in old code | Requires upfront abstraction design |
| Existing tests don't need to change | Over-abstraction if applied too early |
| Teams can add features without touching shared code | More classes, more files |
| Enables plugin/extension architectures | Indirection can make code harder to trace |

---

## Real-World Examples

| System | OCP in Practice |
|---|---|
| Django REST Framework | Serializers, renderers, parsers are pluggable — add a new format without touching core |
| FastAPI middleware | Middleware stack is open for extension (add new middleware) without modifying the framework |
| Pytest | Plugin system (`conftest.py`, hooks) — extend test behaviour without modifying pytest source |
| Webpack / Vite | Loaders and plugins extend the build system without modifying core bundler |
| Payment gateways | Stripe, PayPal, Adyen all implement the same interface — add new provider without touching checkout |

---

## Interview Q&A

**Q: What does "open for extension, closed for modification" mean practically?**
A: When you add new behaviour, you write new code (a new class, a new implementation). You do not edit existing, already-tested code. This prevents regressions and reduces the scope of re-testing.

**Q: How do you implement OCP?**
A: Through abstractions — abstract classes or interfaces. Callers depend on the abstraction. New behaviour is a new concrete implementation. The Strategy, Template Method, and Observer patterns are all expressions of OCP.

**Q: When should you NOT apply OCP?**
A: When the variation is unlikely or when you don't have two concrete cases yet. YAGNI applies — add the abstraction when you have the second implementation, not speculatively on the first.

**Q: How is OCP related to the Strategy pattern?**
A: The Strategy pattern is the most direct implementation of OCP. The context class is closed for modification (never changes); each new strategy is an extension (new class implementing the interface).

**Q: What's a real-world sign that OCP is being violated?**
A: A long `if/elif` chain that grows every time a new feature is added. Also: a core class that requires modification every sprint because new business rules are being added.

---

## Numbers to Remember

- Each new `elif` added to an existing chain = 1 new regression risk
- Strategy pattern adds ~1 new file per variant (acceptable cost for no modification to existing code)
- "Rule of Three" — abstract on the 3rd variation, not the 1st
- Plugin architectures (Webpack, Pytest, Django) support hundreds of extensions with zero core modification