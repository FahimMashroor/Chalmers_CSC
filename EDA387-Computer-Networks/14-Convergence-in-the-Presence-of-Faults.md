# Convergence in the Presence of Faults: Clock Synchronization

**Course:** EDA387/DIT663 Computer Networks, part 2 (Fault-tolerant Algorithms for Computer Networks), lecture "Convergence in the Presence of Faults (Ch. 6)". The lecture opens with a review of the Network Time Protocol (NTP), whose prelude slides "are based on the ones by Judah Levine" (NIST, Boulder), and then follows Dolev, *Self-Stabilization*, §6.1 "Digital Clock Synchronization" and §6.2 "Stabilization in Spite of Napping".

**Revised:** 21 October 2026

**† Supplementary material:** content that comes only from hidden PowerPoint slides. It may not be part of this year's examinable material. In this lecture the hidden material is the second half of the NTP review: leap seconds, stratum, authentication, the two-way delay measurement, measurement modes and typical performance (sections 1.4 to 1.6). The lecture also says that the details of the constant-states lower-bound proof were skipped this year (section 2.6).

**How to read this guide.** Every part starts with **In one sentence**, the whole idea in one line. Then comes the explanation in small steps, a **Kid example** (the same idea told with everyday things), pictures, and a **Remember** list with what you must know for the exam. **Technical clarification** marks a place where the slides are wrong or unclear, and gives the correct version. **Additional context** adds something useful that is not on the slides.

---

## Glossary

| Term | Full name | Meaning in simple words |
|---|---|---|
| Adjustment | — | A processor "adjusts" its clock in a step if it does **not** simply add 1. The requirement "no adjustment" means: just tick by one. |
| Agreement | — | All (working) processors show the same clock value. |
| Asymmetry error | — | The NTP error caused by the trip to the server and the trip back not taking equally long. |
| Authenticator | — | Extra data added to an NTP packet so the receiver can check who sent it and that it was not changed. |
| Behind | — | In the napping algorithm: P is behind Q when P.count[Q] + 1 (mod 3) = Q.count[P]; it means Q has been active more recently than P. |
| Byzantine fault | — | A faulty processor may do anything, even lie differently to different neighbours. |
| Clock pulse | — | A global "tick" that makes all processors take one step at the same moment (lock-step). |
| Crash failure | — | A processor stops and never comes back. |
| Diameter (d) | — | The largest number of hops between two processors. |
| Digital clock | — | An integer counter kept by each processor, meant to go up by one every pulse. |
| Hybrid faults | — | Transient faults (arbitrary start) **plus** faults that keep happening during and after convergence, such as napping. |
| Leap second | — | An extra second occasionally added to UTC to keep it in step with the Earth's rotation. |
| M | — | The bound on clock values: clocks are counted modulo M (0 … M − 1). |
| MD5 | Message Digest 5 | A hash function that NTP's (older) authentication uses. |
| Napping fault | — | A processor stops for a while and may later resume. A crash is a nap that never ends. |
| NB | Not Behind | The set of processors that are not behind any other processor (called R on one slide, A in quiz 14). |
| NTP | Network Time Protocol | The Internet protocol that synchronizes computer clocks with UTC. |
| Offset | — | How far one clock is ahead of another. |
| Pigeonhole principle | — | If more items than boxes, some box has two items; equally, few items in many boxes leave a large empty stretch. |
| PTP | Precision Time Protocol (IEEE 1588) | Another clock-synchronization protocol, mentioned on a hidden NTP slide. |
| Round-trip delay (Δ) | — | Total time for the request to go to the server and the reply to come back, minus the server's own processing time. |
| Slewing | — | Correcting a clock gradually (running it a bit fast or slow) instead of jumping. |
| SNTP | Simple Network Time Protocol | A simplified NTP client (a single query). |
| Stratum | — | In NTP: the number of links between a server and a UTC reference. |
| Transient fault | — | A short fault that corrupts memory and then stops; it leaves the system in an arbitrary configuration. |
| UDP | User Datagram Protocol | A connectionless transport protocol with no delivery guarantee. NTP runs over UDP port 123. |
| Uniform algorithm | — | All processors run the same program and have no IDs. |
| UTC | Coordinated Universal Time | The world's reference time scale; it replaced Greenwich Mean Time (GMT). |
| Wait-free | — | A working processor makes progress no matter what the others do, even if all n − 1 others are faulty. |

## Contents

- [Before you start: the words you need](#before-you-start-the-words-you-need)
- [1. The Network Time Protocol (NTP)](#1-the-network-time-protocol-ntp)
  - [1.1 What this lecture is about](#11-what-this-lecture-is-about)
  - [1.2 NTP: message format and transport](#12-ntp-message-format-and-transport)
  - [1.3 NTP: the time datum](#13-ntp-the-time-datum)
  - [1.4 † NTP: other parameters, authentication](#14--ntp-other-parameters-authentication)
  - [1.5 † NTP: the two-way protocol and its error](#15--ntp-the-two-way-protocol-and-its-error)
  - [1.6 † NTP: measurement modes and performance](#16--ntp-measurement-modes-and-performance)
- [2. Digital clock synchronization (Dolev §6.1)](#2-digital-clock-synchronization-dolev-61)
  - [2.1 Digital clock synchronization: the problem](#21-digital-clock-synchronization-the-problem)
  - [2.2 The unbounded algorithm (max + 1)](#22-the-unbounded-algorithm-max--1)
  - [2.3 Why unbounded clocks are a problem](#23-why-unbounded-clocks-are-a-problem)
  - [2.4 Bounded version I: max, modulo (n + 1)d + 1](#24-bounded-version-i-max-modulo-n--1d--1)
  - [2.5 Bounded version II: min, modulo 2d + 1](#25-bounded-version-ii-min-modulo-2d--1)
  - [2.6 A constant number of states is impossible](#26-a-constant-number-of-states-is-impossible)
- [3. Stabilization in spite of napping (Dolev §6.2)](#3-stabilization-in-spite-of-napping-dolev-62)
  - [3.1 Napping faults: wait-free self-stabilization](#31-napping-faults-wait-free-self-stabilization)
  - [3.2 The unbounded algorithm with k = 1](#32-the-unbounded-algorithm-with-k--1)
  - [3.3 The bounded algorithm: rock, paper, scissors](#33-the-bounded-algorithm-rock-paper-scissors)
  - [3.4 Theorem 6.1: the algorithm works with k = 2](#34-theorem-61-the-algorithm-works-with-k--2)
- [4. Summary and review](#4-summary-and-review)
  - [4.1 Summary and review questions](#41-summary-and-review-questions)
  - [4.2 Recap](#42-recap)
  - [4.3 Old exam questions on this lecture](#43-old-exam-questions-on-this-lecture)
  - [4.4 Related quiz topics not on the slides](#44-related-quiz-topics-not-on-the-slides)
- [Revision notes and references](#revision-notes-and-references)

## Before you start: the words you need

- **Clock value:** a number a processor keeps, like the time shown on a watch.
- **Pulse:** a common "tick" that reaches every processor at the same moment (like a school bell).
- **Synchronized:** all clocks show the **same** value, and all go up by **one** at every pulse.
- **Diameter d:** the largest number of hops between any two processors.
- **mod M (modulo):** counting on a dial with M numbers: after M − 1 comes 0 again.
- **Transient fault:** a one-time mess (memory gets random values), then things work again.
- **Napping fault:** a processor falls asleep (stops) for a while and later wakes up again, with its old values.
- **Crash fault:** a processor stops forever (a nap that never ends).

**The goal of clock synchronization, in a picture:**

```mermaid
flowchart LR
  A["messy start:<br/>clocks 4, 5, 8"] -- "a few pulses" --> B["all equal:<br/>8, 8, 8"] -- "every pulse" --> C["all +1 together:<br/>0, 0, 0 (mod 9)"]
```

> **Kid example.** Every classroom has its own wall clock, and the school bell rings once a minute for everybody. The clocks might all show different times after a power cut. The question of this lecture: if each classroom may only look at the clocks of the rooms next door, can they all end up showing the same time, and keep ticking together? And what if some rooms "fall asleep" now and then?

## 1. The Network Time Protocol (NTP)

**What this chapter is about:** how the real Internet keeps clocks right: NTP's message format, its 64-bit time value, and how it measures and limits the error caused by network delay.

### 1.1 What this lecture is about

**In one sentence:** after a look at NTP, we study simple clock synchronization algorithms that self-stabilize even when faults keep happening during the run.

The plan of the lecture:

- review the **Network Time Protocol (NTP)**;
- then study **simpler clock synchronization algorithms**;
- the focus: **integrating self-stabilization with other fault models, such as crash failures, that can occur during the execution and not only before the first configuration.**

Until now, "self-stabilizing" meant: faults happen, then stop, and the system recovers from whatever state they left. This chapter adds faults that **keep happening** (here, processors that stop and resume) while the system must still converge.

The chapter roadmap:

- **6.1 Digital Clock Synchronization**
- **6.2 Stabilization in Spite of Napping**

```mermaid
flowchart LR
  A["before: faults happen,<br/>THEN stop, THEN recover"] ~~~ B["now: faults (naps, crashes)<br/>keep happening WHILE<br/>the system converges"]
```

**Remember:**

- Plan: **NTP**, then **simple clock synchronization** (§6.1), then **napping** (§6.2).
- New focus: self-stabilization **combined with faults that keep happening** during the execution.

### 1.2 NTP: message format and transport

**In one sentence:** NTP sends small UDP packets with no delivery guarantee, which is fine because it repeats its measurements all the time.

**Message format and parameters.**

- The format is defined in **RFC 1305**.
- **Packets are UDP/IP**:
  - checksum-based error detection;
  - simple connectionless service;
  - **No guarantee of delivery**;
  - **No detection of lost packets**;
  - "Like mailing a letter to a recipient who does not expect it".
- **Message authentication is optional**: an **authenticator** is added to the message.

This is fine for time: NTP does not need every packet. It relies on **repeated exchanges** and statistics, and a fresh time value matters more than a retransmitted old one.

**Transport.**

- **Independent of the physical layer in principle**, but **performance depends on the physical layer in practice** (speed, delay and loss of the network affect accuracy).
- **All versions of NTP can use IPv4; newer versions also support IPv6.**

**Technical clarification.** The lecture's explanation says "NTP employs a checksum mechanism ... calculated over the entire NTP packet". The checksum belongs to **UDP** (and IP), not to NTP itself; the lecture's own list correctly puts it under "Packets are UDP/IP". NTP's own integrity check is the optional authenticator.

**Additional context.** RFC 1305 is NTP version 3 (1992). The current version, NTPv4, is defined in RFC 5905 (2010). NTP uses UDP port 123 (section 1.6 † says so too).

> **Kid example.** NTP is like sending postcards to ask "what time is it?". A postcard may get lost and nobody tells you, but you keep sending one every few minutes, so losing one does not matter. An old answer that arrives late is useless anyway; a fresh one is what you want.

**Remember:**

- Format: **RFC 1305** (NTPv3); today **RFC 5905** (NTPv4). **UDP port 123**.
- UDP: checksum, connectionless, **no delivery guarantee**, **no loss detection**.
- Authentication is **optional** (an authenticator). The checksum belongs to **UDP**, not NTP.
- Works over **IPv4**; newer versions also **IPv6**. Accuracy depends on the physical network in practice.

### 1.3 NTP: the time datum

**In one sentence:** an NTP time is 64 bits: 32 for whole seconds since 1900 and 32 for the fraction, so it is very precise but wraps around in 2036.

**The NTP time datum:**

- The reference scale is **UTC** (it used to be called Greenwich Mean Time).
- Time values are **64 bits** long:
  - **Seconds since 1900.0**: 32 bits, unsigned;
  - **Fraction of a second**: 32 bits, unsigned.
- **Dynamic range: 136+ years**, so there is a **rollover in 2036**.
- **Resolution: 2⁻³² s ≈ 232 picoseconds (ps).** For comparison, a common 3.0 GHz CPU needs about 330 ps for one cycle.
- A time of **0.0** is usually treated as an **error**.
- "What can go wrong?"

| Field | Bits | Range or step |
|---|---|---|
| Seconds since 1 January 1900 | 32 | 2³² s ≈ 136 years, so it wraps in 2036 |
| Fraction of a second | 32 | step 2⁻³² s ≈ 0.23 ns |

**What can go wrong:** after the **2036 rollover**, software that does not handle it would think it is 1900 again (wrong time order, expired certificates, broken logs). A time of **0.0** would mean "exactly 1 January 1900", which is never a real timestamp, so it is used as "no valid time".

**Additional context (arithmetic check).** 2³² s = 4,294,967,296 s ≈ 136.1 years, and 1900 + 136.1 ≈ 2036 (the exact date is 7 February 2036). 2⁻³² s ≈ 0.233 ns = 233 ps. One cycle at 3.0 GHz = 1/3,000,000,000 s ≈ 333 ps.

```mermaid
flowchart LR
  A["64-bit NTP time"] --> S["32 bits: SECONDS<br/>since 1 Jan 1900<br/>(wraps in 2036)"]
  A --> F["32 bits: FRACTION<br/>step 2⁻³² s ≈ 232 ps"]
```

> **Kid example.** An old car's distance counter has a fixed number of digits. When it reaches 999 999 it rolls back to 000 000, and the car looks brand new. NTP's seconds counter is the same: it has room for about 136 years from 1900, so in 2036 it rolls over to "1900" again unless software handles it.

**Remember:**

- Reference: **UTC**. **64 bits** = 32 bits seconds since **1900** + 32 bits fraction.
- Range **136+ years** ⇒ rollover in **2036**. Resolution **2⁻³² s ≈ 232 ps** (a 3 GHz CPU cycle ≈ 330 ps).
- Time **0.0** = error ("no valid time").

### 1.4 † NTP: other parameters, authentication

**In one sentence:** † NTP packets also carry leap-second warnings, the stratum (hops from UTC), and an optional MD5 authenticator that proves who sent the packet.

**Some other parameters.**

- **Advance notice of a leap second**: "leap second at end of today" when the flag is set.
- **Positive leap second in progress**: transmit 23:59:59 **twice**, which effectively **stops the system clock** for one second.
- **Slewing methods (Google)**: spread the extra second out instead of repeating one.
- **Clock unsynchronized flag**: "don't trust the time message".
- Health and leap-second information **use the same bits**.

**More parameters.**

- message authenticator (optional);
- **Stratum**, precision and reference signal of the server (reference IP or local clock type). **The stratum is the number of links to UTC, not the quality of the oscillator**;
- administrative data, not completely defined in the specification, which can usually be ignored.

**Authentication: the idea.**

- Keys are identified by an index number into a local file that has the key value (`<Key Number> M <key value>`); the configuration names a `trustedkey`.
- The sender computes the **MD5 hash of key + packet** and appends the result and the key index.
- The receiver detects the additional data, computes the hash with the key with that number, and compares.
- There is no decryption: MD5 has no known inverse.
- **Distribution of keys is a headache.**

**Authentication: how it works.**

- Authentication **only identifies the server** and **guarantees integrity**.
- It does **not** affect accuracy or traceability.
- A "man in the middle" can still **delay** a valid message or **replay** an old one.

**Additional context.** Leap seconds are inserted as an extra second, 23:59:60, usually at the end of 30 June or 31 December. MD5 is no longer considered collision-resistant, so modern NTP deployments prefer stronger hashes or Network Time Security (NTS, RFC 8915). Delay attacks matter because NTP's accuracy depends on measuring the delay (section 1.5).

**How the optional authentication works:**

```mermaid
flowchart LR
  A["sender: MD5(key + packet)<br/>+ key number appended"] --> B["receiver: same key number,<br/>computes MD5 again"] --> C["equal? then the packet is<br/>from the server and unchanged"]
```

> **Kid example.** You and a friend share a secret word. When your friend sends you a note, they also write a "check code" made from the note plus the secret word. You make the same code yourself; if it matches, the note really came from your friend and nobody changed it. But a bully could still hold the note back for an hour before giving it to you, and then the time on it is wrong.

**Remember:**

- † Leap second: advance notice flag; a positive leap second repeats **23:59:59** (or Google-style **slewing**).
- † **Stratum** = number of links to UTC, **not** oscillator quality. Unsynchronized flag = "don't trust".
- † Authentication: **MD5(key + packet)** + key index; identifies the server and guarantees **integrity** only; **key distribution** is a headache; delay and replay are still possible.

### 1.5 † NTP: the two-way protocol and its error

**In one sentence:** † four timestamps give the round-trip delay Δ; assuming the delay is split equally, the error is at most half of Δ.

The client sends a request at **t1** (client clock). The server receives it at **t2** and answers at **t3** (server clock). The client receives the answer at **t4** (client clock). The one-way delay is **d**.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    Note over C: t1 = send time (client clock)
    C->>S: request (one-way delay d)
    Note over S: t2 = receive time, t3 = reply time (server clock)
    S->>C: reply
    Note over C: t4 = receive time (client clock)
```

The formulas:

- **Round-trip delay:** Δ = (t4 − t1) − (t3 − t2). (Total time minus the time the server held the request.)
- **Offset:** the lecture writes t(server) − t(client) = (t1 + d) − t2. When the request arrives, the client's clock shows t1 + d and the server's shows t2.
- **Assume** the delay is the same both ways: d = Δ/2. Then the estimate is (t1 + Δ/2) − t2.
- **If the truth is** d = kΔ with 0 ≤ k ≤ 1, the **error** is (t1 + kΔ − t2) − (t1 + Δ/2 − t2) = **Δ(k − 0.5)**.
- So **MaxError = ±0.5Δ** (k = 1 gives +Δ/2, k = 0 gives −Δ/2).
- The error is at most half the round trip, so **"Smaller delay has smaller asymmetry error."**

**Technical clarification.** With the usual meaning (positive when the server is ahead), the server-minus-client offset is t2 − (t1 + d); the lecture's expression is the client-minus-server difference. Only the sign differs, so the size of the error, ±Δ/2, is unchanged.

**Additional context (worked example).** t1 = 100.000 s, t2 = 100.060, t3 = 100.061, t4 = 100.041 (client clock). Δ = (100.041 − 100.000) − (100.061 − 100.060) = 0.041 − 0.001 = 0.040 s. Estimated one-way delay Δ/2 = 0.020 s. Server ahead by t2 − (t1 + 0.020) = 100.060 − 100.020 = **0.040 s**. The same as the standard NTP formula ((t2 − t1) + (t3 − t4)) / 2 = (0.060 + 0.020) / 2 = 0.040 s. Whatever the real split of the delay, the error is at most ±0.020 s.

**The formulas in one place:**

```math
\Delta = (t_4 - t_1) - (t_3 - t_2), \qquad \text{error} = \Delta\,(k - 0.5), \qquad |\text{error}| \le \tfrac{\Delta}{2}
```

> **Kid example.** You throw a ball to a friend and time how long it takes to come back: 4 seconds, and your friend held it for 0 seconds. You guess it took 2 seconds each way. If really it took 3 seconds there and 1 back, your guess is off by 1 second, which is at most half of the 4 seconds. A faster round trip means a smaller possible mistake.

**Remember:**

- † **Δ = (t4 − t1) − (t3 − t2)** (round trip minus server time).
- † Assume **d = Δ/2**; if really d = kΔ, error = **Δ(k − 0.5)**, so **MaxError = ±Δ/2**.
- † **Smaller delay ⇒ smaller asymmetry error.** Standard offset: ((t2 − t1) + (t3 − t4)) / 2.

### 1.6 † NTP: measurement modes and performance

**In one sentence:** † NTP can query servers, listen to broadcasts, or ask once; its accuracy is limited by how well the network delay can be measured.

**Measurement protocols.**

| Mode | How it works |
|---|---|
| Two-way daemon with measured delay | UDP port 123 on server and client; periodic queries to **multiple** servers; client-server or peer-to-peer |
| Broadcast mode | **no measured delay**; one server, many clients; local LAN or Internet multicast |
| Single-shot client (ntpdate, SNTP) | server port UDP 123, any client port; measures the delay with the usual two-way method |

**Typical performance.**

- Typical asymmetry: a few percent of the delay.
- Small LAN: "10 milliseconds best case ever on 2-node LAN", "220 ms on real-world small LAN".
- Typical large-building LAN: 2 ms RMS.
- Internet with few hops: 10–20 ms.
- Long distance and/or slow or busy link: 100 ms – 1 s.
- **Accuracy is set by the accuracy of the network delay measurement**, not by the message format. **PTP/1588 is not better on the same network.** Accuracy degrades on networks with **asymmetric** delays (for example web servers).

**Technical clarification.** The two "small LAN" numbers in the lecture cannot be right as written: a small LAN cannot be worse (220 ms) than a large building LAN (2 ms) or an Internet path (10–20 ms). They were very probably **microseconds** (10 µs and 220 µs) in the original source, and the unit was changed by mistake.

**Additional context.** PTP (IEEE 1588) usually reaches much better accuracy than NTP in practice, but only because it uses hardware timestamps and PTP-aware switches that remove the delay uncertainty. That fits the lecture's point: what limits you is how well you can measure the network delay.

> **Kid example.** Whether you ask the time by phone or by video call, your watch can only be as accurate as your guess of how long the voice took to arrive. A fancier way of asking does not help if the line itself is slow and uneven.

**Remember:**

- † Modes: **two-way daemon** (port 123, many servers), **broadcast** (no delay measured), **single-shot** (ntpdate, SNTP).
- † LAN 2 ms RMS, Internet 10–20 ms, long or busy links 100 ms – 1 s (the "small LAN" numbers were probably µs).
- † Accuracy is set by **how well the delay is measured**; asymmetric delays hurt; PTP is not better on the same network.

## 2. Digital clock synchronization (Dolev §6.1)

**What this chapter is about:** simple algorithms that make all clocks equal using a common pulse: "biggest + 1", two bounded versions, and why a fixed small number of states can never work.

### 2.1 Digital clock synchronization: the problem

**In one sentence:** with a common pulse, every processor reads its neighbours' clocks and must end up with the same clock as everyone, going up by 1 per pulse.

**Motivation.** Multi-processor computers need synchronization for coordination, that is, clocks. Possible setups:

- a global clock pulse and a global clock value;
- **A global clock pulse and individual clock values** (this chapter);
- individual clock pulses and individual clock values;
- fault-tolerant clock synchronization.

Such systems can be resilient to faults, but the number of faults grows with the number of processors, so fault tolerance is vitally important.

**The problem.** In every pulse, each processor **reads the clocks of its neighbours** and uses these values to compute its **new clock value**. The goal:

1. **Identical clock values;**
2. **The clock values are incremented by one in every pulse.**

There are n **identical** processors, all connected to a **global common clock pulse**, each keeping a digital clock value.

**Additional context.** Identical processors are no obstacle here, because the goal (everybody equal) is itself symmetric. That is unlike leader election, where symmetry must be broken (Dolev §6.1).

> **Kid example.** The school bell rings once a minute for every classroom. At each bell, every classroom looks at the clocks of the rooms next door and sets its own clock. The goal: all clocks show the same time, and at every bell all of them go up by one minute.

**Remember:**

- Setting: **global clock pulse, individual clock values**; *n* **identical** processors.
- Each pulse: read neighbours' clocks, compute the new value.
- Goal: (1) **identical** values; (2) **+1 every pulse**.
- Identical processors are fine here: the goal is symmetric (unlike leader election).

### 2.2 The unbounded algorithm (max + 1)

**In one sentence:** at every pulse, take the biggest clock you can see and add 1; after d pulses everybody has the same value.

```text
01 upon a pulse
02   forall Pj ∈ N(i) do send(j, clock_i)
03   max := clock_i
04   forall Pj ∈ N(i) do
05     receive(clock_j)
06     if clock_j > max then max := clock_j
07   od
08   clock_i := max + 1
```

In words: **take the biggest clock you can see (yours or a neighbour's) and add 1.**

**Why it works.** "A simple induction can prove that this version of the algorithm is correct: if Pm holds the max clock value, by the i-th pulse every processor of distance i from Pm holds the maximal clock value." After **d** pulses (d = diameter), everyone has the same value, and from then on everyone adds 1 to the same number.

```mermaid
flowchart LR
    A["pulse 0<br/>Pm has the maximum"] --> B["pulse 1<br/>Pm's neighbours have it"]
    B --> C["pulse 2<br/>distance 2 has it"]
    C --> D["pulse d<br/>everyone has it"]
```

**A tiny run on a line p1 – p2 – p3 (d = 2):**

| Pulse | p1 | p2 | p3 |
|---|---|---|---|
| 0 | 3 | 7 | 1 |
| 1 | 8 | 8 | 8 |
| 2 | 9 | 9 | 9 |

Here p2 holds the maximum 7 in the middle, so everyone sees it after 1 pulse (in general it takes up to d pulses).

> **Kid example.** At every bell, each classroom looks at its own clock and the clocks next door, takes the **latest** time it sees and adds one minute. The latest time spreads one room further at each bell, so after d bells every clock shows the same time.

**Remember:**

- Rule: **clock := max(own, neighbours) + 1**.
- Induction: by pulse *i*, everyone within distance *i* of the max holder has the max ⇒ synchronized after **d** pulses.

### 2.3 Why unbounded clocks are a problem

**In one sentence:** real clocks have a maximum, and one fault can put a clock right at the maximum, so "+1" overflows.

**"Unbounded clocks is a drawback in self-stabilizing systems."** Using 2⁶⁴ possible values does **not** create the illusion of "unbounded": **a single transient fault may cause the clock to reach the maximal clock value.** Then the next "+1" overflows, and "take the maximum" no longer works.

> **Kid example.** A clock with only two digits for the hour can show at most 99. If a fault sets it to 99, then "+1" has nowhere to go. In self-stabilization we must assume that this can happen.

**Remember:**

- **Unbounded clocks are a drawback**: 2⁶⁴ values are **not** "unbounded", because **one transient fault** can set a clock to the maximum.

### 2.4 Bounded version I: max, modulo (n + 1)d + 1

**In one sentence:** the same "max + 1" rule on a dial of (n + 1)d + 1 values works, because there is always a big empty gap that slides along until no clock is near the top.

The same algorithm, only line 8 changes:

```text
08   clock_i := (max + 1) mod ((n + 1)d + 1)
```

So the bound is **M = (n + 1)d + 1**. "For ease of description the values we choose for M are not the minimal possible value." This is **converge-to-the-max**, and the key fact is: **the number of different clock values can only decrease, and it is reduced to a single clock value** (every new value is some existing value + 1, so no new distinct value can appear).

**Example.** Three processors on a line p1 – p2 – p3, so n = 3 and d = 2, and M = (3 + 1)·2 + 1 = **9**. The lecture animates the rounds on a dial 0…8; here is a run we computed with the same rule (our own starting values):

| Pulse | p1 | p2 | p3 | What happened |
|---|---|---|---|---|
| 0 | 4 | 5 | 8 | arbitrary start |
| 1 | 6 | 0 | 0 | p2 and p3 saw 8, and 8 + 1 = 9 wraps to 0 |
| 2 | 7 | 7 | 1 | p1 and p2 saw 6; p3 only sees 0s |
| 3 | 8 | 8 | 8 | all equal |
| 4 | 0 | 0 | 0 | they wrap together |

**Why it is correct.** Two cases.

1. **All clock values are less than M − d.** Then synchronization happens **before** the modulo is ever applied: after d pulses there must be convergence, and the max value is still less than M.
2. **Not all values are less than M − d**.
   - **Pigeonhole:** in any configuration there are two clock values x and y with **y − x ≥ d + 1** and **no clock value between them**. (There are only n clocks on a dial of M = (n + 1)d + 1 values, so somewhere there is a big empty stretch.)
   - **After i pulses, no clock value is in (x + i, y + i)**: everyone copies an existing value and adds 1, so the empty stretch just slides up by one per pulse.
   - **After M − y + 1 pulses**, the values above the gap have wrapped around, and all clock values are less than M − d. Now case 1 applies.

```mermaid
flowchart TB
    S["arbitrary configuration"] --> Q{"all clocks<br/>less than M − d?"}
    Q -- "yes" --> A["the max spreads in d pulses<br/>before any wrap-around"]
    Q -- "no" --> G["pigeonhole: an empty gap of size<br/>at least d + 1 between x and y"]
    G --> W["the gap slides up one per pulse;<br/>after M − y + 1 pulses all clocks are below M − d"]
    W --> A
    A --> D["synchronized: all equal, +1 mod M each pulse"]
```

**Additional context.** Once synchronized, all clocks show M − 1 at the same pulse and become 0 together at the next pulse (Dolev, quiz 14 Q10), so the wrap-around does not break agreement.

**The gap idea on a dial of M = 9 values (0 to 8), with clocks 4, 5, 8:**

```mermaid
flowchart LR
  A["used: 4, 5, 8"] --> B["empty stretch 6, 7<br/>and 0, 1, 2, 3"] --> C["each pulse, every value is<br/>an old value + 1: the empty<br/>stretch slides up by one"] --> D["the top values wrap around,<br/>then 'max + 1' works normally"]
```

> **Kid example.** Imagine a round table with 9 chairs and only 3 children, so there must be a long row of empty chairs somewhere. At every bell each child moves to "the chair after the furthest child they can see". Nobody ever jumps into the middle of the empty row; the empty row just moves around the table. Soon all children are bunched together and then move together.

**Remember:**

- Line 8: **clock := (max + 1) mod ((n + 1)d + 1)**; M is not minimal, just easy to prove.
- The number of **distinct** values can only **decrease**.
- Case 1: all clocks < M − d ⇒ the max spreads before any wrap.
- Case 2: **pigeonhole** gap of size ≥ d + 1 that slides up 1 per pulse; after **M − y + 1** pulses all clocks < M − d ⇒ case 1.

### 2.5 Bounded version II: min, modulo 2d + 1

**In one sentence:** using the smallest clock + 1 on a dial of only 2d + 1 values also works, and the dial size does not depend on n.

Use the **minimum** instead of the maximum:

```text
03   min := clock_i
06   if clock_j < min then min := clock_j
08   clock_i := (min + 1) mod (2d + 1)
```

The bound is **M = 2d + 1**, much smaller, and it does not depend on n.

**Why it works.**

- **If no processor assigns 0 during the first d pulses**, synchronization is reached (simple induction: the minimum spreads one hop per pulse).
- **Otherwise**, some processor assigns 0 during the first d pulses. **d pulses after that**, a configuration c is reached in which **no clock value is greater than d**. From c, nobody wraps to 0 for at least d pulses, so the first case applies.

Example (our own run, line p1 – p2 – p3, d = 2, M = 5):

| Pulse | p1 | p2 | p3 | What happened |
|---|---|---|---|---|
| 0 | 4 | 4 | 1 | arbitrary start |
| 1 | 0 | 2 | 2 | p1 only sees 4s, and 4 + 1 wraps to 0 |
| 2 | 1 | 1 | 3 | the 0 spreads to p2 |
| 3 | 2 | 2 | 2 | all equal |

> **Kid example.** Now each classroom takes the **earliest** time it sees and adds one minute, on a clock with only 2d + 1 numbers. If some room wraps to 0, that 0 is the earliest time around, so it spreads to everybody, and then they all tick together.

**Remember:**

- Rule: **clock := (min + 1) mod (2d + 1)**; **M = 2d + 1** (does not depend on *n*).
- No 0 in the first d pulses ⇒ synchronized. Otherwise, d pulses after a 0, all clocks ≤ d, and nobody wraps for d pulses ⇒ synchronized.

### 2.6 A constant number of states is impossible

**In one sentence:** if every processor may only have a fixed number of states, a big enough ring can just rotate its states forever and never synchronize.

**"There is no uniform digital clock-synchronization algorithm that uses only a constant number of states per processor"** (for deterministic algorithms). The question is: can the number of clock values be a fixed constant, related to neither the diameter nor the number of processors? No. The proof uses rings: a lower bound for this special case is a lower bound for general graphs.

**The setting.** (The lecture skipped the details of this proof this year.)

- A processor can read the clock of only a **subset** of its neighbours.
- In a **unidirected ring** every processor has a left and a right neighbour and reads the state of its **left** neighbour: **s_i(t+1) = f(s_(i−1)(t), s_i(t))**, where f is the transition function.
- |S| is the constant number of states.

**The idea.** Choose a **big enough ring** in which the algorithm never stabilizes: the states just **rotate**. In every step, each processor takes on the state of its right neighbour.

**The construction.**

1. Take two states s1 and s2 and build an infinite sequence with **s(l+2) = f(s(l), s(l+1))**.
2. There are only |S|² possible pairs, so in any |S|² + 1 pairs some pair **repeats** (pigeonhole). So there is a piece s_j, s_(j+1), …, s_k with **f(s_(k−1), s_k) = s_j** and **f(s_k, s_j) = s_(j+1)**, with k > j + 2 (equivalently s_(k+1) = s_j and s_(k+2) = s_(j+1)). Any 2(|S|² + 1) consecutive states contain such a piece.
3. Build a ring from s_j … s_k, with the processor in state s_k as the **left neighbour** of the processor in state s_j.
4. Each processor computes from its left neighbour and itself, so s_(j+i) becomes s_(j+i+1), and s_k becomes s_j: **in each pulse the states rotate one place to the left.**

```mermaid
flowchart LR
    A["sj"] --> B["sj+1"] --> C["sj+2"] --> D["..."] --> E["sk"] --> A
```

**The contradiction.** The set of states never changes; it only rotates. If the system were synchronized, every state would have to show the same clock value. But the clock must **increase** every pulse, and the same set of states cannot keep showing a bigger value. So it never stabilizes. "In the more complicated case of bidirectional rings, the lower-bound proof uses a similar approach."

Two natural follow-up questions, which Dolev answers:

- **"Do we have to assume that the ring is unidirectional?"** No. For bidirectional rings, f depends on three states, so a repeat appears within O(|S|³) states, and the same rotation argument works.
- **"Does a probabilistic algorithm exist for this ring?"** Yes. At each pulse, each processor randomly picks **its own clock or a neighbour's** and sets clock := (that + 1) mod M. The number of distinct values can only shrink, and it stabilizes **with probability 1** (if every processor happened to pick the neighbour on a shortest path to one fixed processor, all would agree within d pulses).

**Technical clarification.** The lecture once says "In a undirected ring"; it should be "unidirected" (one-way) ring, as the lecture says elsewhere and as in Dolev.

> **Kid example.** Children sit on a merry-go-round and each one copies a rule from the child on their left. If there are only a few possible "moods", you can choose a big enough ring of children where, at each turn, every mood just moves one seat to the right. The moods never become all the same; they only go round and round, so the children never agree.

**Remember:**

- **No uniform deterministic** clock synchronization with a **constant number of states** per processor.
- Unidirected ring: s_i(t+1) = f(s_(i−1)(t), s_i(t)); a repeated pair within |S|² + 1 pairs gives a ring where the states just **rotate** ⇒ never synchronized.
- Bidirectional rings: same idea with triples (O(|S|³)). **Randomized** version (own or neighbour's clock + 1) stabilizes with probability 1.

## 3. Stabilization in spite of napping (Dolev §6.2)

**What this chapter is about:** clocks that must agree even while some processors keep falling asleep and waking up, solved with a rock-paper-scissors trick.

### 3.1 Napping faults: wait-free self-stabilization

**In one sentence:** processors may nap and wake at any time; an awake processor must, after k steps, stop jumping (adjustment) and agree with every other awake one (agreement).

**Stabilizing in spite of napping:**

- A **wait-free self-stabilizing clock-synchronization algorithm** copes with **transient** and **napping** faults.
- **Each non-faulty operating processor ignores the faulty processors and increments its clock value by one in every pulse.**
- For a fixed integer **k**: once pi has worked correctly for at least k time units and keeps working, then:
  - **Adjustment:** pi does not adjust its clock (it just adds 1);
  - **Agreement:** pi's clock agrees with every other processor that has also worked correctly for at least k time units.

**Additional context.** Dolev §6.2 (and quiz 14):

- **Napping is worse than crashing**: a crashed processor is a napping one that never wakes, but a napping one can come back at any moment with an old value.
- **"Wait-free"**: a working processor synchronizes even if **all n − 1 others are faulty**; it never waits for anyone.
- Each requirement alone is trivial: "add 1 every pulse without talking to anyone" gives adjustment but not agreement; "always show 0" gives agreement but not adjustment. The challenge is both together.
- The setting is a **complete graph**: every processor can read every other processor.

```mermaid
flowchart LR
  A["pi has worked correctly<br/>for at least k steps"] --> B["ADJUSTMENT:<br/>pi only adds 1"]
  A --> C["AGREEMENT:<br/>pi's clock = clock of every<br/>other such processor"]
```

> **Kid example.** In a group project, some friends keep falling asleep and waking up. The rule: once you have been awake for k minutes, your watch must just tick normally (no big jumps), and it must show the same time as every other friend who has also been awake for k minutes. You may not wait for the sleepers.

**Remember:**

- **Wait-free self-stabilizing**: copes with **transient** and **napping** faults; never waits for others (works even if n − 1 are faulty).
- After **k** correct steps: **adjustment** (just +1) and **agreement** (with all others awake for k steps).
- Napping is **worse** than crashing. Complete graph.

### 3.2 The unbounded algorithm with k = 1

**In one sentence:** with unbounded clocks, "max of everybody + 1" works after one step, but with clocks mod M a sleeper stuck at M − 1 ruins it.

A simple example with **k = 1**, using unbounded clocks:

- **In every step, each processor reads the clocks of all the other processors, takes the maximum x, and sets its clock to x + 1.**
- After one step of P1, P1's clock holds the maximal value, and **it does not adjust its clock as long as it does not crash**.
- A napping processor's clock does not change while it naps, so when it wakes it is behind the maximum and does no harm.
- **"Note that this approach won't work using bounded clock values."**

Why not (Dolev): with clocks mod M, a napping processor stuck at **M − 1** is always the maximum, so every active processor keeps computing (M − 1 + 1) mod M = **0**. That is an adjustment at every step, for every k.

```mermaid
sequenceDiagram
    participant A as Active processors
    participant N as Napping processor (clock M − 1)
    Note over N: asleep, clock frozen at M − 1
    A->>N: read clocks: max is M − 1
    Note over A: clock := (M − 1 + 1) mod M = 0
    A->>N: read again: max is still M − 1
    Note over A: clock := 0 again, an adjustment every pulse
```

> **Kid example.** "Look at every watch, take the latest time, add one minute." A friend who sleeps has a frozen watch that is behind, so it does not matter. But on a 12-hour dial, a sleeping friend frozen at 11:59 always looks like the "latest" time, so everyone keeps jumping to 12:00, again and again.

**Remember:**

- **k = 1**, unbounded: clock := **max of all + 1**; after one step a working processor never adjusts again.
- With **mod M** it fails: a napper frozen at **M − 1** makes everyone jump to **0** every pulse.

### 3.3 The bounded algorithm: rock, paper, scissors

**In one sentence:** each pair keeps small counters 0, 1, 2 that show who is "behind" (napping), and only the clocks of processors that are not behind are used.

**The idea:** **identify the crashed (napping) processors and ignore their values.** Each processor P has:

- **P.clock ∈ {0, …, M − 1}**;
- for every other processor Q, **P.count[Q] ∈ {0, 1, 2}**.

**P is behind Q if P.count[Q] + 1 (mod 3) = Q.count[P].**

| P.count[Q] | Q.count[P] | Result |
|---|---|---|
| 0 | 1 | P is behind Q |
| 1 | 2 | P is behind Q |
| 2 | 0 | P is behind Q |
| equal values | equal values | no direction: neither is behind |

**Rock, paper, scissors.** Like the children's game, the three values beat each other in a circle (2 beats 1, 1 beats 0, 0 beats 2), so there is always a well-defined "who is ahead" between two different values, without needing bigger numbers.

**The program for P:**

```text
do forever (* executed by processor pi *)
  for each pj ∈ P do read count_j and clock_j
  let NB be the set of processors that are not behind any other processor
  if NB ≠ ∅ then
    let pk be a processor with the maximal clock in NB
    clock_i := clock_k + 1 (mod M)
  for each pj ∈ P do
    if pj is not behind pi then
      count_i[j] := count_i[j] + 1 (mod 3)
```

In words: take the maximum clock only among the processors that are **not behind anyone** (the recently active ones), add 1. Then "push the arrow towards yourself" for every processor that is not already behind you. If NB is empty, the clock is not changed.

**How the arrow behaves** (Dolev; checked with the rule above):

| Situation | count_i[j], count_j[i] | Meaning |
|---|---|---|
| start | 0, 0 | no direction |
| both step | 1, 1 | still no direction |
| only Pi steps (Pj naps) | 2, 1 | Pj is behind Pi, so Pj's clock is ignored |
| Pi steps again, Pj still naps | 2, 1 | Pi does not increment (Pj is already behind) |
| Pj wakes and steps | 2, 2 | Pj catches up, no direction again |

```mermaid
stateDiagram-v2
    direction LR
    NoDirection --> PjBehind: only Pi steps
    PjBehind --> PjBehind: Pi steps, Pj still napping
    PjBehind --> NoDirection: Pj wakes and steps
    NoDirection --> NoDirection: both step
```

**Run sample.** Four processors P1 to P4 in a complete graph with k = 2; the figure marks the active processors, the "behind" connections and which processors are in NB, and shows their clocks converging.

**Technical clarification.** The variable names change during the lecture: **count** in the first version of the code, **order** (order_ij) in the proof and in Dolev, **R** for the set NB in an older hidden version of the code, and **A** in quiz 14. They all mean the same things. One line of the code ends with a stray word "processors", and its "if NB ∅" has lost the "≠" sign.

**Rock, paper, scissors on the values 0, 1, 2** (an arrow means "is ahead of"):

```mermaid
flowchart LR
  one["1"] --> zero["0"]
  two["2"] --> one
  zero --> two
```

> **Kid example.** Every pair of friends has a little score card for each other with the numbers 0, 1, 2 that beat each other in a circle, like rock, paper, scissors. Each time you are awake and your friend is not already "behind" you, you push your number one step further. A friend who sleeps cannot push back, so after one step they are "behind" you, and you ignore their old watch. When they wake up and push, they catch up and count again.

**Remember:**

- Variables: **clock ∈ {0, …, M − 1}**, **count[Q] ∈ {0, 1, 2}** for every other Q.
- **P is behind Q** iff **P.count[Q] + 1 (mod 3) = Q.count[P]**.
- **NB** = processors not behind anyone; clock := **max clock in NB + 1 (mod M)**; if NB is empty, no change.
- Then increment count[j] (mod 3) for every pj **not behind** you.
- Names: count = order = R/A in different versions.

### 3.4 Theorem 6.1: the algorithm works with k = 2

**In one sentence:** after its first step a working processor is in NB, after its second step all of NB agree, so the bounded algorithm works with k = 2.

**Theorem 6.1:** the algorithm is a **wait-free self-stabilizing clock-synchronization algorithm with k = 2**.

**Proof** (starting with **any** values in the count/order and clock variables):

1. **Same view.** All processors that take a step in the same pulse see the same view (they can read all fields of all registers), so they compute the **same NB**.
2. **NB is not empty after pi's first step**, and while pi keeps taking steps it **stays in NB**: in each step it increments every order_ij for which pj is not behind pi, so pi is never behind anyone.
3. **Same clock after pi's second step.** Since NB is not empty and everyone who steps sees the same NB, all of them choose the same clock value. So after pi's second step (and while pi keeps going), all processors in NB have the same clock.
4. **Agreement:** every processor that executes a step belongs to NB, and all clocks in NB are equal.
5. **Adjustment:** every processor picks the maximal clock m in NB and sets m + 1 mod M; pi is in NB with that clock, so it just ticks by one.
6. The proof assumed an **arbitrary starting configuration**, so the algorithm is **both wait-free and self-stabilizing**. ■

| Pi's steps | What is guaranteed |
|---|---|
| 1st step | NB ≠ ∅ and pi ∈ NB |
| 2nd step | all processors in NB (and everyone who steps) have the same clock |
| later steps | agreement and adjustment hold, so k = 2 |

**Technical clarification.** The lecture (like Dolev) writes "the maximal clock value m of a processor in NB" and "increments m by 1 modulo M"; Dolev's printed text even uses the same letter M for both the value and the modulus. They are two different things: m is a clock value, M is the bound.

**Additional context.** Compared with the unbounded k = 1 algorithm, this one needs one more step (k = 2) but uses only bounded memory (clocks mod M and counters mod 3). That is its main point (quiz 14 Q44, Q49).

> **Kid example.** After your first awake step, you are surely in the "not behind" group. After your second awake step, everyone in that group has looked at the same watches and picked the same time. From then on you just tick along with them.

**Remember:**

- **Theorem 6.1**: wait-free self-stabilizing clock synchronization with **k = 2**.
- Same view ⇒ same NB. 1st step: **pi ∈ NB**. 2nd step: **all of NB agree**. Then agreement + adjustment.
- Trade-off: **k = 2** instead of 1, but **bounded** memory (clocks mod M, counts mod 3).

## 4. Summary and review

**What this chapter is about:** the whole lecture on one page, review questions, old exam questions and extra quiz points.

### 4.1 Summary and review questions

**In one sentence:** clock synchronization can self-stabilize even with faults during the run; the review questions test the bounds and rules.

**Summary:** we looked at some common fault models and saw how self-stabilizing clock synchronization can take them into account.

**Review questions:**

1. Can a number smaller than (n + 1)d + 1 be used in line 8 of Figure 6.2 without changing anything else? Prove stabilization or give a counterexample.
2. On a line P1, P2, …, Pn with **no sense of direction**, will Figure 6.2 stabilize if the increments are **modulo 3**? Prove it or give a counterexample.
3. Will the unbounded algorithm of Figure 6.1 stabilize if the **minimal** clock value plus one is assigned? What if the **average** (integer part) is used?

**Additional context (hints, not official answers).** 

- For question 1, the lecture already says that M = (n + 1)d + 1 was chosen "for ease of description" and is "not the minimal possible value", so smaller values can work; the min version of section 2.5 shows that 2d + 1 suffices for the min rule.
- For question 3 with the minimum, yes: let m be the smallest clock at the start. No clock ever goes below the current minimum, and after t pulses every processor within distance t of a processor holding m has m + t (it sees that value and nothing smaller). After d pulses all clocks equal m + d. This is the same induction as for the max rule.

### 4.2 Recap

**In one sentence:** NTP measures delay to set real clocks; digital clocks synchronize with max/min + 1 on a big enough dial; napping needs the rock-paper-scissors counters.

- **NTP** runs over UDP (port 123), uses 64-bit timestamps (seconds since 1900 + fraction, wrap in 2036, ~232 ps resolution). † Its two-way exchange measures the round-trip Δ; assuming equal halves gives an error of at most ±Δ/2, so smaller delay means smaller error.
- **Digital clock synchronization:** common pulse; goal = identical clocks that go up by one per pulse.
  - **Max + 1** (unbounded): synchronized after d pulses, but unbounded clocks are fragile.
  - **Max + 1 mod (n + 1)d + 1**: correct by the pigeonhole gap argument.
  - **Min + 1 mod 2d + 1**: correct, smaller M.
  - **Constant states are impossible** (deterministic, uniform): the rotating-ring argument. Randomization fixes it (probability 1).
- **Napping:** wait-free self-stabilization = adjustment + agreement after k steps.
  - **Max + 1 with k = 1** works with unbounded clocks, but mod M fails because of a napping M − 1.
  - **Bounded:** count/order variables mod 3 ("behind", rock-paper-scissors), take the max over **NB**; **Theorem 6.1: k = 2**.

```mermaid
flowchart LR
  A["NTP: real clocks,<br/>error ≤ ±Δ/2"] ~~~ B["digital clocks:<br/>max/min + 1 on a<br/>big enough dial"] ~~~ C["no constant-state<br/>deterministic solution"] ~~~ D["napping:<br/>count mod 3 + NB, k = 2"]
```

> **Kid example.** The whole story: real computers ask a time server and guess the delay (NTP). Simple classroom clocks agree by copying the latest (or earliest) neighbour time and adding one, as long as the dial is big enough. And if some classrooms keep dozing off, a rock-paper-scissors score card tells the awake ones whose clocks to ignore.

### 4.3 Old exam questions on this lecture

**In one sentence:** the exams ask about the unbounded max algorithm, the constant-states impossibility, the NTP "contradiction" and the count/NB trick.

On the exam cheat sheet these are **Algorithm 13** (unbounded max), **Algorithm 14** (max mod (n + 1)d + 1), **Algorithm 15** (min mod 2d + 1) and **Algorithm 16** (the wait-free bounded algorithm).

**October 2018, Q3d (true/false).** *"It is impossible to design self-stabilizing algorithms for digital clock synchronization in synchronous and anonymous networks … when each node can have only a constant number of states."*
**TRUE** (Dolev §6.1, section 2.6): in a large enough ring the states just rotate forever.

**October 2021, Q1j and October 2022 (true/false).** *"In class, we showed that there is no uniform digital clock-synchronization algorithm that uses only a constant number of states per processor. NTP uses a constant number of 64 bits to represent the clock values. Such contradictions can exist between theory and practice."*
**FALSE.** There is no contradiction: the lower bound is for **uniform** algorithms (identical processors, no special ones). NTP is **hierarchical**, with distinguished servers (strata) connected to reference clocks, so the lower bound does not apply.

**October 2019, Q3 (8 points).** The unbounded algorithm (clock_i := 1 + max of own and neighbours' clocks).
- (a) Fill in the task: **agreement:** identical clock values; **progress:** the clock values are incremented by one every pulse.
- (b) Why is it not self-stabilizing in practice? The variable is really bounded (say 64 bits); one transient fault can set it to the maximum, it overflows, and "take the max" no longer gives agreement.
- (c) Prove that the task holds within O(d) pulses: induction on t, "no clock is larger than M + t, and every processor within distance t of Pm has exactly M + t", where M is the largest value at the start.

**August 2020, Q2 (26 points).** A self-stabilizing "firing squad" in a synchronous network, built on **Algorithm 15** (min mod 2d + 1): once the clocks agree, they all reach 0 at the same pulse, which starts a common epoch.

**October 2025, Q3 (reconstructed).** Build a mechanism that tells each processor which others were recently active, in an asynchronous anonymous network with napping. The teacher: the **count variables and the set NB** of Algorithm 16 are the right tool, but **not** its global clock (a common clock does not fit the model); use a local counter ("local stopper") instead.

**Exam tip.** "Constant number of states" + "uniform/anonymous" → impossible (TRUE). "NTP contradicts the lower bound" → FALSE, because NTP has special servers. "Why is the unbounded clock not self-stabilizing?" → one transient fault can jump straight to the maximum.

**Remember:**

- Cheat sheet: **Algorithm 13** (unbounded max), **14** (max mod (n + 1)d + 1), **15** (min mod 2d + 1), **16** (wait-free bounded).
- Constant states + uniform ⇒ impossible (**TRUE**). "NTP contradicts the bound" ⇒ **FALSE** (NTP is hierarchical, not uniform).

### 4.4 Related quiz topics not on the slides

**In one sentence:** extra quiz points about pulses, bounds on M, bidirectional rings, randomization and the NB set.

The course's pre-lecture quiz on this chapter (quiz 14, 56 questions on §6.1 and §6.2) follows the lecture closely. Points it asks that the lecture leaves implicit:

| Question | Answer (quiz key, Dolev) |
|---|---|
| What happens in one pulse? (Q1) | Read the neighbours, compute the next value, then update the clock (lock-step: all reads finish before anyone writes). |
| How many pulses for the unbounded max + 1? (Q3) | At least d, the diameter. |
| How many pulses for bounded version I after finding the gap x, y? (Q9) | M − y + 1 pulses to get every clock below M − d, then d more for the max to spread. |
| What bound on M suffices for the min version? (Q11) | M > 2d (the code uses 2d + 1). |
| Why can the number of distinct values never grow? (Q12, Q53) | Every new value is an existing value + 1, so values can only merge. |
| Bidirectional rings? (Q19) | The same argument with triples of states, sequences of length O(\|S\|³). |
| How does the randomized constant-state algorithm work? (Q20, Q52–Q56) | Pick your own or a neighbour's clock at random, add 1 mod M; stabilizes with probability 1. In the scheduler-luck argument, luck makes every processor pick the neighbour on a shortest path to a fixed Pl, which synchronizes everyone within d pulses. |
| What are hybrid faults? (Q21) | Transient faults plus permanent faults during and after convergence. |
| If NB (A) is empty? (Q36) | The processor does not change its clock in that pulse. |
| Trade-off of the bounded wait-free algorithm? (Q44) | k = 2 instead of k = 1, but bounded memory. |

## Revision notes and references

Technical clarifications, where the lecture material is corrected or made precise, appear in sections [1.2](#12-ntp-message-format-and-transport), [1.5](#15--ntp-the-two-way-protocol-and-its-error), [1.6](#16--ntp-measurement-modes-and-performance), [2.6](#26-a-constant-number-of-states-is-impossible), [3.3](#33-the-bounded-algorithm-rock-paper-scissors), [3.4](#34-theorem-61-the-algorithm-works-with-k--2). Where the slides and the course book differ, this guide follows the book and says so at that point.

- S. Dolev, *Self-Stabilization*, MIT Press, 2000: §6.1 digital clock synchronization and §6.2 stabilization in spite of napping.
- [RFC 5905 — Network Time Protocol version 4](https://www.rfc-editor.org/rfc/rfc5905): the current NTP specification.
