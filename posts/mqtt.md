# Using MQTT and EMQX for Reliable Desktop Agent Communication

In one of our systems we needed a reliable way for backend services to deliver events and commands to a desktop agent installed on user machines.

The desktop agent consists of two components:

- A **Windows Service** responsible for background processing
- A **WPF application** that provides the user interface

Most of the system logic runs in the Windows Service, which needs to receive notifications and instructions from the backend.

Initially we relied on **Windows Push Notification Services (WNS)**. However, several practical limitations eventually led us to adopt **MQTT with EMQX as the messaging layer**.

This article describes the architecture we implemented, the problems we faced, and the lessons learned along the way.

---

# Initial Approach: Windows Push Notifications

Our original system used **Windows Push Notification Services (WNS)** to deliver messages to the desktop application.

While WNS works well for certain application scenarios, it introduced several limitations for our architecture.

## Packaging Constraints

WNS requires applications to be packaged using **MSIX**.

Our desktop agent was distributed using a traditional installer. Introducing MSIX created several challenges:

- MSIX packaging significantly increased the **size of the installation package**
- Additional runtime dependencies had to be bundled
- Deployment became more complicated in enterprise environments
- Our existing installation and update mechanisms would have needed major changes

Adopting MSIX purely to support WNS added unnecessary complexity.

## UI Dependency

Another limitation was that **only the UI application can receive push notifications**.

The Windows Service cannot directly receive WNS messages. This forced the system to depend on the UI layer being active even though most of the processing happens in the background service.

## Session 0 Limitations

The Windows Service runs in **Session 0**, while WNS notifications are delivered to the user session.

This made it difficult to reliably route notifications to the background service.

## Operational Issues

We also experienced several operational problems:

- Required Windows services were sometimes missing or disabled on user machines
- Notification delivery was inconsistent and difficult to troubleshoot
- Dependencies on Azure infrastructure that were outside our operational control
- Limited visibility into delivery failures

Because of these issues, the system became difficult to operate reliably at scale.

---

# Requirements for a Better Solution

When redesigning the notification system, we needed a solution that:

- Works directly with the **Windows Service**
- Supports **targeted device communication**
- Provides **reliable delivery**
- Avoids backend state management
- Integrates with existing authentication infrastructure
- Scales easily with many connected devices

After evaluating several approaches, we chose **MQTT**.

---

# Why Not HTTP Polling

Polling from the desktop agent was the simplest option but had several drawbacks:

- Constant network requests even when no events exist
- Increased backend load
- Higher latency between event generation and delivery
- Backend storage required for pending messages

As the number of devices grows, polling becomes inefficient.

---

# Why Not WebSockets

WebSockets provide real-time communication but using them through **AWS API Gateway** introduced additional complexity:

- Managing large numbers of persistent connections
- Handling reconnection logic
- Backend infrastructure required to track connection IDs
- Message routing logic required in backend services

This added unnecessary operational overhead.

---

# Why MQTT

MQTT is a lightweight messaging protocol designed for systems with large numbers of clients and intermittent connectivity.

It provides several advantages:

- Lightweight protocol designed for persistent connections
- Built-in **publish–subscribe messaging model**
- Topic-based routing handled by the broker
- Reliable delivery using **QoS levels**
- Support for **offline message buffering**

Using MQTT allowed backend services to remain **stateless**, while delegating routing and delivery responsibilities to the broker.

---

# High-Level Architecture

The system consists of:

- Desktop Agent (Windows Service + WPF UI)
- MQTT Broker (EMQX)
- Backend Services
- Authentication through Cognito
- API Gateway for authentication validation

Each device subscribes to a deterministic topic pattern:

```
devices/{deviceId}
```

---

# Device Connection Flow

When the desktop agent starts, the Windows Service connects to the MQTT broker.

The connection flow works as follows:

1. The Windows Service attempts to connect to EMQX.
2. The client includes its **existing access token**.
3. EMQX receives the connection request.
4. EMQX performs **HTTP authentication** by calling an API Gateway endpoint.
5. The API Gateway validates the token using **Cognito**.
6. If authentication succeeds, EMQX accepts the connection.
7. The client subscribes to its device topic.

Once subscribed, the desktop agent can receive messages from the backend.

---

## Implementation Details

The desktop application is implemented in **C#.NET** and runs primarily inside a **Windows Service**.  
The service establishes and maintains a persistent MQTT connection with the broker using a .NET MQTT client library.

On the backend side, publishers are implemented in **Java** using the **HiveMQ MQTT client library**. Backend services publish messages directly to deterministic device topics without maintaining any state about connected devices.

Because routing and delivery guarantees are handled by the MQTT broker, backend services remain stateless and horizontally scalable.

---


# Stateless Publisher Design

Backend services publish messages directly to the topic:

```
devices/{deviceId}
```

The publisher does **not maintain any state about connected clients**.

If the device is connected, the message is delivered immediately.  
If the device is offline, the broker stores the message until the device reconnects.

Advantages:

- No subscriber tracking in backend services
- Simple publishing logic
- Horizontal scalability
- Loose coupling between services and clients

---

# Message Delivery Model

The system uses:

- **QoS 1 (at-least-once delivery)**
- **Message TTL of 60 days**

QoS 1 ensures messages are delivered reliably.

If a device is temporarily offline, EMQX stores the message and delivers it when the client reconnects.

Because QoS 1 provides *at-least-once delivery*, duplicate messages are possible. The client processes messages in an **idempotent manner**.

---

# Connection Resilience

When large numbers of clients reconnect simultaneously (for example after a broker restart), connection spikes can occur.

To prevent this, the desktop agent uses **exponential backoff with random jitter**.

Example retry pattern:

```
Retry 1 → 1s
Retry 2 → 2s
Retry 3 → 4s
Retry 4 → 8s
Retry 5 → 16s
```

Random jitter spreads reconnection attempts across time and prevents a **thundering herd problem**.

---

# Broker Deployment

The MQTT broker runs using **EMQX deployed in Kubernetes**.

EMQX was chosen because:

- Multiple authentication options
- Support for HTTP-based authentication
- High performance for large numbers of connections
- Easy deployment in containerized environments

Since our backend already runs in Kubernetes, integrating EMQX was straightforward.

---

# Leveraging a Pluggable Notification Architecture

Our platform already supported **pluggable notification channels**.

Previously, the system used Windows Push Notifications.

Because of this modular design, introducing MQTT required minimal changes. We implemented a new MQTT notification provider and replaced the previous mechanism without breaking other parts of the system.

This demonstrated the value of designing systems with **clear abstraction boundaries**.

---

# Lessons Learned

Some key takeaways from implementing this architecture:

- MQTT works extremely well for device communication scenarios
- Topic naming should remain simple and deterministic
- Broker-side authentication simplifies identity integration
- Reconnection strategies are essential for stability
- Designing systems with pluggable notification channels enables easier evolution

---

# Final Thoughts

Replacing Windows Push Notifications with MQTT significantly improved reliability and observability in our system.

By delegating connection management, routing, and delivery guarantees to the MQTT broker, we simplified backend services while improving system robustness.

EMQX combined with Kubernetes and HTTP-based authentication integrated cleanly with our existing infrastructure and provided a scalable messaging layer for desktop agents.