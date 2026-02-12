# SRT (Secure Reliable Transport) — Complete Technical Reference

---

## 1. Introduction

### 1.1 What is SRT?

**SRT (Secure Reliable Transport)** is an open-source video transport protocol that enables the delivery of high-quality, low-latency live video across unpredictable networks such as the public internet. It sits at the **application layer** of the OSI model, built on top of **UDP**, and provides three core capabilities that raw UDP lacks: **reliability**, **security**, and **timing recovery**.

```
OSI Model Placement:

┌──────────────────────────────────────────┐
│  Layer 7 — Application                    │
│  ┌────────────────────────────────────┐  │
│  │  SRT Protocol                       │  │
│  │  (ARQ, TSBPD, Encryption, Handshake)│  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│  Layer 4 — Transport                      │
│  ┌────────────────────────────────────┐  │
│  │  UDP                                │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│  Layer 3 — Network                        │
│  ┌────────────────────────────────────┐  │
│  │  IP (IPv4 / IPv6)                   │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│  Layers 1-2 — Physical / Data Link        │
└──────────────────────────────────────────┘
```

SRT is **content-agnostic** — it can carry any byte stream, though it is most commonly used with **MPEG-TS (Transport Stream)** containing H.264/HEVC video and AAC/MPEG audio.

---

### 1.2 History and Origin

```
Timeline:

2012    Haivision develops SRT internally for their video encoders.
        Built on top of UDT (UDP-based Data Transfer protocol),
        an academic project from the University of Illinois at Chicago
        designed for high-speed scientific data transfer.

2013    Haivision deploys SRT commercially in their Makito and
        Kulabyte product lines for low-latency contribution.

2017    Haivision open-sources SRT under the Mozilla Public License 2.0 (MPLv2).
        The SRT Alliance is formed with founding members including
        Haivision, Wowza, and others.

2018    SRT Alliance grows to 200+ members. Adoption accelerates in
        broadcast, OTT, and cloud video platforms.
        Microsoft, Harmonic (VOS), Telestream, and others integrate SRT.

2020    SRT v1.4 released — improved congestion control, stream groups,
        and access control via Stream ID.

2021    SRT becomes a de facto standard for internet-based video
        contribution, replacing many RTMP-based workflows.

2023    SRT v1.5 released — connection bonding for multi-path redundancy,
        improved group management.

2024+   SRT is integrated into FFmpeg, GStreamer, OBS Studio, VLC,
        and virtually every professional video platform including
        Harmonic VOS, AWS Elemental, Wowza, and Nimble Streamer.
```

#### UDT Heritage

SRT inherited its core architecture from **UDT (UDP-based Data Transfer)**, developed by Dr. Yunhong Gu at UIC:

```
UDT Contributions to SRT:
  - UDP-based reliable transport framework
  - Selective ARQ with NAK-based loss detection
  - Congestion control infrastructure
  - Connection handshake protocol

SRT Added on Top of UDT:
  - TSBPD (Timestamp-Based Packet Delivery) — live video timing
  - AES encryption — content security
  - Stream ID — connection multiplexing and routing
  - Too-Late Packet Drop — bounded latency for live media
  - Improved congestion control for constant-bitrate media
  - Connection bonding (v1.5+)
```

UDT was designed for **bulk data transfer** (large files at high speed). SRT repurposed its reliable transport mechanics for **live media**, adding the critical concept that **late data is as useless as lost data** — a principle that fundamentally shapes every design decision in SRT.

---

### 1.3 Core Design Principles

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. Low Latency First                                               │
│     Live video demands sub-second delay. SRT achieves this by       │
│     using UDP (not TCP) and bounded retransmission windows.         │
│                                                                     │
│  2. Reliability Within a Time Budget                                │
│     Lost packets are retransmitted, but ONLY if there's still      │
│     time. Late packets are dropped, never buffered indefinitely.   │
│                                                                     │
│  3. Timing Fidelity                                                 │
│     TSBPD reconstructs the sender's original packet cadence at     │
│     the receiver, removing all network jitter.                     │
│                                                                     │
│  4. Security by Default                                             │
│     AES encryption is built into the protocol, not bolted on.      │
│     Every production deployment should use it.                     │
│                                                                     │
│  5. Firewall/NAT Traversal                                          │
│     Multiple connection modes (Caller/Listener/Rendezvous)         │
│     ensure SRT works across NATs without special infrastructure.   │
│                                                                     │
│  6. Content Agnostic                                                │
│     SRT treats its payload as opaque bytes. It works with          │
│     MPEG-TS, raw H.264/HEVC, or any byte stream.                  │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

### 1.4 Technical Glossary

| Term | Full Name | Definition |
|---|---|---|
| **ARQ** | Automatic Repeat reQuest | Error-control mechanism: detect lost packets, request retransmission |
| **NAK** | Negative Acknowledgment | Receiver tells sender: "I'm missing these packets" |
| **ACK** | Acknowledgment | Receiver tells sender: "I've received up to this packet" |
| **ACKACK** | ACK of ACK | Sender acknowledges the ACK (used for RTT measurement) |
| **TSBPD** | Timestamp-Based Packet Delivery | Receiver delivers packets at sender's original timing + latency |
| **TLPKTDROP** | Too-Late Packet Drop | Drop packets that miss their delivery deadline |
| **RTT** | Round-Trip Time | Time for a packet to travel sender→receiver→sender |
| **SEK** | Session Encryption Key | AES key used to encrypt payload data |
| **KEK** | Key Encryption Key | Key used to wrap/protect the SEK during exchange |
| **KM** | Key Material | Encryption key information exchanged during handshake |
| **PBKDF2** | Password-Based Key Derivation Function 2 | Derives KEK from the passphrase |
| **Stream ID** | — | String identifier for multiplexing connections on a single port |
| **UDT** | UDP-based Data Transfer | Academic protocol that SRT is built upon |
| **PCR** | Program Clock Reference | 27 MHz clock reference inside MPEG-TS |
| **PTS** | Presentation Timestamp | 90 kHz timestamp — when to display a frame |
| **DTS** | Decode Timestamp | 90 kHz timestamp — when to decode a frame |
| **FEC** | Forward Error Correction | Proactive redundancy (SRT uses ARQ instead, but supports FEC as optional) |
| **ppm** | Parts Per Million | Clock drift measurement (e.g., 100 ppm = 100μs drift per second) |

---

### 1.5 SRT vs Alternatives

| Protocol | Latency | Reliability | Encryption | Firewall Friendly | Open Source |
|---|---|---|---|---|---|
| **SRT** | Low (sub-second) | ARQ retransmit | AES built-in | Yes (UDP) | Yes (MPLv2) |
| RTMP | Medium (1-3s) | TCP-based | TLS (optional) | Yes (TCP 1935) | Partially |
| RIST | Low | ARQ retransmit | DTLS | Yes (UDP) | Yes |
| Zixi | Low | ARQ + FEC | AES | Yes (UDP) | No (proprietary) |
| Raw UDP/TS | Lowest | None | None | Yes | N/A |
| WebRTC | Very Low | NACK + FEC | DTLS-SRTP | Yes (ICE/STUN) | Yes |

---

## 2. Protocol Foundation — Why UDP, Not TCP

SRT is built on **UDP**, not TCP. This is a deliberate architectural decision driven by the requirements of live video.

### 2.1 The Comparison

| | TCP | UDP (raw) | SRT (over UDP) |
|---|---|---|---|
| Reliability | Full retransmission | None | Selective ARQ |
| Latency | High (head-of-line blocking) | Lowest | Low & controlled |
| Congestion control | Yes (aggressive) | None | Yes (custom) |
| Ordering | Strict | None | Yes (reorder buffer) |

### 2.2 Reliability — Why Each Behaves Differently

#### TCP — Full Retransmission

```
Sender                           Receiver
  |--- Seg 1 ------------------>|  ✓
  |--- Seg 2 -----X  (lost)     |
  |--- Seg 3 ------------------>|  (held, can't deliver — waiting for Seg 2)
  |--- Seg 4 ------------------>|  (held)
  |                              |
  |  (timeout or 3 dup ACKs)    |
  |--- Seg 2 (retransmit) ----->|  NOW delivers Seg 2, 3, 4 together
```

TCP was designed in the 1980s for **general-purpose data** (files, web pages, emails) where **every single byte matters**. A missing byte in a file = corrupted file. So TCP guarantees every byte is delivered, in order, with no duplicates. If any segment is lost, TCP **stops delivering everything after it** until the gap is filled. This is fine for a file download but devastating for live video.

#### Raw UDP — No Reliability

```
Sender                           Receiver
  |--- Pkt 1 ------------------>|  ✓ got it
  |--- Pkt 2 -----X  (lost)     |  gone forever, nobody notices
  |--- Pkt 3 ------------------>|  ✓ got it
```

UDP was designed to be a **minimal transport** — just a thin wrapper over IP. There are no sequence numbers, no ACKs, no retransmission logic. The protocol has **no concept of whether the receiver got the packet**. For some applications (DNS lookups, game state updates, live audio), getting old data late is worse than not getting it at all. UDP leaves the reliability decision to the application layer.

#### SRT — Selective ARQ (the sweet spot)

```
Sender                           Receiver
  |--- Pkt 1 ------------------>|  ✓ buffered in slot 1
  |--- Pkt 2 -----X  (lost)     |
  |--- Pkt 3 ------------------>|  ✓ buffered in slot 3  (NOT discarded!)
  |--- Pkt 4 ------------------>|  ✓ buffered in slot 4  (NOT discarded!)
  |                              |
  |                              |  Gap detected: slot 2 is empty
  |<-------- NAK (Pkt 2) -------|  "I'm missing packet 2"
  |--- Pkt 2 (retransmit) ----->|  ✓ fills slot 2
  |                              |  Deliver all in order
```

SRT was designed specifically for **live video**, which has a unique requirement: "I want reliability, but only within a time budget." Only the lost packet is retransmitted. Pkt 3 is NOT blocked by missing Pkt 2. If the retransmission doesn't arrive before the playout deadline, the packet is **dropped** rather than stalling the entire stream. For video, it's better to lose 1 frame than to freeze the entire stream waiting for retransmission.

### 2.3 Latency — Why Each Behaves Differently

#### TCP — High Latency (Head-of-Line Blocking)

```
Timeline:
  t=0ms    Pkt 1 arrives  → can't deliver (Pkt 0 still missing)
  t=0ms    Pkt 2 arrives  → can't deliver (Pkt 0 still missing)
  t=0ms    Pkt 3 arrives  → can't deliver (Pkt 0 still missing)
           ┊
           ┊  ← entire pipeline FROZEN, decoder starved
           ┊
  t=200ms  Pkt 0 retransmit arrives → NOW deliver 0,1,2,3 all at once (burst)
```

This is called **head-of-line (HOL) blocking**. One lost packet blocks everything behind it, even if later packets have already arrived. Retransmission timeout (RTO) can be 200ms–1s. After loss, TCP assumes congestion and **dramatically reduces** sending rate, then slowly ramps back up. For live video, a 200ms freeze followed by a burst of frames causes **visible stutter**.

#### Raw UDP — Lowest Latency

```
t=0ms    Pkt sent → arrives ~RTT/2 later → immediately available
```

There is literally **nothing in the way**: no connection state, no ACK to wait for, no ordering to enforce, no congestion window. The packet goes out on the wire immediately. The downside: if the network drops 5% of packets, your video has 5% corruption.

#### SRT — Low & Controlled Latency

```
t=0ms     Pkt sent (timestamp=T)
t=10ms    Pkt arrives at receiver
t=10ms    Placed in latency buffer, scheduled for delivery at T + latency
           ┊
           ┊  ← latency buffer (e.g., 120ms)
           ┊
t=120ms   Pkt delivered to decoder at exactly the right moment
```

SRT adds latency **deliberately and precisely**. No HOL blocking — later packets aren't held up by earlier losses. TSBPD means packets are delivered at a steady rate, not in bursts. The latency is bounded and predictable — you set it and that's the maximum added delay. TCP's delay is unbounded and unpredictable.

### 2.4 Congestion Control — Why Each Behaves Differently

#### TCP — Aggressive (Sawtooth Pattern)

```
Window Size
  ▲
  │    ╱╲        ╱╲
  │   ╱  ╲      ╱  ╲        ← sawtooth pattern
  │  ╱    ╲    ╱    ╲
  │ ╱      ╲  ╱      ╲
  │╱        ╲╱        ╲
  └──────────────────────→ Time
       loss    loss
       event   event
```

TCP was designed for **shared networks** where fairness matters. On packet loss, it **cuts the sending rate in half** (multiplicative decrease). Video generates a **constant bitrate**. When TCP cuts the rate in half after a loss, the encoder is still producing data at the same rate — TCP buffers pile up, latency spikes, then bursts of old data flood out. TCP's congestion control is designed for **elastic traffic** (web browsing, file transfer). Video is **inelastic** — the data rate is fixed by the content.

#### Raw UDP — None

```
[Encoder: 10 Mbps] ──────────────────────> [Network: 5 Mbps link]
                                                  │
                                             50% packets dropped
```

UDP has no feedback mechanism — the sender has no idea if the receiver is getting packets or if the network is congested. It just sends. This can overwhelm the network and cause packet loss for everyone.

#### SRT — Custom, Video-Aware

```
Sending Rate
  ▲
  │ ─────────────────────────  ← steady rate matching video bitrate
  │            ↑ small bump for retransmissions
  └──────────────────────────→ Time
```

SRT's congestion control is purpose-built for live video: rate-based (not window-based), sends at the rate the video requires. Reserves ~25% overhead for retransmissions. Does NOT halve the rate on loss. Packets are sent at even intervals (pacing), not in bursts. TCP assumes loss = congestion → slow down. SRT assumes loss = transient → retransmit within budget.

### 2.5 Ordering — Why Each Behaves Differently

#### TCP — Strict (Blocking)

TCP presents a **byte stream abstraction** — bytes flow in exactly the order they were sent. Even if Byte 5000 arrived before Byte 4999 on the wire, TCP holds Byte 5000 and delivers 4999 first. Enforcing order requires buffering out-of-order packets and blocking delivery until the gap is filled (back to HOL blocking).

#### Raw UDP — None

Each UDP datagram is independent — the protocol has no concept of "this packet comes after that one." They can take different routes and arrive in any order.

#### SRT — Ordered but Non-Blocking

```
Wire arrival:     Pkt 3,  Pkt 1,  Pkt 2
                    │       │       │
                    ▼       ▼       ▼
              ┌─────────────────────────┐
              │   SRT Reorder Buffer     │
              │   Slot 1: Pkt 1  ✓      │
              │   Slot 2: Pkt 2  ✓      │
              │   Slot 3: Pkt 3  ✓      │
              └─────────────────────────┘
                    │
                    ▼ (delivered in order at scheduled time)
              Pkt 1 → Pkt 2 → Pkt 3
```

SRT uses sequence numbers so it knows the correct order. Out-of-order packets are placed in their correct slot. The latency buffer gives time for reordering. If a packet never arrives before its playout deadline, SRT **skips it and moves on** — it does NOT block all subsequent packets like TCP.

### 2.6 Summary — The Design Philosophy

| | TCP | Raw UDP | SRT |
|---|---|---|---|
| **Designed for** | General data | Minimal transport | Live video |
| **Reliability** | "Every byte, no matter the cost" | "Not my problem" | "Best effort within a time budget" |
| **Latency** | "Correctness > speed" | "Speed > everything" | "Speed + controlled recovery" |
| **Congestion** | "Be fair, back off on loss" | "Full speed, no awareness" | "Steady rate + retransmit budget" |
| **Ordering** | "Block until order is perfect" | "No order guaranteed" | "Order when possible, skip when necessary" |

The fundamental insight: **live video is time-sensitive data where late data is as useless as lost data**.

---

## 3. SRT Packet Structure

Every SRT data packet carries a 16-byte header:

```
┌────────────────────────────────────────────────────┐
│  SRT Data Packet Header (16 bytes)                 │
├────────────────────────────────────────────────────┤
│  Bit 0:     Packet Type (0 = data, 1 = control)   │
│  Bits 1-31: Sequence Number (31-bit, wraps around) │
│  Bits 32-63: Message Number + flags                │
│  Bits 64-95: Timestamp (μs since connection start) │
│  Bits 96-127: Destination Socket ID                │
├────────────────────────────────────────────────────┤
│  Payload (MPEG-TS data, up to 1316 bytes typical)  │
└────────────────────────────────────────────────────┘
```

Key fields:
- **Sequence Number**: 31-bit, monotonically increasing — how gaps are detected for ARQ
- **Timestamp**: microsecond precision — used for TSBPD scheduling (see Section 6)
- **Destination Socket ID**: routes packets to the correct connection on a shared listener port

The typical payload size of **1316 bytes** comes from MPEG-TS: 7 TS packets × 188 bytes = 1316 bytes, which fits within a standard Ethernet MTU of 1500 bytes (after UDP/IP headers).

---

## 4. ARQ (Automatic Repeat reQuest)

ARQ is the **heart of SRT's reliability**. It is an error-control mechanism: if something is lost, detect it and ask for it again.

### 4.1 Classical ARQ Strategies

There are three classical ARQ strategies:

```
1. Stop-and-Wait ARQ
2. Go-Back-N ARQ
3. Selective Repeat ARQ   ← SRT uses this
```

SRT implements **Selective Repeat ARQ** (also called Selective Reject / SREJ), enhanced with time-bounded delivery and NAK-based feedback.

#### Stop-and-Wait ARQ

```
Sender                        Receiver
  |--- Pkt 1 --------------->|
  |        (wait...)          |
  |<------ ACK 1 ------------|
  |--- Pkt 2 --------------->|
  |        (wait...)          |
  |<------ ACK 2 ------------|
```

- Send one packet, wait for ACK, then send next
- **Extremely slow** — the link is idle while waiting for ACK
- Throughput = 1 packet per RTT
- **Unusable for video**: at 10 Mbps with 50ms RTT, you'd get ~0.02 Mbps effective throughput

#### Go-Back-N ARQ

```
Sender                        Receiver
  |--- Pkt 1 --------------->|  ✓
  |--- Pkt 2 -----X (lost)   |
  |--- Pkt 3 --------------->|  ✗ discarded (out of order)
  |--- Pkt 4 --------------->|  ✗ discarded (out of order)
  |                           |
  |  (timeout — no ACK for 2) |
  |                           |
  |--- Pkt 2 --------------->|  ✓ (retransmit from Pkt 2 onwards)
  |--- Pkt 3 --------------->|  ✓
  |--- Pkt 4 --------------->|  ✓
```

- Sender keeps a **window** of N unacknowledged packets in flight
- On loss, receiver **discards all packets after the gap**
- Sender **retransmits from the lost packet onwards**
- **Wastes bandwidth**: Pkt 3 and 4 were already received once but got discarded

#### Selective Repeat ARQ (SRT's approach)

```
Sender                        Receiver
  |--- Pkt 1 --------------->|  ✓ buffered in slot 1
  |--- Pkt 2 -----X (lost)   |
  |--- Pkt 3 --------------->|  ✓ buffered in slot 3  (NOT discarded!)
  |--- Pkt 4 --------------->|  ✓ buffered in slot 4  (NOT discarded!)
  |                           |
  |                           |  Receiver detects gap: slot 2 is empty
  |<------ NAK (Pkt 2) ------|  "I'm missing packet 2"
  |                           |
  |--- Pkt 2 (retransmit) -->|  ✓ buffered in slot 2
  |                           |
  |                           |  All slots 1-4 filled → deliver in order
```

**Why SRT uses Selective Repeat:**

1. **No wasted bandwidth** — only the lost packet is retransmitted, not everything after it
2. **Receiver keeps out-of-order packets** — Pkt 3 and 4 are buffered, not discarded
3. **Minimal retransmission overhead** — on a link with 2% loss, only ~2% extra bandwidth is used
4. **Works within the latency budget** — retransmit only what's lost, and only if there's still time

---

### 4.2 SRT's ARQ Implementation — Detailed Mechanics

#### Sender's Send Buffer

The sender maintains a **Send Buffer** keeping copies of packets until ACKed:

```
Send Buffer:
┌─────────────────────────────────────────────────────────┐
│ Seq 100 │ Seq 101 │ Seq 102 │ Seq 103 │ Seq 104 │ ...  │
│  ACKed  │  ACKed  │ in-flight│ in-flight│ in-flight│     │
│ (can    │ (can    │ (keep   │ (keep   │ (keep   │     │
│  free)  │  free)  │  copy)  │  copy)  │  copy)  │     │
└─────────────────────────────────────────────────────────┘
          ↑                                        ↑
     Last ACKed                              Last Sent
```

The sender:
1. **Keeps a copy** of every sent packet until it's ACKed
2. **Tracks which packets have been NAKed** and schedules retransmission
3. **Limits retransmissions** — won't retransmit if the packet's playout time has already passed
4. **Paces retransmissions** — doesn't flood the network with retransmits

#### Receiver's Receive Buffer

The receiver maintains a **Receive Buffer** with slots indexed by sequence number:

```
Receive Buffer (indexed by sequence number):
┌───────┬───────┬───────┬───────┬───────┬───────┬───────┐
│Seq 200│Seq 201│Seq 202│Seq 203│Seq 204│Seq 205│Seq 206│
│  ✓    │  ✓    │ EMPTY │  ✓    │  ✓    │ EMPTY │  ✓    │
│       │       │ (gap!)│       │       │ (gap!)│       │
└───────┴───────┴───────┴───────┴───────┴───────┴───────┘
                  ↑                       ↑
              NAK sent               NAK sent
              for 202               for 205
```

The receiver:
1. **Places each arriving packet** in its correct slot (by sequence number)
2. **Detects gaps** — if Seq 201 and Seq 203 arrive but not Seq 202, there's a gap
3. **Sends NAK** (Negative Acknowledgment) for missing packets
4. **Sends periodic ACK** to confirm the highest consecutively received sequence number

---

### 4.3 Control Packet Types

SRT uses several control packet types for ARQ:

```
┌──────────────────────────────────────────────────────────────┐
│ Control Packet Type │ Direction       │ Purpose               │
├──────────────────────────────────────────────────────────────┤
│ ACK                 │ Receiver→Sender │ "I have received up   │
│                     │                 │  to sequence N"        │
├──────────────────────────────────────────────────────────────┤
│ ACKACK              │ Sender→Receiver │ "I got your ACK"      │
│                     │                 │  (used for RTT calc)   │
├──────────────────────────────────────────────────────────────┤
│ NAK (Loss Report)   │ Receiver→Sender │ "I am missing these   │
│                     │                 │  sequence numbers"     │
├──────────────────────────────────────────────────────────────┤
│ Keep-Alive          │ Both directions │ "I'm still here"      │
├──────────────────────────────────────────────────────────────┤
│ Shutdown            │ Both directions │ "I'm closing"         │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.4 The ACK/ACKACK/NAK Dance — Step by Step

```
Time    Sender                         Receiver
─────────────────────────────────────────────────────────
t=0     Send Pkt 1 (seq=1) -------->  Recv Pkt 1 → buffer slot 1  ✓
t=1     Send Pkt 2 (seq=2) -------->  Recv Pkt 2 → buffer slot 2  ✓
t=2     Send Pkt 3 (seq=3) ---X       (lost in transit)
t=3     Send Pkt 4 (seq=4) -------->  Recv Pkt 4 → buffer slot 4  ✓
                                       Gap detected! Slot 3 is empty
t=4
t=5                          <--------  NAK [seq=3]
                                        "I'm missing packet 3"
t=6     Received NAK for seq=3
        Lookup seq=3 in send buffer
        Found! Retransmit.
        Send Pkt 3 (seq=3) -------->  Recv Pkt 3 → buffer slot 3  ✓
                                       Gap filled!
t=7
t=10                         <--------  ACK [ack_seq=4]
                                        "I've received everything up to seq 4"
t=10    Received ACK for seq=4
        Free send buffer slots 1-4
        Send ACKACK ----------->       Receiver calculates RTT from
                                        ACK send time vs ACKACK recv time
```

---

### 4.5 NAK Timing — When Does the Receiver Send a NAK?

The receiver doesn't send a NAK immediately — it must distinguish real loss from reordering:

```
Event: Pkt 4 arrives, but Pkt 3 hasn't arrived yet

  Is this a real loss, or just reordering?

  Strategy: Wait a short period before sending NAK

  ┌─────────────────────────────────────────────────┐
  │  NAK Delay = max(20ms, RTT/2)                   │
  │                                                  │
  │  If Pkt 3 arrives within this window:            │
  │    → No NAK needed (it was just reordering)      │
  │                                                  │
  │  If Pkt 3 still missing after NAK delay:         │
  │    → Send NAK for Pkt 3                          │
  └─────────────────────────────────────────────────┘
```

---

### 4.6 Periodic NAK (Re-NAK)

What if the NAK itself gets lost, or the retransmitted packet gets lost?

```
t=5ms     NAK sent for Pkt 3
t=55ms    Expected retransmit hasn't arrived (RTT has elapsed)
          → Send NAK again for Pkt 3  (re-NAK)
t=105ms   Still missing?
          → Send NAK again (if still within latency budget)
t=120ms   Latency deadline reached for Pkt 3
          → Give up. Drop Pkt 3. Deliver Pkt 4 onwards.
```

The re-NAK interval is typically based on **RTT** — if one RTT has passed since the last NAK and the packet still hasn't arrived, request it again.

---

### 4.7 The Full Retransmission Timeline

```
                    Latency Budget (e.g., 120ms)
    ├──────────────────────────────────────────┤

    t=0ms        Pkt sent by sender
    t=10ms       Pkt should have arrived (based on RTT/2)
    t=20ms       Gap detected at receiver
    t=30ms       NAK delay expires → NAK sent
    t=40ms       NAK arrives at sender
    t=40ms       Sender retransmits Pkt
    t=50ms       Retransmitted Pkt arrives at receiver ✓

    Time used: 50ms out of 120ms budget
    Remaining: 70ms (enough for 1-2 more retransmit attempts if needed)

    ─────────────────────────────────────────────

    If retransmit ALSO lost:

    t=50ms       Expected retransmit didn't arrive
    t=80ms       Re-NAK sent (after RTT interval)
    t=90ms       Re-NAK arrives at sender
    t=90ms       Sender retransmits again
    t=100ms      Retransmitted Pkt arrives ✓

    Time used: 100ms out of 120ms budget
    Remaining: 20ms (tight — maybe one more attempt)
```

This is why the **latency setting should be >= 4x RTT** — it allows for:
- 1st attempt: detection + NAK + retransmit = ~2x RTT
- 2nd attempt: re-NAK + retransmit = ~1x RTT
- Safety margin: ~1x RTT

---

### 4.8 Too-Late Packet Drop (TLPKTDROP)

```
Latency budget: 120ms

  t=0ms      Pkt 3 sent (timestamp=T, scheduled delivery = T + 120ms)
  t=120ms    Delivery deadline for Pkt 3 reached

             Pkt 3 still missing?
             ┌─────────────────────────────┐
             │  YES → DROP Pkt 3           │
             │     → Move on to Pkt 4      │
             │     → Decoder sees a gap    │
             │       (minor glitch)        │
             │                             │
             │  This is BETTER than:       │
             │     → Holding Pkt 4,5,6...  │
             │     → Freezing the stream   │
             │     → Growing latency       │
             └─────────────────────────────┘
```

TLPKTDROP enforces the principle: **late data is useless data** for live video.

---

### 4.9 ACK Types — Full and Light

#### Full ACK (every 10ms or every 64 packets)

```
┌─────────────────────────────────────────────────────────┐
│ Full ACK Contents:                                       │
│                                                          │
│  - Last ACK Sequence Number (highest in-order received)  │
│  - RTT (measured round-trip time)                        │
│  - RTT Variance                                          │
│  - Available Buffer Size (receiver's remaining capacity) │
│  - Packets Receiving Rate                                │
│  - Estimated Link Capacity                               │
│  - Receiving Rate (bytes/sec)                            │
└─────────────────────────────────────────────────────────┘
```

This rich ACK provides the sender with **network telemetry** for congestion control.

#### Light ACK (between full ACKs)

```
┌─────────────────────────────────┐
│ Light ACK Contents:              │
│                                  │
│  - Last ACK Sequence Number only │
└─────────────────────────────────┘
```

Lightweight, just to keep the sender informed about progress without overhead.

---

### 4.10 Real-World Scenario

Scenario: 5 Mbps video stream, 50ms RTT, 0.5% packet loss, 200ms latency setting.

```
Packets per second: ~5,000,000 / (1316 × 8) ≈ 475 packets/sec
Packets lost per second: 475 × 0.005 ≈ 2-3 packets/sec
RTT: 50ms
Latency budget: 200ms (allows ~4 retransmit attempts)

Timeline for a typical lost packet:

  t=0ms      Sender sends Pkt #1000
  t=25ms     Pkt #1000 should arrive (RTT/2 = 25ms)
  t=25ms     It doesn't. Next packets arrive, gap detected.
  t=45ms     NAK delay expires (20ms). Receiver sends NAK for #1000
  t=70ms     NAK reaches sender (25ms transit). Sender retransmits #1000
  t=95ms     Retransmitted #1000 arrives at receiver ✓
  t=200ms    Pkt #1000's scheduled delivery time → delivered to decoder

  Recovery time: 95ms (well within 200ms budget)
  Retransmit attempts possible: 200ms / 50ms RTT = 4 attempts

Result: Stream plays perfectly with 200ms latency.
        Viewer sees zero glitches despite 0.5% network loss.
        Overhead bandwidth: ~0.5% extra (just the retransmits).
```

---

## 5. Connection Modes — Caller / Listener / Rendezvous

### 5.1 Why Three Modes?

The fundamental problem SRT solves with connection modes is **NAT and firewall traversal**:

```
The Internet Reality:

  [Encoder]                                              [Decoder/VOS]
  192.168.1.50                                           10.0.0.100
       │                                                      │
       ▼                                                      ▼
  ┌─────────┐          ┌──────────┐          ┌─────────┐
  │  NAT/   │          │          │          │  NAT/   │
  │Firewall │──────────│ Internet │──────────│Firewall │
  │(Source) │          │          │          │(Dest)   │
  └─────────┘          └──────────┘          └─────────┘
  Public: 203.0.1.5                          Public: 198.51.2.10

  Problem: Neither side can directly reach the other's
           private IP address through the NAT/firewall.
```

Each connection mode solves this differently, depending on **which side has an accessible port**.

---

### 5.2 The SRT Handshake — Foundation for All Modes

SRT uses a **two-phase handshake**:

#### Phase 1: Induction (legacy UDT-based)

```
Caller                              Listener
  │                                    │
  │  ┌──────────────────────────┐      │
  │  │ Handshake Request        │      │
  │  │  Version: 4 (initial)    │      │
  │  │  Type: INDUCTION         │      │
  │  │  SRT Socket ID: random   │──────>│
  │  │  Syn Cookie: 0           │      │
  │  └──────────────────────────┘      │
  │                                    │
  │      ┌──────────────────────────┐  │
  │      │ Handshake Response       │  │
  │      │  Version: 5 (SRT)       │  │
  │<──────│  Type: INDUCTION        │  │
  │      │  SRT Socket ID: random  │  │
  │      │  Syn Cookie: computed   │  │
  │      └──────────────────────────┘  │
  │                                    │
```

The **SYN Cookie** is a hash-based token (similar to TCP SYN cookies) that prevents:
- **Connection flooding attacks** — the listener doesn't allocate state until the cookie is returned
- **IP spoofing** — the caller must receive and return the cookie, proving it can receive traffic at its claimed address

#### Phase 2: Conclusion (SRT-specific negotiation)

```
Caller                              Listener
  │                                    │
  │  ┌──────────────────────────┐      │
  │  │ Handshake Request        │      │
  │  │  Version: 5 (SRT)       │      │
  │  │  Type: CONCLUSION        │      │
  │  │  Syn Cookie: (from Ph1) │──────>│
  │  │  SRT Flags:             │      │
  │  │    - TSBPDSND           │      │
  │  │    - TSBPDRCV           │      │
  │  │    - AES-256            │      │
  │  │    - Stream ID          │      │
  │  │  Latency: 120ms        │      │
  │  │  Passphrase → KM msg   │      │
  │  └──────────────────────────┘      │
  │                                    │
  │      ┌──────────────────────────┐  │
  │      │ Handshake Response       │  │
  │<──────│  Version: 5 (SRT)       │  │
  │      │  Type: CONCLUSION        │  │
  │      │  Negotiated params:     │  │
  │      │    - Latency: 200ms     │  │ (listener wanted higher)
  │      │    - Encryption: OK     │  │
  │      │    - TSBPD: enabled     │  │
  │      └──────────────────────────┘  │
  │                                    │
  │  ══════ CONNECTION ESTABLISHED ══════
  │         Data can now flow           │
```

Key negotiation rules:
- **Latency**: the **maximum** of both sides' requests is used
- **Encryption**: both sides must use the same passphrase or the handshake fails
- **TSBPD**: both sides agree to use timestamp-based delivery

---

### 5.3 Caller Mode — "I initiate the connection"

The Caller is the side that **initiates** the connection. It knows the Listener's IP address and port in advance.

```
Analogy: Making a phone call
  - You (Caller) dial a number
  - The other person (Listener) must have their phone on and be waiting
  - You need to know their number; they don't need to know yours in advance
```

#### Network Requirements

```
  [Caller]                                [Listener]
  Can be behind NAT ✓                     Must have accessible port ✓
  No port forwarding needed               Port must be open in firewall
  Outbound UDP must be allowed            Inbound UDP on specific port
```

**Why the Caller can be behind NAT:**
When the Caller sends a UDP packet outbound, the NAT creates a **mapping** (pinhole) that allows return traffic:

```
Caller (192.168.1.50:54321)
    │
    ▼
NAT translates to (203.0.1.5:62000) ──────> Listener (198.51.2.10:9000)
    │                                              │
    │  NAT mapping created:                        │
    │  203.0.1.5:62000 ←→ 192.168.1.50:54321     │
    │                                              │
    │  Return traffic from 198.51.2.10:9000        │
    │  to 203.0.1.5:62000 is allowed through       │
    │<─────────────────────────────────────────────│
```

#### Detailed Flow

```
Step 1: Caller resolves the Listener's address
        Target: srt://198.51.2.10:9000

Step 2: Caller creates UDP socket, binds to ephemeral port (OS assigns)
        Local: 0.0.0.0:54321

Step 3: Caller sends Induction handshake to Listener
        ┌─────────────────────────────────────────────┐
        │  UDP: 203.0.1.5:62000 → 198.51.2.10:9000    │
        │  Payload: SRT Handshake (INDUCTION)          │
        └─────────────────────────────────────────────┘

        NAT creates mapping for return traffic.

Step 4: Listener responds with SYN cookie
        ┌─────────────────────────────────────────────┐
        │  UDP: 198.51.2.10:9000 → 203.0.1.5:62000    │
        │  Payload: SRT Handshake (INDUCTION + cookie) │
        └─────────────────────────────────────────────┘

        NAT allows this because it matches the outbound mapping.

Step 5: Caller sends Conclusion with cookie + SRT parameters
Step 6: Listener responds with Conclusion (negotiated params)
Step 7: Connection established. Data flows bidirectionally.
```

#### Caller Retry Logic

```
If no response to handshake:

  t=0ms      Send handshake
  t=250ms    No response → retransmit handshake
  t=750ms    No response → retransmit handshake
  t=1750ms   No response → retransmit handshake
  ...
  t=timeout  Connection timeout (default: 3000ms) → FAIL

  Retry intervals increase exponentially (like TCP backoff)
```

---

### 5.4 Listener Mode — "I wait for connections"

The Listener **binds to a port and waits** for incoming connections. It doesn't know who will connect or when. Multiple Callers can connect to the same Listener (one-to-many).

```
Analogy: Running a phone hotline
  - You (Listener) publish your number and wait
  - Anyone (Caller) can call you
  - You can handle multiple simultaneous calls
```

#### Network Requirements

```
  [Listener]
  Must have:
    ✓ A known, stable IP address (or DNS name)
    ✓ A specific UDP port open in the firewall
    ✓ The port NOT used by another application

  Firewall rule needed:
    ALLOW UDP INBOUND on port 9000 from ANY (or specific source IPs)
```

#### Detailed Flow

```
Step 1: Listener creates UDP socket
        bind(0.0.0.0:9000)  ← listens on all interfaces, port 9000

Step 2: Listener enters LISTENING state
        ┌─────────────────────────────────────────┐
        │  Listener State Machine:                 │
        │                                          │
        │  LISTENING                               │
        │    │                                     │
        │    ├── Recv INDUCTION from Caller A      │
        │    │     → Generate cookie for A         │
        │    │     → Send INDUCTION response       │
        │    │     → (no state allocated yet!)      │
        │    │                                     │
        │    ├── Recv CONCLUSION from Caller A     │
        │    │     → Verify cookie                 │
        │    │     → Allocate connection state      │
        │    │     → Negotiate parameters           │
        │    │     → Send CONCLUSION response       │
        │    │     → State: CONNECTED               │
        │    │                                     │
        │    └── Continue LISTENING for others      │
        └─────────────────────────────────────────┘
```

#### Why No State Until Cookie Verification?

```
Attack scenario WITHOUT cookies:

  Attacker sends 100,000 fake INDUCTION packets from spoofed IPs

  Bad design:  Listener allocates memory for each → memory exhaustion (DoS)

  SRT design:  Listener computes cookie = hash(source_IP, source_port, timestamp)
               No memory allocated. Cookie embedded in response.
               Only when CONCLUSION arrives with valid cookie does
               the Listener allocate connection state.
               Attacker can't receive the cookie (IP is spoofed),
               so they can't complete the handshake.
```

#### Multiple Callers on One Listener (using Stream ID)

```
Listener on port 9000

  Caller A connects with streamid="feed-london"
  Caller B connects with streamid="feed-tokyo"
  Caller C connects with streamid="feed-newyork"

  ┌──────────────────────────────────────────────────┐
  │ Listener Port 9000                                │
  │                                                   │
  │  Connection Table:                                │
  │  ┌──────────┬──────────────────┬──────────────┐  │
  │  │ Socket ID│ Remote Address    │ Stream ID     │  │
  │  ├──────────┼──────────────────┼──────────────┤  │
  │  │ 0x1A2B  │ 203.0.1.5:62000  │ feed-london  │  │
  │  │ 0x3C4D  │ 45.67.89.10:5100 │ feed-tokyo   │  │
  │  │ 0x5E6F  │ 72.14.20.1:8800  │ feed-newyork │  │
  │  └──────────┴──────────────────┴──────────────┘  │
  │                                                   │
  │  Packets demultiplexed by SRT Socket ID in header │
  └──────────────────────────────────────────────────┘
```

The Listener uses the **Destination Socket ID** in each packet's header to route incoming packets to the correct connection — not the source port, which may change with NAT.

---

### 5.5 Rendezvous Mode — "Both sides initiate simultaneously"

Rendezvous mode is used when **neither side has an accessible port** — both are behind NAT/firewalls. Both sides act as Caller AND Listener simultaneously.

```
Analogy: Two people trying to call each other at the same time
  - Both pick up the phone and dial simultaneously
  - Whoever "gets through" first establishes the call
  - Works because both sides have opened their NAT pinholes
```

#### The NAT Traversal Problem

```
  [Encoder]                              [VOS]
  192.168.1.50                           10.0.0.100
       │                                      │
  ┌─────────┐                           ┌─────────┐
  │  NAT A  │                           │  NAT B  │
  │203.0.1.5│                           │198.51.2.10│
  └─────────┘                           └─────────┘

  Problem:
    - Encoder can't send to 10.0.0.100 (private, unreachable)
    - Encoder sends to 198.51.2.10:9000 → NAT B DROPS it
      (NAT B has no mapping for this — nobody inside asked for it)
    - VOS sends to 203.0.1.5:9000 → NAT A DROPS it
      (NAT A has no mapping for this either)

  Neither side can reach the other!
```

#### How Rendezvous Solves It

```
Step 1: Both sides must know each other's public IP:port (exchanged out-of-band)
        Encoder told: "Send to 198.51.2.10:9000"
        VOS told:     "Send to 203.0.1.5:9000"

Step 2: Both sides bind to their configured port and start sending
        handshake packets simultaneously.

Timeline:

t=0ms   Encoder sends HS to 198.51.2.10:9000
        NAT A creates mapping: 203.0.1.5:9000 ←→ 192.168.1.50:9000
        BUT NAT B drops it (no matching mapping yet)

t=0ms   VOS sends HS to 203.0.1.5:9000
        NAT B creates mapping: 198.51.2.10:9000 ←→ 10.0.0.100:9000
        NAT A NOW has a mapping for 203.0.1.5:9000
        → This packet gets through! ✓

t=1ms   Encoder's NEXT HS packet to 198.51.2.10:9000
        NAT B NOW has a mapping for 198.51.2.10:9000
        → This packet gets through! ✓

Both NATs now have pinholes open. Bidirectional traffic flows.
```

```
  ┌──────────┐     punched hole →     ┌──────────┐
  │  NAT A   │ ←──────────────────→   │  NAT B   │
  │203.0.1.5 │     ← punched hole     │198.51.2.10│
  │  :9000   │                         │  :9000   │
  └──────────┘                         └──────────┘
       ↕                                    ↕
  [Encoder]                              [VOS]
```

#### Rendezvous Handshake Detail

```
Side A                                Side B
  │                                    │
  │  HS (WAVEAHAND) ──────────────>    │  (may be dropped by NAT)
  │    <────────────── HS (WAVEAHAND)  │  (may be dropped by NAT)
  │                                    │
  │  (retry — NAT pinholes now open)   │
  │                                    │
  │  HS (WAVEAHAND) ──────────────>    │  ✓ received!
  │    <────────────── HS (WAVEAHAND)  │  ✓ received!
  │                                    │
  │  Both sides detect: "This is       │
  │  a Rendezvous — we both sent       │
  │  WAVEAHAND simultaneously"         │
  │                                    │
  │  HS (CONCLUSION) ─────────────>    │
  │    <───────────── HS (CONCLUSION)  │
  │                                    │
  │  ══════ CONNECTION ESTABLISHED ══════
```

The **WAVEAHAND** state is unique to Rendezvous — it signals "I'm looking for a peer, not acting as a server."

#### Rendezvous Limitations

```
┌────────────────────────────────────────────────────────────────┐
│ IMPORTANT LIMITATIONS:                                          │
│                                                                 │
│ 1. Symmetric NAT:  Rendezvous FAILS with symmetric NAT         │
│    because the NAT assigns different external ports for          │
│    different destinations. The pinhole created for one          │
│    destination doesn't work for traffic from another.           │
│                                                                 │
│ 2. Out-of-band coordination: Both sides must know each         │
│    other's public IP:port BEFORE connecting. This requires     │
│    a signaling mechanism (manual config, STUN server, etc.)    │
│                                                                 │
│ 3. Timing sensitivity: Both sides should start roughly at      │
│    the same time. SRT handles some timing skew with retries,   │
│    but large delays (minutes) may cause timeouts.              │
│                                                                 │
│ 4. One-to-one only: Unlike Listener mode, Rendezvous           │
│    establishes a SINGLE connection between two specific peers. │
│    No multiplexing.                                             │
└────────────────────────────────────────────────────────────────┘
```

---

### 5.6 Mode Comparison

```
                    Caller              Listener            Rendezvous
                    ──────              ────────            ──────────
Initiates:          Yes                 No (waits)          Both
NAT/firewall:       Can be behind NAT   Needs open port     Both behind NAT
Public IP needed:   No                  Yes                 Both (or STUN)
Multiple streams:   N/A                 Yes (Stream ID)     No (1-to-1)
Use case:           Source pushes       Server receives     Peer-to-peer
Security:           Connects out        Exposes port        Minimal exposure
VOS typical role:   Caller (pulls)      Listener (receives) Rare
```

---

### 5.7 Handshake State Machines (Complete)

```
                    CALLER                          LISTENER
                    ──────                          ────────

               ┌──────────┐                    ┌───────────┐
               │  IDLE     │                    │ LISTENING  │
               └─────┬─────┘                    └─────┬─────┘
                     │                                │
            Send INDUCTION                    Recv INDUCTION
                     │                          │
                     ▼                          ▼
              ┌────────────┐            ┌────────────────┐
              │  WAITING   │            │ Compute cookie  │
              │ for resp.  │            │ Send INDUCTION  │
              └──────┬─────┘            │ response        │
                     │                  └────────┬───────┘
             Recv INDUCTION                      │
             response (cookie)             Recv CONCLUSION
                     │                    (with valid cookie)
                     ▼                          │
              ┌────────────┐                    ▼
              │Send         │            ┌────────────────┐
              │CONCLUSION   │            │ Allocate state  │
              │(cookie +    │            │ Negotiate params│
              │ SRT params) │            │ Send CONCLUSION │
              └──────┬─────┘            │ response        │
                     │                  └────────┬───────┘
             Recv CONCLUSION                     │
             response                            │
                     │                           │
                     ▼                           ▼
              ┌────────────┐            ┌────────────────┐
              │ CONNECTED   │            │  CONNECTED      │
              └────────────┘            └────────────────┘


                    RENDEZVOUS
                    ──────────

               Side A                      Side B
          ┌──────────┐                ┌──────────┐
          │  IDLE     │                │  IDLE     │
          └─────┬─────┘                └─────┬─────┘
                │                            │
          Bind to port                 Bind to port
                │                            │
                ▼                            ▼
          ┌──────────┐                ┌──────────┐
          │WAVEAHAND │─── send HS ──→│WAVEAHAND │
          │          │←── send HS ───│          │
          └─────┬─────┘                └─────┬─────┘
                │                            │
          Recv WAVEAHAND              Recv WAVEAHAND
          from peer                   from peer
                │                            │
                ▼                            ▼
          ┌────────────┐              ┌────────────┐
          │CONCLUSION  │── send HS ──→│CONCLUSION  │
          │            │←── send HS ──│            │
          └─────┬──────┘              └──────┬─────┘
                │                            │
          Recv CONCLUSION             Recv CONCLUSION
                │                            │
                ▼                            ▼
          ┌──────────┐                ┌──────────┐
          │CONNECTED  │                │CONNECTED  │
          └──────────┘                └──────────┘
```

---

### 5.8 Security Considerations by Mode

```
┌────────────────┬──────────────────────────────────────────────────┐
│ Mode           │ Security Profile                                  │
├────────────────┼──────────────────────────────────────────────────┤
│ Listener       │ - Exposes a port on the public internet           │
│                │ - Vulnerable to port scanning / discovery         │
│                │ - SYN cookie prevents state-exhaustion DoS        │
│                │ - Passphrase prevents unauthorized streams        │
│                │ - Consider IP allowlisting at firewall level     │
│                │ - Stream ID can be used for access control        │
├────────────────┼──────────────────────────────────────────────────┤
│ Caller         │ - No exposed port (outbound only)                │
│                │ - Lower attack surface                           │
│                │ - But: destination must be trusted               │
│                │ - Passphrase protects content in transit          │
├────────────────┼──────────────────────────────────────────────────┤
│ Rendezvous     │ - No persistently open port                      │
│                │ - Pinholes are temporary and narrow              │
│                │ - Smallest attack surface                        │
│                │ - But: requires pre-shared addressing info       │
│                │ - Passphrase still essential                     │
└────────────────┴──────────────────────────────────────────────────┘
```

---

## 6. TSBPD (Timestamp-Based Packet Delivery)

TSBPD is the mechanism that makes SRT's output smooth and jitter-free. It is what turns SRT from "a retransmission protocol" into "a broadcast-quality transport."

### 6.1 The Problem TSBPD Solves

On the network, packets arrive with **jitter** — irregular spacing caused by routing, queuing, and congestion:

```
Sender sends packets at steady 1ms intervals:

  Pkt1    Pkt2    Pkt3    Pkt4    Pkt5    Pkt6
  |──1ms──|──1ms──|──1ms──|──1ms──|──1ms──|

But receiver gets them with jitter:

  Pkt1  Pkt2      Pkt3 Pkt4         Pkt5  Pkt6
  |─0.5ms─|──3ms──|─0.2ms─|───4ms───|─0.3ms─|

  The spacing is completely distorted.
```

If you hand these packets directly to the decoder at arrival time, the decoder sees **bursts and gaps**:
- Buffer underruns (no data when expected → freeze)
- Buffer overruns (too much data at once → overflow/drop)
- Visible stuttering and audio glitches

---

### 6.2 How TSBPD Works

TSBPD reconstructs the **original sending cadence** at the receiver.

#### Step 1: Sender stamps each packet

```
When the application hands data to SRT:

  SRT Timestamp = current_time - connection_start_time

  Pkt1: timestamp = 0μs       (submitted at connection start)
  Pkt2: timestamp = 1000μs    (submitted 1ms later)
  Pkt3: timestamp = 2000μs    (submitted 2ms later)
  Pkt4: timestamp = 3000μs    (submitted 3ms later)
```

This captures the **exact rhythm** at which the sender produced data.

#### Step 2: Receiver calculates delivery time

```
For each packet:

  Delivery_Time = SRT_Timestamp + Latency + Time_Drift_Correction

Example with Latency = 120ms (120,000μs):

  Pkt1: deliver at   0 + 120,000 = 120,000μs after connection start
  Pkt2: deliver at 1,000 + 120,000 = 121,000μs
  Pkt3: deliver at 2,000 + 120,000 = 122,000μs
  Pkt4: deliver at 3,000 + 120,000 = 123,000μs
```

#### Step 3: Receiver holds packets until their scheduled time

```
                        Latency buffer (120ms)
              ┌──────────────────────────────────────┐
              │                                      │
  Network     │   Reorder     Hold until             │   Application
  (jittery) ──│──→ buffer ──→ scheduled  ──→ deliver │──→ (smooth)
              │               time                   │
              └──────────────────────────────────────┘

Timeline:

  t=50ms    Pkt1 arrives (early — hold for 70ms more)
  t=53ms    Pkt3 arrives (out of order — put in slot 3, hold)
  t=54ms    Pkt2 arrives (put in slot 2, hold)
  t=58ms    Pkt4 arrives (put in slot 4, hold)

  t=120ms   Pkt1's scheduled time → DELIVER Pkt1 ✓
  t=121ms   Pkt2's scheduled time → DELIVER Pkt2 ✓
  t=122ms   Pkt3's scheduled time → DELIVER Pkt3 ✓
  t=123ms   Pkt4's scheduled time → DELIVER Pkt4 ✓

  Output: perfectly spaced at 1ms intervals, exactly as sent.
  All jitter absorbed. All reordering fixed.
```

#### The visual picture

```
SENDER OUTPUT (steady):
  ──●──●──●──●──●──●──●──●──●──●──
    1ms spacing

NETWORK (jittery):
  ●●───●─●────●●──●───●●────●─●──
    irregular, bursty

TSBPD BUFFER (absorbing jitter):
  ┌─────────────────────────────────────────┐
  │ slot1:✓  slot2:✓  slot3:✓  slot4:✓  ... │
  │         held until scheduled time        │
  └─────────────────────────────────────────┘

RECEIVER OUTPUT (smooth again):
  ──●──●──●──●──●──●──●──●──●──●──
    1ms spacing restored
    (just shifted by latency amount)
```

---

### 6.3 TSBPD and ARQ Working Together

TSBPD and ARQ are tightly coupled — the latency buffer serves **both** purposes:

```
              Latency = 120ms
  ├──────────────────────────────────────┤

  Purpose 1: JITTER ABSORPTION
  Packets that arrive early are held.
  Packets that arrive late (but before deadline) are still on time.

  Purpose 2: RETRANSMISSION WINDOW
  Lost packets can be NAKed and retransmitted
  as long as the retransmit arrives before the delivery deadline.

  ├────── ARQ recovery time ──────┤
  ├────── Jitter absorption ──────┤
  Same budget, shared between both functions.
```

#### What happens at the delivery deadline

```
t = scheduled delivery time for Pkt N

  Case 1: Pkt N is in the buffer ✓
    → Deliver to application
    → Normal operation

  Case 2: Pkt N is NOT in buffer (still missing after all NAK retries)
    → TLPKTDROP: skip Pkt N
    → Deliver Pkt N+1 at its scheduled time
    → Decoder sees a gap (minor glitch)
    → Stream does NOT freeze

  TSBPD NEVER waits beyond the deadline.
  This is the fundamental difference from TCP.
```

---

### 6.4 Why Not Just Use a Simple Jitter Buffer?

Traditional jitter buffers (used in VoIP, RTP) are similar but simpler:

```
Simple jitter buffer:
  - Fixed size (e.g., hold 5 packets)
  - Delivers when buffer reaches threshold
  - No sender timestamps — uses arrival time + fixed delay

TSBPD:
  - Time-based, not count-based
  - Uses sender's original timestamps (preserves exact cadence)
  - Integrated with ARQ (shared latency budget)
  - Clock drift correction built in
  - Too-late drop policy
```

```
Simple jitter buffer:
  Sender:   ●──●──●──●──●──●     (1ms intervals)
  Receiver: ●──●─●───●──●──●     (approximately 1ms, but drifts)

TSBPD:
  Sender:   ●──●──●──●──●──●     (1ms intervals)
  Receiver: ●──●──●──●──●──●     (exactly 1ms intervals, guaranteed)
```

This matters for video because the decoder expects a **constant frame rate** — even tiny timing irregularities can cause the decoder's buffer to underrun or overrun.

---

## 7. SRT Timestamp vs PCR/PTS/DTS

### 7.1 The Two Timestamp Layers

SRT packets carry their own timestamp. The MPEG-TS payload inside also contains PCR, PTS, and DTS. These are **completely different timestamps at different protocol layers**.

```
Layer Stack:

┌─────────────────────────────────────────────────┐
│  MPEG-TS Payload                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  PCR (Program Clock Reference)           │    │  ← Decoder clock sync
│  │  PTS (Presentation Timestamp)            │    │  ← When to display frame
│  │  DTS (Decode Timestamp)                  │    │  ← When to decode frame
│  └─────────────────────────────────────────┘    │
├─────────────────────────────────────────────────┤
│  SRT Header                                      │
│  ┌─────────────────────────────────────────┐    │
│  │  SRT Timestamp (μs since connection)     │    │  ← TSBPD scheduling
│  └─────────────────────────────────────────┘    │
├─────────────────────────────────────────────────┤
│  UDP Header                                      │
├─────────────────────────────────────────────────┤
│  IP Header                                       │
└─────────────────────────────────────────────────┘
```

### 7.2 SRT Timestamp

The SRT timestamp is a **relative time in microseconds** since the SRT connection was established. It has **nothing to do with the content** inside the packet.

```
Connection established at t=0

Packet 1 submitted to SRT at t=5000μs    → SRT timestamp = 5000
Packet 2 submitted to SRT at t=10000μs   → SRT timestamp = 10000
Packet 3 submitted to SRT at t=15000μs   → SRT timestamp = 15000
```

**How it's set:** When the application (encoder) hands a chunk of data to the SRT library for sending, SRT stamps it with `current_time - connection_start_time`. The sender's SRT library does this automatically — not the application.

**What it's used for:** Exclusively for **TSBPD** — the receiver uses it to schedule when to deliver each packet to the receiving application.

```
Sender side:                          Receiver side:

App gives data to SRT                 SRT receives packet
  → SRT stamps: T = 5000μs             → Read timestamp: T = 5000μs
  → Sends packet                        → Delivery time = T + Latency
                                         → Deliver at 5000 + 200000 = 205000μs
                                           after connection start
```

### 7.3 PCR — Completely Different Purpose

PCR lives **inside the MPEG-TS payload**. SRT is completely unaware of it — SRT treats the TS payload as **opaque bytes**.

```
SRT sees this:
┌──────────────────────────────────────────┐
│ SRT Header │ opaque payload bytes...      │
│ (seq, ts)  │ [SRT doesn't parse this]     │
└──────────────────────────────────────────┘

But inside those bytes, the MPEG-TS structure contains:
┌──────────────────────────────────────────────────────┐
│ TS Header │ Adaptation Field    │ PES / Section Data  │
│ (PID,CC)  │ ┌─────────────────┐│                     │
│           │ │ PCR: 27MHz clock ││ PTS/DTS inside PES  │
│           │ │ (42-bit + ext)   ││                     │
│           │ └─────────────────┘│                     │
└──────────────────────────────────────────────────────┘
```

### 7.4 What Each MPEG-TS Timestamp Does

```
┌───────┬─────────────┬──────────────────────────────────────────────┐
│ Field │ Clock Base   │ Purpose                                      │
├───────┼─────────────┼──────────────────────────────────────────────┤
│ PCR   │ 27 MHz       │ Encoder's clock reference. The decoder       │
│       │ (90kHz base  │ locks its own 27MHz clock to this.           │
│       │  + 300 ext)  │ Carried in Adaptation Field of specific PIDs.│
│       │              │ Sent every 40ms (max) per MPEG spec.         │
├───────┼─────────────┼──────────────────────────────────────────────┤
│ PTS   │ 90 kHz       │ "Display this frame at this time"            │
│       │              │ Used by the decoder to schedule presentation.│
│       │              │ Every video/audio frame has one.             │
├───────┼─────────────┼──────────────────────────────────────────────┤
│ DTS   │ 90 kHz       │ "Decode this frame at this time"             │
│       │              │ Only present when decode order ≠ display     │
│       │              │ order (B-frames in video).                   │
└───────┴─────────────┴──────────────────────────────────────────────┘
```

### 7.5 Why They Must Be Separate

```
PCR/PTS are about CONTENT timing:
  "This video frame should be displayed at exactly 00:01:23.456"
  They are absolute, content-specific, and meaningful to the decoder.

SRT timestamp is about TRANSPORT timing:
  "This packet was submitted 5ms after the connection started"
  It is relative, content-agnostic, and meaningful only to TSBPD.
```

#### Why SRT can't just use PCR:

```
Problem 1: SRT is content-agnostic
  SRT can carry MPEG-TS, but also raw H.264, SRT-over-SRT, or any byte stream.
  PCR only exists in MPEG-TS. SRT can't depend on it.

Problem 2: PCR is not in every packet
  PCR appears only in specific TS packets (typically every 40ms on one PID).
  But SRT needs a timestamp on EVERY packet for TSBPD scheduling.

Problem 3: PCR can have discontinuities
  Channel switches, ad insertions, and splicing cause PCR jumps.
  SRT's transport timing must be continuous and monotonic.

Problem 4: Parsing overhead
  If SRT had to parse MPEG-TS to find PCR, it would add latency,
  complexity, and break the clean layer separation.
```

### 7.6 How They Interact at the Receiver

```
Sender                    Network              VOS Receiver
───────                   ───────              ────────────

[Encoder produces         [SRT packets         [SRT layer]
 TS packets with          transit the           │
 PCR/PTS/DTS inside]      internet]             ├─ TSBPD uses SRT timestamp
                                                │  to deliver packet at the
[SRT stamps each                                │  right transport time
 packet with SRT                                │
 timestamp]                                     ├─ Delivers raw bytes to
                                                │  application layer
                                                │
                                                ▼
                                               [TS Demux layer]
                                                │
                                                ├─ Parses TS packets
                                                ├─ Extracts PCR → feeds to
                                                │  clock recovery module
                                                ├─ Extracts PTS/DTS → feeds
                                                │  to decoder for A/V sync
                                                │
                                                ▼
                                               [Decoder]
                                                │
                                                ├─ Decodes at DTS time
                                                ├─ Displays at PTS time
                                                └─ Clock locked to PCR
```

#### The full timing chain

```
Encoder clock (27MHz)
    → PCR embedded in TS
        → TS packetized into SRT packets
            → SRT timestamp = "when this was submitted to SRT"
                → Transit over network (jitter, reordering, loss)
                    → SRT receiver: TSBPD restores original pacing
                        → TS bytes delivered at steady rate
                            → Demuxer extracts PCR
                                → Decoder clock locks to PCR
                                    → Frames displayed at PTS

Two clocks recovered:
  1. SRT timestamp → restores transport pacing (network jitter removed)
  2. PCR → restores encoder's media clock (decoder syncs to source)
```

### 7.7 Summary

| | SRT Timestamp | PCR |
|---|---|---|
| **Layer** | Transport (SRT header) | Content (MPEG-TS adaptation field) |
| **Clock** | μs since connection start | 27 MHz (absolute or relative to stream) |
| **Set by** | SRT library automatically | Encoder/muxer |
| **Present in** | Every SRT packet | Only specific TS packets (~every 40ms) |
| **Used for** | TSBPD — when to deliver to app | Decoder clock recovery |
| **Content-aware** | No (opaque payload) | Yes (tied to video/audio timing) |
| **Monotonic** | Always | Can have discontinuities |

SRT deliberately operates **below** the content layer. It treats the payload as opaque bytes and uses its own timestamp purely to reconstruct the sending cadence at the receiver. PCR, PTS, and DTS are intact inside those bytes for the decoder to use independently.

---

## 8. Clock Drift and Correction

### 8.1 The Problem

The sender and receiver have **different physical clock crystals** that tick at slightly different rates. Without correction, the TSBPD buffer would slowly drain or fill over time.

```
Sender's clock crystal:   exactly 1,000,000 ticks per second
Receiver's clock crystal: slightly off (always, in real hardware)
```

---

### 8.2 Case 1: Receiver Clock is FASTER (runs ahead)

Receiver's clock gains **1μs every 10ms** (100 ppm drift — realistic for cheap oscillators).

Setup: sender produces one packet every 10ms. TSBPD latency is 120ms.

```
Sender's perspective (stamps packets):
  Pkt 1:  timestamp =     0ms
  Pkt 2:  timestamp =    10ms
  Pkt 3:  timestamp =    20ms
  ...

Scheduled delivery = timestamp + 120ms latency

Time as measured by SENDER vs RECEIVER:

  Sender says    Receiver's clock    Difference
  ──────────     ────────────────    ──────────
     0ms              0ms              0
   100ms            100.01ms          +0.01ms
  1,000ms          1,000.1ms          +0.1ms
 10,000ms         10,001ms            +1ms
 60,000ms         60,006ms            +6ms        (after 1 minute)
600,000ms        600,060ms            +60ms       (after 10 minutes)
```

#### What happens over time

```
At t=10 minutes, sender sends Pkt #60000:
  Sender stamps it:    timestamp = 600,000ms
  Scheduled delivery:  600,000 + 120 = 600,120ms (receiver clock)

But receiver's clock already reads:  600,060ms when the packet arrives

  EXPECTED wait in buffer: 120ms
  ACTUAL wait in buffer:   600,120 - 600,060 = 60ms  ← ONLY 60ms!

  The packet is delivered 60ms EARLIER than intended.
```

```
At t=20 minutes:
  Receiver clock is ahead by 120ms

  Sender stamps:       timestamp = 1,200,000ms
  Scheduled delivery:  1,200,000 + 120 = 1,200,120ms

  Receiver clock reads: 1,200,120ms when packet arrives
  Wait in buffer: 0ms  ← ZERO buffer time!

At t=21 minutes:
  Receiver clock is ahead by 126ms

  Scheduled delivery:  1,260,000 + 120 = 1,260,120ms
  Receiver clock reads: 1,260,126ms when packet arrives

  1,260,126 > 1,260,120  ← receiver thinks delivery time ALREADY PASSED!
  Buffer is EMPTY when application asks for next packet → UNDERRUN
```

#### Visualizing the buffer draining

```
Buffer fill level over time:

100% │
     │ ████████
     │ █████████████
     │ ██████████████████
 50% │ ███████████████████████
     │ ████████████████████████████
     │ █████████████████████████████████
     │ ██████████████████████████████████████
  0% │──────────────────────────────────────────── UNDERRUN!
     └──────────────────────────────────────────→
     0min     5min     10min    15min    20min

  Receiver clock runs fast
  → "current time" overtakes scheduled delivery times
  → packets are released earlier and earlier
  → effective buffer shrinks each minute
  → eventually buffer is empty → decoder starves → FREEZE
```

---

### 8.3 Case 2: Receiver Clock is SLOWER (falls behind)

Same drift rate, opposite direction. Receiver **loses** 1μs every 10ms.

```
Time as measured by SENDER vs RECEIVER:

  Sender says    Receiver's clock    Difference
  ──────────     ────────────────    ──────────
     0ms              0ms              0
   100ms             99.99ms          -0.01ms
  1,000ms            999.9ms          -0.1ms
 10,000ms          9,999ms            -1ms
 60,000ms         59,994ms            -6ms
600,000ms        599,940ms            -60ms      (after 10 minutes)
```

#### What happens over time

```
At t=10 minutes, sender sends Pkt #60000:
  Sender stamps it:    timestamp = 600,000ms
  Scheduled delivery:  600,000 + 120 = 600,120ms (receiver clock)

But receiver's clock only reads:  599,940ms when the packet arrives

  EXPECTED wait in buffer: 120ms
  ACTUAL wait in buffer:   600,120 - 599,940 = 180ms  ← 180ms, not 120!

  The packet sits in the buffer 60ms LONGER than intended.
```

```
At t=20 minutes:
  Receiver clock is behind by 120ms
  Wait in buffer: 240ms  ← DOUBLE the intended buffer!

At t=30 minutes:
  Receiver clock is behind by 180ms
  Wait in buffer: 300ms

At t=40 minutes:
  Wait in buffer: 360ms ← buffer keeps growing!
```

```
Buffer fill level over time:

     │                                          ████ OVERFLOW!
     │                                     █████████
     │                                ██████████████
     │                           ███████████████████
100% │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  buffer max
     │                      ████████████████████████
     │                 █████████████████████████████
 50% │            ██████████████████████████████████
     │       ███████████████████████████████████████
     │  ████████████████████████████████████████████
     └──────────────────────────────────────────────→
     0min     5min     10min    15min    20min

  Receiver clock runs slow
  → scheduled delivery times drift further into the "future"
  → packets held longer and longer
  → new packets keep arriving but old ones aren't released
  → buffer grows without bound → OVERFLOW → packets dropped
```

---

### 8.4 Side-by-Side Comparison

What happens to Pkt #60000 (sent at the 10-minute mark):

```
                        Normal        Receiver FAST    Receiver SLOW
                        ──────        ──────────────   ──────────────
Sender timestamp:       600,000ms     600,000ms        600,000ms
Scheduled delivery:     600,120ms     600,120ms        600,120ms
Receiver clock when
  packet arrives:       600,000ms     600,060ms        599,940ms
                                       (+60ms ahead)    (-60ms behind)

Time held in buffer:    120ms         60ms             180ms
                        (correct)     (too short!)     (too long!)

Retransmission time
  available:            120ms         60ms             180ms
                        (good)        (risky — less    (wasteful —
                                       room for ARQ)    holding memory)
```

---

### 8.5 How Drift Correction Fixes This

SRT measures drift using ACK/ACKACK round-trips and adjusts delivery times gradually:

```
Receiver                          Sender
  │                                 │
  │── ACK (recv_time=T1) ────────>  │
  │                                 │── ACKACK (send_time=T2) ──>
  │  <── ACKACK arrives at T3 ──── │
  │                                 │
  │  RTT = T3 - T1                  │
  │  One-way delay ≈ RTT / 2        │
  │  Clock offset ≈ T2 - T1 - RTT/2│
```

SRT tracks this offset over time and applies a gradual drift correction:

```
Without correction:
  deliver_at = sender_timestamp + latency

With correction:
  deliver_at = sender_timestamp + latency + drift_offset

  Where drift_offset is continuously updated:

  Minute 0:   drift_offset =  0ms      (no correction needed)
  Minute 5:   drift_offset = +30ms     (compensating for fast clock)
  Minute 10:  drift_offset = +60ms     (compensating more)
  Minute 20:  drift_offset = +120ms    (fully compensated)

  Buffer stays at steady 120ms regardless of how long the stream runs.
```

```
Buffer fill level WITH drift correction:

     │
100% │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
     │
     │  ████████████████████████████████████████████████
 50% │  ████████████████████████████████████████████████  ← STABLE
     │  ████████████████████████████████████████████████
     │
  0% │──────────────────────────────────────────────────
     └──────────────────────────────────────────────────→
     0min     5min     10min    15min    20min   forever

  Drift correction keeps buffer fill constant.
  Stream can run indefinitely without underrun or overflow.
```

Without drift correction, **no SRT stream could run longer than a few minutes to hours** before the buffer either drains or overflows, depending on which clock is faster.

---

## 9. Encryption

### 9.1 Architecture

```
Sender                                          Receiver
  |                                                |
  |  [Passphrase] → PBKDF2 → KEK                  |
  |  [Random SEK generated]                        |
  |  [SEK wrapped with KEK → KM message]           |
  |                                                |
  |  KM message exchanged during handshake ------->|
  |                                                |
  |  [Passphrase] → PBKDF2 → KEK (same)           |
  |  [Unwrap SEK using KEK]                        |
  |                                                |
  |  [Payload + AES-CTR(SEK)] ---- over UDP -----> |
  |                          [Decrypt with SEK]    |
```

### 9.2 Key Hierarchy

```
┌─────────────────────────────────────────────────────┐
│ Passphrase (10-79 characters, pre-shared)            │
│     │                                                │
│     ▼  PBKDF2 (Password-Based Key Derivation)        │
│                                                      │
│ KEK (Key Encryption Key)                             │
│     │                                                │
│     ▼  AES Key Wrap (RFC 3394)                       │
│                                                      │
│ SEK (Session Encryption Key)                         │
│     │  - Randomly generated per session              │
│     │  - Periodically rotated (even/odd keys)        │
│     │                                                │
│     ▼  AES-CTR encryption                            │
│                                                      │
│ Encrypted Payload                                    │
└─────────────────────────────────────────────────────┘
```

### 9.3 Key Rotation

SRT maintains **two SEKs** (even and odd) for seamless rotation:

```
Time ──────────────────────────────────────────────→

  Using SEK-even          Transition         Using SEK-odd
  ████████████████████  ████████████████  ████████████████████
                        │              │
                   New SEK-odd      Switch to
                   distributed      SEK-odd
                   (wrapped in KEK)

  No interruption in the encrypted stream.
  Old key retired, new key generated for next rotation.
```

---

## 10. SRT in the VOS Architecture

### 10.1 End-to-End Flow

```
                          ┌─────────────────────────────┐
                          │          VOS Platform        │
                          │                              │
[Remote Encoder] ──SRT──> │  [SRT Receiver/Demux]        │
  (HEVC/H.264            │         │                    │
   + AAC/MPEG Audio      │         ▼                    │
   in MPEG-TS)           │  [TS Demultiplexer]          │
                          │         │                    │
                          │    ┌────┴────┐               │
                          │    ▼         ▼               │
                          │ [Video]   [Audio]            │
                          │ [Decode]  [Decode]           │
                          │    │         │               │
                          │    ▼         ▼               │
                          │ [Processing / Transcoding]   │
                          │         │                    │
                          │         ▼                    │
                          │  [Output: HLS/DASH/UDP/etc]  │
                          └─────────────────────────────┘
```

### 10.2 Practical VOS Deployment Patterns

#### Pattern 1: VOS as Listener (most common)

```
  [Remote Encoder A] ──── Caller ────→  ┌──────────────┐
                                         │  VOS          │
  [Remote Encoder B] ──── Caller ────→  │  Listener     │
                                         │  Port 9000    │
  [Remote Encoder C] ──── Caller ────→  │              │
                                         │  streamid     │
                                         │  routing      │
                                         └──────────────┘

  VOS config:
    Mode: Listener
    Port: 9000
    Passphrase: "s3cur3_k3y_2024"
    Latency: 200ms

  Encoder config:
    Mode: Caller
    Target: srt://vos-public-ip:9000?streamid=feed-name
    Passphrase: "s3cur3_k3y_2024"
    Latency: 120ms

  Negotiated latency: max(200, 120) = 200ms
```

**Why this pattern?**
- VOS is in a data center with a **static public IP** and open firewall
- Encoders are in the field (arenas, studios) behind unpredictable NATs
- Encoders can connect/disconnect dynamically; VOS just keeps listening
- **Stream ID** lets VOS route different feeds to different pipelines on one port

#### Pattern 2: VOS as Caller (pull mode)

```
  ┌──────────────┐          ┌──────────────┐
  │  VOS          │          │  Cloud        │
  │  Caller       │── pull ──│  Transcoder   │
  │               │          │  Listener     │
  │               │          │  Port 4900    │
  └──────────────┘          └──────────────┘
```

**Why this pattern?**
- The source is a well-known server with a public endpoint
- VOS controls when to start/stop receiving
- Useful for scheduled events where VOS connects at airtime

#### Pattern 3: Rendezvous (rare for VOS)

```
  [Encoder behind NAT] ←──── Rendezvous ────→ [VOS behind NAT]
       :5000                                        :5000
```

**Why rare?**
- VOS is almost always in a data center with accessible ports
- Rendezvous adds complexity (timing, STUN coordination)
- No Stream ID multiplexing

---

## 11. Critical Parameters for VOS Ingestion

| Parameter | Typical Value | Purpose |
|---|---|---|
| **Latency** | 120ms – 2000ms | Retransmission window; set based on RTT |
| **Max Bandwidth** | `-1` (auto) or explicit | Overhead budget for retransmissions |
| **Passphrase** | 10-79 chars | Shared secret for AES encryption |
| **Key Length** | 16/24/32 (128/192/256-bit) | AES key size |
| **Stream ID** | string | Identifies the stream on a shared listener |
| **Peer Idle Timeout** | 5000ms | How long to wait before declaring peer dead |
| **Too-Late Drop** | enabled | Drop packets that arrive past their playout time |
| **Connection Timeout** | 3000ms | Handshake timeout |

**Latency tuning rule of thumb**: latency >= **4x RTT** to allow multiple retransmission attempts.

---

## 12. Monitoring & Troubleshooting

Key SRT statistics to watch on the VOS receiver side:

| Statistic | Healthy | Problem Indicator |
|---|---|---|
| **RTT** | Stable, low variance | Spikes = network path instability |
| **Packet Loss (post-ARQ)** | 0% | >0% = latency too low for this network |
| **Retransmit Rate** | Low, proportional to loss | High retransmit + no final loss = SRT working well |
| **Send/Recv Buffer** | Stable fill level | Growing = latency too low; draining = clock issue |
| **Bandwidth** | Matches video bitrate + ~25% overhead | Much higher = excessive retransmissions |
| **Dropped Packets** | 0 | >0 = packets arriving past TSBPD deadline |

```
Troubleshooting Decision Tree:

  Stream has glitches?
    │
    ├── Check "Packets Lost" (post-recovery)
    │     └── > 0 → Increase latency setting
    │              → Current latency < 4x RTT? Increase it.
    │
    ├── Check RTT
    │     └── High/unstable → Network problem (not SRT's fault)
    │                        → Consider alternate network path
    │
    ├── Check Retransmit Rate
    │     └── Very high → Network has significant packet loss
    │                   → Consider FEC (Forward Error Correction) add-on
    │                   → Or increase bandwidth overhead budget
    │
    └── Check Buffer Level
          └── Growing over time → Clock drift not being corrected
                                → Check SRT version (older versions had bugs)
          └── Draining over time → Same cause, opposite direction
```
