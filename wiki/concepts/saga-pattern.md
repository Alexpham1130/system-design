---
title: "Saga Pattern"
category: concept
tags: [saga, distributed-transactions, microservices, eventual-consistency, choreography, orchestration]
sources: []
updated: 2026-07-11
---

# Saga Pattern

A pattern for managing distributed transactions across multiple microservices, where each service has its own database and a traditional ACID transaction is impossible.

---

## The problem: distributed transactions

In a monolith, a database transaction is atomic — everything commits or everything rolls back:

```
BEGIN TRANSACTION
  INSERT order
  DEDUCT account balance
  RESERVE inventory
  SCHEDULE delivery
COMMIT  ← all succeed, or none do
```

In microservices, each service owns its own database. There is no shared transaction:

```
Order Svc       Payment Svc      Inventory Svc    Shipping Svc
    │                │                │                │
    ├─ create order  │                │                │
    │                ├─ charge card   │                │
    │                │                ├─ reserve stock │
    │                │                │                ├─ FAILS ✗
    │                │                │                │
    Now what?
    Order exists, card charged, stock reserved — but no delivery.
    You can't rollback across 4 separate databases.
```

---

## The Saga solution

Break the distributed transaction into a sequence of **local transactions**, each paired with a **compensating action** that undoes it if something later fails.

```
Step 1: Create Order          ↔  compensate: Cancel Order
Step 2: Charge Card           ↔  compensate: Refund Card
Step 3: Reserve Inventory     ↔  compensate: Release Inventory
Step 4: Schedule Delivery     ✗  FAILS

Trigger compensations in reverse:
  Release Inventory → Refund Card → Cancel Order
```

The system reaches a consistent state — not by rolling back a transaction, but by executing explicit undo operations.

---

## Two coordination styles

### Choreography — event-driven, no central coordinator

Each service listens for events and reacts. No single service knows the full flow.

```
Order Svc          Payment Svc       Inventory Svc     Shipping Svc
    │                   │                  │                 │
    ├─ create order      │                  │                 │
    ├─ emit: OrderCreated►                  │                 │
    │                   ├─ charge card      │                 │
    │                   ├─ emit: PaymentDone►                 │
    │                   │                  ├─ reserve stock   │
    │                   │                  ├─ emit: StockReserved►
    │                   │                  │                 ├─ schedule
    │                   │                  │                 ├─ emit: DeliveryScheduled
    │                   │                  │                 │
    ▼                   ▼                  ▼                 ▼
  (listens for      (listens for       (listens for     (listens for
   PaymentFailed     OrderCancelled     PaymentFailed    StockReleased)
   → cancel order)   → refund card)     → release stock)

+ Fully decoupled — services don't know each other
+ No single point of failure
- Hard to trace the full flow across services
- Business logic scattered across many services
- Difficult to add new steps
```

**Failure path (choreography):**

```
                    Shipping Svc FAILS
                          │
                          ├─ emit: DeliveryFailed
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    Inventory Svc    Payment Svc      Order Svc
    release stock    refund card      cancel order
    (reacts to        (reacts to      (reacts to
     DeliveryFailed)   StockReleased)  PaymentRefunded)

Each service reacts to the previous compensation event.
```

---

### Orchestration — central saga orchestrator coordinates the flow

One orchestrator service drives the saga, calling each service in sequence and handling failures.

```
                    ┌──────────────────────┐
                    │   Saga Orchestrator   │
                    │                      │
                    │  1. Create Order  ──────────► Order Svc
                    │     ✓ success         │◄── OK
                    │                      │
                    │  2. Charge Card   ──────────► Payment Svc
                    │     ✓ success         │◄── OK
                    │                      │
                    │  3. Reserve Stock ──────────► Inventory Svc
                    │     ✓ success         │◄── OK
                    │                      │
                    │  4. Schedule      ──────────► Shipping Svc
                    │     ✗ FAILS           │◄── ERROR
                    │                      │
                    │  → run compensations: │
                    │  3. Release Stock ──────────► Inventory Svc
                    │  2. Refund Card   ──────────► Payment Svc
                    │  1. Cancel Order  ──────────► Order Svc
                    └──────────────────────┘

+ Single place to read the entire flow
+ Easy to add steps, change order, handle failures
+ Clear audit trail
- Orchestrator is a coordination bottleneck
- Can become a "god service" if it absorbs business logic
```

---

## Choreography vs Orchestration

| | Choreography | Orchestration |
|---|---|---|
| Coordinator | None — events drive flow | Central orchestrator service |
| Coupling | Low — services only know events | Medium — orchestrator knows all services |
| Traceability | Hard — flow is implicit | Easy — flow is in one place |
| Failure handling | Distributed across services | Centralized in orchestrator |
| Adding new steps | Hard — may require changing many services | Easy — change the orchestrator |
| Best for | Simple flows, few steps | Complex flows, many steps, strict ordering |

---

## Saga vs 2-Phase Commit (2PC)

An alternative approach to distributed transactions is Two-Phase Commit — a protocol that locks all participating services until all agree to commit.

```
2PC:
  Coordinator sends PREPARE → all services lock resources
  All respond READY         → coordinator sends COMMIT
  All commit simultaneously

Problems:
  - All services locked during the protocol (blocking)
  - If coordinator crashes after PREPARE, services stay locked forever
  - Doesn't scale across many services or network partitions
```

```
Saga vs 2PC:

              Saga                        2PC
              ────                        ───
Blocking?     No — each step commits      Yes — all locked until done
              immediately
Failure?      Compensating transactions   Rollback across all participants
Consistency?  Eventual                    Strong (ACID)
Scale?        High                        Low
Complexity?   Compensations needed        Protocol complexity + locks
Use when?     Microservices, high scale   Monolith, low scale, strict ACID
```

---

## Real-world example: Payment saga (from PayPal-like system)

```
Payment Saga Orchestrator
        │
        ├─ 1. Verify balance (Account Svc) ────────────────────────────────┐
        │       ✓ sufficient funds                                          │
        │                                                                   │
        ├─ 2. Lock funds (Account Svc) ─────── lock amount in DB           │
        │       ✓ locked                                                    │
        │                                                                   │
        ├─ 3. Call external payment gateway (Visa/Mastercard) ─────────────┤
        │       ✓ approved                                                  │
        │                                                                   │
        ├─ 4. Record transaction (Transaction Svc) ──────────────────────── │
        │       ✓ recorded                                                  │
        │                                                                   │
        ├─ 5. Deduct balance (Account Svc) ─────── commit deduction         │
        │       ✓ deducted                                                  │
        │                                                                   │
        └─ 6. Notify user (Notification Svc) ───── async, fire and forget   │
                                                                            │
If step 3 (external gateway) FAILS:                                        │
  compensate step 2: unlock funds ◄──────────────────────────────────────┘
  return error to user
```

---

## Key considerations

**Idempotency is mandatory.** Compensating actions may be retried on failure. Every step must be safe to call multiple times with the same result. Use idempotency keys.

**Sagas are eventually consistent, not immediately consistent.** Between steps, the system is in a partially committed state. Design the UX accordingly — show "processing" states.

**Compensations are not always reversible.** "Send email" can't be unsent. Design these as forward-only steps (send a "sorry, payment failed" email instead of trying to unsend).

---

## See also

- [[microservice-communication]] — async event patterns used to implement sagas
- [[api-gateway-microservices-pattern]] — service topology sagas operate within
- [[database-scaling]] — each service in a saga owns its own database
