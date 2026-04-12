# 73. Builder

**Section:** LLD — Design Patterns — Creational

---

## What is it?

The Builder pattern **separates the construction of a complex object from its representation**, allowing the same construction process to create different representations.

In practice: instead of one massive constructor with 12 parameters (most optional), you accumulate state through chainable setter methods and produce the final object via `build()`.

---

## Why does it matter?

### The telescoping constructor anti-pattern

```python
# What you're forced to do without Builder:
person = Person("Alice", 30, None, None, None, "alice@example.com", None, "NYC", None, True, None, "admin")
#                name  age  phone  fax  website  email  twitter  city  zip  active  bio  role
# Which None is phone and which is fax? Unreadable and error-prone.
```

Python's keyword arguments help, but don't enforce construction order or validate the final state:

```python
person = Person(name="Alice", email="alice@example.com", role="admin")
# No way to force role only when active=True, or require email XOR phone
```

Builder solves: **step-by-step construction with validation at the end**.

---

## How does it work?

### Example: HTTP Request Builder

```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any
import json

# The final product — immutable
@dataclass(frozen=True)
class HttpRequest:
    method: str
    url: str
    headers: Dict[str, str]
    body: Optional[bytes]
    timeout_seconds: float
    max_retries: int

    def __repr__(self):
        return f"HttpRequest({self.method} {self.url}, timeout={self.timeout_seconds}s, retries={self.max_retries})"


class RetryPolicy:
    def __init__(self, max_retries: int = 3, backoff_factor: float = 2.0):
        self.max_retries = max_retries
        self.backoff_factor = backoff_factor


class HttpRequestBuilder:
    def __init__(self):
        self._method: Optional[str] = None
        self._url: Optional[str] = None
        self._headers: Dict[str, str] = {}
        self._body: Optional[bytes] = None
        self._timeout: float = 30.0
        self._retry_policy: RetryPolicy = RetryPolicy()

    def method(self, method: str) -> "HttpRequestBuilder":
        self._method = method.upper()
        return self

    def url(self, url: str) -> "HttpRequestBuilder":
        self._url = url
        return self

    def header(self, key: str, value: str) -> "HttpRequestBuilder":
        self._headers[key] = value
        return self

    def bearer_token(self, token: str) -> "HttpRequestBuilder":
        """Convenience method — sets Authorization header."""
        return self.header("Authorization", f"Bearer {token}")

    def json_body(self, data: Any) -> "HttpRequestBuilder":
        self._body = json.dumps(data).encode("utf-8")
        self._headers.setdefault("Content-Type", "application/json")
        return self

    def body(self, data: bytes, content_type: str = "application/octet-stream") -> "HttpRequestBuilder":
        self._body = data
        self._headers.setdefault("Content-Type", content_type)
        return self

    def timeout(self, seconds: float) -> "HttpRequestBuilder":
        self._timeout = seconds
        return self

    def retry_policy(self, max_retries: int, backoff_factor: float = 2.0) -> "HttpRequestBuilder":
        self._retry_policy = RetryPolicy(max_retries, backoff_factor)
        return self

    def build(self) -> HttpRequest:
        # Validation happens here — not scattered across setters
        if not self._method:
            raise ValueError("HTTP method is required")
        if not self._url:
            raise ValueError("URL is required")
        if self._method in ("GET", "HEAD") and self._body:
            raise ValueError(f"Body not allowed for {self._method} requests")
        if self._timeout <= 0:
            raise ValueError("Timeout must be positive")

        return HttpRequest(
            method=self._method,
            url=self._url,
            headers=dict(self._headers),
            body=self._body,
            timeout_seconds=self._timeout,
            max_retries=self._retry_policy.max_retries,
        )


# Method chaining (fluent interface)
request = (
    HttpRequestBuilder()
    .method("POST")
    .url("https://api.example.com/users")
    .bearer_token("eyJhbGciOiJIUzI1NiJ9...")
    .json_body({"name": "Alice", "role": "admin"})
    .timeout(10.0)
    .retry_policy(max_retries=3)
    .build()
)
print(request)
# HttpRequest(POST https://api.example.com/users, timeout=10.0s, retries=3)
```

---

### Example: SQL Query Builder

```python
class SQLQueryBuilder:
    def __init__(self):
        self._select_fields = []
        self._table = None
        self._joins = []
        self._where_clauses = []
        self._order_by = []
        self._limit = None
        self._offset = None

    def select(self, *fields) -> "SQLQueryBuilder":
        self._select_fields.extend(fields)
        return self

    def from_table(self, table: str, alias: str = None) -> "SQLQueryBuilder":
        self._table = f"{table} {alias}" if alias else table
        return self

    def join(self, table: str, on: str, join_type: str = "INNER") -> "SQLQueryBuilder":
        self._joins.append(f"{join_type} JOIN {table} ON {on}")
        return self

    def where(self, condition: str) -> "SQLQueryBuilder":
        self._where_clauses.append(condition)
        return self

    def order_by(self, field: str, direction: str = "ASC") -> "SQLQueryBuilder":
        self._order_by.append(f"{field} {direction}")
        return self

    def limit(self, n: int) -> "SQLQueryBuilder":
        self._limit = n
        return self

    def offset(self, n: int) -> "SQLQueryBuilder":
        self._offset = n
        return self

    def build(self) -> str:
        if not self._table:
            raise ValueError("FROM table is required")
        fields = ", ".join(self._select_fields) if self._select_fields else "*"
        sql = f"SELECT {fields} FROM {self._table}"
        if self._joins:
            sql += " " + " ".join(self._joins)
        if self._where_clauses:
            sql += " WHERE " + " AND ".join(self._where_clauses)
        if self._order_by:
            sql += " ORDER BY " + ", ".join(self._order_by)
        if self._limit:
            sql += f" LIMIT {self._limit}"
        if self._offset:
            sql += f" OFFSET {self._offset}"
        return sql + ";"


query = (
    SQLQueryBuilder()
    .select("u.id", "u.name", "o.total")
    .from_table("users", alias="u")
    .join("orders o", on="o.user_id = u.id")
    .where("u.active = true")
    .where("o.total > 100")
    .order_by("o.total", "DESC")
    .limit(20)
    .offset(40)
    .build()
)
# SELECT u.id, u.name, o.total FROM users u
# INNER JOIN orders o ON o.user_id = u.id
# WHERE u.active = true AND o.total > 100
# ORDER BY o.total DESC LIMIT 20 OFFSET 40;
```

---

### Prompt Builder for LLMs

```python
@dataclass(frozen=True)
class LLMPrompt:
    system: str
    context_documents: list
    user_message: str
    max_tokens: int
    temperature: float


class PromptBuilder:
    def __init__(self):
        self._system = "You are a helpful assistant."
        self._docs = []
        self._user_message = None
        self._max_tokens = 1000
        self._temperature = 0.7

    def system_prompt(self, prompt: str) -> "PromptBuilder":
        self._system = prompt
        return self

    def add_context(self, document: str) -> "PromptBuilder":
        self._docs.append(document)
        return self

    def user_message(self, message: str) -> "PromptBuilder":
        self._user_message = message
        return self

    def max_tokens(self, n: int) -> "PromptBuilder":
        self._max_tokens = n
        return self

    def temperature(self, t: float) -> "PromptBuilder":
        if not (0.0 <= t <= 2.0):
            raise ValueError("Temperature must be between 0.0 and 2.0")
        self._temperature = t
        return self

    def build(self) -> LLMPrompt:
        if not self._user_message:
            raise ValueError("User message is required")
        return LLMPrompt(
            system=self._system,
            context_documents=list(self._docs),
            user_message=self._user_message,
            max_tokens=self._max_tokens,
            temperature=self._temperature,
        )

prompt = (
    PromptBuilder()
    .system_prompt("You are a senior Python engineer. Answer concisely.")
    .add_context(retrieved_doc_1)
    .add_context(retrieved_doc_2)
    .user_message("How do I implement a thread-safe singleton in Python?")
    .max_tokens(500)
    .temperature(0.3)
    .build()
)
```

---

## The Director Pattern

A `Director` encodes common construction sequences. Clients don't need to know which builder calls to make or in what order:

```python
class HttpRequestDirector:
    """Encodes reusable request construction sequences."""

    @staticmethod
    def build_json_get(url: str, token: str) -> HttpRequest:
        return (
            HttpRequestBuilder()
            .method("GET")
            .url(url)
            .bearer_token(token)
            .header("Accept", "application/json")
            .timeout(5.0)
            .retry_policy(max_retries=3)
            .build()
        )

    @staticmethod
    def build_webhook_post(url: str, payload: dict) -> HttpRequest:
        return (
            HttpRequestBuilder()
            .method("POST")
            .url(url)
            .json_body(payload)
            .timeout(2.0)
            .retry_policy(max_retries=1)  # Webhooks: fail fast
            .build()
        )

# Client uses director — doesn't care about builder details
request = HttpRequestDirector.build_json_get(
    url="https://api.example.com/users/me",
    token=auth_token
)
```

---

## Validation in `build()`

The critical design decision: validate in `build()`, not in setters.

**Validate in `build()` because:**
- Some validations are **cross-field**: `start_date < end_date`, only valid together
- Some fields are **conditionally required**: body only if POST
- You can accumulate errors and report all of them at once
- Setters stay simple — no early-failure confusion

```python
def build(self) -> DateRangeQuery:
    errors = []
    if not self._start:
        errors.append("start_date required")
    if not self._end:
        errors.append("end_date required")
    if self._start and self._end and self._start > self._end:
        errors.append("start_date must be before end_date")
    if errors:
        raise ValueError(f"Invalid query: {'; '.join(errors)}")
    return DateRangeQuery(self._start, self._end, self._granularity)
```

---

## Immutable Objects via Builder

Builder is the standard way to construct immutable objects that need complex configuration:

```python
@dataclass(frozen=True)  # frozen=True makes it immutable — no __setattr__
class ServerConfig:
    host: str
    port: int
    max_connections: int
    ssl_enabled: bool
    allowed_origins: tuple  # tuples are immutable, lists are not

# You can't do: config.port = 9000  → raises FrozenInstanceError
# To "update": create a new builder from existing instance
def with_port(config: ServerConfig, new_port: int) -> ServerConfig:
    return (
        ServerConfigBuilder()
        .from_existing(config)  # copy all fields
        .port(new_port)
        .build()
    )
```

---

## Trade-offs

| Concern | Detail |
|---|---|
| ✅ Eliminates telescoping constructors | No `Person(None, None, None, email, None, ...)` |
| ✅ Readable construction | `builder.method("POST").url(...).body(...)` — self-documenting |
| ✅ Validation at build time | Catch cross-field errors before object is used |
| ✅ Immutable products | Builder accumulates mutable state, `build()` returns frozen object |
| ✅ Reusable partial builders | Configure a base builder, clone for variations |
| ❌ More classes | One builder class per product |
| ❌ Mutable intermediate state | Builder itself is mutable — be careful with shared builders |
| ❌ Overkill for simple objects | 2-3 required fields → just use a constructor with keyword args |

---

## Python Alternatives

For simple cases, Python offers alternatives that avoid the full Builder pattern:

```python
# For objects with mostly-optional fields: dataclass with defaults
@dataclass
class QueryConfig:
    table: str
    limit: int = 100
    offset: int = 0
    order_by: str = "created_at"
    active_only: bool = True

# For truly complex, validated, immutable config: pydantic
from pydantic import BaseModel, validator

class ServerConfig(BaseModel):
    host: str
    port: int
    max_connections: int = 100

    @validator("port")
    def port_must_be_valid(cls, v):
        if not (1 <= v <= 65535):
            raise ValueError("Port must be 1–65535")
        return v

    class Config:
        frozen = True  # Immutable
```

Use Builder when: construction logic is complex, cross-field validation is needed, construction order matters, or you want a fluent interface.

---

## Real-World Examples

| Example | Builder usage |
|---|---|
| `requests.Request` | Builders via session config + `.prepare()` |
| SQLAlchemy `select()` | `select(User).where(User.active == True).order_by(User.name).limit(20)` |
| Python `pathlib.Path` | Chainable `/` operator builds paths |
| Elasticsearch DSL | `Search().query("match", title="python").sort("-date").extra(size=10)` |
| LangChain | `ConversationalRetrievalChain.from_llm(llm, retriever, ...) ` |
| Java `StringBuilder` | `.append().append().toString()` |
| Kotlin `buildString {}` | DSL-style builder |

---

## Interview Q&A

**Q: What problem does the Builder pattern solve?**

A: The telescoping constructor problem. When an object has many optional parameters, a constructor becomes unreadable and error-prone. Builder accumulates state through named methods, performs cross-field validation in `build()`, and produces a clean final object. It also enables immutable products — the builder is mutable, the product is frozen.

**Q: Why put validation in `build()` rather than in each setter?**

A: Some validations are cross-field — `start_date < end_date` can only be checked when both are set. Some fields are conditionally required — body only allowed for POST. Validating in `build()` centralises all validation, can report all errors at once, and keeps setters simple.

**Q: What is the Director in the Builder pattern?**

A: A class that encodes reusable construction sequences. Instead of every client repeating `builder.method("GET").header("Accept","application/json").timeout(5).retry(3)`, the Director's `build_json_get()` method does it. Clients use Directors when construction order or combinations are complex but reused.

**Q: How does Builder differ from Abstract Factory?**

A: Builder constructs one complex object step-by-step — the process takes multiple calls and ends with `build()`. Abstract Factory creates multiple distinct compatible objects, each in a single call. Builder is about the process of constructing one thing; Abstract Factory is about ensuring a family of things are compatible.

**Q: When is Builder overkill?**

A: When the object has 2–3 fields, all required. A plain `__init__` with keyword args is clearer. Python dataclasses with defaults handle most "many optional fields" cases. Reach for Builder when you need: cross-field validation at construction time, a fluent API, an immutable product, or different "configurations" of the same complex object.

---

## Numbers to Remember

- Telescoping constructor smell: more than **4–5 constructor parameters** — consider Builder or dataclass
- `@dataclass(frozen=True)` uses `__setattr__` override — immutability enforced at runtime (not compile time)
- Fluent interface: each setter returns `self` — O(1) overhead per call
- Validation in `build()`: O(1) — check fields, raise or return