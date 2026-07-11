# Wiki Log

Append-only chronological record of all wiki operations.

Grep last 10 entries: `grep "^## \[" wiki/log.md | tail -10`

---

## [2026-07-11] note | Pending topics for next session
- Sharding vs Partitioning — surface covered in [[database-scaling]], needs dedicated deep-dive page. Key angle: partitioning = same server, transparent; sharding = multiple servers, app must route. Cover range/hash/list strategies, shard key selection pitfalls, cross-shard query problem, rebalancing.
- DB Replication on-prem — AWS (RDS Multi-AZ, Aurora) makes it a checkbox. On-prem: Postgres WAL streaming, MySQL binlog, failover tooling (Patroni, Orchestrator), replication lag, WAL slot bloat, split-brain.
- DB Indexing deep dive — surface in [[database-scaling]]. Needs: B-Tree internals, index selectivity, covering indexes, EXPLAIN ANALYZE, index bloat and VACUUM.

## [2026-07-11] query | Reverse proxy concept and verdict
- Source: conversation
- Pages created: `wiki/concepts/reverse-proxy.md`
- Pages updated: `wiki/index.md`
- Notes: Forward vs reverse proxy distinction, core functions, monolith vs microservices usage, reverse proxy vs API Gateway vs CDN comparison table, common software (nginx/HAProxy/Traefik/Envoy/ALB), when-to-use verdict, full layered architecture diagram.

## [2026-07-11] query | Saga pattern + Database scaling
- Source: conversation
- Pages created: `wiki/concepts/saga-pattern.md`, `wiki/concepts/database-scaling.md`
- Pages updated: `wiki/index.md`
- Notes: Saga covers distributed transaction problem, choreography vs orchestration, 2PC comparison, payment example, idempotency. DB scaling covers full ladder (vertical → cache → connection pool → index → replicas → partitioning → CQRS → sharding → batching) with diagrams for each.

## [2026-07-11] query | Microservice communication patterns
- Source: conversation
- Pages created: `wiki/concepts/microservice-communication.md`
- Pages updated: `wiki/index.md`
- Notes: Covered REST vs gRPC (sync), SQS / SNS+SQS / Kinesis / Kafka (async). Cascading failure diagram, fan-out pattern, decision tree, hybrid e-commerce example. Stubs: [[saga-pattern]], [[aws-sqs]], [[apache-kafka]].

## [2026-07-11] query | API Gateway + Microservices pattern
- Source: conversation
- Pages created: `wiki/concepts/api-gateway-microservices-pattern.md`
- Pages updated: `wiki/index.md`
- Notes: Captured shared-GW / per-service-ALB pattern, ownership split, VPC Link, three variations (serverless, ALB-only, service mesh), comparison table. Stubs: [[vpc-link]].

## [2026-07-11] query | AWS API Gateway — 5 core capabilities
- Source: conversation (no raw file)
- Pages created: `wiki/systems/aws-api-gateway.md`
- Pages updated: `wiki/index.md`
- Notes: Covered unified routing, edge auth, throttling (with DDoS caveat), HTTP vs REST API split, caching gotcha (REST only). Added ALB and self-managed comparison table. Stubs for [[aws-lambda]], [[amazon-cognito]], [[aws-waf]], [[websocket-pattern]], [[application-load-balancer]].

## [2026-07-11] init | Wiki initialized
- Structure created: concepts/, systems/, trade-offs/, case-studies/, sources/
- Schema documented in CLAUDE.md
- Notes: Ready for first ingest.
