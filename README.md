# D1 — Real-Time Notifications Framework

## Real-Time Language Change for Miko3

**Product:** Miko3
**System:** Custom Real-Time Notification Framework
**Primary Use Case:** Parent-controlled language switching
**Target Load:** 5,000 RPM
**Target Latency:** <10 ms
**Availability Target:** 99.99%

---

## 1. Overview

This project designs and implements a **custom-built real-time notification framework** that enables parents to remotely change the language of a child's Miko3 device through the Miko Parent application.

Miko3 is a multilingual product supporting eight languages, with English as the default language. Parents can manage different aspects of the child's Miko3 experience through the Miko Parent app, including the device language.

The objective of this system is to propagate a language-change command from the **Miko Parent App** to the **Miko3 App** in real time, apply the change immediately on the child device, and return an acknowledgement to the parent application.

The framework is designed with a focus on:

* Low-latency communication
* Horizontal scalability
* Reliable message delivery
* Device connectivity
* Fault tolerance
* Secure communication
* Idempotent command processing
* Real-time acknowledgement
* 99.99% availability

> **Design constraint:** The notification infrastructure is built from scratch without relying on external push-notification platforms.

---

# 2. Problem Statement

A parent changes the language of the child's Miko3 device from the Miko Parent application.

For example:

```text
Parent App
    |
    | Change Language → French
    |
    v
Real-Time Notification Framework
    |
    | Real-Time Command
    |
    v
Miko3 Device
    |
    | Apply French
    |
    v
Acknowledgement
    |
    v
Parent App
```

The system must ensure that the command reaches the correct Miko3 device with minimal latency while maintaining reliability and security.

---

# 3. Requirements

## Functional Requirements

### 3.1 Parent-Side Language Change

The Miko Parent application should allow a parent to select one of the supported languages for the child's Miko3 device.

Example:

```json
{
  "deviceId": "miko-device-123",
  "language": "fr"
}
```

Once the parent confirms the change, the Parent App sends the request to the backend.

---

### 3.2 Real-Time Notification

The backend should deliver the language-change command to the corresponding Miko3 device through a persistent real-time connection.

The system should avoid polling wherever possible.

```text
Parent App
    |
    | HTTPS
    v
API Gateway
    |
    v
Notification Service
    |
    v
Connection Manager
    |
    | Persistent Connection
    v
Miko3
```

---

### 3.3 Child-Side Language Update

The Miko3 application receives the command and immediately updates its language configuration.

The language change should:

* Happen without restarting the application
* Avoid disrupting the child's current experience
* Persist the new language locally
* Return an acknowledgement to the backend

Example acknowledgement:

```json
{
  "commandId": "cmd-8f71a",
  "deviceId": "miko-device-123",
  "status": "APPLIED",
  "language": "fr"
}
```

---

### 3.4 Delivery Acknowledgement

The backend should track whether the command was successfully delivered and processed.

Possible states:

```text
CREATED
   ↓
SENT
   ↓
DELIVERED
   ↓
APPLIED
```

Failure states:

```text
FAILED
TIMEOUT
EXPIRED
```

---

# 4. Non-Functional Requirements

| Requirement             |           Target |
| ----------------------- | ---------------: |
| Throughput              |        5,000 RPM |
| Average request latency |           <10 ms |
| Availability            |           99.99% |
| Scalability             |       Horizontal |
| Platforms               |    Android + iOS |
| Supported languages     |                8 |
| Communication           |        Encrypted |
| Data at rest            |        Encrypted |
| Delivery                |         Reliable |
| Failure handling        | Retry + fallback |

> **Latency clarification:** The <10 ms target is treated as the backend notification-path latency target. End-to-end device update latency will also depend on network conditions, device connectivity, and client-side processing.

---

# 5. High-Level Architecture

```text
                         ┌──────────────────┐
                         │   Parent App     │
                         │ Android / iOS    │
                         └────────┬─────────┘
                                  │
                                  │ HTTPS
                                  ▼
                         ┌──────────────────┐
                         │   API Gateway    │
                         └────────┬─────────┘
                                  │
                                  ▼
                     ┌─────────────────────────┐
                     │ Notification API        │
                     │ / Language Change       │
                     └────────────┬────────────┘
                                  │
                                  ▼
                     ┌─────────────────────────┐
                     │ Command / Routing Layer │
                     └────────────┬────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │                         │
                     ▼                         ▼
              ┌─────────────┐          ┌──────────────┐
              │ Device      │          │ Persistent   │
              │ Registry    │          │ Connection   │
              │             │          │ Manager      │
              └─────────────┘          └──────┬───────┘
                                              │
                                    WebSocket / TCP
                                              │
                                              ▼
                                     ┌────────────────┐
                                     │    Miko3       │
                                     │ Android / iOS  │
                                     └───────┬────────┘
                                             │
                                             │ ACK
                                             ▼
                                    ┌──────────────────┐
                                    │ Acknowledgement  │
                                    │ Processing       │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                      Parent App
```

---

# 6. Core Components

## 6.1 API Gateway

The API Gateway provides the entry point for requests from the Miko Parent application.

Responsibilities:

* Authentication
* Authorization
* Request validation
* Rate limiting
* Request routing
* TLS termination
* Observability

Example API:

```http
POST /v1/devices/{deviceId}/language
```

Request:

```json
{
  "language": "fr"
}
```

Response:

```json
{
  "commandId": "cmd-8f71a",
  "status": "ACCEPTED"
}
```

---

## 6.2 Notification Service

The Notification Service is responsible for creating and managing real-time commands.

Responsibilities:

* Generate command IDs
* Validate language changes
* Resolve the target device
* Route commands
* Track command state
* Handle acknowledgements
* Trigger retries

---

## 6.3 Device Registry

The Device Registry maintains the relationship between users, children, and Miko3 devices.

Example:

```text
Parent
  |
  └── Child
        |
        └── Miko3 Device
```

Example device record:

```json
{
  "deviceId": "miko-device-123",
  "userId": "user-456",
  "connectionNode": "node-17",
  "status": "ONLINE",
  "lastSeen": "2026-09-18T12:00:00Z"
}
```

The `connectionNode` allows the system to determine which connection-manager instance currently owns the device connection.

---

# 7. Persistent Device Connection

The Miko3 device maintains a persistent connection with the notification infrastructure.

Conceptually:

```text
Miko3
   │
   │ Persistent Connection
   ▼
Connection Manager
```

A persistent connection avoids the overhead of repeatedly establishing connections and enables the server to push commands immediately.

Each connected device is associated with a connection-manager node.

```text
                 Connection Manager Cluster

        ┌────────────┐
        │   Node 1   │── Device A
        └────────────┘

        ┌────────────┐
        │   Node 2   │── Device B
        └────────────┘

        ┌────────────┐
        │   Node 3   │── Device C
        └────────────┘
```

---

# 8. Device Routing

A critical part of the design is determining where a particular Miko3 device is connected.

A distributed device registry can maintain:

```text
deviceId → connectionNode
```

Example:

```text
miko-001 → node-1
miko-002 → node-3
miko-003 → node-2
```

When a language-change request arrives:

```text
deviceId
   ↓
Device Registry
   ↓
Connection Node
   ↓
Persistent Connection
   ↓
Miko3
```

This avoids broadcasting a command to every connection-manager instance.

---

# 9. Command Model

Every real-time operation receives a unique `commandId`.

Example:

```json
{
  "commandId": "01J9ABCXYZ",
  "type": "LANGUAGE_CHANGE",
  "deviceId": "miko-device-123",
  "language": "fr",
  "timestamp": 1726660800000,
  "expiresAt": 1726660860000
}
```

The command ID provides:

* Idempotency
* Traceability
* Retry tracking
* Observability
* Duplicate detection

---

# 10. Delivery Semantics

The system should provide **at-least-once command delivery** with idempotent processing.

Why?

Network failures can occur after the server sends a command but before it receives the acknowledgement.

Therefore, blindly using exactly-once delivery at the infrastructure level is difficult and expensive.

Instead:

```text
At-Least-Once Delivery
          +
Idempotent Consumer
          =
Effectively Once Application
```

The Miko3 client maintains recently processed command IDs.

Example:

```text
commandId = cmd-123

First delivery:
    APPLY language = French
    ACK cmd-123

Duplicate delivery:
    commandId already processed
    Do not apply again
    ACK cmd-123
```

---

# 11. Reliability and Retry

If the Miko3 device does not acknowledge the command within the configured timeout, the notification service can retry.

Example:

```text
Attempt 1
   ↓
Timeout
   ↓
Attempt 2
   ↓
Timeout
   ↓
Attempt 3
   ↓
Failure / Offline Queue
```

Retries should use exponential backoff with jitter.

Example:

```text
100 ms
250 ms
500 ms
1 sec
```

The exact retry policy should be configurable.

---

# 12. Offline Device Handling

A Miko3 device may not always have an active connection.

Possible states:

```text
ONLINE
OFFLINE
CONNECTING
UNKNOWN
```

If the device is offline:

```text
Parent App
    ↓
Language Change
    ↓
Backend
    ↓
Device OFFLINE
    ↓
Persist Command
    ↓
Miko3 reconnects
    ↓
Deliver Latest Valid Command
```

For language changes, the system can optimize storage by retaining only the latest unexpired language command for a device.

For example:

```text
French
German
Spanish
French
```

If the device was offline throughout this period, the system does not necessarily need to replay all four commands. The desired state is:

```text
French
```

This is an example of **state-oriented command compaction**.

---

# 13. Database Design

The system requires low-latency access to device connection state and command metadata.

A distributed key-value store can be used for the hot path.

Example logical schema:

### Device Registry

```text
Key:
device:{deviceId}

Value:
{
    userId,
    connectionNode,
    status,
    lastSeen
}
```

### Command State

```text
Key:
command:{commandId}

Value:
{
    deviceId,
    type,
    payload,
    status,
    createdAt,
    expiresAt
}
```

The database should not become a bottleneck in the real-time path.

The architecture should distinguish between:

```text
Hot Path
    ↓
Connection / Routing / Command Delivery

Cold Path
    ↓
Analytics / Audit / Historical Data
```

---

# 14. Security

All communication between components should use encrypted transport.

```text
Parent App
    │
    │ TLS
    ▼
API Gateway
    │
    │ mTLS / TLS
    ▼
Backend Services
    │
    │ TLS
    ▼
Miko3
```

Security controls include:

* TLS for data in transit
* Encryption at rest
* Device authentication
* Parent authentication
* Authorization checks
* Device ownership validation
* Command validation
* Replay protection
* Command expiration
* Rate limiting
* Audit logging

A language-change request must verify that the authenticated parent is authorized to modify the target Miko3 device.

---

# 15. Failure Scenarios

## Scenario 1 — Connection Manager Failure

```text
Miko3
  ↓
Node A

Node A fails
  ↓
Miko3 reconnects
  ↓
Node B
```

The device registry should update the device's connection location after reconnection.

---

## Scenario 2 — Network Failure

```text
Backend
   ↓
Send command
   ↓
Network failure
   ↓
No ACK
   ↓
Retry
```

---

## Scenario 3 — Duplicate Command

```text
Command cmd-123
      ↓
Delivered
      ↓
ACK lost
      ↓
Retry cmd-123
      ↓
Miko3 detects duplicate
      ↓
ACK again
```

The language should not be applied twice.

---

## Scenario 4 — Device Offline

```text
Parent
  ↓
Language Change
  ↓
Device Offline
  ↓
Persist desired state
  ↓
Device reconnects
  ↓
Apply latest valid state
```

---

# 16. Scalability

The target load is:

```text
5,000 RPM
```

Equivalent average request rate:

```text
5,000 / 60 ≈ 83 requests/sec
```

Although the average rate is relatively modest, the architecture should support significant bursts.

The services should therefore be horizontally scalable:

```text
                    Load Balancer
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       API-1           API-2           API-3
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                 Notification Layer
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Node-1           Node-2          Node-3
          │              │               │
       Devices         Devices         Devices
```

Stateless API services can be scaled independently from connection-manager nodes.

---

# 17. Latency Considerations

The <10 ms requirement requires careful control of the synchronous request path.

The target path is:

```text
Parent App
    ↓
API Gateway
    ↓
Notification Service
    ↓
Device Routing
    ↓
Connection Manager
    ↓
Miko3
```

The design should minimize:

* Network hops
* Synchronous database calls
* Serialization overhead
* Lock contention
* Cross-region communication
* Unnecessary service-to-service calls

Hot connection state should be kept close to the connection-management layer.

For a globally distributed deployment, region-aware routing should be considered so that devices connect to a nearby region.

---

# 18. Observability

The system should provide end-to-end visibility using a correlation ID.

Example:

```text
requestId
    ↓
commandId
    ↓
deviceId
    ↓
connectionNode
    ↓
delivery
    ↓
acknowledgement
```

Important metrics include:

### API Metrics

```text
request_rate
request_latency
error_rate
p95_latency
p99_latency
```

### Notification Metrics

```text
commands_created
commands_delivered
commands_failed
commands_retried
acknowledgement_latency
offline_devices
```

### Connection Metrics

```text
active_connections
connections_created
connections_closed
reconnections
connection_errors
```

### Infrastructure Metrics

```text
CPU
Memory
Network
Database latency
Connection-manager utilization
```

---

# 19. Distributed Tracing

A single language-change operation should be traceable across the complete system.

```text
Parent App
    │
    └── requestId
          │
          ▼
      API Gateway
          │
          ▼
 Notification Service
          │
          ├── commandId
          │
          ▼
    Device Registry
          │
          ▼
 Connection Manager
          │
          ▼
        Miko3
          │
          ▼
         ACK
```

This makes it possible to identify whether latency was introduced by the API, routing layer, connection manager, network, or device.

---

# 20. API Design

## Change Language

```http
POST /v1/devices/{deviceId}/language
```

Request:

```json
{
  "language": "fr"
}
```

Response:

```json
{
  "commandId": "cmd-123",
  "status": "ACCEPTED"
}
```

---

## Device Connection

```http
POST /v1/devices/connect
```

Example:

```json
{
  "deviceId": "miko-device-123",
  "deviceToken": "..."
}
```

The actual real-time communication channel is maintained separately.

---

## Command Acknowledgement

```http
POST /v1/commands/{commandId}/ack
```

Request:

```json
{
  "deviceId": "miko-device-123",
  "status": "APPLIED",
  "language": "fr"
}
```

---

# 21. End-to-End Flow

```text
1. Parent selects French
            │
            ▼
2. Parent App sends HTTPS request
            │
            ▼
3. API validates authentication/authorization
            │
            ▼
4. Notification Service creates commandId
            │
            ▼
5. Device Registry resolves connection node
            │
            ▼
6. Command sent through persistent connection
            │
            ▼
7. Miko3 receives command
            │
            ▼
8. Miko3 changes language to French
            │
            ▼
9. Miko3 sends acknowledgement
            │
            ▼
10. Backend records command as APPLIED
            │
            ▼
11. Parent App receives confirmation
```

---

# 22. Design Principles

The architecture follows several distributed-systems principles:

### Stateless Services

API and business services remain stateless wherever possible, allowing horizontal scaling.

### Stateful Edge

Persistent device connections are managed by dedicated connection-manager nodes.

### Idempotency

Commands can safely be retried without producing incorrect state.

### Failure Isolation

Failure of one connection-manager node should not bring down the complete notification system.

### Backpressure

The system should protect downstream components from sudden traffic spikes.

### Time-Bounded Commands

Language-change commands should have an expiration time to prevent stale updates.

### Observability First

Every command should be traceable from parent request to device acknowledgement.

---

# 23. Capacity Planning

The initial requirement is:

```text
5,000 RPM
≈ 83 requests/sec
```

The production design should not be sized only for the average rate.

Capacity planning should consider:

```text
Peak traffic
Traffic bursts
Number of connected devices
Average connection lifetime
Messages/device
Retry rate
Offline-device percentage
Database operations
Network bandwidth
```

A production deployment should be load-tested at multiple levels, for example:

```text
1x   → 5,000 RPM
2x   → 10,000 RPM
5x   → 25,000 RPM
10x  → 50,000 RPM
```

The purpose of the additional load levels is to validate the scaling behavior and identify the eventual bottleneck rather than assuming the initial requirement is the system's maximum capacity.

---

# 24. Testing Strategy

## Functional Testing

* Language change
* Device acknowledgement
* Invalid language
* Unauthorized device access
* Duplicate commands
* Expired commands

## Load Testing

Measure:

```text
5,000 RPM
10,000 RPM
25,000 RPM
50,000 RPM
```

Metrics:

```text
P50
P95
P99
Error rate
Throughput
CPU
Memory
Network
Database latency
```

## Failure Testing

Test:

* Connection manager crash
* Database failure
* Network partition
* Device disconnect
* Duplicate delivery
* Delayed acknowledgement
* Service restart
* Traffic spikes

---

# 25. Key Architectural Trade-offs

## WebSocket vs Polling

**Polling**

```text
Device → periodically ask → Server
```

Creates unnecessary requests and increases notification latency.

**Persistent Connection**

```text
Device ←────────→ Server
```

Allows the server to push updates immediately.

---

## At-Least-Once vs Exactly-Once

Exactly-once delivery across distributed systems is difficult to guarantee end-to-end.

Therefore, this design uses:

```text
At-Least-Once Delivery
+
Idempotent Command Processing
```

This provides reliable application-level semantics while keeping the infrastructure simpler.

---

## Synchronous vs Asynchronous Processing

The parent request should remain lightweight.

```text
Parent
  ↓
Validate
  ↓
Create Command
  ↓
Route
  ↓
Return ACCEPTED
```

Device processing and acknowledgement can happen asynchronously.

This prevents slow device/network conditions from unnecessarily blocking the Parent App request.

---

# 26. Future Extensions

The framework can be generalized beyond language changes.

For example:

```text
LANGUAGE_CHANGE
VOLUME_CHANGE
PARENTAL_CONTROL_UPDATE
CONTENT_RESTRICTION
DEVICE_CONFIGURATION
REMOTE_COMMAND
FIRMWARE_CONFIGURATION
```

A generic command envelope can therefore be used:

```json
{
  "commandId": "cmd-123",
  "type": "LANGUAGE_CHANGE",
  "deviceId": "miko-device-123",
  "payload": {
    "language": "fr"
  },
  "createdAt": 1726660800000,
  "expiresAt": 1726660860000
}
```

This allows the notification framework to evolve into a general **device command and real-time communication platform**.

---

# 27. Repository Structure

```text
real-time-notification-framework/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── api.md
│   ├── sequence-diagrams.md
│   └── scalability.md
│
├── services/
│   ├── api-gateway/
│   ├── notification-service/
│   ├── connection-manager/
│   ├── device-registry/
│   └── acknowledgement-service/
│
├── clients/
│   └── miko3/
│
├── infrastructure/
│   ├── docker/
│   ├── kubernetes/
│   └── terraform/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── load/
│   └── failure/
│
└── observability/
    ├── dashboards/
    └── alerts/
```

---

# 28. What This Project Demonstrates

This system demonstrates practical understanding of:

* Distributed systems
* Real-time communication
* Persistent connections
* Device routing
* Horizontal scalability
* Low-latency system design
* Idempotency
* Retry strategies
* Fault tolerance
* Distributed state
* API design
* Security
* Observability
* Capacity planning
* Load testing
* Failure handling

---

# 29. Summary

The proposed architecture provides a scalable foundation for real-time communication between the Miko Parent application and Miko3 devices.

The key architectural approach is to separate the system into:

```text
API Layer
     ↓
Command Layer
     ↓
Device Routing
     ↓
Connection Management
     ↓
Persistent Device Connection
     ↓
Acknowledgement
```

The framework is designed around **horizontal scalability, low-latency delivery, reliable at-least-once semantics, idempotent processing, secure communication, and failure recovery**.

Although the initial use case is language switching, the architecture is intentionally generalized so that the same infrastructure can support additional real-time device commands in the future.
