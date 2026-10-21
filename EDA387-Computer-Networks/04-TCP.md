# End-to-End Communication and Congestion Control (TCP)

**Course:** EDA387/DIT660 Computer Networks, lecture "End-to-end Communication and Congestion Control" (Review 04), by Elad Michael Schiller. Part A repeats the transport layer: multiplexing, reliability, sliding windows, flow control and loss-based congestion control (Tahoe, Reno, NewReno). Part B, "Beyond EDA344", is new: bufferbloat, active queue management, BBR, L4S, QUIC and how TCP fails. Several slides follow the Kurose and Ross textbook.

**Revised:** 21 October 2026

**† Supplementary material:** content that comes only from hidden PowerPoint slides. It may not be part of this year's examinable material. In this lecture the hidden material is an archived window trace, an archived "real-world applications" overview, and three sets of review questions.

**How to read this guide.** Every part starts with **In one sentence**, the whole idea in one line. Then comes the explanation in small steps, a **Kid example** (the same idea told with everyday things), pictures, and a **Remember** list with what you must know for the exam. **Technical clarification** marks a place where the slides are wrong or unclear, and gives the correct version. **Additional context** adds something useful that is not on the slides.

---

## Glossary

| Term | Full name | Meaning in simple words |
|---|---|---|
| ACK | Acknowledgement | A message from the receiver saying what has arrived. In TCP the ACK number means "the next byte I expect". |
| AIMD | Additive Increase, Multiplicative Decrease | Grow the window by a fixed amount per round trip; cut it in half on congestion. |
| AQM | Active Queue Management | Routers drop or mark packets **before** the buffer is full, to keep queues short. |
| Bandwidth | — | How many bits per second a link can carry, like the width of a road. |
| Bandwidth-delay product (BDP) | — | Link rate × round-trip time: how much data fits "in the pipe" at once. |
| BBR | Bottleneck Bandwidth and Round-trip propagation time | Google's congestion control that measures the path and sends at the estimated BDP instead of waiting for loss. |
| Buffer | — | A waiting area (memory) in a router or computer where packets queue up. |
| Bufferbloat | — | Very large router buffers filled by loss-based TCP, causing huge delays while throughput looks fine. |
| CoDel | Controlled Delay | An AQM scheme that drops based on how **long** packets have waited, not how many are queued. |
| Congestion collapse | — | The network is busy but delivers almost nothing useful, mostly retransmissions. |
| Congestion control | — | Stops senders from overloading the **network**. |
| CUBIC | — | The Linux default congestion control: the window grows along a cubic curve in time since the last loss. |
| CWND | Congestion Window | How much unacknowledged data the sender may have in flight, as allowed by congestion control. |
| DevRTT | Deviation of the RTT | A running average of how much RTT samples differ from the estimate; used for the safety margin. |
| Duplicate ACK | — | An ACK that repeats the previous ACK number because a later segment arrived before a missing one. |
| ECN | Explicit Congestion Notification | Routers **mark** packets instead of dropping them; the receiver echoes the mark back to the sender. |
| EstimatedRTT | — | TCP's smoothed RTT: an exponentially weighted moving average of the samples. |
| EWMA | Exponentially Weighted Moving Average | new = (1 − α) × old + α × sample: old values fade away step by step. |
| Fast recovery | — | After a fast retransmit, halve the window (plus 3) and keep sending instead of restarting from 1. |
| Fast retransmit | — | Resend a segment after **3 duplicate ACKs**, without waiting for the timeout. |
| Flow control | — | Stops a sender from overflowing the **receiver's** buffer, using the advertised receive window. |
| FQ-CoDel | Fair Queueing with CoDel | Separate queues per flow, each managed by CoDel, so one big download cannot delay interactive traffic. |
| Karn's algorithm | — | Do not take an RTT sample from a retransmitted segment (you cannot tell which copy was ACKed). |
| L4S | Low Latency, Low Loss, Scalable Throughput | A redesigned congestion signal: frequent ECN marks in a separate low-latency queue (RFC 9330–9332). |
| Latency | — | Delay: how long one packet takes to get there. |
| MSS | Maximum Segment Size | The largest amount of data TCP puts in one segment. |
| Multiplexing / demultiplexing | — | Gathering data from many sockets into segments (mux); delivering each arriving segment to the right socket (demux). |
| NewReno | — | Reno improved to repair several losses from one window during a single fast recovery. |
| Pipelining (sliding window) | — | Allowing several unacknowledged segments in flight at once. |
| Port | — | A 16-bit number that says which program on a computer a segment is for. |
| QUIC | — | A modern transport protocol that runs over UDP inside the application (user space). |
| RcvWindow (rwnd) | Receive window | The free buffer space the receiver advertises in every segment. |
| Reno | — | TCP with slow start, congestion avoidance, fast retransmit **and fast recovery**. |
| RTO / TimeoutInterval | Retransmission timeout | How long the sender waits for an ACK before resending: EstimatedRTT + 4 × DevRTT. |
| RTT | Round-trip time | Time for a segment to reach the receiver plus the time for its ACK to come back. |
| Segment | — | The unit of data exchanged between transport layers (a packet is the network layer's unit; a frame is the link layer's). |
| Slow start | — | Start with CWND = 1 and add 1 per ACK, so the window **doubles** every RTT. |
| Socket | — | The "door" between an application process and the transport layer. |
| ssthresh | Slow start threshold | The window size where slow start ends and congestion avoidance begins. |
| Stop-and-wait | — | Send one packet, wait for its ACK, then send the next. |
| Tahoe | — | TCP with slow start, congestion avoidance and fast retransmit; on any loss, CWND goes back to 1. |
| Throughput | — | How much useful data actually gets through per second. |
| UDP | User Datagram Protocol | Connectionless "best effort" transport: no reliability, flow control or congestion control. |
| Vegas | — | Delay-based TCP: slows down when the RTT rises, before any loss. |
| Westwood | — | TCP that estimates bandwidth from the rate of returning ACKs and sets the window from it after a loss. |

## Contents

- [Before you start: the words you need](#before-you-start-the-words-you-need)
- [1. End-to-end transport](#1-end-to-end-transport)
  - [1.1 What "end-to-end" means](#11-what-end-to-end-means)
  - [1.2 Layers and the transport layer](#12-layers-and-the-transport-layer)
  - [1.3 What applications need, and what TCP and UDP offer](#13-what-applications-need-and-what-tcp-and-udp-offer)
  - [1.4 Multiplexing and demultiplexing](#14-multiplexing-and-demultiplexing)
- [2. Reliable data transfer and flow control](#2-reliable-data-transfer-and-flow-control)
  - [2.1 Reliable data transfer and stop-and-wait](#21-reliable-data-transfer-and-stop-and-wait)
  - [2.2 Pipelining and sliding windows](#22-pipelining-and-sliding-windows)
  - [2.3 When to retransmit: RTT estimation and timeouts](#23-when-to-retransmit-rtt-estimation-and-timeouts)
  - [2.4 Fast retransmit](#24-fast-retransmit)
  - [2.5 Flow control](#25-flow-control)
- [3. Congestion control and classical TCP](#3-congestion-control-and-classical-tcp)
  - [3.1 Congestion: causes, consequences and goals](#31-congestion-causes-consequences-and-goals)
  - [3.2 The congestion window and end-to-end feedback](#32-the-congestion-window-and-end-to-end-feedback)
  - [3.3 AIMD and fairness](#33-aimd-and-fairness)
  - [3.4 Van Jacobson's TCP: slow start and congestion avoidance](#34-van-jacobsons-tcp-slow-start-and-congestion-avoidance)
  - [3.5 Tahoe, fast recovery and Reno](#35-tahoe-fast-recovery-and-reno)
  - [3.6 TCP variants: NewReno, Vegas, Westwood, CUBIC, and ECN](#36-tcp-variants-newreno-vegas-westwood-cubic-and-ecn)
  - [3.7 The history of TCP](#37-the-history-of-tcp)
- [4. Modern transport: beyond loss-based control](#4-modern-transport-beyond-loss-based-control)
  - [4.1 Part B: thirty years you have not been taught](#41-part-b-thirty-years-you-have-not-been-taught)
  - [4.2 Bufferbloat and AQM](#42-bufferbloat-and-aqm)
  - [4.3 BBR](#43-bbr)
  - [4.4 L4S and QUIC](#44-l4s-and-quic)
  - [4.5 How TCP fails, and summary](#45-how-tcp-fails-and-summary)
- [5. Review questions](#5-review-questions)
  - [5.1 Check yourself: worked answers](#51-check-yourself-worked-answers)
  - [5.2 Review questions †](#52-review-questions-)
  - [5.3 Related quiz topics not in the lecture](#53-related-quiz-topics-not-in-the-lecture)
- [Revision notes and references](#revision-notes-and-references)

## Before you start: the words you need

TCP is about sending data **safely** and **politely**. A few words come back in every section.

- **Segment:** one piece of data that TCP sends, with a TCP header on the front. A big file is cut into many segments.
- **ACK (acknowledgement):** the receiver's reply "I got it". TCP's ACK says "I have everything before byte N; send me byte N next".
- **RTT (round-trip time):** how long it takes for a segment to get there **and** the ACK to come back.
- **Window:** how much data the sender is allowed to have "on the way" (sent but not yet ACKed) at one time.
- **Congestion:** too much traffic for the network, like a traffic jam. Routers then have to queue or throw away packets.

```mermaid
sequenceDiagram
    participant S as Sender
    participant R as Receiver
    S->>R: segment (bytes 1 to 100)
    R-->>S: ACK 101 ("next I want byte 101")
    Note over S,R: the time for this round trip is the RTT
```

> **Kid example.** TCP is like sending a long story to a pen-pal, a few numbered pages at a time. Your pen-pal writes back "I have everything up to page 7". You keep track of which pages are still on the way, you resend pages that get lost, and you send more slowly when the post office is overloaded.

## 1. End-to-end transport

**What this chapter is about:** why reliability is built by the two end computers and not by the network, where the transport layer sits, what TCP and UDP offer, and how arriving data finds the right program.

### 1.1 What "end-to-end" means

**In one sentence:** the network only moves packets (and may lose them); the two end computers build everything else, such as reliability and order.

*(Self-study slide: "What does End2End communications mean?")*

**End-to-end** means that the properties we care about (reliability, ordering, rate control) are built **between the two end computers**, not by the network in between. The network's job is only to move packets, and it is **allowed to lose, reorder or delay them**. Anything stronger is built by the hosts at either end.

That is a **design decision**, not a law of nature. The telephone network built reliability into the network itself. The Internet chose to **keep the middle simple and push complexity to the edges**, and almost everything in this lecture follows from that choice.

> **Kid example.** The postal service (the network) just carries envelopes and sometimes loses one. If you and your pen-pal want to be sure every letter arrives, you number your letters and write back "I got letters 1 to 5". The post office does not help; the two of you (the ends) do all the checking.

**Remember:**
- The network may **lose, reorder or delay**; the **end hosts** build reliability.
- "Simple middle, smart edges" is a **design choice** (the phone network chose the opposite).

### 1.2 Layers and the transport layer

**In one sentence:** the transport layer sits between the application and IP, runs only in the two end computers, and routers never look inside it.

#### The Internet protocol stack

*(Self-study.)*

| Layer | Job | Examples |
|---|---|---|
| Application | The programs' own protocols | FTP, SMTP, HTTP |
| **Transport** | Process-to-process delivery | **TCP, UDP** |
| Network | Routing datagrams from source to destination | IP, routing protocols |
| Link | Moving data between neighbouring network elements | PPP, Ethernet |
| Physical | Bits "on the wire" | — |

Each layer offers a service to the one above and uses the one below, so in principle any layer can be replaced without disturbing its neighbours. Part B shows QUIC deliberately blurring the transport boundary.

#### Logical versus physical communication

Each layer is **distributed**: "entities" implement the layer's functions at each node, perform actions, and exchange messages **with their peers** at the same layer. This conversation is **logical**: no packet travels "sideways" between two transport layers. Every message goes **down** the stack, across the wire, and **up** the other side.

**Transport protocols run in the end systems:** they take data from the application, add addressing (and maybe reliability information) to form a **segment**, send it to the peer, and (maybe) wait for the peer to ACK it.

```mermaid
flowchart LR
  subgraph A["Host A: data goes DOWN"]
    direction TB
    a5["application"] --> a4["transport"] --> a3["network"] --> a2["link"] --> a1["physical"]
  end
  subgraph R["Router: up and down"]
    direction TB
    r1["physical"] --> r2["link"] --> r3["network (reads IP only)"] --> r4["link"] --> r5["physical"]
  end
  subgraph B["Host B: data goes UP"]
    direction BT
    b1["physical"] --> b2["link"] --> b3["network"] --> b4["transport"] --> b5["application"]
  end
  A -->|"wire"| R
  R -->|"wire"| B
```

The arrows show the real path: down host A's stack, through the router (which climbs **only up to the network layer** and then goes down again), and up host B's stack. The two transport layers (in host A and host B) talk to each other only logically, through this path. A router opens the IP header and no further; the TCP header is just payload to it. That is what makes the transport layer end-to-end in a concrete sense, and why **routers do not take part in TCP** (they do not acknowledge or retransmit, and classically do not even know a connection exists). It is also why TCP can be changed by upgrading two hosts instead of the whole Internet.

> **Kid example.** Two friends in different cities play chess by post. They "talk chess" to each other (the logical conversation), but every move really goes into an envelope, through the post office, and out at the other end. The post office workers (routers) only read the address on the envelope, never the chess move inside.

**Remember:**
- Stack: application, **transport (TCP, UDP)**, network (IP), link, physical.
- Transport runs **only in the end hosts**; routers go up to the **network layer** only.
- Peer layers talk **logically**; real data goes down, across and up.

### 1.3 What applications need, and what TCP and UDP offer

**In one sentence:** TCP gives a reliable, ordered, polite connection; UDP just sends single messages with no promises; neither guarantees timing.

**What do end users need from the transport layer?** Some care about:

- **connection management**;
- **reliability** (guaranteed arrival);
- **timing**, for example for streamed communication.

Not everyone wants all three. A file transfer wants reliability and does not care much about timing. A live video call wants timing and can tolerate a lost frame. A DNS query wants neither badly enough to pay for a connection setup. That is why the Internet offers **two** transport protocols.

| | **TCP** | **UDP** |
|---|---|---|
| Setup | Connection-oriented | Connectionless |
| Reliability | Reliable transport between sender and receiver | None: "put stuff in an envelope and rely on IP for delivery (or not)" |
| Flow control | Yes: the sender won't overwhelm the receiver | No |
| Congestion control | Yes: acts when the network is overloaded | No |
| Timing / bandwidth guarantees | No | No |

> **Kid example.** TCP is like a phone call: you first say "hello, can you hear me?", then you talk, and if something is unclear you repeat it. UDP is like throwing paper planes with notes over a fence: quick and simple, but some planes get lost and nobody tells you.

**Flow control protects the receiver. Congestion control protects the network.** They use similar mechanisms (both limit how much the sender may have in flight) but respond to different signals and serve different parties. Students mix them up constantly, and examiners know it. Congestion control is described as "an extra service, both for users and for the network's benefit": a sender that ignores congestion also hurts itself, because a saturated network drops its packets too.

```mermaid
flowchart LR
    S["Sender"] -->|"congestion control:<br/>do not overload the NETWORK"| N["Network (routers)"]
    N -->|"flow control:<br/>do not overflow the RECEIVER"| R["Receiver"]
```

**Remember:**
- **TCP:** connection, reliable, flow control, congestion control. **UDP:** none of these.
- Neither gives **timing or bandwidth guarantees**.
- **Flow control protects the receiver; congestion control protects the network.**

### 1.4 Multiplexing and demultiplexing

**In one sentence:** port numbers let many programs share one network connection: the sender labels each segment with ports, and the receiver uses them to hand it to the right program.

A host runs many processes but has one network interface.

- **Multiplexing** (at the sending host): gathering data from several sockets and wrapping it with a header (later used for demultiplexing).
- **Demultiplexing** (at the receiving host): delivering each received segment to the **correct socket**.

The sender adds exactly the information the receiver needs to undo what the sender did: a general pattern in protocol design.

> **Kid example.** A family shares one mailbox (the network interface). Every letter carries a name on the envelope (the port number). Putting several family members' letters into the one mailbox, each with its name, is multiplexing; sorting the arriving letters to the right person by the name is demultiplexing.

#### How it works

A host uses **IP addresses and port numbers** to direct a segment to the right socket. Every TCP/UDP segment carries a **16-bit source port** and a **16-bit destination port**:

```mermaid
block-beta
  columns 2
  sp["source port # (16 bits)"] dp["dest port # (16 bits)"]
  o["other header fields"]:2
  d["application data (payload)"]:2
```

16 bits give 65,536 ports. That is a real limit: it is why a carrier-grade NAT can only squeeze so many subscribers behind one IPv4 address. Note that the ports are in the transport header but the IP addresses are in the network header, so demultiplexing needs information from **two layers** at once.

#### UDP demultiplexing

When a host receives a UDP datagram, it **checks the destination port** and sends the datagram to the socket with that port. That is the whole rule. So **datagrams with the same destination port but different source IP addresses and/or source ports go to the same socket.** One UDP socket can receive from many senders, and a UDP server needs no per-client state; the application must check the source itself if it cares who sent what.

The lecture's Java example: a server socket on port 6428 (`new DatagramSocket(6428)`), and client sockets on 5775 and 9157. Both clients send to destination port 6428; the server replies to whatever **source port** each datagram carried (5775 or 9157), which each client's operating system chose.

```mermaid
flowchart LR
    c1["client, port 5775"] -->|"to port 6428"| s["one server socket<br/>port 6428"]
    c2["client, port 9157"] -->|"to port 6428"| s
```

#### TCP demultiplexing

A TCP socket is identified by a **4-tuple**: **source IP, source port, destination IP, destination port**. The receiver uses **all four** values to pick the socket. So a server can have many simultaneous TCP sockets, **one per connection**. Web servers have a different socket for each client (and non-persistent HTTP even has a different socket for each request).

**The lecture's example.** Three segments arrive at server B, all with destination `B, port 80`:

| Segment from | Source IP, port | Goes to |
|---|---|---|
| Host A | A, 9157 | socket 1 |
| Host C | C, 9157 | socket 2 |
| Host C | C, 5775 | socket 3 |

```mermaid
flowchart LR
  s1["from A:9157 to B:80"] --> k1["socket (A, 9157, B, 80)"]
  s2["from C:9157 to B:80"] --> k2["socket (C, 9157, B, 80)"]
  s3["from C:5775 to B:80"] --> k3["socket (C, 5775, B, 80)"]
  k1 --> P["server processes on host B"]
  k2 --> P
  k3 --> P
```

Under UDP's rule all three would land in **one** socket. The extra three values let the server keep **separate state per connection** (sequence numbers, windows, congestion state). That per-connection state is the price of everything TCP offers, and why a TCP server can be exhausted by too many connections. (The same 4-tuple, plus the protocol number, is what ECMP hashes to choose a path in the routing lecture.)

> **Kid example.** UDP is like one big post box at a school office labelled "Lost property": everyone's letters to it land in the same box. TCP is like a separate private chat with each person: the school keeps one folder per conversation, named by "who, from which room, to whom, in which room".

**Remember:**
- Ports are **16 bits** (65,536); in the transport header.
- **UDP** socket = destination port only → many senders share one socket.
- **TCP** socket = **4-tuple** (src IP, src port, dst IP, dst port) → one socket **per connection**.

## 2. Reliable data transfer and flow control

**What this chapter is about:** how TCP makes sure every byte arrives (ACKs, timers, retransmission), how it sends many segments at once (sliding windows), how it picks the timer, how it notices loss early (fast retransmit), and how it avoids overflowing the receiver.

### 2.1 Reliable data transfer and stop-and-wait

**In one sentence:** the simplest reliable protocol sends one packet, waits for the ACK and resends on a timeout, but it wastes almost all the link's time.

Reliable data transfer matters in the application, transport and link layers: "in the top-10 list of important networking topics". **The characteristics of the unreliable channel decide how complex the protocol must be:**

| The channel may… | So the protocol needs… |
|---|---|
| corrupt bits | a checksum and retransmission |
| also lose packets | timers |
| also reorder or duplicate | sequence numbers |

#### Positive acknowledgement with retransmission (stop-and-wait)

1. The **receiver sends an ACK** when data arrives.
2. The **sender starts a timer** whenever it transmits.
3. The sender **retransmits if the timer expires** before the ACK arrives.

"Positive" means the receiver acknowledges what **arrived**; it cannot easily notice what failed to arrive.

> **Kid example.** Stop-and-wait is like sending one text message and refusing to send the next until your friend answers "got it". If no answer comes before your egg timer rings, you send the same message again. It is safe, but very slow when your friend is far away.

#### Any problem with stop-and-wait? Performance

**Example from the lecture:** a 1 Gbps link, 15 ms propagation delay, 8,000-bit packets.

- Transmission time: D_trans = L / R = 8,000 bits / 10⁹ bits/s = **8 microseconds**.
- Round-trip time: RTT = 2 × 15 ms = **30 ms**.
- **Sender utilisation** (fraction of time the sender is busy sending):

```math
U_{\text{sender}} = \frac{L/R}{RTT + L/R} = \frac{0.008\ \text{ms}}{30.008\ \text{ms}} \approx 0.00027 = 0.027\%
```

So a 1 Gbps link delivers one 8,000-bit packet every 30 ms ≈ **267 kbit/s**. The link is not the bottleneck; **the protocol is**. Making the link faster makes utilisation **worse**, because L/R shrinks while the propagation delay stays the same.

```mermaid
sequenceDiagram
  participant S as Sender
  participant R as Receiver
  Note over S: t = 0: first bit sent
  S->>R: packet (L/R = 8 microseconds on the wire)
  Note over R: last bit arrives about 15 ms later: send ACK
  R-->>S: ACK (about 15 ms back)
  Note over S: t = RTT + L/R = 30.008 ms: send the next packet
  S->>R: next packet
```

The sender's activity is a thin sliver followed by a long silence. Everything that follows (pipelining, windows and, in Part B, the bandwidth-delay product) is an attempt to **fill that silence**.

**Remember:**
- Bit errors → checksum + retransmit; loss → timers; reordering/duplicates → sequence numbers.
- Stop-and-wait: ACK, timer, retransmit on timeout.
- Utilisation = (L/R) / (RTT + L/R); the 1 Gbps example gives **0.027%** (≈ 267 kbit/s).
- A faster link makes stop-and-wait **worse**, not better.

### 2.2 Pipelining and sliding windows

**In one sentence:** let the sender have many unacknowledged segments on the way at once; the best amount is the bandwidth-delay product.

**Idea:** stop waiting. **Pipelining** (also called **sliding window**) lets the sender have **multiple "in-flight", not-yet-ACKed segments**.

- **Choice of N** (the window size): ideally large enough that the sender can **transmit continuously during the whole round trip**.
- It still needs ACKs and retransmission. When a segment in the sequence is missing, the receiver **acknowledges the last correctly received** in-order data.

That ideal window is the **bandwidth-delay product (BDP)**: link rate × RTT. A window smaller than the BDP leaves the link idle; a window **larger** than the BDP puts the extra **into a queue somewhere**. "Remember that last clause. It is the whole subject of Part B."

> **Kid example.** Think of a conveyor belt from the kitchen to a far-away table. Stop-and-wait puts one plate on the belt and waits for the empty plate to come back. Pipelining keeps putting plates on the belt. The best number of plates on the belt at once is "belt speed × trip time": that is the bandwidth-delay product. Fewer plates leave gaps on the belt; more plates pile up in a queue at the end.

**Worked numbers (the 1 Gbps example link).** BDP = 10⁹ bit/s × 0.030 s = 3 × 10⁷ bits = 3,750 packets of 8,000 bits. Counting the transmission time too, a window of about (30.008 ms) / (0.008 ms) ≈ **3,751 packets** keeps the sender busy all the time. With a window of **3**, utilisation is 3 × 0.027% ≈ **0.08%**: "increase utilization by a factor of 3".

```mermaid
sequenceDiagram
  participant S as Sender
  participant R as Receiver
  S->>R: packet 1
  S->>R: packet 2
  S->>R: packet 3
  Note over S: window of 3 used up: wait
  R-->>S: ACK 1 (window slides: send packet 4)
  R-->>S: ACK 2 (send packet 5)
  R-->>S: ACK 3 (send packet 6)
```

**As ACKs arrive, the window moves forward.** The window is not a fixed batch that empties and refills; it is a **range** that slides. Each ACK moves the left edge, which allows new data at the right edge. This makes TCP **self-clocking**: data leaves at the rate ACKs return, which is the rate the network delivers. (The cost: the sender's behaviour now also depends on the reverse path.)

#### TCP's sliding window counts bytes

TCP's window is measured in **byte positions**, not packets. The lecture's example:

```mermaid
block-beta
  columns 4
  a["bytes 1 to 2: acknowledged"] b["bytes 3 to 6: sent, not yet ACKed"] c["bytes 7 to 9: may be sent now"] d["bytes 10 and up: outside the window"]
```

The window covers bytes 3 to 9. When the ACK for byte 3 arrives, the whole window slides one byte to the right.

> **Kid example.** Imagine a row of numbered sweets and a frame you can slide along it. You may only hand out sweets inside the frame. Each time your friend says "got the next one", you slide the frame one step right, and a new sweet comes inside the frame.

Because TCP counts bytes, it can re-segment (resend data in a differently sized segment), and it gives the application a **byte stream with no message boundaries**. If an application calls `send` twice, the receiver is **not** guaranteed two matching reads; applications that need message boundaries must add them themselves (a classic source of bugs).

**Remember:**
- **Pipelining / sliding window** = several unACKed segments in flight.
- Best window = **BDP = rate × RTT**; less wastes the link, more builds a **queue**.
- 1 Gbps example: BDP = 3,750 packets (≈ 3,751 counting L/R); window 3 → 0.08%.
- The window **slides** with each ACK (**self-clocking**); TCP counts **bytes** (a byte stream, no message boundaries).

### 2.3 When to retransmit: RTT estimation and timeouts

**In one sentence:** TCP keeps a smooth average of the measured RTT plus a safety margin based on how much it varies, and uses that as its timer.

**Too short a timer** → the sender retransmits data that was only delayed (wasting capacity and, as Part B shows, possibly worsening congestion). **Too long** → the sender sits idle after a real loss. The right value depends on **this path's RTT right now**, which changes, so TCP **measures it continuously**.

#### EstimatedRTT

```math
\text{EstimatedRTT} = (1 - \alpha) \times \text{EstimatedRTT} + \alpha \times \text{SampleRTT}, \qquad \alpha = 0.125 \text{ typically}
```

This is an **exponentially weighted moving average (EWMA)**: the influence of a past sample decreases exponentially fast. It tracks recent behaviour smoothly and needs to store **one number** per connection (important when a server has hundreds of thousands of connections). The lecture's figure shows SampleRTT jumping around between gaia.cs.umass.edu and fantasia.eurecom.fr, with EstimatedRTT as a smooth line through it.

> **Kid example.** EstimatedRTT is like guessing how long the walk to school takes. Every day you change your guess just a little towards today's time (one eighth of the difference, for α = 0.125). One very slow day, behind a tractor, does not ruin your guess, but if every day gets slower, your guess slowly follows.

#### The timeout: EstimatedRTT plus a safety margin

```math
\text{DevRTT} = (1 - \beta) \times \text{DevRTT} + \beta \times |\text{SampleRTT} - \text{EstimatedRTT}|, \qquad \beta = 0.25 \text{ typically}
```

```math
\text{TimeoutInterval} = \text{EstimatedRTT} + 4 \times \text{DevRTT}
```

A large variation in the samples gives a **larger safety margin**. On a stable path the timeout sits just above the estimate, so losses are noticed quickly; on a jittery path the margin widens automatically, so normal jitter does not cause needless retransmissions. "The protocol is adapting not just to how long the path takes, but to how predictable it is."

```mermaid
flowchart LR
    S["new SampleRTT"] --> E["update EstimatedRTT<br/>(α = 0.125)"]
    S --> D["update DevRTT<br/>(β = 0.25)"]
    E --> T["Timeout =<br/>EstimatedRTT + 4 × DevRTT"]
    D --> T
```

> **Kid example.** If your bus is always exactly on time, you leave home just a minute early. If your bus is sometimes 10 minutes late, you leave much earlier, just in case. DevRTT measures "how unreliable is the bus", and the timeout adds four times that as a safety margin.

#### Worked example from the lecture

At the start there is no previous estimate, so EstimatedRTT = the first sample:

| Step | SampleRTT | EstimatedRTT |
|---|---|---|
| 1st ACK | 0.052671 − 0.024047 = 0.028624 s | 0.028624 s (first sample used directly) |
| 2nd ACK | 0.052676 − 0.024048 = 0.028628 s | 0.875 × 0.028624 + 0.125 × 0.028628 = 0.0286245 s |

Each sample is the time the ACK arrived minus the time the segment was sent. Notice how little the estimate moves per sample: that damping is intended.

**Technical clarification.** The lecture's first line writes the full update formula with the same number on both sides ("0.875 × 0.028624 + 0.125 × (0.052671 − 0.024047) = 0.028624"). That is correct arithmetic, but the reason is simply that the first sample is used as the starting estimate.

**Karn's algorithm**: TCP does **not** take an RTT sample from a **retransmitted** segment, because it cannot tell whether the ACK belongs to the first or the second transmission.

```mermaid
sequenceDiagram
  participant S as Sender
  participant R as Receiver
  S->>R: segment, first copy (delayed, not lost)
  Note over S: timer fires: send again
  S->>R: segment, second copy
  R-->>S: ACK
  Note over S: ACK for copy 1 or copy 2? Skip this sample.
```

> **Kid example.** You ring a friend's doorbell, wait, and ring again. Then the door opens. Was it the first ring or the second that they heard? You cannot know, so you do not use this visit to judge how fast your friend opens the door.

**Remember:**
- **EstimatedRTT** = 0.875 × old + 0.125 × sample (EWMA).
- **DevRTT** = 0.75 × old + 0.25 × |sample − estimate|.
- **Timeout = EstimatedRTT + 4 × DevRTT**.
- First sample: EstimatedRTT = that sample.
- **Karn:** never sample a retransmitted segment.

### 2.4 Fast retransmit

**In one sentence:** three duplicate ACKs tell the sender a segment was lost, so it resends it at once instead of waiting for the timer.

The timeout can be **relatively long**, so a lost packet may wait a long time to be resent. But TCP has a faster signal:

- When the receiver gets segments **out of order** (because one is missing), it keeps acknowledging the **last in-order byte**: it sends **duplicate ACKs**.
- Senders often send many segments back to back, so **if one segment is lost there will likely be many duplicate ACKs.**
- **If the sender receives 3 duplicate ACKs for the same data**, it assumes the segment after the ACKed data was lost and **resends it before the timer expires**: **fast retransmit**.

**The lecture's example** (Host A to Host B):

```mermaid
sequenceDiagram
  participant A as Host A
  participant B as Host B
  A->>B: Seq=92, 8 bytes (arrives)
  A-xB: Seq=100, 20 bytes (lost)
  A->>B: Seq=120, more data
  A->>B: Seq=140, more data
  A->>B: Seq=160, more data
  B-->>A: ACK=100 (for Seq=92)
  B-->>A: ACK=100 (duplicate 1)
  B-->>A: ACK=100 (duplicate 2)
  B-->>A: ACK=100 (duplicate 3)
  Note over A: triple duplicate ACK: resend Seq=100 now
  A->>B: Seq=100, 20 bytes (retransmission)
```

The sequence numbers after 100 (120, 140, 160) are illustrative; the lecture draws the extra segments without numbers. ACK = 100 means "I have everything up to byte 99; I want byte 100 next".

> **Kid example.** You post numbered pages 1, 2, 3, 4, 5 to a friend, who texts back after each one "I have everything up to page N". If page 2 is lost, your friend keeps answering "up to page 1", "up to page 1", "up to page 1" when pages 3, 4 and 5 arrive. After three such repeated answers you do not wait for the timer: you resend page 2 at once.

**Why 3 and not just 2?** One or two duplicates can easily be caused by mild **reordering** (packets taking slightly different paths). Retransmitting on the first duplicate would cause constant needless retransmissions on paths that reorder. Three makes reordering an unlikely explanation while still reacting much faster than the timeout.

Also note: the sender resends **only** the missing segment, not everything after it; the duplicates are evidence that the later ones arrived. Repairing several losses from one window is harder, which is what **SACK** (selective acknowledgement) was introduced for.

**Additional context.** Delayed ACKs (the receiver waits up to about 200 ms, never more than 500 ms, to acknowledge two segments with one ACK) can slow fast retransmit down, because duplicate ACKs come out more slowly. A receiver should send an ACK **immediately** when an out-of-order segment arrives (RFC 5681).

**Remember:**
- Out-of-order arrival → **duplicate ACKs** for the last in-order byte.
- **3 duplicate ACKs** → resend the missing segment now (**fast retransmit**).
- 3, not 1 or 2, so that mild **reordering** does not trigger it.
- Only the missing segment is resent; **SACK** helps with several losses.

### 2.5 Flow control

**In one sentence:** in every segment the receiver says how much free buffer space it has, and the sender never has more than that on the way.

**What do ACKs achieve besides reliability?** **Flow control**: the receiver can acknowledge its **receiving capacity** (the free space in its buffer, the "receiver's window"), so the sender does not swamp it.

- The **receiver** explicitly tells the sender how much free buffer space it has, in the **RcvWindow** field of every TCP segment. It changes all the time.
- The **sender** keeps the amount of **transmitted, un-ACKed data below the most recently received RcvWindow**.
- So the sender won't overrun the receiver's buffer by sending too much, too fast.

**RcvBuffer** = the size of the TCP receive buffer; **RcvWindow** = the spare room in it.

```mermaid
block-beta
  columns 4
  used["data waiting for the application to read it"]:2 free["RcvWindow: spare room"]:2
```

The whole bar is RcvBuffer.

> **Kid example.** Flow control is like pouring juice into a friend's glass: the friend keeps telling you how much room is still left in the glass (the receiver's window), and you never pour more than that. If the friend stops drinking, the glass fills up and you must stop pouring.

As the application reads, the window grows; if the application stops reading, the window shrinks towards **zero**. A **zero window** is legal and means "stop". The sender then **probes** now and then, because a window update could itself be lost and a sender that waited passively might wait forever.

**Protocol economy:** the ACK has to travel back anyway; adding one field tells the sender two things in one message: what arrived, and how much more the receiver can hold.

**Remember:**
- **RcvWindow** = free space in the receiver's buffer, sent in **every** segment.
- Un-ACKed data in flight ≤ **RcvWindow**.
- Zero window = "stop"; the sender **probes** so it does not wait forever.

## 3. Congestion control and classical TCP

**What this chapter is about:** what congestion is and why it is dangerous, how TCP guesses that the network is congested, the AIMD rule that makes flows share fairly, and the classic versions of TCP (Tahoe, Reno, NewReno, Vegas, Westwood, CUBIC) plus ECN.

### 3.1 Congestion: causes, consequences and goals

**In one sentence:** congestion is too much traffic for the network; it causes queues, loss and, at worst, a collapse where almost nothing useful gets through.

#### Need for congestion control

**Congestion occurs when demand for network resources (bandwidth, processing) exceeds capacity.** It is not just theory: in the mid-1980s the Internet suffered **congestion collapses**, where throughput fell by orders of magnitude although nothing was physically broken. Senders retransmitted into a saturated network, and the retransmissions used up the capacity that would have let anything succeed.

**Informally:** "too many sources sending too much data too fast for the network to handle."

**Manifestations (how you notice it):**

- **long delays** (queueing in router buffers);
- **packet loss** (buffer overflow at routers).

> **Kid example.** Congestion is a traffic jam. Too many cars try to use one road at once, so they queue (delay), and when the queue gets too long, some cars are sent away (loss). If the drivers who were sent away all immediately try again, the jam gets even worse.

| | Congestion control | Flow control |
|---|---|---|
| Problem | Too many senders, sending too fast | One sender too fast for one receiver |
| Who is protected | The network | The receiver |
| Feedback | None explicit: the sender must **guess** congestion | Explicit: the receiver reports its buffer |

#### Consequences

- **Packet loss**, when overwhelmed routers drop packets.
- **Increased latency**, as packets queue.
- **Reduced throughput**, as capacity is used by traffic that is dropped or retransmitted.
- **Degraded user experience**: slow websites, stuttering video, sluggish apps.

These **compound** in a feedback loop:

```mermaid
flowchart LR
  L["packet loss"] --> RT["retransmissions"]
  RT --> LD["more load"]
  LD --> Q["longer queues"]
  Q --> D["more delay"]
  D --> TO["timeouts fire"]
  TO --> RT
  Q --> L
```

Congestion control breaks this loop by making senders **send less** when they see signs of congestion. Notice that no component has failed anywhere in the loop.

#### Goals of congestion control

- **Fair resource sharing** among users, applications and connections.
- **Preventing collapse**: keeping the network stable.
- **Ensuring QoS** (quality of service): a minimum level of performance, such as minimum bandwidth, low latency, reliable delivery.
- **Enhancing user experience.**

The goals pull against each other, and **fairness is hard to define**: equal rate per connection, per user, or per application? A user who opens ten connections gets ten shares under the first definition (which is exactly what browsers learned to do).

**Remember:**
- Congestion = **demand > capacity**; signs are **delay** and **loss**.
- 1980s **congestion collapses**: busy network, almost no useful work.
- Loss → retransmissions → more load → more loss: a **feedback loop**.
- Congestion control: protects the **network**, no explicit feedback; flow control: protects the **receiver**, explicit feedback.

### 3.2 The congestion window and end-to-end feedback

**In one sentence:** TCP limits how much is in flight with a congestion window, and since the network gives no direct signal, it treats lost packets as the sign of congestion.

- **Congestion window (CWND):** the maximum number of unacknowledged packets a sender may have in transit at any time.
- TCP **continuously adjusts CWND** based on feedback from ACKs and other indicators.

**CWND limits data in flight, not rate.** The rate follows from it:

```math
\text{throughput} \approx \frac{w \times \text{MSS}}{\text{RTT}} \ \text{bytes/sec}, \quad w = \text{CWND in segments}
```

The sender may have at most min(CWND, RcvWindow) un-ACKed bytes: the smaller of the congestion window and the receive window wins.

```mermaid
flowchart LR
    C["CWND<br/>(network limit)"] --> M{"take the smaller"}
    R["RcvWindow<br/>(receiver limit)"] --> M
    M --> F["data the sender may<br/>have in flight"]
```

**Two consequences:**

1. The same window gives a **higher rate on a shorter path**.
2. **RTT unfairness:** two flows sharing a bottleneck with the same algorithm but different RTTs do not get equal shares; the short-RTT flow gets feedback sooner and grows faster.

> **Kid example.** Two children carry buckets of sand, each allowed 3 buckets at a time. One walks to a sandpit 10 metres away, the other to one 100 metres away. The near one moves much more sand per hour with the same 3 buckets. Same window, shorter trip, higher rate.

**End-to-end feedback.** Think of congestion as an **alarm flag observable at the end stations**. The network does not raise the flag; routers just drop packets when buffers overflow, and the sender notices missing ACKs. So **loss is used as a stand-in for congestion**, with a known defect: packets can be lost for other reasons (for example a wireless bit error), and TCP cannot tell the difference.

#### The simple feedback model

Every RTT, feedback arrives via ACKs; **duplicate ACKs indicate congestion**.

```mermaid
flowchart LR
  F["feedback every RTT from ACKs"] --> Q{"signs of congestion<br/>(duplicate ACKs, loss)?"}
  Q -->|"high congestion"| D["decrease rate:<br/>smaller CWND"]
  Q -->|"low congestion"| I["increase rate:<br/>bigger CWND"]
  D --> F
  I --> F
```

The sender treats "no trouble" as permission to send more and **never stops probing upward**, then backs off on trouble, then probes again. Capacity changes all the time as other flows come and go, so a fixed answer would be wrong within seconds. But it means TCP **causes the very loss it responds to**: "a startling thing to say out loud, and true".

**Remember:**
- **CWND** limits data in flight; throughput ≈ **CWND × MSS / RTT**.
- In flight ≤ **min(CWND, RcvWindow)**.
- Shorter RTT → higher rate (**RTT unfairness**).
- TCP uses **loss as the sign of congestion** (wrong on wireless); it keeps probing upward.

### 3.3 AIMD and fairness

**In one sentence:** grow the window slowly (add 1 per RTT) and cut it fast (halve on loss); this makes competing flows end up with fair, equal shares.

The lecture compares three ways to update the window:

| Scheme | On congestion | No congestion | Performance | Fairness |
|---|---|---|---|---|
| **Multiplicative** | CWND / 2 | 2 × CWND | Fast response | **Ratios unchanged** (stays unfair) |
| **Additive** | CWND − 1 | CWND + 1 | Slow response | Divides spare bandwidth equally, but the **difference remains unchanged** |
| **AIMD** | CWND / 2 (multiplicative decrease) | CWND + 1 (additive increase) | Fast response to congestion | Additive increase **improves the ratio**; multiplicative decrease keeps the ratio but **shrinks the difference** |

So with AIMD, **connections that over-use resources lose more during multiplicative decreases**, and **connections below their fair share can increase gradually**.

**TCP congestion control = AIMD.** Increase the sending rate by **1 MSS every RTT** until loss is detected; **cut it in half** at each loss event. The result is the **sawtooth**: "probing for bandwidth". Grow slowly, retreat quickly.

```mermaid
%%{init: {"themeVariables": {"xyChart": {"plotColorPalette": "#1f5fbf"}}}}%%
xychart-beta
  title "The AIMD sawtooth"
  x-axis "RTT" [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]
  y-axis "CWND" 0 --> 24
  line [10, 11, 12, 13, 14, 15, 16, 8, 9, 10, 11, 12, 13, 14, 7]
```

Each loss cuts the window in half; between losses it climbs by one per RTT.

> **Kid example.** AIMD is like walking more and more quickly along a busy corridor: you speed up a little each step (additive increase). The moment you bump into someone, you halve your speed at once (multiplicative decrease). Fast walkers lose more speed when they bump, so in the end everyone walks at about the same speed: that is the fairness.

#### Two users, one link (Chiu and Jain, 1989)

Plot user 1's rate on one axis and user 2's on the other. The **bandwidth limit** is the line where the two rates add up to capacity; the **fairness line** is where both are equal. Additive increase moves the point **parallel to the fairness line** (both add the same amount, so the difference stays the same). Multiplicative decrease moves it **towards the origin** (both halve, so the ratio stays the same and the difference halves). Repeating the two moves drives the point to where the fairness line meets the capacity line: **a fair and efficient allocation, from any starting point, without any communication between the users**.

**Worked example.** Capacity 100. User 1 starts at 70, user 2 at 10. Each step both add 5; whenever the total goes over 100, both halve:

```mermaid
%%{init: {"themeVariables": {"xyChart": {"plotColorPalette": "#1f5fbf, #d62728"}}}}%%
xychart-beta
  title "AIMD: two users converge (capacity 100)"
  x-axis "step" [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21]
  y-axis "rate" 0 --> 100
  line [70, 75, 80, 85, 42.5, 47.5, 52.5, 57.5, 62.5, 67.5, 33.75, 38.75, 43.75, 48.75, 53.75, 58.75, 29.375, 34.375, 39.375, 44.375, 49.375, 54.375]
  line [10, 15, 20, 25, 12.5, 17.5, 22.5, 27.5, 32.5, 37.5, 18.75, 23.75, 28.75, 33.75, 38.75, 43.75, 21.875, 26.875, 31.875, 36.875, 41.875, 46.875]
```

The blue (upper) line is user 1, the red (lower) line user 2. The gap between them is 60 at the start, then 30, 15 and 7.5 after the first, second and third halvings: it halves every time, so the two rates converge.

> **Kid example.** Two children share a cake that keeps being refilled. Each time, both take one more spoonful (the gap between them stays the same). When the plate runs out, both give back half of what they have (the gap halves). After a few rounds they have almost exactly the same amount, without ever talking to each other.

**Additional context.** A sending scheme is called **TCP-friendly** if it gets roughly the same throughput as a standard TCP flow under the same conditions. Unfairness arises when one flow backs off after loss while another keeps growing aggressively.

**Remember:**
- Multiplicative only: ratio stays → unfair. Additive only: difference stays → unfair.
- **AIMD**: +1 MSS per RTT, ÷2 on loss → **fair and efficient** (Chiu and Jain, 1989).
- Each halving **halves the gap** between two flows (60 → 30 → 15 → 7.5).
- The **sawtooth** = probing for bandwidth.

### 3.4 Van Jacobson's TCP: slow start and congestion avoidance

**In one sentence:** TCP starts with a window of 1 and doubles it every RTT (slow start) until a threshold, then adds just 1 per RTT (congestion avoidance).

**Van Jacobson's** algorithms for TCP helped solve the congestion problem and are used in over 90% of Internet hosts. The history: in **1986** the link between Lawrence Berkeley Laboratory and UC Berkeley, a few hundred metres, dropped from **32 kbit/s to 40 bit/s**: a thousand-fold collapse with no hardware fault. Jacobson diagnosed it, and his **1988** fixes are slow start and congestion avoidance. A network of correctly working components entered a lasting near-total failure and needed a **change of algorithm**, not a repair, to leave it: the same theme as BGP wedgies in the routing lecture.

**End-to-end (no network assistance), window-based congestion control:**

- Designed by Van Jacobson in the late 1980s, based on the AIMD algorithm of Dah-Ming Chiu and Raj Jain.
- **Dynamic window size** [Van Jacobson, 1988].
- **Initialization:** the **slow start** phase (labelled "MI" in the lecture).
- **Steady state:** upon reaching a threshold, **AIMD** for **congestion avoidance**.
- It has worked well: the Internet's bandwidth has grown more than 200,000 times.
- Many versions, with different ways to detect a congestion event:
  - **TCP Tahoe:** less optimised (congestion = timeout).
  - **TCP Reno:** common implementation (congestion = timeout **or** 3 duplicate ACKs).
  - **TCP Vegas:** not currently used.

**Technical clarification.** The lecture expands MI as "Mild Increase". In the AIMD vocabulary, slow start is **multiplicative (exponential) increase**: the window doubles every RTT, which is anything but mild ("not so slow!", as the lecture says next). The lecture also spells Chiu as "Chu".

#### Slow start

```
initialize: CWND = 1
for (each segment ACKed)
    CWND++
until (congestion event OR CWND > threshold)
```

Adding 1 **per ACK** doubles the window **every RTT**: send 1 segment, get 1 ACK → CWND = 2; send 2, get 2 ACKs → CWND = 4; then 8, 16, … "Exponential increase (per RTT) in window size (not so slow!)". The name means starting from **one** segment instead of blasting at full speed. The lecture's second figure shows bursts growing from 6 to 12 to 24 segments.

```mermaid
flowchart LR
    r1["RTT 1:<br/>1 segment"] -->|"1 ACK, +1"| r2["RTT 2:<br/>2 segments"] -->|"2 ACKs, +2"| r3["RTT 3:<br/>4 segments"] -->|"4 ACKs, +4"| r4["RTT 4:<br/>8 segments"] -->|"8 ACKs, +8"| r5["RTT 5:<br/>16 segments"]
```

> **Kid example.** Slow start is like a new player in a game who starts with one ball, then two, then four, then eight, doubling each round until something goes wrong or a remembered limit (the threshold) is reached. After that, the player only adds one ball per round.

**On a timeout:** Slow Start Threshold = CWND / 2 (remember where trouble started).

#### Congestion avoidance

Once CWND > threshold, grow by about **one segment per RTT** (additive increase). In practice the sender adds MSS × MSS / CWND bytes per ACK, which adds up to one MSS per round trip.

```mermaid
stateDiagram-v2
  [*] --> SlowStart: CWND = 1
  SlowStart --> SlowStart: on each ACK, CWND + 1 (doubles per RTT)
  SlowStart --> CongestionAvoidance: CWND reaches ssthresh
  CongestionAvoidance --> CongestionAvoidance: every RTT, CWND + 1
  SlowStart --> SlowStart: on timeout, ssthresh = CWND/2 and CWND = 1
  CongestionAvoidance --> SlowStart: on timeout, ssthresh = CWND/2 and CWND = 1
```

This diagram shows the parts that Tahoe and Reno share. What happens on **three duplicate ACKs** is where they differ ([section 3.5](#35-tahoe-fast-recovery-and-reno)).

Most Internet connections are short (a few kilobytes) and **finish before ever leaving slow start**. Steady-state congestion avoidance describes a minority of flows, but those carry most of the bytes.

**Remember:**
- 1986: LBL to UC Berkeley fell from **32 kbit/s to 40 bit/s**; Jacobson's fix in **1988**.
- **Slow start:** CWND = 1, +1 per ACK → **doubles every RTT** (exponential, not slow).
- **Congestion avoidance:** above ssthresh, **+1 per RTT** (MSS × MSS / CWND per ACK).
- **Timeout:** ssthresh = CWND / 2, CWND = 1.
- Tahoe: loss = timeout. Reno: loss = timeout **or** 3 duplicate ACKs.

### 3.5 Tahoe, fast recovery and Reno

**In one sentence:** after three duplicate ACKs, Tahoe goes back to a window of 1, while Reno only halves the window and keeps sending (fast recovery).

#### TCP Tahoe

After slow start (CWND > threshold), add about one segment per RTT; on a loss event:

```
threshold = CWND / 2
CWND = 1
perform slow start
```

**Back to one**, whatever kind of loss it was. Tahoe throws away everything it learned about the path; on a long, fast link it takes many round trips just to get back to where it was.

**Technical clarification.** The lecture title and code spell it "Taheo"; the lecture's explanation points out it should be **Tahoe**.

#### Fast retransmit in the congestion-control story

- **Purpose:** detect and recover from loss quickly, without waiting for the retransmission timeout (RTO).
- **How:** 3 duplicate ACKs for the same packet → assume the next packet was lost.
- **Action:** retransmit immediately.

**Two loss signals of different severity:** three duplicate ACKs mean **later segments are still arriving**, so the path works. A **timeout** means **nothing came back**: a much worse sign. Tahoe's mistake was to react to both the same way.

> **Kid example.** If a friend says "I got pages 3, 4 and 5 but not page 2", the post is clearly still working; you just resend page 2. If your friend goes completely silent, something is badly wrong. It would be silly to react to both in the same way.

#### Fast recovery

- **Purpose:** recover quickly after a fast retransmit by **adjusting CWND** instead of restarting.
- Instead of going down to one segment, the sender **halves** the window and **keeps sending new packets**.

The arithmetic:

```
Threshold = CWND / 2
CWND = Threshold + 3
for each further duplicate ACK: CWND++
when a new ACK arrives (the retransmission was received):
    CWND = Threshold
    go back to congestion avoidance
```

**Why + 3?** The three duplicate ACKs mean three segments **left the network** and reached the receiver, so three fewer are in flight than the sender's count suggests; adding 3 corrects the count. Likewise, **each extra duplicate ACK** shows one more segment has left the network, so the sender may send one more without adding load. This is called **window inflation**: careful bookkeeping of what is really in flight. When the new ACK arrives, the inflation is undone (CWND = Threshold).

**Technical clarification.** The lecture spells "flight" as "flisht".

#### The key differences

| | Fast retransmit | Fast recovery |
|---|---|---|
| Objective | Quickly **resend** the lost packet | **Manage CWND** after the fast retransmit |
| Timing | Immediately on the 3rd duplicate ACK | The phase that **follows** fast retransmit |
| Effect on CWND | None directly | Halves it (plus 3), then grows it gradually |

They are always used together and they are not the same thing; an exam question about "the difference" is asking exactly this.

#### The whole picture

The figure plots CWND against round-trip times for Reno:

- **Slow start** from 1 up to **ssthresh = 28** (around RTT 5).
- **Congestion avoidance**: gentle climb to **36** at around RTT 13.
- **Three duplicate ACKs:** ssthresh(new) = 36 / 2 = **18**; CWND(new) = 18 + 3 = **21**.
- **Fast recovery:** CWND inflates a little with each further duplicate ACK.
- **ACK for the retransmitted segment:** CWND = ssthresh(new) = **18**; congestion avoidance resumes.

```mermaid
flowchart LR
    A["slow start<br/>1 → 28"] --> B["congestion avoidance<br/>28 → 36"]
    B -->|"3 duplicate ACKs"| C["ssthresh = 18<br/>CWND = 21"]
    C -->|"more duplicate ACKs:<br/>CWND + 1 each"| D["fast recovery"]
    D -->|"new ACK"| E["CWND = 18<br/>congestion avoidance"]
```

**Tahoe vs Reno, worked.** Initial ssthresh 16; three duplicate ACKs arrive when CWND = 24. Reno drops to 12 (ignoring the brief +3 inflation) and continues linearly; Tahoe drops to 1 and slow-starts up to its new threshold 12:

```mermaid
%%{init: {"themeVariables": {"xyChart": {"plotColorPalette": "#1f5fbf, #d62728"}}}}%%
xychart-beta
  title "CWND per RTT: Reno (blue) vs Tahoe (red)"
  x-axis "RTT" [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25]
  y-axis "CWND (segments)" 0 --> 26
  line [1, 2, 4, 8, 16, 17, 18, 19, 20, 21, 22, 23, 24, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24]
  line [1, 2, 4, 8, 16, 17, 18, 19, 20, 21, 22, 23, 24, 1, 2, 4, 8, 12, 13, 14, 15, 16, 17, 18, 19, 20]
```

The two lines are identical until RTT 12. After the loss, Tahoe needs four extra round trips to climb back to 12, and stays 4 segments behind Reno from then on in this example. On a long, fast path the difference is much larger.

> **Kid example.** You trip while running up a long staircase. Tahoe goes back to the very bottom step and starts again. Reno just steps back down to halfway and keeps climbing from there.

#### TCP Congestion Window Trace †

*(Hidden, archived slide.)* A real CWND trace is messier than the idealised sawtooth: variable peaks, uneven intervals, occasional deep cuts, because real paths have competing flows arriving and leaving, route changes and cross traffic.

#### TCP variants summary

- **TCP Tahoe:** slow start, congestion avoidance and fast retransmit. On 3 duplicate ACKs: fast retransmit and **CWND = 1**.
- **TCP Reno:** slow start, congestion avoidance, fast retransmit **and fast recovery**. On 3 duplicate ACKs: fast retransmit and **CWND = threshold + 3** "to keep the pipe full" (fast recovery).
- In both, the **receiver buffers out-of-order packets**; when the lost packet arrives via fast retransmit, the receiver sends a **cumulative ACK** covering everything it now has in order.
- Other implementations: Vegas, NewReno and SACK TCP.

| Event | Tahoe | Reno |
|---|---|---|
| 3 duplicate ACKs | ssthresh = CWND/2, CWND = 1, slow start | ssthresh = CWND/2, CWND = ssthresh + 3, fast recovery |
| Timeout | ssthresh = CWND/2, CWND = 1, slow start | ssthresh = CWND/2, CWND = 1, slow start |

**Technical clarification.** The lecture calls NewReno "the most commonly implemented on webservers today, according to a survey". That survey is old; the lecture's explanation says the algorithm in Linux today is **CUBIC**, which is what most web servers run.

**Remember:**
- **Tahoe:** any loss → ssthresh = CWND/2, **CWND = 1**, slow start.
- **Reno, 3 dup ACKs:** ssthresh = CWND/2, **CWND = ssthresh + 3**, +1 per extra dup ACK, then **CWND = ssthresh** on the new ACK.
- **Reno, timeout:** same as Tahoe (CWND = 1).
- **Fast retransmit** = resend now; **fast recovery** = manage CWND afterwards.
- Lecture figure: 36 → ssthresh 18, CWND 21 → 18.

### 3.6 TCP variants: NewReno, Vegas, Westwood, CUBIC, and ECN

**In one sentence:** later TCP versions repair several losses at once (NewReno), watch delay (Vegas), estimate bandwidth (Westwood), grow faster on big links (CUBIC), or let routers warn before loss (ECN).

#### Reno vs NewReno

- **Reno** improves congestion control after loss by entering fast recovery on duplicate ACKs.
- But Reno enters fast recovery even if only a single packet is lost, and it handles **several losses in one window** badly.
- **NewReno** reduces slowdowns and uses bandwidth better.

What NewReno actually does (quiz key, RFC 6582): Reno **leaves** fast recovery at the first new ACK, so when several segments from one window are lost it exits and re-enters fast recovery repeatedly, one round trip each time. NewReno uses **partial ACKs**: if a new ACK advances but does not cover everything that was outstanding when recovery began, another segment must also be lost; NewReno retransmits it at once and **stays in fast recovery until all lost packets of that window are acknowledged**. Buffer overflow tends to drop several consecutive packets, so this matters a lot.

```mermaid
sequenceDiagram
    participant S as NewReno sender
    participant R as Receiver
    Note over S,R: segments 2 and 5 of one window were lost
    R-->>S: 3 duplicate ACKs for 2
    S->>R: resend 2 (enter fast recovery)
    R-->>S: partial ACK 5 (moved forward, but not to the end)
    S->>R: resend 5 at once, still in fast recovery
    R-->>S: ACK covering the whole window
    Note over S: now leave fast recovery
```

**Technical clarification.** The lecture says NewReno "allows multiple packets to be transmitted before entering Fast Recovery". That is not the standard description; learn the partial-ACK description above, which matches the lecture's own explanation, RFC 6582 and the quiz key.

#### Vegas

**Vegas** watches the **variation in RTTs** and adjusts the sending rate: a rising RTT means a queue is building, so Vegas slows down **before** anything is dropped. It aims to keep queueing delay minimal and behaviour steady: **delay-based** congestion control, two decades before BBR.

Why it failed in practice: a delay-based sender backs off when it sees a queue forming, while a loss-based sender next to it keeps pushing until the buffer overflows. **Sharing a link, the polite algorithm gets less.**

> **Kid example.** At a buffet, a polite child steps back as soon as the queue gets long; a pushy child keeps pushing until the food runs out. The polite child ends up with less food. That is Vegas next to loss-based TCP.

**Technical clarification.** The lecture says Vegas is "not currently used (higher latency)". Vegas was designed for **lower** latency; the reason it is not used is the one above (it loses bandwidth to loss-based flows).

#### Westwood

**Westwood** estimates the available **bandwidth from the rate of ACKs** and sets CWND from that estimate (after a loss, it sets the window from the estimate instead of blindly halving). It aims for good performance on high-speed, long-distance networks, and the quiz associates it with **wireless** links, where many losses are not caused by congestion.

#### CUBIC

- **Default in Linux**, and the most popular TCP for web servers.
- A **smoother adjustment of CWND** during congestion avoidance, using a **cubic function** of the time since the last congestion event.
- **K** is the point in time at which the window will again reach **W_max**, the size it had when the last loss happened.

Additive increase is too slow on high-BDP paths: adding one segment per RTT to rebuild a window of tens of thousands of segments takes far too long. CUBIC instead grows **fast** while far below W_max, **flattens** near W_max (the level that caused trouble last time), then **speeds up again** beyond it to probe for new capacity. Growth depends on **elapsed time**, not on ACKs received, which makes CUBIC less unfair to long-RTT flows.

**Technical clarification.** The lecture's text reads "Large increase when near K, small increase near K", which is garbled. The correct statement: **small increase near K** (near W_max), **large increase far from K**.

**Additional context.** The CUBIC window (RFC 9438) is:

```math
W(t) = C \times (t - K)^3 + W_{\max}, \qquad K = \sqrt[3]{\frac{W_{\max} \times (1 - \beta)}{C}}
```

with β = 0.7 (after a loss the window drops to 70% of W_max) and C = 0.4. With W_max = 100 segments, K = ∛75 ≈ 4.2 seconds.

```mermaid
%%{init: {"themeVariables": {"xyChart": {"plotColorPalette": "#1f5fbf, #d62728"}}}}%%
xychart-beta
  title "CUBIC window after a loss (W_max = 100, K = 4.2 s)"
  x-axis "seconds since the loss" [0, 1, 2, 3, 4, 5, 6, 7, 8]
  y-axis "CWND (segments)" 60 --> 125
  line [70, 86.7, 95.6, 99.3, 100, 100.2, 102.3, 108.6, 121.7]
```

Fast at first, almost flat around 100 (near K), then faster again: the cubic shape.

> **Kid example.** You fell off your bike at a certain speed. Next time you speed up quickly at first, then ride carefully as you get close to the speed where you fell, and only when that feels safe do you speed up again to try going faster.

#### TCP BBR (as a variant)

BBR estimates the available **bottleneck bandwidth** and sets CWND to use the link fully **without causing queueing delay**. See [section 4.3](#43-bbr).

#### Explicit Congestion Notification (ECN)

- Improve congestion control via **router signals before packets are lost**.
- **ECN bits:** special bits in the **IP header**. A congested router **marks** a packet instead of dropping it.
- **Receiver's role:** its ACK tells the sender that congestion was seen (via ECN fields in the **TCP header**).
- **Sender's response:** reduces its rate **in advance**, before any loss.

```mermaid
sequenceDiagram
  participant S as Sender
  participant R as Congested router
  participant D as Receiver
  S->>R: packet (ECN-capable)
  R->>D: same packet, marked "congestion experienced" in the IP header
  D-->>S: ACK with the congestion echo flag in the TCP header
  Note over S: reduce CWND, without any packet having been lost
```

> **Kid example.** Without ECN, the bus driver just leaves people behind when the bus is full. With ECN, the driver puts a sticker on your ticket saying "the bus is getting full", your friend at the other end tells you about the sticker, and next time you take an earlier bus. Nobody had to be left behind.

No data is destroyed: no retransmission and no waiting for loss detection. But the signal crosses a layer boundary in each direction (IP → TCP → sender), so **router, receiver and sender** must all support it, which slowed deployment. And because a classic ECN sender reacts to a mark **exactly as to a loss** (halving), marks must stay rare: the limitation L4S removes.

**Remember:**
- **NewReno** (RFC 6582): **partial ACKs**, stays in fast recovery until the whole window is repaired.
- **Vegas**: delay-based; loses bandwidth to loss-based flows (not "higher latency").
- **Westwood**: bandwidth from the ACK rate; good for **wireless**.
- **CUBIC** (Linux default): cubic in time; **small increase near K / W_max**, large far from it; β = 0.7, C = 0.4.
- **ECN**: router **marks** (IP header), receiver **echoes** (TCP header), sender slows down; all three must support it.

### 3.7 The history of TCP

**In one sentence:** TCP was described in 1974, standardised in 1981, and got its congestion control only in 1988, after the network had already collapsed.

| Year | Event |
|---|---|
| 1974 | TCP described by Vint Cerf and Bob Kahn (IEEE Transactions on Communications). |
| 1975 | Three-way handshake (Ray Tomlinson, SIGCOMM 75). |
| 1981 | RFC 793 and RFC 791 standardise TCP and IP. |
| 1983 | BSD Unix 4.2 supports TCP/IP. |
| 1984 | Nagle's algorithm reduces the overhead of small packets and predicts congestion collapse. |
| 1986 | Congestion collapse first observed. |
| 1987 | Karn's algorithm for better RTT estimation. |
| 1988 | Van Jacobson's algorithms: slow start, congestion avoidance, fast retransmit (4.3BSD Tahoe, SIGCOMM 88). |
| 1990 | 4.3BSD Reno: fast recovery, delayed ACKs. |
| 1993–94 | TCP Vegas (Brakmo et al., not widely implemented), T/TCP (Braden), ECN proposed (Floyd). |
| 1996 | NewReno (modified fast recovery), SACK (Floyd et al.), FACK (Mathis et al.), improved startup (Hoe). |

```mermaid
timeline
    title TCP history
    1974 : Cerf and Kahn describe TCP
    1981 : RFC 793 and RFC 791
    1984 : Nagle predicts collapse
    1986 : collapse observed
    1988 : Jacobson (Tahoe)
    1990 : Reno
    1996 : NewReno and SACK
```

Two observations: **Nagle predicted congestion collapse before it happened, and it happened anyway**; and several good ideas took decades to matter (ECN, proposed in 1994, had little effect until L4S in the 2020s; Vegas's idea "measure delay, not loss" only succeeded with BBR in 2016). "Good ideas do not deploy because they are good. They deploy when the conditions that make them necessary arrive, and when enough parties can act at once."

**Technical clarification.** The lecture's timeline spells Nagle as "Nagel". The years in the table are given in the lecture's explanation; its timeline picture shows the events without all the dates.

**Remember:**
- 1974 Cerf and Kahn; 1981 RFC 793/791; 1986 collapse; **1988 Jacobson (Tahoe)**; 1990 Reno; 1996 NewReno and SACK.
- Congestion control was **not in the original TCP**.
- Good ideas (ECN, delay-based control) can wait decades for the right conditions.

## 4. Modern transport: beyond loss-based control

**What this chapter is about:** what changed after 1996: huge buffers made the Internet slow (bufferbloat), routers learned to manage their queues (AQM), and new ideas (BBR, L4S, QUIC) try to keep delay low. It ends with the ways TCP can fail while every part works "correctly".

### 4.1 Part B: thirty years you have not been taught

**In one sentence:** the textbook story stops in 1996, but since then CUBIC, ECN, bufferbloat, BBR, QUIC and L4S have changed transport a lot.

*(Part B, "Beyond EDA344": new compared with an introductory course.)*

The standard story ends around **1996**: Tahoe, Reno, NewReno, SACK. Since then:

| Development | What it is |
|---|---|
| **CUBIC** as default (2006) | A cubic function decides how fast the window grows. |
| **ECN** deployment | Routers warn endpoints by **marking** packets instead of dropping them. |
| **Bufferbloat** (named about 2010–11) | Harmful delay caused by excessively large buffers in routers and switches. |
| **BBR** (2016) | Google's algorithm that models the path's capacity instead of reacting to loss. |
| **QUIC** (standardised 2021) | A transport protocol running over UDP, designed to replace TCP for most web traffic. |
| **L4S** (RFCs 2023) | Low Latency, Low Loss, Scalable throughput: a redesigned congestion signal to cut queueing delay. |

```mermaid
timeline
    title After the textbook (1996)
    2006 : CUBIC becomes the default
    2010 : bufferbloat named
    2016 : BBR
    2021 : QUIC standardised
    2023 : L4S RFCs
```

"Congestion control stopped being a solved problem and became an active one." Every assumption under the 1996 design changed: buffers became huge, wireless made loss a poor congestion signal, **latency** (not throughput) became what users notice, and transport moved **out of the operating system**. None of Part A is wrong (CUBIC is still the Linux default and a direct descendant); the story is just **incomplete**.

**Remember:**
- After 1996: **CUBIC (2006)**, **ECN**, **bufferbloat (≈ 2010–11)**, **BBR (2016)**, **QUIC (2021)**, **L4S (2023)**.
- What changed: huge buffers, wireless loss, **latency matters most**, transport moved into apps.

### 4.2 Bufferbloat and AQM

**In one sentence:** loss-based TCP fills every buffer it meets, so huge buffers cause huge delays; the fix is for routers to drop or mark packets early (AQM).

#### Bufferbloat

- **Loss-based congestion control fills whatever buffer it finds.**
- Memory got cheap, so buffers got deep: **seconds** of queueing delay.
- **Throughput stayed fine; latency collapsed.**
- "More memory made the Internet slower. The protocol was working as designed."

A loss-based sender keeps increasing its window until something is dropped. With a small buffer that happens quickly and the queue stays short. With a **huge** buffer, the sender keeps going, **filling memory rather than the link**, until the buffer is full. The link is busy all along, so throughput looks fine, but every packet now waits behind a buffer-full of someone else's big download. On home routers of the late 2000s this reached **whole seconds**. Vendors added memory believing that fewer drops meant better performance; they were right about drops and wrong about performance. The name and diagnosis come from **Jim Gettys** around 2010.

```mermaid
flowchart TD
  A["data in flight below the BDP:<br/>link not full"] -->|"in flight = BDP"| B["link full, no queue:<br/>best point (BBR aims here)"]
  B -->|"more data in flight"| C["queue grows:<br/>same throughput, more delay"]
  C -->|"in flight = BDP + buffer size"| D["buffer full: loss<br/>(loss-based TCP backs off here)"]
```

Everything between the BDP point and the buffer-full point is **queue: pure delay with no gain in throughput**. Bigger buffers just stretch that middle region.

> **Kid example.** Bufferbloat is a huge waiting room: nobody is turned away, but you wait an hour. A small waiting room turns people away sooner, so the crowd learns to come back later, and everyone inside waits only a little.

#### AQM: managing the queue instead

- **If senders cannot detect a full buffer, the buffer must tell them.**
- **CoDel:** drop based on **how long** packets have been queued, not **how many** are queued.
- **FQ-CoDel:** **per-flow queueing**, so one bulk transfer cannot starve an interactive one.
- **The fix moved from the endpoint to the middle of the network.**

CoDel changes what triggers a drop: queue length in packets is a poor stand-in for delay (it breaks whenever bandwidth changes), so CoDel measures how long packets actually sit and acts on **persistent standing delay**: a control rule on **time**, not occupancy. FQ-CoDel adds **fair queueing**: separate queues per flow, so a download and a video call do not share one queue. Most of the improvement people notice comes from the fair-queueing part.

```mermaid
flowchart LR
  in["arriving packets"] --> cls["sort by flow"]
  cls --> q1["queue: video call<br/>(short, CoDel)"]
  cls --> q2["queue: bulk download<br/>(CoDel drops if packets wait too long)"]
  cls --> q3["queue: web page<br/>(CoDel)"]
  q1 --> sch["fair scheduler<br/>takes turns"]
  q2 --> sch
  q3 --> sch
  sch --> out["output link"]
```

> **Kid example.** FQ-CoDel is like a supermarket with a separate checkout lane for each customer group, and a cashier who serves the lanes in turn. Someone with three full trolleys (the big download) no longer blocks the person buying one apple (the video call). And if anyone has been waiting too long, the manager asks a few trolley-pushers to come back later (CoDel's drop).

**Architectural tension:** the end-to-end argument says to put functions at the endpoints, but here the endpoints **could not** solve the problem (a sender cannot observe a queue nobody tells it about), so the network had to step in.

**Additional context.** The quiz also asks about **DropTail** and **RED**. DropTail is the plain rule "when the queue is full, drop every new packet": many flows lose packets at the same moment, all halve together and grow together (**synchronised losses**), wasting capacity and hurting fairness. **RED (Random Early Detection)**, an early AQM scheme, drops or marks a few **random** packets as the **average** queue grows, before it is full, so senders slow down early and at different times.

**Remember:**
- **Bufferbloat:** loss-based TCP fills big buffers → seconds of delay, throughput still fine (Jim Gettys, ≈ 2010).
- Between "in flight = BDP" and "buffer full" there is **only extra delay**.
- **CoDel:** drop on **time waited**; **FQ-CoDel:** one queue per flow.
- **DropTail:** synchronised losses; **RED:** random early drops on the average queue.

### 4.3 BBR

**In one sentence:** BBR measures the path's bottleneck bandwidth and minimum RTT and sends exactly enough to fill the link without building a queue.

#### BBR: asking a different question

- Loss-based congestion control asks: **has something been dropped yet?**
- BBR asks: **what is the bottleneck bandwidth, and what is the minimum RTT?**
- Send at the **estimated bandwidth-delay product**; **do not fill the queue at all**.
- "**A model of the path, not a reaction to its failure.**"

Every algorithm in Part A is **reactive**: send more until something breaks, then less; by the time loss arrives, the damage (a full buffer) is done. BBR continuously estimates two properties of the path: the **bottleneck bandwidth** (the highest delivery rate seen recently) and the **minimum RTT** (the RTT when no queue has built). Their product is how much data can be in flight with the link fully used and the queue empty. In the flowchart of [section 4.2](#42-bufferbloat-and-aqm), loss-based control operates at the **buffer-full** point; BBR targets the **BDP** point. BBR is not a refinement of AIMD or CUBIC; it is a **different paradigm**.

```mermaid
flowchart LR
    BW["bottleneck bandwidth<br/>(highest delivery rate seen)"] --> X["×"]
    RT["minimum RTT<br/>(RTT with no queue)"] --> X
    X --> T["target in flight = BDP:<br/>link full, queue empty"]
```

> **Kid example.** Loss-based TCP pours water into a funnel until it overflows, then pours less. BBR first measures how fast the funnel drains and how long the water takes to go through, and then pours exactly that fast, so the funnel never fills up.

#### BBR's status

- **BBRv3** is specified in **draft-ietf-ccwg-bbr, revision 06, July 2026**.
- Still **Experimental** after **nine years and three major versions**.
- The draft specifies **no ECN response**: an open question, not an oversight.
- "Widely deployed and not yet standardised are compatible states."
- (The lecture says to verify the current revision at datatracker.ietf.org.)

People assume standardisation comes before deployment; in transport it often runs the other way: an operator with enough traffic can deploy on its own, and the standard then documents what already exists. The pattern has now appeared three times: **RPKI** (about two-thirds of prefixes signed, about an eighth of networks checking), **ASPA** (standardised, adopted by well under 1%), and **BBR** (deployed globally, still Experimental). "Specified, deployed, and adopted are three different states."

#### Does BBR play fair?

- An **independent evaluation** (PAM 2024, by MPI-INF and VU Amsterdam) tested the claim.
- **BBRv3 keeps BBRv1-like unfairness towards CUBIC in shallow buffers.**
- A **persistent bias favouring long-RTT flows** remains.
- "Measured behaviour contradicting the deployment narrative."

Be precise: this does **not** show that BBR is bad or that its designers were dishonest. It shows that a central claim about a globally deployed algorithm was tested **by outsiders** and did not fully hold. "When a claim is made about a deployed system, ask who measured it, on what, and what the method could not see."

**Remember:**
- BBR targets **bottleneck bandwidth × minimum RTT** (the BDP point), not the buffer-full point.
- **BBRv3**: draft revision 06 (July 2026), still **Experimental**, **no ECN response** specified.
- PAM 2024: BBRv3 still **unfair to CUBIC in shallow buffers**, **favours long-RTT flows**.

### 4.4 L4S and QUIC

**In one sentence:** L4S gives low-latency traffic its own queue with frequent, gentle congestion marks, and QUIC moves congestion control out of the operating system into the application.

#### L4S: Low Latency, Low Loss, Scalable Throughput

- Specified in **RFC 9330, 9331, 9332**.
- **Two queues**: classic traffic and scalable traffic, separated by the **ECT(1)** marking.
- **Mark early and often** rather than drop, so senders react **before** a queue builds.
- "**Congestion signalling redesigned** rather than congestion response retuned."

Classic ECN senders treat a mark like a loss and halve their window, so marks must be rare and the signal stays coarse. L4S separates the traffic: packets marked **ECT(1)** go into a **low-latency queue** whose AQM marks **early and often**, and senders using a **scalable** congestion control respond to each mark with a **small, proportional** step instead of halving. Classic traffic keeps its own queue and behaviour, and the two queues are **coupled** so neither starves the other.

```mermaid
flowchart LR
  in["arriving packets"] --> sep{"ECN field = ECT(1)?"}
  sep -->|"yes: L4S traffic"| lq["low-latency queue:<br/>marks early and often"]
  sep -->|"no: classic traffic"| cq["classic queue:<br/>classic AQM"]
  lq --> link["output link"]
  cq --> link
  lq <-.->|"coupled AQM so<br/>neither queue starves"| cq
```

> **Kid example.** Classic ECN is a teacher who only says "STOP!" very rarely, and then everyone stops completely. L4S is a teacher who gently says "a bit slower… a bit slower… OK, a bit faster" all the time, and the children adjust a little each time. The class never has to stop, and nobody piles up at the door.

Compare with CoDel: CoDel manages **one shared queue** better; L4S **separates** the traffic classes and **changes what the signal means**. (An L4S deployment runs AQM in both queues.)

#### L4S is actually deployed

- **Comcast** has rolled it out across its production network.
- **Apple** devices support it from **iOS 17 and macOS Sonoma** onward.
- Not a paper standard: running in consumer networks now.
- Compare with ASPA: standardised, available, barely used.

L4S needs the **network and the endpoint to cooperate**, which normally makes deployment harder. It happened because **two organisations big enough to matter each controlled one side** and coordinated. ASPA needs thousands of independent networks to act with no coordinator. "The difference is not technical merit. It is how many parties must move, and whether any single one of them benefits before the others do."

#### L4S measured, honestly

- A **March 2026 field study**: **83 households**, about **120,000 controlled experiments**.
- **Up to about 25% tail-latency reduction** for interactive traffic.
- **Minimal gains** for some workloads; benefits concentrated at **peak hours**.
- "A real improvement, bounded and situational, not a universal win."

The improvement is in the **tail** (the worst delays), not the **mean**: the right metric for interactive traffic like video calls (which suffer from the occasional long delay), and the wrong one for a throughput chart. And benefits appear at peak hours, when there is congestion to relieve, which a vendor benchmark on an idle link would miss.

> **Kid example.** "Tail latency" is about your worst days, not your average day. If your bus is usually on time but once a week is 30 minutes late, you remember the late days. L4S mostly makes those rare bad days less bad.

#### QUIC moves congestion control to userspace

- TCP congestion control lives **in the kernel** (the core of the operating system): changing it means changing the operating system.
- **QUIC runs in the application**, so the algorithm **ships with the browser**.
- **Deployment speed increases; so does fragmentation** across implementations.
- "**Who controls the algorithm determines how fast it can change.**"

```mermaid
flowchart TB
  subgraph tcp["TCP"]
    app1["application"] --> k["kernel: TCP with its congestion control<br/>(changes with OS releases: years)"]
    k --> ip1["IP"]
  end
  subgraph quic["QUIC"]
    app2["application with QUIC and its own congestion control<br/>(changes with app releases: weeks)"] --> udp["kernel: UDP"]
    udp --> ip2["IP"]
  end
  tcp ~~~ quic
```

A browser vendor can change congestion control for hundreds of millions of users in weeks; BBR spread quickly partly because it did not have to wait for operating systems. The cost: there is **no longer one congestion control** on the Internet. Different applications run different algorithms with different fairness properties on the same links, with no coordination. "If every application shipped its own congestion control, what would prevent a race to the most aggressive one?"

> **Kid example.** Changing TCP is like changing the traffic rules of a whole country: every car (operating system) must be updated. QUIC is like a delivery company that writes its own driving rules into its own app: it can change them tomorrow, but different companies may then drive by different rules.

#### Real-world applications †

*(Hidden, archived slide.)* Congestion control matters for **web browsing** (pages without needless delay), **streaming** (less buffering), **file downloads** (fast without destabilising the network), **online gaming** (stable, low lag) and **cloud services** (responsiveness). They map to Part B: gaming cares about the **tail latency** that bufferbloat destroys and L4S improves; streaming wants steady throughput, which BBR's rate estimation targets; downloads on long fat paths are what CUBIC was designed for.

**Remember:**
- **L4S** (RFC 9330–9332): **ECT(1)** traffic in a low-latency queue, **marked early and often**, senders react **proportionally**; coupled with the classic queue.
- Deployed by **Comcast** and **Apple (iOS 17, macOS Sonoma)**.
- Field study (March 2026, 83 homes, ≈ 120,000 tests): **up to ≈ 25% less tail latency**, mainly at peak hours.
- **QUIC**: congestion control in the **app** → fast changes, but **many different algorithms**.

### 4.5 How TCP fails, and summary

**In one sentence:** TCP can follow its rules perfectly and still end up in a bad state: slowing down for wireless noise, settling into an unfair split, or trusting an old RTT.

#### How TCP fails

- A sender **cannot tell congestion loss from wireless loss**, and backs off either way.
- **Competing algorithms reach an equilibrium neither would choose alone.**
- A **route flap changes the RTT** under an estimator built assuming it is stable.
- "**Convergence to a stable state is not convergence to a good one.**"

Each is correct behaviour producing a bad outcome:

1. On a lossy wireless path, TCP throttles for no reason, and no signal is available to tell the cases apart.
2. Two senders with different algorithms settle into a split neither would have chosen. Nothing failed; the system **converged, somewhere bad**.
3. After a route change, the RTT estimator (and its variance) describes a path that no longer exists.

```mermaid
flowchart LR
    W["wireless noise<br/>drops a packet"] --> T["TCP thinks:<br/>congestion!"] --> H["halves its speed<br/>for no reason"]
```

> **Kid example.** A careful driver brakes whenever a car in front brakes. On a bumpy road, the bumps look like braking too, so the careful driver keeps slowing down even though the road is empty. The driver follows the rule perfectly and still drives badly.

This is the fourth of six closing slides with the same theme: BGP wedgies (converged to a state nobody wanted), DNS (serves wrong answers it cannot detect), IPv6 (failures masked by the mechanism meant to smooth them), and now TCP. "What would a protocol need in order to leave a bad state by itself? The second half of this course answers it" (self-stabilization).

#### Summary: TCP

- The **major transport service** on the Internet.
- **Connection-oriented.**
- Provides **end-to-end reliability.**
- Includes **flow/error control and congestion control.**

Each line was contested at some point: connection orientation (QUIC revisits it), end-to-end reliability (ECN and L4S move some responsibility into the network), and congestion control, which was **not in the original protocol at all**: it was added in 1988 after the network began collapsing.

#### Interactive exercises

*(Self-study.)* The lecture lists the Kurose–Ross interactive exercises and animations (gaia.cs.umass.edu/kurose_ross/interactive/): TCP sequence and ACK numbers with segment loss; RTT and timeout; congestion window evolution; retransmissions with ACK loss; animations of Go-Back-N, Selective Repeat, flow control and TCP congestion control. If you are short of time, the lecture recommends the **congestion window** and the **RTT and timeout** exercises.

**Remember:**
- Three failures: **wireless loss mistaken for congestion**, **bad equilibrium between algorithms**, **stale RTT after a route change**.
- "Converging to a stable state is not converging to a good one" → motivates **self-stabilization**.
- TCP: connection-oriented, reliable, flow/error control and congestion control (added **1988**).

## 5. Review questions

**What this chapter is about:** worked answers to the lecture's check questions, the hidden review questions, and quiz topics that the lecture does not cover.

### 5.1 Check yourself: worked answers

These are answers to some of the "Check yourself" questions in the lecture, worked out with the lecture's formulas.

| Question | Answer |
|---|---|
| Stop-and-wait: transmitting takes 1 ms and the RTT is 100 ms. What fraction of time is the sender sending? | 1 / (100 + 1) ≈ **1%**. |
| The 1 Gbps example link with a satellite path of 250 ms each way? | U = 0.008 / (500 + 0.008) ≈ **0.0016%**: even worse; the propagation delay is what matters. |
| What window saturates the 1 Gbps example link? | About (30 + 0.008) / 0.008 ≈ **3,751 packets** (the BDP is 3,750 packets of 8,000 bits). Twice that just builds a queue. |
| CWND = 64 KB, RTT = 20 ms: throughput? | 65,536 × 8 bits / 0.020 s ≈ **26 Mbit/s**. If the path's RTT grows, the rate falls in proportion. |
| Two flows with RTTs 10 ms and 100 ms share a link: who wins? | The **10 ms** flow, roughly **10 : 1** for classic window-based TCP (throughput ∝ 1/RTT). |
| Slow start from CWND = 1: round trips to reach 64? | **6** (1 → 2 → 4 → 8 → 16 → 32 → 64). Additive increase would need **63**. |
| A 20 kB transfer with an initial window of 10 segments: does it reach congestion avoidance? | With a 1,460-byte MSS, 10 segments ≈ 14.6 kB in the first RTT and the rest in the second, so it **finishes in slow start**. |
| With α = 1/8, how fast does the estimate follow a step change? | After n samples, (7/8)ⁿ of the old value remains: about **half after 5 samples** and about 90% of the change after 17. |
| If the true RTT suddenly doubles, is the first timeout too short or too long? | Probably **too short** (the estimate lags), causing a needless retransmission; TCP then backs off (doubles the timeout) and Karn's rule stops it from learning from the retransmitted segment. |
| What does a receive window pinned near zero say about an application? | The application is **not reading** its data fast enough: the receiver, not the network, is the bottleneck. |
| Which three parties must support ECN? | The **sender**, the **routers** that mark, and the **receiver** that echoes. If one is missing, routers fall back to dropping, or the mark is never echoed. |
| How long until a 32-bit sequence number space wraps at 10 Gbit/s? | 2³² bytes ≈ 34.4 Gbit, so about **3.4 seconds**. |
| What window fills a 1 Gbit/s path with a 100 ms RTT? | 10⁹ × 0.1 = 10⁸ bits = **12.5 MB**, about 190 times the 64 KB maximum of the original header (hence the window scale option). |
| For each of Tahoe and Reno, CWND after 3 duplicate ACKs and after a timeout? | Tahoe: 1 and 1. Reno: ssthresh + 3 (then ssthresh) and 1. |
| Reno vs NewReno when three segments are lost from one window? | Reno repairs about **one loss per round trip** (re-entering recovery, often ending in a timeout); NewReno repairs all three in **one recovery episode**, one partial ACK per lost segment. |

**How to do the throughput question step by step.** 64 KB = 65,536 bytes = 524,288 bits. Divide by 0.020 s: 524,288 / 0.02 = 26,214,400 bits/s ≈ **26 Mbit/s**.

### 5.2 Review questions †

*(Hidden material †.)*

**1. A finite sequence number field.** "TCP uses a finite field to contain stream sequence numbers. How does it allow an arbitrary-length stream?" The 32-bit numbers **wrap around** and are compared with modulo arithmetic. The constraint is how fast you may wrap: on a very fast link, numbers can wrap while old segments are still in the network, which is what **PAWS** (protection against wrapped sequence numbers, using timestamps) addresses.

> **Kid example.** A car's odometer with only six digits goes from 999,999 back to 000,000. That is fine as long as you never confuse "10 km" with "1,000,010 km". Sequence numbers wrap the same way.

**2. Unnecessary retransmissions.** "Under what conditions of delay, bandwidth, load and packet loss will TCP retransmit significant volumes of data unnecessarily?" When timeouts fire on segments that were only **delayed** (sudden RTT increases, high jitter, heavy load), and when a whole window is resent although only one segment was missing (for example in Go-Back-N-like behaviour, or after a timeout in Tahoe).

**3. A lost ACK need not force retransmission.** Because TCP ACKs are **cumulative**: a later ACK (for example ACK = 300) also confirms everything an earlier, lost ACK (ACK = 200) would have confirmed, as long as it arrives before the timer expires.

**4. Discarding out-of-order segments.** "Imagine a TCP that discards segments that arrive out of order, even inside the window. Does it work?" **Yes, it is still correct**: anything discarded will be retransmitted. But **performance collapses**: one loss forces retransmission of everything after it, and duplicate ACKs (the basis of fast retransmit) become far less useful. "A protocol can be correct and useless."

**5. Initial sequence number always 1.** "Explain how a crash and restart can confuse a remote system into believing that the old connection remained open." After a restart, the host opens a new connection with the **same** ports and the **same** starting sequence number 1. Segments of the new connection then fall inside the sequence space the remote side still expects for the **old** connection, so the remote side accepts them as if the old connection had continued (and old delayed segments can also be mistaken for new ones). That is why initial sequence numbers are chosen from a clock or randomly.

**6. Window-limited throughput.** "Maximum window 64 KB, unbounded bandwidth, average RTT 20 ms: maximum throughput? And with RTT 40 ms?" Throughput = window / RTT = 64 KB / 20 ms ≈ **26 Mbit/s** (3.2 MB/s). With 40 ms it **halves** to about **13 Mbit/s**. On a path with infinite bandwidth, the **window and RTT alone** decide throughput: the reason for the window scale option.

### 5.3 Related quiz topics not in the lecture

**Additional context.** The course's pre-lecture TCP quiz (29 questions) asks about a few things that the lecture does not cover directly. Short explanations:

- **The three-way handshake.** TCP opens a connection with **SYN** (client: "let's talk, my first sequence number is x"), **SYN-ACK** (server: "OK, mine is y, I expect x + 1") and **ACK** (client: "I expect y + 1").

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  C->>S: SYN, seq = x
  S-->>C: SYN-ACK, seq = y, ack = x + 1
  C->>S: ACK, ack = y + 1
  Note over C,S: connection open: data can flow both ways
```

> **Kid example.** The handshake is like starting a phone call: "Hello, can you hear me?" — "Yes, I can hear you; can you hear me?" — "Yes!" Only then do you start talking.

- **Sequence and ACK numbers.** The sequence number of a segment is the number of its **first data byte** in the stream; the ACK number is "the **next byte** I expect". Example: seq = 1 with 100 bytes (1–100) is answered by ACK = 101. Sequence numbers are 32 bits so that old, delayed segments are very unlikely to be confused with new ones when numbers wrap.
- **RTO too small or too large.** Too small: **needless retransmissions** of packets that were only slow. Too large: **slow recovery** after a real loss.
- **SYN flood.** An attacker sends thousands of SYNs and never completes the handshake; the server keeps memory for each half-open connection and runs out.
- **SYN cookies.** The server keeps **no** memory for half-open connections. It encodes the needed information (a cryptographic hash of addresses, ports and time) into the **initial sequence number** it sends in the SYN-ACK. A real client's final ACK returns that number + 1, so the server can rebuild the state then.
- **TCP reset (RST) attack.** A forged RST segment kills a connection if its sequence number falls inside the expected window; **predictable** sequence numbers make this feasible.
- **Wireless.** On Wi-Fi many losses come from radio noise, not congestion; standard TCP treats every loss as congestion and halves its speed for nothing (the first failure in [section 4.5](#45-how-tcp-fails-and-summary)).

> **Kid example.** A SYN flood is like a prankster ringing a restaurant 1,000 times to book a table and never turning up: the restaurant keeps all the tables reserved and real guests cannot get in. SYN cookies are like the restaurant saying "we don't reserve anything; just show us this special code when you actually arrive".

## Revision notes and references

Technical clarifications, where the lecture material is corrected or made precise, appear in sections [2.3](#23-when-to-retransmit-rtt-estimation-and-timeouts), [3.4](#34-van-jacobsons-tcp-slow-start-and-congestion-avoidance), [3.5](#35-tahoe-fast-recovery-and-reno), [3.6](#36-tcp-variants-newreno-vegas-westwood-cubic-and-ecn), [3.7](#37-the-history-of-tcp). Where the slides and the course book differ, this guide follows the book and says so at that point.

- J. Kurose and K. Ross, *Computer Networking: A Top-Down Approach*: transport layer chapter.
- [RFC 9293 — TCP](https://www.rfc-editor.org/rfc/rfc9293), [RFC 6298 — computing TCP's retransmission timer](https://www.rfc-editor.org/rfc/rfc6298) and [RFC 5681 — TCP congestion control](https://www.rfc-editor.org/rfc/rfc5681).
- V. Jacobson, "Congestion Avoidance and Control", ACM SIGCOMM 1988: slow start and congestion avoidance.
- [RFC 6582 — NewReno](https://www.rfc-editor.org/rfc/rfc6582), [RFC 3168 — ECN](https://www.rfc-editor.org/rfc/rfc3168) and [RFC 9438 — CUBIC](https://www.rfc-editor.org/rfc/rfc9438): later loss-based TCP variants and explicit congestion notification.
- N. Cardwell et al., "BBR: Congestion-Based Congestion Control", *ACM Queue* 14(5), 2016; [RFC 9330 — L4S architecture](https://www.rfc-editor.org/rfc/rfc9330); [RFC 9000 — QUIC](https://www.rfc-editor.org/rfc/rfc9000).
