---
title: "Routing and Switching"
date: 2026-03-05
author: mahi
categories: [software, networking]
tags: [gcs, networking, routing]
---

## Overview

Routing and Switching are the two core mechanisms for moving data across networks.

| Mechanism | Purpose | Scope |
|-----------|---------|-------|
| **Switching** | Moves data *within* the same network | Local Area Network (LAN) |
| **Routing** | Moves data *between* different networks | Between LANs / WAN |

**In a rover system:**
- **Switch** → Handles internal subsystem communication (cameras, sensors, compute)
- **Router** → Connects the rover's network to the Ground Control Station (GCS) network

---

## How Routers vs Switches Differ

| Feature | Switch | Router |
|---------|--------|--------|
| **OSI Layer** | Data Link Layer (L2) | Network Layer (L3) |
| **Address Used** | MAC Address | IP Address |
| **Function** | Connect devices within a LAN | Connect different networks |
| **Traffic Type** | Internal (same network) | External (between networks) |
| **Forwarding Decision** | Based on MAC table | Based on routing table |
| **Device Examples** | Managed/Unmanaged Switch | Home Router, Core Router |

**Switch flow:**  
`Device A → Switch → [MAC lookup] → Device B` (same LAN)

**Router flow:**  
`Device A → Router → [IP lookup + routing table] → Different Network → Device B`

---

## NAT — Network Address Translation

NAT allows devices with **private IP addresses** to communicate with the **public internet** through a single public IP.

### How NAT Works

```
Rover Internal Network           Internet
─────────────────────            ──────────────
192.168.1.10 ──►                 
192.168.1.20 ──► [Router/NAT] ──► 49.x.x.x (Public IP)
192.168.1.30 ──►                 
```

1. Device sends packet from private IP (`192.168.1.x`)
2. Router replaces private IP with its public IP in the packet header
3. Response comes back to the public IP
4. Router maps it back to the correct private IP

### Why NAT is Needed

- IPv4 has a limited number of public addresses (~4.3 billion)
- Private IPs (`192.168.x.x`, `10.x.x.x`, `172.16-31.x.x`) are **not routable** on the public internet
- NAT allows thousands of private devices to share one public IP

### Types of NAT

| Type | Description |
|------|-------------|
| **Static NAT** | One-to-one mapping (1 private IP ↔ 1 public IP) |
| **Dynamic NAT** | Pool of public IPs assigned dynamically |
| **PAT (Port Address Translation)** | Many private IPs share one public IP using different ports (most common) |

> IPv6 eliminates the need for NAT by providing enough addresses for every device globally.

---

## Port Forwarding

Port Forwarding allows **external devices to access a service running inside a private network** by mapping a public IP + port to an internal IP + port.

### How Port Forwarding Works

```
External GCS (Internet)
        │
        │  Requests: 49.x.x.x:1883
        ▼
    [Router]  ← Port Forwarding Rule
        │
        │  Forwards to: 192.168.1.20:1883
        ▼
  MQTT Broker (inside rover)
  192.168.1.20:1883
```

### Example: MQTT Broker on Rover

```
MQTT Broker running at:  192.168.1.20 : 1883

Router port forward rule:
  External: 49.x.x.x : 1883
      ↓
  Internal: 192.168.1.20 : 1883

Result: Remote GCS connects to 49.x.x.x:1883 and reaches rover broker
```

### Common Ports

| Service | Protocol | Default Port |
|---------|----------|-------------|
| MQTT | TCP | 1883 |
| MQTT (TLS) | TCP | 8883 |
| HTTP | TCP | 80 |
| HTTPS | TCP | 443 |
| SSH | TCP | 22 |
| FTP | TCP | 21 |

---

## VLAN — Virtual LAN

A VLAN (Virtual Local Area Network) logically **separates a physical network into multiple isolated networks** without requiring separate physical hardware.

### Why Use VLANs?

| Benefit | Description |
|---------|-------------|
| **Reduced Congestion** | Traffic is separated; broadcast domains are smaller |
| **Improved Security** | Systems can't communicate across VLANs without a router |
| **Better Traffic Control** | Critical traffic (e.g. control commands) is isolated |
| **Easier Management** | Logical grouping of devices regardless of physical location |

### Example: Rover VLAN Segmentation

```
Physical Network: One switch onboard the rover

VLAN 10 → Camera System        (192.168.10.x)
VLAN 20 → Navigation System    (192.168.20.x)
VLAN 30 → Communication System (192.168.30.x)
```

Each VLAN is isolated — a fault or security compromise in the camera system does not affect the navigation or communication systems.

### VLAN Tagging (802.1Q)

VLAN traffic is tagged with an ID in the Ethernet frame header. Managed switches use this tag to route traffic to the correct VLAN.

```
Ethernet Frame: [Dest MAC | Src MAC | VLAN Tag (802.1Q) | EtherType | Data | FCS]
```

- **Access Port** — Carries traffic for a single VLAN (untagged, connects end devices)
- **Trunk Port** — Carries traffic for multiple VLANs (tagged, connects switches/routers)

---

## Routing Tables

A routing table is a **database stored in a router** that determines where incoming data packets should be forwarded.

### Routing Table Structure

| Destination Network | Subnet Mask | Next Hop | Interface | Metric |
|--------------------|-------------|----------|-----------|--------|
| 192.168.1.0 | /24 | — | eth0 | 0 |
| 10.0.0.0 | /8 | 192.168.1.1 | eth0 | 1 |
| 0.0.0.0 | /0 | 192.168.1.1 | eth0 | 1 |

**Fields explained:**
- **Destination** — Target network or host
- **Subnet Mask** — Defines the network boundary
- **Next Hop** — IP of the next router to forward the packet to
- **Interface** — Which network interface to use
- **Metric** — Cost of the route (lower = preferred)

### Default Route

```
Destination: 0.0.0.0/0
```

The default route (`0.0.0.0/0`) is a **catch-all** — any packet with an unknown destination is forwarded to the **default gateway** (usually the upstream router).

---

## Static Routes

A static route is a **manually configured** path in a router's routing table.

### When to Use Static Routes

- Small networks with predictable traffic paths
- Specific paths that should never change (e.g. rover ↔ GCS link)
- Backup routes
- Security — prevent route changes from external sources

### Configuring a Static Route (Example)

**Scenario:**  
The rover needs to send data to the GCS network at `10.0.0.0/24`, reachable through the gateway at `192.168.1.1`.

```bash
# Linux (ip route command)
ip route add 10.0.0.0/24 via 192.168.1.1

# Verify
ip route show
```

**Resulting routing table entry:**
```
Destination: 10.0.0.0/24
Gateway:     192.168.1.1
Interface:   eth0
```

Now the rover knows to forward all packets destined for `10.0.0.0/24` through the gateway at `192.168.1.1`.

### Static vs Dynamic Routing

| Feature | Static Routing | Dynamic Routing |
|---------|----------------|-----------------|
| Configuration | Manual | Automatic |
| Adaptability | Fixed paths | Adapts to topology changes |
| Overhead | None | Protocol overhead (OSPF, BGP) |
| Complexity | Simple | More complex |
| Use Case | Small/stable networks | Large/complex networks |
| Protocols | — | RIP, OSPF, BGP, EIGRP |

---

## Summary: Data Path in a Rover System

```
[Rover Subsystem] 
      │ (VLAN-separated LAN)
      ▼
[Onboard Switch]  ← Layer 2, MAC-based forwarding
      │
      ▼
[Onboard Router]  ← Layer 3, IP routing + NAT
      │
      │ (Public IP via ISP / LTE / Radio link)
      ▼
[Ground Control Station]
```

- Subsystems communicate internally through the **switch**
- Telemetry and video reach the GCS through the **router**
- **NAT** translates private rover IPs to the public-facing IP
- **Port forwarding** allows the GCS to initiate connections to rover services
- **Static routes** ensure reliable, predictable routing between rover and GCS networks
