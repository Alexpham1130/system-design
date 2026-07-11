# Wiki Log

Append-only chronological record of all wiki operations.

Grep last 10 entries: `grep "^## \[" wiki/log.md | tail -10`

---

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
