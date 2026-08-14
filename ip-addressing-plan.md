# IP Addressing Plan

## Overview

This network uses the private address range `10.0.0.0/16` as its overall address space, broken into three `/24` subnets. Private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) are reserved by RFC 1918 for internal use and aren't routable on the public internet —which is exactly why they're appropriate here for everything except the one component that genuinely needs a public IP (the load balancer).

## Why three separate /24 subnets instead of one big network

Splitting the network into public, app, and database subnets isn't just for tidiness it means routing and firewall rules can be applied per-subnet. For example, "the database subnet has no route to the internet" is a property of the network design itself, not just a rule someone has to remember to configure on each device individually.

## Subnet Breakdown

### 1. Public/Edge Subnet — `10.0.1.0/24`

| Field | Value |
|---|---|
| Network address | 10.0.1.0 |
| Usable range | 10.0.1.1 – 10.0.1.254 |
| Broadcast address | 10.0.1.255 |
| Total addresses | 256 (254 usable) |

| Device | IP |
|---|---|
| Edge Router (internal interface) | 10.0.1.1 |
| Firewall | 10.0.1.2 |
| NAT Gateway | 10.0.1.3 |
| Load Balancer | 10.0.1.10 |

The load balancer is the only device in this subnet — or in the whole network — that also holds a public IP address (assigned separately by the cloud provider/ISP; not part of the private plan above). Everything else in this subnet is reachable only from inside the network.

### 2. Private App Subnet — `10.0.2.0/24`

| Field | Value |
|---|---|
| Network address | 10.0.2.0 |
| Usable range | 10.0.2.1 – 10.0.2.254 |
| Broadcast address | 10.0.2.255 |
| Total addresses | 256 (254 usable) |

| Device | IP |
|---|---|
| App Server 1 | 10.0.2.10 |
| App Server 2 | 10.0.2.11 |

No device in this subnet has a public IP. The only way in is through the load balancer in the public subnet.

### 3. Private Database Subnet — `10.0.3.0/24`

| Field | Value |
|---|---|
| Network address | 10.0.3.0 |
| Usable range | 10.0.3.1 – 10.0.3.254 |
| Broadcast address | 10.0.3.255 |
| Total addresses | 256 (254 usable) |

| Device | IP |
|---|---|
| Database Server | 10.0.3.10 |

This subnet has no route to the internet at all, not even outbound through NAT, since a database has no real need to reach the internet directly.

## CIDR Notation — Beginner Explanation

CIDR notation looks like `10.0.1.0/24`. The number after the slash tells you how many bits of the 32-bit IP address are fixed as the "network" part — the rest are free for individual devices ("hosts").

- `/24` fixes the first 24 bits → 8 bits left for hosts → 2⁸ = 256 addresses → 254 usable (one address is reserved for the network itself, one for broadcast).
- A smaller number after the slash (like `/16`) means fewer bits are fixed, so there's a *bigger* network. A bigger number (like `/28`) fixes more bits, so there's a *smaller* network.

I chose `/24` for each subnet because it's large enough for a real startup's early infrastructure (254 devices per subnet) without being so large that IP planning becomes unwieldy, and it's the size most beginner networking material uses as a default — which made it easier for me to reason about while learning.

## Public vs. Private, Summarized

| | Public | Private |
|---|---|---|
| Reachable from internet? | Yes | No |
| Example in this design | Load Balancer | App Servers, Database |
| Needs NAT to reach internet? | No (already public) | Yes, for outbound only |
