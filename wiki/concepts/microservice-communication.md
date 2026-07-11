---
title: "Microservice Communication Patterns"
category: concept
tags: [microservices, rest, grpc, sqs, sns, kinesis, kafka, event-driven, async, sync]
sources: []
updated: 2026-07-11
---

# Microservice Communication Patterns

Two fundamental axes: **synchronous** (caller waits for a response) and **asynchronous** (caller fires and moves on). Most real systems use both.

---

## Synchronous Communication

### How it works

```
Service A                        Service B
   │                                │
   │──── HTTP/gRPC request ────────►│
   │                                │  (A is blocked, waiting)
   │                                │  processes...
   │◄─── response ─────────────────│
   │                                │
   │  (A continues)                 │
```

### REST (HTTP/JSON)

```
┌─────────────┐    GET /users/42     ┌─────────────┐
│  Order Svc  │ ──────────────────► │  User Svc   │
│             │ ◄────────────────── │             │
└─────────────┘   { id: 42, ... }   └─────────────┘

+ Simple, universal, human-readable, browser-friendly
+ Easy to debug (curl it, read it)
- Tight coupling: Order Svc must know User Svc's URL
- Both services must be up at the same time
```

### gRPC (HTTP/2 + Protocol Buffers)

```
┌─────────────┐   GetUser(id=42)    ┌─────────────┐
│  Order Svc  │ ──────────────────► │  User Svc   │
│  [binary]   │ ◄────────────────── │  [binary]   │
└─────────────┘   User{id:42,...}   └─────────────┘

+ Much faster than REST (binary, multiplexed HTTP/2)
+ Strongly typed contracts (.proto files)
+ Native streaming support
- Binary protocol — hard to debug without tooling
- Not browser-friendly (needs gRPC-Web proxy)
- .proto files must be shared across teams
```

### The cascading failure problem

```
User clicks "Place Order"

Order Svc ──► Payment Svc ──► Fraud Svc ──► Risk API (slow)
   │               │               │              │
   │               │               │         5s timeout
   │               │          waiting...          │
   │          waiting...           │              │
   waiting...      │               │              │
                   ▼               ▼              ▼
              ⏱ timeout       ⏱ timeout      ⏱ timeout

Result: one slow external API stalls the entire chain.
User sees a spinning loader, then an error.
```

---

## Asynchronous Communication

### How it works

```
Service A                   Message Broker            Service B
   │                              │                       │
   │──── publish event ──────────►│                       │
   │                              │                       │
   │  (A continues immediately)   │──── deliver ─────────►│
   │                              │                       │  processes
   │                              │                       │  at its own
   │                              │                       │  pace
```

### SQS — Point-to-point queue

One producer, one consumer group. Each message is processed by exactly one consumer.

```
                        ┌──────────────────────┐
                        │         SQS          │
Order Svc ─── enqueue ─►│  [msg][msg][msg][msg]│──► Fulfillment Svc
  "order placed"        │                      │    (one worker picks
                        └──────────────────────┘     each message)

Use for: task queues, background jobs, worker pools
Example: "resize this image", "send this email", "process this payment"
```

### SNS — Pub/Sub fan-out

One producer, many subscribers. Every subscriber gets every message.

```
                    ┌─────────┐
                    │   SNS   │
Order Svc ─────────►│  Topic  │──────────────────────────────┐
  "order placed"    └─────────┘                              │
                         │                                   │
              ┌──────────┼──────────┐                        │
              ▼          ▼          ▼                        ▼
         Email Svc  Warehouse   Analytics            ┌──────────────┐
                                                     │  (all three  │
                                                     │  get the msg)│
                                                     └──────────────┘

Use for: broadcasting events to multiple independent consumers
```

### SNS + SQS — Fan-out + Queue (most common pattern)

SNS broadcasts; each subscriber has its own SQS queue for reliability and independent scaling.

```
                    ┌─────────┐
Order Svc ─────────►│   SNS   │
  "order placed"    └─────────┘
                    /    |    \
                   /     |     \
                  ▼      ▼      ▼
              ┌──────┐ ┌──────┐ ┌──────┐
              │ SQS  │ │ SQS  │ │ SQS  │
              └──┬───┘ └──┬───┘ └──┬───┘
                 ▼         ▼        ▼
             Email Svc  Warehouse  Analytics

Why the queues? If Warehouse Svc is down, its SQS queue buffers
messages until it recovers. Without SQS, SNS drops undeliverable messages.
```

### Kinesis — Ordered event stream

High-throughput, ordered within a shard, replayable. Multiple independent consumers read the same stream at their own pace.

```
                    ┌──────────────────────────────────────┐
                    │              Kinesis Stream           │
                    │                                      │
Clickstream ───────►│  Shard 1: [e1]─[e2]─[e3]─[e4]─[e5] │
                    │  Shard 2: [e1]─[e2]─[e3]─[e4]─[e5] │
                    │  Shard 3: [e1]─[e2]─[e3]─[e4]─[e5] │
                    └──────────────────────────────────────┘
                           │              │            │
                           ▼              ▼            ▼
                      Analytics      ML Model      Audit Log
                      (at pos 5)    (at pos 3)    (at pos 5)
                                   (behind, but
                                    catching up)

Key properties:
- Each consumer tracks its own position (no message deleted on read)
- Replay: rewind to any point within 7 days
- Ordered within a shard (not across shards)
```

### Kafka — Ordered stream (self-managed or MSK)

Same model as Kinesis but more control — longer retention, more complex topologies, stronger guarantees.

```
                    ┌──────────────────────────────────────┐
                    │           Kafka Topic: orders         │
                    │                                      │
Order Svc ─────────►│  P0: [o1]─[o3]─[o5]─[o7]            │
                    │  P1: [o2]─[o4]─[o6]─[o8]            │
                    └──────────────────────────────────────┘
                           │                    │
               ┌───────────┘                    └───────────┐
               ▼                                            ▼
    Consumer Group A                            Consumer Group B
    (Fulfillment)                               (Analytics)
    ┌──────┬──────┐                             ┌──────┬──────┐
    │  C1  │  C2  │                             │  C1  │  C2  │
    │  P0  │  P1  │                             │  P0  │  P1  │
    └──────┴──────┘                             └──────┴──────┘
    (each partition consumed                    (independent offset —
     by one consumer in group)                   reads same events)
```

---

## Comparison

| | REST | gRPC | SQS | SNS+SQS | Kinesis | Kafka |
|---|---|---|---|---|---|---|
| Pattern | Sync | Sync | Async queue | Async fan-out | Async stream | Async stream |
| Response | Immediate | Immediate | None | None | None | None |
| Ordering | No | No | No (FIFO opt) | No | Per shard | Per partition |
| Replay | No | No | No | No | Yes (7d) | Yes (configurable) |
| Multiple consumers | No | No | No (one wins) | Yes | Yes | Yes |
| Managed (AWS) | Yes | Yes | Yes | Yes | Yes | MSK only |
| Best for | User-facing reads | Internal high-perf | Task queues | Broadcast | High-throughput streams | Same + more control |

---

## When to use which

```
Does the caller need an answer right now?
        │
        ├── YES → Synchronous
        │           │
        │           ├── External / browser-facing? ──► REST
        │           └── Internal / high-perf?      ──► gRPC
        │
        └── NO → Asynchronous
                    │
                    ├── One consumer processes each message?
                    │       └── YES ──► SQS
                    │
                    ├── Multiple independent consumers?
                    │       ├── Simple broadcast  ──► SNS + SQS
                    │       └── Need ordering / replay ──► Kinesis or Kafka
                    │
                    └── Need replay + long retention + full control?
                                └── Kafka (MSK)
```

---

## Real-world hybrid example

A realistic e-commerce system uses both:

```
                        ┌─────────────────┐
Browser ───────────────►│   API Gateway   │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
        ┌──────────┐      ┌──────────┐       ┌──────────┐
        │ User Svc │      │Order Svc │       │Product   │
        │  (REST)  │      │  (REST)  │       │  Svc     │
        └──────────┘      └────┬─────┘       │  (REST)  │
                               │             └──────────┘
              "order placed" event
                               │
                               ▼
                    ┌─────────────────┐
                    │   SNS Topic     │
                    └────────┬────────┘
                   ┌─────────┼──────────┐
                   ▼         ▼          ▼
               ┌──────┐  ┌──────┐  ┌──────┐
               │ SQS  │  │ SQS  │  │Kinesis│
               └──┬───┘  └──┬───┘  └──┬───┘
                  ▼          ▼         ▼
              Email Svc  Warehouse  Analytics
              (async)    (async)    (stream)

Synchronous: user-facing reads (login, browse, checkout form)
Asynchronous: everything that happens after "order confirmed"
```

## See also

- [[aws-api-gateway]] — entry point before requests reach services
- [[api-gateway-microservices-pattern]] — how services are structured behind the gateway
- [[saga-pattern]] — how to handle distributed transactions over async events
- [[aws-sqs]] — deep dive on queue mechanics
- [[apache-kafka]] — deep dive on Kafka architecture
