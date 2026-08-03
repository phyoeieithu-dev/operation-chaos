# Assessment 1 - High-Level Design for the Score Entry System on AWS

## Table of Contents

1. Architecture Overview
2. Assumptions
3. Architecture Decisions
4. Options Considered
5. Consequences
6. AWS Architecture Diagram

---

# 1. Architecture Overview

This document describes the high-level AWS architecture for the **Operation Chaos Score Entry System**.

The system contains two services:

- **Score Ingest API** – Receives WebSocket connections, publishes score events to Redis Streams, and sends updates to clients.
- **Redis** – Self-managed on EC2 and used as the messaging layer.

## AWS Architecture Diagram

![AWS Architecture](docs/architecture.png)

### Constraints

| Item | Value |
|------|------|
| Baseline Load | ~4,000 connections |
| Peak Load | 50,000+ connections |
| Availability | 99.5% |
| Region | ap-southeast-1 |
| Availability Zones | 2 |
| Managed Database | Not Allowed |

---

# 2. Assumptions

- WebSocket connections are long-lived.
- Score events are small.
- Redis Streams store temporary data.
- Redis replication is used for high availability.
- Auto Scaling is enabled for the API servers.

---

# 3. Architecture Decisions

## Score Ingest API

| Item | Decision |
|------|----------|
| Scaling | Horizontal |
| EC2 | **c7g.large** |
| Load Balancer | Network Load Balancer (NLB) |
| Auto Scaling | Min: 2, Desired: 2, Max: 8 |

**Reason**

- Handles many concurrent WebSocket connections.
- Stateless service, making horizontal scaling simple.
- Auto Scaling increases capacity during peak traffic.

---

## Redis

| Item | Decision |
|------|----------|
| Scaling | Vertical |
| EC2 | **r6g.large** |
| Deployment | Primary + Replica |

**Reason**

- Redis is memory-intensive.
- Vertical scaling provides better performance with lower complexity.
- Replica improves availability across Availability Zones.

---

## Load Balancer

**Selected:** Network Load Balancer (NLB)

### Reason

The Score Ingest API uses WebSocket connections that remain open for a long time and can reach more than 50,000 concurrent connections during peak match periods.

NLB was selected because it:

- Operates at Layer 4 (TCP), making it well-suited for WebSocket traffic.
- Provides lower latency and higher throughput than Layer 7 load balancing.
- Can efficiently handle a large number of long-lived connections.
- Distributes traffic across API instances in multiple Availability Zones.
- Scales automatically as connection volume increases.

### Alternative Considered

**Application Load Balancer (ALB)**

ALB also supports WebSockets and would work for this architecture. However, its Layer 7 features provide limited value because the application only exposes a single WebSocket service and does not require:

- Path-based routing
- Host-based routing
- HTTP header-based routing
- Multiple backend services

### Trade-off

| Option | Advantages | Disadvantages |
|----------|------------|---------------|
| **NLB (Selected)** | Better performance for TCP/WebSocket traffic, lower latency, simpler architecture | Does not provide advanced HTTP routing features |
| **ALB (Rejected)** | Supports path-based and host-based routing, easier integration with HTTP applications | Additional Layer 7 processing that is unnecessary for this workload |
---

# 4. Options Considered

## Score Ingest API

| Option | Result |
|--------|--------|
| Horizontal Scaling | ✅ Selected |
| Vertical Scaling | ❌ Rejected (single point of failure and limited scalability) |

## Redis

| Option | Result |
|--------|--------|
| Vertical Scaling | ✅ Selected |
| Redis Cluster | ❌ Rejected (higher operational complexity) |

---

# 5. Consequences

## Advantages

| Decision | Benefit |
|----------|---------|
| Horizontal scaling for Score Ingest API | Can handle increasing WebSocket connections by adding more EC2 instances. |
| Auto Scaling Group | Automatically scales from **2 to 8 instances** during traffic spikes. |
| Deploy across 2 AZs | Improves availability if one Availability Zone fails. |
| Vertical scaling for Redis | Provides high memory performance with a simple architecture. |
| Redis Primary + Replica | Improves availability and supports failover if the primary becomes unavailable. |

## Disadvantages

| Decision | Drawback |
|----------|----------|
| Horizontal scaling | Requires a Load Balancer and Auto Scaling configuration. |
| Multiple API instances | Clients connected to a failed instance must reconnect. |
| Vertical scaling for Redis | Scaling is limited by the maximum EC2 instance size. |
| Redis Primary + Replica | Writes still go to the primary node, which can become a bottleneck under very high write traffic. |

---

# 6. AWS Architecture Diagram

See the attached **Draw.io architecture diagram**.

### Deployment Summary

| Component | Choice |
|-----------|--------|
| Region | ap-southeast-1 |
| Availability Zones | ap-southeast-1a, ap-southeast-1b |
| Load Balancer | Network Load Balancer |
| API EC2 | c7g.large |
| Redis EC2 | r6g.large |
| Auto Scaling Group | Min: 2, Desired: 2, Max: 8 |
| Redis HA | Primary + Replica |
