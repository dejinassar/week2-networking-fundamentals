# Week 2 Notes — Networking Fundamentals

Raw notes from working through IP addressing, DNS, HTTP/HTTPS, firewalls, NAT,
and load balancing this week.

## IP Addressing

- Every device on a network needs a unique IP address to send/receive traffic.
- Private IP ranges (won't work on the open internet, only inside a network):
  - 10.0.0.0 – 10.255.255.255
  - 172.16.0.0 – 172.31.255.255
  - 192.168.0.0 – 192.168.255.255
- Public IPs are the ones actually routable on the internet — assigned by an
  ISP or cloud provider.
- A device with only a private IP can't be reached directly from the internet.
  That's a feature, not a bug — it's why I put the app servers and database
  on private IPs only.

## CIDR

- `/24` = 256 addresses, 254 usable.
- The number after the slash = how many bits are "locked" for the network.
  Fewer locked bits = bigger network. More locked bits = smaller network.
- Took me a couple of tries to stop thinking of the slash number as "how many
  addresses" and start thinking of it as "how many bits are fixed."

## DNS

- DNS is basically the internet's phone book — turns a domain name into an
  IP address.
- Without DNS you'd have to remember IP addresses instead of domain names.
- `nslookup` was useful for actually seeing this happen instead of just
  reading about it.
- Important distinction I kept blurring together at first: the domain name
  and the IP address are two different things, and DNS is the translation
  step in between — it's not "part of" the IP address.

## HTTP / HTTPS

- HTTP = how a browser and a server talk to each other (request → response).
- HTTPS = HTTP but encrypted, using TLS. Port 443 instead of port 80.
- Basically every production app should be HTTPS-only at this point.

## Firewalls

- A firewall decides what traffic is allowed in or out, usually based on
  port number and/or IP address.
- In my design, the firewall only allows ports 80 and 443 through from the
  internet — everything else gets dropped by default.
- Difference from a router: a router's job is to move traffic toward its
  destination. A firewall's job is to decide whether that traffic should be
  allowed at all. They can live on the same device but they're doing
  different jobs.

## NAT

- This one confused me at first. My initial (wrong) assumption was that NAT
  is what lets a user's request reach the app server.
- What NAT actually does: lets devices with only a private IP reach *out* to
  the internet, by translating their private IP to a shared public IP at the
  edge of the network.
- So NAT in my design is outbound-only — app/database servers reaching out
  for updates, not users reaching in. The load balancer is what handles
  inbound traffic, because it's the one component that actually has a
  public IP.

## Load Balancing

- A load balancer sits in front of multiple servers and spreads incoming
  requests across them.
- Two reasons this matters: (1) handling more traffic than one server could
  on its own, and (2) redundancy — if one server goes down, the load
  balancer stops sending it traffic and the app stays up.
- "Health checks" = the load balancer periodically checking whether each
  server is actually responding.

## Router vs. Firewall vs. Load Balancer (the thing I kept mixing up)

- Router: moves traffic between networks, gets it pointed in the right
  direction.
- Firewall: decides whether traffic is allowed through at all.
- Load balancer: once traffic IS allowed through, decides *which specific
  server* should handle it.

Three different jobs, even though all three can sit near "the edge" of a
network and it's easy to lump them together when you're new to this.

## Random things I want to look into next

- What actually happens during a TLS handshake (I know HTTPS uses it, don't
  fully understand the steps yet).
- The difference between Layer 4 (transport) and Layer 7 (application) load
  balancing — I think my load balancer here is more of a Layer 7 concept
  since it's routing based on HTTP, but I want to confirm that.
- Actually setting up `ufw` rules instead of just checking status with it.
