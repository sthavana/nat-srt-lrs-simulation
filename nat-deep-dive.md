# NAT Deep Dive — Why SRT Needs Caller, Listener, and Rendezvous

## Table of Contents

1. [Why NAT Exists](#1-why-nat-exists)
2. [How NAT Works — Packet Level](#2-how-nat-works--packet-level)
3. [The NAT Translation Table](#3-the-nat-translation-table)
4. [The Fundamental NAT Problem](#4-the-fundamental-nat-problem)
5. [Types of NAT](#5-types-of-nat)
6. [NAT Mapping Lifetime and Keep-Alives](#6-nat-mapping-lifetime-and-keep-alives)
7. [Port Forwarding — The Manual Override](#7-port-forwarding--the-manual-override)
8. [Firewalls vs NAT — They Are Different](#8-firewalls-vs-nat--they-are-different)
9. [Why SRT Uses UDP (and Why It Matters for NAT)](#9-why-srt-uses-udp-and-why-it-matters-for-nat)
10. [SRT Connection Mode 1: Caller / Listener](#10-srt-connection-mode-1-caller--listener)
11. [SRT Connection Mode 2: Rendezvous](#11-srt-connection-mode-2-rendezvous)
12. [The Handshake Through NAT](#12-the-handshake-through-nat)
13. [How to Decide: Caller vs Listener vs Rendezvous](#13-how-to-decide-caller-vs-listener-vs-rendezvous)
14. [Applying This to VOS 360 and LRS](#14-applying-this-to-vos-360-and-lrs)
15. [Caller/Listener Is About Connection, Not Data Direction](#15-callerlistener-is-about-connection-not-data-direction)
16. [Common Misconceptions](#16-common-misconceptions)
17. [Troubleshooting NAT Issues with SRT](#17-troubleshooting-nat-issues-with-srt)

---

## 1. Why NAT Exists

IPv4 provides ~4.3 billion addresses. There are 20+ billion connected devices. The math doesn't work.

**NAT (Network Address Translation)** solves this by letting many devices share one public IP address. Your home, office, venue, or data center uses **private IP ranges** internally:

| Range | Class | Addresses |
|-------|-------|-----------|
| 10.0.0.0 – 10.255.255.255 | Class A | 16.7 million |
| 172.16.0.0 – 172.31.255.255 | Class B | 1 million |
| 192.168.0.0 – 192.168.255.255 | Class C | 65,536 |

These private IPs **do not exist on the public internet**. A router with NAT sits at the boundary and translates between private and public addresses.

```
    Private Network                         Public Internet
 ┌─────────────────┐     ┌──────────┐
 │ 192.168.1.10    │     │  NAT     │     Public IP: 203.0.113.50
 │ 192.168.1.11    │────→│  Router  │────→ Internet
 │ 192.168.1.12    │     │          │
 │ 192.168.1.13    │     └──────────┘
 └─────────────────┘
   All share ONE public IP
```

---

## 2. How NAT Works — Packet Level

### Outbound: Your Encoder Sends to the Cloud

```
Step 1: Encoder creates a UDP packet
   ┌─────────────────────────────────────────────┐
   │  Source:      192.168.1.10:5000             │
   │  Destination: 52.1.2.3:9000  (cloud ingest)│
   └─────────────────────────────────────────────┘

Step 2: Packet hits the NAT router (public IP: 203.0.113.50)
   Router REWRITES the source address:
   ┌─────────────────────────────────────────────┐
   │  Source:      203.0.113.50:49152  ← rewritten│
   │  Destination: 52.1.2.3:9000                 │
   └─────────────────────────────────────────────┘
   Router picks a random external port (49152)

Step 3: Router creates a MAPPING TABLE entry:
   ┌──────────────────────────────────────────────────────────┐
   │  Internal              External               Remote    │
   │  192.168.1.10:5000  ↔  203.0.113.50:49152  →  52.1.2.3 │
   └──────────────────────────────────────────────────────────┘

Step 4: Packet arrives at 52.1.2.3:9000
   The cloud server sees source = 203.0.113.50:49152
   It has NO idea 192.168.1.10 exists
```

### Inbound Response: Cloud Replies

```
Step 5: Cloud server sends response
   ┌─────────────────────────────────────────────┐
   │  Source:      52.1.2.3:9000                 │
   │  Destination: 203.0.113.50:49152            │
   └─────────────────────────────────────────────┘

Step 6: Packet arrives at your NAT router
   Router looks up mapping table: port 49152 → 192.168.1.10:5000
   Router REWRITES the destination:
   ┌─────────────────────────────────────────────┐
   │  Source:      52.1.2.3:9000                 │
   │  Destination: 192.168.1.10:5000  ← rewritten│
   └─────────────────────────────────────────────┘

Step 7: Packet delivered to encoder. Round trip complete.
```

---

## 3. The NAT Translation Table

The NAT table is the heart of everything. It's a dynamic table that the router maintains:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     NAT TRANSLATION TABLE                          │
├───────────────────────┬──────────────────────┬─────────────────────┤
│  Internal             │  External            │  Remote             │
├───────────────────────┼──────────────────────┼─────────────────────┤
│  192.168.1.10:5000    │  203.0.113.50:49152  │  52.1.2.3:9000     │
│  192.168.1.10:5001    │  203.0.113.50:49153  │  198.51.100.7:8000 │
│  192.168.1.11:3000    │  203.0.113.50:49154  │  52.1.2.3:9000     │
│  (empty)              │  (empty)             │  (empty)            │
└───────────────────────┴──────────────────────┴─────────────────────┘
```

**Key rules:**
- An entry is **created** when an internal device sends a packet **outbound**
- An entry **allows** return traffic from the remote to come back through
- An entry **expires** after a timeout if no packets flow (typically 30s–3min for UDP)
- There is **no entry** for unsolicited inbound traffic → it gets **dropped**

---

## 4. The Fundamental NAT Problem

This is the critical concept for understanding SRT connection modes.

### Outbound always works:

```
Internal device sends packet OUT → NAT creates mapping → return traffic flows back
✓ This always works. No configuration needed.
```

### Unsolicited inbound is BLOCKED:

```
External device sends packet IN → NAT looks up mapping → NO ENTRY → DROPPED

   External 198.51.100.7:7000
       │
       ▼
   ┌──────────┐
   │ NAT      │ ← Looks up table for port 5000...
   │ Router   │    NO MATCHING ENTRY
   │          │    → PACKET SILENTLY DROPPED
   └──────────┘
       ✗
   192.168.1.10  (unreachable from outside)
```

**NAT was never designed as a security device, but it acts as a one-way gate:**
- Traffic OUT → creates a door (mapping) → responses can come back IN through that door
- Traffic IN without a door → dropped

**This is exactly why SRT needs different connection modes.** Someone has to "open the door" by sending the first packet outbound.

---

## 5. Types of NAT

Not all NATs behave identically. The type determines what return traffic is allowed through a mapping.

### Type 1: Full Cone NAT (most permissive)

```
192.168.1.10:5000 sends to 52.1.2.3:9000
   → Mapping created: 203.0.113.50:49152

   ANY external host (any IP, any port) can now send to 203.0.113.50:49152
   and it reaches 192.168.1.10:5000

   ✓ Easiest to traverse
   ✗ Rare in practice (security concerns)
```

### Type 2: Address-Restricted Cone NAT

```
192.168.1.10:5000 sends to 52.1.2.3:9000
   → Mapping created: 203.0.113.50:49152 → restricted to 52.1.2.3

   Only 52.1.2.3 (ANY port) can send back through the mapping.
   Other IPs are blocked.

   ✓ Moderate — common in small office routers
```

### Type 3: Port-Restricted Cone NAT (most common)

```
192.168.1.10:5000 sends to 52.1.2.3:9000
   → Mapping created: 203.0.113.50:49152 → restricted to 52.1.2.3:9000

   Only 52.1.2.3:9000 (exact IP AND port) can send back.
   Same IP, different port? Blocked.

   ✓ This is what most home/venue routers use
   ✓ Still traversable by SRT Caller/Listener
```

### Type 4: Symmetric NAT (most restrictive)

```
192.168.1.10:5000 sends to 52.1.2.3:9000
   → Mapping: 203.0.113.50:49152 → 52.1.2.3:9000

192.168.1.10:5000 sends to 198.51.100.7:8000
   → Mapping: 203.0.113.50:49153 → 198.51.100.7:8000
                            ^^^^^ DIFFERENT external port for each destination!

   ✗ Hardest to traverse
   ✗ Rendezvous hole-punching fails if BOTH sides have Symmetric NAT
   ✗ Common in enterprise networks, carrier-grade NAT (CGNAT), cellular
```

### Which types work with which SRT modes?

| NAT Type | Caller/Listener | Rendezvous |
|----------|----------------|------------|
| Full Cone | Works | Works |
| Address-Restricted | Works | Works |
| Port-Restricted | Works | Works |
| Symmetric (one side) | Works | Works (usually) |
| Symmetric (BOTH sides) | Works (if one has port forwarding) | **FAILS** |

---

## 6. NAT Mapping Lifetime and Keep-Alives

NAT mappings are **temporary**. The router deletes them after a period of inactivity:

| Protocol | Typical Timeout |
|----------|----------------|
| UDP | 30 seconds – 3 minutes |
| TCP | 5 – 30 minutes |

**The problem for SRT:** If a live stream has a brief pause (e.g. source goes quiet), the NAT mapping can expire. When packets resume, the router has no mapping → packets are dropped → stream breaks.

**SRT's solution:** Keep-alive packets sent approximately every second:

```
Active stream:
   Encoder ──[video data]──→ Router mapping stays alive

Stream pauses (no video for 30 seconds):
   Without keep-alive: mapping expires → stream breaks
   With SRT keep-alive: ──[keep-alive]──→ mapping refreshed → stream resumes
```

SRT's bidirectional control channel (ACK, NAK, keep-alive packets) ensures mappings never expire during a session.

---

## 7. Port Forwarding — The Manual Override

Port forwarding is a **permanent, manually configured** NAT mapping:

```
Router admin configures:
   "Forward all traffic arriving on public port 9000
    to internal 192.168.1.10:9000"

This creates a PERMANENT entry:
   ┌─────────────────────────────────────────────┐
   │  External 203.0.113.50:9000                 │
   │  → Always forwards to 192.168.1.10:9000     │
   │  → No timeout, no restrictions              │
   │  → ANY external host can reach it            │
   └─────────────────────────────────────────────┘
```

**Port forwarding makes a device behind NAT reachable from outside** — effectively turning it into a publicly accessible endpoint for that specific port.

**Limitations:**
- Requires admin access to the router
- Not possible on cellular networks, hotel WiFi, many venue networks
- Needs coordination between network team and video engineering team
- Static internal IP required (or DHCP reservation)

**This is why the Haivision/Harmonic presentations note that Caller/Listener mode with firewalls "needs IT administrators to setup the firewalls — may take a lot of time, especially in big organizations."**

---

## 8. Firewalls vs NAT — They Are Different

A common confusion: NAT and firewalls are different things that often coexist on the same router.

| | NAT | Firewall |
|---|-----|----------|
| **Purpose** | Address translation (share one public IP) | Security (block unwanted traffic) |
| **Blocks inbound by** | No mapping in table (side effect) | Explicit deny rules |
| **Stateful?** | Yes (tracks mappings) | Often yes (tracks connections) |
| **Configuration** | Automatic (outbound creates mappings) | Manual rules by admin |
| **Can be port-forwarded** | Yes (static NAT rule) | Yes (allow rule on specific port) |

For SRT to work through both NAT and a firewall:

```
NAT requirement:  A mapping must exist (outbound traffic creates one,
                  OR port forwarding configured)

Firewall requirement: UDP traffic must be ALLOWED on the SRT port
                      (both inbound and outbound)
                      Stateful firewalls: if outbound is allowed,
                      return traffic is automatically allowed
```

From the Haivision deployment guide, for Caller/Listener with firewalls:
- The **Listener's** firewall must allow inbound UDP on the SRT port
- Port forwarding must map the public port to the Listener's internal IP:port
- The firewall must support **outbound NAT source port** (not rewrite it)
- Packet filtering must be disabled for SRT traffic

---

## 9. Why SRT Uses UDP (and Why It Matters for NAT)

SRT is built on **UDP**, not TCP. This is a deliberate choice that directly affects NAT traversal:

### TCP NAT traversal is hard:

```
TCP requires a 3-way handshake:
   Client → SYN → Server
   Server → SYN-ACK → Client
   Client → ACK → Server

TCP "simultaneous open" (both send SYN) exists in theory
but most NAT devices DON'T properly support it.
```

### UDP NAT traversal is straightforward:

```
UDP is connectionless — any packet to the right IP:port works.

Side A sends packet → creates NAT mapping on A's router
Side B sends packet → creates NAT mapping on B's router
Next packets from each side flow through the other's mapping.

No handshake coordination needed at the protocol level.
```

**This is one of the key reasons SRT chose UDP.** UDP hole-punching (used in Rendezvous mode) is well-understood and works reliably with most NAT types.

From the Harmonic SRT presentation: SRT is described as:
- "Point to point connection-oriented protocol"
- "Based on UDP"
- "One way multimedia data (video + audio + metadata)"
- "Bidirectional control messages"

The multimedia data flows one direction (source → destination), but **control messages (ACK, NAK, keep-alive) flow in both directions** through the same NAT mappings.

---

## 10. SRT Connection Mode 1: Caller / Listener

### The Concept

This is modeled on the traditional client/server pattern:

```
LISTENER: "I'm waiting on port 9000. Someone will connect to me."
           (Like a phone that's turned on, waiting for a call)

CALLER:   "I know the Listener's address. I'll initiate the connection."
           (Like dialing a phone number)
```

### Why It Works Through NAT

```
┌──────────────────────┐        Internet        ┌───────────────────────┐
│  CALLER              │                         │  LISTENER             │
│  192.168.1.10        │                         │  52.1.2.3:9000        │
│  Behind NAT          │                         │  Public IP            │
│  (venue, field, etc) │                         │  (cloud, data center) │
└──────────────────────┘                         └───────────────────────┘

1. Caller sends SRT handshake OUTBOUND through its NAT:
   192.168.1.10:5000 → [NAT rewrites] → 203.0.113.50:49152 → 52.1.2.3:9000
   NAT MAPPING CREATED ✓

2. Listener at 52.1.2.3:9000 receives handshake, sends response:
   52.1.2.3:9000 → 203.0.113.50:49152
   NAT mapping EXISTS → forwarded to 192.168.1.10:5000 ✓

3. Handshake completes. Bidirectional control + unidirectional media flows.
```

**The Caller's outbound packet creates the NAT mapping. All subsequent traffic — including the Listener's responses, ACKs, NAKs — flows through that mapping.**

### When to Use Caller/Listener

From the Haivision Deployment Guide:

**Use Caller when:**
1. Point-to-point streaming where you initiate the connection (e.g. encoder → cloud)
2. Device is behind a firewall/NAT and you cannot configure port forwarding
3. Device has a dynamic IP (e.g. portable encoder using DHCP/cellular)
4. Device is on a network you don't control (venue, hotel, cellular)

**Use Listener when:**
1. Device is on a public IP or has port forwarding configured
2. Device is behind a firewall you have admin control over
3. Device is directly exposed to the internet (cloud VM, data center)
4. You want multiple Callers to connect to one endpoint

### The "Call Home" Pattern (Reversed)

A common misconception: "The video source should be the Caller." **Not necessarily.**

The Haivision guide shows two patterns:

**Pattern A — Source is Caller (most common):**
```
SRT Source          SRT Destination
(Encoder)           (Decoder/Ingest)
  Caller      →      Listener

Source initiates connection, then sends media.
Use when: source is behind NAT, destination has public IP.
```

**Pattern B — Destination is Caller:**
```
SRT Source          SRT Destination
(Encoder)           (Decoder/Ingest)
  Listener    ←      Caller

Destination initiates connection, source responds, then source sends media.
Use when: source has public IP, destination is behind NAT.
```

**Key insight from the Haivision guide:** *"Once communication is established, the notion of which device is Caller and which is Listener becomes unimportant. What matters is the source/destination relationship, which is decoupled from the caller/listener relationship."*

---

## 11. SRT Connection Mode 2: Rendezvous

### The Problem It Solves

When **BOTH** sides are behind NAT and neither has port forwarding:

```
┌──────────────────────┐        Internet        ┌───────────────────────┐
│  SITE A              │                         │  SITE B               │
│  Behind NAT          │                         │  Behind NAT           │
│  No port forwarding  │                         │  No port forwarding   │
│  Public: 203.0.113.50│                         │  Public: 198.51.100.7 │
└──────────────────────┘                         └───────────────────────┘

A sends to B → B's NAT has no mapping → DROPPED ✗
B sends to A → A's NAT has no mapping → DROPPED ✗

Neither side can reach the other!
```

### UDP Hole Punching — How Rendezvous Works

Both sides must know each other's public IP:port (configured manually or via signaling). Then:

```
Step 1: Both sides start sending SIMULTANEOUSLY

   Site A sends:  203.0.113.50:5000 → 198.51.100.7:5000
      A's NAT creates mapping ✓
      Packet arrives at B's NAT → NO mapping yet → DROPPED ✗

   Site B sends:  198.51.100.7:5000 → 203.0.113.50:5000
      B's NAT creates mapping ✓
      Packet arrives at A's NAT → A has mapping (from above) → FORWARDED ✓

Step 2: Now A's next packet arrives at B's NAT
   B now has mapping (from step 1) → FORWARDED ✓

Step 3: Both mappings are open. Connection established!
```

**Timeline visualization:**

```
     A's NAT                                    B's NAT
t=0  A sends→B  creates mapping          B sends→A  creates mapping
     B drops A's pkt (no mapping yet)     A drops B's pkt (no mapping yet)

t=1  A sends→B                           B sends→A
     B's mapping now exists → ✓           A's mapping exists → ✓

t=2  ←── Bidirectional flow established ──→
     Both NATs have active mappings
     SRT handshake completes
```

Each side's outbound packet "punches a hole" in its own NAT. The other side's packets then flow through that hole.

### Rendezvous Limitations

From the Harmonic SRT project presentation (slide 13), **important real-world findings:**

> *"This is what Haivision/SRT Alliance pretends. However, due to trials done over the Internet, no real benefit was found for 'Rendez-vous' mode."*

**Why Rendezvous can fail in practice:**
1. **Symmetric NAT on both sides** — the external port changes per destination, so the "hole" is punched for the wrong port
2. **Strict enterprise firewalls** — may block outbound UDP entirely or use application-layer filtering
3. **Carrier-grade NAT (CGNAT)** — cellular and some ISPs use double-NAT, making hole-punching unreliable
4. **Timing** — both sides must send nearly simultaneously; large clock differences can cause failures
5. **Both sides need each other's public IP:port** — requires out-of-band coordination

From the Haivision deployment guide's firewall constraints:
- Firewalls must be **stateful** (track connection state)
- Firewalls must allow the control packets from the decoder back to the encoder
- Rendezvous relies on the "connection tracking" feature of stateful firewalls

---

## 12. The Handshake Through NAT

SRT uses a **two-phase handshake** specifically designed to work through NAT.

### Caller/Listener Handshake

```
Phase 1 — Induction:
   Caller → Listener:  "I want to connect"
                        (version, encryption flags, SRT params)
                        ↑ This outbound packet creates the Caller's NAT mapping

   Listener → Caller:  "Here's a SYN cookie"
                        (anti-DoS: proves Caller's address is real)
                        ↑ Travels back through the NAT mapping

Phase 2 — Conclusion:
   Caller → Listener:  "Here's your cookie + my final settings"
   Listener → Caller:  "Accepted. Connection established."

All four packets use the SAME source port on the Caller,
so they all flow through the SAME NAT mapping.
```

### Rendezvous Handshake

```
Both sides simultaneously send Phase 1:
   A → B:  Induction packet (also punches hole in A's NAT)
   B → A:  Induction packet (also punches hole in B's NAT)

Both sides detect the other's Induction, proceed to Phase 2:
   A → B:  Conclusion packet (flows through B's hole)
   B → A:  Conclusion packet (flows through A's hole)

The handshake packets themselves serve double duty:
   1. Carry SRT protocol negotiation data
   2. Create the NAT mappings needed for communication
```

---

## 13. How to Decide: Caller vs Listener vs Rendezvous

### The Decision Flowchart

```
                          Is Side A reachable         Is Side B reachable
                          from the internet?          from the internet?
                          (public IP, port forward,   (public IP, port forward,
                           or Express Route)           or Express Route)

                              YES                         YES
                               → Either can be Listener
                                 (choose based on operational preference)

                              YES                         NO
                               → A = Listener, B = Caller

                              NO                          YES
                               → B = Listener, A = Caller

                              NO                          NO
                               → Rendezvous (if NAT types allow)
                                 OR set up port forwarding on one side
```

### Quick Reference

| Scenario | Mode | Why |
|----------|------|-----|
| Cloud/DC ingest ← field encoder | Encoder = **Caller**, Cloud = **Listener** | Cloud has public IP; encoder is behind venue NAT |
| Two cloud VMs | Either can be Listener | Both have known IPs, NAT is not an issue |
| Encoder behind cellular NAT → studio | Encoder = **Caller**, Studio = **Listener** | Cellular NAT cannot be port-forwarded |
| Two venues, both behind NAT | **Rendezvous** (or set up port forwarding on one) | Neither can receive unsolicited inbound |
| Same LAN / VLAN | Either, or **Rendezvous** | No NAT involved at all |
| Via Express Route (private peering) | Either can be Listener | Express Route bypasses public internet NAT |

---

## 14. Applying This to VOS 360 and LRS

### The VOS 360 Architecture

Based on the architecture diagram:

```
                    Express Route                      Azure Cloud
┌─────────┐       ┌──────────┐     ┌─────────────────────────────────────────┐
│  XOS    │       │ Express  │     │   LRS VMs        Ingest    Stream     │
│ Encoder │──────→│ Route    │────→│  ┌──────────┐    VMs      Processor  │
│ (West)  │       │ Colorado │     │  │Cloud     │──→┌──────┐──→┌───────┐ │
└─────────┘       └──────────┘     │  │Pounder   │   │Ingest│   │Stream │ │
                                   │  └──────────┘   │ VMs  │   │Proc   │ │
┌─────────┐       ┌──────────┐     │  ┌──────────┐   └──────┘   └───────┘ │
│  XOS    │       │ Express  │     │  │Cloud     │                        │
│ Encoder │──────→│ Route    │────→│  │Edge      │                        │
│ (East)  │       │ Ashburn  │     │  └──────────┘                        │
└─────────┘       └──────────┘     │                                      │
                                   │  Regions: US West 2, US East 2       │
                                   └─────────────────────────────────────────┘
                                          │
                                          ▼
                                   Egress VMs → Cache Proxy → CDN Origins
```

### What Is LRS (Live Routing Service)?

LRS VMs are **SRT relay/routing nodes** deployed in the Azure cloud. They:
1. **Receive** SRT streams from source encoders (XOS)
2. **Route** streams to the appropriate Ingest VMs
3. Provide **redundancy** — multiple LRS VMs across regions (US West 2, US East 2)
4. Handle **failover** — if one path fails, traffic routes through another region

The LRS has two roles visible in the diagram:
- **Cloud Pounder** — first hop LRS, receives from Express Route
- **Cloud Edge** — second hop LRS, forwards to Ingest VMs

### Who Is Caller and Who Is Listener?

#### XOS Encoder → LRS (via Express Route)

```
┌──────────┐    Express Route    ┌──────────┐
│   XOS    │ ──────────────────→ │   LRS    │
│ Encoder  │    (private link)   │   VM     │
│          │                     │          │
│ CALLER   │                     │ LISTENER │
└──────────┘                     └──────────┘
```

**Why this way:**
- Express Route is a **private peering** connection — it bypasses the public internet entirely
- No public NAT is involved, but the pattern still applies:
  - LRS VMs have known, stable Azure IPs → natural **Listener**
  - XOS encoders initiate the connection → natural **Caller**
- The LRS Listener can accept connections from multiple XOS encoders on the same port (using SRT Stream ID to differentiate)

#### XOS Encoder → LRS (over public internet, no Express Route)

```
┌──────────┐      Public Internet     ┌──────────┐
│   XOS    │                          │   LRS    │
│ Encoder  │ ────────────────────────→│   VM     │
│ Behind   │                          │ Azure    │
│ venue NAT│                          │ public IP│
│          │                          │          │
│ CALLER   │                          │ LISTENER │
└──────────┘                          └──────────┘
```

**Why:**
- XOS encoder at venue is behind NAT → **must be Caller** (outbound creates mapping)
- LRS VM in Azure has a public IP → **must be Listener** (reachable from internet)
- If the encoder were Listener, venue NAT would block the LRS's connection attempt

#### LRS → Ingest VMs (within Azure)

```
┌──────────┐     Azure VNet      ┌──────────┐
│   LRS    │ ──────────────────→ │  Ingest  │
│ Cloud    │    (same network)   │   VM     │
│ Edge     │                     │          │
│ CALLER   │                     │ LISTENER │
└──────────┘                     └──────────┘
```

**Why:**
- Both are in the same Azure VNet — no NAT involved
- Ingest VM as **Listener** is the natural choice: it's the stable endpoint that receives streams
- LRS as **Caller**: it knows which Ingest VM to send to and initiates the connection
- Alternatively, Ingest could be Caller if it needs to "pull" from LRS — the mode is decoupled from data direction

#### The Numbered Flow in the Architecture Diagram

```
 1  → XOS encoder originates the stream
 2  → Stream hits West/East XOS node
 3,4 → Stream enters Express Route (Colorado or Ashburn)
 5,6 → Cloud Pounder LRS VMs receive (SRT Listener)
 7,8 → Cloud Edge LRS VMs receive (forwarded via SRT)
 9-12 → Ingest VMs in US West 2 / US East 2 receive
 13  → Stream Processor transcodes
 14  → Egress VMs output
 15,16 → Redundant paths for failover
```

At each SRT hop, the upstream device is typically the **Caller** and the downstream device is the **Listener**, because:
- Downstream devices have known, stable addresses
- Upstream devices may be dynamic or behind NAT
- Listeners can accept multiple callers (fan-in)

---

## 15. Caller/Listener Is About Connection, Not Data Direction

This is the most important concept to understand. From the Haivision Deployment Guide:

> *"Regardless of which device is in which mode, when the SRT handshake is completed both source and destination continue to exchange control packets containing information about network conditions, dropped packets, etc. Once communication is established, the notion of which device is Caller and which is Listener becomes unimportant."*

```
         CALLER/LISTENER              SOURCE/DESTINATION
         (connection role)            (data flow role)

         WHO initiates the            WHO sends and WHO receives
         SRT handshake?               the actual video data?

         Determined by:               Determined by:
         Network topology             Application requirement
         (NAT, firewalls)             (encoder → decoder)

         Matters only during:         Matters for entire session:
         Connection setup             Media flows source → destination
         (first 2-3 packets)          continuously
```

**Example: The four possible combinations:**

| Connection Role | Data Flow | When to use |
|-----------------|-----------|-------------|
| Source = Caller, Dest = Listener | Source → Dest | Encoder behind NAT, decoder in cloud (most common) |
| Source = Listener, Dest = Caller | Source → Dest | Encoder in cloud, decoder pulls from it |
| Both = Rendezvous | Source → Dest | Both behind NAT |
| Either (same LAN) | Source → Dest | No NAT, doesn't matter |

In all cases, **video always flows from Source to Destination**. The Caller/Listener choice only affects who sends the first handshake packet.

---

## 16. Common Misconceptions

### "The Caller is always the sender of video"
**Wrong.** Caller is about who initiates the connection. A decoder can be the Caller that connects to an encoder (Listener) to pull a stream.

### "Rendezvous is better than Caller/Listener"
**Not necessarily.** The Harmonic evaluation found no real benefit for Rendezvous over the internet. Caller/Listener is simpler and more reliable when one side has a public IP or port forwarding.

### "NAT is the same as a firewall"
**No.** NAT translates addresses; firewalls enforce security rules. A device can be behind NAT but have no firewall, or have a firewall but no NAT (public IP with firewall rules).

### "SRT needs port forwarding on both sides"
**No.** Only the **Listener** needs to be reachable. The Caller creates its own NAT mapping by initiating the connection outward. That's the whole point of the Caller/Listener model.

### "Express Route eliminates all NAT concerns"
**Mostly true but not entirely.** Express Route provides private peering that bypasses public internet NAT. However, the internal network at each end may still have its own NAT or firewall rules.

### "If the connection works, NAT isn't involved"
**Wrong.** NAT is always translating packets on the path — you just don't notice it when it works. It becomes visible only when it breaks (mapping timeout, wrong mode choice, etc.).

---

## 17. Troubleshooting NAT Issues with SRT

### Symptom: SRT connection never establishes

```
Possible causes:
1. Listener is behind NAT without port forwarding
   → Fix: Make the Listener side publicly reachable, or swap roles

2. Firewall blocking UDP on the SRT port
   → Fix: Allow UDP inbound on Listener's port, outbound on Caller's port

3. Wrong IP configured — using private IP instead of public
   → Fix: Caller must target the Listener's PUBLIC IP, not its private IP

4. Symmetric NAT on both sides (Rendezvous mode)
   → Fix: Switch to Caller/Listener with port forwarding on one side
```

### Symptom: Connection establishes but drops after 30-60 seconds

```
Possible cause: NAT mapping timeout
   → The NAT mapping expired during a quiet period
   → SRT keep-alives should prevent this, but some aggressive NATs
     have very short timeouts (< 15 seconds)

Fix:
   → Check SRT keep-alive interval (should be ~1 second)
   → Check NAT timeout settings on the router
   → Consider using a longer SRT peer idle timeout
```

### Symptom: Stream works for hours then suddenly fails

```
Possible causes:
1. NAT table overflow — router ran out of mapping slots
   → Common with consumer routers handling many streams

2. Router firmware rebooted / NAT table cleared
   → SRT should auto-reconnect, but check reconnect settings

3. Public IP changed (DHCP lease renewal on WAN)
   → The Caller's public IP changed, breaking the mapping
   → Fix: Use DDNS or static public IP
```

### Diagnostic Commands

```bash
# Check if the SRT port is reachable from outside (on Listener side)
nc -vzu <public_ip> <srt_port>

# Check NAT type (using STUN)
stunclient stun.l.google.com:19302

# Monitor SRT connection statistics
srt-live-transmit ... -stats 1

# Check for UDP traffic on the SRT port
tcpdump -i any udp port 9000 -nn

# Verify port forwarding is working
# (run from an external network)
nmap -sU -p 9000 <public_ip>
```

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| **NAT** | Allows private IPs to share one public IP by translating addresses |
| **NAT mapping** | Created by outbound traffic; allows return traffic in; expires after timeout |
| **The NAT problem** | Unsolicited inbound traffic is dropped — no mapping exists |
| **Caller** | Initiates connection outward, creating NAT mapping. Works from behind any NAT |
| **Listener** | Waits for connections. Must be reachable (public IP or port forwarding) |
| **Rendezvous** | Both sides send simultaneously to punch holes in each other's NAT |
| **Port forwarding** | Manual permanent NAT mapping — makes internal device reachable from outside |
| **In VOS 360** | XOS encoders = Caller, LRS VMs = Listener, Ingest VMs = Listener |
| **Connection vs Data** | Caller/Listener determines who handshakes first; Source/Destination determines who sends video |
| **Keep-alives** | SRT sends periodic packets to prevent NAT mapping expiration |
