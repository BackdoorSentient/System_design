# 67. Liskov Substitution Principle (LSP)

**Section:** LLD — SOLID Principles

---

## What is it?

> **If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering the correctness of the program.**

Barbara Liskov, 1987 (Turing Award 2008).

In plain terms: **any code that works with a parent class must work correctly when given a subclass instead.** Subclasses must honour every contract, every guarantee, every invariant of their parent.

LSP is what makes **polymorphism safe**. If you can't trust that a subclass behaves like its parent, you can't safely use the parent class reference.

---

## Why does it matter?

Without LSP:
- You must check the actual type of an object before calling methods (`isinstance` checks everywhere)
- Polymorphism breaks — you can't trust `for animal in animals: animal.speak()`
- Subclassing becomes a trap — the relationship "is-a" in code doesn't reflect reality

With LSP:
- Polymorphism is safe and reliable
- Collections of parent-typed objects work correctly with any subclass
- You can extend hierarchies without breaking callers

---

## The Four Rules of LSP

### Rule 1 — Preconditions cannot be strengthened in subclasses

If the parent accepts `price >= 0`, the subclass cannot require `price >= 100`.

```python
class Discount:
    def apply(self, price: float) -> float:
        assert price >= 0  # parent precondition
        return price * 0.9

class VIPDiscount(Discount):
    def apply(self, price: float) -> float:
        assert price >= 100  # VIOLATION: stronger precondition than parent
        return price * 0.7
```

Code that passes `price = 50` to a `Discount` will break when given a `VIPDiscount`.

### Rule 2 — Postconditions cannot be weakened in subclasses

If the parent guarantees the return value is positive, the subclass cannot return negative values.

```python
class TemperatureSensor:
    def read(self) -> float:
        # Guaranteed: returns value in Celsius, range [-273.15, +1000]
        return self._hardware.read()

class FaultySensor(TemperatureSensor):
    def read(self) -> float:
        return float('nan')  # VIOLATION: weakened postcondition
```

### Rule 3 — Invariants must be preserved

Every invariant the parent guarantees must hold for the subclass.

### Rule 4 — No new exceptions

A subclass method should not throw exceptions not declared or expected from the parent.

```python
class FileReader:
    def read(self, path: str) -> str:
        with open(path) as f:
            return f.read()

class NetworkFileReader(FileReader):
    def read(self, path: str) -> str:
        raise NetworkTimeoutError("...")  # VIOLATION: caller expects IOError at most
```

---

## The Classic Violation — Square extends Rectangle

This is the canonical LSP violation in every textbook.

```python
class Rectangle:
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height

    def set_width(self, w: float) -> None:
        self._width = w

    def set_height(self, h: float) -> None:
        self._height = h

    def area(self) -> float:
        return self._width * self._height


class Square(Rectangle):
    """A square's invariant: width == height always."""

    def set_width(self, w: float) -> None:
        self._width = w
        self._height = w  # Must keep them equal

    def set_height(self, h: float) -> None:
        self._width = h
        self._height = h  # Must keep them equal
```

**LSP violation in action:**

```python
def test_area(r: Rectangle) -> None:
    r.set_width(4)
    r.set_height(5)
    assert r.area() == 20  # Passes for Rectangle

test_area(Rectangle(1, 1))  # ✅ area = 20
test_area(Square(1))        # ❌ area = 25 (set_height set both to 5)
```

The function `test_area` is written for `Rectangle`. Substituting a `Square` breaks it. **LSP is violated.**

The problem: a Square *geometrically* is a Rectangle, but a Square *behaviorally* is NOT a Rectangle in the sense of `set_width` and `set_height` being independent.

**Is-a in geometry ≠ Is-a in behaviour.** LSP cares about behaviour, not taxonomy.

---

## How to Detect LSP Violations

Ask:
1. Does the subclass override a method and **throw an exception** the parent doesn't throw?
2. Does the subclass method require **stricter input** than the parent?
3. Does the subclass method return **less** than the parent guarantees?
4. Does the subclass **break invariants** that the parent establishes?
5. **Does the parent's test suite pass on the subclass?** — If not, LSP is violated.

---

## How to Fix LSP Violations

### Fix 1 — Use a common interface, not inheritance

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

class Square(Shape):
    def __init__(self, side: float):
        self.side = side

    def area(self) -> float:
        return self.side ** 2
```

Square and Rectangle share the `area()` contract but are independent. No inheritance relationship — no LSP issue.

### Fix 2 — Split the hierarchy

```python
# VIOLATION
class Bird:
    def fly(self) -> None: ...

class Penguin(Bird):
    def fly(self) -> None:
        raise UnsupportedOperationException("Penguins cannot fly")
        # Caller that calls bird.fly() breaks when given a Penguin

# FIX
class Bird(ABC):
    @abstractmethod
    def make_sound(self) -> str: ...

class FlyingBird(Bird):
    @abstractmethod
    def fly(self) -> None: ...

class Eagle(FlyingBird):
    def make_sound(self) -> str: return "screech"
    def fly(self) -> None: print("soaring")

class Penguin(Bird):
    def make_sound(self) -> str: return "squawk"
    # No fly() — does not inherit the contract
```

---

## Real-World LSP Violations

### `java.util.Stack extends Vector`

`java.util.Stack` inherits from `java.util.Vector`. This means `Stack` exposes methods like `add(int index, E element)`, `remove(int index)`, etc. — which allow inserting and removing elements at arbitrary positions. This breaks the LIFO invariant that defines a stack.

A caller using a `Vector` reference and calling `add(0, element)` gets confusing semantics from a `Stack`.

### `UnsupportedOperationException` in Java Collections

`Arrays.asList()` returns a fixed-size list that implements `List`. Calling `add()` or `remove()` throws `UnsupportedOperationException`. Code that accepts `List<T>` and calls `list.add(element)` breaks when given this implementation — LSP violation.

---

## LSP and Testing

The **Liskov test**: run the parent class's test suite against every subclass.

```python
# Parent test suite
class TestShape:
    def get_shape(self) -> Shape:  # Override in subclasses
        raise NotImplementedError

    def test_area_is_positive(self):
        s = self.get_shape()
        assert s.area() > 0

class TestCircle(TestShape):
    def get_shape(self): return Circle(5)

class TestRectangle(TestShape):
    def get_shape(self): return Rectangle(4, 5)

class TestSquare(TestShape):
    def get_shape(self): return Square(4)
```

If all three test classes pass, LSP is satisfied.

---

## Trade-offs

| Pro | Con |
|---|---|
| Polymorphism is safe and predictable | Forces careful hierarchy design upfront |
| No defensive `isinstance` checks in callers | Sometimes requires abandoning intuitive "is-a" relationships |
| Parent test suites cover all subclasses | Splitting hierarchies adds more classes |
| Callers can trust the contract | Violations are subtle and hard to spot without tests |

---

## LSP vs OCP vs DIP

| Principle | Question it answers |
|---|---|
| OCP | Can I add new behaviour without changing existing code? |
| LSP | Can I safely use subclasses wherever I use the parent? |
| DIP | Do I depend on abstractions rather than concretions? |

LSP makes OCP safe — you can extend with new subclasses only if those subclasses don't break callers.

---

## Interview Q&A

**Q: What is LSP in one sentence?**
A: A subclass must be fully substitutable for its parent — any code that works with the parent must work correctly with the subclass.

**Q: Explain the Square-Rectangle problem.**
A: `Rectangle` has independent `set_width` and `set_height`. `Square` overrides both to keep width == height. Code that sets width=4, height=5 and expects area=20 gets area=25 for a `Square`. The subclass breaks the parent's contract. Fix: make both implement a `Shape` interface rather than inheriting.

**Q: What are the four LSP rules?**
A: (1) Subclass cannot strengthen preconditions. (2) Subclass cannot weaken postconditions. (3) Subclass must preserve parent invariants. (4) Subclass cannot throw new exceptions.

**Q: How do you detect LSP violations?**
A: Run the parent's test suite against the subclass. If any test fails, LSP is violated. Also look for `UnsupportedOperationException`, `isinstance` checks in callers, and methods that throw exceptions not declared by the parent.

**Q: How does `java.util.Stack extends Vector` violate LSP?**
A: `Vector` allows indexed insertion and removal at any position. `Stack` inherits these, breaking the LIFO invariant that defines a stack. Code using a `Stack` via a `Vector` reference can corrupt the stack by inserting at index 0.

**Q: Is "is-a" in English always valid for inheritance?**
A: No. "A square is a rectangle" is geometrically true but behaviourally false in the context of independent width/height mutation. LSP defines "is-a" in terms of *behavioural* substitutability, not taxonomic classification.

---

## Numbers to Remember

- Barbara Liskov's paper: 1987 (Turing Award: 2008)
- `java.util.Stack` — classic real-world violation, in JDK since Java 1.0
- The parent's test suite is the most reliable LSP checker — if N tests cover the parent, all N must pass for every subclass