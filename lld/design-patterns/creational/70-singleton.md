# 70. Singleton

**Section:** LLD — Design Patterns — Creational

---

## What is it?

The Singleton pattern ensures a class has **only one instance** throughout the application's lifetime, and provides a **global access point** to that instance.

The two guarantees:
1. Only one instance is ever created
2. There is a single, well-known way to access it

---

## Why does it matter?

Some resources are expensive to create or must be shared to maintain consistency:

| Use Case | Why Singleton |
|---|---|
| Database connection pool | Pool creation is expensive (~100ms+), all callers must share the same pool |
| Config manager | Read from file/env once at startup, share everywhere — no re-parsing |
| Logger | Single log stream, consistent formatting, no interleaving writes |
| Thread pool | Fixed number of workers, shared task queue — duplicating wastes threads |
| Cache (in-process) | Must be the same object for cache hits to work |

---

## How does it work?

### Classic implementation (Python)

```python
import threading

class DatabasePool:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialised = False
        return cls._instance

    def __init__(self, dsn: str = "postgresql://localhost/mydb", pool_size: int = 10):
        if self._initialised:
            return
        self._initialised = True
        self._dsn = dsn
        self._pool_size = pool_size
        self._connections = []
        self._available = []
        self._lock = threading.Lock()
        self._init_pool()

    def _init_pool(self):
        print(f"[DatabasePool] Creating {self._pool_size} connections to {self._dsn}")
        for i in range(self._pool_size):
            conn = self._create_connection()
            self._connections.append(conn)
            self._available.append(conn)

    def _create_connection(self):
        # In real code: psycopg2.connect(self._dsn)
        return {"id": len(self._connections), "active": False}

    def get_connection(self):
        with self._lock:
            if not self._available:
                raise RuntimeError("Connection pool exhausted")
            conn = self._available.pop()
            conn["active"] = True
            return conn

    def release_connection(self, conn):
        with self._lock:
            conn["active"] = False
            self._available.append(conn)

    @classmethod
    def reset(cls):
        """For testing only — resets singleton state."""
        cls._instance = None


# Usage
pool1 = DatabasePool(dsn="postgresql://localhost/app", pool_size=5)
pool2 = DatabasePool()  # Returns same instance, __init__ skipped
print(pool1 is pool2)   # True
```

### Metaclass-based singleton (cleaner for library code)

```python
class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    instance = super().__call__(*args, **kwargs)
                    cls._instances[cls] = instance
        return cls._instances[cls]


class ConfigManager(metaclass=SingletonMeta):
    def __init__(self):
        self._config = {}
        self._load()

    def _load(self):
        # Load from env / file
        import os
        self._config = {"env": os.getenv("ENV", "dev"), "debug": True}

    def get(self, key: str):
        return self._config.get(key)
```

### Module-level singleton (Python idiom)

```python
# logger.py
import logging

_logger = logging.getLogger("app")
_logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
_logger.addHandler(handler)

# Any module importing this gets the same logger object
# Python's import system caches modules — imports are singletons by default
```

---

## Thread Safety

### The Race Condition

```
Thread A: checks _instance is None  →  True
Thread B: checks _instance is None  →  True   (context switch before A creates)
Thread A: creates instance
Thread B: creates instance           →  Two instances! ❌
```

### Fix: Double-Checked Locking

```python
if cls._instance is None:          # First check (no lock — fast path)
    with cls._lock:
        if cls._instance is None:  # Second check (with lock — safe)
            cls._instance = super().__new__(cls)
```

Without the second check: Thread A and B both pass the first check, both acquire the lock (one after the other), and both create an instance.

### Python's GIL

The GIL means only one thread executes Python bytecode at a time. **Module-level singletons are practically thread-safe** in CPython because the entire assignment `_instance = Class()` happens without a context switch mid-operation.

However:
- Not guaranteed by the language specification
- Doesn't apply to Jython, PyPy with STM, or async contexts
- **Explicit locking is always safer and self-documenting**

---

## Trade-offs

| Concern | Detail |
|---|---|
| ✅ Lazy initialisation | Created only when first needed — not at import time (unless you use module-level) |
| ✅ Encapsulates creation | Subclasses or mocks can override `__new__` / metaclass to return a different instance |
| ❌ Hidden global state | Any code can call `Singleton.get_instance()` — creates implicit coupling |
| ❌ Testing nightmare | State persists across tests. Test A's side effects contaminate Test B |
| ❌ Multithreading complexity | Without care, you get race conditions or unnecessary locking overhead |
| ❌ Anti-pattern in DI systems | If you're using dependency injection, you should inject the single shared instance rather than fetching it globally |

---

## Singleton vs Global Variable

| | Global Variable | Singleton |
|---|---|---|
| Initialisation | At import time (eager) | On first access (lazy) |
| Encapsulation | None — just a value | Creation logic hidden in class |
| Subclassing | Not possible | Possible (`class TestPool(DatabasePool)`) |
| Mockability | Hard | Can override `_instance` in tests |
| Thread safety | None | Can add locking |
| Error on bad state | Silent | Can raise in `__init__` |

---

## Monostate Pattern (Alternative)

Instead of enforcing one instance, allow multiple instances that **share the same state** via class-level attributes:

```python
class AppConfig:
    _shared_state = {}  # All instances share this dict

    def __init__(self):
        self.__dict__ = AppConfig._shared_state  # All instances point to same dict

    def set(self, key, value):
        self._shared_state[key] = value

    def get(self, key):
        return self._shared_state.get(key)

c1 = AppConfig()
c2 = AppConfig()
c1.set("env", "prod")
print(c2.get("env"))  # "prod" — shared state
print(c1 is c2)        # False — different instances
```

Monostate is less confusing in tests because `c1 is c2` is False — but you still need to reset `_shared_state` between tests.

---

## Testing Problem and Fix

```python
# Problem: test order matters
def test_pool_size_5():
    pool = DatabasePool(pool_size=5)
    assert len(pool._connections) == 5

def test_pool_size_10():
    pool = DatabasePool(pool_size=10)
    # ❌ FAILS — same instance from test_pool_size_5, _initialised=True, __init__ skipped
    assert len(pool._connections) == 10


# Fix: expose reset for testing
@pytest.fixture(autouse=True)
def reset_singleton():
    yield
    DatabasePool.reset()  # DatabasePool._instance = None
```

---

## Dependency Injection Alternative

Rather than global access, inject the shared instance:

```python
# Bad: hidden singleton access inside method
class UserService:
    def get_user(self, user_id):
        pool = DatabasePool()  # ← hidden dependency
        conn = pool.get_connection()
        ...

# Good: inject the pool
class UserService:
    def __init__(self, pool: DatabasePool):
        self._pool = pool

# At startup (composition root):
pool = DatabasePool(dsn="...", pool_size=10)
user_service = UserService(pool=pool)

# In tests:
mock_pool = MockDatabasePool()
user_service = UserService(pool=mock_pool)  # clean, no singleton state
```

DI gives you: testability, explicit dependencies, no global state, same "one instance" guarantee (you just wire it once at startup).

---

## Real-World Examples

- **Python's `logging` module**: `logging.getLogger("app")` always returns the same logger for the same name — singleton registry
- **Django settings**: `django.conf.settings` — module-level singleton wrapping the settings object
- **SQLAlchemy `Engine`**: you create it once and pass it everywhere (or use a global) — recommended to create once per process
- **Flask `app` object**: application instance shared across all views and extensions
- **`None`, `True`, `False` in Python**: these are singletons — `None is None` is always `True`

---

## Interview Q&A

**Q: What problem does the Singleton pattern solve?**

A: It ensures that only one instance of a resource-heavy or state-sharing object (like a DB connection pool or config manager) is created, and provides a controlled global access point. It's especially useful when duplicate instantiation would cause bugs (double-logging, pool exhaustion) or waste resources.

**Q: Why is Singleton considered an anti-pattern by many engineers?**

A: It introduces hidden global state — any class can call `Singleton.get_instance()` without declaring the dependency. This makes code tightly coupled, hard to test (state leaks between tests), and violates the Dependency Inversion Principle. Modern frameworks prefer DI: wire the single shared instance at startup and inject it explicitly.

**Q: How do you make a Singleton thread-safe in Python?**

A: Use double-checked locking with `threading.Lock()`. Check if `_instance is None` before acquiring the lock (cheap fast path), then check again after acquiring (safe slow path). Python's GIL provides some protection, but is not a language guarantee — always use explicit locking for correctness.

**Q: How do you test code that uses Singletons?**

A: Expose a `reset()` class method that sets `_instance = None`, and call it in a pytest `autouse` fixture. Or better — refactor to DI and inject a mock instead, eliminating the singleton dependency from the test target entirely.

**Q: Singleton vs module-level instance in Python — which should you use?**

A: For most cases, a module-level instance is simpler and idiomatic Python: `pool = DatabasePool()` at the bottom of `db.py`, imported wherever needed. The class-based singleton makes sense when: you need lazy initialisation, you want to subclass or mock the singleton class, or you want to control instantiation explicitly (e.g., in a framework).

---

## Numbers to Remember

- Thread pool typical size: **2× CPU cores** for CPU-bound, **10–50×** for I/O-bound
- DB connection pool typical size: **10–20 connections** per application instance (Postgres default `max_connections = 100`)
- Double-checked locking overhead: near-zero after first check (lock not acquired on the hot path)
- `threading.Lock()` acquisition: ~50–100ns in CPython on a modern CPU