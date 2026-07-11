---
title: "AWS API Gateway"
category: system
tags: [aws, api-gateway, routing, auth, rate-limiting, websocket, caching]
sources: []
updated: 2026-07-11
---

# AWS API Gateway

A fully managed AWS service that acts as the front door for backend services — routing, securing, throttling, and optionally caching API traffic before it ever reaches compute.

## Primary use case

Expose microservices, Lambda functions, EC2 instances, or other AWS backends through a single, managed entry point without running your own reverse proxy or API management layer.

## Key capabilities

### 1. Unified Entry Point (Routing)

One domain (e.g. `api.mycompany.com`) instead of exposing 20 service URLs to clients. Routes by path:

- `/users` → Lambda function
- `/products` → EC2 instance
- `/checkout` → Step Function workflow

API Gateway also supports **direct AWS service integrations** (e.g. writing to DynamoDB without a Lambda hop), but this pattern is uncommon in practice — it leaks your data model into the API layer and requires VTL mapping templates, which are difficult to maintain.

### 2. Security and Authorization

Handles auth at the edge before requests touch compute. Three native mechanisms:

| Mechanism | Use case |
|---|---|
| **Amazon Cognito** | User pool auth (JWT tokens) |
| **AWS IAM** | Internal AWS-to-AWS calls |
| **Lambda Authorizers** | Custom OAuth/JWT logic, third-party identity providers |

The architectural value is consolidation: auth logic lives in one place rather than being reimplemented inconsistently across every service. This is the canonical "offload cross-cutting concerns to the edge" pattern.

### 3. Rate Limiting and Throttling

Usage plans let you cap requests per second per client. Clients that exceed the limit receive a `429 Too Many Requests` before the request reaches your backend.

**Important distinction**: this protects against accidental overload — a noisy client, a runaway script, a burst of legitimate traffic. It is **not** a DDoS mitigation strategy. For actual DDoS protection, add **AWS WAF** and **AWS Shield** in front of API Gateway.

### 4. Protocol Support

Hosts multiple API types under one roof:

| Type | Description |
|---|---|
| **HTTP APIs** | Newer, ~70% cheaper, lower latency. No request/response transformation, no direct service integrations, no caching. |
| **REST APIs** | Older, full-featured. Supports transformation, integrations, caching, usage plans. |
| **WebSocket APIs** | Persistent bi-directional connections. Gateway manages connection state; backend Lambdas remain stateless. |

gRPC is **not** natively supported.

### 5. Response Caching

API Gateway can cache backend responses at configurable TTLs, avoiding redundant compute and database load for repeated identical requests.

**Gotcha**: caching is only available on **REST APIs**, not HTTP APIs. If you chose HTTP APIs for cost reasons, you cannot add caching later without migrating to REST APIs.

## HTTP API vs REST API — how to choose

| | HTTP API | REST API |
|---|---|---|
| Cost | ~70% cheaper | More expensive |
| Latency | Lower | Higher |
| Request/response transformation | No | Yes (VTL) |
| Direct AWS service integrations | No | Yes |
| Response caching | No | Yes |
| WebSocket | No (separate product) | No (separate product) |
| **Default choice** | Greenfield / most use cases | When you need transformation or caching |

## Strengths

- Zero infrastructure to manage; scales automatically
- Deep AWS ecosystem integration (IAM, Cognito, CloudWatch, X-Ray)
- WebSocket support without maintaining connection state in backend
- Pay-per-request pricing; no idle cost

## Weaknesses

- REST API VTL mapping templates are hard to write and debug
- HTTP APIs lack caching — a meaningful gap for read-heavy APIs
- Vendor lock-in: tightly coupled to AWS
- Not suitable for gRPC or binary protocols
- Cold start latency when paired with Lambda (mitigated by provisioned concurrency)

## Comparison with alternatives

| | API Gateway | [[application-load-balancer]] | Self-managed (Kong, nginx) |
|---|---|---|---|
| Management overhead | None | Low | High |
| Auth built-in | Yes | No | Plugin/custom |
| Rate limiting | Yes | No | Plugin/custom |
| WebSocket | Yes | Yes | Yes |
| gRPC | No | Yes | Yes |
| Cost at high volume | Can get expensive | Cheaper | Infrastructure cost |
| Vendor lock-in | High | High | Low |

Use API Gateway when you're already in AWS and want zero operational overhead. Use ALB when you need gRPC or want simpler routing at lower cost. Use self-managed when you need portability or features neither AWS option provides.

## See also

- [[aws-lambda]] — primary compute backend for serverless API patterns
- [[amazon-cognito]] — user pool auth integration
- [[aws-waf]] — DDoS and bot protection to layer on top
- [[websocket-pattern]] — persistent connection architecture
