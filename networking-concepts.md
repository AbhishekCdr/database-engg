# Computer Networking — A Comprehensive Guide

> A from-the-ground-up reference covering the models, protocols, devices, security, and tooling that make modern networks work. Every section pairs concepts with concrete examples you can run or reason about.

---

## Table of Contents

1. [What Is a Network?](#1-what-is-a-network)
2. [Network Types & Topologies](#2-network-types--topologies)
3. [The OSI Model](#3-the-osi-model)
4. [The TCP/IP Model](#4-the-tcpip-model)
5. [Physical Layer](#5-physical-layer)
6. [Data Link Layer](#6-data-link-layer)
7. [Network Layer — IP, Routing, NAT](#7-network-layer--ip-routing-nat)
8. [Subnetting & CIDR](#8-subnetting--cidr)
9. [Transport Layer — TCP & UDP](#9-transport-layer--tcp--udp)
10. [Application Layer Protocols](#10-application-layer-protocols)
11. [DNS — The Internet's Phonebook](#11-dns--the-internets-phonebook)
12. [HTTP & HTTPS Deep Dive](#12-http--https-deep-dive)
13. [TLS / SSL & PKI](#13-tls--ssl--pki)
14. [Network Devices](#14-network-devices)
15. [Wireless Networking](#15-wireless-networking)
16. [Network Security](#16-network-security)
17. [VPNs & Tunneling](#17-vpns--tunneling)
18. [Load Balancing, Proxies & CDNs](#18-load-balancing-proxies--cdns)
19. [Modern Application Protocols (REST, gRPC, GraphQL, WebSockets)](#19-modern-application-protocols)
20. [Cloud & Virtual Networking](#20-cloud--virtual-networking)
21. [Performance — Latency, Throughput, Jitter](#21-performance--latency-throughput-jitter)
22. [Network Troubleshooting Tools](#22-network-troubleshooting-tools)
23. [Common Real-World Scenarios](#23-common-real-world-scenarios)
24. [Glossary & Quick Reference](#24-glossary--quick-reference)
25. [End-to-End Request Flow: Client → ALB → Kubernetes → Pod → DB/VM](#25-end-to-end-request-flow-client--alb--kubernetes--pod--dbvm)

---

## 1. What Is a Network?

A **computer network** is a set of interconnected devices ("nodes") that exchange data over a shared medium using agreed-upon rules ("protocols"). The simplest network is two laptops connected by an Ethernet cable; the largest is the **Internet** — a global network of networks.

Three foundational ideas that underpin everything else:

1. **Addressing** — every node needs a unique identifier so others can reach it (MAC, IP, port, URL).
2. **Protocols** — formal rules covering message format, ordering, error handling, and timing (TCP, HTTP, DNS…).
3. **Layering** — networks are built as stacks of layers, each providing services to the layer above and using services from the one below. This decoupling is why you can change Wi-Fi to Ethernet without rewriting your browser.

---

## 2. Network Types & Topologies

### By scale
| Type | Range | Example |
|---|---|---|
| **PAN** Personal Area Network | < 10 m | Bluetooth headphones, AirDrop |
| **LAN** Local Area Network | Single building | Office Wi-Fi, home router |
| **CAN** Campus Area Network | Cluster of buildings | University campus |
| **MAN** Metropolitan Area Network | A city | City-wide ISP backbone |
| **WAN** Wide Area Network | Country / globe | The Internet, MPLS leased lines |
| **SAN** Storage Area Network | Datacenter | Fibre Channel storage fabric |

### By topology

```
Bus           Star               Ring              Mesh
A-B-C-D       A   B            A--B               A---B
              \ / \           /    \             |\ /|
               H   D         D      C            | X |
              / \ /           \    /             |/ \|
              C   E            \--/              C---D
```

- **Bus** — single backbone, simple but a single break kills the network. (Old coax Ethernet.)
- **Star** — every node connects to a central switch. The dominant LAN topology today.
- **Ring** — packets pass node-to-node. Used by Token Ring (legacy) and SONET/SDH backbones.
- **Mesh** — every node connected to many others; resilient but expensive. Used in the Internet's core and by Wi-Fi mesh routers.
- **Tree / Hybrid** — combinations; campus networks are typically tree-shaped.

---

## 3. The OSI Model

A 7-layer reference model from ISO. Mostly conceptual today — TCP/IP is the model in practice — but OSI's vocabulary is universal.

| # | Layer | Job | PDU | Examples |
|---|---|---|---|---|
| 7 | **Application** | Network services to apps | Data | HTTP, DNS, SMTP, FTP, SSH |
| 6 | **Presentation** | Encoding, compression, encryption | Data | TLS, JPEG, MIME, ASCII/UTF-8 |
| 5 | **Session** | Open / close / manage sessions | Data | NetBIOS, RPC, SOCKS |
| 4 | **Transport** | End-to-end delivery, reliability | Segment / Datagram | TCP, UDP, QUIC |
| 3 | **Network** | Routing across networks | Packet | IP, ICMP, OSPF, BGP |
| 2 | **Data Link** | Node-to-node frames on one link | Frame | Ethernet, Wi-Fi (802.11), PPP, ARP |
| 1 | **Physical** | Bits on a wire/fiber/radio | Bit | Cat6 cable, fiber, radio waves, RJ-45 |

**Mnemonic (top-down):** *All People Seem To Need Data Processing.*
**Mnemonic (bottom-up):** *Please Do Not Throw Sausage Pizza Away.*

### How data flows down and back up

```
Sender                                    Receiver
  App data                                  App data
    │ + HTTP header                           ▲ strip HTTP
    ▼                                         │
  TCP segment (port + seq#)                 TCP segment
    │ + IP header                             ▲ strip TCP
    ▼                                         │
  IP packet (src/dst IP)                    IP packet
    │ + Ethernet header                       ▲ strip IP
    ▼                                         │
  Ethernet frame (MACs)                     Ethernet frame
    │                                         ▲ strip Ethernet
    ▼                                         │
  Bits on wire ──────────────────────────► Bits on wire
```

This wrapping process is called **encapsulation**; the reverse is **decapsulation**.

---

## 4. The TCP/IP Model

The 4-layer model that the actual Internet uses.

| TCP/IP Layer | OSI Equivalent | Examples |
|---|---|---|
| **Application** | 5–7 | HTTP, DNS, SMTP, SSH |
| **Transport** | 4 | TCP, UDP |
| **Internet** | 3 | IP, ICMP |
| **Network Access (Link)** | 1–2 | Ethernet, Wi-Fi |

When in doubt: **think TCP/IP for engineering, OSI for terminology.**

---

## 5. Physical Layer

The wire (or air) and the electrical/optical/radio signaling on it.

### Common media
- **Twisted pair copper** (Cat5e/6/6a/7) — RJ-45 connector, 1–10 Gbps over 100 m.
- **Coaxial** — legacy LAN, still used in cable TV/Internet (DOCSIS).
- **Fiber optic** — single-mode (long haul, lasers) and multi-mode (short haul, LEDs). 10/40/100/400 Gbps.
- **Wireless** — radio (Wi-Fi, Bluetooth, cellular), microwave, satellite.

### Concepts
- **Bandwidth** — maximum bit rate the medium supports (e.g., 1 Gbps).
- **Throughput** — bits/sec actually delivered after overhead (always ≤ bandwidth).
- **Encoding** — converting bits to signal levels (Manchester, NRZ, 4B/5B, PAM-4).
- **Full-duplex / half-duplex** — can both ends transmit simultaneously?

---

## 6. Data Link Layer

Moves frames between two devices on the **same physical network** (one Ethernet broadcast domain or one Wi-Fi cell).

### MAC address
- 48-bit hardware address, written as `00:1A:2B:3C:4D:5E`.
- First 24 bits identify the manufacturer (OUI), last 24 are device-unique.
- Burned into the NIC at the factory (can be spoofed).

### Ethernet frame layout

```
+----------+----------+--------+----------+-----+
| Dest MAC | Src MAC  | EtherT | Payload  | FCS |
| 6 bytes  | 6 bytes  | 2 B    | 46-1500B | 4B  |
+----------+----------+--------+----------+-----+
```

`EtherType` says what's in the payload (0x0800 = IPv4, 0x86DD = IPv6, 0x0806 = ARP).

### Switches vs. hubs
- **Hub** — repeats every signal out every port (a single collision domain). Effectively obsolete.
- **Switch** — learns which MAC lives behind which port and forwards only there. Each port is its own collision domain. The default LAN device.

### ARP — Address Resolution Protocol
Translates IP → MAC on the local network.

```
$ arp -a
? (192.168.1.1) at 5c:5b:35:11:22:33 on en0 ifscope [ethernet]
```

When your laptop wants to send a packet to `192.168.1.1`, it broadcasts: "Who has 192.168.1.1?" The router replies with its MAC, and your laptop caches the mapping.

### VLANs
**Virtual LAN** — logically partition one switch into multiple broadcast domains using **802.1Q tags** (4 bytes inserted into the frame, including a 12-bit VLAN ID). Lets you put HR and Engineering on the same hardware but isolate them.

---

## 7. Network Layer — IP, Routing, NAT

The layer that gets packets from any host on the Internet to any other.

### IPv4
- 32-bit address: `192.168.1.10` — four 8-bit octets.
- ~4.3 billion addresses; long since exhausted publicly.
- Header (20 bytes): version, TTL, protocol, source, destination, header checksum, etc.

#### Address classes (legacy, replaced by CIDR)
| Class | Range | Default mask |
|---|---|---|
| A | 1.0.0.0 – 126.255.255.255 | /8 |
| B | 128.0.0.0 – 191.255.255.255 | /16 |
| C | 192.0.0.0 – 223.255.255.255 | /24 |
| D | 224.0.0.0 – 239.255.255.255 | (multicast) |
| E | 240.0.0.0 – 255.255.255.255 | (reserved) |

#### Special / private ranges
| Range | Purpose |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private (RFC 1918) |
| `127.0.0.0/8` | Loopback (`localhost`) |
| `169.254.0.0/16` | Link-local (DHCP failure) |
| `224.0.0.0/4` | Multicast |
| `255.255.255.255` | Limited broadcast |

### IPv6
- 128-bit address: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`.
- Shortened: `2001:db8:85a3::8a2e:370:7334` (consecutive zero-groups compress to `::`, once).
- 3.4 × 10³⁸ addresses — enough for every grain of sand to have its own /48.
- No broadcast; uses multicast and **Neighbor Discovery Protocol (NDP)** instead of ARP.
- `::1` = loopback, `fe80::/10` = link-local, `fc00::/7` = unique local (private).

### Routing
A **router** forwards packets between networks. Each router holds a **routing table**:

```
Destination       Next Hop        Interface
192.168.1.0/24    -               eth0     (directly connected)
10.0.0.0/8        192.168.1.1     eth0
0.0.0.0/0         192.168.1.1     eth0     (default route)
```

The **longest prefix match** wins. When a packet for `8.8.8.8` arrives, the router walks the table, picks the most specific matching prefix (here `0.0.0.0/0`), and forwards out the corresponding interface.

#### Routing protocols
| Protocol | Type | Where used |
|---|---|---|
| **RIP** | Distance-vector | Tiny / legacy networks |
| **OSPF** | Link-state | Inside a single organization (interior) |
| **EIGRP** | Hybrid (Cisco) | Cisco enterprise |
| **IS-IS** | Link-state | ISP backbones |
| **BGP** | Path-vector | **Between** ISPs — runs the public Internet |

### ICMP
**Internet Control Message Protocol** — diagnostics and error messages (TTL exceeded, host unreachable). Powers `ping` and `traceroute`.

### NAT — Network Address Translation
Lets many private hosts share one public IP. Your home router runs **PAT** (Port Address Translation, "NAT overload"):

```
Internal client                Router (NAT)               Internet server
192.168.1.10:54321  ─────►  203.0.113.5:60001  ─────►   8.8.8.8:443
192.168.1.10:54321  ◄─────  203.0.113.5:60001  ◄─────   8.8.8.8:443
```

The router maintains a **NAT translation table** keyed by `(internal IP, port) ↔ (external IP, port)` so reply packets find their way home. This is why most Internet hosts can't initiate connections to your laptop without help (port forwarding, hole punching, or IPv6).

---

## 8. Subnetting & CIDR

**CIDR** — *Classless Inter-Domain Routing*. Replaces fixed classes with a `/prefix` bit count.

`192.168.1.0/24` means: the first 24 bits are the network, the last 8 are host.

### Quick mental math

| Prefix | Subnet mask | # Addresses | # Usable hosts |
|---|---|---|---|
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |

(Two addresses always reserved per subnet: network address and broadcast.)

### Worked example

You're given `10.10.0.0/22` and need 4 equal subnets:
- /22 = 1024 addresses → split into four /24s.
- Subnets: `10.10.0.0/24`, `10.10.1.0/24`, `10.10.2.0/24`, `10.10.3.0/24`.

For `10.10.1.0/24`:
- Network: `10.10.1.0`
- Broadcast: `10.10.1.255`
- Usable hosts: `10.10.1.1` – `10.10.1.254`

### VLSM
**Variable Length Subnet Masking** — different subnets at different sizes inside the same parent block, sized to actual demand instead of waste.

---

## 9. Transport Layer — TCP & UDP

The transport layer adds **ports** (so multiple apps on one host can use the network) and end-to-end semantics.

### Ports
- 16-bit number, 0–65535.
- **Well-known**: 0–1023 (HTTP 80, HTTPS 443, SSH 22, DNS 53, SMTP 25).
- **Registered**: 1024–49151.
- **Ephemeral**: 49152–65535 (assigned to client outgoing connections).

A connection is uniquely identified by the **5-tuple**: `(protocol, src IP, src port, dst IP, dst port)`.

### TCP — Transmission Control Protocol

Reliable, connection-oriented, in-order, byte-stream. Used by HTTP, SSH, SMTP, etc.

#### Three-way handshake
```
Client                         Server
  │  ── SYN  (seq=x) ───────►   │
  │  ◄── SYN-ACK (seq=y, ack=x+1) │
  │  ── ACK (ack=y+1) ──────►   │
  │                              │
  ╞═══════ established ══════════╡
```

#### Four-way close
```
Client                         Server
  │  ── FIN ─────────────────►   │
  │  ◄── ACK ─────────────────   │
  │  ◄── FIN ─────────────────   │
  │  ── ACK ──────────────────►  │
                                  (TIME_WAIT for 2×MSL)
```

#### TCP states
`LISTEN`, `SYN_SENT`, `SYN_RECEIVED`, `ESTABLISHED`, `FIN_WAIT_1/2`, `CLOSE_WAIT`, `CLOSING`, `LAST_ACK`, `TIME_WAIT`, `CLOSED`. View on Linux/macOS:

```
$ netstat -an | grep ESTABLISHED
$ ss -tan
```

#### Reliability mechanisms
- **Sequence numbers** — every byte is numbered.
- **ACKs** — receiver acknowledges what it has received.
- **Retransmission** — sender resends if no ACK within RTO (retransmission timeout).
- **Sliding window** — receiver advertises how much buffer space it has (`rwnd`).
- **Flow control** — sender never sends more than `rwnd`.
- **Congestion control** — sender slows down when it sees loss (Reno, CUBIC, BBR).
- **Checksum** — corrupted segments are dropped.

### UDP — User Datagram Protocol

Connectionless, unreliable, message-oriented. Used by DNS, DHCP, video/voice, gaming, QUIC.

```
+------+------+--------+----------+----------+
| Src  | Dst  | Length | Checksum | Payload  |
| port | port | 2 B    | 2 B      |          |
+------+------+--------+----------+----------+
```

8-byte header vs TCP's 20+. No handshake, no retries, no ordering — just "fire and hope."

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | Yes (handshake) | No |
| Reliability | Guaranteed | None |
| Ordering | In-order | Out-of-order possible |
| Speed | Slower | Faster |
| Header size | 20+ B | 8 B |
| Use cases | Web, email, SSH, file transfer | DNS, DHCP, VoIP, games, streaming |

### QUIC
Google-originated, now standardized (RFC 9000). Runs over UDP, gives you TCP-like reliability + TLS 1.3 + multiplexed streams in one handshake. **HTTP/3 runs on QUIC.**

---

## 10. Application Layer Protocols

| Protocol | Port | Transport | Purpose |
|---|---|---|---|
| HTTP | 80 | TCP | Web |
| HTTPS | 443 | TCP/QUIC | Encrypted web |
| FTP | 21 (control), 20 (data) | TCP | File transfer (legacy) |
| SFTP | 22 | TCP (over SSH) | Secure file transfer |
| SSH | 22 | TCP | Remote shell |
| Telnet | 23 | TCP | Plaintext shell (deprecated) |
| SMTP | 25 / 587 | TCP | Send mail |
| IMAP | 143 / 993 | TCP | Read mail (server-side mailbox) |
| POP3 | 110 / 995 | TCP | Read mail (download) |
| DNS | 53 | UDP (TCP for large) | Name resolution |
| DHCP | 67/68 | UDP | Auto IP config |
| NTP | 123 | UDP | Time sync |
| SNMP | 161/162 | UDP | Device management |
| LDAP | 389 / 636 | TCP | Directory service |
| RDP | 3389 | TCP | Windows remote desktop |

---

## 11. DNS — The Internet's Phonebook

Translates `www.google.com` → `142.250.190.78`.

### Hierarchy (resolved right-to-left)

```
.                       (root)
└── com.                (TLD)
    └── google.com.     (authoritative)
        └── www.google.com.
```

### Resolution flow

```
Browser ──► Stub resolver (OS) ──► Recursive resolver (8.8.8.8 / ISP)
                                         │
                       cache miss        ▼
                                  Root servers ──► .com TLD ──► google.com auth NS
                                         │
                                         ▼
                                  Returns A record: 142.250.190.78
```

### Common record types

| Type | Meaning | Example |
|---|---|---|
| **A** | IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 address | `example.com → 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Canonical alias | `www.example.com → example.com` |
| **MX** | Mail exchanger | `example.com → 10 mail.example.com` |
| **NS** | Authoritative name server | `example.com → ns1.example.com` |
| **TXT** | Free-form text | SPF, DKIM, domain verification |
| **PTR** | Reverse DNS (IP→name) | `34.216.184.93.in-addr.arpa → example.com` |
| **SOA** | Start of authority | Zone metadata, serial number |
| **SRV** | Service location | `_sip._tcp.example.com → 0 5 5060 sipserver` |
| **CAA** | Allowed certificate authorities | `example.com → 0 issue "letsencrypt.org"` |

### Tooling
```
$ dig example.com +short
$ dig example.com MX
$ dig @8.8.8.8 example.com
$ nslookup example.com
$ host example.com
```

### TTL & caching
Every record has a **TTL** (e.g., 300s). Resolvers cache for that long. Lower TTL before a planned IP change so clients pick up the new record quickly.

---

## 12. HTTP & HTTPS Deep Dive

### HTTP request anatomy
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.4.0
Accept: text/html
Cookie: session=abc123

```

### HTTP response anatomy
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1256
Cache-Control: max-age=3600
Set-Cookie: session=def456; HttpOnly

<!DOCTYPE html>...
```

### Methods
| Method | Idempotent? | Safe? | Body? | Use |
|---|---|---|---|---|
| GET | ✓ | ✓ | No | Read |
| HEAD | ✓ | ✓ | No | Read headers only |
| POST | ✗ | ✗ | Yes | Create / arbitrary |
| PUT | ✓ | ✗ | Yes | Replace |
| PATCH | ✗ (usually) | ✗ | Yes | Partial update |
| DELETE | ✓ | ✗ | Maybe | Remove |
| OPTIONS | ✓ | ✓ | No | Discover (CORS preflight) |

### Status codes
- **1xx** Informational (`100 Continue`, `101 Switching Protocols`)
- **2xx** Success (`200 OK`, `201 Created`, `204 No Content`)
- **3xx** Redirect (`301 Moved Permanently`, `302 Found`, `304 Not Modified`)
- **4xx** Client error (`400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`)
- **5xx** Server error (`500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`)

### HTTP versions

| Version | Year | Highlights |
|---|---|---|
| **HTTP/1.0** | 1996 | One request per TCP connection |
| **HTTP/1.1** | 1997 | Persistent connections, pipelining (rarely used), `Host:` header, chunked encoding |
| **HTTP/2** | 2015 | Binary framing, multiplexed streams over one TCP connection, header compression (HPACK), server push |
| **HTTP/3** | 2022 | Runs on QUIC (UDP), no head-of-line blocking, faster handshakes |

### Cookies, sessions, CORS
- **Cookies** — small key/value pairs the server stores on the client. Flags: `HttpOnly` (no JS access), `Secure` (HTTPS only), `SameSite=Strict|Lax|None` (CSRF defense).
- **Sessions** — server keeps state, client just holds a session ID cookie.
- **CORS** — *Cross-Origin Resource Sharing*. Browsers block JS from reading responses across origins unless the server sends `Access-Control-Allow-Origin`. Preflight `OPTIONS` for non-simple requests.

### Caching
- `Cache-Control: max-age=3600, public` — cache for 1 hour, anywhere.
- `ETag: "abc123"` — conditional GET; client sends `If-None-Match`, server returns `304 Not Modified`.
- `Last-Modified` / `If-Modified-Since` — older timestamp variant.

---

## 13. TLS / SSL & PKI

**TLS** (Transport Layer Security; "SSL" is the deprecated predecessor) provides:
1. **Confidentiality** — symmetric encryption (AES-GCM, ChaCha20-Poly1305).
2. **Integrity** — AEAD ciphers detect tampering.
3. **Authentication** — X.509 certificates signed by a trusted CA.

### TLS 1.3 handshake (1-RTT)

```
Client                                    Server
  │ ── ClientHello (ciphers, key_share) ►  │
  │ ◄── ServerHello (cipher, key_share, certificate, finished) │
  │ ── Finished ─────────────────────────► │
  │                                         │
  ╞══════ encrypted application data ══════╡
```

(Older TLS 1.2 needed 2 RTTs.)

### Certificates
An X.509 certificate binds a public key to a domain and is signed by a **Certificate Authority (CA)**:

```
$ openssl s_client -connect example.com:443 -servername example.com </dev/null \
    | openssl x509 -noout -subject -issuer -dates
subject= CN=*.example.com
issuer= CN=DigiCert TLS RSA SHA256 2020 CA1
notBefore=Jan  1 00:00:00 2026 GMT
notAfter=Jan  1 23:59:59 2027 GMT
```

Browsers ship with a **trust store** of root CA certificates. Servers present a chain (leaf → intermediate → root). **Let's Encrypt** issues free 90-day certs via the **ACME** protocol (`certbot`).

### mTLS
**Mutual TLS** — both client and server present certificates. Common in service-mesh / zero-trust networks.

---

## 14. Network Devices

| Device | OSI Layer | Function |
|---|---|---|
| **Repeater / Hub** | 1 | Regenerates / broadcasts signal |
| **Bridge** | 2 | Connects two LAN segments |
| **Switch** | 2 (some L3) | Forwards frames by MAC; many ports |
| **Router** | 3 | Forwards packets between networks |
| **L3 switch** | 2+3 | Switch with routing in hardware |
| **Firewall** | 3–7 | Filters traffic by rules |
| **Load balancer** | 4 or 7 | Distributes traffic across servers |
| **Proxy** | 7 | Intermediary for client requests |
| **Reverse proxy** | 7 | Intermediary fronting servers (Nginx, HAProxy) |
| **WAP** | 1–2 | Wireless Access Point |
| **Modem** | 1 | Modulates digital ↔ analog (DSL, cable) |
| **Gateway** | 3+ | Connects networks using different protocols |
| **NAS / SAN** | varies | Storage over network |

---

## 15. Wireless Networking

### Wi-Fi (IEEE 802.11)

| Standard | Marketing name | Year | Max link rate | Band |
|---|---|---|---|---|
| 802.11n | Wi-Fi 4 | 2009 | 600 Mbps | 2.4 / 5 GHz |
| 802.11ac | Wi-Fi 5 | 2014 | 6.9 Gbps | 5 GHz |
| 802.11ax | Wi-Fi 6 / 6E | 2019 / 2021 | 9.6 Gbps | 2.4 / 5 / 6 GHz |
| 802.11be | Wi-Fi 7 | 2024 | 46 Gbps | 2.4 / 5 / 6 GHz |

### Concepts
- **SSID** — network name.
- **BSSID** — AP's MAC address.
- **Channel** — frequency slice (2.4 GHz: 1, 6, 11 are non-overlapping in NA).
- **Security** — WEP (broken), WPA, WPA2 (AES-CCMP), **WPA3** (current). Use WPA3 / WPA2-PSK at minimum.
- **Roaming** — client moves between APs sharing one SSID; 802.11r/k/v speed it up.

### Cellular generations
2G (GSM, voice + SMS) → 3G (UMTS, basic data) → 4G (LTE, IP-only, ~100 Mbps) → 5G (sub-6 GHz + mmWave, gigabit, ultra-low latency for IoT/industrial).

---

## 16. Network Security

### Threat model basics
**CIA triad** — Confidentiality, Integrity, Availability. Every security control protects at least one.

### Common attacks
- **Sniffing / eavesdropping** — passive capture; defeated by encryption (TLS).
- **MITM** — attacker sits between client and server; defeated by certificate validation.
- **ARP spoofing** — poisons LAN ARP cache to become MITM.
- **DNS spoofing / cache poisoning** — defeated by **DNSSEC** and **DoH/DoT**.
- **DDoS** — distributed denial of service; mitigated by upstream scrubbing (Cloudflare, AWS Shield), rate limiting, anycast.
- **Port scanning** — `nmap -sS target`; firewalls and IDS catch it.
- **SQL injection / XSS / CSRF** — application layer attacks; not strictly "network" but often filtered by WAFs.
- **Phishing** — social; defended by DMARC/SPF/DKIM, training.
- **Replay** — defeated by nonces and timestamps.

### Defenses

#### Firewalls
- **Packet-filter** — stateless, rules on src/dst IP+port.
- **Stateful** — tracks connection state, allows replies.
- **Application-layer (NGFW / WAF)** — inspects payload (deep packet inspection).

Linux example with `nftables`:
```
$ sudo nft add rule inet filter input tcp dport 22 accept
$ sudo nft add rule inet filter input drop
```

macOS uses **PF** (`/etc/pf.conf`).

#### IDS / IPS
- **IDS** — Intrusion Detection (alerts). Snort, Suricata.
- **IPS** — Intrusion Prevention (blocks).

#### Zero Trust
"Never trust, always verify." No implicit trust based on network location; every request authenticated and authorized. Examples: Google BeyondCorp, Cloudflare Access.

#### Encryption everywhere
- TLS for transport.
- Disk encryption (LUKS, FileVault, BitLocker) for at-rest.
- Hashing (SHA-256, BLAKE2) for integrity.
- Asymmetric keys (RSA, ECDSA, Ed25519) for identity & key exchange.

---

## 17. VPNs & Tunneling

A **VPN** wraps your packets inside encrypted packets and sends them through a remote server, so the public Internet sees the VPN's IP and the local network sees only encrypted traffic.

### Common protocols
| Protocol | Notes |
|---|---|
| **IPSec** | Standardized, widely supported, works at L3. Modes: transport, tunnel. |
| **OpenVPN** | TLS-based, runs over UDP/TCP, very flexible. |
| **WireGuard** | Modern, tiny codebase, ChaCha20-Poly1305, fast. Default for new deployments. |
| **L2TP/IPSec** | Legacy, often slow. |
| **PPTP** | Insecure — don't. |

### Tunnel example (WireGuard)
```
[client wg0]  10.0.0.2  ──encrypted UDP──►  [server wg0]  10.0.0.1
                       │                                 │
                       ▼                                 ▼
                 Encapsulates:                     Decapsulates,
                 IP packets you                    forwards to
                 send become                       Internet/LAN
                 encrypted UDP
```

### SSH tunneling (poor-man's VPN for one port)
```
$ ssh -L 8080:localhost:80 user@server
# Now visiting http://localhost:8080 on your laptop hits server's port 80
```

---

## 18. Load Balancing, Proxies & CDNs

### Load balancers

| Type | Layer | Sees | Examples |
|---|---|---|---|
| **L4** | Transport | TCP/UDP only | AWS NLB, HAProxy in TCP mode |
| **L7** | Application | HTTP headers, paths, cookies | Nginx, Envoy, AWS ALB, HAProxy in HTTP mode |

### Algorithms
- **Round-robin** — simple rotation.
- **Least connections** — pick the backend with fewest open conns.
- **IP hash** — same client always hits same backend (sticky sessions).
- **Weighted** — assign capacity weights.
- **Latency-based** — pick fastest backend.

### Health checks
LB regularly probes backends (`GET /health`) and removes failures from rotation.

### Proxies
- **Forward proxy** — sits between *clients* and the Internet. Used for filtering, caching, anonymity (corporate proxy, Squid).
- **Reverse proxy** — sits between Internet and *servers*. Adds TLS termination, caching, compression, routing (Nginx, Caddy, Traefik).
- **Transparent proxy** — clients don't know it's there.

### CDN — Content Delivery Network
Geographically distributed caches close to users. Static assets are served from the nearest **edge** PoP, dramatically lowering latency. Modern CDNs (Cloudflare, Fastly, Akamai, CloudFront) also do TLS termination, DDoS scrubbing, image optimization, and **edge compute** (Workers / Lambda@Edge).

### Anycast
One IP advertised from many locations; BGP routes each user to the nearest one. Used by DNS roots (`8.8.8.8`, `1.1.1.1`) and CDNs.

---

## 19. Modern Application Protocols

### REST
HTTP + JSON convention. Resources are nouns (`/users/42`), HTTP methods are verbs.
```
GET    /users/42
POST   /users
PUT    /users/42
DELETE /users/42
```
Stateless, cacheable, self-descriptive. Loose-fit for streaming or long-running ops.

### gRPC
Google-built RPC over **HTTP/2** with **Protocol Buffers** for the schema.
- **Strong typing** via `.proto` files.
- **Multiplexed streams** — bidirectional streaming.
- Used heavily inside microservices.
- Browser support requires **gRPC-Web** + a proxy (Envoy).

### GraphQL
Single endpoint (`POST /graphql`); clients ask for exactly the fields they want.
```graphql
query {
  user(id: 42) {
    name
    posts(limit: 5) { title }
  }
}
```
Solves over-fetching/under-fetching at the cost of more complex caching.

### WebSockets
Persistent **bidirectional** TCP connection over HTTP `Upgrade`. Enables chat, live dashboards, multiplayer games, collaborative editing.
```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### Server-Sent Events (SSE)
One-way (server → client) streaming over plain HTTP. Lighter than WebSockets when you only need pushes.

### MQTT
Pub/sub for IoT; runs over TCP, tiny header, QoS levels 0/1/2.

---

## 20. Cloud & Virtual Networking

### VPC — Virtual Private Cloud
Your private slice of a public cloud, with your own IP range.

```
VPC: 10.0.0.0/16
├── Public subnet  10.0.1.0/24    (Internet Gateway attached)
│     └── EC2 web server  10.0.1.10  (also has public IP)
└── Private subnet 10.0.2.0/24    (no IGW)
      └── DB server 10.0.2.20
      └── Egress via NAT Gateway in public subnet
```

### Building blocks
- **Subnets** — slice the VPC into AZs/tiers.
- **Route tables** — per-subnet routing.
- **Internet Gateway (IGW)** — gives a VPC a path to the Internet.
- **NAT Gateway** — lets private hosts initiate outbound Internet without being reachable.
- **Security groups** — *stateful* firewalls attached to instances.
- **NACLs** — *stateless* firewalls attached to subnets (rule order matters).
- **VPC Peering / Transit Gateway** — connect VPCs to each other.
- **Direct Connect / ExpressRoute** — dedicated fiber from your datacenter.

### Service mesh
Sidecar proxies (Envoy) between every service. Provides mTLS, retries, circuit breaking, observability without app changes. Examples: Istio, Linkerd, Consul Connect.

### Container networking
- **Docker bridge** — default; container gets a private IP, NAT'd through host.
- **Host network** — container shares host's stack (no isolation).
- **Overlay (VXLAN)** — multi-host networking (Swarm, Kubernetes Calico/Flannel).
- **CNI** — Container Network Interface, the plugin spec Kubernetes uses.

---

## 21. Performance — Latency, Throughput, Jitter

### Definitions
- **Latency** — time for one packet, one way (or RTT, round-trip).
- **Throughput / Goodput** — data rate; goodput excludes retransmits/headers.
- **Bandwidth** — theoretical capacity.
- **Jitter** — variation in latency over time. Critical for VoIP/video.
- **Packet loss** — % of packets that don't arrive.

### Bandwidth-Delay Product
`BDP = bandwidth × RTT`. The amount of data "in flight" on a fully utilized link.

> 1 Gbps × 80 ms RTT = 10 MB. Your TCP window must be ≥ 10 MB or you'll cap throughput far below the link's capacity. This is why long-fat networks need window scaling.

### Where latency comes from
1. **Propagation delay** — distance ÷ speed of light in fiber (~200 000 km/s).
2. **Transmission delay** — packet size ÷ link bandwidth.
3. **Queuing delay** — time spent in router buffers.
4. **Processing delay** — header inspection, decryption.

### Common optimizations
- **Keep-alive** — reuse TCP connections.
- **HTTP/2 multiplexing** — many requests on one connection.
- **Compression** — gzip/brotli responses.
- **Caching** — at every layer (browser, CDN, app, DB).
- **CDN** — bring the content closer to users.
- **Connection pooling** — DB and HTTP clients should pool.
- **Prefetching** — fetch likely-needed resources early.

---

## 22. Network Troubleshooting Tools

### Reachability
```
$ ping 8.8.8.8                    # ICMP echo, basic up/down
$ ping -c 4 example.com           # 4 packets and exit
$ mtr example.com                 # ping + traceroute combined, live
```

### Path
```
$ traceroute example.com          # hop-by-hop path (UDP by default)
$ tracepath example.com           # like traceroute, no root needed
```

### DNS
```
$ dig +trace example.com          # walk from root to authoritative
$ dig -x 8.8.8.8                  # reverse lookup
```

### TCP / sockets
```
$ ss -tnlp                        # listening TCP sockets with PIDs (Linux)
$ netstat -an | grep LISTEN       # macOS / portable
$ lsof -i :8080                   # what process holds port 8080
```

### Packet capture
```
$ sudo tcpdump -i en0 -nn 'tcp port 443'
$ sudo tcpdump -i any -w capture.pcap 'host 1.2.3.4'
$ wireshark capture.pcap          # GUI analysis
```

### HTTP
```
$ curl -v https://example.com
$ curl -I https://example.com               # headers only
$ curl --resolve example.com:443:1.2.3.4 \
       https://example.com                  # bypass DNS for testing
$ httpie:  http GET example.com
```

### Throughput
```
$ iperf3 -s                       # server side
$ iperf3 -c server.example.com    # client side, measure bandwidth
```

### Interfaces
```
$ ip a                            # Linux interface info
$ ifconfig                        # macOS / BSD
$ ip route                        # routing table
$ route -n get default            # macOS default route
```

### Telnet / nc for protocol-level pokes
```
$ nc -vz example.com 443          # TCP connect test
$ nc -u host 53                   # UDP
$ telnet smtp.example.com 25      # talk SMTP by hand
```

---

## 23. Common Real-World Scenarios

### "Why is `example.com` slow?"
1. `ping example.com` — is it reachable? what's the RTT?
2. `mtr example.com` — where in the path is the latency?
3. `dig example.com` — DNS healthy and TTLs sane?
4. `curl -w '@curl-format.txt' -o /dev/null -s https://example.com` — break down DNS, connect, TLS, TTFB, total.
5. Browser DevTools → Network tab → waterfall.
6. Check CDN status / cache hit ratio.

### "I can't connect to my server"
- Is the server **listening**? `ss -tnlp` on the host.
- Is the **port reachable** from outside? `nc -vz host port` from your machine.
- Firewall? Cloud security group? Local `ufw` / `pf`?
- DNS pointing to the right IP? `dig +short host`.
- Health check failing on the LB? Check LB target health.

### "What's eating my bandwidth?"
- `iftop`, `nload`, `bandwhich` (per-process).
- `tcpdump` with filters by host/port.
- ISP traffic graphs.

### Designing a scalable web service
```
            Anycast DNS (Route 53 / Cloudflare)
                       │
                       ▼
                 CDN (cache static)
                       │ miss
                       ▼
                 Global LB / Anycast IP
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Region A LB       Region B LB
              │                 │
        ┌─────┴─────┐     ┌─────┴─────┐
        ▼     ▼     ▼     ▼     ▼     ▼
       app   app   app   app   app   app
         \   |    /        \    |    /
          ▼  ▼   ▼          ▼   ▼   ▼
         primary DB ◄── replica ──► replica
```

### Microservices networking checklist
- mTLS between services (or service mesh).
- Timeouts at every call (and budgets that compose).
- Retries with **jitter + backoff**, but only on idempotent ops.
- Circuit breakers.
- Distributed tracing (OpenTelemetry).
- Rate limits per caller.

---

## 24. Glossary & Quick Reference

| Term | One-line meaning |
|---|---|
| **MAC** | 48-bit hardware address on a NIC |
| **IP** | Internet address (v4 32-bit, v6 128-bit) |
| **Port** | 16-bit number identifying an app endpoint |
| **Socket** | OS handle for one end of a network connection |
| **Packet** | L3 unit (IP) |
| **Frame** | L2 unit (Ethernet) |
| **Segment** | L4 TCP unit |
| **Datagram** | L4 UDP unit (also generally "connectionless packet") |
| **MTU** | Max bytes per L2 frame (1500 default Ethernet) |
| **MSS** | Max TCP payload bytes (MTU − 40 for IPv4/TCP) |
| **RTT** | Round-trip time |
| **TTL** | Time to live; hop counter in IP, expiry in DNS |
| **CIDR** | Classless prefix notation (`/24`) |
| **NAT** | Translates many private IPs into one public |
| **DHCP** | Auto-assigns IP, gateway, DNS to clients |
| **DNS** | Name → address resolution |
| **TLS** | Transport-layer encryption + auth |
| **PKI** | Public-key infrastructure (CAs, certs) |
| **VPN** | Encrypted tunnel between sites/hosts |
| **CDN** | Edge cache network |
| **WAF** | Web Application Firewall (L7) |
| **mTLS** | Mutual TLS — client also presents a cert |
| **BGP** | Inter-AS routing protocol of the public Internet |
| **AS** | Autonomous System (one routing domain) |
| **VPC** | Virtual Private Cloud |
| **CNI** | Container Network Interface |

---

## 25. End-to-End Request Flow: Client → ALB → Kubernetes → Pod → DB/VM

> Trace a single HTTPS request all the way from a user's browser, through DNS, the AWS Application Load Balancer, into a Kubernetes cluster, through CNI/iptables, into the application container, out via NAT to a database VM, and back. Every hop is named and explained — this is the picture you want in your head when debugging "why is my request slow / failing / weirdly routed."

---

### 25.1 The Big Picture

```
                ┌─────────────────────────────────────────────────────┐
                │                     User's browser                    │
                │           https://api.example.com/orders/42          │
                └───────────────────────┬─────────────────────────────┘
                                        │
                       (1) DNS lookup → (2) TLS handshake
                                        │
                                        ▼
                ╔══════════════════ AWS Public Edge ═══════════════════╗
                ║   Route 53 (DNS)   →   CloudFront (optional CDN)      ║
                ║                   →   AWS Shield / WAF                ║
                ║                   →   ALB (Application Load Balancer)║
                ╚═══════════════════════════════════════════════════════╝
                                        │
                                        ▼
                ┌───────────── VPC: 10.0.0.0/16 ──────────────────────┐
                │                                                       │
                │  Public Subnets (per AZ)                              │
                │  ┌─────────────┐  ┌─────────────┐                     │
                │  │ ALB Node az-a│  │ ALB Node az-b│                    │
                │  └──────┬──────┘  └──────┬──────┘                     │
                │         │                 │                           │
                │         ▼                 ▼                           │
                │  Private Subnets (EKS worker nodes)                   │
                │  ┌─────────────────────┐  ┌─────────────────────┐    │
                │  │ Node EC2  10.0.2.10 │  │ Node EC2  10.0.3.10 │    │
                │  │ ┌──────┐ ┌────────┐ │  │ ┌──────┐ ┌────────┐ │    │
                │  │ │kube- │ │ kubelet│ │  │ │kube- │ │ kubelet│ │    │
                │  │ │proxy │ │        │ │  │ │proxy │ │        │ │    │
                │  │ └──────┘ └────────┘ │  │ └──────┘ └────────┘ │    │
                │  │ ┌────────────────┐  │  │ ┌────────────────┐  │    │
                │  │ │ Pod  10.244.1.5│  │  │ │ Pod  10.244.2.7│  │    │
                │  │ │ app:8080       │  │  │ │ app:8080       │  │    │
                │  │ └────────────────┘  │  │ └────────────────┘  │    │
                │  └─────────────────────┘  └─────────────────────┘    │
                │                                                       │
                │       ↓ outbound (e.g. external API, S3, RDS)        │
                │                                                       │
                │  Public Subnet                                        │
                │  ┌─────────────────┐                                  │
                │  │  NAT Gateway    │ → Internet Gateway → public IP   │
                │  └─────────────────┘                                  │
                │                                                       │
                │  Database Subnet (private, no IGW)                    │
                │  ┌─────────────────┐                                  │
                │  │ RDS / DB VM     │  10.0.5.20:5432                  │
                │  └─────────────────┘                                  │
                │                                                       │
                └───────────────────────────────────────────────────────┘
```

The next sections walk every numbered hop.

---

### 25.2 Step 1 — Browser → DNS Resolution

The browser needs an IP for `api.example.com`.

1. **Browser cache** — checked first (TTL-bound).
2. **OS resolver cache** — `systemd-resolved`, `mDNSResponder`, etc.
3. **Stub resolver** sends a UDP/53 (or DoH/DoT) query to the configured **recursive resolver** (e.g., 1.1.1.1 or your ISP's).
4. Recursive resolver walks: **root → `.com` TLD → Route 53 authoritative NS → answer**.
5. Route 53 returns either:
   - A direct **A/AAAA** record, or
   - An **alias** record pointing at the ALB's DNS name (`my-alb-1234567890.us-east-1.elb.amazonaws.com`), which itself resolves to **multiple A records** — one per AZ-local ALB node, behind anycast-style health-aware DNS.

**Key behavior:** AWS returns *multiple* IPs and rotates them; the client picks one and connects. ALB IPs can change at any time → never hard-code them.

```
$ dig api.example.com +short
my-alb-1234567890.us-east-1.elb.amazonaws.com.
54.231.10.5
54.231.11.7
```

---

### 25.3 Step 2 — TCP + TLS Handshake to the ALB

Once the client has an IP (say `54.231.10.5`), it opens a connection.

1. **TCP 3-way handshake** (SYN, SYN-ACK, ACK) to ALB on port 443.
2. **TLS handshake**:
   - ClientHello includes **SNI** (`api.example.com`) — ALB uses SNI to select the correct certificate from **ACM** (AWS Certificate Manager).
   - ServerHello + certificate chain (leaf + intermediate, signed by Amazon's public CA).
   - Key exchange (ECDHE), session keys derived, encrypted application data begins.
3. **TLS termination at the ALB** — the ALB decrypts the request. The connection from ALB → backend pod is a **separate** TCP/HTTP connection (ALB pools and reuses these to backends).

> Trust boundary insight: the user's TLS session ends at the ALB. From ALB onward you're inside your VPC. You can re-encrypt to the pod (HTTPS target group), but most setups use plain HTTP inside the VPC and rely on VPC isolation + security groups.

---

### 25.4 Step 3 — ALB Internals: Listener → Rule → Target Group

The ALB is an L7 proxy (built on AWS-internal proxies). Inside it:

```
HTTPS Listener (port 443)
    │
    ├── Default rule
    │
    └── Rule  IF Host header == api.example.com AND Path == /orders/*
              THEN forward to Target Group "orders-tg"
                          │
                          └── Targets: instance/IP/lambda/ALB
```

**Target type for EKS** is normally **`ip`** (IP-mode), with the **AWS Load Balancer Controller** registering pod IPs directly into the target group. Alternative is **`instance`** mode, where ALB sends to a NodePort and `kube-proxy` re-routes — adds an extra hop.

ALB also:
- Runs **health checks** (`GET /healthz` every N seconds; an unhealthy target is removed from rotation).
- Picks a target via **round-robin** or **least outstanding requests**.
- Adds headers: `X-Forwarded-For` (real client IP), `X-Forwarded-Proto`, `X-Forwarded-Port`, `X-Amzn-Trace-Id`.
- Logs each request to **S3 access logs** (if enabled).
- Cross-AZ routing depends on the **cross-zone load balancing** flag (on by default for ALB).

---

### 25.5 Step 4 — ALB → Pod (IP-mode)

With IP-mode targets, ALB has the pod's IP (`10.244.1.5`) directly. The packet leaves the ALB ENI and traverses the VPC:

1. ALB ENI sits in a **public subnet** (`10.0.1.0/24`).
2. Destination `10.244.1.5` is the pod IP from the **CNI overlay** range. The VPC route table for the ALB subnet has a route for the pod CIDR pointing at the worker node's ENI (when using AWS VPC CNI, pod IPs come from the *VPC* range and are assigned to **secondary IPs on the node's ENI**, so no overlay is needed at all).

#### AWS VPC CNI specifics
- Each EKS worker EC2 has a primary ENI plus optional secondary ENIs.
- Each ENI has a **primary private IP** (the node's IP) and many **secondary private IPs** assigned to pods.
- The pod IP **is a real VPC IP**. ALB → pod traffic is just normal VPC routing — no overlay encapsulation.

**Result:** The packet arrives on the worker node, kernel sees that destination IP belongs to a pod's veth pair, and forwards into the pod's network namespace.

---

### 25.6 Step 5 — Inside the Worker Node: veth, netns, iptables/eBPF

A pod on a Linux node lives in its own **network namespace** (`netns`). Connecting it to the host:

```
   Pod netns (10.244.1.5)         Host netns (10.0.2.10)
   ┌──────────────────────┐      ┌──────────────────────┐
   │   eth0 (veth-pod)    │◄────►│ vethXXXX  (veth-host)│
   │   10.244.1.5         │      │  bridged / routed    │
   └──────────────────────┘      └──────────────────────┘
                                           │
                                           ▼
                                     ENI (eth0) 10.0.2.10
```

**veth pair** = a virtual cable: one end inside the pod, one end on the host. The host end is either:
- Plugged into a **Linux bridge** (`cbr0`/`docker0` style — used by Flannel/bridge), or
- **Routed** point-to-point (used by AWS VPC CNI, Calico in BGP mode).

Once the packet enters the host netns:
- The host's **routing table** decides next hop.
- **iptables / nftables / eBPF** chains run (`PREROUTING → FORWARD → POSTROUTING`).
- **kube-proxy** programs these rules to implement Kubernetes Services.

#### What kube-proxy actually does
For a `Service` of type `ClusterIP` (e.g., `payments-svc → 10.96.45.10:80`), kube-proxy installs rules so any packet to that virtual IP is **DNAT**'d to one of the matching pod IPs:

- **iptables mode:** chains of `-m statistic --mode random --probability 0.33` for round-robin.
- **IPVS mode:** kernel L4 LB, scales better.
- **Cilium / eBPF:** replaces kube-proxy entirely with eBPF programs attached to socket / TC hooks.

For our flow with **IP-mode ALB**, the ALB has the pod IP directly, so no Service-level DNAT is needed — the packet lands on the pod's veth and goes straight in.

---

### 25.7 Step 6 — Pod Receives the Request

Inside the pod's netns:
1. Kernel matches the destination port (8080) to the listening socket of the application process (e.g., a Node.js / Go / Java server).
2. Kernel completes any remaining TCP work (the connection from ALB → pod is separate from client → ALB; this is a fresh TCP connection that the ALB opened).
3. The application reads the HTTP request, runs business logic, and prepares a response.

**Container-vs-pod note:** A *pod* shares one netns across all its containers. So a sidecar (e.g., Envoy in a service mesh) on `localhost:15001` can transparently intercept outbound traffic before it leaves the pod.

---

### 25.8 Step 7 — Pod → Database (Private Subnet)

The application now needs to query the database. Two common cases:

#### Case A: DB is **inside the VPC** (RDS, EC2, another pod)
1. App opens a TCP connection to `db.internal.example.com:5432`.
2. Kubernetes **CoreDNS** resolves this:
   - If it's a Kubernetes Service (`postgres.default.svc.cluster.local`), CoreDNS returns the ClusterIP.
   - If it's an external name (`db.internal.example.com`), CoreDNS forwards to the **VPC resolver** (`169.254.169.253`), which returns the RDS endpoint's private IP.
3. Outbound packet leaves the pod's veth, enters the host netns.
4. Host routes via the **subnet route table**: destination `10.0.5.20` is in the VPC, so a "local" route applies. No NAT, no IGW.
5. **Security group on RDS** must allow port 5432 from the worker node (or the pod's secondary IP — security groups apply to ENIs).
6. RDS responds; reverse path is symmetric.

> Performance note: when using AWS VPC CNI, the pod IP is a real ENI IP, so security groups can be scoped to **pod-level granularity** with **Security Groups for Pods** (uses Trunk ENIs).

#### Case B: DB / API is **outside the VPC** (e.g., third-party API, public S3 endpoint without VPC endpoint)
This is the **NAT Gateway** case.

---

### 25.9 Step 8 — Outbound to the Public Internet via NAT Gateway

Pods in private subnets cannot reach the Internet directly (no route to IGW). The pattern:

```
Pod 10.244.1.5  ──►  Worker Node 10.0.2.10  ──►  Subnet route table:
                                                    0.0.0.0/0 → NAT Gateway
                                                  ──►  NAT GW  10.0.1.99
                                                  ──►  IGW
                                                  ──►  Internet target  1.2.3.4
```

#### What the NAT Gateway actually does
- It's a managed AWS service running in a public subnet, with an **Elastic IP** as its public-facing address.
- For each new outbound connection, NAT GW rewrites:
  - Source IP `10.244.1.5` → its EIP `54.123.45.67`
  - Source port → an ephemeral port chosen from its pool
- It maintains a translation table keyed by `(src ip, src port, dst ip, dst port, proto)`.
- The remote server sees only the EIP. It replies to `54.123.45.67:ephem`. NAT GW looks up the table and rewrites destination back to `10.244.1.5:original_port`.

#### Limits worth knowing
- **55,000 simultaneous connections per (destination IP, destination port, protocol)** per NAT Gateway. High-fanout systems hitting the same external target can exhaust this — usually solved by adding more NAT GWs (one per AZ already gives you 3) or by using **VPC endpoints**.
- ~45 Gbps throughput per NAT GW; scales automatically.
- NAT GW lives in *one* AZ — for HA, deploy one per AZ and route each private subnet to the local one.

#### Alternative: VPC Endpoints
For AWS services (S3, DynamoDB, ECR, Secrets Manager, etc.), use a **Gateway** or **Interface VPC Endpoint** to talk to the service privately, bypassing the NAT GW entirely. This saves NAT data-processing cost and removes a SPOF.

---

### 25.10 Step 9 — Talking to a Plain VM (EC2) Backend

Sometimes a pod calls a legacy service running on an EC2 instance (a "VM") in the same VPC.

```
Pod 10.244.1.5  ──► Node 10.0.2.10  ──► VPC routing  ──► EC2 instance 10.0.4.30:8080
```

1. CoreDNS / pod resolver resolves `legacy.internal` to the EC2's private IP.
2. Packet egresses the pod's veth, enters the host.
3. Host netns has a route for `10.0.0.0/16` pointing "local" (within VPC).
4. AWS's underlying SDN delivers the packet to the EC2's ENI in the destination subnet.
5. **Security groups** on both sides enforce L4 allow rules. **NACLs** on the subnets enforce stateless allow/deny.
6. EC2's local kernel receives, demultiplexes to the listening service.

**No NAT, no overlay** — flat L3 reachability inside the VPC.

---

### 25.11 Step 10 — Response Travels Back

Symmetric reverse path:

1. App writes response → kernel queues TCP segments → out the pod's eth0.
2. Worker node forwards to the ALB's ENI (this connection was opened *by the ALB*; reply goes back over the same socket).
3. ALB sees the backend response, applies any response-rule transformations, encrypts onto the **client-side** TLS session, and sends it to the user.
4. Browser reads the encrypted bytes, decrypts, parses, renders.

---

### 25.12 Where Each Layer's State Lives

| Hop | Stateful entities | If it dies, what breaks |
|---|---|---|
| DNS | Recursive cache, browser cache | Initial lookup fails; cached lookups still work |
| TLS | Session keys (per connection); Session Tickets (resumption) | Reconnect & re-handshake |
| ALB | Connection table client-side & backend-side; sticky sessions (if enabled) | Client reconnects; new ALB node picks up via Route 53 |
| kube-proxy / CNI | iptables/IPVS rules; conntrack | Existing flows may drop on rule reload (rare); new flows reroute |
| NAT Gateway | Connection table | On AZ failure, connections via that NAT GW fail; new ones via another AZ |
| Pod | App-level state, sockets | Service routes around dead pod via readiness probe |
| RDS / DB VM | DB sessions, transactions | App reconnects; in-flight TX rolled back |

---

### 25.13 Failure Domains & Resilience Patterns

- **Multi-AZ everywhere:** ALB across ≥2 AZs, EKS nodes across ≥2 AZs (one node group per AZ), one NAT Gateway per AZ, RDS Multi-AZ.
- **Pod-level HA:** `replicas ≥ 2`, **Pod Disruption Budget**, **anti-affinity** to spread pods across nodes/AZs.
- **Health checks at every layer:** ALB target health, Kubernetes liveness/readiness probes, application-level circuit breakers.
- **Connection draining:** ALB "deregistration delay" (default 300s) lets in-flight requests complete before a target leaves.
- **Graceful pod shutdown:** `preStop` hook + `terminationGracePeriodSeconds` so the pod stops accepting new requests, finishes in-flight ones, then exits.
- **Idempotent retries with jitter** when calling downstream services.

---

### 25.14 Observability Hooks Along the Path

| Hop | Where to look |
|---|---|
| DNS | CloudWatch Route 53 query logs; `dig` from client |
| TLS / ALB | ALB **access logs** (S3), CloudWatch metrics: `RequestCount`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count` |
| VPC routing | **VPC Flow Logs** (5-tuple + bytes/packets per flow) |
| kube-proxy / CNI | `iptables -L -t nat`, `ipvsadm -Ln`, Cilium Hubble for eBPF |
| Pod | Kubernetes events, `kubectl logs`, `kubectl describe pod` |
| App | OpenTelemetry traces (request → DB), Prometheus metrics |
| NAT GW | CloudWatch: `ActiveConnectionCount`, `ErrorPortAllocation`, `BytesOutToDestination` |
| RDS | Performance Insights, slow query log, `pg_stat_activity` |

---

### 25.15 Common Pitfalls Seen in Production

- **`ErrorPortAllocation` on NAT GW** — too many concurrent flows to the same external `(ip, port)`. Fix: add NAT GWs, use VPC endpoints, or pool/keep-alive your outbound HTTP clients.
- **5xx spike during deploy** — pods removed from Service before `preStop` drains. Fix: add `preStop: sleep 10`, set `terminationGracePeriodSeconds: 60`, ensure ALB deregistration delay covers it.
- **Asymmetric routing breaking conntrack** — happens when traffic enters via ALB in AZ-a but the pod replies via a route that exits AZ-b. Usually handled by AWS, but worth keeping in mind with custom transit setups.
- **Security Group is open, but request still fails** — check the *other* end's SG, the **NACL** on the subnet (stateless!), and the **route table**.
- **Pod cannot reach external API but other pods can** — NodeLocal DNSCache misconfig, NetworkPolicy blocking egress, or the pod scheduled on a node in a subnet without a NAT route.
- **Inconsistent latency from a single pod** — noisy neighbor on the node; check CPU throttling (`container_cpu_cfs_throttled_seconds_total`).
- **TLS handshake errors at the ALB** — SNI mismatch, expired ACM cert, or client using a TLS version below the listener's minimum.

---

### 25.16 Putting It All Together — A Single Numbered Trace

```
[1]  User → DNS lookup api.example.com (Route 53 alias → ALB IP set)
[2]  User → TCP+TLS to ALB IP 54.231.10.5:443  (cert from ACM, SNI matched)
[3]  ALB → matches Listener rule → Target Group "orders-tg"
[4]  ALB → opens TCP to pod IP 10.244.1.5:8080  (IP-mode target via AWS LB Ctrl)
[5]  Packet → ALB ENI (public subnet) → VPC routing → Worker ENI (private subnet)
[6]  Worker host → veth pair → Pod netns → app socket on :8080
[7]  App handles request; needs DB → CoreDNS resolves db.internal → 10.0.5.20
[8]  Pod → Worker ENI → VPC routing → RDS in DB subnet (no NAT, just SG check)
[9]  App also calls third-party API → Pod → Node → NAT GW → IGW → Internet
[10] Responses traverse the reverse paths; ALB encrypts back onto user's TLS
```

That single trace touches: DNS, TCP, TLS, HTTP, ALB target groups, AWS VPC CNI, Linux veth/netns, kube-proxy/iptables, security groups, NAT, IGW — i.e., every concept above.

---

## Recommended Further Reading

- ***Computer Networking: A Top-Down Approach*** — Kurose & Ross
- ***TCP/IP Illustrated, Volume 1*** — W. Richard Stevens
- ***High Performance Browser Networking*** — Ilya Grigorik (free at hpbn.co)
- **RFCs you'll actually re-read** — 791 (IP), 793/9293 (TCP), 1035 (DNS), 2616/9110 (HTTP), 8446 (TLS 1.3), 9000 (QUIC)
- **Cloudflare Learning Center** — `learn.cloudflare.com` (great free explainers)
