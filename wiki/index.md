# Wiki Index

Last updated: 2026-07-12 | 0 raw sources | 10 pages

## Concepts
- [[api-gateway-microservices-pattern]] — One shared API Gateway at the edge; each microservice owns its own private ALB and fleet
- [[microservice-communication]] — Sync (REST, gRPC) vs async (SQS, SNS, Kinesis, Kafka) — trade-offs, patterns, when to use which
- [[saga-pattern]] — Distributed transactions across microservices via compensating actions; choreography vs orchestration
- [[database-scaling]] — Layered techniques for massive datasets: caching, replicas, partitioning, sharding, CQRS
- [[reverse-proxy]] — Sits in front of backends for SSL, routing, load balancing; foundation that API Gateway builds on
- [[service-to-service-auth]] — mTLS (workload identity, transport layer) + propagated JWT (user identity); how it fits REST/gRPC and the service mesh
- [[oauth-oidc]] — OAuth = delegated authorization (orthogonal to session-vs-JWT); OIDC adds authentication via a JWT ID token; access tokens can be opaque or JWT
- [[cors]] — Browser relaxation of Same-Origin Policy; not a security boundary; credentialed-request rule; same-origin proxy avoids it entirely

## Systems
- [[aws-api-gateway]] — Managed AWS front door: routing, auth, throttling, caching, WebSocket support

## Trade-offs
- [[session-vs-jwt-auth]] — Stateful sessions vs stateless JWT; revocation vs statelessness; per-boundary fit (browser/mobile/service-to-service), BFF pattern

## Case Studies
<!-- How companies solved specific problems -->

## Sources
<!-- Per-article summary pages -->
