---
title: "Disaster Recovery: Hot, Warm, and Cold Standby"
category: concept
tags: [disaster-recovery, availability, rto, rpo, replication, failover, multi-region]
sources: []
updated: 2026-07-13
---

# Disaster Recovery: Hot, Warm, and Cold Standby

## What it is

Disaster Recovery (DR) is the set of strategies and infrastructure that allow a system to resume operation after a catastrophic failure — region outage, data corruption, accidental deletion, or hardware loss. Every DR strategy is defined by two objectives:

- **RTO** (Recovery Time Objective): how long the system can be down before business impact becomes unacceptable.
- **RPO** (Recovery Point Objective): how much data loss (measured in time) is acceptable.

Lower RTO/RPO requires more pre-provisioned infrastructure and tighter replication — and therefore costs more. The relationship is a step function, not a smooth curve.

## How it works

### Cold Standby

Backups exist (S3, snapshots, tape) but no secondary infrastructure is running.

- **RTO**: 4–24 hours — must provision infra, restore data, reconfigure DNS from scratch.
- **RPO**: 1–24 hours — depends on backup frequency.
- **Cost**: ~1.05× production (storage only).
- **Risk**: The restore process is rarely exercised. Backup corruption or stale snapshots are discovered during the actual disaster, not before.

### Warm Standby

A scaled-down version of production is running continuously, receiving async data replication. It can serve traffic but not at full production load.

- **RTO**: 15 minutes–1 hour — infra already exists; scale it up and flip DNS.
- **RPO**: 1–15 minutes — async replication lag is the gap.
- **Cost**: ~1.3× production (smaller fleet idling).
- **Risk**: The scale-up step is the hidden failure mode — can the secondary actually reach full production capacity under the stress of a real incident?

### Hot Standby (Active-Passive)

A full-capacity secondary stack runs continuously and stays in sync with production. Failover is automated or requires a single operator action.

- **RTO**: Seconds to minutes — health checks + DNS failover (e.g. Route 53) can be fully automated.
- **RPO**: Near-zero — synchronous replication, or very low async lag.
- **Cost**: ~1.9× production (two full stacks).
- **Risk**: Synchronous replication adds write latency and couples the two regions. A slow or lagging replica can throttle the primary's write throughput.

### Active-Active

Both regions serve live traffic simultaneously. "Failover" is just traffic rerouting — the secondary was already warm with real users.

- **RTO**: Effectively zero.
- **RPO**: Zero.
- **Cost**: ~2.5×+ production, plus significant design complexity.
- **Risk**: This is an architecture choice, not just a DR choice. The system must handle split-brain scenarios and distributed write conflicts from day one. See [[saga-pattern]] for one approach to cross-region transaction design.

## Summary table

| Tier | RTO | RPO | Cost multiplier |
|---|---|---|---|
| Cold | 4–24 h | 1–24 h | ~1.05× |
| Warm | 15 min–1 h | 1–15 min | ~1.3× |
| Hot (A-P) | < 5 min | < 1 min | ~1.9× |
| Active-Active | ~0 | ~0 | ~2.5×+ |

## When to use / when not to use

**Cold standby** is appropriate for non-critical systems, internal tooling, or batch workloads where multi-hour downtime is tolerable. The key discipline is *scheduled restore drills* — otherwise the RPO is "undefined."

**Warm standby** is the most common choice for B2B SaaS or internal platforms: meaningful protection without paying for a second full fleet. The scale-up risk must be validated regularly.

**Hot standby** is appropriate for customer-facing systems with SLAs in the range of minutes (e.g., payment processors, e-commerce checkout). Synchronous replication cost on latency must be benchmarked.

**Active-Active** is appropriate for global consumer platforms where regional latency matters as much as availability (e.g., a CDN, a global social platform). Do not adopt this without first designing the data model and conflict-resolution strategy — retrofitting active-active onto a system designed for a single primary is extremely costly.

## Trade-offs

- **Cost vs. recovery speed**: The central axis. Each tier roughly doubles cost to halve RTO/RPO.
- **Replication sync vs. async**: Synchronous replication (zero RPO) adds write latency proportional to cross-region round-trip (~60–120ms for US regions). Async replication introduces a lag window (the RPO gap) but doesn't add latency to the primary write path.
- **Operational complexity vs. automation**: Hot and active-active setups require automated health checks, DNS failover, and runbook-free recovery paths. Simpler tiers are cheaper to run but require humans in the loop — who may not be available at 3 AM.
- **Trust in the backup**: Cold DR is cheap to create and expensive to trust. Hot DR is expensive to run but is continuously exercised — the restore path is validated every day.

## Real-world examples

- **AWS RDS Multi-AZ**: Synchronous standby replica in a different AZ. Automatic failover in ~1–2 minutes. This is hot standby at the AZ level, not region level.
- **Aurora Global Database**: Sub-second replication across regions; promotes a secondary region in < 1 minute. Bridges warm and hot standby.
- **Netflix** (Chaos Engineering): Their active-active architecture across AWS regions is why they intentionally inject failures — they need the active-active paths exercised constantly to trust them.
- **AWS S3**: 11 nines durability via internal multi-AZ replication. Customers still need cross-region replication enabled explicitly for region-loss scenarios.

## See also

- [[database-scaling]] — replication strategies (synchronous vs async, read replicas) that underpin DR
- [[saga-pattern]] — distributed transaction coordination relevant to active-active designs
- [[microservice-communication]] — async messaging patterns that can decouple write paths and reduce replication coupling
