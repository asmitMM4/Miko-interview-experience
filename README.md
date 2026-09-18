# Miko Interview Experience

This repository documents my interview preparation and experience for a **Lead Software Engineer role at Miko**.

The interview process covered three major areas:

1. **Coding & Problem Solving**
2. **Project & Technical Discussion**
3. **Home Take-Home System Design / Architecture Assignment**

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
* Writing clean and maintainable code

The focus was not only on arriving at the correct solution but also on explaining the approach, analyzing complexity, and discussing edge cases.

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

Typical discussion pattern:

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

The discussion emphasized understanding **why a particular architectural decision was made**, rather than simply knowing which technology was used.

---

# 3. Home Take-Home Assignment

## D1 — Real-Time Notifications

### Requirement

Design and develop a **custom real-time notification framework** for Miko3.

The primary use case is enabling a parent using the **Miko Parent App** to change the language of the child's Miko3 application in real time.

### Objective

The system should allow a language change initiated from the Parent App to reach the Miko3 device immediately and reliably.

The initial requirements include:

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
Language updated
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

The system should:

1. Authenticate the parent.
2. Verify that the parent owns/controls the target Miko3 device.
3. Create a language-change command.
4. Locate the device's active connection.
5. Deliver the command through the real-time channel.
6. Miko3 applies the language change.
7. Miko3 sends an acknowledgement.
8. The backend records the result.
9. The Parent App receives confirmation.

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
        │ Device Registry│  │Connection       │
        │                │  │Manager Cluster  │
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

The design focuses on several distributed-system concepts:

### Persistent Connections

A persistent connection between Miko3 and the backend enables server-initiated communication without relying on polling.

### Device Routing

The backend maintains the relationship:

```text
deviceId → connectionNode
```

This allows a command to be routed directly to the connection-manager instance responsible for that device.

### Idempotency

Commands have a unique `commandId`.

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

If the same command is delivered more than once, the Miko3 client can recognize the previously processed `commandId` and avoid applying the operation multiple times.

### Retry & Failure Handling

The system supports retry mechanisms for:

* Network failures
* Connection failures
* Missing acknowledgements
* Temporary service failures

### Offline Devices

If a device is offline, the requested state can be persisted and applied when the device reconnects.

For state-changing operations such as language selection, retaining the **latest desired state** can prevent unnecessary replay of obsolete commands.

---

# 4. What I Learned

The take-home assignment was particularly useful for thinking about real-time systems beyond simply sending a notification.

The important engineering questions include:

* How do we locate a specific connected device?
* How do we scale persistent connections?
* What happens when a connection-manager node fails?
* How do we prevent duplicate command execution?
* What happens when the device is offline?
* How should retries work?
* How do we maintain low latency?
* How do we observe an end-to-end command?
* How do we maintain security between parent, backend, and device?
* How does the architecture evolve from 5,000 RPM to significantly higher traffic?

The assignment therefore becomes a practical exercise in **distributed systems, real-time communication, scalability, reliability, and system architecture**.
