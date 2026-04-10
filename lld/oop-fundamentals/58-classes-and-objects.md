# 58. Classes & Objects
**Section:** LLD — OOP Fundamentals

---

## What is a Class?

A **class** is a blueprint or template that defines:
- **Data** (attributes/fields) — the state an object will hold
- **Behaviour** (methods) — what operations can be performed on that state

A class definition does **not** occupy meaningful heap memory by itself. Memory is allocated only when an instance (object) is created.

```python
class BankAccount:
    interest_rate = 0.05  # class attribute — shared across all instances

    def __init__(self, account_number: str, owner: str, initial_balance: float = 0):
        if initial_balance < 0:
            raise ValueError("Initial balance cannot be negative")
        self.account_number = account_number  # instance attribute
        self.owner = owner
        self._balance = initial_balance       # convention: "protected"

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._balance += amount

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount

    def get_balance(self) -> float:
        return self._balance

    def __repr__(self) -> str:
        return f"BankAccount(account_number={self.account_number!r}, owner={self.owner!r}, balance={self._balance})"
```

---

## What is an Object?

An **object** is a concrete instance of a class — it exists in heap memory and has its own copy of instance attributes.

```python
acc1 = BankAccount("ACC001", "Alice", 1000)
acc2 = BankAccount("ACC002", "Bob", 500)

# acc1 and acc2 are separate objects — independent state
acc1.deposit(200)
print(acc1.get_balance())  # 1200
print(acc2.get_balance())  # 500 — unaffected
```

---

## Q: What is a Constructor? When is it called?

**A:** A constructor is a special method invoked automatically when an object is created. It initialises the object's state.

| Language | Constructor Syntax | Notes |
|---|---|---|
| Python | `__init__(self, ...)` | Called after `__new__` allocates the object |
| Java | `ClassName(...)` | Overloading supported — multiple signatures |
| C++ | `ClassName(...)` | Overloading + initialiser lists |

**Constructor overloading in Python** — Python doesn't support method overloading natively. Use default arguments or class methods as factory constructors:

```python
class BankAccount:
    def __init__(self, account_number, owner, balance=0):
        ...

    @classmethod
    def from_dict(cls, data: dict) -> "BankAccount":
        return cls(data["account_number"], data["owner"], data.get("balance", 0))

    @classmethod
    def zero_balance(cls, account_number, owner) -> "BankAccount":
        return cls(account_number, owner, 0)
```

---

## Q: What is a Destructor? When should you use it?

**A:** A destructor is called when an object is about to be destroyed (goes out of scope or is garbage collected).

| Language | Destructor | Trigger |
|---|---|---|
| Python | `__del__(self)` | CPython: reference count reaches 0; PyPy: non-deterministic |
| C++ | `~ClassName()` | Deterministic — called when scope ends (RAII) |
| Java | `finalize()` | **Deprecated since Java 9** — use `AutoCloseable`/`try-with-resources` |

**When to use destructors:**
- Releasing file handles
- Closing database connections
- Freeing network sockets
- Releasing OS-level resources

```python
class DatabaseConnection:
    def __init__(self, dsn: str):
        self.conn = connect(dsn)

    def __del__(self):
        if self.conn:
            self.conn.close()

    # Better: use context manager protocol
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False
```

> **Warning:** In Python, `__del__` is non-deterministic in non-CPython runtimes. Prefer context managers (`with` statement) for resource cleanup.

---

## Q: Class Attributes vs Instance Attributes — What's the difference?

**A:**

| | Class Attribute | Instance Attribute |
|---|---|---|
| Defined in | Class body (outside `__init__`) | Inside `__init__` (or other methods) via `self`) |
| Memory | Shared — one copy for entire class | Unique per object |
| Access | `ClassName.attr` or `self.attr` | `self.attr` only |
| Use case | Default values, counters, constants | Per-object state |

```python
class Employee:
    company = "Acme Corp"           # class attribute
    _instance_count = 0             # class attribute used as counter

    def __init__(self, name: str, salary: float):
        self.name = name            # instance attribute
        self.salary = salary        # instance attribute
        Employee._instance_count += 1

    @classmethod
    def get_count(cls) -> int:
        return cls._instance_count

e1 = Employee("Alice", 90000)
e2 = Employee("Bob", 85000)
print(Employee.get_count())   # 2
print(e1.company)             # "Acme Corp"

# Assigning to self.company creates a NEW instance attribute, shadowing the class attribute
e1.company = "NewCorp"
print(e1.company)             # "NewCorp" — instance attribute shadows class attribute
print(e2.company)             # "Acme Corp" — unaffected
```

---

## Q: Static Methods vs Instance Methods vs Class Methods — when to use each?

**A:**

| Method Type | First Param | Access | Use Case |
|---|---|---|---|
| Instance method | `self` | Instance + class state | Normal behaviour — operates on object data |
| Class method | `cls` | Class state only | Factory methods, accessing/modifying class attributes |
| Static method | none | Neither | Utility/helper functions logically grouped with the class |

```python
class MathUtils:
    pi = 3.14159

    def scale(self, x):              # instance method — needs self
        return x * self.pi

    @classmethod
    def from_string(cls, s: str):    # factory: "3.5" → MathUtils instance
        return cls()

    @staticmethod
    def add(a, b):                   # utility — no need for self or cls
        return a + b
```

---

## Q: What is `self` / `this`? How does Python handle it differently from Java?

**A:**

- **`self` (Python):** explicit first parameter of every instance method; Python passes the object reference automatically when you call `obj.method()`
- **`this` (Java/C++):** implicit keyword — available inside any instance method without being declared

```python
# Python — explicit
class Dog:
    def bark(self):          # self is explicit
        print(f"{self.name} barks")
```

```java
// Java — implicit
class Dog {
    void bark() {           // 'this' available implicitly
        System.out.println(this.name + " barks");
    }
}
```

The key insight: Python makes the mechanism visible. Java hides it. Both `self` and `this` reference the **same heap-allocated object** that received the method call.

---

## Q: Walk me through memory layout — what happens when you do `a = MyClass(); b = a`?

**A:**

```
Stack                     Heap
──────                    ────────────────────────
a ──────────────────────► [MyClass object @ 0x1A2B]
                           ├── attr1 = "hello"
b ──────────────────────► [same object @ 0x1A2B]
                           └── attr2 = 42
```

1. `MyClass()` calls `__new__` to allocate an object on the **heap**, then `__init__` to initialise it
2. `a` is a **reference (pointer)** stored on the stack — it points to the heap object
3. `b = a` copies the **reference**, not the object — both `a` and `b` now point to the **same object**
4. Mutating `b.attr1 = "world"` is visible through `a.attr1` — they share the same object

This is called **aliasing**. It's a common source of bugs:

```python
a = BankAccount("ACC001", "Alice", 1000)
b = a                   # b is NOT a new account — it's an alias
b.deposit(500)
print(a.get_balance())  # 1500 — a and b are the same object!
```

**Garbage Collection:**
- CPython uses reference counting — object is destroyed when reference count hits 0
- JVM uses tracing GC (generational GC) — objects survive multiple GC cycles if still reachable
- Both `a` and `b` going out of scope drops ref count to 0 → object eligible for collection

---

## Q: Object Identity vs Equality — what's the difference?

**A:**

| Comparison | Python | Java | Meaning |
|---|---|---|---|
| Identity | `is` | `==` (for primitives) | Same memory address — are they the same object? |
| Equality | `==` | `.equals()` | Same value — are they logically equivalent? |

```python
a = BankAccount("ACC001", "Alice", 1000)
b = a
c = BankAccount("ACC001", "Alice", 1000)  # same data, different object

print(a is b)   # True  — same object in memory
print(a is c)   # False — different objects
print(a == c)   # False — unless __eq__ is overridden!
```

**Overriding equality:**

```python
class BankAccount:
    def __eq__(self, other) -> bool:
        if not isinstance(other, BankAccount):
            return NotImplemented
        return self.account_number == other.account_number

    def __hash__(self) -> int:
        return hash(self.account_number)
        # Rule: if __eq__ is defined, __hash__ must be defined too
        # objects that compare equal must have the same hash
```

> **Common trap in Java:** `String a = "hello"; String b = new String("hello"); a == b` is `false` because `==` checks identity (memory address). Always use `.equals()` for value comparison in Java.

---

## Q: Mutable vs Immutable Objects — what are the implications?

**A:**

| | Mutable | Immutable |
|---|---|---|
| Can be changed after creation? | Yes | No — operations create new objects |
| Python examples | `list`, `dict`, `set`, user-defined classes | `int`, `float`, `str`, `tuple`, `frozenset` |
| Java examples | `ArrayList`, `HashMap`, custom objects | `String`, `Integer`, `LocalDate` |

```python
# Immutable — reassignment creates a new object
x = 5
id_before = id(x)
x += 1
id_after = id(x)
print(id_before == id_after)  # False — x now points to a NEW int object

# Mutable — in-place modification
lst = [1, 2, 3]
id_before = id(lst)
lst.append(4)
id_after = id(lst)
print(id_before == id_after)  # True — same list object, modified in place
```

**Implications for function parameters:**

```python
def bad_reset(account, amount):
    account = BankAccount("NEW", "X", 0)  # rebinds local variable — caller unaffected
    amount = 0                             # int is immutable — creates new int, caller unaffected

def good_withdraw(account, amount):
    account.withdraw(amount)               # mutates the object — caller DOES see this change
```

---

## Real-World Examples

| Object | Class | Key Attributes | Key Methods |
|---|---|---|---|
| `User` | `User` | `id`, `email`, `password_hash`, `created_at` | `authenticate()`, `change_password()` |
| `Order` | `Order` | `order_id`, `items`, `status`, `total` | `add_item()`, `cancel()`, `calculate_total()` |
| `Connection` | `DBConnection` | `dsn`, `conn`, `is_connected` | `execute()`, `commit()`, `close()` |
| `Thread` | `Thread` | `thread_id`, `target_fn`, `state` | `start()`, `join()`, `interrupt()` |

---

## Interview Q&A

**Q: What happens in memory when you do `a = MyClass(); b = a`?**
> Both `a` and `b` are references on the stack pointing to the same heap-allocated object. Mutating through `b` is visible through `a`. This is aliasing. To get an independent copy, use `copy.copy()` (shallow) or `copy.deepcopy()` (deep).

**Q: Why use class methods as factory constructors?**
> Factory class methods (`@classmethod`) allow multiple named constructors — `User.from_jwt(token)`, `User.from_oauth(data)`, `User.guest()` — each encapsulating different construction logic while still calling `cls(...)` so they work correctly with subclasses.

**Q: Why prefer context managers over `__del__` for resource cleanup?**
> `__del__` is non-deterministic (especially in non-CPython runtimes) and won't run if circular references prevent garbage collection. Context managers (`__enter__`/`__exit__`) are deterministic — cleanup happens when the `with` block exits, even if an exception is raised.

**Q: What's the difference between `__str__` and `__repr__`?**
> `__repr__` is for developers — unambiguous, should ideally be valid Python to recreate the object. `__str__` is for end users — readable display. `str(obj)` falls back to `__repr__` if `__str__` is not defined.

---

## Numbers to Remember

| Fact | Value |
|---|---|
| Python CPython int cache | Integers −5 to 256 are cached — `a = 256; b = 256; a is b` → True |
| Python string interning | Short strings without spaces are usually interned — `is` may return True |
| Object overhead (CPython) | ~56 bytes for an empty object (`sys.getsizeof(object())`) |
| JVM object header | 12–16 bytes per object (mark word + class pointer) |