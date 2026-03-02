# Module 03: Networking Basics

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 03 of 15](https://img.shields.io/badge/module-03%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Beginner-brightgreen) ![curl 8.5+](https://img.shields.io/badge/curl-8.5%2B-073551?logo=curl&logoColor=white) ![tcpdump 4.99+](https://img.shields.io/badge/tcpdump-4.99%2B-grey) ![netcat 1.10+](https://img.shields.io/badge/netcat-1.10%2B-grey) ![TCP/UDP · DNS · HTTP](https://img.shields.io/badge/protocols-TCP%2FUDP%20%C2%B7%20DNS%20%C2%B7%20HTTP-blue)

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
- [Intermediate: Reverse Proxies](#intermediate-reverse-proxies)
- [Intermediate: Network Troubleshooting](#intermediate-network-troubleshooting)
- [Intermediate: Network Namespaces & Virtual Networking](#intermediate-network-namespaces--virtual-networking)
- [Advanced: DNS at Scale](#advanced-dns-at-scale)
- [Advanced: Service Mesh Introduction](#advanced-service-mesh-introduction)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Networks are the nervous system of modern infrastructure. Every container you run, every server you deploy, every API call your application makes — all of it depends on networking. DevOps engineers who understand networking can diagnose production issues faster, design better infrastructure, and communicate effectively with network teams.

This module covers the key networking concepts every DevOps engineer must know, from TCP/IP basics and DNS fundamentals through load balancing, reverse proxies, and the deep operational mechanics of modern tools like HAProxy and Traefik.

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
- Explain load balancing strategies and configure HAProxy and Nginx as load balancers
- Configure Traefik as a reverse proxy using both Docker and Kubernetes providers
- Set up automatic TLS with Let's Encrypt via Traefik
- Diagnose network problems using command-line tools
- Understand DNS-based load balancing and split-horizon DNS at scale
- Explain what a service mesh is and when to reach for one

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

**Why this matters in practice:**

- A `502 Bad Gateway` is a Layer 7 problem — your reverse proxy reached a backend that returned a bad response
- A connection timeout is often a Layer 3/4 problem — routing or firewall
- A `503 Service Unavailable` from a load balancer means all backends failed health checks (Layer 7)
- SSL certificate errors are Layer 6 — TLS handshake failed before any HTTP was exchanged

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

**Connection teardown** uses a 4-step FIN/ACK exchange. When you see lots of `TIME_WAIT` sockets, connections are being closed and waiting for late packets — normal on busy servers.

```bash
# See TCP connection states
ss -s
# Output shows: established, time-wait, close-wait, etc.
```

### ICMP

ICMP (Internet Control Message Protocol) is used for diagnostics — `ping` uses ICMP echo requests and replies.

```bash
ping -c 4 8.8.8.8       # Send 4 ICMP packets to Google DNS
traceroute 8.8.8.8      # Trace the route packets take
mtr 8.8.8.8             # Combined ping + traceroute (real-time)
```

> **Firewall note**: Many cloud providers block ICMP by default. A failed `ping` doesn't necessarily mean the host is down — test with `nc` or `curl` on a known-open port first.

[↑ Back to TOC](#table-of-contents)

---

## Beginner: IP Addressing & Subnetting

### IPv4 Address Classes

IPv4 addresses are 32-bit numbers written as four octets (e.g., `192.168.1.100`).

| Range | Type | Common Use |
|---|---|---|
| `10.0.0.0/8` | Private | Large enterprise networks, cloud VPCs |
| `172.16.0.0/12` | Private | Mid-size networks, Docker default bridge |
| `192.168.0.0/16` | Private | Home/small office networks |
| `127.0.0.0/8` | Loopback | Localhost (your own machine) |
| `169.254.0.0/16` | Link-local | APIPA, cloud instance metadata |
| Everything else | Public | Internet-routable |

> **Cloud note**: The metadata endpoint `169.254.169.254` is how EC2/GCE/Azure instances retrieve instance metadata and IAM credentials. If you see traffic going to this IP, it's normal cloud behavior.

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
/27  = 32 IPs (30 usable)
/24  = 256 IPs (254 usable) — typical LAN subnet
/16  = 65,536 IPs — VPC/large network
/8   = 16,777,216 IPs — full class A network
```

**Subnet calculation shortcut**: For `/N`, the number of hosts = `2^(32-N) - 2`.
So `/24` = `2^8 - 2` = 254 usable hosts.

### IPv6

IPv6 uses 128-bit addresses written in hexadecimal. Increasingly common in cloud networking.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
# Can be compressed: 2001:db8:85a3::8a2e:370:7334
```

| IPv6 Range | Purpose |
|---|---|
| `::1/128` | Loopback (equivalent to 127.0.0.1) |
| `fe80::/10` | Link-local (auto-configured per interface) |
| `fd00::/8` | Unique local (private, like RFC1918) |
| `2000::/3` | Global unicast (public internet) |

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
4. Resolver queries Root DNS servers (13 root clusters)
5. Root servers point to .com TLD servers
6. TLD servers point to example.com nameservers
7. example.com nameservers return the IP
8. Answer cached per TTL and returned to browser
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
| **CAA** | Certificate Authority Authorization | Restricts which CAs can issue certs |

### DNS Commands

```bash
# Query DNS records
dig example.com                      # Default A record query
dig example.com MX                   # Query MX records
dig example.com NS                   # Query nameservers
dig example.com TXT                  # Query TXT records (SPF, DKIM)
dig @8.8.8.8 example.com            # Query using a specific resolver
dig +short example.com              # Just the IP address
dig +trace example.com              # Full delegation trace from root

nslookup example.com                # Interactive DNS query tool
host example.com                    # Simple DNS lookup

# Reverse DNS lookup
dig -x 1.2.3.4                      # PTR record for an IP

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
| 67, 68 | UDP | DHCP (server/client) |
| 80 | TCP | HTTP |
| 123 | UDP | NTP (time sync) |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL / MariaDB |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP (alternate/dev) |
| 8443 | TCP | HTTPS (alternate) |
| 9090 | TCP | Prometheus |
| 9100 | TCP | Node Exporter (Prometheus) |
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
| `4xx` | Client Error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| `5xx` | Server Error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

**DevOps-specific status codes to know well:**

| Code | Meaning | Common Cause |
|---|---|---|
| `502 Bad Gateway` | Proxy received an invalid response | Backend crashed, wrong port, app error |
| `503 Service Unavailable` | No healthy backend available | All backends failed health checks |
| `504 Gateway Timeout` | Backend too slow to respond | DB query hanging, app overloaded |
| `429 Too Many Requests` | Rate limit exceeded | Client sending too many requests |

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
curl -v https://example.com                       # Verbose — show full exchange
curl --resolve example.com:443:1.2.3.4 https://example.com  # Test specific IP
```

### TLS/SSL

HTTPS = HTTP + TLS. TLS encrypts data in transit and verifies server identity via certificates.

```bash
# Inspect a site's TLS certificate
openssl s_client -connect example.com:443 -showcerts
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check certificate expiry
curl -vI https://example.com 2>&1 | grep -i "expire\|subject\|issuer"

# Test TLS version support
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_3
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
sudo ufw allow from 10.0.0.5 to any port 5432  # Allow specific host to PostgreSQL
sudo ufw delete allow 80/tcp        # Remove a rule
sudo ufw reset                      # Reset all rules

# Rate limiting (brute force protection)
sudo ufw limit 22/tcp               # Allow SSH but rate-limit connection attempts
```

### iptables — Low-Level Firewall

`iptables` operates on **chains** (`INPUT`, `OUTPUT`, `FORWARD`) within **tables** (`filter`, `nat`, `mangle`).

```bash
# List all rules with verbose detail
sudo iptables -L -v -n --line-numbers

# Allow established connections (essential — do this first!)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Rate limit SSH (6 connections per minute)
sudo iptables -A INPUT -p tcp --dport 22 -m limit --limit 6/min -j ACCEPT

# Drop everything else on INPUT
sudo iptables -P INPUT DROP

# NAT — masquerade traffic from a private network (basic router config)
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE

# Save rules (Ubuntu / Debian)
sudo iptables-save > /etc/iptables/rules.v4

# Restore rules on boot (add to /etc/rc.local or use iptables-persistent)
sudo iptables-restore < /etc/iptables/rules.v4
```

### nftables — The Modern Replacement

`nftables` replaces `iptables` on modern Linux systems (RHEL 8+, Debian 10+).

```bash
# List all rules
sudo nft list ruleset

# A basic nftables config (/etc/nftables.conf)
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept
        iif lo accept
        tcp dport 22 accept
        tcp dport { 80, 443 } accept
    }
    chain forward {
        type filter hook forward priority 0; policy drop;
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
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
| **Random** | Random server selection | Simple, low-overhead distribution |
| **Health-Check Based** | Automatically remove unhealthy servers | Production high-availability |

### Layer 4 vs Layer 7 Load Balancing

| Type | Layer | Sees | Examples | Speed |
|---|---|---|---|---|
| **Layer 4** | Transport (TCP/UDP) | IP + port only | AWS NLB, HAProxy TCP mode | Fastest — no HTTP parsing |
| **Layer 7** | Application (HTTP) | Full request — URL, headers, cookies, body | AWS ALB, Nginx, HAProxy HTTP mode, Traefik | Flexible, feature-rich |

**When to use Layer 4**: ultra-low latency, non-HTTP protocols (MySQL, gRPC, Redis), TLS passthrough.  
**When to use Layer 7**: path-based routing, header manipulation, A/B testing, authentication, canary deployments.

### Nginx as Load Balancer

```nginx
upstream backend {
    least_conn;                        # Least connections algorithm
    server web01.example.com:8080 weight=3;  # Higher weight = more traffic
    server web02.example.com:8080 weight=1;
    server web03.example.com:8080 backup;    # Only used when others are down
    keepalive 32;                      # Reuse upstream connections
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

### HAProxy as Load Balancer

HAProxy is the gold standard for high-performance Layer 4 and Layer 7 load balancing. It is battle-tested at enormous scale (GitHub, Reddit, Stack Overflow all use it).

**Basic HAProxy configuration (`/etc/haproxy/haproxy.cfg`):**

```haproxy
#--------------------------------------------------------------------
# Global settings
#--------------------------------------------------------------------
global
    log /dev/log local0 info
    maxconn 50000
    user haproxy
    group haproxy
    daemon
    stats socket /run/haproxy/admin.sock mode 660 level admin

#--------------------------------------------------------------------
# Default settings (inherited by all frontends/backends)
#--------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    option  http-server-close
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    timeout http-request 10s

#--------------------------------------------------------------------
# Frontend — where HAProxy listens
#--------------------------------------------------------------------
frontend http_in
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem

    # Redirect HTTP → HTTPS
    http-request redirect scheme https unless { ssl_fc }

    # Route based on Host header
    acl is_api   hdr(host) -i api.example.com
    acl is_admin hdr(host) -i admin.example.com

    use_backend api_servers   if is_api
    use_backend admin_servers if is_admin
    default_backend web_servers

#--------------------------------------------------------------------
# Backends — where traffic goes
#--------------------------------------------------------------------
backend web_servers
    balance leastconn
    option httpchk GET /health HTTP/1.1\r\nHost:\ example.com
    http-check expect status 200

    server web01 10.0.1.10:8080 check inter 5s fall 3 rise 2 weight 1
    server web02 10.0.1.11:8080 check inter 5s fall 3 rise 2 weight 1
    server web03 10.0.1.12:8080 check inter 5s fall 3 rise 2 weight 1 backup

backend api_servers
    balance roundrobin
    option httpchk GET /api/health HTTP/1.1\r\nHost:\ api.example.com
    server api01 10.0.2.10:3000 check
    server api02 10.0.2.11:3000 check

backend admin_servers
    balance source
    server admin01 10.0.3.10:4000 check

#--------------------------------------------------------------------
# Statistics dashboard
#--------------------------------------------------------------------
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:strongpassword
    stats show-legends
    stats show-node
```

### HAProxy Health Checks

```haproxy
backend web_servers
    # HTTP health check — checks a specific endpoint
    option httpchk GET /health HTTP/1.1\r\nHost:\ example.com
    http-check expect status 200

    # TCP health check (default) — just tests the connection opens
    # option tcp-check

    # Health check parameters per server:
    # check           = enable health checking
    # inter 5s        = check every 5 seconds
    # fall 3          = mark down after 3 consecutive failures
    # rise 2          = mark up after 2 consecutive successes
    server web01 10.0.1.10:8080 check inter 5s fall 3 rise 2

    # Slow start — ramp up traffic to a freshly recovered server
    server web02 10.0.1.11:8080 check inter 5s fall 3 rise 2 slowstart 60s
```

### Sticky Sessions (Session Persistence)

When users must always reach the same backend (e.g., server-side sessions), use sticky sessions:

```haproxy
backend web_servers
    balance roundrobin

    # Cookie-based stickiness — HAProxy sets a cookie
    cookie SERVERID insert indirect nocache

    server web01 10.0.1.10:8080 check cookie web01
    server web02 10.0.1.11:8080 check cookie web02
    server web03 10.0.1.12:8080 check cookie web03
```

> **Best practice**: Prefer stateless applications and external session stores (Redis) over sticky sessions. Sticky sessions make deployments and scaling harder. Reserve them for legacy apps that cannot be made stateless.

### Connection Draining (Graceful Server Removal)

When removing a server from the pool (deploy, maintenance), drain it first to avoid dropping active connections:

```bash
# Using the HAProxy runtime API (admin socket)
echo "set server web_servers/web01 state drain" | \
    sudo socat stdio /run/haproxy/admin.sock

# Check the current server state
echo "show servers state web_servers" | \
    sudo socat stdio /run/haproxy/admin.sock

# After sessions drain to zero, take it out of rotation
echo "set server web_servers/web01 state maint" | \
    sudo socat stdio /run/haproxy/admin.sock
```

The `drain` state means: accept no new sessions, but finish active ones. The `maint` state means: offline completely.

### Connection Limits

```haproxy
global
    maxconn 50000          # Total connections across all frontends

frontend http_in
    bind *:80
    maxconn 20000          # Max connections on this frontend

backend web_servers
    # Limit per-server concurrent connections
    server web01 10.0.1.10:8080 check maxconn 500
    server web02 10.0.1.11:8080 check maxconn 500
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Reverse Proxies

A **reverse proxy** sits in front of your application servers. Clients talk to the reverse proxy — they never connect directly to your backends.

**What a reverse proxy gives you:**
- TLS termination in one place
- Load balancing across multiple backends
- Centralized authentication and access control
- Caching, compression, response modification
- Path-based and host-based routing
- Observability (access logs, metrics)

### Nginx as Reverse Proxy

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";  # WebSocket support
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}
```

---

### Traefik: The Cloud-Native Reverse Proxy

Traefik is a modern reverse proxy and load balancer built for dynamic cloud environments. Unlike Nginx (where you write static config files), Traefik **auto-discovers** services from Docker, Kubernetes, Consul, and more — your configuration is embedded in your containers and manifests.

**Traefik v3 is the current version.** All examples below use v3 syntax.

#### What is Traefik & How It Differs from Nginx

| Feature | Traefik | Nginx |
|---|---|---|
| Configuration | Dynamic — auto-discovers from Docker/K8s labels | Static files — requires reload on change |
| Let's Encrypt | Built-in ACME client, zero config | Requires certbot + cron |
| Dashboard | Built-in web UI showing all routes | Not included |
| Docker integration | Native — reads container labels | Manual config per container |
| Kubernetes | Native IngressRoute CRD | Requires ingress-nginx controller |
| Learning curve | Concepts are different (EntryPoints/Routers) | Config is familiar to sysadmins |
| Performance | Excellent | Fastest (C-based, battle-tested) |
| Middleware | Rich middleware chain built-in | Requires modules or Lua |
| WebSockets | Automatic | Manual `Upgrade` headers |
| Use case | Microservices, containers, K8s | High-traffic web serving, fine-grained config |

#### The Four Core Concepts

Every request through Traefik follows this path:

```
Client Request
      │
      ▼
┌─────────────┐
│ EntryPoint  │  ← Where Traefik listens (port 80, 443)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Router    │  ← Rules: which requests go where? (Host, Path, Headers)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Middleware  │  ← Transformations: auth, redirect, headers, rate-limit
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Service   │  ← The actual backend (load balancer + servers)
└─────────────┘
```

1. **EntryPoint** — A network port Traefik listens on (`web` = port 80, `websecure` = port 443)
2. **Router** — Rules that match incoming requests (by Host, Path, Method, Header) and route them to a service
3. **Middleware** — Request/response transformations applied to matching traffic (redirects, auth, headers)
4. **Service** — The actual backend servers, with load balancing configuration

#### Static vs Dynamic Configuration

Traefik has two config layers:

- **Static config** (`traefik.yml`): Entrypoints, providers, certificate resolvers, logging. Set once at startup.
- **Dynamic config**: Routes, middlewares, services. Updated at runtime from providers (Docker labels, K8s CRDs, file).

**Full `traefik.yml` example:**

```yaml
# /etc/traefik/traefik.yml  (or mounted at /etc/traefik/traefik.yml in container)

# --- Entrypoints ---
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt

# --- Providers ---
providers:
  docker:
    exposedByDefault: false   # Opt-in: containers need label traefik.enable=true
    network: traefik-public   # Only route on this Docker network
  file:
    directory: /etc/traefik/dynamic/  # Load dynamic config from files too
    watch: true

# --- Certificate Resolvers (Let's Encrypt) ---
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme/acme.json
      httpChallenge:
        entryPoint: web        # Use HTTP-01 challenge

# --- API & Dashboard ---
api:
  dashboard: true
  insecure: false              # Must be protected by a router + middleware

# --- Logging ---
log:
  level: INFO
  filePath: /var/log/traefik/traefik.log

accessLog:
  filePath: /var/log/traefik/access.log
  bufferingSize: 100

# --- Metrics ---
metrics:
  prometheus: {}
```

#### Traefik with Docker

The most common Traefik deployment: run Traefik as a container, configure your app containers with labels.

**`docker-compose.yml` — Traefik + two apps:**

```yaml
version: "3.8"

networks:
  traefik-public:
    external: true   # Create once: docker network create traefik-public

services:
  traefik:
    image: traefik:v3
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Reads Docker events
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./acme:/acme                                  # Persists Let's Encrypt certs
      - ./logs:/var/log/traefik
    networks:
      - traefik-public
    labels:
      # Enable Traefik on itself (for the dashboard)
      - "traefik.enable=true"
      # Dashboard router — HTTPS only
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
      # Basic auth middleware for dashboard
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$..."
      # (generate password: htpasswd -nb admin password | sed 's/\$/\$\$/g')

  webapp:
    image: nginx:alpine
    restart: unless-stopped
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      # HTTPS router
      - "traefik.http.routers.webapp.rule=Host(`app.example.com`)"
      - "traefik.http.routers.webapp.entrypoints=websecure"
      - "traefik.http.routers.webapp.middlewares=security-headers"
      # Service (what port does the container expose?)
      - "traefik.http.services.webapp.loadbalancer.server.port=80"
      # Security headers middleware
      - "traefik.http.middlewares.security-headers.headers.stsSeconds=31536000"
      - "traefik.http.middlewares.security-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.security-headers.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.security-headers.headers.frameDeny=true"

  api:
    image: myapp/api:latest
    restart: unless-stopped
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
      # Sticky sessions (if needed)
      - "traefik.http.services.api.loadbalancer.sticky.cookie=true"
      - "traefik.http.services.api.loadbalancer.sticky.cookie.name=lb_session"
```

**Create the external network first:**

```bash
docker network create traefik-public
```

**Generate a bcrypt password for the dashboard:**

```bash
# Install apache2-utils if needed
sudo apt install -y apache2-utils
htpasswd -nb admin mypassword | sed 's/\$/\$\$/g'
# Double-escaped $ is required in Docker labels
```

#### Traefik with Kubernetes

Traefik supports two approaches in Kubernetes:

**1. Standard `Ingress` resource** (compatible with any ingress controller):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp
                port:
                  number: 80
```

**2. Native `IngressRoute` CRD** (Traefik-specific, more powerful):

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: webapp
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: webapp
          port: 80
      middlewares:
        - name: security-headers
  tls:
    certResolver: letsencrypt
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: default
spec:
  headers:
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    contentTypeNosniff: true
    frameDeny: true
    customResponseHeaders:
      X-Powered-By: ""         # Remove this header
---
# Path-based routing — API on /api, frontend on /
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: split-route
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api-service
          port: 3000
    - match: Host(`example.com`)
      kind: Rule
      services:
        - name: frontend-service
          port: 80
  tls:
    certResolver: letsencrypt
```

#### Automatic TLS with Let's Encrypt

Traefik has a built-in ACME client. Two challenge types:

| Challenge | How it Works | Requirement |
|---|---|---|
| **HTTP-01** | Let's Encrypt makes an HTTP request to `/.well-known/acme-challenge/TOKEN` | Port 80 must be reachable from internet |
| **DNS-01** | Let's Encrypt checks for a TXT record in your DNS | DNS provider API access; works for wildcard certs |

**HTTP-01 challenge config (already in `traefik.yml` above):**

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme/acme.json
      httpChallenge:
        entryPoint: web
```

**DNS-01 challenge (wildcard certs, Cloudflare example):**

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

```bash
# Environment variable required for DNS provider
CF_DNS_API_TOKEN=your_cloudflare_api_token
```

**Staging vs production:**

```yaml
# Use staging first to avoid rate limits while testing!
certificatesResolvers:
  letsencrypt-staging:
    acme:
      email: admin@example.com
      storage: /acme/acme-staging.json
      caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
      httpChallenge:
        entryPoint: web
```

> ⚠️ **Rate limits**: Let's Encrypt production limits you to 5 duplicate certificates per week. Always test with staging first. Staging certificates are not browser-trusted but are structurally valid.

**`acme.json` — protect this file:**

```bash
touch /acme/acme.json
chmod 600 /acme/acme.json   # Must be 600 — Traefik will refuse to start otherwise
```

#### The Dashboard — Enabling It Securely

The Traefik dashboard shows all routers, services, middlewares, and providers in real time. Never expose it without authentication.

```yaml
# In traefik.yml
api:
  dashboard: true
  insecure: false   # Never true in production
```

```yaml
# Docker labels to protect the dashboard
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
  - "traefik.http.routers.dashboard.entrypoints=websecure"
  - "traefik.http.routers.dashboard.service=api@internal"
  - "traefik.http.routers.dashboard.middlewares=dashboard-auth@docker"
  - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$ruca84Hq$$mbjdMZBAG.KWn7dvfz/8D0"
```

#### Middleware Deep-Dive: Security Headers

```yaml
# In a dynamic config file or as Docker labels
http:
  middlewares:
    security-headers:
      headers:
        stsSeconds: 31536000                # HSTS: 1 year
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true            # X-Content-Type-Options: nosniff
        frameDeny: true                     # X-Frame-Options: DENY
        browserXssFilter: true             # X-XSS-Protection
        referrerPolicy: "strict-origin-when-cross-origin"
        contentSecurityPolicy: "default-src 'self'; script-src 'self'"
        customResponseHeaders:
          X-Powered-By: ""                 # Remove server fingerprinting headers
          Server: ""
        customRequestHeaders:
          X-Real-IP: ""                    # Let Traefik set this (don't trust client)
```

#### ForwardAuth — Outsourcing Authentication

Traefik can delegate authentication to an external service. Every request is forwarded to an auth server first; if it returns 200, the request proceeds; otherwise, the client gets a 401/403.

```yaml
# Middleware definition
http:
  middlewares:
    forward-auth:
      forwardAuth:
        address: "http://auth-service:4181/verify"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-Auth-User"      # Pass user info to backend
          - "X-Auth-Email"
```

This pattern is used with tools like [Authelia](https://www.authelia.com/), [Vouch Proxy](https://github.com/vouch/vouch-proxy), or a custom auth service.

#### TCP and UDP Routing

Traefik can route non-HTTP protocols using SNI (for TLS) or port-based rules.

```yaml
# Route MySQL (TLS) based on SNI
tcp:
  routers:
    mysql-router:
      entryPoints:
        - mysql-entrypoint
      rule: "HostSNI(`db.example.com`)"
      service: mysql-service
      tls:
        passthrough: true   # Don't terminate TLS — pass it to the backend

  services:
    mysql-service:
      loadBalancer:
        servers:
          - address: "10.0.1.10:3306"
          - address: "10.0.1.11:3306"
```

```yaml
# entryPoints in traefik.yml for non-standard ports
entryPoints:
  mysql-entrypoint:
    address: ":3306"
```

#### Traefik Observability

Traefik emits rich observability data out of the box:

**Access logs:**

```yaml
accessLog:
  filePath: "/var/log/traefik/access.log"
  format: json         # json or common
  bufferingSize: 100
  fields:
    defaultMode: keep
    headers:
      defaultMode: drop
      names:
        User-Agent: keep
        Authorization: redact   # Redact sensitive headers
```

**Prometheus metrics** (scraped at `/metrics`):

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0
```

**OpenTelemetry tracing:**

```yaml
tracing:
  otlp:
    grpc:
      endpoint: "otel-collector:4317"
      insecure: true
```

#### Traefik vs Nginx: When to Use Which

| Decision Criterion | Use Traefik | Use Nginx |
|---|---|---|
| Dynamic container/microservice environment | ✅ Auto-discovery | ❌ Manual config |
| Simple static site or few services | ❌ Overkill | ✅ Simple and fast |
| Kubernetes as the platform | ✅ Native IngressRoute CRD | ✅ ingress-nginx works well too |
| Let's Encrypt auto-renewal | ✅ Built-in | ❌ Needs certbot + cron |
| Need Lua scripting or OpenResty | ❌ Not supported | ✅ Native |
| Very high traffic (100k+ RPS static) | ⚠️ Benchmark first | ✅ Consistently excellent |
| Per-request auth/middleware chains | ✅ Native middleware | ⚠️ Needs auth_request module |
| Team knows nginx config syntax | ⚠️ Different mental model | ✅ Familiar |
| Need TCP/UDP routing | ✅ Built-in | ❌ Limited |

> **Rule of thumb**: If you're running containers or Kubernetes and want zero-config TLS + routing, use Traefik. If you're running a few well-defined services on VMs and need maximum performance or fine-grained control, use Nginx.

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
ip link show                        # Show interface status (up/down)
ip route show                       # Show routing table
ip route get 8.8.8.8               # Show exact route to a destination
ethtool eth0                        # Show NIC details, speed, duplex

# Connectivity
ping -c 4 8.8.8.8                  # Test IP connectivity
traceroute 8.8.8.8                 # Trace route (UDP by default)
traceroute -T -p 443 example.com   # TCP traceroute on port 443
mtr --report 8.8.8.8              # Combined ping + traceroute

# DNS debugging
dig +trace example.com             # Full DNS resolution trace
dig example.com @1.1.1.1          # Query Cloudflare DNS directly
dig example.com @8.8.8.8          # Query Google DNS
resolvectl status                  # Show systemd-resolved status (modern distros)

# Port and service testing
nc -zv example.com 443             # Test TCP port (verbose)
nc -zv -w 3 example.com 443       # Test with 3s timeout
ss -tulnp                          # Show all listening services
ss -s                              # Socket statistics summary
ss -tp                             # Show established TCP connections

# Traffic capture
sudo tcpdump -i eth0 port 80       # Capture HTTP traffic
sudo tcpdump -i eth0 host 1.2.3.4  # Capture traffic to/from an IP
sudo tcpdump -i eth0 -w capture.pcap  # Save to file for Wireshark
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # SYN/FIN only
```

### Interpreting Common Failures

| Symptom | Likely Cause | First Check |
|---|---|---|
| `Connection refused` | Service not running on that port | `ss -tulnp \| grep PORT` |
| `Connection timed out` | Firewall dropping packets | `iptables -L`, check security groups |
| `Name resolution failed` | DNS issue | `dig hostname`, check `/etc/resolv.conf` |
| `502 Bad Gateway` | Proxy can't reach backend | Backend running? Correct port? |
| `503 Service Unavailable` | All backends failed health checks | Check backend health endpoint directly |
| High latency to one server | Routing issue or overloaded hop | `mtr hostname` — look for packet loss |

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
sudo ip netns exec myns ip addr show   # Run a single command in ns

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

# Enable IP forwarding (for routing between namespaces)
sudo sysctl -w net.ipv4.ip_forward=1
```

> **DevOps tip**: This is exactly how Docker and Podman create network isolation between containers. Each container is a network namespace connected to a bridge (`docker0` or `podman0`) via a veth pair.

### Bridge Networking

```bash
# Create a Linux bridge (software switch)
sudo ip link add br0 type bridge
sudo ip link set br0 up

# Attach a veth to the bridge
sudo ip link set veth0 master br0

# Show bridge info
bridge link show
bridge fdb show            # Forwarding database (MAC table)
```

### Overlay Networks (VXLAN)

In multi-host container environments (Docker Swarm, Kubernetes), overlay networks encapsulate container traffic inside UDP tunnels.

```
Host A (10.0.1.1)                Host B (10.0.1.2)
┌──────────────┐                ┌──────────────┐
│ Container    │                │ Container    │
│ 10.200.0.2   │                │ 10.200.0.3   │
│     │        │                │     │        │
│  veth pair   │                │  veth pair   │
│     │        │                │     │        │
│   bridge     │                │   bridge     │
│     │        │                │     │        │
│   VXLAN      │ ─── UDP 4789 ─▶│   VXLAN      │
│  (vtep)      │                │  (vtep)      │
└──────────────┘                └──────────────┘
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: DNS at Scale

### Split-Horizon DNS

Split-horizon (also called split-brain) DNS returns **different answers** to the same query depending on who is asking. Internal clients get private IPs; external clients get public IPs.

**Use case**: You have `app.example.com`. Internally it should resolve to `10.0.1.50` (your private LAN). Externally it should resolve to `203.0.113.1` (your public IP).

```
Internal client → Internal nameserver → app.example.com → 10.0.1.50
External client → Public nameserver  → app.example.com → 203.0.113.1
```

**Implementation with BIND/named:**

```
# /etc/named.conf
acl "internal" { 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16; };

view "internal" {
    match-clients { "internal"; };
    zone "example.com" {
        type master;
        file "/etc/named/zones/example.com.internal";
    };
};

view "external" {
    match-clients { any; };
    zone "example.com" {
        type master;
        file "/etc/named/zones/example.com.external";
    };
};
```

### DNS-Based Load Balancing

Return multiple A records for a hostname — clients pick one randomly. Simple and effective for basic distribution across CDN nodes or geographic clusters.

```
app.example.com.  60  IN  A  203.0.113.1
app.example.com.  60  IN  A  203.0.113.2
app.example.com.  60  IN  A  203.0.113.3
```

**Limitations**:
- No health checks — if one server dies, DNS still returns it
- TTL prevents instant failover
- Client DNS caching can break distribution

**Better alternative**: Use GeoDNS (Cloudflare, Route53 Latency Routing) to direct users to the nearest region, then use a proper load balancer within each region.

### TTL Strategy

| TTL Value | Use Case |
|---|---|
| `30–60s` | Frequently changing services, canary deployments, migrations |
| `300s (5 min)` | Normal production services |
| `3600s (1 hr)` | Stable services you rarely change |
| `86400s (24 hr)` | Static infrastructure (mail servers, NS records) |

> **Migration tip**: Before changing a DNS record, lower the TTL to 60 seconds several hours in advance. After the migration, raise it back. This minimizes the window where stale records are cached.

### Route 53 Routing Policies

AWS Route 53 supports advanced DNS routing patterns:

| Policy | What it Does | Use Case |
|---|---|---|
| **Simple** | Returns all records | Basic setups |
| **Failover** | Active/passive — switch on health check failure | DR failover |
| **Geolocation** | Route by user's country/continent | Compliance, content regionalization |
| **Latency-based** | Route to lowest-latency region | Global performance |
| **Weighted** | Split traffic by percentage | Canary releases at DNS level |
| **Multi-value** | Return multiple healthy records | Basic load distribution |

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Service Mesh Introduction

### What is a Service Mesh?

A **service mesh** is an infrastructure layer that handles service-to-service communication within a cluster. Instead of every application implementing retries, timeouts, circuit breaking, and mTLS itself, the mesh handles all of it transparently via sidecar proxies.

```
Without service mesh:
  App A ──────────────────────────────▶ App B
  (each app manages retries, timeouts, TLS, tracing)

With service mesh:
  App A ──▶ [Sidecar Proxy] ──────────▶ [Sidecar Proxy] ──▶ App B
             (Envoy/Linkerd)              (Envoy/Linkerd)
             handles everything           handles everything
```

### What a Service Mesh Gives You

| Feature | Description |
|---|---|
| **mTLS everywhere** | All service-to-service traffic encrypted and mutually authenticated |
| **Traffic management** | Canary, A/B testing, circuit breaking, retries, timeouts via config |
| **Observability** | Automatic distributed tracing, metrics, and traffic topology maps |
| **Zero-trust security** | Policies define which services can talk to which |
| **Load balancing** | L7 load balancing with circuit breaking (not just round-robin) |

### Major Service Mesh Options

| Mesh | Proxy | Complexity | Best For |
|---|---|---|---|
| **Istio** | Envoy | High | Large orgs, full feature set, Kubernetes |
| **Linkerd** | Linkerd-proxy (Rust) | Low | Simplicity, CNCF graduated, K8s |
| **Consul Connect** | Envoy | Medium | Multi-platform (VMs + K8s), HashiCorp ecosystem |
| **Cilium (eBPF mesh)** | eBPF (no sidecar) | Medium | High performance, no sidecar overhead |

### When to Use a Service Mesh

**Use a service mesh when:**
- You have 10+ microservices communicating with each other
- You need end-to-end encryption (mTLS) without changing application code
- You need fine-grained traffic control for canary deployments
- Distributed tracing and topology maps are required

**Don't use a service mesh when:**
- You have a monolith or a small number of services
- Your team is small and the operational overhead isn't justified
- You're still figuring out basic Kubernetes operations

> **Rule of thumb**: A service mesh adds significant operational complexity. Get your basic Kubernetes workloads stable first. Reach for a mesh when you're solving concrete problems (mTLS enforcement, canary rollouts, circuit breaking), not pre-emptively.

### Istio Quick Concepts

```yaml
# Traffic splitting — send 90% to v1, 10% to v2 (canary)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v2
          weight: 10
---
# Circuit breaker
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

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
| `iptables` / `nftables` | Advanced firewall rules |
| `openssl s_client` | TLS certificate inspection |
| `haproxy` | High-performance Layer 4/7 load balancer |
| `traefik` | Cloud-native reverse proxy with auto-discovery |
| `socat` | HAProxy runtime API, advanced TCP piping |
| `resolvectl` | systemd-resolved DNS status and flush |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 3.1 — DNS Investigation

1. Use `dig example.com` to query A records
2. Use `dig example.com MX` to query mail records
3. Use `dig +trace example.com` to see the full resolution path from root servers
4. Compare results from `dig @8.8.8.8 example.com` vs `dig @1.1.1.1 example.com`
5. Edit `/etc/hosts` to override `example.com` to `127.0.0.1` and test with `curl`
6. Remove the override and confirm normal resolution resumes
7. Lower a test domain's TTL and observe how quickly changes propagate

### Lab 3.2 — Port Scanning & Services

1. Run `ss -tulnp` and identify every service listening on your machine
2. Use `nc -zv localhost 22` to confirm SSH is running
3. Start a simple HTTP server: `python3 -m http.server 8080 &`
4. Confirm it appears in `ss -tulnp`
5. Test it: `curl http://localhost:8080`
6. Capture the HTTP request with `sudo tcpdump -i lo port 8080 -A`
7. Stop the server with `kill $(lsof -ti:8080)`

### Lab 3.3 — Firewall Rules

1. Install and check `ufw` status
2. Set default policy to deny incoming: `sudo ufw default deny incoming`
3. Allow SSH: `sudo ufw allow 22/tcp`
4. Allow HTTP: `sudo ufw allow 80/tcp`
5. Enable the firewall and verify: `sudo ufw status verbose`
6. Attempt to connect on a blocked port and observe the behavior
7. Add a rate limit: `sudo ufw limit 22/tcp`

### Lab 3.4 — HTTP Deep Dive

1. Use `curl -v https://example.com` to see the full TLS handshake and HTTP headers
2. Check the TLS certificate expiry date with `openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates`
3. Make a POST request to `https://httpbin.org/post` with a JSON body
4. Use `curl -w "%{http_code} %{time_namelookup}s %{time_connect}s %{time_total}s\n" -o /dev/null https://example.com`

### Lab 3.5 — HAProxy Load Balancer

1. Install HAProxy: `sudo apt install -y haproxy`
2. Start three backend servers on different ports:
   ```bash
   python3 -m http.server 8081 &
   python3 -m http.server 8082 &
   python3 -m http.server 8083 &
   ```
3. Write a basic HAProxy config balancing across the three backends
4. Enable the stats page and observe request distribution
5. Kill one backend and observe HAProxy's health check remove it
6. Restore the backend and watch it re-enter rotation

### Lab 3.6 — Traefik with Docker

1. Create a Docker network: `docker network create traefik-public`
2. Write a `traefik.yml` with Docker provider and HTTP entrypoint
3. Launch Traefik: `docker compose up -d traefik`
4. Launch a test container (e.g., `traefik/whoami`) with Traefik labels
5. Access it via the hostname configured in the labels
6. Enable the dashboard and explore the route/service/middleware views
7. Add a second container and observe Traefik auto-discover it

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Computer Networking: A Top-Down Approach](https://gaia.cs.umass.edu/kurose_ross/) — Kurose & Ross
- [HAProxy Documentation](https://www.haproxy.org/download/2.8/doc/configuration.txt)
- [Traefik v3 Documentation](https://doc.traefik.io/traefik/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
- [DNS in Detail (TryHackMe)](https://tryhackme.com/room/dnsindetail)
- [The Service Mesh — CNCF](https://www.cncf.io/blog/2017/04/26/service-mesh-critical-component-cloud-native-stack/)
- [Istio in Action](https://www.manning.com/books/istio-in-action) — Manning
- [Glossary: DNS](./glossary.md#d), [Firewall](./glossary.md#f), [Load Balancer](./glossary.md#l), [TCP/IP](./glossary.md#t)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
