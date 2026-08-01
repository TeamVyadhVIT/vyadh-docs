---
title: "IP Addressing"
date: 2026-03-05
author: mahi
categories: [software, networking]
tags: [gcs, networking, ip]
---

## What is IP?

IP (Internet Protocol) is a **Network Layer (Layer 3)** protocol used to uniquely identify devices on a network and enable data communication between them.

---

## IPv4

IPv4 (Internet Protocol Version 4) is the most widely used version of IP for identifying devices on a network.

- **Address size:** 32-bit
- **Format:** Decimal, separated by dots
- **Example:** `192.168.1.25`
- Each section is called an **octet** (8 bits each → 4 octets = 32 bits)

### IPv4 Structure

Every IPv4 address has two parts:

| Part | Purpose |
|------|---------|
| **Network ID** | Identifies the network |
| **Host ID** | Identifies the device within that network |

The split between Network ID and Host ID is determined by the **subnet mask**.

**Example:**

```
IP Address:   192.168.1.10
Subnet Mask:  255.255.255.0
```

→ First 24 bits = Network (`192.168.1`)  
→ Last 8 bits = Host (`.10`)

---

## IPv4 Classes

IPv4 addresses are divided into classes based on their range:

| Class | Range | Default Subnet Mask | Use |
|-------|-------|---------------------|-----|
| **A** | 1.0.0.0 – 126.255.255.255 | 255.0.0.0 | Large networks |
| **B** | 128.0.0.0 – 191.255.255.255 | 255.255.0.0 | Medium networks |
| **C** | 192.0.0.0 – 223.255.255.255 | 255.255.255.0 | Small networks |
| **D** | 224.0.0.0 – 239.255.255.255 | Multicast | Streaming |
| **E** | 240.0.0.0 – 255.255.255.255 | Reserved | Research |

> **Note:** `127.x.x.x` is reserved for loopback (localhost).

---

## Private vs Public IP Ranges

Not all IP addresses are routable on the public internet. Private ranges are used inside local networks (LANs):

| Class | Private Range | Usage |
|-------|--------------|-------|
| A | 10.0.0.0 – 10.255.255.255 | Large private networks |
| B | 172.16.0.0 – 172.31.255.255 | Medium private networks |
| C | 192.168.0.0 – 192.168.255.255 | Home/small office networks |

**Public IPs** are assigned by ISPs and are routable on the internet.  
**Private IPs** require **NAT** (Network Address Translation) to communicate externally.

---

## Subnetting and CIDR Notation

### Subnetting

Subnetting means dividing a large network into smaller sub-networks (subnets).

**Why we subnet:**
- Reduce network traffic
- Improve performance
- Better device management
- Security separation between segments

### Subnet Mask

The subnet mask defines which portion of an IP is the network and which is the host.

| Subnet Mask | CIDR | Hosts Available |
|-------------|------|-----------------|
| 255.0.0.0 | /8 | 16,777,214 |
| 255.255.0.0 | /16 | 65,534 |
| 255.255.255.0 | /24 | 254 |
| 255.255.255.128 | /25 | 126 |
| 255.255.255.192 | /26 | 62 |

### CIDR Notation

CIDR (Classless Inter-Domain Routing) is a compact way to express a subnet mask using a prefix length.

```
192.168.1.0/24
```

→ `/24` means the first 24 bits are the network portion.

### Wildcard Mask

A wildcard mask is the inverse of a subnet mask. Used in routing and firewall rules.

```
Subnet Mask:   255.255.255.0
Wildcard Mask: 0.0.0.255
```

---

## Static vs DHCP Addressing

### Static IP

- Manually configured on each device
- IP address does not change
- Used for: servers, routers, printers, rover onboard computers

**Example:**
```
IP:      192.168.1.20
Mask:    255.255.255.0
Gateway: 192.168.1.1
```

### DHCP (Dynamic Host Configuration Protocol)

- IP address automatically assigned by a **DHCP server**
- Lease-based (address can change over time)
- Used for: client devices, laptops, mobile devices

**DHCP process (DORA):**
1. **Discover** – Client broadcasts request
2. **Offer** – Server offers an IP
3. **Request** – Client accepts the offer
4. **Acknowledge** – Server confirms the lease

---

## Gateway and DNS Basics

### Default Gateway

The default gateway is the router IP address that a device sends traffic to when the destination is outside the local network.

```
Device IP:       192.168.1.10
Default Gateway: 192.168.1.1   ← Router
```

### DNS (Domain Name System)

DNS translates human-readable domain names into IP addresses.

```
google.com  →  142.250.190.14
```

| DNS Type | Example |
|----------|---------|
| Public DNS (Google) | 8.8.8.8 |
| Public DNS (Cloudflare) | 1.1.1.1 |
| Local DNS | 192.168.1.1 (router) |

---

## IPv6

IPv6 was developed to overcome IPv4 address exhaustion.

- **Address size:** 128-bit
- **Format:** Hexadecimal, separated by colons
- **Example:** `2001:db8::1`

### IPv4 vs IPv6 Comparison

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address Size | 32-bit | 128-bit |
| Format | Decimal | Hexadecimal |
| Example | 192.168.1.1 | 2001:db8::1 |
| NAT Required | Yes | No |
| Address Space | ~4.3 billion | 340 undecillion |
| Header Complexity | More complex | Simplified |

### Key Benefits of IPv6
- Virtually unlimited address space
- No need for NAT
- Built-in IPSec support
- Simplified header for better routing performance
- Better support for mobile networks
