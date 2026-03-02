# Module 03: Networking Basics

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: The OSI Model](#beginner-the-osi-model)
- [Beginner: TCP/IP Fundamentals](#beginner-tcpip-fundamentals)
- [Beginner: IP Addressing & Subnetting](#beginner-ip-addressing--subnetting)
- [Beginner: DNS](#beginner-dns)
- [Beginner: Common Ports & Protocols](#beginner-common-ports--protocols)
- [Beginner: HTTP & HTTPS](#beginner-http--https)
- [Intermediate: Firewalls & iptables](#intermediate-firewalls--iptables)
- [Intermediate: Load Balancing](#intermediate-load-balancing)
- [Intermediate: Network Troubleshooting](#intermediate-network-troubleshooting)
- [Intermediate: Network Namespaces & Virtual Networking](#intermediate-network-namespaces--virtual-networking)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Networks are the nervous system of modern infrastructure. Every container you run, every server you deploy, every API call your application makes — all of it depends on networking. DevOps engineers who understand networking can diagnose production issues faster, design better infrastructure, and communicate effectively with network teams.

This module covers the key networking concepts every DevOps engineer must know, from TCP/IP basics to DNS, firewalls, and load balancing.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain the TCP/IP model and how data travels across a network
- Understand IP addressing, subnets, and CIDR notation
- Explain DNS resolution and debug DNS problems
- Identify common ports and protocols by number
- Understand HTTP request/response cycles including status codes
- Configure basic firewall rules with `iptables` and `ufw`
- Explain load balancing strategies and use cases
- Diagnose network problems using command-line tools

[↑ Back to TOC](#table-of-contents)

---

## Beginner: The OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework describing how data moves across a network in 7 layers.

| Layer | Name | Examples | What it Does |
|---|---|---|---|
| 7 | Application | HTTP, DNS, SSH, FTP | User-facing protocols |
| 6 | Presentation | TLS/SSL, encoding | Encryption, compression |
| 5 | Session | Sockets | Manages connections |
| 4 | Transport | TCP, UDP | End-to-end delivery, ports |
| 3 | Network | IP, ICMP | Routing between networks |
| 2 | Data Link | Ethernet, MAC | Node-to-node delivery |
| 1 | Physical | Cables, WiFi, fiber | Raw bit transmission |

> **DevOps tip**: You most commonly work at layers 3–7. When something fails, work from the bottom up — is the physical/IP layer working? Then transport? Then application?

[↑ Back to TOC](#table-of-contents)

---

## Beginner: TCP/IP Fundamentals

### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordered packets | No guarantee |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP/S, SSH, databases | DNS, video streaming, VoIP |

### The TCP 3-Way Handshake

```
Client          Server
  │── SYN ──────▶│    "Can we connect?"
  │◀── SYN-ACK ──│    "Yes, I'm ready"
  │── ACK ───────▶│    "Great, let's go"
  │═══ DATA ══════│    Connection established
```

### ICMP

ICMP (Internet Control Message Protocol) is used for diagnostics — `ping` uses ICMP echo requests and replies.

```bash
ping -c 4 8.8.8.8       # Send 4 ICMP packets to Google DNS
traceroute 8.8.8.8      # Trace the route packets take
mtr 8.8.8.8             # Combined ping + traceroute (real-time)
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: IP Addressing & Subnetting

### IPv4 Address Classes

IPv4 addresses are 32-bit numbers written as four octets (e.g., `192.168.1.100`).

| Range | Type | Common Use |
|---|---|---|
| `10.0.0.0/8` | Private | Large enterprise networks |
| `172.16.0.0/12` | Private | Mid-size networks |
| `192.168.0.0/16` | Private | Home/small office networks |
| `127.0.0.0/8` | Loopback | Localhost (your own machine) |
| Everything else | Public | Internet-routable |

### CIDR Notation

CIDR (Classless Inter-Domain Routing) expresses IP addresses and their network masks together.

```
192.168.1.0/24
│             └── 24 bits are network bits → 256 addresses (254 usable)
└── Network address

Common CIDR blocks:
/32  = 1 IP address (single host)
/30  = 4 IPs (2 usable — point-to-point links)
/29  = 8 IPs (6 usable)
/28  = 16 IPs (14 usable)
/24  = 256 IPs (254 usable) — typical LAN subnet
/16  = 65,536 IPs — VPC/large network
/8   = 16,777,216 IPs — full class A network
```

### IPv6

IPv6 uses 128-bit addresses written in hexadecimal. Increasingly common in cloud networking.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
# Can be compressed: 2001:db8:85a3::8a2e:370:7334
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: DNS

DNS (Domain Name System) translates human-readable domain names into IP addresses.

### How DNS Resolution Works

```
Browser asks: "What is the IP for app.example.com?"

1. Check local cache (fastest)
2. Check /etc/hosts file
3. Ask configured DNS resolver (e.g., 8.8.8.8)
4. Resolver queries Root DNS servers
5. Root servers point to .com TLD servers
6. TLD servers point to example.com nameservers
7. example.com nameservers return the IP
8. Answer cached and returned to browser
```

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| **A** | Domain → IPv4 address | `app.example.com → 1.2.3.4` |
| **AAAA** | Domain → IPv6 address | `app.example.com → 2001:db8::1` |
| **CNAME** | Domain → another domain | `www → app.example.com` |
| **MX** | Mail server for domain | `example.com → mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM, verification) | `v=spf1 include:...` |
| **NS** | Nameservers for domain | `example.com NS ns1.example.com` |
| **PTR** | IP → domain (reverse DNS) | `1.2.3.4 → app.example.com` |
| **SRV** | Service location (host + port) | Used by Kubernetes, SIP |

### DNS Commands

```bash
# Query DNS records
dig example.com                      # Default A record query
dig example.com MX                   # Query MX records
dig example.com NS                   # Query nameservers
dig @8.8.8.8 example.com            # Query using a specific resolver
dig +short example.com              # Just the IP address

nslookup example.com                # Interactive DNS query tool
host example.com                    # Simple DNS lookup

# Check /etc/hosts first
cat /etc/hosts                      # Local hostname overrides
cat /etc/resolv.conf               # Configured DNS resolvers
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Common Ports & Protocols

| Port | Protocol | Service |
|---|---|---|
| 20, 21 | TCP | FTP (File Transfer Protocol) |
| 22 | TCP | SSH (Secure Shell) |
| 25 | TCP | SMTP (email sending) |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL / MariaDB |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP (alternate/dev) |
| 8443 | TCP | HTTPS (alternate) |
| 27017 | TCP | MongoDB |

```bash
# Check what's listening on your system
ss -tulnp                           # Modern — show TCP/UDP listeners
netstat -tulnp                      # Classic equivalent
lsof -i :80                         # Show what's using port 80

# Test if a port is open
nc -zv hostname 443                 # netcat port test
telnet hostname 22                  # Test SSH port (legacy)
curl -v telnet://hostname:22        # Test via curl
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: HTTP & HTTPS

HTTP is the protocol that powers the web. As a DevOps engineer you will work with it constantly — debugging APIs, configuring load balancers, and setting up TLS.

### HTTP Request Structure

```
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGc...
Content-Type: application/json
Accept: application/json

{"name": "Alice"}
```

### HTTP Status Codes

| Range | Category | Common Codes |
|---|---|---|
| `1xx` | Informational | `100 Continue` |
| `2xx` | Success | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| `4xx` | Client Error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found` |
| `5xx` | Server Error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable` |

### curl for HTTP Testing

```bash
curl https://example.com                          # GET request
curl -I https://example.com                       # HEAD — headers only
curl -X POST https://api.example.com/users \      # POST with JSON body
     -H "Content-Type: application/json" \
     -d '{"name": "Alice"}'
curl -u user:password https://api.example.com     # Basic auth
curl -H "Authorization: Bearer TOKEN" https://...  # Bearer token
curl -o output.html https://example.com           # Save to file
curl -w "%{http_code}" -o /dev/null https://...   # Print status code only
curl -k https://self-signed.example.com           # Skip TLS verification
```

### TLS/SSL

HTTPS = HTTP + TLS. TLS encrypts data in transit and verifies server identity via certificates.

```bash
# Inspect a site's TLS certificate
openssl s_client -connect example.com:443 -showcerts
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check certificate expiry
curl -vI https://example.com 2>&1 | grep -i "expire"
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Firewalls & iptables

A firewall controls which network traffic is allowed to enter or leave a system.

### ufw — Uncomplicated Firewall (Ubuntu)

```bash
sudo ufw enable                     # Enable firewall
sudo ufw status verbose             # Show rules and status
sudo ufw allow 22/tcp               # Allow SSH
sudo ufw allow 80/tcp               # Allow HTTP
sudo ufw allow 443/tcp              # Allow HTTPS
sudo ufw deny 3306/tcp              # Block MySQL from outside
sudo ufw allow from 192.168.1.0/24  # Allow entire subnet
sudo ufw delete allow 80/tcp        # Remove a rule
sudo ufw reset                      # Reset all rules
```

### iptables — Low-Level Firewall

```bash
# List all rules
sudo iptables -L -v -n

# Allow established connections (essential — do this first!)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop everything else
sudo iptables -P INPUT DROP

# Save rules (Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Load Balancing

A load balancer distributes incoming traffic across multiple servers to improve availability and performance.

### Load Balancing Algorithms

| Algorithm | Description | Best For |
|---|---|---|
| **Round Robin** | Requests distributed in rotation | Stateless apps with equal capacity servers |
| **Least Connections** | Send to server with fewest active connections | Long-lived connections, varying request complexity |
| **IP Hash** | Same client IP always goes to same server | Session-sticky applications |
| **Weighted Round Robin** | Servers get traffic proportional to weight | Mixed-capacity server pools |
| **Health-Check Based** | Automatically remove unhealthy servers | Production high-availability |

### Layer 4 vs Layer 7 Load Balancing

| Type | Layer | Sees | Examples |
|---|---|---|---|
| **Layer 4** | Transport (TCP/UDP) | IP + port only | AWS NLB, HAProxy TCP mode |
| **Layer 7** | Application (HTTP) | Full HTTP request — URL, headers, cookies | AWS ALB, Nginx, HAProxy HTTP mode |

### Nginx as Load Balancer

```nginx
upstream backend {
    least_conn;                        # Least connections algorithm
    server web01.example.com:8080;
    server web02.example.com:8080;
    server web03.example.com:8080;
    keepalive 32;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Network Troubleshooting

A systematic approach to diagnosing network problems:

```
Layer 1/2: Can the NIC send/receive? (ip link, ethtool)
Layer 3: Can we reach the gateway? (ping gateway IP)
Layer 3: Can we reach external IPs? (ping 8.8.8.8)
Layer 7/DNS: Can we resolve names? (dig example.com)
Layer 7/App: Can we reach the service? (curl, telnet, nc)
```

### Diagnostic Commands

```bash
# Interface status
ip addr show                        # Show IP addresses
ip link show                        # Show interface status
ip route show                       # Show routing table
ip route get 8.8.8.8               # Show route to specific destination

# Connectivity
ping -c 4 8.8.8.8                  # Test IP connectivity
traceroute 8.8.8.8                 # Trace route
mtr --report 8.8.8.8              # Combined ping + traceroute

# DNS debugging
dig +trace example.com             # Full DNS resolution trace
dig example.com @1.1.1.1          # Query Cloudflare DNS directly

# Port and service testing
nc -zv example.com 443             # Test TCP port
ss -tulnp                          # Show all listening services
ss -s                              # Socket statistics summary

# Traffic capture
sudo tcpdump -i eth0 port 80       # Capture HTTP traffic
sudo tcpdump -i eth0 host 1.2.3.4  # Capture traffic to/from an IP
sudo tcpdump -i eth0 -w capture.pcap  # Save to file for Wireshark
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Network Namespaces & Virtual Networking

Linux network namespaces are the technology behind container networking. Each container gets its own isolated network stack.

```bash
# Create a network namespace
sudo ip netns add myns

# List namespaces
ip netns list

# Run a command inside a namespace
sudo ip netns exec myns bash

# Create a veth pair (virtual ethernet — like a virtual cable)
sudo ip link add veth0 type veth peer name veth1

# Move one end into the namespace
sudo ip link set veth1 netns myns

# Assign IPs
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip netns exec myns ip addr add 10.0.0.2/24 dev veth1

# Bring interfaces up
sudo ip link set veth0 up
sudo ip netns exec myns ip link set veth1 up

# Test connectivity between host and namespace
ping 10.0.0.2
```

> **DevOps tip**: This is exactly how Docker and Podman create network isolation between containers. Understanding namespaces helps you debug container networking issues.

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `ping` | Test ICMP connectivity |
| `traceroute` / `mtr` | Trace packet route |
| `dig` / `nslookup` / `host` | DNS lookups |
| `curl` | HTTP requests and testing |
| `nc` (netcat) | TCP/UDP port testing |
| `ss` / `netstat` | Show listening ports and connections |
| `ip addr` / `ip route` | Show interfaces and routing |
| `tcpdump` | Capture network traffic |
| `ufw` | Simple firewall management |
| `iptables` | Advanced firewall rules |
| `openssl s_client` | TLS certificate inspection |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 3.1 — DNS Investigation

1. Use `dig example.com` to query A records
2. Use `dig example.com MX` to query mail records
3. Use `dig +trace example.com` to see the full resolution path
4. Edit `/etc/hosts` to override `example.com` to `127.0.0.1` and test
5. Remove the override and confirm normal resolution resumes

### Lab 3.2 — Port Scanning & Services

1. Run `ss -tulnp` and identify every service listening on your machine
2. Use `nc -zv localhost 22` to confirm SSH is running
3. Start a simple HTTP server: `python3 -m http.server 8080 &`
4. Confirm it appears in `ss -tulnp`
5. Test it: `curl http://localhost:8080`
6. Stop the server with `kill $(lsof -ti:8080)`

### Lab 3.3 — Firewall Rules

1. Install and check `ufw` status
2. Set default policy to deny incoming: `sudo ufw default deny incoming`
3. Allow SSH: `sudo ufw allow 22/tcp`
4. Allow HTTP: `sudo ufw allow 80/tcp`
5. Enable the firewall and verify: `sudo ufw status verbose`
6. Attempt to connect on a blocked port and observe the behavior

### Lab 3.4 — HTTP Deep Dive

1. Use `curl -v https://example.com` to see the full TLS handshake and HTTP headers
2. Check the TLS certificate expiry date
3. Make a POST request to `https://httpbin.org/post` with a JSON body
4. Examine the HTTP status codes returned by different endpoints
5. Use `curl -w "%{http_code} %{time_total}s\n" -o /dev/null https://example.com`

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Computer Networking: A Top-Down Approach](https://gaia.cs.umass.edu/kurose_ross/) — Kurose & Ross
- [Nginx Documentation](https://nginx.org/en/docs/)
- [iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
- [DNS in Detail (TryHackMe)](https://tryhackme.com/room/dnsindetail)
- [Glossary: DNS](./glossary.md#d), [Firewall](./glossary.md#f), [Load Balancer](./glossary.md#l), [TCP/IP](./glossary.md#t)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
