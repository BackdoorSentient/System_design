# 61. Inheritance
**Section:** LLD — OOP Fundamentals

---

## What is Inheritance?

**Inheritance** is a mechanism where a **child class** acquires the attributes and methods of a **parent class**, establishing an **is-a relationship** and enabling code reuse.

```python
class Animal:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    def breathe(self) -> str:
        return f"{self.name} breathes"

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(name={self.name!r})"


class Dog(Animal):
    def __init__(self, name: str, age: int, breed: str):
        super().__init__(name, age)   # chain to parent constructor
        self.breed = breed

    def bark(self) -> str:
        return f"{self.name} barks!"


class GuideDog(Dog):
    def __init__(self, name: str, age: int, breed: str, owner: str):
        super().__init__(name, age, breed)
        self.owner = owner

    def guide(self) -> str:
        return f"{self.name} is guiding {self.owner}"


g = GuideDog("Rex", 3, "Labrador", "Alice")
print(g.breathe())   # inherited from Animal
print(g.bark())      # inherited from Dog
print(g.guide())     # defined on GuideDog
```

---

## Q: Single vs Multiple Inheritance — what are the trade-offs?

**A:**

**Single inheritance** — one parent class. Supported in all OOP languages. Clean hierarchy.

**Multiple inheritance** — a class inherits from two or more parents. Supported in Python and C++. Not supported in Java (interfaces only).

```python
class Flyable:
    def fly(self) -> str:
        return f"{self.__class__.__name__} is flying"

class Swimmable:
    def swim(self) -> str:
        return f"{self.__class__.__name__} is swimming"

class Duck(Animal, Flyable, Swimmable):
    def quack(self) -> str:
        return "Quack!"

d = Duck("Donald", 2)
print(d.fly())    # from Flyable
print(d.swim())   # from Swimmable
print(d.breathe())# from Animal
```

**Problems with multiple inheritance:** The diamond problem.

---

## Q: What is the Diamond Problem?

**A:** When class `D` inherits from `B` and `C`, and both `B` and `C` inherit from `A`, and `A` defines a method — which version does `D` get?

```
    A
   / \
  B   C
   \ /
    D
```

```python
class A:
    def speak(self) -> str:
        return "A speaks"

class B(A):
    def speak(self) -> str:
        return "B speaks"

class C(A):
    def speak(self) -> str:
        return "C speaks"

class D(B, C):  # diamond
    pass

d = D()
print(d.speak())            # "B speaks" — Python MRO resolves this
print(D.__mro__)            # (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```

Python resolves this deterministically via **C3 Linearisation** (MRO). C++ resolves it differently (virtual inheritance). Java avoids it entirely by not supporting class multiple inheritance.

---

## Q: What is the Method Resolution Order (MRO)?

**A:** MRO is the order in which Python searches the inheritance chain to find a method or attribute. Python uses the **C3 Linearisation algorithm**.

```python
class D(B, C):
    pass

print(D.__mro__)
# (D, B, C, A, object)
# Python checks: D → B → C → A → object
```

**C3 rule simplified:** Start with the class itself, then follow parents left to right, but ensure a class never appears before its parents.

```python
class X: pass
class Y: pass
class Z(X, Y): pass
class W(Y, X): pass

# Z.__mro__ = (Z, X, Y, object)
# W.__mro__ = (W, Y, X, object)

class V(Z, W): pass  # This raises TypeError: Cannot create a consistent MRO
# Because Z wants X before Y, but W wants Y before X — contradiction
```

**Why this matters in practice:** When debugging `super()` chains in multiple inheritance, inspect `__mro__` to know the exact lookup order.

---

## Q: How does `super()` work? When do you use it?

**A:** `super()` returns a proxy object that delegates method calls to the **next class in the MRO**. It does NOT necessarily call the direct parent — it calls whatever is next in the MRO.

```python
class Animal:
    def __init__(self, name: str):
        self.name = name
        print(f"Animal.__init__ called for {name}")

class Dog(Animal):
    def __init__(self, name: str, breed: str):
        super().__init__(name)     # calls Animal.__init__ via MRO
        self.breed = breed
        print(f"Dog.__init__ called for {name}")

class GuideDog(Dog):
    def __init__(self, name: str, breed: str, owner: str):
        super().__init__(name, breed)   # calls Dog.__init__ via MRO
        self.owner = owner
```

**`super()` vs direct parent call:**

```python
# Cooperative (correct with multiple inheritance)
class Mixin:
    def setup(self):
        super().setup()   # passes control up the MRO chain

# Direct call (breaks multiple inheritance)
class Mixin:
    def setup(self):
        Base.setup(self)  # always calls Base — skips MRO
```

Always use `super()` when working with multiple inheritance or mixins.

---

## Q: What is method overriding?

**A:** A child class redefines a method from its parent. At runtime, Python calls the **actual object's type** method, not the reference type's method.

```python
class Shape:
    def area(self) -> float:
        return 0.0

class Circle(Shape):
    def __init__(self, r: float):
        self.r = r

    def area(self) -> float:         # overrides Shape.area
        return 3.14 * self.r ** 2

shapes: list[Shape] = [Circle(5), Shape()]  # list typed as Shape
for s in shapes:
    print(s.area())   # Circle.area() and Shape.area() — runtime dispatch
```

**`@Override` in Java:** Compile-time annotation that verifies the method actually overrides a parent method (catches typos). Python has no equivalent but `@typing.override` was added in Python 3.12.

---

## Q: How do `isinstance()` and `issubclass()` work with inheritance?

**A:** Both respect the inheritance hierarchy.

```python
g = GuideDog("Rex", 3, "Labrador", "Alice")

isinstance(g, GuideDog)   # True — exact type
isinstance(g, Dog)        # True — GuideDog is a Dog
isinstance(g, Animal)     # True — GuideDog is an Animal
isinstance(g, str)        # False

issubclass(GuideDog, Dog)    # True
issubclass(GuideDog, Animal) # True
issubclass(Dog, GuideDog)    # False — parent is not a subclass of child
```

Use `isinstance()` in production code for type-safe dispatch. Avoid checking `type(obj) == SomeClass` — that doesn't respect inheritance.

---

## Q: What is the Fragile Base Class Problem?

**A:** When you modify a parent class method, all subclasses that depend on the old behaviour break — even if you didn't touch them.

```python
class Collection:
    def __init__(self):
        self._items = []

    def add(self, item):
        self._items.append(item)

    def add_all(self, items):
        for item in items:
            self.add(item)   # calls self.add() — could be overridden!


class CountingCollection(Collection):
    def __init__(self):
        super().__init__()
        self._count = 0

    def add(self, item):
        super().add(item)
        self._count += 1      # counts each add

cc = CountingCollection()
cc.add_all([1, 2, 3])
print(cc._count)   # 3 — correct only because add_all calls self.add()
# If base class changes add_all to use self._items.extend() directly:
# _count would be 0 — subclass silently broken
```

This is why deep inheritance hierarchies are fragile. Base class implementation details couple all descendants.

---

## Q: When should you NOT use inheritance?

**A:** When the relationship is not a genuine **is-a** relationship.

**Classic violation — Java's `Stack`:**
```java
// java.util.Stack extends Vector — Stack IS-A Vector???
Stack<Integer> s = new Stack<>();
s.push(1);
s.push(2);
s.add(0, 99);     // Stack exposes Vector's add(index, element) method
                   // This violates stack semantics — should only push/pop
s.get(0);         // exposes get(index) — breaks LIFO abstraction
```

`Stack` is **not** a `Vector`. A `Stack` **uses** a `Vector` internally for storage. The relationship is has-a, not is-a. This should have been composition.

**Rule of thumb:** Before adding `extends`, ask: "Can I substitute a Child wherever a Parent is expected, and will it still make sense?" If no → don't use inheritance.

---

## Deep Hierarchy Warning

```
Vehicle
  └── MotorVehicle
        └── Car
              └── ElectricCar
                    └── ElectricSUV     ← 5 levels deep — a nightmare to reason about
```

**Prefer flat hierarchies.** Two levels is usually fine. Three is pushing it. More than three and you should consider composition.

---

## Full Code Example

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    def breathe(self) -> str:
        return f"{self.name} breathes"

    @abstractmethod
    def speak(self) -> str: ...


class Dog(Animal):
    def __init__(self, name: str, age: int, breed: str):
        super().__init__(name, age)
        self.breed = breed

    def speak(self) -> str:
        return f"{self.name} barks"

    def fetch(self) -> str:
        return f"{self.name} fetches the ball"


class GuideDog(Dog):
    def __init__(self, name: str, age: int, breed: str, owner: str):
        super().__init__(name, age, breed)
        self.owner = owner

    def guide(self) -> str:
        return f"{self.name} guides {self.owner} safely"

    def speak(self) -> str:
        return f"{super().speak()} (quietly)"   # super() in overridden method


# Polymorphic usage
animals: list[Animal] = [Dog("Buddy", 3, "Beagle"), GuideDog("Rex", 5, "Lab", "Alice")]
for a in animals:
    print(a.speak())   # calls the right subclass method
```

---

## Interview Q&A

**Q: What does `super().__init__()` do in Python?**
> It calls the `__init__` of the next class in the MRO, passing the current object. This chains constructor calls up the inheritance hierarchy. Always call `super().__init__()` when you extend a class to ensure parent state is properly initialised.

**Q: Why does Python's `Stack` / Java's `Stack extends Vector` violate good OOP?**
> The is-a relationship is wrong. A Stack is not a Vector — a Stack uses a storage structure internally. Because Stack inherits from Vector, it exposes all of Vector's methods (add at index, random access, etc.) which violate LIFO semantics. This is a has-a relationship masquerading as is-a. Should be composition.

**Q: What's the difference between method overloading and method overriding?**
> Overloading: same method name, different parameter signatures — resolved at compile time (Java supports it; Python doesn't). Overriding: child class redefines a parent class method — resolved at runtime based on the object's actual type (both Java and Python support this).

**Q: What does `D.__mro__` tell you?**
> The exact order Python will search for methods and attributes — starting from D, following through parents left to right, respecting C3 linearisation. Inspect this when debugging multiple inheritance issues or unexpected method resolution.