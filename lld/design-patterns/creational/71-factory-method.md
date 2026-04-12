# 71. Factory Method

**Section:** LLD — Design Patterns — Creational

---

## What is it?

The Factory Method pattern defines an **interface for creating an object**, but lets **subclasses decide which class to instantiate**. The creator class defers instantiation to subclasses.

The key insight: the code that *uses* an object shouldn't need to know the *exact class* of that object. The factory method is the indirection layer.

---

## Why does it matter?

### Problem it solves

You have a creator class that orchestrates logic involving some product, but the exact type of product varies by context:

```python
# Without factory method — rigid, violates OCP
class NotificationService:
    def send(self, channel: str, message: str):
        if channel == "email":
            notifier = EmailNotifier()
        elif channel == "sms":
            notifier = SMSNotifier()
        elif channel == "push":
            notifier = PushNotifier()
        else:
            raise ValueError(f"Unknown channel: {channel}")
        notifier.send(message)
        # Adding Slack means modifying this method ← OCP violation
```

---

## Structure

```
Creator (abstract)
├── create_product()   ← abstract factory method — subclasses override this
└── some_operation()   ← uses the product, doesn't know its concrete class

ConcreteCreatorA  →  creates  →  ConcreteProductA
ConcreteCreatorB  →  creates  →  ConcreteProductB
```

---

## How does it work?

### Example: Notification Factory

```python
from abc import ABC, abstractmethod

# Product interface
class Notification(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> bool:
        pass

    @abstractmethod
    def get_delivery_report(self) -> dict:
        pass


# Concrete products
class EmailNotification(Notification):
    def __init__(self):
        self._sent_count = 0

    def send(self, recipient: str, message: str) -> bool:
        print(f"[Email] Sending to {recipient}: {message}")
        self._sent_count += 1
        return True

    def get_delivery_report(self) -> dict:
        return {"channel": "email", "sent": self._sent_count}


class SMSNotification(Notification):
    def __init__(self, provider: str = "twilio"):
        self._provider = provider
        self._sent_count = 0

    def send(self, recipient: str, message: str) -> bool:
        print(f"[SMS via {self._provider}] Sending to {recipient}: {message}")
        self._sent_count += 1
        return True

    def get_delivery_report(self) -> dict:
        return {"channel": "sms", "provider": self._provider, "sent": self._sent_count}


class PushNotification(Notification):
    def send(self, recipient: str, message: str) -> bool:
        print(f"[Push] Sending to device {recipient}: {message}")
        return True

    def get_delivery_report(self) -> dict:
        return {"channel": "push"}


# Creator (abstract)
class NotificationFactory(ABC):

    @abstractmethod
    def create_notification(self) -> Notification:
        """Factory method — subclasses decide what to create."""
        pass

    def notify(self, recipient: str, message: str) -> dict:
        """Template method that uses the factory method."""
        notifier = self.create_notification()
        success = notifier.send(recipient, message)
        report = notifier.get_delivery_report()
        report["success"] = success
        return report


# Concrete creators
class EmailNotificationFactory(NotificationFactory):
    def create_notification(self) -> Notification:
        return EmailNotification()


class SMSNotificationFactory(NotificationFactory):
    def __init__(self, provider: str = "twilio"):
        self._provider = provider

    def create_notification(self) -> Notification:
        return SMSNotification(provider=self._provider)


class PushNotificationFactory(NotificationFactory):
    def create_notification(self) -> Notification:
        return PushNotification()


# Client code — depends only on NotificationFactory interface
def send_alert(factory: NotificationFactory, user: str, msg: str):
    return factory.notify(user, msg)

# Usage
result = send_alert(EmailNotificationFactory(), "user@example.com", "Your order shipped!")
result = send_alert(SMSNotificationFactory(provider="vonage"), "+1234567890", "OTP: 482910")
```

### LLM Provider Factory (real-world AI engineering example)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class LLMResponse:
    content: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    cost_usd: float


class LLMClient(ABC):
    @abstractmethod
    def complete(self, prompt: str, max_tokens: int = 1000) -> LLMResponse:
        pass

    @abstractmethod
    def get_model_name(self) -> str:
        pass


class OpenAIClient(LLMClient):
    def __init__(self, api_key: str, model: str = "gpt-4o"):
        self._api_key = api_key
        self._model = model

    def complete(self, prompt: str, max_tokens: int = 1000) -> LLMResponse:
        # Real: openai.chat.completions.create(...)
        return LLMResponse(
            content="OpenAI response",
            model=self._model,
            prompt_tokens=len(prompt.split()),
            completion_tokens=50,
            cost_usd=0.003
        )

    def get_model_name(self) -> str:
        return self._model


class AnthropicClient(LLMClient):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self._api_key = api_key
        self._model = model

    def complete(self, prompt: str, max_tokens: int = 1000) -> LLMResponse:
        return LLMResponse(
            content="Anthropic response",
            model=self._model,
            prompt_tokens=len(prompt.split()),
            completion_tokens=50,
            cost_usd=0.002
        )

    def get_model_name(self) -> str:
        return self._model


# Factory interface
class LLMFactory(ABC):
    @abstractmethod
    def create_llm(self) -> LLMClient:
        pass

    def run_prompt(self, prompt: str) -> LLMResponse:
        client = self.create_llm()
        return client.complete(prompt)


class OpenAIFactory(LLMFactory):
    def __init__(self, api_key: str):
        self._api_key = api_key

    def create_llm(self) -> LLMClient:
        return OpenAIClient(api_key=self._api_key)


class AnthropicFactory(LLMFactory):
    def __init__(self, api_key: str):
        self._api_key = api_key

    def create_llm(self) -> LLMClient:
        return AnthropicClient(api_key=self._api_key)


# Switch providers by swapping factory — no other code changes
import os
factory: LLMFactory = AnthropicFactory(api_key=os.getenv("ANTHROPIC_API_KEY"))
response = factory.run_prompt("Summarise the CAP theorem in 2 sentences.")
```

---

## Factory Method vs Constructor

| | Constructor | Factory Method |
|---|---|---|
| Returns | Always same type | Can return subtypes or cached instances |
| Polymorphism | None — always `MyClass()` | Subclass decides concrete type |
| Caching | Not possible | Can return existing instance |
| Naming | Must be class name | Can be descriptive: `from_url()`, `create_read_only()` |
| Subclassing | Not overridable | Overridable in subclass |

```python
# Constructors can't vary return type:
conn = Connection("postgres://...")  # always Connection

# Named factory methods communicate intent:
conn = Connection.read_only("postgres://...")    # returns ReadOnlyConnection
conn = Connection.from_env()                      # reads DSN from env
conn = Connection.pooled("postgres://...", size=10)  # returns PooledConnection
```

---

## Parameterised Factory (Alternative to Subclassing)

Instead of one subclass per product, use a single factory with a type parameter:

```python
class ShapeFactory:
    _registry = {}

    @classmethod
    def register(cls, name: str, shape_class):
        cls._registry[name] = shape_class

    @classmethod
    def create(cls, name: str, **kwargs):
        if name not in cls._registry:
            raise ValueError(f"Unknown shape: {name}")
        return cls._registry[name](**kwargs)

# Register at module load time
ShapeFactory.register("circle", Circle)
ShapeFactory.register("rectangle", Rectangle)
ShapeFactory.register("triangle", Triangle)

# Usage
shape = ShapeFactory.create("circle", radius=5)
```

**Trade-off:**
- Simpler — one class, not N subclasses
- Violates Open/Closed: adding a type requires editing the `create()` method (unless you use a registry like above, which restores OCP)
- Subclass-per-type is better when creation logic is complex or each factory has its own configuration

---

## Connection with Open/Closed Principle

Factory method is the canonical example of OCP in action:

- **Closed for modification**: `NotificationService` (the client) never changes
- **Open for extension**: add `SlackNotificationFactory` — no existing code touched

```
Before: adding Slack means opening NotificationService and adding an elif
After:  adding Slack means writing SlackNotification + SlackNotificationFactory
```

---

## Factory Function (Python idiom)

Python doesn't require a full class hierarchy for factory method. A standalone function works:

```python
def create_storage_client(backend: str, **kwargs):
    """Factory function — simpler than class hierarchy for straightforward cases."""
    backends = {
        "s3": S3StorageClient,
        "gcs": GCSStorageClient,
        "local": LocalStorageClient,
    }
    if backend not in backends:
        raise ValueError(f"Unknown backend: {backend}")
    return backends[backend](**kwargs)

client = create_storage_client("s3", bucket="my-bucket", region="us-east-1")
```

Use class-based factory when: you need to carry factory-level configuration (API keys, connection strings), want subclassing/polymorphism, or the factory itself needs methods beyond just creation.

---

## Real-World Examples

| Example | Factory | Products |
|---|---|---|
| `urllib.request.urlopen` | Returns different response types depending on protocol (http, ftp, file) | `HTTPResponse`, `FTPResponse` |
| Django form fields | `CharField`, `IntegerField` etc. are factory-like classes | `Widget` instances |
| pytest fixtures | `@pytest.fixture` is a factory for test dependencies | Fixture objects |
| SQLAlchemy `create_engine` | Factory function returning different engine types based on DSN scheme | `Engine` subtypes |
| `logging.getLogger` | Returns existing or new `Logger` instance by name | `Logger` |

---

## Trade-offs

| Concern | Detail |
|---|---|
| ✅ Decouples creation from usage | Client doesn't know or care which concrete product it gets |
| ✅ Enables OCP | New product types = new subclasses, no modification |
| ✅ Encapsulates creation logic | Complex initialisation hidden in factory |
| ❌ Can create subclass explosion | One subclass per product type × one factory per product type |
| ❌ Adds indirection | More classes to read and understand |
| ❌ Overkill for simple cases | A factory function or just a constructor is often enough |

---

## Interview Q&A

**Q: What is the Factory Method pattern and when would you use it?**

A: It defines an interface for creating objects, but lets subclasses decide the concrete class. Use it when: the type of object to create depends on context or configuration, creation logic is complex enough to warrant encapsulation, or you want subclasses to control what kind of objects they create. Classic trigger: "if channel == email... elif channel == sms" inside a method that shouldn't know about those details.

**Q: How is Factory Method different from just calling a constructor?**

A: A constructor is rigid — it always creates the same type and can't be overridden. A factory method is polymorphic — subclasses can return different types, cached instances, or subtypes. It also allows meaningful names (`Connection.read_only()` communicates intent; `Connection()` doesn't).

**Q: How does Factory Method enable OCP?**

A: The client depends on the `Creator` interface, not concrete creators. Adding a new product type means adding a new `ConcreteCreator` subclass — zero modification to existing code. Without the pattern, you'd add an `elif` branch every time, modifying a class that was previously working.

**Q: Factory Method vs Abstract Factory — key difference?**

A: Factory Method creates **one product**. Abstract Factory creates **a family of related products** that must be used together. If your factory only produces one kind of object, use Factory Method. If it produces multiple objects that need to be consistent with each other (Button + Scrollbar + Checkbox all from the same theme), use Abstract Factory.

---

## Numbers to Remember

- Pattern applies when you have 2+ concrete product types that need to be swappable
- A parameterised factory with a registry is O(1) lookup vs O(n) if/elif chain
- Django has ~25 built-in form field types — all created via the field class as factory