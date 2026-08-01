---
title: "TCP and UDP"
date: 2026-03-05
author: mahi
categories: [software, networking]
tags: [gcs, networking, tcp, udp]
---

## Overview

TCP and UDP are the two primary **Transport Layer (Layer 4)** protocols used to send data across a network. They serve different purposes based on the trade-off between **reliability** and **speed**.

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Yes (connection-oriented) | No (connectionless) |
| Reliability | High | Low |
| Speed | Slower | Faster |
| Packet Order | Maintained | Not maintained |
| Retransmission | Yes | No |
| Latency | Higher | Lower |
| Use Case | Telemetry, commands | Video streaming, real-time data |

---

## TCP — Transmission Control Protocol

TCP is a **connection-oriented** transport layer protocol designed for **reliable, ordered, and error-checked** data delivery.

### TCP 3-Way Handshake

Before any data is transmitted, TCP establishes a connection through a 3-step handshake:

```
Client                    Server
  │                          │
  │──── SYN ───────────────► │   Step 1: Client requests connection
  │                          │
  │ ◄─── SYN-ACK ───────────│   Step 2: Server acknowledges
  │                          │
  │──── ACK ───────────────► │   Step 3: Client confirms
  │                          │
  │═══════ Data Flow ════════│   Connection established
```

1. **SYN** — Client sends a synchronization request to start communication
2. **SYN-ACK** — Server acknowledges and responds with its own sync signal
3. **ACK** — Client confirms the acknowledgement; connection is established

### Reliability Features

TCP guarantees data delivery through multiple mechanisms:

| Mechanism | Description |
|-----------|-------------|
| **Acknowledgements (ACK)** | Receiver confirms each packet was received |
| **Retransmission** | Lost or corrupted packets are re-sent |
| **Error Checking** | Checksums verify data integrity |
| **Packet Sequencing** | Packets are numbered and reassembled in order |
| **Flow Control** | Prevents sender from overwhelming the receiver |
| **Congestion Control** | Reduces transmission rate if network is congested |

### TCP Connection Termination

TCP closes connections gracefully using a **4-way FIN handshake**:

```
Client ──FIN──► Server
Client ◄──ACK── Server
Client ◄──FIN── Server
Client ──ACK──► Server
```

Both sides confirm they are done sending before the connection is closed.

### When to Use TCP

Use TCP when **accuracy is more important than speed** and data loss is unacceptable.

**Use cases:**
- MQTT telemetry transfer
- File transfers
- Remote command execution
- System health monitoring
- Web communication (HTTP/HTTPS)
- Database queries

---

## UDP — User Datagram Protocol

UDP is a **connectionless** transport layer protocol designed for **fast, low-latency** data transmission.

### Stateless Nature

UDP sends packets directly **without any handshake or session setup**:

- Does not establish a connection before sending
- Does not check whether packets were received
- Does not retransmit lost packets
- Does not maintain packet order
- Does not track session state

```
Sender                    Receiver
  │                          │
  │──── Packet 1 ──────────► │
  │──── Packet 2 ──────────► │   No ACK, no handshake
  │──── Packet 3 ──────────► │   Fire and forget
  │──── Packet 4 ──────────X │   ← Lost, not retransmitted
  │──── Packet 5 ──────────► │
```

### Why UDP is Preferred for Real-Time Data

In real-time systems, **the latest data matters more than complete data**:

- Very low latency — no connection setup delay
- No acknowledgement overhead
- No retransmission delays
- Faster packet transmission
- Tolerant to minor packet loss

**Video streaming example:**

```
If a video frame is lost:
  ✗ TCP behavior: Wait for retransmission → causes visible lag
  ✓ UDP behavior: Skip and display next frame immediately
```

Waiting for retransmission in real-time applications would cause unacceptable delays in:
- Teleoperation control response
- Obstacle detection feeds
- Navigation updates

### When to Use UDP

Use UDP when **speed is more important than guaranteed delivery**.

**Use cases:**
- Live video streaming
- Audio streaming
- IMU sensor data
- Real-time telemetry
- Real-time control systems
- Online gaming
- DNS queries

---

## TCP vs UDP — Detailed Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| **OSI Layer** | Transport (Layer 4) | Transport (Layer 4) |
| **Connection** | Connection-oriented | Connectionless |
| **Handshake** | 3-way handshake | None |
| **Reliability** | Guaranteed delivery | Best-effort delivery |
| **Packet Order** | Maintained with sequencing | Not guaranteed |
| **Retransmission** | Yes, on packet loss | No |
| **Error Checking** | Checksum + ACK | Checksum only |
| **Flow Control** | Yes | No |
| **Congestion Control** | Yes | No |
| **Speed** | Slower (overhead) | Faster (minimal overhead) |
| **Latency** | Higher | Lower |
| **Header Size** | 20 bytes | 8 bytes |
| **Use Cases** | MQTT, HTTP, file transfer | Video, audio, live telemetry |

---

## ICMP — Internet Control Message Protocol

ICMP is a **Network Layer (Layer 3)** protocol used for diagnostics and error reporting rather than data transfer.

**Used for:**
- Network connectivity testing
- Error reporting between routers
- Path diagnostics

### Ping

`ping` is the most common use of ICMP. It tests whether a device is reachable and measures round-trip time.

```bash
ping 192.168.1.1
```

**How it works:**
1. Sends an **ICMP Echo Request** to the target
2. Waits for an **ICMP Echo Reply**
3. Reports round-trip time (RTT) and packet loss

**Output example:**
```
PING 192.168.1.1: 56 bytes of data
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=1.23 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.98 ms

--- ping statistics ---
2 packets transmitted, 2 received, 0% packet loss
```

**Key metrics from ping:**
- **RTT (Round Trip Time)** — Network latency
- **Packet Loss %** — Network reliability
- **TTL (Time to Live)** — How many hops the packet can travel

### Traceroute

`traceroute` uses ICMP to map the path packets take from source to destination:

```bash
traceroute 8.8.8.8
```

Shows each router (hop) along the path with latency at each hop.

---

## HTTP and HTTPS

### HTTP — Hypertext Transfer Protocol

HTTP is an **Application Layer (Layer 7)** protocol used for communication between web clients and servers.

**Characteristics:**
- Stateless — each request is independent
- Request-Response based
- Runs over **TCP** (default port **80**)

**Used in:**
- Web dashboards
- REST APIs
- GCS web interfaces

### HTTPS — Hypertext Transfer Protocol Secure

HTTPS is the **secure version of HTTP**, adding encryption via **SSL/TLS**.

**Ensures:**
- **Confidentiality** — Data is encrypted in transit
- **Integrity** — Data cannot be tampered with
- **Authentication** — Server identity is verified via certificates

**Used in:**
- Remote GCS access over the internet
- Secure REST API communication
- Encrypted control command transmission

### HTTP vs HTTPS

| Feature | HTTP | HTTPS |
|---------|------|-------|
| Security | None | SSL/TLS encrypted |
| Encryption | No | Yes |
| Default Port | 80 | 443 |
| Data Protection | No | Yes |
| Use Case | Local GCS UI | Remote secure access |
| Certificate Required | No | Yes (SSL cert) |
