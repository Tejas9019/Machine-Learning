# System Architecture Design

Welcome to the **System Architecture Design** module. This section provides an in-depth, production-focused engineering guide for designing, scaling, and maintaining high-availability, low-latency, and fault-tolerant software architectures.

These guides are written for senior engineers, detailing request paths, component boundaries, trade-offs, deployment architecture, and relevant system design interview questions.

---

## 🗺️ Module Learning Roadmap

System architecture starts at global routing and drills down into microservices coordination, traffic control, security, and caching layers:

```
 [High-Level Architecture] ──> [API Gateway] ──> [Authentication & Rate Limiting]
            │                                                      │
            ▼                                                      ▼
     [Microservices] ───────> [Event-Driven & Message Queues] ──> [Caching Layers]
```

---

## 📂 Topic Breakdown

Click on any topic below to access the deep-dive architectural guide:

| Topic | Primary Focus | Core Engineering Challenges |
| :--- | :--- | :--- |
| 🌍 **[High-Level Architecture](High_Level_Architecture.md)** | Global Request Routing | DNS Geo-routing, CDNs, Load Balancing, Active-Active multi-region replication. |
| 🧩 **[Microservices](Microservices.md)** | Monolith decomposition | Boundaries, service discovery, database-per-service, Sagas, data consistency. |
| ⚡ **[Event-Driven](Event_Driven.md)** | Asynchronous pub-sub flows | Event sourcing, CQRS, exactly-once processing, out-of-order execution. |
| 📨 **[Message Queues](Message_Queues.md)** | Distributed message brokers | Kafka vs. RabbitMQ, partition scaling, consumer rebalancing, backpressure. |
| 🛡️ **[API Gateway](API_Gateway.md)** | System ingress control | SSL termination, request routing, rate limiting, request/response rewriting. |
| 🔑 **[Authentication](Authentication.md)** | User identity & authorization | JWT vs. Sessions, OAuth2/OIDC protocols, stateless auth validation at scale. |
| 🚥 **[Rate Limiting](Rate_Limiting.md)** | Traffic filtering & protection | Token Bucket, Leaking Bucket, sliding window algorithms, Redis cluster counters. |
| 💾 **[Caching](Caching.md)** | Low-latency state access | Redis/Memcached architectures, Cache-Aside/Write-Through patterns, eviction policies. |

---

## 📐 Core Architecture Goals

Every system design must solve the fundamental requirements of:
1. **High Availability**: Aiming for $99.999\%$ uptime by designing out single points of failure (SPOFs) and introducing automated failovers.
2. **Low Latency**: Placing compute and static data close to users via Edge networks (CDNs, regional gateways) and optimizing memory database caches.
3. **Fault Tolerance**: Isolating service failures via circuit breakers, fallbacks, and retry policies to prevent cascading errors across microservices.
4. **Data Consistency**: Choosing the correct consistency model (Strong vs. Eventual) based on CAP Theorem trade-offs.
