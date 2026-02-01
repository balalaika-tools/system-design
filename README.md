# System Design Guide 🏗️

A comprehensive, practical guide to system design concepts — from networking fundamentals to distributed systems patterns. Written for engineers who want to understand how things actually work, without drowning in unnecessary complexity.

---

## 📚 What's Inside

This repository covers the essential building blocks of modern systems:

| Guide | Topics |
|-------|--------|
| [**01. Networking Fundamentals**](systems/01-networking-fundamentals.md) | HTTP, DNS, ports, TLS/SNI, containers, service discovery |
| [**02. Proxies & Load Balancers**](systems/02-proxies-load-balancers.md) | Forward/reverse proxies, load balancers, API gateways, Ingress |
| [**03. Database Scaling**](systems/03-databases-scaling.md) | Replication, sharding, partitioning, distributed databases |
| [**04. Caching Strategies**](systems/04-caching-strategies.md) | Cache patterns, Redis, CDN, invalidation, cache stampede |
| [**05. Message Queues**](systems/05-message-queues.md) | Async messaging, Kafka, RabbitMQ, event-driven architecture |
| [**06. API Design**](systems/06-api-design.md) | REST, GraphQL, gRPC, versioning, pagination |
| [**07. Architectural Patterns**](systems/07-architectural-patterns.md) | Monolith, microservices, event sourcing, CQRS, serverless |
| [**08. Microservices Patterns**](systems/08-microservices-patterns.md) | Circuit breaker, saga, service mesh, strangler fig |
| [**09. Distributed Systems**](systems/09-distributed-systems.md) | CAP theorem, consistency, consensus, replication |
| [**10. Security & Auth**](systems/10-security-authentication.md) | JWT, OAuth 2.0, OIDC, RBAC, API security |

---

## 🎯 Philosophy

This guide is built around a few core principles:

### **Mental Models Over Tools**
Understanding *why* something works is more valuable than memorizing *how* to configure it. Tools change; concepts transfer.

### **Practical, Not Academic**
Every section includes real-world examples, common gotchas, and interview talking points. Theory serves practice.

### **Right Level of Depth**
Deep enough to make good decisions, not so deep you're lost in implementation details. Know when to dive deeper.

### **Trade-offs, Not Silver Bullets**
Every architectural choice has pros and cons. This guide helps you understand *when* to use what, not just *how*.

---

## 🗺️ Learning Path

### For Interview Prep

1. Start with **Networking Fundamentals** — foundation for everything
2. Read **Databases** and **Caching** — most common interview topics
3. Study **Architectural Patterns** — high-level design decisions
4. Review **Distributed Systems** — CAP theorem comes up often
5. Skim **Microservices Patterns** — know the vocabulary

### For Building Systems

1. **API Design** — your interface with the world
2. **Proxies & Load Balancers** — traffic management
3. **Caching** — make things fast
4. **Message Queues** — decouple and scale
5. **Security** — protect your users

### For Deep Understanding

Read everything in order. Each guide builds on concepts from previous ones.

---

## 🧠 Quick Reference

### The Scaling Playbook

```
1. Start with a monolith on a single server
2. Add caching (Redis) for read-heavy workloads
3. Add read replicas for database scaling
4. Add message queues for async processing
5. Extract microservices when team/complexity demands
6. Shard databases when data exceeds single node
7. Add CDN for global distribution
```

### Common Trade-offs

| Choice | Favors | Sacrifices |
|--------|--------|------------|
| Caching | Speed, throughput | Freshness, complexity |
| Microservices | Scale, team autonomy | Simplicity, consistency |
| Async messaging | Decoupling, resilience | Immediate consistency |
| Strong consistency | Correctness | Latency, availability |
| Eventual consistency | Speed, availability | Immediate correctness |

### Numbers to Know

| Operation | Latency |
|-----------|---------|
| L1 cache | 0.5 ns |
| RAM | 100 ns |
| Redis (local) | 0.5 ms |
| SSD read | 0.1 ms |
| Database query | 5-50 ms |
| Network round-trip (same region) | 1-5 ms |
| Network round-trip (cross-region) | 50-150 ms |

---

## 💡 Core Concepts Cheat Sheet

### CAP Theorem
> During a network partition, choose **Consistency** (all nodes see same data) or **Availability** (all requests get a response).

### Horizontal vs Vertical Scaling
> **Vertical**: Bigger machine. **Horizontal**: More machines. Start vertical, go horizontal when needed.

### Stateless vs Stateful
> **Stateless** services (APIs) scale by adding instances. **Stateful** services (databases) scale by partitioning data.

### Sync vs Async
> **Synchronous**: Wait for response. **Asynchronous**: Fire and forget, process later. Use async for decoupling and resilience.

### Push vs Pull
> **Push**: Server sends updates. **Pull**: Client requests updates. Push is efficient but complex; pull is simple but may be stale.

---

## 🔗 Related Resources

### Books
- *Designing Data-Intensive Applications* by Martin Kleppmann
- *System Design Interview* by Alex Xu
- *Building Microservices* by Sam Newman

### Online
- [ByteByteGo](https://bytebytego.com/) — Visual system design
- [High Scalability](http://highscalability.com/) — Real-world architecture case studies
- [Martin Fowler's Blog](https://martinfowler.com/) — Patterns and practices

---

## 📝 Contributing

Found an error? Have a suggestion? Feel free to open an issue or PR.

---

## 📜 License

This content is provided for educational purposes. Use it, share it, learn from it.

---

*"Simplicity is the ultimate sophistication."* — Leonardo da Vinci
