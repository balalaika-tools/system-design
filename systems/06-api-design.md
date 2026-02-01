# API Design: REST, GraphQL, and gRPC

> **Goal**: Understand the trade-offs between API paradigms and how to design APIs that are intuitive, performant, and evolvable.

---

## 1. API Fundamentals

### What Makes a Good API?

```
┌─────────────────────────────────────────────────┐
│  GOOD API CHARACTERISTICS                       │
│                                                 │
│  • Intuitive — easy to understand and use       │
│  • Consistent — predictable patterns            │
│  • Documented — clear contracts                 │
│  • Versioned — evolvable without breaking       │
│  • Performant — efficient data transfer         │
│  • Secure — authentication & authorization      │
└─────────────────────────────────────────────────┘
```

### API Styles Overview

| Style | Best For | Data Format | Protocol |
|-------|----------|-------------|----------|
| **REST** | CRUD, web/mobile apps | JSON | HTTP |
| **GraphQL** | Complex UIs, flexible queries | JSON | HTTP |
| **gRPC** | Microservices, high performance | Protobuf | HTTP/2 |
| **WebSocket** | Real-time, bidirectional | Any | WS |

---

## 2. REST (Representational State Transfer)

### Core Principles

```
┌─────────────────────────────────────────────────┐
│  REST = Resources + HTTP Methods + Status Codes │
│                                                 │
│  Resource: /users/123                           │
│  Method:   GET, POST, PUT, PATCH, DELETE        │
│  Response: JSON + HTTP Status                   │
└─────────────────────────────────────────────────┘
```

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Read resource | ✅ | ✅ |
| POST | Create resource | ❌ | ❌ |
| PUT | Replace resource | ✅ | ❌ |
| PATCH | Partial update | ❌ | ❌ |
| DELETE | Remove resource | ✅ | ❌ |

**Idempotent**: Same request, same result (can retry safely)
**Safe**: No side effects (read-only)

### URL Design

```
# ✅ Good: Nouns, plural, hierarchical
GET    /users              # List users
GET    /users/123          # Get user
POST   /users              # Create user
PUT    /users/123          # Replace user
PATCH  /users/123          # Update user fields
DELETE /users/123          # Delete user

GET    /users/123/orders   # User's orders
GET    /orders/456/items   # Order's items

# ❌ Bad: Verbs, actions, inconsistent
GET    /getUser?id=123
POST   /createUser
GET    /user/123           # Singular (inconsistent)
POST   /users/123/delete   # Action in URL
```

### Query Parameters

```
# Filtering
GET /orders?status=pending&created_after=2024-01-01

# Pagination
GET /users?page=2&limit=20
GET /users?cursor=abc123&limit=20  # Cursor-based (preferred)

# Sorting
GET /products?sort=price&order=desc

# Field selection
GET /users/123?fields=id,name,email

# Search
GET /products?search=laptop
```

### HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | Successful POST |
| **204** | No Content | Successful DELETE |
| **400** | Bad Request | Invalid input |
| **401** | Unauthorized | Not authenticated |
| **403** | Forbidden | Not authorized |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Resource state conflict |
| **422** | Unprocessable | Validation failed |
| **429** | Too Many Requests | Rate limited |
| **500** | Server Error | Unexpected error |

### Response Format

```json
// Success
{
  "data": {
    "id": "123",
    "name": "Alice",
    "email": "alice@example.com"
  }
}

// Success (list)
{
  "data": [...],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "next_cursor": "abc123"
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      {"field": "email", "message": "Required field"}
    ]
  }
}
```

### REST Anti-Patterns

```
❌ GET /users/123/delete           # Don't use GET for mutations
❌ POST /search                     # Use GET with query params
❌ PUT /users (update all users)   # Too dangerous
❌ Nested resources too deep       # /a/1/b/2/c/3/d/4
❌ Returning 200 with error body   # Use proper status codes
```

---

## 3. GraphQL

### Why GraphQL?

```
┌─────────────────────────────────────────────────────────────────────┐
│  REST Problem: Over-fetching and Under-fetching                     │
│                                                                     │
│  Mobile app needs: user name + last 3 orders                        │
│                                                                     │
│  REST requires:                                                     │
│    GET /users/123              # Get all user fields (too much)     │
│    GET /users/123/orders       # Get all orders (too many)          │
│    GET /orders/1/items         # Get items for each order           │
│    GET /orders/2/items         # 3 more requests...                 │
│    GET /orders/3/items                                              │
│                                                                     │
│  GraphQL: One request, exactly what you need                        │
└─────────────────────────────────────────────────────────────────────┘
```

### GraphQL Basics

```graphql
# Schema defines types and operations
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Order {
  id: ID!
  total: Float!
  items: [OrderItem!]!
}

type Query {
  user(id: ID!): User
  users(limit: Int): [User!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}
```

```graphql
# Client query - get exactly what you need
query {
  user(id: "123") {
    name
    orders(limit: 3) {
      id
      total
    }
  }
}

# Response - exactly what was requested
{
  "data": {
    "user": {
      "name": "Alice",
      "orders": [
        {"id": "1", "total": 99.99},
        {"id": "2", "total": 49.99},
        {"id": "3", "total": 149.99}
      ]
    }
  }
}
```

### Mutations

```graphql
mutation {
  createUser(input: {
    name: "Bob"
    email: "bob@example.com"
  }) {
    id
    name
  }
}
```

### GraphQL Trade-offs

| Pros | Cons |
|------|------|
| Flexible queries | Complex to implement |
| No over-fetching | N+1 query problem |
| Strong typing | Caching is harder |
| Self-documenting | HTTP caching doesn't work |
| Single endpoint | Query complexity attacks |

### When to Use GraphQL

✅ **Good fit:**
- Complex UIs with varying data needs
- Mobile apps (bandwidth sensitive)
- Multiple client types with different needs
- Rapid frontend iteration

❌ **Not ideal:**
- Simple CRUD APIs
- File uploads
- Real-time (WebSockets often better)
- Server-to-server communication

---

## 4. gRPC

### Why gRPC?

```
┌─────────────────────────────────────────────────┐
│  High-performance, strongly-typed RPC           │
│                                                 │
│  • Binary protocol (Protobuf) — smaller, faster │
│  • HTTP/2 — multiplexing, streaming             │
│  • Code generation — type-safe clients          │
│  • Bidirectional streaming                      │
└─────────────────────────────────────────────────┘
```

### Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  string id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
}
```

### Generated Code

```python
# Python client (generated)
from user_pb2_grpc import UserServiceStub
from user_pb2 import GetUserRequest

channel = grpc.insecure_channel('localhost:50051')
stub = UserServiceStub(channel)

user = stub.GetUser(GetUserRequest(id="123"))
print(user.name)
```

### gRPC Streaming

```
┌─────────────────────────────────────────────────┐
│  STREAMING TYPES                                │
│                                                 │
│  Unary:           Client → Server → Response    │
│  Server stream:   Client → Server →→→ Responses │
│  Client stream:   Client →→→ Server → Response  │
│  Bidirectional:   Client ↔↔↔ Server             │
└─────────────────────────────────────────────────┘
```

### gRPC Trade-offs

| Pros | Cons |
|------|------|
| Fast (binary, HTTP/2) | Not browser-friendly |
| Type-safe | Learning curve |
| Streaming support | Harder to debug |
| Code generation | Need proto files |

### When to Use gRPC

✅ **Good fit:**
- Microservices communication
- High-performance requirements
- Streaming data
- Polyglot environments (many languages)

❌ **Not ideal:**
- Public APIs (browsers can't use directly)
- Simple web applications
- Human debugging needed

---

## 5. API Versioning

### URL Versioning

```
GET /v1/users/123
GET /v2/users/123
```

**Pros:** Clear, easy to implement
**Cons:** URL pollution, breaks REST purity

### Header Versioning

```
GET /users/123
Accept: application/vnd.api+json; version=2
```

**Pros:** Clean URLs
**Cons:** Harder to test, less visible

### Query Parameter

```
GET /users/123?version=2
```

**Pros:** Easy to switch
**Cons:** Optional, easy to forget

### Recommendation

```
URL versioning (/v1/) is most common and practical.
Only increment major version for breaking changes.
Support N-1 version during transition period.
```

---

## 6. Pagination

### Offset-Based

```
GET /users?page=3&limit=20

# Response
{
  "data": [...],
  "pagination": {
    "total": 500,
    "page": 3,
    "limit": 20,
    "total_pages": 25
  }
}
```

**Problem:** Inconsistent with real-time data (items shift as you paginate)

### Cursor-Based (Recommended)

```
GET /users?cursor=eyJpZCI6MTIzfQ&limit=20

# Response
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

**Pros:**
- Consistent with real-time data
- Efficient for large datasets
- Works with infinite scroll

---

## 7. Rate Limiting

### Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1609459200
```

### 429 Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Retry after 60 seconds."
  }
}
```

### Strategies

| Strategy | Description |
|----------|-------------|
| Fixed Window | N requests per time window |
| Sliding Window | N requests in rolling time period |
| Token Bucket | Tokens replenish, each request costs token |
| Leaky Bucket | Requests queue, process at fixed rate |

---

## 8. Authentication & Authorization

### Authentication Methods

| Method | How | Best For |
|--------|-----|----------|
| **API Key** | Header: `X-API-Key: xxx` | Server-to-server |
| **Bearer Token** | Header: `Authorization: Bearer xxx` | User sessions |
| **OAuth 2.0** | Token exchange flow | Third-party access |
| **JWT** | Self-contained token | Stateless auth |

### JWT Structure

```
header.payload.signature

# Decoded
{
  "header": {"alg": "HS256", "typ": "JWT"},
  "payload": {
    "sub": "user123",
    "exp": 1609459200,
    "roles": ["user", "admin"]
  },
  "signature": "..."
}
```

### Authorization Patterns

```
# Role-Based Access Control (RBAC)
user.roles = ["editor"]
if "editor" in user.roles:
    allow_edit()

# Attribute-Based Access Control (ABAC)
if user.department == resource.department and user.level >= 3:
    allow_access()
```

---

## 9. API Design Checklist

```
□ Consistent naming conventions
□ Proper HTTP methods and status codes
□ Pagination for list endpoints
□ Filtering, sorting, field selection
□ Versioning strategy
□ Rate limiting
□ Authentication/Authorization
□ Error response format
□ Request validation
□ Documentation (OpenAPI/Swagger)
□ CORS configuration
□ Compression (gzip)
□ Caching headers
```

---

## 10. Comparison Table

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Protocol** | HTTP | HTTP | HTTP/2 |
| **Data format** | JSON | JSON | Protobuf |
| **Schema** | Optional (OpenAPI) | Required | Required (Proto) |
| **Caching** | HTTP caching | Custom | Custom |
| **Browser support** | ✅ Native | ✅ Native | ❌ Needs proxy |
| **Learning curve** | Low | Medium | Medium |
| **Flexibility** | Fixed endpoints | Flexible queries | Fixed methods |
| **Performance** | Good | Good | Excellent |
| **Streaming** | Limited | Subscriptions | Native |
| **Best for** | Web apps, CRUD | Complex UIs | Microservices |

---

## 11. Real-World Architecture

### Public API: REST

```
Mobile App ──→ API Gateway ──→ REST API ──→ Services
Web App   ──┘
```

### Internal: gRPC

```
Service A ──gRPC──→ Service B
          ──gRPC──→ Service C
```

### Complex UI: GraphQL

```
React App ──→ GraphQL Gateway ──→ REST/gRPC Services
```

### Combined

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Public:   REST API (simple, cacheable, well-understood)            │
│                ↓                                                    │
│  Gateway:  GraphQL (aggregate, transform for frontend needs)        │
│                ↓                                                    │
│  Internal: gRPC (fast, typed, efficient between services)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

| API Style | When to Use |
|-----------|-------------|
| **REST** | Default choice for web APIs, CRUD, public APIs |
| **GraphQL** | Complex frontends, multiple clients, flexible queries |
| **gRPC** | Microservices, performance-critical, streaming |

**Key Principles:**
- Use nouns for resources, HTTP verbs for actions
- Return proper status codes
- Version your API from day one
- Paginate large collections
- Rate limit to protect your system
- Document everything

---

## Quick Reference

### REST Endpoint Template

```
GET    /v1/{resource}          # List
POST   /v1/{resource}          # Create
GET    /v1/{resource}/{id}     # Read
PUT    /v1/{resource}/{id}     # Replace
PATCH  /v1/{resource}/{id}     # Update
DELETE /v1/{resource}/{id}     # Delete
```

### Common Headers

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {token}
X-Request-ID: {uuid}
```

### Interview Talking Points

1. "REST is resource-oriented; use nouns for URLs and HTTP methods for actions"
2. "GraphQL solves over-fetching/under-fetching but adds complexity and breaks HTTP caching"
3. "gRPC uses binary Protobuf over HTTP/2—fast but not browser-friendly"
4. "Version APIs in the URL (/v1/) and support N-1 version during transitions"
5. "Cursor-based pagination is more reliable than offset-based for real-time data"
