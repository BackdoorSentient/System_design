# 74. Prototype

**Section:** LLD — Design Patterns — Creational

---

## What is it?

The Prototype pattern creates new objects by **cloning an existing object** (the prototype) rather than constructing from scratch.

The prototype acts as a template — clone it, then tweak the clone for your specific needs.

---

## Why does it matter?

### Problem it solves

Object creation can be expensive:
- Database lookup to populate a complex object
- API call to fetch initial state
- Complex initialisation logic (parsing, loading, computing)

If you need **many similar objects** that differ only slightly, creating each from scratch wastes time. Clone the prototype and modify only what's different.

```
Without Prototype:
  new Monster(load_from_db("goblin"))   → 50ms DB call × 1000 goblins = 50s ❌

With Prototype:
  goblin_template = load_from_db("goblin")   → 50ms once
  for _ in range(1000):
      monster = copy.deepcopy(goblin_template)   → ~1μs × 1000 = 1ms ✅
```

---

## Shallow Copy vs Deep Copy

This is the most important concept in the Prototype pattern.

### Shallow copy (`copy.copy()`)

```python
import copy

@dataclass
class Address:
    city: str
    country: str

@dataclass
class User:
    name: str
    address: Address
    tags: list

original = User("Alice", Address("NYC", "US"), ["admin", "user"])
shallow = copy.copy(original)

# Primitive fields are independent
shallow.name = "Bob"
print(original.name)   # "Alice" ✅

# But nested objects are SHARED
shallow.address.city = "LA"
print(original.address.city)   # "LA" ← changed! ❌

shallow.tags.append("guest")
print(original.tags)   # ["admin", "user", "guest"] ← changed! ❌
```

Shallow copy copies the **top-level object only**. Nested objects (references) are shared between original and copy.

### Deep copy (`copy.deepcopy()`)

```python
deep = copy.deepcopy(original)

deep.address.city = "SF"
print(original.address.city)   # "NYC" ✅ — fully independent

deep.tags.append("guest")
print(original.tags)   # ["admin", "user"] ✅ — independent copy
```

Deep copy **recursively copies all nested objects**. Original and copy are fully independent.

### Rule of thumb

| Scenario | Use |
|---|---|
| Prototype has only primitives and immutable values | `copy.copy()` is sufficient |
| Prototype has mutable nested objects (lists, dicts, custom objects) | `copy.deepcopy()` required |
| Prototype has shared resources that shouldn't be duplicated (DB connections, file handles, thread locks) | `__deepcopy__` to control what gets copied |

---

## How does it work?

### Basic prototype with registry

```python
import copy
from abc import ABC, abstractmethod

class Prototype(ABC):
    @abstractmethod
    def clone(self) -> "Prototype":
        pass


@dataclass
class MonsterPrototype(Prototype):
    name: str
    health: int
    damage: int
    speed: float
    abilities: list
    loot_table: dict

    def clone(self) -> "MonsterPrototype":
        return copy.deepcopy(self)

    def with_health(self, health: int) -> "MonsterPrototype":
        """Convenience modifier — clone and override one field."""
        clone = self.clone()
        clone.health = health
        return clone

    def with_abilities(self, *abilities) -> "MonsterPrototype":
        clone = self.clone()
        clone.abilities = list(abilities)
        return clone


# Prototype Registry
class MonsterRegistry:
    _prototypes: dict = {}

    @classmethod
    def register(cls, name: str, prototype: MonsterPrototype):
        cls._prototypes[name] = prototype

    @classmethod
    def spawn(cls, name: str) -> MonsterPrototype:
        if name not in cls._prototypes:
            raise KeyError(f"No prototype registered for: {name}")
        return cls._prototypes[name].clone()


# Register base prototypes once (e.g., loaded from game data files)
MonsterRegistry.register("goblin", MonsterPrototype(
    name="Goblin",
    health=50,
    damage=8,
    speed=1.2,
    abilities=["scratch", "bite"],
    loot_table={"gold": (1, 5), "dagger": 0.1}
))

MonsterRegistry.register("elite_goblin", MonsterPrototype(
    name="Elite Goblin",
    health=100,
    damage=15,
    speed=1.5,
    abilities=["scratch", "bite", "shield_bash"],
    loot_table={"gold": (5, 15), "dagger": 0.3, "armor_fragment": 0.2}
))

# Spawn instances — cheap clones, no DB or config reload
goblin1 = MonsterRegistry.spawn("goblin")
goblin2 = MonsterRegistry.spawn("goblin")
boss_goblin = MonsterRegistry.spawn("goblin").with_health(200).with_abilities("scratch", "bite", "roar")

goblin1.health = 30  # wounded in combat
print(goblin2.health)  # 50 — unaffected, fully independent ✅
```

---

### Document Template System

```python
import copy
from datetime import datetime

@dataclass
class DocumentSection:
    title: str
    content: str
    metadata: dict

@dataclass
class Document:
    title: str
    sections: list
    author: str
    created_at: datetime
    tags: list
    is_template: bool = True

    def clone(self) -> "Document":
        new_doc = copy.deepcopy(self)
        new_doc.is_template = False
        new_doc.created_at = datetime.utcnow()
        new_doc.author = ""  # New doc needs a new author
        return new_doc


# Template: created once, costly to construct
nda_template = Document(
    title="Non-Disclosure Agreement",
    sections=[
        DocumentSection("Parties", "This agreement is between [PARTY_A] and [PARTY_B]", {}),
        DocumentSection("Definition", "Confidential Information means...", {}),
        DocumentSection("Obligations", "Each party agrees to...", {}),
        DocumentSection("Term", "This agreement shall remain effective for [DURATION]", {}),
    ],
    author="Legal Team",
    created_at=datetime.utcnow(),
    tags=["legal", "nda", "template"]
)

# Create a new NDA from template — fast clone, then customise
alice_nda = nda_template.clone()
alice_nda.title = "NDA — Acme Corp"
alice_nda.author = "Alice Smith"
alice_nda.sections[0].content = "This agreement is between ACME CORP and BOB LLC"
alice_nda.tags = ["legal", "nda", "acme-corp"]

# Original template unmodified ✅
print(nda_template.sections[0].content)  # "...between [PARTY_A] and [PARTY_B]"
```

---

### Custom `__deepcopy__` — Controlling What Gets Copied

Some objects contain resources that shouldn't be duplicated:

```python
import copy
import threading

class DatabaseSession:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self._connection = self._connect()  # Expensive DB connection
        self._lock = threading.Lock()        # Per-session lock — must NOT be shared
        self.query_count = 0
        self.last_query = None

    def _connect(self):
        print(f"[DB] Opening connection to {self.dsn}")
        return {"conn_id": id(self), "active": True}

    def __deepcopy__(self, memo):
        """
        Custom deep copy: copy config, but create a NEW connection and NEW lock.
        Don't share the existing connection or lock between copies.
        """
        cls = self.__class__
        result = cls.__new__(cls)
        memo[id(self)] = result

        # Copy primitive fields
        result.dsn = self.dsn
        result.query_count = 0         # Reset for new session
        result.last_query = None

        # Create new resources — don't copy the actual connection or lock
        result._connection = result._connect()  # New connection
        result._lock = threading.Lock()          # New lock

        return result
```

---

## Prototype Registry

A registry maps names to prototype instances. Clients request clones by name rather than constructing manually:

```python
class PrototypeRegistry:
    """Central store of named prototype objects."""

    def __init__(self):
        self._prototypes: dict = {}

    def register(self, key: str, prototype):
        self._prototypes[key] = prototype

    def unregister(self, key: str):
        del self._prototypes[key]

    def get(self, key: str):
        """Returns a clone, not the original."""
        if key not in self._prototypes:
            raise KeyError(f"Prototype '{key}' not found")
        return copy.deepcopy(self._prototypes[key])

    def keys(self):
        return list(self._prototypes.keys())


# Used in test factories:
registry = PrototypeRegistry()
registry.register("admin_user", User(name="Admin", role="admin", permissions=["read", "write", "delete"]))
registry.register("read_only_user", User(name="ReadOnly", role="viewer", permissions=["read"]))

# In tests:
def test_admin_can_delete():
    user = registry.get("admin_user")
    user.name = "Alice"          # Customise for test
    assert user.can_delete()     # Original prototype unaffected
```

---

## Performance Comparison

```
Operation                          | Time
-----------------------------------|----------
Deep copy a simple dataclass (~5 fields) | ~1–5 μs
Deep copy a complex object graph (~50 nested objects) | ~50–500 μs
DB query to construct an object    | 1–50 ms
HTTP call to fetch initial state   | 10–500 ms
File parsing (JSON, 100KB)         | 1–10 ms
```

When `deepcopy` is measurably too slow for your use case (e.g., spawning 100k game entities per second), consider:
- Manual clone method (only copy what's needed)
- Flyweight pattern (share immutable state, don't copy it)
- Copy-on-write (don't copy until mutation is needed)

---

## Prototype vs Flyweight

These are opposite strategies for similar goals:

| | Prototype | Flyweight |
|---|---|---|
| Strategy | **Copy** shared state into each instance | **Share** state across instances — no copying |
| Independence | Full independence — mutations don't propagate | Mutations to shared state propagate to all |
| Memory | More — each instance has its own copy | Less — shared state is referenced, not duplicated |
| Use when | Instances will be mutated independently | Instances are effectively read-only |

---

## Real-World Examples

| Example | Prototype usage |
|---|---|
| **JavaScript prototypal inheritance** | Every object has a `__proto__` — objects inherit from prototypes, not classes |
| **Java `Object.clone()`** | `Cloneable` interface enables prototype cloning — shallow by default |
| **pytest fixtures with factories** | `base_user` fixture cloned per test with overrides |
| **Game spawners** | Prefab/template game objects cloned when enemies spawn |
| **Browser tab duplication** | "Duplicate Tab" clones page state rather than reloading |
| **ORM test factories (Factory Boy)** | `UserFactory.build()` clones base user attributes with overrides |

---

## Trade-offs

| Concern | Detail |
|---|---|
| ✅ Avoids expensive re-construction | Clone in μs vs reconstruct in ms |
| ✅ Produces independent instances | Modify each clone without affecting others |
| ✅ Encapsulates creation logic | Clone + customise is cleaner than re-running complex init |
| ❌ Deep copy can be expensive | Complex object graphs with many nested objects — profile first |
| ❌ Must implement `__deepcopy__` carefully | Shared resources (connections, locks) must not be naively copied |
| ❌ Hidden aliasing bugs | Forgetting deep copy and using shallow copy → mutations propagate unexpectedly |
| ❌ Circular references | `deepcopy` handles these, but can be slow on complex graphs |

---

## Interview Q&A

**Q: What is the Prototype pattern and when would you use it?**

A: Create new objects by cloning an existing prototype rather than constructing from scratch. Use it when: object creation is expensive (DB/API lookup, complex initialisation), you need many similar objects that differ only in minor details, or you want to pre-configure "template" objects that serve as starting points.

**Q: What is the difference between shallow and deep copy, and which does Prototype need?**

A: Shallow copy creates a new top-level object but shares references to nested objects — modifying a nested object in the copy affects the original. Deep copy recursively copies all nested objects — original and copy are fully independent. Prototype pattern almost always needs deep copy, since the purpose is to produce independent instances. Use `__deepcopy__` when some resources (DB connections, locks) shouldn't be copied at all and should be freshly created.

**Q: When would you use Prototype over just calling the constructor?**

A: When the construction is expensive (network/DB/file I/O), when you want to preserve some pre-computed or configured state that would be expensive to reproduce, or when you need many slight variations of a base object. The canonical example: load a complex game entity template from a database once, then spawn thousands of instances via `deepcopy` at 1μs each rather than 50ms each.

**Q: How does Prototype differ from Flyweight?**

A: They're opposite strategies. Prototype copies shared state into each independent instance — mutations don't propagate. Flyweight shares state across instances without copying — instances are essentially read-only. Prototype: each object owns its state. Flyweight: objects borrow shared immutable state from a pool.

**Q: What is a Prototype Registry?**

A: A dictionary mapping names to prototype instances. Clients call `registry.get("admin_user")` which returns a deep copy of the stored prototype. This centralises "template" objects and ensures clients always work with independent clones, never the original.

---

## Numbers to Remember

- `copy.copy()` (shallow): ~0.5–1 μs for a simple object
- `copy.deepcopy()` (deep, simple object): ~5–10 μs
- `copy.deepcopy()` (deep, complex graph, 50 nested objects): ~100–500 μs
- DB query equivalent: 1,000–50,000× slower than deepcopy
- Python's `deepcopy` handles circular references using a `memo` dict — prevents infinite loops