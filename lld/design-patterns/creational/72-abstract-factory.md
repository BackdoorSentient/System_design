# 72. Abstract Factory

**Section:** LLD — Design Patterns — Creational

---

## What is it?

Abstract Factory provides an **interface for creating families of related or dependent objects** without specifying their concrete classes.

If Factory Method is "a factory for one product", Abstract Factory is "a factory for a whole family of products that must work together".

---

## Why does it matter?

### The core problem

You need to create multiple objects that must be **compatible with each other**. If you mix objects from different families, things break:

```
❌ DarkButton + LightCheckbox + DarkTextField  → visual inconsistency
✅ DarkButton + DarkCheckbox + DarkTextField   → cohesive dark theme

❌ AWSBlobStorage + GCPQueue + AWSFunction     → cross-cloud calls, auth failures
✅ AWSBlobStorage + AWSQueue + AWSFunction     → all within one cloud context
```

Abstract Factory **enforces family consistency at compile/type-check time**.

---

## Structure

```
AbstractFactory
├── create_product_a()    → returns AbstractProductA
├── create_product_b()    → returns AbstractProductB
└── create_product_c()    → returns AbstractProductC

ConcreteFactoryX  →  creates  →  ConcreteProductA_X, ConcreteProductB_X, ConcreteProductC_X
ConcreteFactoryY  →  creates  →  ConcreteProductA_Y, ConcreteProductB_Y, ConcreteProductC_Y
```

---

## How does it work?

### Example: Database Factory

```python
from abc import ABC, abstractmethod
from typing import Any, List

# Abstract products
class Connection(ABC):
    @abstractmethod
    def execute(self, query: str) -> List[dict]:
        pass

    @abstractmethod
    def close(self):
        pass


class QueryBuilder(ABC):
    @abstractmethod
    def select(self, *fields) -> "QueryBuilder":
        pass

    @abstractmethod
    def from_table(self, table: str) -> "QueryBuilder":
        pass

    @abstractmethod
    def where(self, condition: str) -> "QueryBuilder":
        pass

    @abstractmethod
    def build(self) -> str:
        pass


class Transaction(ABC):
    @abstractmethod
    def begin(self):
        pass

    @abstractmethod
    def commit(self):
        pass

    @abstractmethod
    def rollback(self):
        pass


# Abstract factory
class DatabaseFactory(ABC):
    @abstractmethod
    def create_connection(self, dsn: str) -> Connection:
        pass

    @abstractmethod
    def create_query_builder(self) -> QueryBuilder:
        pass

    @abstractmethod
    def create_transaction(self, connection: Connection) -> Transaction:
        pass


# ── PostgreSQL family ──────────────────────────────────────────
class PostgreSQLConnection(Connection):
    def __init__(self, dsn: str):
        print(f"[PostgreSQL] Connecting to {dsn}")
        self._dsn = dsn

    def execute(self, query: str) -> List[dict]:
        print(f"[PostgreSQL] Executing: {query}")
        return [{"id": 1, "name": "Alice"}]

    def close(self):
        print("[PostgreSQL] Connection closed")


class PostgreSQLQueryBuilder(QueryBuilder):
    def __init__(self):
        self._parts = {"select": "*", "from": "", "where": []}

    def select(self, *fields) -> "QueryBuilder":
        self._parts["select"] = ", ".join(fields)
        return self

    def from_table(self, table: str) -> "QueryBuilder":
        self._parts["from"] = table
        return self

    def where(self, condition: str) -> "QueryBuilder":
        self._parts["where"].append(condition)
        return self

    def build(self) -> str:
        sql = f"SELECT {self._parts['select']} FROM {self._parts['from']}"
        if self._parts["where"]:
            sql += " WHERE " + " AND ".join(self._parts["where"])
        return sql + ";"


class PostgreSQLTransaction(Transaction):
    def __init__(self, conn: PostgreSQLConnection):
        self._conn = conn

    def begin(self):
        self._conn.execute("BEGIN")

    def commit(self):
        self._conn.execute("COMMIT")

    def rollback(self):
        self._conn.execute("ROLLBACK")


class PostgreSQLFactory(DatabaseFactory):
    def create_connection(self, dsn: str) -> Connection:
        return PostgreSQLConnection(dsn)

    def create_query_builder(self) -> QueryBuilder:
        return PostgreSQLQueryBuilder()

    def create_transaction(self, connection: Connection) -> Transaction:
        return PostgreSQLTransaction(connection)


# ── MongoDB family ────────────────────────────────────────────
class MongoDBConnection(Connection):
    def __init__(self, dsn: str):
        print(f"[MongoDB] Connecting to {dsn}")

    def execute(self, query: str) -> List[dict]:
        # MongoDB uses JSON-style queries, not SQL
        print(f"[MongoDB] Running pipeline: {query}")
        return [{"_id": "abc123", "name": "Alice"}]

    def close(self):
        print("[MongoDB] Connection closed")


class MongoDBQueryBuilder(QueryBuilder):
    def __init__(self):
        self._pipeline = {}

    def select(self, *fields) -> "QueryBuilder":
        self._pipeline["projection"] = {f: 1 for f in fields}
        return self

    def from_table(self, table: str) -> "QueryBuilder":
        self._pipeline["collection"] = table
        return self

    def where(self, condition: str) -> "QueryBuilder":
        self._pipeline["filter"] = condition
        return self

    def build(self) -> str:
        import json
        return json.dumps(self._pipeline)


class MongoDBTransaction(Transaction):
    def __init__(self, conn: MongoDBConnection):
        self._conn = conn

    def begin(self):
        print("[MongoDB] Starting session")

    def commit(self):
        print("[MongoDB] Committing session")

    def rollback(self):
        print("[MongoDB] Aborting session")


class MongoDBFactory(DatabaseFactory):
    def create_connection(self, dsn: str) -> Connection:
        return MongoDBConnection(dsn)

    def create_query_builder(self) -> QueryBuilder:
        return MongoDBQueryBuilder()

    def create_transaction(self, connection: Connection) -> Transaction:
        return MongoDBTransaction(connection)


# ── Client code — depends only on abstractions ────────────────
class UserRepository:
    def __init__(self, factory: DatabaseFactory, dsn: str):
        self._factory = factory
        self._conn = factory.create_connection(dsn)
        self._qb = factory.create_query_builder()

    def find_active_users(self) -> List[dict]:
        query = (
            self._qb
            .select("id", "name", "email")
            .from_table("users")
            .where("active = true")
            .build()
        )
        return self._conn.execute(query)

    def update_in_transaction(self, user_id: str, data: dict):
        txn = self._factory.create_transaction(self._conn)
        txn.begin()
        try:
            self._conn.execute(f"UPDATE users SET ... WHERE id = {user_id}")
            txn.commit()
        except Exception:
            txn.rollback()
            raise


# Swap entire database stack by changing one line:
import os
if os.getenv("DB_BACKEND") == "mongo":
    factory = MongoDBFactory()
    dsn = os.getenv("MONGO_DSN", "mongodb://localhost:27017/app")
else:
    factory = PostgreSQLFactory()
    dsn = os.getenv("PG_DSN", "postgresql://localhost/app")

repo = UserRepository(factory=factory, dsn=dsn)
```

### UI Theme Factory (classic example)

```python
# Abstract products
class Button(ABC):
    @abstractmethod
    def render(self) -> str: pass

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> str: pass

class TextField(ABC):
    @abstractmethod
    def render(self) -> str: pass


# Abstract factory
class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass

    @abstractmethod
    def create_checkbox(self) -> Checkbox: pass

    @abstractmethod
    def create_text_field(self) -> TextField: pass


# Concrete families
class DarkButton(Button):
    def render(self): return "<button class='dark'>Click</button>"

class DarkCheckbox(Checkbox):
    def render(self): return "<input type='checkbox' class='dark'>"

class DarkTextField(TextField):
    def render(self): return "<input type='text' class='dark'>"


class DarkThemeFactory(UIFactory):
    def create_button(self): return DarkButton()
    def create_checkbox(self): return DarkCheckbox()
    def create_text_field(self): return DarkTextField()


class LightButton(Button):
    def render(self): return "<button class='light'>Click</button>"

class LightCheckbox(Checkbox):
    def render(self): return "<input type='checkbox' class='light'>"

class LightTextField(TextField):
    def render(self): return "<input type='text' class='light'>"


class LightThemeFactory(UIFactory):
    def create_button(self): return LightButton()
    def create_checkbox(self): return LightCheckbox()
    def create_text_field(self): return LightTextField()


# Client renders UI — never knows which theme is active
def render_login_form(factory: UIFactory):
    btn = factory.create_button()
    cb = factory.create_checkbox()
    tf = factory.create_text_field()
    return f"{tf.render()}\n{cb.render()}\n{btn.render()}"
```

---

## Cloud Provider Factory (modern example)

```python
class BlobStorage(ABC):
    @abstractmethod
    def upload(self, key: str, data: bytes): pass
    @abstractmethod
    def download(self, key: str) -> bytes: pass

class MessageQueue(ABC):
    @abstractmethod
    def publish(self, topic: str, message: str): pass
    @abstractmethod
    def consume(self, topic: str) -> str: pass

class FunctionRuntime(ABC):
    @abstractmethod
    def invoke(self, function_name: str, payload: dict) -> dict: pass


class CloudFactory(ABC):
    @abstractmethod
    def create_storage(self, bucket: str) -> BlobStorage: pass

    @abstractmethod
    def create_queue(self, region: str) -> MessageQueue: pass

    @abstractmethod
    def create_function_runtime(self) -> FunctionRuntime: pass


class AWSFactory(CloudFactory):
    def create_storage(self, bucket: str) -> BlobStorage:
        return S3Storage(bucket)  # S3-specific implementation

    def create_queue(self, region: str) -> MessageQueue:
        return SQSQueue(region)

    def create_function_runtime(self) -> FunctionRuntime:
        return LambdaRuntime()


class GCPFactory(CloudFactory):
    def create_storage(self, bucket: str) -> BlobStorage:
        return GCSStorage(bucket)

    def create_queue(self, region: str) -> MessageQueue:
        return PubSubQueue(region)

    def create_function_runtime(self) -> FunctionRuntime:
        return CloudFunctionRuntime()
```

---

## Abstract Factory vs Factory Method

| | Factory Method | Abstract Factory |
|---|---|---|
| Products created | **One** product per factory | **Family** of related products |
| Entry point | Single `create_product()` method | Multiple creation methods (one per product type) |
| Use case | Varying one product type | Varying a whole ecosystem of products together |
| Typical structure | Creator class + one ConcreteCreator per variant | AbstractFactory + one ConcreteFactory per family |

Rule of thumb: if you find yourself writing multiple Factory Methods that must always be used together → collapse them into an Abstract Factory.

---

## Abstract Factory vs Builder

| | Abstract Factory | Builder |
|---|---|---|
| What it builds | **Multiple** distinct objects (a family) | **One** complex object |
| Creation | All at once via `create_*()` calls | Step-by-step via `set_*()` calls then `build()` |
| Result | Different objects compatible with each other | One fully configured object |
| Example | `DarkThemeFactory.create_button()` + `.create_checkbox()` | `QueryBuilder().select(...).where(...).build()` |

---

## Extending Abstract Factory: The OCP Trade-off

Adding a **new family** (new `ConcreteFactory`) is easy — just add a class, nothing else changes.

Adding a **new product type** (e.g., `Tooltip`) requires:
1. Adding `create_tooltip()` to `AbstractFactory` (interface change)
2. Implementing it in **every** existing concrete factory
3. Adding `Tooltip` abstract product + all concrete implementations

This is expensive. Abstract Factory is **closed to adding new product types** without touching existing code. This is the known OCP trade-off — design for the axis of change you expect.

---

## Trade-offs

| Concern | Detail |
|---|---|
| ✅ Enforces product family consistency | Can't accidentally mix a DarkButton with LightCheckbox |
| ✅ Swappable families | Swap entire cloud provider / theme / DB stack by changing one factory instance |
| ✅ Decouples client from concrete classes | Client only uses abstract interfaces |
| ❌ Expensive to add new product types | Must update AbstractFactory interface and all ConcreteFactories |
| ❌ More classes than Factory Method | One class per product per family — class count grows as N × M |
| ❌ Overkill for 1 product | If you only have one product type, just use Factory Method |

---

## Real-World Examples

| System | Abstract Factory | Families |
|---|---|---|
| Java AWT/Swing | `Toolkit` creates OS-native widgets | Windows, macOS, Linux families |
| .NET `DbProviderFactory` | Creates `DbConnection`, `DbCommand`, `DbReader` | SQL Server, SQLite, Oracle families |
| Angular | Module factories create services, guards, pipes | Test vs production environments |
| Terraform providers | `aws`, `gcp`, `azurerm` providers create compatible infra resources | AWS vs GCP vs Azure families |

---

## Interview Q&A

**Q: What is the Abstract Factory pattern? How does it differ from Factory Method?**

A: Abstract Factory creates **families of related objects** through a single interface with multiple creation methods. Factory Method creates **one product** through a single creation method overridden by subclasses. Use Factory Method when one product type varies; use Abstract Factory when multiple product types must be consistent with each other.

**Q: Give a real-world example where mixing products from different families causes bugs.**

A: A cloud storage application that uses AWS S3 for blobs but Google Pub/Sub for queues. The authentication, region configuration, and retry semantics differ between providers — mixing them causes auth failures and unexpected latency. An `AWSFactory` ensures every component it creates is within the same AWS authentication context.

**Q: What is the main extensibility weakness of Abstract Factory?**

A: It's hard to add new product types. Adding a `Tooltip` to a UI factory requires changing the `AbstractFactory` interface and implementing `create_tooltip()` in every concrete factory. The pattern is optimised for adding new families (new ConcreteFactory), not new product types.

**Q: When would you choose Abstract Factory over Builder?**

A: Abstract Factory when you need to produce several distinct but related objects that must be compatible. Builder when you need to construct one complex object step-by-step with many optional configuration steps. Key tell: Abstract Factory returns multiple different types; Builder returns one type from `build()`.

---

## Numbers to Remember

- Abstract Factory class count: **N product types × M families** ConcreteProduct classes + **M** ConcreteFactory classes + **N** AbstractProduct interfaces + 1 AbstractFactory = grows quadratically with families
- Adding a new family: O(N) — one new implementation per product type
- Adding a new product type: O(M) — must update all M existing factories