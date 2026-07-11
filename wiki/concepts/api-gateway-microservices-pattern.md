---
title: "API Gateway + Microservices Pattern"
category: concept
tags: [api-gateway, microservices, load-balancer, architecture, aws]
sources: []
updated: 2026-07-11
---

# API Gateway + Microservices Pattern

A standard AWS microservices architecture where a single shared [[aws-api-gateway]] sits at the public internet edge, routing to independently-scaled services each managing their own internal load balancing.

## Architecture

### High-level topology

```
┌─────────────────────────────────────────────────────────┐
│                        Internet                         │
└─────────────────────────┬───────────────────────────────┘
                          ↓
              ┌───────────────────────┐
              │      API Gateway      │  api.mycompany.com
              │  (one, shared)        │  auth · rate limit · routing
              └───┬───────────┬───┬───┘
                  │           │   │
         /users   │  /products│   │ /orders
                  ↓           ↓   ↓
┌─────────────────────────────────────────────────────────┐
│  VPC  (private — internet cannot reach this directly)   │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  User Svc   │  │ Product Svc │  │  Order Svc  │     │
│  │    ALB      │  │    ALB      │  │    ALB      │     │
│  │  ┌──┐ ┌──┐ │  │  ┌──┐ ┌──┐ │  │  ┌──┐ ┌──┐ │     │
│  │  │EC│ │EC│ │  │  │EC│ │EC│ │  │  │EC│ │EC│ │     │
│  │  └──┘ └──┘ │  │  └──┘ └──┘ │  │  └──┘ └──┘ │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Request flow (step by step)

```
Browser/App
    │
    │  POST api.mycompany.com/orders
    ↓
┌─────────────────────────────┐
│        API Gateway          │
│  1. Validate JWT token      │
│  2. Check rate limit        │
│  3. Route: /orders → ALB    │──── VPC Link (private tunnel)
└─────────────────────────────┘
                                         ↓
                              ┌─────────────────────┐
                              │   Order Service ALB  │
                              │  4. Pick healthy     │
                              │     instance         │
                              └──────────┬──────────┘
                                         │
                              ┌──────────┴──────────┐
                              ↓                     ↓
                         ┌────────┐           ┌────────┐
                         │ EC2 #1 │           │ EC2 #2 │
                         │(idle)  │           │(picked)│
                         └────────┘           └────────┘
```

### Mixed backend (EC2 + Lambda)

Not every service needs an ALB. Lambda auto-scales, so API Gateway routes directly to it:

```
              API Gateway
        ↓           ↓           ↓
   /users        /products   /notify
      ↓               ↓          ↓
    ALB            ALB         Lambda   ← no fleet, no ALB
   ┌──┬──┐       ┌──┬──┐
   EC  EC         EC  EC
```

### Ownership boundary

```
┌──────────────────────────────────────────────────────┐
│  Platform / Infra Team                               │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │              API Gateway                       │  │
│  │  domain · SSL · auth · routing · rate limits   │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
          ↓                   ↓                   ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User Team   │    │ Product Team │    │  Order Team  │
│              │    │              │    │              │
│  ALB         │    │  ALB         │    │  ALB         │
│  EC2 fleet   │    │  EC2 fleet   │    │  EC2 fleet   │
│  health chks │    │  health chks │    │  health chks │
│  scaling     │    │  scaling     │    │  scaling     │
└──────────────┘    └──────────────┘    └──────────────┘
```

## Why one API Gateway, many ALBs

**API Gateway is a cross-cutting concern.** Auth, rate limiting, and the public domain are shared across the whole system. There's no reason to duplicate them per service — that's exactly the problem API Gateway solves.

**ALB is an internal concern.** Load distribution within a service (which instance handles this request, health checks, rolling deployments) belongs to the service team. Other services don't care and shouldn't need to know.

## The ALBs are private

ALBs live inside the VPC with no public IP. The frontend never knows they exist. API Gateway reaches them via **VPC Link** — a private channel from API Gateway into your VPC. This means:

- External traffic can only enter through API Gateway
- Each service's internal topology is fully encapsulated
- You can change how a service is deployed (swap EC2 for ECS, resize the fleet) without touching API Gateway config

## Ownership split

| Layer | Owned by | Concerns |
|---|---|---|
| API Gateway | Platform / infra team | Public domain, routing table, auth, rate limits |
| ALB + fleet | Each service team | Scaling policy, health checks, target groups, deployments |

This boundary keeps platform concerns out of service teams' way and service internals out of the platform team's way.

## Variations

**Fully serverless** — no ALBs anywhere. All services are Lambda. API Gateway routes directly to functions. Simplest operationally; Lambda cold starts are the main trade-off.

**ALB as public front door** — skip API Gateway entirely. One ALB handles all routing using path-based rules. Cheaper at high throughput, supports gRPC, but you lose managed auth, rate limiting, and caching. Teams must implement those themselves.

**API Gateway + ALB + service mesh** — for large orgs, a service mesh (Envoy/Istio) handles east-west traffic (service-to-service) while API Gateway handles north-south (client-to-service). ALBs sit between them.

## Trade-offs

| | This pattern | ALB-only | Fully serverless |
|---|---|---|---|
| Operational overhead | Low (managed GW) | Medium | Lowest |
| Cost at high volume | GW adds per-request cost | Cheaper | Cheapest for sporadic |
| Auth / rate limiting | Built into GW | DIY | Built into GW |
| gRPC support | No | Yes | No |
| Cold starts | No (EC2/ECS) | No | Yes (Lambda) |

## See also

- [[aws-api-gateway]] — the shared edge layer
- [[application-load-balancer]] — per-service internal load balancing
- [[aws-lambda]] — serverless alternative that removes the ALB layer
- [[vpc-link]] — how API Gateway reaches private ALBs inside the VPC
