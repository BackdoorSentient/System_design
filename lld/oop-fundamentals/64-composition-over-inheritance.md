# 64. Composition over Inheritance
**Section:** LLD — OOP Fundamentals

---

## What is Composition?

**Composition** means building complex objects by combining simpler objects — a **has-a relationship**. Instead of inheriting behaviour, a class holds references to objects that provide the behaviour it needs.

```python
# Inheritance (is-a)
class Car(Engine):   # Car IS-A Engine? No.

# Composition (has-a)
class Car:
    def __init__(self, engine: Engine):
        self._engine = engine   # Car HAS-A Engine
```

---

## Q: What is the problem with deep inheritance hierarchies?

**A:** Inheritance hierarchies become rigid and brittle when you try to model behaviour combinations.

**Classic example — the Flying Animal problem:**

```
Animal
├── FlyingAnimal
│   ├── Bird
│   └── FlyingFish ← doesn't exist
└── SwimmingAnimal
    ├── Fish
    └── Duck ← swims AND flies — where does it go?
```

```python
# First attempt
class Animal: ...
class FlyingAnimal(Animal):
    def fly(self): ...
class SwimmingAnimal(Animal):
    def swim(self): ...

# Duck can fly AND swim — forced into awkward multiple inheritance
class Duck(FlyingAnimal, SwimmingAnimal): ...

# What about RubberDuck? Can't fly, can't swim — but is a duck
class RubberDuck(Animal): ...   # no fly, no swim — completely different

# What about FlyingSquirrel? Is-a squirrel, can fly — but not a bird
class FlyingSquirrel(FlyingAnimal): ...  # misleading — it glides, not truly flies
```

Every new combination of behaviours requires a new subclass or awkward multiple inheritance. The hierarchy explodes.

---

## Q: How does composition solve it?

**A:** Extract behaviour into separate objects and inject them:

```python
from abc import ABC, abstractmethod
from typing import Optional

# Behaviour interfaces
class FlyBehaviour(ABC):
    @abstractmethod
    def fly(self) -> str: ...

class QuackBehaviour(ABC):
    @abstractmethod
    def quack(self) -> str: ...

# Concrete behaviour implementations
class FlyWithWings(FlyBehaviour):
    def fly(self) -> str:
        return "Flying with wings!"

class FlyNoWay(FlyBehaviour):
    def fly(self) -> str:
        return "Cannot fly"

class FlyRocket(FlyBehaviour):
    def fly(self) -> str:
        return "ROCKET BOOST!"

class Quack(QuackBehaviour):
    def quack(self) -> str:
        return "Quack!"

class Squeak(QuackBehaviour):
    def quack(self) -> str:
        return "Squeak!"

class MuteQuack(QuackBehaviour):
    def quack(self) -> str:
        return "<silence>"


# Duck has-a FlyBehaviour and has-a QuackBehaviour
class Duck:
    def __init__(self, name: str, fly_behaviour: FlyBehaviour, quack_behaviour: QuackBehaviour):
        self.name = name
        self._fly = fly_behaviour
        self._quack = quack_behaviour

    def perform_fly(self) -> str:
        return self._fly.fly()

    def perform_quack(self) -> str:
        return self._quack.quack()

    # Behaviour can be swapped at RUNTIME
    def set_fly_behaviour(self, fb: FlyBehaviour) -> None:
        self._fly = fb


# Different ducks — different behaviour compositions
mallard = Duck("Mallard", FlyWithWings(), Quack())
rubber_duck = Duck("Rubber Duck", FlyNoWay(), Squeak())
decoy_duck = Duck("Decoy", FlyNoWay(), MuteQuack())
rocket_duck = Duck("Rocket Duck", FlyRocket(), Quack())

print(mallard.perform_fly())        # Flying with wings!
print(rubber_duck.perform_fly())    # Cannot fly
print(rocket_duck.perform_fly())    # ROCKET BOOST!

# Upgrade at runtime — no new class needed
rubber_duck.set_fly_behaviour(FlyRocket())
print(rubber_duck.perform_fly())    # ROCKET BOOST!
```

This is the **Strategy pattern** — the canonical example of composition over inheritance.

---

## Q: Why is Java's `Stack` a famous inheritance violation?

**A:** `java.util.Stack` extends `Vector`. This is wrong because:

- `Stack` **is not a** `Vector` — it's a LIFO data structure that **uses** a backing storage
- By inheriting from `Vector`, `Stack` exposes ALL of `Vector`'s methods: `add(index, element)`, `get(index)`, `remove(index)`, `set(index, element)`, etc.
- These methods directly violate LIFO semantics — a stack should only `push`, `pop`, and `peek`

```java
Stack<Integer> s = new Stack<>();
s.push(1);
s.push(2);
s.push(3);
s.add(0, 99);       // inserts at index 0 — breaks LIFO! Exposed from Vector
System.out.println(s.get(0));   // 99 — random access on a stack!
```

**Correct design via composition:**
```python
class Stack:
    def __init__(self):
        self._storage: list = []   # has-a list — encapsulated, not exposed

    def push(self, item) -> None:
        self._storage.append(item)

    def pop(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._storage.pop()

    def peek(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._storage[-1]

    def is_empty(self) -> bool:
        return len(self._storage) == 0

    def __len__(self) -> int:
        return len(self._storage)
```

Callers can only push/pop/peek — the LIFO contract is enforced.

---

## Q: What is the "Banana-Gorilla-Jungle" problem?

**A:** Joe Armstrong (Erlang creator): *"You wanted a banana but you got a gorilla holding the banana and the whole jungle."*

When you inherit from a class, you pull in:
- All parent class methods (even irrelevant ones)
- All parent class state
- All parent class dependencies
- All transitive parent classes (grandparents, great-grandparents)

```python
class JSONMixin:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)

class AuditMixin:
    def log_access(self) -> None:
        print(f"Accessed at {datetime.now()}")

class CacheMixin:
    _cache: dict = {}   # class-level shared cache — shared by ALL subclasses
    def cache_result(self, key, value): ...

class User(JSONMixin, AuditMixin, CacheMixin):
    # Now User has a shared class-level cache it didn't ask for
    # Mutating CacheMixin._cache in one subclass affects ALL subclasses
    ...
```

With composition, `User` would receive an injected cache object — no hidden shared state.

---

## Q: What are Mixins? Are they composition or inheritance?

**A:** Mixins are a limited form of **multiple inheritance** used to add specific behaviours to a class without a deep hierarchy. They're a pragmatic compromise — closer to composition in intent but implemented via inheritance.

```python
class LoggingMixin:
    def log(self, message: str) -> None:
        print(f"[{self.__class__.__name__}] {message}")

class JSONMixin:
    def to_json(self) -> str:
        import json
        return json.dumps({k: v for k, v in self.__dict__.items()
                          if not k.startswith('_')})

class TimestampMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.created_at = datetime.now()


class User(TimestampMixin, JSONMixin, LoggingMixin):
    def __init__(self, name: str, email: str):
        super().__init__()
        self.name = name
        self.email = email

u = User("Alice", "alice@example.com")
print(u.to_json())          # from JSONMixin
u.log("User created")       # from LoggingMixin
print(u.created_at)         # from TimestampMixin
```

**Mixin rules:**
- Mixins should have no state of their own (or minimal)
- Mixins should not inherit from concrete classes — only from `object`
- Always call `super().__init__()` in Mixin `__init__` if it has one
- Mixin names should clearly indicate what they add (`LoggingMixin`, `SerializableMixin`)

---

## Q: How does Dependency Injection relate to composition?

**A:** DI is the most powerful form of composition. Instead of a class creating its own dependencies, they are **injected** from outside. This makes classes:
- Testable (inject mocks)
- Configurable (inject different implementations)
- Loosely coupled (depends on abstraction, not concrete class)

```python
# BAD — creates its own dependencies (tight coupling)
class OrderService:
    def __init__(self):
        self._payment = StripePaymentGateway()   # hardcoded
        self._email = SendGridEmailService()      # hardcoded

# GOOD — dependencies injected (loose coupling)
class OrderService:
    def __init__(self, payment: PaymentGateway, email: EmailService):
        self._payment = payment   # any PaymentGateway implementation
        self._email = email       # any EmailService implementation

    def process_order(self, order: Order) -> None:
        self._payment.charge(order.total, order.payment_method)
        self._email.send_confirmation(order.customer_email)

# In production
service = OrderService(
    payment=StripePaymentGateway(api_key=config.STRIPE_KEY),
    email=SendGridEmailService(api_key=config.SENDGRID_KEY)
)

# In tests
service = OrderService(
    payment=MockPaymentGateway(),
    email=MockEmailService()
)
```

---

## Q: Interface + Composition — the clean pattern

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Define contracts
class AttackStrategy(Protocol):
    def attack(self, target: str) -> str: ...

class MoveStrategy(Protocol):
    def move(self, destination: str) -> str: ...

# Concrete implementations
class SwordAttack:
    def attack(self, target: str) -> str:
        return f"Slashing {target} with sword"

class BowAttack:
    def attack(self, target: str) -> str:
        return f"Shooting arrow at {target}"

class WalkMove:
    def move(self, destination: str) -> str:
        return f"Walking to {destination}"

class TeleportMove:
    def move(self, destination: str) -> str:
        return f"Teleporting to {destination}"


# Character composes behaviours — no inheritance needed
class Character:
    def __init__(self, name: str, attack: AttackStrategy, move: MoveStrategy):
        self.name = name
        self._attack = attack
        self._move = move

    def perform_attack(self, target: str) -> str:
        return f"{self.name}: {self._attack.attack(target)}"

    def perform_move(self, destination: str) -> str:
        return f"{self.name}: {self._move.move(destination)}"


# No subclasses needed — just different compositions
warrior = Character("Thor", SwordAttack(), WalkMove())
archer = Character("Legolas", BowAttack(), WalkMove())
wizard = Character("Gandalf", BowAttack(), TeleportMove())

# Change behaviour at runtime
warrior._attack = BowAttack()   # Thor learned archery — no new class
```

---

## Real-World Examples

**React** — UI components composed of smaller components, not inherited from a base UI class.

**Go** — No classes, only structs and interfaces. Composition via struct embedding:
```go
type Logger struct {}
func (l Logger) Log(msg string) { ... }

type UserService struct {
    Logger         // embedded — UserService gets Log() method
    db *DB
}
```

**Django class-based views** — Built with mixins: `ListView`, `CreateView`, `LoginRequiredMixin` composed together.

**Python's `pathlib.Path`** — inherits from different base classes per OS, but the user-facing API is composed uniformly.

---

## Inheritance vs Composition Decision Tree

```
Is the relationship genuinely is-a?
├── No → Use COMPOSITION
└── Yes
    ├── Will subclasses need ALL of parent's public methods?
    │   ├── No → Use COMPOSITION (don't expose what you don't need)
    │   └── Yes
    │       ├── Is the hierarchy deeper than 2-3 levels?
    │       │   ├── Yes → Flatten with COMPOSITION
    │       │   └── No → Inheritance is OK
    │       └── Are behaviour combinations combinatorial?
    │           ├── Yes → Use COMPOSITION (Strategy pattern)
    │           └── No → Inheritance is OK
```

---

## Interview Q&A

**Q: "Favour composition over inheritance" — but inheritance still has valid uses, right?**
> Yes. Inheritance is appropriate when: the relationship is genuinely is-a, subclasses honour the parent's full contract, and the hierarchy is shallow (≤ 3 levels). Classic valid uses: abstract base classes with template methods, exception hierarchies (`ValueError` is-a `Exception`), framework extension points (`BaseHTTPHandler`). Composition is the default; inheritance is the exception.

**Q: What's wrong with mixins?**
> Mixins are multiple inheritance under a friendlier name — they bring the same risks: MRO complexity, hidden state interactions, fragile base class problem. Use them sparingly, keep them stateless, and always call `super().__init__()`. For new code, prefer injected collaborators over mixins.

**Q: How does composition improve testability?**
> With composition + DI, each collaborator can be replaced with a mock or fake in tests. With inheritance, the parent class's behaviour is baked in — you test the subclass but also implicitly test the parent, and you can't easily substitute the parent's behaviour.

**Q: Stack extends ArrayList/Vector is a famous mistake. Why wasn't it caught earlier?**
> Because it worked functionally — you could push and pop. The mistake is subtle: the public interface of Stack exposes Vector's full API, violating encapsulation and the principle that a class should only expose what it needs to. It wasn't until the design was scrutinised for correctness (not just functionality) that the violation became obvious. This is why design reviews matter — tests verify behaviour, but reviews catch contract violations.