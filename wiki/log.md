# Wiki Log

Append-only chronological record of all wiki operations.

Grep last 10 entries: `grep "^## \[" wiki/log.md | tail -10`

---

## [2026-07-11] query | Session vs JWT authentication
- Source: conversation
- Pages created: `wiki/trade-offs/session-vs-jwt-auth.md`
- Pages updated: `wiki/index.md`
- Notes: Stateful sessions (reference + shared store) vs stateless JWT (self-contained, signature-verified). Core trade-off is revocation/control vs statelessness, not decode-vs-query time. Framed by boundary rather than "microservices vs client-server": HttpOnly cookies for server-rendered browser, JWT for mobile, HttpOnly-cookie/BFF for SPA (localStorage JWT = anti-pattern), JWT-for-identity-propagation + mTLS-for-service-authn internally. BFF pattern and hybrid-is-normal. TODO next: service-to-service with mTLS and how it fits REST/gRPC — likely a dedicated mTLS or service-to-service-auth concept page.

## [2026-07-12] query | Service-to-service auth (mTLS + JWT)
- Source: conversation
- Pages created: `wiki/concepts/service-to-service-auth.md`
- Pages updated: `wiki/index.md`, `wiki/trade-offs/session-vs-jwt-auth.md`
- Notes: mTLS = workload identity at transport layer (two-way cert verification, CA-signed) answering "which service"; propagated JWT = user identity at app layer answering "which user" — complementary, both carried per internal call. mTLS sits under REST and gRPC (gRPC has first-class support via channel credentials). Real-world delivery via service mesh (Istio/Linkerd/Consul) + Envoy sidecars terminating mTLS transparently, control plane auto-rotating SPIFFE/SPIRE certs = zero-trust. Cross-linked to [[session-vs-jwt-auth]] and [[api-gateway-microservices-pattern]].

## [2026-07-12] query | OAuth 2.0 and OIDC
- Source: conversation
- Pages created: `wiki/concepts/oauth-oidc.md`
- Pages updated: `wiki/index.md`, `wiki/trade-offs/session-vs-jwt-auth.md`, `wiki/concepts/service-to-service-auth.md`
- Notes: Corrected "OAuth = session token" misconception. OAuth 2.0 = delegated-authorization framework, orthogonal to the session-vs-JWT axis; defines token roles not formats. Access token can be opaque (introspection = stateful/session-like) OR JWT (stateless). Refresh token = stateful. OIDC = auth layer on top of OAuth; ID token always a JWT. BFF is the clean example that OAuth ≠ browser JWT. Cross-linked into the auth cluster.

## [2026-07-12] query | CORS
- Source: conversation
- Pages created: `wiki/concepts/cors.md`
- Pages updated: `wiki/index.md`
- Notes: Corrected "CORS enables the call" → CORS is a *relaxation* of the Same-Origin Policy (SOP is the wall, CORS the door; browser enforces both). Origin = scheme+host+port, so myapp.com vs api.myapp.com is cross-origin. Covered simple vs preflight (OPTIONS), the credentialed-request rule (Allow-Credentials:true cannot combine with Allow-Origin:*, must echo origin + Vary:Origin), where headers live (app middleware vs edge proxy/gateway). Key framing: CORS is NOT a security boundary (browser-only; curl/Postman ignore it) and NOT a CSRF defense. Architectural alternative: same-origin reverse proxy / gateway at myapp.com/api/* kills CORS and keeps cookies first-party. Cross-linked [[reverse-proxy]], [[api-gateway-microservices-pattern]], [[aws-api-gateway]], [[oauth-oidc]], [[service-to-service-auth]]. Stub: [[csrf]] (next topic).

## [2026-07-12] query | CSRF
- Source: conversation
- Pages created: `wiki/concepts/csrf.md`
- Pages updated: `wiki/index.md`
- Notes: CSRF = ambient-authority write attack; browser attaches cookies by destination regardless of initiator. SOP/CORS do NOT stop it (damage on send, not read) — direct callback to [[cors]] "not a CSRF defense". Vulnerability follows token location: cookie (session OR JWT-in-cookie) = vulnerable; Authorization:Bearer header = immune. Defenses: SameSite (Lax default, Strict, None) with the Lax top-level-GET catch → GET must be safe; synchronizer token (stateful); double-submit cookie (stateless, weaker under subdomain/XSS); custom-header requirement for JSON APIs. Key tension captured: HttpOnly-cookie XSS defense reintroduces CSRF (BFF must add SameSite+token) — XSS and CSRF defenses pull against each other. Cross-linked [[session-vs-jwt-auth]], [[oauth-oidc]], [[service-to-service-auth]], [[reverse-proxy]]. Web-security cluster now: cors + csrf; XSS still a gap.

## [2026-07-12] query | XSS (completes web-security cluster)
- Source: conversation
- Pages created: `wiki/concepts/xss.md`
- Pages updated: `wiki/index.md`, `wiki/concepts/csrf.md`, `wiki/concepts/cors.md` (backlinks)
- Notes: XSS = attacker JS running in your origin; strictly more powerful than [[csrf]] (become the page vs send a request). Key relationship: XSS defeats nearly every CSRF defense by reading the token/double-submit cookie; only HttpOnly cookie resists — which is why the XSS↔CSRF tension is real and both must be defended. Three types: stored (worm-able), reflected (crafted link), DOM-based (never hits server). Defenses in priority: context-aware output encoding (the real fix; framework auto-escape + dangerouslySetInnerHTML/v-html/innerHTML escape hatches), CSP + Trusted Types (backstop), allowlist sanitization/DOMPurify for rich HTML, HttpOnly (blast-radius only), input validation (NOT primary — fix at output). Mental model: CSRF fixed at boundary (SameSite/tokens), XSS fixed at point of output (encode for context). Web-security cluster now complete: [[cors]] + [[csrf]] + [[xss]] mutually cross-linked. Candidate next: synthesis page trade-offs/browser-security-model or xss-vs-csrf tying the three together.

## [2026-07-13] query | Disaster Recovery — hot/warm/cold standby
- Source: conversation
- Pages created: `wiki/concepts/disaster-recovery.md`
- Pages updated: `wiki/index.md`
- Notes: Core DR framework: RTO (downtime tolerance) + RPO (data-loss tolerance) as the two defining axes. Four tiers: cold (backup only, 4–24h RTO, ~1.05× cost), warm (scaled-down fleet + async replication, 15min–1h RTO, ~1.3×), hot active-passive (full fleet + sync replication, <5min RTO, ~1.9×), active-active (~0 RTO/RPO, ~2.5×+ but requires split-brain-aware architecture from day one). Key insight: cold DR is cheap to create and expensive to trust — restore path is rarely exercised. Hot DR is continuously validated by being live. Sync replication adds write latency (cross-region RTT ~60–120ms); async replication is zero write-latency but creates the RPO gap. Cross-linked [[database-scaling]], [[saga-pattern]], [[microservice-communication]].

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
