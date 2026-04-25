# Tofu Summary for Odin Audience

## What is Tofu?

An async messaging protocol library. Message-oriented, not stream-oriented.

## Why "Tofu"?

Minimal ingredients, takes the flavor of what you add to it.

## Core Concepts

### Three Primitives

| Primitive | Purpose |
|-----------|---------|
| **Ampe** (Engine) | Owns resources, runs poll loop in background thread |
| **ChannelGroup** | Async message exchange interface |
| **Message** | Data container + command to engine |

### Four Operations

| Operation | Purpose |
|-----------|---------|
| `get()` | Get message from pool |
| `put()` | Return message to pool |
| `post()` | Submit message for async processing |
| `waitReceive()` | Wait for incoming messages/completions |

### Message Types

| Type | Purpose |
|------|---------|
| WelcomeRequest/Response | Start listener (internal) |
| HelloRequest/Response | Connect to peer (over network) |
| Request/Response | Application data exchange |
| Signal | One-way notification |
| ByeRequest/Response | Graceful close |
| ByeSignal | Immediate close |

## Key Design Decisions

1. **Single thread reactor** — All socket I/O in one background thread
2. **Message pool** — Pre-allocated messages reduce runtime allocations
3. **Channel abstraction** — Work with channels, tofu manages sockets
4. **Async completion** — `post()` submits, `waitReceive()` gets results
5. **Symmetric peers** — After handshake, either side can send any message type

## Memory Model

- Engine owns allocator
- Messages allocated from pool or on-demand
- `put()` returns messages to pool
- Pool has configurable initial/max sizes
- `pool_empty` signal when receive pool exhausted

## What Tofu is NOT

- Not an actor framework
- Not a message queue (no persistence)
- Not HTTP-based
- Not trying to be zeromq/nanomsg

## Suitable For

- Custom protocols between services
- Game networking
- IoT device communication
- Any scenario needing message-oriented async I/O
