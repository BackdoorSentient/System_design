# 59. Encapsulation
**Section:** LLD — OOP Fundamentals

---

## What is Encapsulation?

**Encapsulation** is the bundling of data (attributes) and the methods that operate on that data within a single class, while **restricting direct external access** to the internal state.

Two goals:
1. **Data hiding** — prevent invalid state from being set externally
2. **Implementation hiding** — callers don't need to know *how* something works, only *what* interface to use

---

## Q: What are access modifiers and how do Python and Java differ?

**A:**

| Modifier | Java | Python Convention | Meaning |
|---|---|---|---|
| Public | `public` | no prefix | Accessible from anywhere |
| Protected | `protected` | `_name` (single underscore) | Accessible in class + subclasses (convention only in Python) |
| Private | `private` | `__name` (double underscore) | Class-only access (enforced by compiler in Java; name-mangled in Python) |

**Java enforces access at compile time:**
```java
public class User {
    private String passwordHash;   // compiler error if accessed from outside

    public boolean checkPassword(String raw) {
        return hash(raw).equals(this.passwordHash);
    }
}
```

**Python uses convention + name mangling:**
```python
class User:
    def __init__(self, email: str, password: str):
        self.email = email               # public
        self._role = "user"              # protected — convention only
        self.__password_hash = hash(password)  # "private" — name mangled

    def check_password(self, raw: str) -> bool:
        return hash(raw) == self.__password_hash
```

**Name mangling:** `__password_hash` becomes `_User__password_hash` at the bytecode level. It's not truly private — you can still access it — but it signals intent and prevents accidental override in subclasses.

```python
u = User("alice@example.com", "secret")
print(u._User__password_hash)   # accessible but explicitly "you're doing something wrong"
print(u.__password_hash)        # AttributeError — name has been mangled
```

---

## Q: Why does encapsulation matter? What problems does it solve?

**A:**

### 1. Prevents Invalid State

Without encapsulation:
```python
# No protection
class BankAccount:
    def __init__(self, balance):
        self.balance = balance

acc = BankAccount(100)
acc.balance = -9999999   # invalid state — nothing stops this
```

With encapsulation:
```python
class BankAccount:
    def __init__(self, balance: float):
        if balance < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = balance

    def withdraw(self, amount: float) -> None:
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount

    @property
    def balance(self) -> float:
        return self._balance
    # No setter — balance is read-only externally
```

### 2. Hides Implementation Details

Callers are decoupled from the internals. You can refactor how `_balance` is stored (e.g., switch from float to Decimal for precision) without changing the public API.

### 3. Reduces Coupling

Components that only interact through a defined interface can change independently. This is the foundation of modular, maintainable systems.

---

## Q: How do getters and setters work? What's the `@property` decorator in Python?

**A:** Getters and setters provide **controlled access** — read or write private state with validation/transformation logic.

**Java style:**
```java
public class Employee {
    private int age;

    public int getAge() { return age; }

    public void setAge(int age) {
        if (age < 18 || age > 100)
            throw new IllegalArgumentException("Invalid age: " + age);
        this.age = age;
    }
}
```

**Python `@property` — cleaner syntax, same idea:**
```python
class Employee:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age           # triggers the setter

    @property
    def age(self) -> int:
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        if not (18 <= value <= 100):
            raise ValueError(f"Invalid age: {value}")
        self._age = value

    @property
    def is_senior(self) -> bool:
        return self._age >= 60   # computed property — no stored attribute
```

`@property` lets you start with a public attribute and add validation later **without changing the caller's code**. This is the key advantage over Java's getters/setters from day one.

---

## Q: What is name mangling in Python? Is `__attr` truly private?

**A:** **No — it is not truly private.** Python transforms `__attr` into `_ClassName__attr` at compile time (bytecode generation). This is name mangling.

**Why it exists:**
- Prevents subclass attribute name clashes (not access control)
- If `Dog` inherits from `Animal`, `Animal.__name` and `Dog.__name` won't collide because they become `_Animal__name` and `_Dog__name`

```python
class Animal:
    def __init__(self):
        self.__internal_id = "A001"   # becomes _Animal__internal_id

class Dog(Animal):
    def __init__(self):
        super().__init__()
        self.__internal_id = "D001"   # becomes _Dog__internal_id — no collision!

d = Dog()
print(d._Animal__internal_id)  # "A001"
print(d._Dog__internal_id)     # "D001"
```

> Convention: Use `_single_underscore` for "protected" (don't touch unless you know what you're doing). Use `__double_underscore` only when you specifically need to prevent subclass name collision.

---

## Q: What are class invariants and how does encapsulation enforce them?

**A:** A **class invariant** is a condition that must always be true for valid objects of that class.

Examples:
- `BankAccount`: `balance >= 0`
- `Circle`: `radius > 0`
- `DateRange`: `start_date <= end_date`

Encapsulation enforces invariants by ensuring all state changes go through methods that check the invariant:

```python
class DateRange:
    def __init__(self, start: date, end: date):
        self._start = start
        self._end = end
        self._validate()

    def _validate(self) -> None:
        if self._start > self._end:
            raise ValueError(f"start {self._start} must be <= end {self._end}")

    @property
    def start(self) -> date:
        return self._start

    @start.setter
    def start(self, value: date) -> None:
        if value > self._end:
            raise ValueError("start cannot be after end")
        self._start = value

    @property
    def duration_days(self) -> int:
        return (self._end - self._start).days
```

---

## Q: What is the "Tell, Don't Ask" principle?

**A:** Instead of **asking** an object for its data and then making decisions externally, **tell** the object what to do and let it decide internally.

**Violation — Ask style (breaks encapsulation):**
```python
# Caller asks for state and makes decisions
if account.balance >= amount:
    account.balance -= amount
    print("Withdrawn")
else:
    print("Insufficient funds")
```

**Correct — Tell style (respects encapsulation):**
```python
# Caller tells the object what to do, object decides
try:
    account.withdraw(amount)
    print("Withdrawn")
except ValueError as e:
    print(f"Failed: {e}")
```

The logic that decides whether to allow withdrawal **belongs to the class that owns the data**. Moving it outside creates coupling and duplicates the validation everywhere the caller is used.

---

## Q: How does returning mutable objects break encapsulation?

**A:** If a getter returns a direct reference to a mutable internal object, callers can modify the internal state without going through the class's validation logic.

```python
class ShoppingCart:
    def __init__(self):
        self._items = []

    def get_items(self):
        return self._items          # DANGEROUS — returns direct reference

cart = ShoppingCart()
items = cart.get_items()
items.append("malicious_item")      # bypasses any add_item() validation!
```

**Fix: return a defensive copy or immutable view:**
```python
class ShoppingCart:
    def get_items(self) -> list:
        return list(self._items)    # defensive copy — caller gets a snapshot

    def get_items_readonly(self) -> tuple:
        return tuple(self._items)   # immutable view — can't be modified
```

---

## Real-World Example: `User` Password Encapsulation

Never store or expose raw passwords. Encapsulate the hashing logic inside the class:

```python
import hashlib
import os

class User:
    def __init__(self, email: str, raw_password: str):
        self.email = email
        self._password_hash = self._hash_password(raw_password)
        # The hash is private — no getter that returns it

    def _hash_password(self, raw: str) -> str:
        salt = os.urandom(32)
        key = hashlib.pbkdf2_hmac("sha256", raw.encode(), salt, 100_000)
        return salt.hex() + key.hex()

    def verify_password(self, raw: str) -> bool:
        # Returns bool — never exposes the hash
        salt = bytes.fromhex(self._password_hash[:64])
        key = bytes.fromhex(self._password_hash[64:])
        new_key = hashlib.pbkdf2_hmac("sha256", raw.encode(), salt, 100_000)
        return new_key == key

    def change_password(self, old: str, new: str) -> None:
        if not self.verify_password(old):
            raise PermissionError("Current password incorrect")
        if len(new) < 8:
            raise ValueError("New password too short")
        self._password_hash = self._hash_password(new)
```

---

## Real-World Example: Java's `LocalDate`

`LocalDate` encapsulates calendar logic. You can't create an invalid date:
- `LocalDate.of(2024, 13, 1)` → `DateTimeException` — month 13 doesn't exist
- `LocalDate.of(2024, 2, 30)` → `DateTimeException` — Feb 30 doesn't exist
- The internal representation (how the date is stored) is hidden — you interact only through the API

---

## Encapsulation vs Information Hiding

| | Encapsulation | Information Hiding |
|---|---|---|
| Definition | Bundling data + methods in one unit | Hiding *how* something is implemented |
| Question answered | "Are data and behaviour together?" | "Does the caller need to know the internals?" |
| Relationship | Mechanism | Principle |
| Example | `BankAccount` class holds balance and withdraw logic together | Caller doesn't know if balance is `float` or `Decimal` |

Both work together. Encapsulation is how you *achieve* information hiding.

---

## Interview Q&A

**Q: What's the difference between `_name` and `__name` in Python?**
> `_name` is a convention meaning "don't access this from outside unless you know what you're doing." `__name` triggers name mangling — it becomes `_ClassName__name` to prevent accidental subclass collisions. Neither is truly private; Python doesn't enforce access control at runtime.

**Q: Can you have encapsulation without access modifiers?**
> Yes. Encapsulation is about bundling data and behaviour and protecting state through a defined interface. You can achieve this with conventions (Python), design discipline, or code review — the language doesn't need to enforce it with keywords.

**Q: When would you skip a setter and make an attribute read-only?**
> When the attribute should only be set at construction and never changed — like a UUID, creation timestamp, or account number. A `@property` with no setter makes the intent explicit: this value is fixed after object creation.

**Q: How do Java records and Python dataclasses relate to encapsulation?**
> Java records and Python `@dataclass` with `frozen=True` create immutable value objects. All attributes are effectively read-only after construction, which is a strong form of encapsulation. They're appropriate for data transfer objects (DTOs), value objects, and configuration holders.