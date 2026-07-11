---
title: "Reverse Proxy"
category: concept
tags: [reverse-proxy, nginx, load-balancer, ssl, cdn, api-gateway]
sources: []
updated: 2026-07-11
---

# Reverse Proxy

A server that sits in front of backend servers, forwarding client requests to them. The client only ever sees the proxy — never the backend directly.

---

## Forward proxy vs Reverse proxy

These are often confused. The direction of who is being hidden is opposite.

```
Forward Proxy — hides the CLIENT:

  Client A ──┐
  Client B ──┼──► Forward Proxy ──► Internet
  Client C ──┘
  (server sees proxy's IP, not the client's)
  Used for: corporate firewalls, VPNs, anonymization

Reverse Proxy — hides the SERVER:

  Internet ──► Reverse Proxy ──┬──► Server 1
                               ├──► Server 2
                               └──► Server 3
  (client sees proxy's IP, not the backend's)
  Used for: web servers, APIs, microservices
```

---

## What a reverse proxy does

```
Client                  Reverse Proxy              Backend
  │                          │                        │
  │── HTTPS request ────────►│                        │
  │                          │  SSL termination        │
  │                          │  (decrypt here)         │
  │                          │── HTTP request ────────►│
  │                          │◄─ HTTP response ────────│
  │                          │  (optional: compress,   │
  │                          │   cache, transform)     │
  │◄─ HTTPS response ────────│                        │
```

**Core responsibilities:**

| Function | What it does |
|---|---|
| Request forwarding | Route client requests to the right backend |
| SSL termination | Handle HTTPS at the proxy; backends speak plain HTTP internally |
| Hide backend topology | Clients never see internal IPs or server count |
| Basic load balancing | Distribute requests across multiple backend instances |
| Compression | Gzip responses before sending to client |
| Static caching | Cache backend responses for repeated identical requests |

That's it. No auth, no rate limiting, no API management — those belong to [[aws-api-gateway]].

---

## Reverse proxy in a monolith

The simplest and most common use case. One app, one proxy in front of it.

```
Internet
    ↓
┌─────────────────────────────┐
│       nginx / HAProxy        │
│                             │
│  • SSL termination          │
│  • Serve static files       │
│  • Compress responses       │
│  • Hide server IP           │
└──────────────┬──────────────┘
               │
               ▼
        ┌─────────────┐
        │  Monolith   │
        │  (HTTP)     │
        └─────────────┘
```

---

## Reverse proxy in microservices

Inside a microservices architecture, reverse proxies appear at multiple levels — not just the top edge.

```
Internet
    ↓
API Gateway              ← public edge: auth, rate limiting, routing
    ↓
┌───────────┬────────────┬───────────┐
▼           ▼            ▼           ▼
ALB         ALB          ALB        Lambda
(reverse    (reverse     (reverse   (no proxy
 proxy)      proxy)       proxy)     needed)
↓           ↓            ↓
EC2 fleet   ECS fleet    EC2 fleet

ALBs are managed reverse proxies — they forward and load balance,
nothing more. API Gateway does the API management on top.
```

---

## Reverse proxy vs API Gateway vs CDN

These three are often confused because they overlap in some capabilities.

```
                    Reverse Proxy    API Gateway    CDN
                    ─────────────    ───────────    ───
Request forwarding       ✓               ✓           ✓
SSL termination          ✓               ✓           ✓
Hide backends            ✓               ✓           ✗
Load balancing           ✓               ✓           ✗
Authentication           ✗               ✓           ✗
Rate limiting            ✗               ✓           partial
Request transform        ✗               ✓           ✗
Static asset caching     partial         partial      ✓ (purpose-built)
Global edge nodes        ✗               ✗           ✓
DDoS absorption          ✗               ✗           ✓

Rule: don't use a reverse proxy to cache static images — that's CDN's job.
CDN has global edge nodes; a reverse proxy caches locally on one server.
```

---

## Common reverse proxy software

```
nginx       → most widely used; high performance; can be extended
              with modules to approach API Gateway territory
              (auth, rate limiting via plugins)

HAProxy     → pure load balancing and proxying; extremely fast;
              no API management features

Traefik     → cloud-native; auto-discovers services in Docker/K8s;
              popular in container environments

Envoy       → high-performance; used as the data plane in service
              meshes (Istio); east-west traffic (service-to-service)
              inside a cluster

AWS ALB     → managed reverse proxy on AWS; integrates with ECS,
              EKS, Lambda; supports path-based routing and WebSocket
```

---

## When to use which (verdict)

```
Situation                                    Use
──────────────────────────────────────────────────────────────────
Monolith — just need SSL + hide server      Reverse proxy (nginx)
Static assets / images / media              CDN (CloudFront)
Microservices — internal load balancing     Reverse proxy / ALB
Microservices — public edge                 API Gateway
Exposing API to external developers         API Gateway
Need auth, rate limiting, traffic control   API Gateway
All of the above at scale                   All three, layered
```

**The strongest signal for API Gateway** is not microservices per se — it's **external access control**. You can run 20 microservices behind a plain ALB if they're all internal. The moment you're exposing endpoints to outside developers and need to manage who can call what, API Gateway earns its place.

---

## The full layered picture

```
┌─────────────────────────────────────────────────────┐
│                    Internet                         │
└──────────────────────────┬──────────────────────────┘
                           ↓
               ┌───────────────────────┐
               │  CDN (CloudFront)     │  ← static assets, DDoS
               └───────────┬───────────┘
                           ↓
               ┌───────────────────────┐
               │    API Gateway        │  ← auth, rate limiting, routing
               └───┬───────────┬───┬───┘
                   ↓           ↓   ↓
┌──────────────────────────────────────────────────────┐
│  VPC                                                 │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │  ALB    │   │  ALB    │   │  ALB    │ ← internal  │
│  │(reverse │   │(reverse │   │(reverse │   proxies   │
│  │ proxy)  │   │ proxy)  │   │ proxy)  │             │
│  └────┬────┘   └────┬────┘   └────┬────┘            │
│       ↓             ↓             ↓                  │
│   EC2 fleet     ECS fleet     EC2 fleet              │
└──────────────────────────────────────────────────────┘

Each layer has a single responsibility.
No layer tries to do another layer's job.
```

---

## See also

- [[aws-api-gateway]] — reverse proxy + auth + rate limiting + routing
- [[api-gateway-microservices-pattern]] — where reverse proxies (ALBs) fit in microservices
- [[microservice-communication]] — east-west traffic where Envoy / service mesh proxies appear
