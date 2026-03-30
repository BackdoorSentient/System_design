# 05_api_design.md — API Design

---

**Q: What are REST best practices for designing a production-grade API?**

REST (Representational State Transfer) is an architectural style built on HTTP. Well-designed REST APIs are intuitive, consistent, and evolvable.

**Resource naming**:
- Use nouns, not verbs: `/users`, `/orders`, not `/getUsers`, `/createOrder`.
- Plural nouns: `/users/123`, not `/user/123`.
- Hierarchical resources: `/users/123/orders`, `/orders/456/items`.
- Use kebab-case for multi-word: `/user-profiles`, not `/userProfiles`.

**HTTP method semantics**:
- `GET`: Retrieve. Safe and idempotent. No side effects.
- `POST`: Create a new resource. Not idempotent.
- `PUT`: Full replacement of a resource. Idempotent.
- `PATCH`: Partial update. May or may not be idempotent depending on implementation.
- `DELETE`: Remove. Idempotent (deleting twice = same result).

**HTTP status codes**:
- `200 OK`, `201 Created`, `204 No Content` (for successful DELETE/PUT).
- `400 Bad Request` (malformed input), `401 Unauthorized` (not authenticated), `403 Forbidden` (authenticated but not allowed), `404 Not Found`, `409 Conflict` (duplicate, optimistic lock failure), `422 Unprocessable Entity` (validation error), `429 Too Many Requests`.
- `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`.

**Consistent error format**:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is invalid",
    "field": "email",
    "request_id": "req_abc123"
  }
}
```

**Versioning**: Use URL path versioning (`/v1/users`) for major breaking changes. Header versioning (`API-Version: 2024-01-01`) for evolution without path clutter (Stripe's approach).

**HATEOAS**: Include links to related resources in responses. Rarely used in practice but theoretically correct REST.

---

**Q: What is idempotency in APIs? How do you implement it for POST requests?**

**Idempotency**: Making the same request multiple times produces the same result as making it once. GET, PUT, and DELETE are naturally idempotent. POST is not — calling `POST /orders` twice creates two orders.

**Why it matters**: In distributed systems, networks are unreliable. A client sends a request, gets a timeout, and doesn't know if the server processed it. Should it retry? If the operation is not idempotent, retrying doubles the charge, creates duplicate orders, etc.

**Implementation with idempotency keys**:
1. Client generates a unique `Idempotency-Key` (UUID) and includes it in the request header.
2. Server checks if it has already processed a request with that key.
   - If yes: Return the cached response (same status code and body as the original).
   - If no: Process the request, store the response keyed by the idempotency key (in Redis with TTL), return the response.
3. Client can safely retry on timeout — subsequent retries return the cached response.

```
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Response:
{
  "payment_id": "pay_xyz",
  "status": "succeeded"
}
```

Retry with the same key → same `{"payment_id": "pay_xyz", "status": "succeeded"}` response, no duplicate charge.

**Stripe, Braintree, and Twilio** all implement idempotency keys. Stripe stores keys for 24 hours.

**Storage**: Redis `SETNX` (set if not exists) with TTL. On the first request, acquire the key. On subsequent requests, return the stored response.

---

**Q: How does gRPC work and how does it compare to REST/JSON?**

gRPC is a high-performance RPC framework developed by Google. It uses:
- **HTTP/2** as the transport (multiplexed streams, header compression, binary framing).
- **Protocol Buffers (protobuf)** as the serialization format (compact binary, ~3–10x smaller than JSON).
- **IDL (Interface Definition Language)**: Service and message types are defined in `.proto` files, which generate client and server code in any supported language.

**Comparison**:

| Dimension | REST/JSON | gRPC |
|---|---|---|
| Transport | HTTP/1.1 or HTTP/2 | HTTP/2 only |
| Serialization | JSON (text, human-readable) | Protobuf (binary, compact) |
| Schema | Optional (OpenAPI) | Required (.proto) |
| Type safety | Loose (at runtime) | Strong (compile-time) |
| Streaming | Awkward (SSE, polling) | Native bidirectional streaming |
| Browser support | Excellent | Limited (needs gRPC-Web proxy) |
| Performance | Good | Excellent (~5–10x faster, 3x smaller payloads) |
| Human readability | Easy to debug with curl | Requires tooling |
| Code generation | Optional | First-class (auto-generates client stubs) |

**gRPC streaming types**:
- Unary: client sends one request, gets one response (like REST).
- Server streaming: one request, stream of responses (e.g., subscribe to live updates).
- Client streaming: stream of requests, one response (e.g., bulk upload).
- Bidirectional streaming: both sides stream simultaneously (e.g., real-time chat).

**When to use gRPC**:
- Internal microservice-to-microservice communication where performance matters.
- Polyglot environments where strong type contracts across languages are needed.
- When you need streaming (server push, real-time updates).

**When to use REST**:
- Public APIs (browser-friendly, no tooling required).
- Simple CRUD operations where developer experience is paramount.
- When consumers span many different tech stacks and you want maximum compatibility.

---

**Q: What is GraphQL? What are its trade-offs vs REST?**

GraphQL is a query language for APIs where the client specifies exactly what data it needs in a single request. Developed by Facebook (2015).

**Core concepts**:
- **Schema**: A type system defining all available data and operations.
- **Query**: Read-only fetch. Client specifies exactly the fields it wants.
- **Mutation**: Write operation.
- **Subscription**: Real-time event stream.
- **Resolver**: Server-side function that fetches the data for each field.

**Example**:
```graphql
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
      items { name, price }
    }
  }
}
```
This single request replaces what would be multiple REST calls: `GET /users/123`, `GET /users/123/orders`, `GET /orders/id/items`.

**Benefits**:
- **No over-fetching**: Mobile client only gets the fields it needs (saves bandwidth).
- **No under-fetching**: One request for nested data (no N+1 REST round trips).
- **Schema is the contract**: Self-documenting, introspectable. Great developer experience.
- **Rapid frontend iteration**: Frontend team can add new fields without waiting for backend API changes.

**Problems**:
- **N+1 problem on the server**: Each resolver independently fetches data. Loading 10 users each with their orders triggers 1 + 10 DB queries. Solved with **DataLoader** (batches and deduplicates DB calls).
- **Authorization complexity**: In REST, each endpoint can have its own auth logic. In GraphQL, you must authorize at the field/resolver level.
- **Caching is harder**: REST leverages HTTP caching (URL-keyed). GraphQL uses `POST /graphql` — every request to the same URL may have different query content. Need query-level caching (Apollo, persisted queries).
- **Rate limiting**: Hard to limit by "number of requests" — a single GraphQL query can be arbitrarily expensive. Use query complexity analysis + depth limiting.
- **Overly complex for simple APIs**: If your API is simple CRUD, GraphQL overhead isn't worth it.

**When to use GraphQL**: Public APIs consumed by many diverse clients (web, iOS, Android) with different data needs. Facebook, GitHub, Shopify, Twitter use GraphQL for their developer APIs.

---

**Q: How do you design pagination for APIs? Compare cursor-based vs offset-based.**

**Offset/limit pagination**:
```
GET /posts?limit=20&offset=40
```
- Simple to implement.
- Problems:
  1. **Skipped/duplicated items**: If items are inserted while paginating, items shift. Page 3 may contain items already seen on page 2.
  2. **Performance**: `OFFSET 10000` in SQL must scan and discard 10,000 rows before returning results. Very slow for deep pages.
  3. **Total count**: Providing a total count requires an extra `COUNT(*)` query.

**Cursor-based pagination**:
```
GET /posts?limit=20&cursor=eyJpZCI6MTAwfQ==  # base64 of {"id": 100}
```
- The cursor encodes the position (e.g., the `id` or `created_at` of the last seen item).
- The query is `WHERE id > cursor_id ORDER BY id LIMIT 20`.
- Benefits: Stable results even as data changes. Efficient — uses an index seek rather than a scan. Works for real-time feeds.
- Limitations: Cannot jump to page 5 directly (no random access). Cursor must encode sort position, so changing sort order invalidates cursors.

**Page-based pagination**: Like offset but `page=3` instead of `offset=40`. Same problems as offset.

**Keyset pagination** (database term for cursor-based): Where the cursor is the actual sort key value (not an opaque token). More explicit but requires clients to know the sort key.

**API response format**:
```json
{
  "data": [...],
  "pagination": {
    "has_next_page": true,
    "next_cursor": "eyJpZCI6MTIwfQ==",
    "total_count": 1540  // only if you can afford the COUNT(*)
  }
}
```

**Rule of thumb**: Use cursor-based for feeds and real-time data. Offset is acceptable for admin panels where deep-page performance isn't critical.

---

**Q: What are API versioning strategies and their trade-offs?**

**URL path versioning**: `/v1/users`, `/v2/users`
- Pros: Explicit, easy to see, works with all HTTP tooling, simple to route.
- Cons: Breaks REST purity (the version is not a resource attribute). Clients must update URLs on upgrade.
- Used by: Twitter, Twilio, most public APIs.

**HTTP header versioning**: `Accept: application/vnd.api.v2+json` or `API-Version: 2`
- Pros: URL stays clean. Resources are not "versioned" at the path level.
- Cons: Harder to test with browser/curl. Less visible. Requires custom routing logic.
- Used by: GitHub API.

**Query parameter versioning**: `GET /users?version=2`
- Pros: Easy to test.
- Cons: Gets lost in logs. Not RESTful. Easy to miss.

**Date-based versioning**: `Stripe-Version: 2023-08-16`
- Each version is a date. The API evolves continuously; breaking changes are gated behind a newer date. Your account's "version" is set when you create the API key and doesn't change unless you opt in.
- Pros: Granular, additive-only changes. Old clients never break unexpectedly.
- Cons: Requires maintaining the semantics of every past version in the codebase.

**Avoiding versioning through additive changes**:
- Never remove or rename existing fields. Only add new ones.
- Make all new fields optional with sensible defaults.
- Use the `Sunset` HTTP header to announce deprecation timelines.

---

**Q: What are the key considerations for API rate limiting, authentication, and security?**

**Authentication**:
- **API Keys**: Simple. Generated token sent in `Authorization: Bearer <key>` header or query param. Easy to revoke. No expiry unless explicitly implemented.
- **JWT (JSON Web Tokens)**: Self-contained token with claims (user ID, roles, expiry). Stateless — server doesn't need to look up the token. Risk: Tokens are valid until expiry even if the user is banned (mitigation: short expiry + refresh tokens, or token blocklist).
- **OAuth 2.0**: Delegated authorization. For third-party apps acting on behalf of a user. Google, GitHub login. Involves access tokens + refresh tokens.

**Security headers**:
- `HTTPS only` — never expose APIs over HTTP.
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- CORS: Restrict `Access-Control-Allow-Origin` to known client domains.

**Input validation**:
- Validate all inputs server-side. Never trust the client.
- Limit request body size (e.g., Nginx `client_max_body_size 1m`).
- Sanitize string inputs to prevent injection.

**Rate limiting**: (see dedicated topic). Apply at the API gateway level. Return `429 Too Many Requests` with `Retry-After` header.

**Sensitive data**: Never return passwords, internal IDs, or PII in error messages. Log request IDs not full payloads. Mask card numbers, SSNs in logs.
