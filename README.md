# Week 2 — Networking Fundamentals

## Project Overview

This project is my Week 2 assignment for the Cloud Engineer by Audacity community. The scenario is simple: a startup has built an app and wants to understand how that app becomes reachable by users anywhere on the internet and what network components sit between a user typing a URL and the app actually responding.

Instead of just drawing a diagram, I designed an IP addressing plan, worked out how a request travels through each layer, and explained the reasoning behind every decision. This builds directly on Week 1, where I worked with Linux fundamentals, users/sudo, and SSH — this week moves from "inside a single machine" to "how machines talk to each other over a network."

## Architecture

[View Network Architecture Diagram](./network-architecture.png)

The diagram shows the full path from a user on the internet down to the database, with public and private zones clearly separated.

## Network Design

**Public subnet (10.0.1.0/24):** This is the only part of the network that is reachable from the internet. It holds the edge router, the firewall, the load balancer, and a NAT gateway.

**Private app subnet (10.0.2.0/24):** Two application servers live here. They have no public IP addresses — they can only be reached through the load balancer.

**Private database subnet (10.0.3.0/24):** The database sits in its own subnet, separate from the app servers. Only the app servers can talk to it. Nothing about it is reachable from the internet.

**DNS:** Before a user's device can even send a request, it needs to turn a domain name (like `app.startup.com`) into an IP address. DNS handles that translation and points the domain at the load balancer's public IP.

**Firewall:** Sits right after the edge router and only allows traffic on ports 80 (HTTP) and 443 (HTTPS) through to the load balancer. Everything else is blocked by default.

**NAT (Network Address Translation):** This is easy to get backwards, so I want to be explicit about it. NAT in this design is **outbound only** — it lets the private app and database servers reach the internet (for things like software updates) even though they don't have public IP addresses. NAT is not what lets a user's request *in*. Inbound requests reach the load balancer directly because the load balancer is the one component in the private layers that's assigned a public IP.

**Load balancer:** Receives all incoming requests on the public IP and distributes them across the two app servers.

## IP Addressing & CIDR

| Component | IP / Subnet | Purpose |
|---|---|---|
| Public/Edge subnet | 10.0.1.0/24 | Internet-facing components (router, firewall, load balancer, NAT gateway) |
| Edge Router | 10.0.1.1 | Entry/exit point between the internet and the internal network |
| Firewall | 10.0.1.2 | Filters traffic; only allows ports 80/443 through |
| NAT Gateway | 10.0.1.3 | Gives private subnets outbound-only internet access |
| Load Balancer | 10.0.1.10 | Public IP; distributes requests across app servers |
| App subnet | 10.0.2.0/24 | Private — application servers |
| App Server 1 | 10.0.2.10 | Handles application logic |
| App Server 2 | 10.0.2.11 | Handles application logic (redundancy) |
| Database subnet | 10.0.3.0/24 | Private — database only |
| Database Server | 10.0.3.10 | Stores application data, not internet-reachable |

**CIDR, explained simply:** CIDR notation (like `/24`) tells you how many addresses are in a network. The number after the slash is how many bits are "fixed" for the network itself — the rest are available for individual devices. A `/24` fixes the first 24 bits and leaves 8 bits free, which works out to 256 total addresses (254 usable once you subtract the network and broadcast addresses). I used three separate `/24` subnets instead of one big network so that the public-facing components, the app servers, and the database are cleanly separated from each other — that separation is what makes it possible to say "the database subnet has no route to the internet" as a rule, not just a hope.

## How the Internet Request Works

1. A user types `app.startup.com` into their browser.
2. Their device queries DNS, which resolves the domain to the load balancer's public IP address.
3. The browser opens a connection using HTTPS (port 443) to that IP.
4. The request reaches the edge router, which is the network's entry point from the internet.
5. The firewall evaluates the traffic. Since it's on port 443, it's allowed through; anything on a port that isn't 80/443 would be dropped here.
6. The load balancer receives the request.
7. The load balancer forwards it to whichever app server (10.0.2.10 or 10.0.2.11) is currently healthy and available.
8. If the request needs data, the app server queries the database server (10.0.3.10) over the private network — this traffic never leaves the internal subnets.
9. The app server builds a response and sends it back through the load balancer, back through the firewall/edge router, and out to the user.

## Security

- **Firewall:** Only ports 80 and 443 are allowed in from the internet. Every other port is blocked by default, which limits what an attacker can even attempt to reach.
- **Private subnets:** The app and database subnets have no public IP addresses at all. There's no direct path from the internet to either of them — the load balancer is the only entry point.
- **Least exposure:** Only the load balancer is internet-facing. Everything downstream of it is intentionally unreachable from outside.
- **HTTPS:** Encrypts traffic between the user and the load balancer so data can't be read or tampered with in transit.
- **NAT:** Lets private servers reach out for updates without needing a public IP that could be attacked directly.
- **Why the database isn't exposed:** If the database had a public IP, anyone on the internet could attempt to connect to it directly. Keeping it in a private subnet, reachable only from the app servers, means the only way to reach the database is by going through the application layer first — which is where authentication and validation actually happen.

## Load Balancing

I used two application servers instead of one so the app doesn't have a single point of failure. The load balancer continuously checks whether each app server is responding (a "health check"). If App Server 1 stops responding, the load balancer stops sending it traffic and routes all requests to App Server 2 until the first one recovers. Users shouldn't notice anything — the request still gets answered, just by the remaining healthy server.

## What I Learned

Going into this week, I understood networking as a vague idea — "the internet connects computers." Actually laying out an IP plan made it a lot more concrete. Figuring out where the NAT gateway actually belongs was the part that shifted my thinking the most — I originally assumed NAT was involved in letting user requests reach the app, and realized that's backwards once I worked through what NAT is actually translating and in which direction.

Separating the network into public/app/database subnets also made the security reasoning click for me. It's not just "add a firewall and hope" — the private subnets literally have no route to the internet, so there's a structural reason the database can't be reached from outside, not just a rule someone has to remember to enforce.

## Challenges

- Understanding CIDR notation and what the `/24` actually restricts took a couple of tries to stick.
- Keeping public and private IP ranges straight, and being consistent about which subnet each device belongs in.
- Understanding DNS as a separate step from the IP address itself — it's easy to blur "the domain" and "the address" together when you're new to this.
- Working out the actual difference between a router, a firewall, and a load balancer, since all three sit at "the edge" and it's tempting to treat them as interchangeable.
- Tracing how traffic moves through multiple layers (edge → firewall → load balancer → app → database) and keeping straight which hops are public and which are private.

## Future Improvements

These are things I haven't done yet, listed as next steps rather than claims about this project:

- Implement this design for real in AWS (VPC, subnets, security groups, an actual Application Load Balancer).
- Add VLANs to practice segmentation at a lower network layer.
- Configure real firewall rules (e.g., with `ufw` or a cloud security group) instead of describing them conceptually.
- Use Terraform to define this infrastructure as code.
- Deploy an actual load balancer (e.g., AWS ALB or NGINX) in front of real app servers.
- Add basic monitoring/alerting so I'd actually know if an app server went down.
- Add real HTTPS certificates (e.g., via Let's Encrypt or AWS Certificate Manager).

## Files in This Project

- [`network-architecture.png`](./network-architecture.png) — the architecture diagram
- [`ip-addressing-plan.md`](./ip-addressing-plan.md) — detailed IP addressing and subnet breakdown
- [`commands.txt`](./commands.txt) — Linux networking commands I practiced this week
- [`notes.md`](./notes.md) — my raw learning notes
