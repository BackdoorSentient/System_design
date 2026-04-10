# 62. Polymorphism
**Section:** LLD — OOP Fundamentals

---

## What is Polymorphism?

**Polymorphism** — "one interface, many forms." The ability to call the same method on different object types and get type-appropriate behaviour, without the caller needing to know the concrete type.

```python
shapes = [Circle(5), Rectangle(4, 6), Triangle(3, 4, 5)]

for shape in shapes:
    print(shape.area())  # Same call, different implementations
```

This is the mechanism that enables open/closed design — you can add `Pentagon` without changing the loop.

---

## Q: What is compile-time polymorphism (method overloading)?

**A:** Same method name, different parameter signatures. The correct version is selected at **compile time** based on the argument types/count.

**Java supports it:**
```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }
}

Calculator c = new Calculator();
c.add(1, 2);          // int version — resolved at compile time
c.add(1.0, 2.0);      // double version
```

**Python does NOT support overloading** — the last definition wins:
```python
class Calculator:
    def add(self, a, b):
        return a + b

    def add(self, a, b, c):   # THIS REPLACES the previous add
        return a + b + c

c = Calculator()
c.add(1, 2)     # TypeError: add() missing 1 required positional argument: 'c'
```

**Python workaround — default arguments or `*args`:**
```python
class Calculator:
    def add(self, *args) -> float:
        return sum(args)

c.add(1, 2)       # 3
c.add(1, 2, 3)    # 6
```

---

## Q: What is runtime polymorphism (dynamic dispatch)?

**A:** The method called is determined at **runtime** based on the object's actual type, not the reference type. This is the most important form of polymorphism.

```python
from abc import ABC, abstractmethod
from math import pi

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...


class Circle(Shape):
    def __init__(self, r: float):
        self.r = r

    def area(self) -> float:
        return pi * self.r ** 2


class Rectangle(Shape):
    def __init__(self, w: float, h: float):
        self.w = w
        self.h = h

    def area(self) -> float:
        return self.w * self.h


class Triangle(Shape):
    def __init__(self, base: float, height: float):
        self.base = base
        self.height = height

    def area(self) -> float:
        return 0.5 * self.base * self.height


# Dynamic dispatch — Python checks the object's actual type at runtime
shapes: list[Shape] = [Circle(5.0), Rectangle(4.0, 6.0), Triangle(3.0, 8.0)]
total_area = sum(s.area() for s in shapes)   # correct implementation called for each
```

Adding a new `Shape` subclass requires zero changes to this loop — this is the **open/closed principle** in action.

---

## Q: What is duck typing? How does Python use it for polymorphism?

**A:** "If it walks like a duck and quacks like a duck, it's a duck." Python doesn't check the object's type — it checks whether the required method exists.

```python
class Dog:
    def speak(self) -> str:
        return "Woof!"

class Cat:
    def speak(self) -> str:
        return "Meow!"

class Robot:
    def speak(self) -> str:
        return "Beep boop"

# No common base class — but all three work polymorphically
def make_speak(thing) -> None:
    print(thing.speak())   # works for anything with a speak() method

for creature in [Dog(), Cat(), Robot()]:
    make_speak(creature)
```

**Python resolves method calls at runtime** — if the object has a `speak` method, it works. No inheritance required.

**Type hints + duck typing with Protocol:**
```python
from typing import Protocol

class Speakable(Protocol):
    def speak(self) -> str: ...

def make_speak(thing: Speakable) -> None:   # mypy checks structural compatibility
    print(thing.speak())
```

---

## Q: What is the connection between polymorphism and the Liskov Substitution Principle?

**A:** Polymorphism **requires** LSP to work correctly. The Liskov Substitution Principle says: wherever a `Shape` is expected, any `Shape` subclass must be substitutable without breaking the program.

**LSP violation breaks polymorphism:**
```python
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Square(Shape):
    def area(self) -> float:
        return self.side ** 2

class WeirdShape(Shape):
    def area(self) -> float:
        raise NotImplementedError("We don't support this")  # LSP violation
```

If `WeirdShape.area()` raises instead of returning a float, the polymorphic loop `sum(s.area() for s in shapes)` breaks. LSP is the contract that makes polymorphism safe.

---

## Q: What is operator overloading?

**A:** Defining `__dunder__` methods so Python's built-in operators work on custom objects.

```python
from dataclasses import dataclass

@dataclass
class Vector2D:
    x: float
    y: float

    def __add__(self, other: "Vector2D") -> "Vector2D":
        return Vector2D(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar: float) -> "Vector2D":
        return Vector2D(self.x * scalar, self.y * scalar)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector2D):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __abs__(self) -> float:
        return (self.x**2 + self.y**2) ** 0.5

    def __repr__(self) -> str:
        return f"Vector2D({self.x}, {self.y})"

    def __str__(self) -> str:
        return f"({self.x}, {self.y})"


v1 = Vector2D(1, 2)
v2 = Vector2D(3, 4)
print(v1 + v2)     # Vector2D(4, 6) — calls __add__
print(v1 * 3)      # Vector2D(3, 6) — calls __mul__
print(abs(v2))     # 5.0 — calls __abs__
print(str(v1))     # (1, 2) — calls __str__
print(repr(v1))    # Vector2D(1, 2) — calls __repr__
```

**Key dunder methods:**

| Operator | Dunder Method |
|---|---|
| `+` | `__add__` |
| `-` | `__sub__` |
| `*` | `__mul__` |
| `==` | `__eq__` |
| `<` | `__lt__` |
| `len()` | `__len__` |
| `str()` | `__str__` |
| `repr()` | `__repr__` |
| `[]` | `__getitem__` |
| `in` | `__contains__` |

---

## Q: What is parametric polymorphism (Generics)?

**A:** One data structure or function works correctly for **any type** — type safety without code duplication.

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

    def peek(self) -> T:
        return self._items[-1]

    def is_empty(self) -> bool:
        return len(self._items) == 0


int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push(2)
print(int_stack.pop())   # 2

str_stack: Stack[str] = Stack()
str_stack.push("hello")
str_stack.push("world")
print(str_stack.pop())   # "world"
```

---

## Q: What is `@singledispatch` — ad-hoc polymorphism in Python?

**A:** Type-based dispatch without inheritance. Different implementations of the same function for different argument types.

```python
from functools import singledispatch

@singledispatch
def process(data) -> str:
    return f"Unknown type: {type(data)}"

@process.register(int)
def _(data: int) -> str:
    return f"Processing integer: {data * 2}"

@process.register(str)
def _(data: str) -> str:
    return f"Processing string: {data.upper()}"

@process.register(list)
def _(data: list) -> str:
    return f"Processing list of {len(data)} items"

print(process(42))            # "Processing integer: 84"
print(process("hello"))       # "Processing string: HELLO"
print(process([1, 2, 3]))     # "Processing list of 3 items"
```

---

## Real-World Examples

**Django ORM — `.save()` on any model:**
```python
class User(models.Model):
    email = models.EmailField()

class Order(models.Model):
    total = models.DecimalField()

# Same .save() call — different SQL generated based on concrete model type
user = User(email="a@b.com")
user.save()    # INSERT INTO users ...

order = Order(total=99.99)
order.save()   # INSERT INTO orders ...
```

**Django REST Framework — `.validate()` on any serializer:**
```python
class UserSerializer(serializers.Serializer):
    def validate(self, data): ...

class OrderSerializer(serializers.Serializer):
    def validate(self, data): ...

# Polymorphic — same validate() call dispatches to the right serializer
```

---

## Interview Q&A

**Q: Does Python support method overloading?**
> No. Python only keeps the last definition of a method — earlier definitions are overwritten. Use default arguments, `*args`, or `@singledispatch` as workarounds.

**Q: How is duck typing different from classical polymorphism?**
> Classical polymorphism requires objects to share a common base class or interface — the relationship is explicit via inheritance. Duck typing only requires that the object has the right method — no inheritance or explicit contract needed. Python enables both; use `Protocol` type hints for duck typing with static type checking.

**Q: What is the difference between `__str__` and `__repr__`?**
> `__repr__` is for developers: unambiguous, ideally valid Python to recreate the object (used in REPL, debugging). `__str__` is for end users: human-readable display (used by `print()`, string formatting). `str(obj)` falls back to `__repr__` if `__str__` is not defined. Rule of thumb: always define `__repr__`; add `__str__` when the user-facing display differs from the debug representation.

**Q: Explain how `sum(s.area() for s in shapes)` is polymorphic.**
> The variable `s` is typed as `Shape`, but at runtime each element is a concrete subtype (Circle, Rectangle, Triangle). Python resolves `s.area()` at runtime based on the object's actual type — this is dynamic dispatch. The `sum()` function has no knowledge of shapes; it just receives floats. Adding new shape types never requires changing this expression.