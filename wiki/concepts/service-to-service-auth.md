---
title: "Service-to-Service Authentication (mTLS + JWT)"
category: concept
tags: [authentication, mtls, jwt, security, microservices, service-mesh, grpc, rest, zero-trust]
sources: []
updated: 2026-07-12
---

# Service-to-Service Authentication (mTLS + JWT)

## What it is

Internal service-to-service auth answers two **different** questions, with two different tools:

- **mTLS** — *which service* is calling? (**workload identity**, at the transport layer)
- **Propagated JWT** — *which end-user* is this request acting for? (**user identity + claims**, at the application layer)

They are **complementary, not competing**. A typical internal call carries both: mTLS proves `order-service` is really talking to `payment-service`, and inside that encrypted channel a JWT header says "acting on behalf of user 12345 with roles X, Y." See [[session-vs-jwt-auth]] for the JWT/session side.

## How mTLS works

Regular TLS (the "S" in HTTPS) is **one-way**: the client verifies the server's certificate; the server does not verify the client via a cert.

**Mutual TLS (mTLS)** makes it **two-way**: during the TLS handshake **both sides present certificates and both verify each other**. The connection completes only if the client trusts the server's cert *and* the server trusts the client's cert.

- Identity is baked into the certificate (Subject / SAN = "I am `payments-service`"), signed by a **CA** both sides trust.
- It answers *"is the thing on the other end of this socket who it claims to be?"* at the **transport layer** — before any REST/gRPC payload is processed.

| | mTLS | JWT (propagated) |
|---|---|---|
| Question answered | Which *service* is calling? | Which *end-user* is this request for? |
| Layer | Transport (TLS handshake) | Application (a header) |
| Verifies | The caller's machine/workload | The user session that started upstream |

## How it fits REST and gRPC

mTLS sits **underneath both** — REST and gRPC are just *what you send over the connection*; mTLS secures the connection itself, so it's largely protocol-agnostic. Practical differences:

- **REST (HTTP/1.1 or HTTP/2 + JSON)** — mTLS works, but historically each service had to be configured with certs, CA trust, peer verification, and rotation, per service. Tedious at scale.
- **gRPC (HTTP/2 + protobuf)** — **first-class mTLS support** in its standard credentials API (channel credentials). Born in Google's internal microservices world, mutual auth was a design assumption, not a bolt-on — more idiomatic to enable.

## When to use / when not to

- **Use** mTLS for any internal hop where you need to trust the *caller's* identity — i.e. essentially all service-to-service traffic in a zero-trust model.
- **Don't** hand-roll it per service at scale — cert issuance, rotation (often every few hours), revocation, and per-service trust config don't survive dozens of services. Use a mesh (below).
- mTLS does **not** replace user-level auth — pair it with propagated JWTs for end-user identity and authorization.

## The real-world delivery: service mesh + sidecars

Nobody hand-wires mTLS into every service at scale. A **service mesh** (Istio, Linkerd, Consul) handles it **transparently**:

1. A **sidecar proxy** (usually [[reverse-proxy|Envoy]]) sits next to each service.
2. All traffic in/out flows through the sidecar.
3. Sidecars **terminate and originate mTLS between each other** — the app code just makes a plain `http://localhost` REST or gRPC call, unaware.
4. The mesh control plane **issues and rotates certs automatically** (often via SPIFFE/SPIRE identities).

The developer writes ordinary REST/gRPC; the mesh silently upgrades every hop to mutual-authenticated, encrypted mTLS.

## Trade-offs

- **Security vs operational complexity** — mTLS gives strong, cryptographic workload identity and encryption-in-transit, but at the cost of a CA/PKI and cert lifecycle. The mesh trades that for its own operational surface (control plane, sidecar resource overhead, added latency per hop).
- **Transport identity vs user identity** — mTLS alone tells you the *service*, not the *user*. Without propagated JWTs you lose end-user authorization inside the chain.

## Real-world examples

- **Zero-trust networking** — "never trust the network, authenticate every hop." mTLS is the mechanism; the mesh makes it invisible. This is why mTLS only became practical for microservices once meshes existed.
- **Google BeyondProd / SPIFFE** — workload identity issued as short-lived certs, the model most meshes emulate.
- The [[api-gateway-microservices-pattern]] gateway validates the user **once** at the edge, then identity is propagated inward as JWT while mTLS secures each hop.

## See also
- [[session-vs-jwt-auth]] — sessions vs JWT, and where JWT propagation fits
- [[api-gateway-microservices-pattern]] — validate-once-at-edge, propagate-inward
- [[reverse-proxy]] — Envoy as the sidecar proxy that terminates mTLS
