# Websockets

## The One-Line Summary

> WebSockets replace repeated polling with a single persistent bidirectional connection. The server pushes data to the client instantly without the client asking.
> 

---

## The Problem — Polling

Traditional HTTP is request-response. Client asks, server answers. For real-time apps like chat, the client has to keep asking "any new messages?"

**Regular Polling:**

Client asks every second. Server responds immediately — even if empty.

- 10 million users polling every second = 10 million requests/sec
- Most responses are empty — pure waste
- Not truly real-time — up to 1 second delay

**Long Polling:**

Client asks. Server holds the connection open up to 30 seconds waiting for a message. If message arrives — responds immediately. If nothing — responds empty after 30 seconds, client polls again.

- Fewer empty responses — more efficient
- Still half-duplex — client must always initiate
- Still holds connections open — server resource burden at scale

---

## WebSockets — The Solution

WebSocket upgrades an HTTP connection into a **persistent, full-duplex connection**.

```
Client → Server: "HTTP Upgrade to WebSocket"
Server → Client: "Upgrade accepted"
--- persistent connection established ---
Server → Client: "New message from User A" (server pushes anytime)
Client → Server: "Message read" (client sends anytime)
Server → Client: "User A is typing..."
```

- **Persistent** — one connection stays open for the entire session
- **Full-duplex** — both sides send anytime without waiting
- **No repeated handshakes** — one connection, no overhead per message
- **Truly real-time** — server pushes instantly when something happens

---

## The Concurrency Problem at Scale

10 million WebSocket connections = 10 million open connections on your servers.

**Traditional servers (Java thread-per-connection):**

- Each thread ~1MB memory
- 10 million threads = 10TB memory — impossible
- OS context switching overhead between threads

**The solution — lightweight concurrency:**

| Approach | How | Memory per connection | Scale |
| --- | --- | --- | --- |
| Java threads | OS-managed | ~1MB | ~10K connections |
| Node.js event loop | Non-blocking callbacks, single thread | Very low | ~100K connections |
| Go goroutines | Go runtime-managed, not OS | ~2KB | ~1M connections |
| Erlang processes | Erlang VM-managed | ~300 bytes | Millions |

**Why Go goroutines are different from Java threads:**

- OS threads wait idle consuming 1MB even when doing nothing
- Go runtime **parks** waiting goroutines — frees the OS thread for other goroutines
- No expensive OS context switch — Go scheduler handles it internally
- Result: millions of concurrent connections cheaply

**WhatsApp** used Erlang for the same reason — same lightweight process model. 55 engineers, 450 million users.

---

## Polling vs Long Polling vs WebSockets

|  | Regular Polling | Long Polling | WebSockets |
| --- | --- | --- | --- |
| **How** | Client asks every N seconds | Client asks, server waits up to 30s | Persistent open connection |
| **Empty responses** | Many | Few | None |
| **Real-time** | No — up to N second delay | Near real-time | Yes — instant |
| **Direction** | Half-duplex | Half-duplex | Full-duplex |
| **Server load** | High | Medium | Low per message, high connections |
| **Complexity** | Low | Low | Medium |

---

## When to Use What

| Scenario | Use |
| --- | --- |
| Chat app, live notifications | WebSockets |
| Collaborative document editing | WebSockets |
| Live sports scores | WebSockets or long polling |
| Dashboard refreshing every 30 seconds | Regular polling |
| Simple status check | Regular polling |
| Low user base, infrequent updates | Regular polling |
| Corporate networks blocking WebSockets | Long polling |

---

## Three-Question Ritual

**What problem do WebSockets solve?**

Repeated polling wastes server resources with empty responses and isn't truly real-time. WebSockets establish a persistent bidirectional connection so the server pushes data instantly without the client asking.

**What breaks without WebSockets in a chat app?**

At millions of users polling every second — millions of empty requests flood the server. Messages aren't truly real-time. Server costs explode.

**When NOT to use WebSockets?**

Low user base where polling is sufficient. When real-time isn't required (30 second refresh is fine). When infrastructure (corporate firewalls) blocks WebSocket connections.

---

## AWS Equivalents

| Concept | AWS Service |
| --- | --- |
| WebSocket API | Amazon API Gateway WebSocket API |
| Long polling | Amazon SQS long polling (built-in) |
| Real-time pub/sub | AWS AppSync (GraphQL subscriptions) |
| IoT real-time | AWS IoT Core |