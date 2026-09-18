# Miko Interview Experience

This repository documents my interview experience for a **Lead Software Engineer role at Miko**, covering coding and problem solving, project and technical discussions, and a take-home system design assignment.

The interview process covered three major areas:

1. **Coding & Problem Solving**
2. **Project & Technical Discussion**
3. **Take-Home System Design / Architecture Assignment**

---

## 1. Coding Questions

The coding round focused on problem-solving ability, data structures, algorithms, and writing production-quality code.

Topics covered included:

* Arrays and Strings
* Hash Maps / Sets
* Two Pointers
* Sliding Window
* Stacks and Queues
* Trees and Graphs
* Recursion
* Dynamic Programming
* Time and Space Complexity
* Clean and maintainable code

The discussion focused not only on arriving at the correct solution, but also on explaining the approach, analyzing time and space complexity, identifying edge cases, and evaluating alternative solutions.

---

## 2. Project-Related Questions

A significant part of the interview focused on previous projects and engineering experience.

Discussion areas included:

* System architecture
* Backend design
* API design
* Database selection and schema design
* Scalability
* Performance optimization
* Distributed systems
* Kubernetes and cloud infrastructure
* High-throughput systems
* Failure handling and resilience
* Observability
* Production debugging
* Engineering trade-offs

A typical discussion flow was:

```text
Problem
   ↓
Requirements
   ↓
Architecture
   ↓
Technology Choices
   ↓
Implementation
   ↓
Scalability
   ↓
Failure Handling
   ↓
Monitoring
   ↓
Trade-offs
```

The discussion emphasized the reasoning behind architectural decisions rather than simply identifying the technologies used.

---

# 3. Take-Home Assignment

## D1 — Real-Time Notifications

### Requirement

Design and develop a **custom real-time notification framework** for Miko3.

The primary use case is enabling a parent using the **Miko Parent App** to change the language of the child's Miko3 application in real time.

### Objective

The system must propagate a language-change request from the Parent App to the target Miko3 device with low latency and reliable delivery.

### Initial Requirements

| Requirement                    |        Target |
| ------------------------------ | ------------: |
| Supported languages            |             8 |
| Throughput                     |     5,000 RPM |
| Backend latency target         |        <10 ms |
| Availability                   |        99.99% |
| Platforms                      | Android + iOS |
| Scalability                    |    Horizontal |
| Communication                  |     Real-time |
| External notification platform |   Not allowed |

---

## User Flow

```text
Parent selects language
        ↓
Miko Parent App
        ↓
Real-Time Notification Framework
        ↓
Miko3 Device
        ↓
Language Updated
        ↓
Acknowledgement
        ↓
Miko Parent App
```

### Example

A parent selects:

```text
French
```

from the Miko Parent App.

The system must:

1. Authenticate the parent.
2. Verify that the parent is authorized to control the target Miko3 device.
3. Create a language-change command.
4. Resolve the device's active connection.
5. Deliver the command through the real-time channel.
6. Apply the language change on Miko3.
7. Send an acknowledgement from Miko3.
8. Record the command result.
9. Return confirmation to the Parent App.

---

## High-Level Architecture

```text
                  ┌───────────────────┐
                  │    Parent App     │
                  │   Android / iOS   │
                  └─────────┬─────────┘
                            │
                           HTTPS
                            │
                            ▼
                  ┌───────────────────┐
                  │    API Gateway    │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ Notification API  │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │ Command / Routing │
                  │      Layer        │
                  └───────┬───────────┘
                          │
                 ┌────────┴────────┐
                 ▼                 ▼
        ┌────────────────┐  ┌─────────────────┐
        │ Device Registry│  │ Connection      │
        │                │  │ Manager Cluster │
        └────────────────┘  └────────┬────────┘
                                     │
                              Persistent
                              Connection
                                     │
                                     ▼
                            ┌────────────────┐
                            │     Miko3      │
                            │ Android / iOS  │
                            └───────┬────────┘
                                    │
                                   ACK
                                    │
                                    ▼
                            Notification API
                                    │
                                    ▼
                               Parent App
```

---

## Important Design Decisions

The architecture addresses the key distributed-system requirements of the assignment.

### Persistent Connections

Miko3 maintains a persistent connection with the backend, enabling server-initiated communication without relying on polling.

This provides a low-latency path for delivering commands to connected devices.

### Device Routing

The backend maintains the relationship:

```text
deviceId → connectionNode
```

This allows a command to be routed directly to the connection-manager instance responsible for the target device rather than broadcasting it across the entire cluster.

### Idempotency

Every command receives a unique `commandId`.

```json
{
  "commandId": "cmd-123",
  "type": "LANGUAGE_CHANGE",
  "deviceId": "miko-001",
  "payload": {
    "language": "fr"
  }
}
```

If the same command is delivered more than once, the Miko3 client can identify the previously processed `commandId` and avoid applying the operation multiple times.

The resulting delivery model is:

```text
At-Least-Once Delivery
        +
Idempotent Processing
        ↓
Reliable Application Semantics
```

### Retry & Failure Handling

The system supports controlled retries for:

* Network failures
* Connection failures
* Missing acknowledgements
* Temporary service failures

Retries should use configurable timeouts, exponential backoff, and jitter to prevent synchronized retry storms.

### Offline Devices

A Miko3 device may not always have an active connection.

When the device is offline, the requested state can be persisted and reconciled when the device reconnects.

For state-oriented operations such as language selection, retaining the **latest desired state** prevents unnecessary replay of obsolete commands.

For example:

```text
French
  ↓
German
  ↓
Spanish
  ↓
French
```

If the device was offline during all four updates, only the latest valid desired state needs to be applied.

---

# 4. Architecture Evaluation

The take-home assignment evaluated the ability to design a production-grade real-time distributed system under explicit scalability, latency, reliability, and security constraints.

The architecture addresses:

* Persistent device connections
* Real-time communication
* Device-to-connection-node routing
* Horizontal scalability of connection managers
* Low-latency command delivery
* At-least-once delivery
* Idempotent command processing
* Retry and failure recovery
* Offline-device handling
* Desired-state synchronization
* Command expiry and stale-state prevention
* Authentication and authorization
* End-to-end observability
* Distributed tracing
* Capacity planning
* Load testing
* High availability
* Fault isolation

## Core Architectural Model

```text
Parent Application
        │
        │ HTTPS
        ▼
   API Gateway
        │
        ▼
 Notification API
        │
        ▼
 Command / Routing Layer
        │
        ├───────────────┐
        ▼               ▼
Device Registry    Connection Manager
                        │
                        │ Persistent Connection
                        ▼
                      Miko3
                        │
                        │ ACK
                        ▼
                 Acknowledgement
```

The architecture separates the **request-processing path** from the **persistent connection layer**, allowing API services and device connections to scale independently.

Unique command IDs, idempotent consumers, retry policies, command expiry, and desired-state synchronization provide reliable behavior in the presence of network failures, duplicate delivery, device disconnections, and service failures.

---

# 5. Engineering Considerations

The design addresses the core questions involved in operating a real-time device communication platform:

* How is a specific connected device located?
* How are persistent connections distributed across multiple nodes?
* How does the system recover when a connection-manager node fails?
* How are duplicate commands handled?
* What happens when a device is offline?
* How are retries controlled?
* How is the low-latency path optimized?
* How is an individual command traced end-to-end?
* How are parent, backend, and device authorization boundaries enforced?
* How does the architecture scale beyond the initial 5,000 RPM requirement?

The assignment therefore spans **distributed systems, real-time communication, scalability, reliability, security, observability, and system architecture**.

---

# 6. Final Outcome

### Interview Verdict

**Selected ✅**

### Offer Status

**Offer received — not accepted**

I successfully cleared the Miko interview process, including the coding rounds, project discussions, and the take-home system design assignment.

The offer was subsequently **not accepted**, and I did not join Miko.

---

## Disclaimer

This document describes the interview experience and a technical interpretation of the take-home system-design requirements.

It does not reproduce any confidential interview material, proprietary Miko implementation details, internal architecture, or non-public information.
