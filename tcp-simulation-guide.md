# TCP Fundamentals — Interactive Simulation Guide

## Overview

The **TCP Fundamentals Interactive Simulation** (`tcp-simulation.html`) is a single-file HTML/JavaScript application that visually demonstrates the core mechanisms of TCP (Transmission Control Protocol). It serves as a **precursor to the SRT simulation** — understand TCP's approach first, then see how SRT improves on it for live video.

| Tab | What It Demonstrates |
|-----|---------------------|
| **3-Way Handshake** | How TCP establishes a connection before any data can flow (SYN, SYN-ACK, ACK) |
| **Transmission & ACK** | Sliding window, cumulative acknowledgments, and how throughput relates to window size and RTT |
| **Retransmission** | RTO timer-based retransmission vs. Fast Retransmit (3 duplicate ACKs), and head-of-line blocking |
| **SACK & Limitations** | Selective Acknowledgment for efficient gap recovery, and why TCP fundamentally fails for live video |

The file has **zero dependencies** — just open it in any modern browser.

---

## Key TCP Concepts

Before diving into the tabs, here are the foundational terms used throughout the simulation:

### RTT (Round-Trip Time)

The time for a packet to travel from sender to receiver **and** for the response to come back. RTT depends on physical distance and network conditions:

| Route | Typical RTT |
|-------|-------------|
| Same city | ~5–20 ms |
| Same continent | ~30–80 ms |
| Cross-continent | ~100–200 ms |
| Intercontinental | ~200–400 ms |

RTT matters because TCP's recovery mechanisms are all measured in multiples of RTT.

### SEQ (Sequence Number)

A counter that tracks **every byte** in a TCP connection. Key rules:

- Each side picks a random **Initial Sequence Number (ISN)** at connection start
- Sequence numbers increment by the number of bytes sent (not packets)
- Example: Sending 500 bytes from seq=1001 → next packet starts at seq=1501
- In the simulation, we use ISN=1000 (client) and ISN=5000 (server) for readability

### ACK (Acknowledgment) — Cumulative

TCP ACKs are **cumulative** — the ACK number means "I have received everything up to this byte; send me this one next."

```
ACK 1501 = "I have bytes 1–1500. Send me byte 1501 next."
```

This cumulative design becomes problematic when packets are lost (see Tabs 3 and 4).

### RTO (Retransmission Timeout)

The timer TCP starts after sending a packet. If no ACK arrives before it expires, TCP assumes the packet was lost and retransmits it.

- **Calculated from RTT**: RTO ≈ RTT + safety margin (typically 1–3 seconds initially)
- **Exponential backoff**: If retransmit also fails, RTO doubles each time (1s → 2s → 4s → 8s)
- This prevents flooding a congested network but causes long delays for live video

---

## Tab 1: 3-Way Handshake (Connection Establishment)

### What Is the 3-Way Handshake?

TCP is a **connection-oriented** protocol — it must establish a logical session before any data can transfer. This process is called **connection establishment** and requires three packets:

```
CLIENT                                    SERVER
  |                                          |
  |--- SYN: seq=1000 ---------------------->|  "I want to connect, starting at byte 1000"
  |                                          |
  |<-- SYN-ACK: seq=5000, ack=1001 ---------|  "OK, I start at 5000, I expect your 1001 next"
  |                                          |
  |--- ACK: seq=1001, ack=5001 ------------>|  "Confirmed, I expect your 5001 next"
  |                                          |
  |           CONNECTION ESTABLISHED         |
  |           (cost: 1.5 round trips)        |
```

### What Happens After the Handshake?

After connection establishment, both sides know where to start and data flows immediately:

```
CLIENT (seq=1001)                          SERVER (seq=5001)
  |                                          |
  |--- DATA: seq=1001, 500 bytes ---------->|  "Here's bytes 1001–1500"
  |                                          |
  |<-- ACK: ack=1501 ----------------------|  "Got it, send me 1501 next"
  |                                          |
  |--- DATA: seq=1501, 500 bytes ---------->|  "Here's bytes 1501–2000"
  |                                          |
  |<-- ACK: ack=2001 + DATA: seq=5001 -----|  "Got it. Also here's MY data"
  |                                          |
  |--- ACK: ack=5501 --------------------->|  "Got your 500 bytes too"
```

Key points:
- **Sequence numbers track bytes, not packets** — sending 500 bytes from seq=1001 means the next packet starts at seq=1501
- **ACK = "next byte I expect"** — when Server sends ack=1501, it means "I have everything up to 1500"
- **Full-duplex** — TCP supports simultaneous data flow in both directions; ACKs can be piggybacked on data packets
- **Sequence numbers never reset** during a connection — they keep incrementing until the connection closes (with a FIN handshake)

### What the Simulation Shows

A message sequence chart with CLIENT (left lifeline) and SERVER (right lifeline). Animated packets fly between them showing the SYN/SYN-ACK/ACK exchange.

### Scenarios

| Scenario | What Happens |
|----------|-------------|
| **Normal Handshake** | Clean SYN → SYN-ACK → ACK. Connection established in 1.5 RTTs. |
| **SYN Lost (Retry)** | SYN is lost mid-transit (fades red). Client waits for RTO (~1–3s), then retransmits. Connection succeeds on retry. |
| **Server Unreachable** | All SYNs are lost. RTO doubles each attempt (exponential backoff: ~1s, ~2s, ~4s). Connection fails after ~7+ seconds of dead time. |

### Controls

| Control | Range | Default | Effect |
|---------|-------|---------|--------|
| Scenario | 3 buttons | Normal | Selects which handshake scenario to animate |
| RTT | 50–500 ms | 150 ms | Controls visual packet flight speed |

### Stats

| Stat | Meaning |
|------|---------|
| Status | Idle / SYN Sent / SYN-ACK Received / ESTABLISHED / FAILED |
| Round Trips | Count of half-RTTs completed (1.5 for normal handshake) |
| Client Seq | Current client sequence number |
| Server Seq | Current server sequence number |
| SYN Retries | Number of SYN retransmissions (0 for normal, 1+ for loss scenarios) |

### SRT Comparison

In SRT, a similar handshake occurs between **Caller** and **Listener**, but SRT's handshake is richer — it also negotiates:
- **Latency buffer size**
- **Encryption keys** (if AES encryption is enabled)
- **Stream ID** (for multiplexing multiple streams on one port)

---

## Tab 2: Transmission & ACK (Sliding Window)

### What Is the Sliding Window?

After the handshake, TCP doesn't send one packet and wait — that would be painfully slow. Instead, it uses a **sliding window**: the sender can have up to *window size* packets in flight simultaneously without waiting for ACKs.

```
Segments:  [1][2][3][4][5][6][7][8][9][10]...
            ✓  ✓  ✓  ────window────  not yet sendable
           acked     [4][5][6][7]
                      in flight
```

As ACKs arrive, the window slides forward:

```
After ACK 5: [1][2][3][4][5][6][7][8][9][10]...
              ✓  ✓  ✓  ✓  ────window────
                      acked  [5][6][7][8]
                              in flight
```

**Throughput = Window Size / RTT**

| Window Size | RTT | Throughput |
|-------------|-----|-----------|
| 1 (stop-and-wait) | 100ms | 10 segments/sec |
| 4 | 100ms | 40 segments/sec |
| 8 | 100ms | 80 segments/sec |
| 4 | 200ms | 20 segments/sec |

### What the Simulation Shows

Two visual areas:

1. **Top: Sliding Window Strip** — A row of numbered segment boxes with a bracket showing the current window position
   - **Teal**: Sent and ACKed (behind the window)
   - **Yellow**: In flight (inside the window)
   - **Red**: Lost
   - **Grey**: Not yet sendable (ahead of the window)

2. **Bottom: Message Sequence Chart** — Data packets (teal pills) fly left-to-right, ACKs (green diamonds) fly right-to-left

### Controls

| Control | Range | Default | Effect |
|---------|-------|---------|--------|
| Window Size | 1–8 | 4 | Number of segments allowed in flight simultaneously |
| Packet Loss % | 0–30% | 0% | Probability of a segment being lost in transit |
| Auto Send | On/Off | On | Automatically sends segments when window has room |

### Stats

| Stat | Meaning |
|------|---------|
| Segments Sent | Total segments transmitted |
| ACKs Received | Total cumulative ACKs received |
| Window | Current window range (e.g., "3–6") |
| In Flight | Number of unacknowledged segments currently in transit |
| Lost | Total segments lost in transit |

### Key Insight

Try setting **Window Size = 1** — this is "stop-and-wait" mode where the sender sends one segment, waits for ACK, then sends the next. Notice how slow it is. Then increase to 8 — throughput jumps dramatically. This is why TCP's window mechanism exists.

**But note what's missing**: TCP has no concept of **packet timing**. It delivers bytes in order, but doesn't preserve the sender's original cadence. SRT adds **TSBPD (Timestamp-Based Packet Delivery)** to reconstruct the original timing — critical for smooth video playback.

---

## Tab 3: Retransmission (RTO vs. Fast Retransmit)

### Two Recovery Mechanisms

When a segment is lost, TCP has two ways to detect and recover:

#### 1. RTO Timer (Slow)

```
SENDER                                    RECEIVER
  |--- SEQ 1 ---------------------------->| ✓
  |--- SEQ 2 ---------------------------->| ✓
  |--- SEQ 3 -------X                     | LOST
  |--- SEQ 4 ---------------------------->| ✓ (but can't deliver — waiting for 3)
  |                                        |
  |  [RTO timer ticking... 1s... 2s...]    |
  |  RTO EXPIRED!                          |
  |                                        |
  |--- SEQ 3 (retransmit) --------------->| ✓ Recovered
  |                                        |
  |  Total delay: ~RTO (1–3 seconds)       |
```

The sender just waits for the timer. During this time, **ALL data after SEQ 3 is blocked** at the receiver (head-of-line blocking).

#### 2. Fast Retransmit (Faster)

```
SENDER                                    RECEIVER
  |--- SEQ 1 ---------------------------->| ✓ ACK 2
  |--- SEQ 2 ---------------------------->| ✓ ACK 3
  |--- SEQ 3 ---------------------------->| ✓ ACK 4
  |--- SEQ 4 -------X                     | LOST
  |--- SEQ 5 ---------------------------->| out of order → DUP ACK 4 (#1)
  |--- SEQ 6 ---------------------------->| out of order → DUP ACK 4 (#2)
  |--- SEQ 7 ---------------------------->| out of order → DUP ACK 4 (#3)
  |                                        |
  |  3 Dup ACKs! FAST RETRANSMIT!         |
  |                                        |
  |--- SEQ 4 (retransmit) --------------->| ✓ Recovered → ACK 8
  |                                        |
  |  Total delay: ~1.5 RTT (much faster)   |
```

The receiver keeps sending duplicate ACKs for the missing sequence. After 3 duplicates, the sender immediately retransmits without waiting for the RTO timer. This is much faster (~1.5 RTT vs. 1–3 seconds).

### Head-of-Line Blocking (The Fundamental Problem)

In both mechanisms, the receiver **cannot deliver any data past the gap** until the missing segment arrives. This is called **head-of-line blocking**:

```
Receiver buffer:  [1][2][3][_][5][6][7]
                            ↑
                     GAP — everything after this is BLOCKED
                     Application cannot read bytes 5, 6, 7
                     until byte 4 arrives
```

For live video at 30fps:
- A 200ms RTO = **6 frozen frames**
- Even fast retransmit at 1.5 RTT with 100ms RTT = **4.5 frozen frames**

### What the Simulation Shows

A message sequence chart with animated packets. The RTO scenario shows a timer bar filling up (teal → yellow → red). The Fast Retransmit scenario shows the duplicate ACK counter incrementing (DUP ACKs: 1/3, 2/3, 3/3 — RETRANSMIT!). Both show the head-of-line blocking indicator on the receiver side.

### Scenarios

| Scenario | Lost Segment | Recovery | Blocking Time |
|----------|-------------|----------|---------------|
| **RTO Timer** | SEQ 3 | Waits for full RTO timeout, then retransmits | ~RTO (1–3 sec) |
| **Fast Retransmit** | SEQ 4 | 3 duplicate ACKs trigger immediate retransmit | ~1.5 RTT |

### Stats

| Stat | Meaning |
|------|---------|
| Segments Sent | Total data segments transmitted |
| Lost | Segments lost in transit |
| Retransmissions | Number of retransmitted segments |
| Dup ACKs | Duplicate ACKs received by sender |
| Blocking Time | How long the receiver was blocked waiting for recovery |
| Mechanism | Which recovery mechanism was used |

### SRT Comparison

SRT avoids these problems entirely:
- **NAK-based ARQ**: Receiver immediately reports what's missing (no waiting for timer or dup ACK threshold)
- **TLPKTDROP**: If a packet arrives too late to be useful, SRT drops it rather than blocking everything
- **No head-of-line blocking**: Application receives data as it arrives, with timestamps to maintain playback timing

---

## Tab 4: SACK & TCP Limitations

### What Is SACK?

Standard TCP ACKs are **cumulative** — "I have everything up to byte N." This creates a problem when multiple segments are lost:

**Without SACK (cumulative ACK only):**
```
Lost: SEQ 4 and SEQ 7

Cycle 1: Receiver sends ACK 4, ACK 4, ACK 4 (dup) → sender retransmits SEQ 4
         Receiver now has 1–6, sends ACK 7
Cycle 2: Receiver sends ACK 7, ACK 7, ACK 7 (dup) → sender retransmits SEQ 7
         Receiver now has everything

Total: 2 recovery cycles × ~3 RTTs each = ~6 RTTs
```

**With SACK:**
```
Lost: SEQ 4 and SEQ 7

Receiver sends: ACK 4 + SACK{5–6, 8–12}
Sender sees: "Ah, only 4 and 7 are missing"
Sender retransmits BOTH SEQ 4 and SEQ 7 simultaneously

Total: 1 recovery cycle = ~1 RTT
```

SACK lets the receiver report **exactly which byte ranges** it has received, so the sender can fill all gaps in one shot.

### The Receiver Buffer

The simulation shows a horizontal row of segment slots:

```
[1][2][3][ ][5][6][ ][8][9][10][11][12]
 ✓  ✓  ✓  ✗  ✓  ✓  ✗  ✓  ✓   ✓   ✓   ✓

 Teal = received in order
 Green = received out of order (SACK reported)
 Red = gap (missing)
 Grey = not yet received
```

### The Application Read Pointer

Even with SACK, the application read pointer is **stuck at the gap**:

```
[1][2][3][ ][5][6][ ][8][9][10][11][12]
         ↑
    BLOCKED — App can read up to byte 3
    Bytes 5–12 are buffered but NOT delivered
    until gaps at 4 and 7 are filled
```

This is **head-of-line blocking** — TCP's fundamental limitation for real-time applications. Even the most efficient recovery (SACK) cannot avoid it because TCP guarantees **in-order delivery**.

### What the Simulation Shows

Two modes compared side by side (via mode buttons):

| Mode | Recovery | Cycles | Time |
|------|----------|--------|------|
| **Without SACK** | One gap at a time, each needing 3 dup ACKs | N cycles (one per gap) | ~3N RTTs |
| **With SACK** | All gaps identified and filled at once | 1 cycle | ~1 RTT |

The receiver buffer visualization updates in real-time, and the "BLOCKED" indicator shows the application read pointer stuck at the first gap.

### Loss Configurations

| Option | Lost Segments | Gaps |
|--------|--------------|------|
| Lose #4 | 4 | 1 gap |
| Lose #4 & #7 | 4, 7 | 2 gaps |
| Lose #4, #7, #10 | 4, 7, 10 | 3 gaps |

With more gaps, the difference between SACK and cumulative ACK becomes dramatic.

### Stats

| Stat | Meaning |
|------|---------|
| Segments Sent | Total segments transmitted |
| Gaps | Number of gaps (lost segments) |
| Retransmissions | Total retransmitted segments |
| Recovery Cycles | 1 for SACK, N for cumulative (where N = number of gaps) |
| Recovery Time | Total RTTs needed for full recovery |
| App Blocked | Duration the application could not read past the first gap |

---

## TCP vs. SRT — Why TCP Fails for Live Video

This simulation exists to set up the "aha moment" when you explore the SRT simulation. Here's the summary:

| Problem | TCP | SRT |
|---------|-----|-----|
| **Connection setup** | 1.5 RTTs before any data (Tab 1) | Similar handshake, but negotiates latency + encryption |
| **Loss recovery** | RTO (slow) or 3 dup ACKs (faster but still delayed) (Tab 3) | **NAK-based ARQ** — receiver immediately reports gaps |
| **Multiple gaps** | One at a time without SACK; all at once with SACK (Tab 4) | **Selective retransmit** — always targets specific missing packets |
| **Head-of-line blocking** | ALL data blocked until gap is filled (Tabs 3, 4) | **No blocking** — TSBPD delivers packets by timestamp |
| **Stale data** | TCP retransmits forever, even if data is too old | **TLPKTDROP** — drops packets past the latency deadline |
| **Congestion response** | Halves window on loss, cutting throughput | SRT maintains steady throughput for constant-bitrate video |
| **Timing** | No concept of packet timing (Tab 2) | **TSBPD** reconstructs sender's original packet cadence |
| **Clock drift** | No awareness | **Drift correction** keeps buffers stable over hours |

### The Key Insight

TCP treats all bytes equally and guarantees in-order delivery. This is perfect for file transfers, web pages, and email — where you'd rather wait than get corrupted data.

But live video has a **time dimension** that TCP ignores:
- A video frame that arrives 500ms late is worse than a frame that never arrives (the decoder can conceal a missing frame, but a late frame causes a stall)
- The receiver needs packets at a steady cadence, not in bursts
- When the network is congested, it's better to skip old data than to slow everything down

**SRT was built specifically for this.** See the **SRT Protocol Interactive Simulation** (`srt-simulation.html`) to explore TSBPD, ARQ, and Clock Drift correction in action.

---

## How the Four Tabs Relate

```
┌──────────────────────────────────────────────────┐
│              TCP Protocol Stack                   │
│                                                   │
│  ┌─────────────┐    ┌──────────────────────────┐  │
│  │  Handshake  │    │  Data Transfer            │  │
│  │  (Tab 1)    │───>│  (Tab 2)                  │  │
│  │             │    │                            │  │
│  │ SYN/SYN-ACK │    │  Sliding window            │  │
│  │ /ACK        │    │  Cumulative ACKs           │  │
│  │ 1.5 RTTs    │    │  Throughput = win/RTT      │  │
│  └─────────────┘    └────────────┬───────────────┘  │
│                                  │ packet loss       │
│                                  ▼                   │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Loss Recovery                                  │ │
│  │  (Tabs 3 & 4)                                   │ │
│  │                                                  │ │
│  │  RTO Timer → slow recovery (~seconds)            │ │
│  │  Fast Retransmit → faster (~1.5 RTT)             │ │
│  │  SACK → efficient (all gaps in ~1 RTT)           │ │
│  │                                                  │ │
│  │  BUT: All cause HEAD-OF-LINE BLOCKING            │ │
│  │  This is TCP's fundamental limitation for video  │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
                         │
                         │ "TCP can't do live video well"
                         ▼
┌──────────────────────────────────────────────────────┐
│              SRT Protocol (see srt-simulation.html)   │
│                                                       │
│  NAK-based ARQ + TSBPD + TLPKTDROP + Drift Correction│
│  = Broadcast-quality video over unpredictable networks│
└──────────────────────────────────────────────────────┘
```

---

## Embedding in Confluence

### Option 1: HTML Macro (Confluence Server/Data Center)

1. Edit the page → Insert → Other macros → HTML
2. Paste the entire contents of `tcp-simulation.html`
3. Save

### Option 2: Iframe (Confluence Cloud or Server)

1. Host the file on an internal web server or S3 bucket
2. Insert an iframe macro:
   ```
   <iframe src="https://your-host/tcp-simulation.html" width="100%" height="900" frameborder="0"></iframe>
   ```

### Option 3: Confluence Attachment + Redirect

1. Attach `tcp-simulation.html` to the Confluence page
2. Link to the attachment — opens in a new browser tab

---

## Technical Details

### Animation Timing Constants

| Constant | Tab 1 | Tab 2 | Tab 3 | Tab 4 |
|----------|-------|-------|-------|-------|
| FLIGHT_MS | 4000 | 4500 | 4000 | 4000 |
| PHASE_GAP/INTERVAL | 4500 | — | 3500 | 3500 |
| SPAWN_MS | — | 1500 | — | — |

### Canvas Rendering

- All four tabs use HTML5 Canvas with `devicePixelRatio` scaling for crisp rendering on Retina/HiDPI displays
- Canvases auto-resize on window resize and tab switch
- Each simulation runs its own independent `requestAnimationFrame` loop inside an IIFE (Immediately Invoked Function Expression) for scope isolation

### File Size

The entire simulation is a single `tcp-simulation.html` file (~1200 lines) with no external dependencies — HTML, CSS, and JavaScript are all inline.
