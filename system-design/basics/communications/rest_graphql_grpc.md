# Rest Vs GraphQL vs gRPC

## The One-Line Summary

> REST is simple and universal. GraphQL lets clients request exactly the data they need. gRPC is fast binary communication for internal services. Pick based on who is consuming your API and what performance you need.
> 

---

## The Problem REST Alone Doesn't Solve

You have a user profile page needing: name, photo, last 3 posts, follower count.

With REST you might call:

- `GET /users/123`
- `GET /users/123/posts`
- `GET /users/123/followers`

Three calls for one screen. You could build an aggregator endpoint — but now you have a web app, mobile app, and smartwatch all needing different data shapes from the same endpoint.

**Over-fetching:** Smartwatch gets name, photo, posts, bio, location — but only needs name and photo. Wasted bandwidth.

**Under-fetching:** Web app needs 3 separate calls to build one screen.

---

## REST

**What it is:** Standard HTTP API. Each resource has its own endpoint. Returns JSON.

**Best for:**

- Public APIs for third party developers
- Simple CRUD operations
- When caching at CDN level is needed

**Strengths:**

- Human readable JSON
- Every tool supports it (Postman, curl, browser)
- HTTP caching works naturally
- Low learning curve

**Weaknesses:**

- Over-fetching and under-fetching
- Multiple endpoints for different data needs
- Versioning gets messy (`/v1/users`, `/v2/users`)

---

## GraphQL

**What it is:** One endpoint. Client specifies exactly what data it needs in the query. Server returns exactly that.

**Example — mobile query:**

```graphql
query {
  user(id: "123") {
    name
    photo
    posts(last: 3) { title }
    followerCount
  }
}
```

**Example — smartwatch query:**

```graphql
query {
  user(id: "123") {
    name
    photo
  }
}
```

Same endpoint. Different data returned. No over-fetching.

**Best for:**

- Multiple clients needing different data shapes (web, mobile, smartwatch)
- Rapidly evolving APIs
- Complex nested data relationships

**Weaknesses:**

- Harder to cache — uses POST requests, CDN can't cache dynamic query bodies
- N+1 problem — querying 10 users + their posts = 1 + 10 DB queries (solvable with DataLoader)
- Schema maintenance overhead
- Overkill for simple APIs

---

## gRPC

**What it is:** High performance RPC framework using Protocol Buffers (binary) over HTTP/2.

**How it works:**

1. Define schema in a `.proto` file
2. Code generator creates client and server code automatically
3. Server serializes response to binary
4. Client deserializes binary back to object using the same schema
5. Both sides must share and keep the `.proto` schema in sync

**Why faster than REST:**

- **Protobuf** — binary format, 3-10x smaller than JSON, 5-10x faster to serialize/deserialize
- **HTTP/2 multiplexing** — multiple requests over one connection simultaneously (vs HTTP/1.1 one request at a time)
- **Streaming** — server, client, or bidirectional streaming natively supported

**HTTP versions recap:**

| Version | Key feature |
| --- | --- |
| HTTP/1.1 | One request at a time per connection |
| HTTP/2 | Multiplexing — many requests over one connection simultaneously |
| HTTP/3 | HTTP/2 + QUIC protocol, faster on unreliable networks |
| HTTPS | HTTP + TLS encryption. Always use in production. |

**Best for:**

- Internal service-to-service communication
- High throughput, low latency requirements
- Streaming (live location, real-time data)

**NOT for:**

- Public APIs — third party developers don't have your `.proto` schema
- Simple low-traffic services where complexity isn't justified

**Weaknesses:**

- Proto file management across teams
- Harder to debug — binary not human readable
- Requires HTTP/2 support in infrastructure

---

## Decision Framework

| Scenario | Pick |
| --- | --- |
| Public API for third party developers | REST |
| Multiple clients needing different data shapes | GraphQL |
| Internal microservices, high throughput | gRPC |
| Simple CRUD, stable data shape | REST |
| Live streaming, bidirectional communication | gRPC |
| Heavy CDN caching needed | REST |

---

## Quick Comparison

|  | REST | GraphQL | gRPC |
| --- | --- | --- | --- |
| **Data format** | JSON | JSON | Binary (protobuf) |
| **Endpoint** | Multiple | Single | Generated from schema |
| **Caching** | Easy | Hard | N/A |
| **Learning curve** | Low | Medium | High |
| **Streaming** | Limited | Limited | Native |
| **Best for** | Public APIs | Multiple clients | Internal services |

---

## Three-Question Ritual

**What problem does GraphQL solve?**

Over-fetching and under-fetching — different clients needing different data shapes from the same API without multiple endpoints or wasted bandwidth.

**What breaks without gRPC in high-throughput internal services?**

REST with JSON adds significant CPU and bandwidth overhead at millions of requests per second. Verbose JSON parsing and HTTP/1.1 connection overhead accumulate at scale.

**When NOT to use GraphQL?**

When all consumers need the same data shape, simple CRUD operations, or when CDN caching is a priority.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| REST API | Amazon API Gateway (REST) |
| GraphQL API | AWS AppSync — managed GraphQL service |
| gRPC | Amazon EKS / ECS with gRPC support, AWS App Mesh |