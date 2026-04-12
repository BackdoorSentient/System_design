# 65. Single Responsibility Principle (SRP)

**Section:** LLD — SOLID Principles

---

## What is it?

> **A class should have only one reason to change.**

Robert C. Martin's formulation is precise: "reason to change" means *actor* — a person, team, or stakeholder who can demand a change. If two different departments can independently force you to modify the same class, that class has two responsibilities.

This is subtly different from "does only one thing." A class can do several closely related things and still have a single responsibility — as long as there is only one actor who cares about all of them.

---

## Why does it matter?

- **Reduces blast radius of changes** — a bug fix or feature for the finance team cannot accidentally break the marketing team's code if they live in separate classes.
- **Enables parallel development** — two teams can modify two classes simultaneously without merge conflicts.
- **Simplifies testing** — a class with one responsibility has fewer dependencies and can be unit-tested in isolation.
- **Improves readability** — a well-named, single-responsibility class (`InvoiceGenerator`) tells you exactly what it does.

---

## The "Reason to Change" Mental Model

Ask: *"Who will ask me to change this class?"*

| Who asks for a change | What they want changed |
|---|---|
| Marketing team | Welcome email wording, subject lines |
| Finance team | PDF invoice layout, tax calculation |
| Security team | Password hashing algorithm |

If the answer is **more than one stakeholder** for a single class, the class has multiple responsibilities.

---

## Violation Example

```python
class UserService:
    def authenticate(self, username, password):
        # Hashed password comparison — owned by Security
        ...

    def send_welcome_email(self, user):
        # SMTP integration — owned by Marketing
        ...

    def generate_pdf_report(self, user):
        # PDF layout, fonts, branding — owned by Finance/Design
        ...
```

Three unrelated stakeholders own this class. Any one of them changing their requirements forces you to touch and re-test the entire `UserService`.

---

## How to Fix It

```python
class AuthService:
    def authenticate(self, username: str, password: str) -> bool:
        # Single owner: Security team
        hashed = hash_password(password)
        return db.query(User).filter_by(username=username, password_hash=hashed).exists()

class UserEmailService:
    def send_welcome_email(self, user: User) -> None:
        # Single owner: Marketing team
        template = load_template("welcome")
        mailer.send(to=user.email, subject="Welcome!", body=template.render(user=user))

class UserReportService:
    def generate_pdf_report(self, user: User) -> bytes:
        # Single owner: Finance/Design team
        return pdf_renderer.render("user_report", context={"user": user})
```

Each class has **one owner, one reason to change**.

---

## SRP at Function Level

SRP applies to functions too, not just classes.

**Violation:**
```python
def process_payment_and_send_email(order):
    charge_card(order.card, order.total)
    send_email(order.user.email, "Payment confirmed")
```

**Fixed:**
```python
def process_payment(order: Order) -> PaymentResult:
    return charge_card(order.card, order.total)

def send_confirmation_email(order: Order) -> None:
    send_email(order.user.email, "Payment confirmed")
```

Rule of thumb: if a function name contains **"and"**, it probably violates SRP.

---

## The God Class Anti-Pattern

A **God Class** is SRP violation taken to the extreme — a single class that "knows too much" and "does too much." Common in legacy codebases where features were added directly to existing classes rather than creating new ones.

```python
class ApplicationManager:
    def handle_user_login(self): ...
    def send_notifications(self): ...
    def generate_reports(self): ...
    def manage_payments(self): ...
    def process_orders(self): ...
    def sync_inventory(self): ...
    def export_to_csv(self): ...
    # 2000 more lines...
```

Symptoms of a God Class:
- The file is 1000+ lines
- It imports 20+ other modules
- Every bug in the system traces back to it
- Nobody on the team fully understands it

---

## SRP and Cohesion

**Cohesion** measures how strongly the methods of a class relate to each other.

- **High cohesion** = every method works on the same data, serves the same purpose — SRP-compliant
- **Low cohesion** = methods are loosely related or unrelated — SRP violation

SRP is essentially a rule to **maximise cohesion**.

```
AuthService
├── authenticate()       → reads user.password_hash
├── generate_token()     → creates JWT for user
└── validate_token()     → validates JWT for user

All methods operate on authentication state. High cohesion. ✅
```

---

## SRP and Testability

A class with one responsibility:
- Has few constructor dependencies
- Can be instantiated without complex mock setup
- Has predictable inputs and outputs

```python
# Easy to test — only dependency is the DB
class AuthService:
    def __init__(self, db: Database):
        self.db = db

    def authenticate(self, username, password) -> bool:
        ...

# Test setup: just mock one thing
def test_authenticate():
    mock_db = MockDatabase(users=[User("alice", hash("secret"))])
    svc = AuthService(mock_db)
    assert svc.authenticate("alice", "secret") is True
```

A class with three responsibilities requires mocking an SMTP server, a PDF renderer, and a database — just to test authentication.

---

## Common Misapplication — Over-Segregation

> SRP says "one reason to change," **not** "one line of code."

Taking SRP too far produces dozens of tiny classes that are hard to navigate:

```python
# Over-engineered — these have no reason to exist independently
class PasswordHasher: ...
class PasswordComparer: ...
class PasswordValidator: ...
class PasswordStrengthChecker: ...
```

The test: could these ever change independently? If not, they belong together.

Balance SRP with **YAGNI** (You Aren't Gonna Need It) — don't split until there are actually two stakeholders.

---

## SRP and the Open-Closed Principle

SRP enables OCP. When responsibilities are separated:
- You can extend `UserEmailService` (add CC/BCC, add tracking pixels) without touching `AuthService`
- You can swap `UserReportService` implementations (PDF → HTML → CSV) without changing anything else

Without SRP, extending one feature risks breaking another.

---

## Before / After — Full Example

### Before (SRP Violation)

```python
class Order:
    def __init__(self, items, user):
        self.items = items
        self.user = user

    def calculate_total(self) -> float:
        # Business logic — Product team
        return sum(item.price for item in self.items)

    def save_to_db(self) -> None:
        # Persistence — Data team
        db.session.add(self)
        db.session.commit()

    def send_confirmation_email(self) -> None:
        # Notification — Marketing team
        mailer.send(self.user.email, "Your order is confirmed!")

    def generate_pdf_invoice(self) -> bytes:
        # Invoice generation — Finance team
        return pdf_engine.render("invoice", order=self)
```

Four stakeholders. Four reasons to change. One class.

### After (SRP Applied)

```python
class Order:
    """Pure domain model. Owned by: Product team."""
    def __init__(self, items: list[Item], user: User):
        self.items = items
        self.user = user

    def calculate_total(self) -> float:
        return sum(item.price for item in self.items)


class OrderRepository:
    """Persistence layer. Owned by: Data/Backend team."""
    def save(self, order: Order) -> None:
        db.session.add(order)
        db.session.commit()

    def find_by_id(self, order_id: int) -> Order:
        return db.session.get(Order, order_id)


class OrderNotifier:
    """Notification logic. Owned by: Marketing team."""
    def send_confirmation(self, order: Order) -> None:
        mailer.send(order.user.email, "Your order is confirmed!")


class InvoiceGenerator:
    """Invoice creation. Owned by: Finance team."""
    def generate_pdf(self, order: Order) -> bytes:
        return pdf_engine.render("invoice", order=order)
```

Each class has one owner, one axis of change.

---

## Trade-offs

| Pro | Con |
|---|---|
| Changes are isolated, low blast radius | More files to navigate |
| Easy to unit test each class | Requires a coordinator to wire them together |
| Teams can work in parallel | More boilerplate for trivial applications |
| High cohesion → readable code | Risk of over-engineering if applied prematurely |

---

## Real-World Examples

| Codebase / Framework | SRP in Practice |
|---|---|
| Django | Models (data), Views (HTTP), Serializers (transformation), Signals (events) — separated by responsibility |
| Spring Boot | `@Repository`, `@Service`, `@Controller` stereotypes enforce SRP by layer |
| FastAPI | Routers, dependencies, schemas, CRUD functions are separate modules |
| Unix philosophy | Every tool does one thing: `grep`, `sort`, `wc` — SRP at program level |

---

## Interview Q&A

**Q: What is SRP in one sentence?**
A: A class should have exactly one reason to change — one stakeholder whose requirements drive its evolution.

**Q: How is SRP different from "a class should do one thing"?**
A: "One reason to change" is about *ownership* (actors/stakeholders), not volume of code. A class can have several related methods and still be SRP-compliant if they all serve the same stakeholder.

**Q: How does SRP relate to cohesion?**
A: SRP maximises cohesion. A cohesive class has all methods related to a single purpose — which means one stakeholder drives all changes.

**Q: What's the God Class anti-pattern?**
A: An extreme SRP violation — one class that knows and does everything. Common in legacy systems where features were bolted on over time. Symptoms: 1000+ line files, 20+ imports, everyone's afraid to touch it.

**Q: Can you apply SRP too aggressively?**
A: Yes. Creating dozens of one-method classes for trivial logic is over-engineering. Split only when there are actually two distinct actors. Balance with YAGNI.

**Q: How does SRP improve testability?**
A: A single-responsibility class has fewer dependencies, so test setup is simpler. You mock one thing, test one behaviour, and assert one outcome.

---

## Numbers to Remember

- A class with 1 responsibility needs ~1–3 mocks in tests
- A God Class in a legacy codebase often has 3,000–10,000 lines
- Splitting a God Class is one of the most common refactoring tasks in senior interviews
- Rule of thumb: if your class has more than 5–7 public methods doing unrelated things, apply SRP