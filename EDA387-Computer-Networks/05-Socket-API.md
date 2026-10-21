# The Socket API

**Course:** EDA387/DIT661 Computer Networks, "Socket API" (three lecture sessions plus self-study material). Session one builds a working client and server and explains why the interface looks the way it does. Session two goes through the elementary calls and addresses. Session three is about many clients at once, attacks on servers, scaling to ten thousand connections and more, and what people build on top of sockets. The code comes from W. Richard Stevens, *UNIX Network Programming*. About two thirds of the material is marked **Self-study**; it is "not filler and not optional", so it is included here like everything else.

**Revised:** 21 October 2026

**† Supplementary material:** content that comes only from hidden PowerPoint slides. It may not be part of this year's examinable material. In this lecture the hidden material is the three self-assessment checklists (section 8.2) and part of the discussion of TIME_WAIT (section 3.4). Text from them is marked with †.

**How to read this guide.** Every part starts with **In one sentence**, the whole idea in one line. Then comes the explanation in small steps, a **Kid example** (the same idea told with everyday things), pictures, and a **Remember** list with what you must know for the exam. **Technical clarification** marks a place where the slides are wrong or unclear, and gives the correct version. **Additional context** adds something useful that is not on the slides.

**No programming knowledge needed.** The lecture contains C code. Every piece of code below is explained line by line in plain words, so you can follow the ideas even if you have never programmed. Three words are enough to start:

- A **function** is a named command (like `connect`) that a program can call. It is like pressing a button on a machine.
- The values in brackets, like `connect(sockfd, …)`, are what you **give** the function, like the coins you put into a vending machine.
- What the function **hands back** is its **return value**, like the snack that drops out. Very often a negative return value means "something went wrong".

---

## Glossary

| Term | Full name | Meaning in simple words |
|---|---|---|
| Active close | — | The side that calls `close` first; it sends the first FIN and later sits in TIME_WAIT. |
| Active open | — | The client side of opening a connection: calling `connect`, which sends a SYN. |
| AF_INET / AF_INET6 | Address Family Internet (version 4 / version 6) | The constant that says "this socket or address uses IPv4" (or IPv6). |
| API | Application Programming Interface | The set of functions a program may call to use some service, here the network. Like the buttons on a vending machine. |
| Backlog | — | The second argument of `listen`: roughly how many waiting connections the kernel may queue. |
| Big-endian | — | Storing a multi-byte number with its most significant ("big") byte first. The network uses this order. |
| Blocking call | — | A call that makes the program wait (sleep) until the work is done. |
| BSD | Berkeley Software Distribution | The version of Unix from the University of California, Berkeley, where sockets were created (4.2BSD, 1983). |
| Buffer | — | A piece of memory where bytes are kept for a while, like a bucket that collects water before you use it. |
| Byte stream | — | A flow of bytes with no built-in boundaries between "messages"; what TCP gives you. |
| C10K / C10M | — | The challenge of one server handling ten thousand (1999) or ten million (2013) connections at once. |
| Client | — | The program that starts a conversation and asks for something (a browser, `ssh`, the daytime client). |
| Concurrent server | — | A server that serves several clients at the same time. |
| Connected descriptor | — | The new descriptor returned by `accept`, used to talk to one particular client. |
| Daemon | — | A long-running background program, such as a server. |
| Descriptor (file descriptor, fd) | — | A small whole number (0, 1, 2, 3 …) that a program uses to name an open file or socket. Like a ticket number at a cloakroom. |
| DPDK | Data Plane Development Kit | A library that lets a program drive the network card directly, bypassing the kernel. |
| EINTR | "Interrupted" error code | A slow call was interrupted by a signal; just try the call again. |
| EOF | End Of File | "No more data will come." On a socket: `read` returns 0 after the peer's FIN. |
| Ephemeral port | — | A short-lived port number that the kernel picks for the client side of a connection. |
| epoll / kqueue | — | Modern Linux (epoll) and BSD (kqueue) replacements for `select` that scale to many connections. |
| errno | — | A variable where the system stores the reason for the last error. |
| fd_set | — | A row of on/off bits, one per descriptor number, used by `select`. |
| FD_SETSIZE | — | The fixed size of `select`'s descriptor set, usually 1024. |
| FIN | Finish | A TCP segment that says "I have finished sending". |
| Four-tuple | — | (local IP, local port, remote IP, remote port): the four values that identify a TCP connection. Also called the socket pair. |
| Full-duplex | — | Data can flow in both directions at the same time, like a phone call (not a walkie-talkie). |
| GDB | GNU Debugger | A tool to run a program step by step and look inside it. |
| getaddrinfo | — | The modern function that turns a host name into a list of addresses (IPv4 and IPv6). |
| Half-close | — | Closing only the sending direction of a connection while still receiving. |
| htons / htonl / ntohs / ntohl | host to network short / long, network to host short / long | Functions that convert numbers between the computer's byte order and network byte order. |
| I/O multiplexing | — | Waiting on many descriptors at once and handling whichever is ready (with `select`). |
| INADDR_ANY | — | The "wildcard" IPv4 address: accept connections on any of this host's addresses. |
| io_uring | — | A modern Linux interface for asynchronous I/O using shared ring buffers. |
| Iterative server | — | A server that serves one client completely before the next. |
| Kernel | — | The protected core of the operating system; it runs TCP/IP and owns the network card. |
| Listening descriptor | — | The descriptor on which a server waits for new connections. |
| Little-endian | — | Storing a multi-byte number with its least significant ("little") byte first (Intel/AMD PCs). |
| MSL | Maximum Segment Lifetime | The longest time a TCP segment can survive in the network. |
| Multihomed host | — | A computer with more than one network interface (and IP address). |
| Network byte order | — | The agreed byte order on the wire: big-endian. |
| Non-blocking I/O | — | A socket mode in which a call that would have to wait returns an error at once instead. |
| Passive close | — | The side that receives the first FIN. |
| Passive open | — | The server side of opening: `socket`, `bind`, `listen`, then waiting. |
| Port | — | A 16-bit number that identifies a program (service) on a host. Like an apartment number in a building. |
| Process | — | A running program. Your browser is one process; a web server is another. |
| RST | Reset | A TCP segment that says "something is wrong, abort this connection". |
| select | — | The classic function for I/O multiplexing. |
| Server | — | The program that waits for clients and answers them (a web server, a mail server, the daytime server). |
| shutdown | — | A function that closes one or both directions of a connection, ignoring reference counts. |
| SO_REUSEADDR | — | A socket option that lets a restarted server bind its port while old connections are in TIME_WAIT. |
| sockaddr / sockaddr_in / sockaddr_storage | — | C structures that hold socket addresses: generic, IPv4-specific, and big-enough-for-anything. |
| Socket | — | To the kernel, an endpoint of communication; to a program, a descriptor it can read and write. |
| Socket address | — | An IP address plus a port, for example 128.2.194.242:3479. |
| SYN | Synchronize | The first segment of the TCP handshake; carries the initial sequence number. |
| System call | — | A request from a program to the kernel. |
| telnet / netcat (nc) | — | Programs that open a raw TCP connection so you can type to a server. |
| TIME_WAIT | — | The TCP state the active closer stays in for 2 × MSL after the connection ends. |
| Well-known port | — | A port number below 1024 registered for a standard service (80 web, 25 mail, …). |
| Wrapper function | — | A function with a capitalised name (`Socket`, `Bind` …) that calls the real one and stops the program on error. |
| XDP / eBPF | eXpress Data Path / extended Berkeley Packet Filter | Small, kernel-verified programs that handle packets inside the network driver. |
| ZeroMQ | — | A messaging library that sends whole messages and hides the byte-stream problem. |

## Contents

- [Before you start: the words you need](#before-you-start-the-words-you-need)
- [1. The client-server model and the socket interface](#1-the-client-server-model-and-the-socket-interface)
  - [1.1 What this block is about](#11-what-this-block-is-about)
  - [1.2 The layers below your program](#12-the-layers-below-your-program)
  - [1.3 Connections, socket addresses and the four-tuple](#13-connections-socket-addresses-and-the-four-tuple)
  - [1.4 Clients, servers and ports](#14-clients-servers-and-ports)
  - [1.5 What a socket is](#15-what-a-socket-is)
  - [1.6 The map of the socket calls](#16-the-map-of-the-socket-calls)
  - [1.7 Where sockets came from: BSD history](#17-where-sockets-came-from-bsd-history)
- [2. A first client and server](#2-a-first-client-and-server)
  - [2.1 The daytime client](#21-the-daytime-client)
  - [2.2 TCP is a byte stream](#22-tcp-is-a-byte-stream)
  - [2.3 Error handling with wrapper functions](#23-error-handling-with-wrapper-functions)
  - [2.4 The daytime server](#24-the-daytime-server)
  - [2.5 Iterative servers](#25-iterative-servers)
- [3. TCP connection establishment, termination and ports](#3-tcp-connection-establishment-termination-and-ports)
  - [3.1 The three-way handshake](#31-the-three-way-handshake)
  - [3.2 Closing a connection and the half-close](#32-closing-a-connection-and-the-half-close)
  - [3.3 The TCP state transition diagram](#33-the-tcp-state-transition-diagram)
  - [3.4 TIME_WAIT](#34-time_wait)
  - [3.5 Port number ranges and ephemeral ports](#35-port-number-ranges-and-ephemeral-ports)
  - [3.6 Many connections on one port](#36-many-connections-on-one-port)
- [4. Byte order, addresses and address structures](#4-byte-order-addresses-and-address-structures)
  - [4.1 Byte ordering](#41-byte-ordering)
  - [4.2 IP addresses and names](#42-ip-addresses-and-names)
  - [4.3 Socket address structures](#43-socket-address-structures)
  - [4.4 Converting addresses between text and binary](#44-converting-addresses-between-text-and-binary)
- [5. Robust I/O and the elementary socket calls](#5-robust-io-and-the-elementary-socket-calls)
  - [5.1 Short reads and writes: readn, writen and readline](#51-short-reads-and-writes-readn-writen-and-readline)
  - [5.2 The socket function](#52-the-socket-function)
  - [5.3 The connect function](#53-the-connect-function)
  - [5.4 The bind function](#54-the-bind-function)
  - [5.5 The listen function and its two queues](#55-the-listen-function-and-its-two-queues)
  - [5.6 The accept function](#56-the-accept-function)
  - [5.7 The close function](#57-the-close-function)
- [6. The echo service, I/O models and select](#6-the-echo-service-io-models-and-select)
  - [6.1 The echo client and server](#61-the-echo-client-and-server)
  - [6.2 I/O models](#62-io-models)
  - [6.3 The select function](#63-the-select-function)
  - [6.4 The echo client with select](#64-the-echo-client-with-select)
  - [6.5 Batch input: stop-and-wait is slow](#65-batch-input-stop-and-wait-is-slow)
  - [6.6 The batch-mode bug and shutdown](#66-the-batch-mode-bug-and-shutdown)
  - [6.7 An echo server built on select](#67-an-echo-server-built-on-select)
- [7. Robust and scalable servers](#7-robust-and-scalable-servers)
  - [7.1 Denial of service and robust servers](#71-denial-of-service-and-robust-servers)
  - [7.2 Debugging and testing tools](#72-debugging-and-testing-tools)
  - [7.3 Kinds of servers](#73-kinds-of-servers)
  - [7.4 C10K and C10M](#74-c10k-and-c10m)
  - [7.5 Above the socket: messaging libraries](#75-above-the-socket-messaging-libraries)
- [8. Summary and review](#8-summary-and-review)
  - [8.1 Summary](#81-summary)
  - [8.2 Self-assessment checklists †](#82-self-assessment-checklists-)
  - [8.3 Old exam questions on sockets](#83-old-exam-questions-on-sockets)
  - [8.4 Related quiz topics not in the lectures](#84-related-quiz-topics-not-in-the-lectures)
- [Revision notes and references](#revision-notes-and-references)

## Before you start: the words you need

This guide is about how a **program** uses the network. A few words come back in every section.

- **Client and server:** the **server** is a program that waits, like a shop that is open. The **client** is a program that comes and asks for something, like a customer.
- **IP address:** which **computer** to talk to (for example `208.216.181.15`).
- **Port:** which **program on that computer** to talk to (for example 80 for the web server).
- **Socket:** the "plug" a program uses to talk to the network. For the program it is just a **number** (a descriptor), and the program **reads** from it and **writes** to it, like a file.
- **Kernel:** the boss part of the operating system. Only the kernel touches the network card. Your program asks the kernel for help through **system calls** such as `socket`, `connect` and `read`.

```mermaid
flowchart LR
  P["Your program<br/>(client)"] -->|"system call:<br/>please send this"| K["Kernel<br/>(TCP/IP)"]
  K -->|"packets on the wire"| N((Internet))
  N --> K2["Kernel of the<br/>server computer"]
  K2 -->|"hands the bytes over"| S["Server program"]
```

> **Kid example.** Think of a big apartment building. The **IP address** is the street address of the building, and the **port** is the apartment number. The **server** is a person who lives in an apartment and waits for visitors. The **client** is a visitor who knocks on the door. The **kernel** is the doorkeeper: visitors never walk around the building themselves; the doorkeeper takes the message to the right apartment. The **socket** is the little window in each door where messages are passed in and out.

## 1. The client-server model and the socket interface

**What this chapter is about:** who talks to whom (clients and servers), how a connection is named (addresses and ports), what a socket really is, and the short list of calls that every network program uses.

### 1.1 What this block is about

**In one sentence:** this block teaches you to write programs that talk to each other over the network through the **socket** interface.

This block is about programming **processes** (running programs) that talk to each other through an application programming interface called **sockets**. Some students know sockets already, because "that model has become synonymous with network programming"; others start from zero.

The aims:

- give guidance for beginners and advanced programmers, for people writing **new** network programs and people **maintaining old** ones (the socket API has been stable for about four decades, so code from 1990 still compiles);
- be able to **explain in detail** how the networking parts of a system work and **how to program them**;
- learn through **code examples**, with the usual split into a **client** and a **server**, and write small TCP client-server programs.

The lecturer's advice: "If you take one thing from this session, make it the ability to write" the daytime client and the daytime server ([section 2.1](#21-the-daytime-client) and [section 2.4](#24-the-daytime-server)) "from memory. Everything afterwards is a variation."

The word to notice is **interface**. A socket is not a protocol and not a network. It is the **boundary between your program and the operating system's networking machinery**.

> **Kid example.** A socket is like the counter window at a post office. You don't go into the back room and sort mail yourself; you hand your letter through the window and pick up letters there. The window is the interface; the post office behind it is the kernel and the network.

**Remember:**

- A socket is an **interface** (a boundary), not a protocol and not a network.
- Learn the daytime client and daytime server by heart; everything else is a variation.
- The socket API has been stable for about four decades.

### 1.2 The layers below your program

**In one sentence:** your program only talks to TCP through the socket; TCP, IP and Ethernet below it do the rest, and none of them knows where your "messages" begin and end.

The lecture reviews a client and a server talking over Ethernet with TCP:

- **Your application** is at the top.
- **TCP** gives a reliable, ordered **byte stream** between two endpoints.
- **IP** moves packets between hosts and has **no idea a connection exists**.
- **Ethernet** moves frames between neighbouring machines on the same link.

The socket API sits at the boundary **between your application and TCP**. You never write an IP header or an Ethernet frame yourself.

```mermaid
flowchart TB
  A[Your program] -->|"socket calls: write, read"| T[TCP in the kernel<br/>reliable byte stream]
  T --> I[IP<br/>moves packets host to host]
  I --> E[Ethernet<br/>moves frames on one link]
```

What each layer promises matters. TCP promises that bytes arrive, **in order, eventually**. It promises **nothing about when**, **nothing about message boundaries**, and **nothing about how many `read` calls** it takes to collect them. Following one message down and back up shows two lessons:

- **Lesson 1: the number of `write` calls at one end has no relation to the number of `read` calls at the other.** TCP may split your bytes over several segments or join them with bytes from a later write. This is "the single most common source of bugs in first network programs".
- **Lesson 2: none of the layers in between knows about your message.** The reliability you rely on is built by the two endpoints, not by the network.

```mermaid
flowchart LR
  W1["write 'Hello'"] --> T((TCP stream))
  W2["write 'World'"] --> T
  T --> R1["read gets 'HelloWo'"]
  T --> R2["read gets 'rld'"]
```

The picture shows one possible result: two writes at the sender, and the receiver gets the same ten bytes in the same order, but cut in a different place.

> **Kid example.** You write a letter and give it to the post office. The post office cuts it into postcards, sends them through several sorting centres, and someone at the other end glues them together again. The sorting centres only read the address on each postcard; they have no idea what your letter says or where one of your sentences ends.

**Remember:**

- The socket sits between **your program** and **TCP**.
- TCP promises: all bytes, in order, **eventually**. No promise about **when**, about **message boundaries**, or about **how many reads**.
- Number of `write` calls ≠ number of `read` calls. This is the most common beginner bug.

### 1.3 Connections, socket addresses and the four-tuple

**In one sentence:** a **socket address** is "IP address : port", and a TCP connection is named by the **two** socket addresses together, four values in all.

The lecture names the things that identify a connection:

- A **socket address** is an **IP address plus a port**, written `address:port`.
  - Client socket address: `128.2.194.242:3479`. Port 3479 is an **ephemeral port** chosen by the kernel.
  - Server socket address: `208.216.181.15:80`. Port 80 is a **well-known port** for web servers.
- A **connection** is identified by the pair of both socket addresses, the **connection socket pair**:
  `(128.2.194.242:3479, 208.216.181.15:80)`. That is four values in total, so it is also called the **four-tuple**.

There are **two common styles** of communication:

| Style | Protocol | Socket type | Properties |
|---|---|---|---|
| Datagrams | UDP | `SOCK_DGRAM` | separate packets, no connection |
| Connections | TCP | `SOCK_STREAM` | **point-to-point, full-duplex (both directions at once), reliable**; the main focus of this block |

```mermaid
flowchart LR
  C["Client<br/>128.2.194.242 : 3479<br/>(ephemeral port)"] <-->|"one TCP connection<br/>= the socket pair"| S["Server<br/>208.216.181.15 : 80<br/>(well-known port)"]
```

The four-tuple is why one server can hold thousands of connections on one port: all connections to port 80 share three of the four values and differ in the client's port (or address). Also, the same four-tuple returns in the data-centre material, where ECMP load balancing hashes exactly these values to choose a path.

```mermaid
flowchart LR
  A["Client A<br/>1.1.1.1 : 5000"] --> S["Server<br/>208.216.181.15 : 80"]
  B["Client A, second tab<br/>1.1.1.1 : 5001"] --> S
  C["Client B<br/>2.2.2.2 : 5000"] --> S
```

All three connections end at the same server port 80, but each four-tuple is different, so the kernel can tell them apart.

> **Kid example.** A phone call is named by **both** phone numbers: who is calling and who answers. A big company has one number (port 80), but it can take many calls at once, because each call comes **from** a different phone. "Call from 555-1234 to the company" and "call from 555-9876 to the company" are two different calls.

**Remember:**

- Socket address = **IP address + port**, for example `208.216.181.15:80`.
- A TCP connection = the **pair** of socket addresses = the **four-tuple** (local IP, local port, remote IP, remote port).
- UDP = `SOCK_DGRAM`, separate packets. TCP = `SOCK_STREAM`, point-to-point, full-duplex, reliable.
- Client ports are usually **ephemeral** (picked by the kernel); server ports are **well-known**.

### 1.4 Clients, servers and ports

**In one sentence:** the IP address gets you to the **machine**, the port gets you to the **program**, and the server is the program that waits while the client is the one that starts.

**Clients.** Examples: web browsers, ftp, telnet, ssh. How does a client find a server?

- The **IP address** in the server's socket address identifies the **host** (more exactly, one network adapter of that host; a machine with several adapters has several addresses).
- The **well-known port** identifies the **service**, and so, indirectly, the server program that performs it.

"Address gets you to the machine; port gets you to the program." Well-known ports are only a **convention** (a registered list); nothing in TCP enforces them. A web server on port 9000 works fine; clients just won't find it unless told.

**Using ports to identify services.** One server host, `128.2.194.242`, runs a web server on port 80 and an echo server on port 7. A request for `128.2.194.242:80` goes to the web server and one for `128.2.194.242:7` to the echo server. The **kernel** (the transport layer) reads the port number and passes each segment to the right socket. This is called **demultiplexing**.

```mermaid
flowchart LR
  R1["request to<br/>128.2.194.242:80"] --> K[Kernel of server host<br/>reads the port]
  R2["request to<br/>128.2.194.242:7"] --> K
  K --> W[Web server<br/>port 80]
  K --> EC[Echo server<br/>port 7]
```

> **Kid example.** The IP address is the street address of an apartment building. The port is the apartment number. "Apartment 80" is where the web server lives; "apartment 25" is where the mail server lives. The doorkeeper (the kernel) looks at the apartment number on each letter and puts it in the right mailbox.

A useful debugging fact: if nothing listens on the requested port, the server's TCP answers with a **reset (RST)**, so the client fails **at once** ("connection refused"). If a firewall silently drops the packets, the client **hangs** until a timeout.

```mermaid
flowchart LR
  C[Client sends SYN] --> Q{What is at<br/>the other end?}
  Q -->|"a program listens"| OK["connection made"]
  Q -->|"nobody listens on that port"| RST["RST comes back:<br/>'connection refused' at once"]
  Q -->|"firewall drops it silently"| T["no answer:<br/>client hangs until timeout"]
```

**The client-server exchange** has four steps:

1. The client **sends a request**.
2. The server **handles the request** (it manages some **resource**, such as information or storage).
3. The server **sends a response**.
4. The client **handles the response**.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server (manages a resource)
  C->>S: 1. request
  Note over S: 2. handle the request
  S->>C: 3. response
  Note over C: 4. handle the response
```

Clients and servers are **processes**, running on the same or on different hosts. The only structural difference is **who starts**: the server prepares and waits, the client reaches out.

**Servers** are **long-running processes** ("daemons"), typically started at boot time by the `init` process (process 1), running until the machine is turned off. Each waits on a well-known port:

| Server | Port(s) | Resource | Service |
|---|---|---|---|
| Echo | 7 | — | sends back whatever it receives |
| FTP | 20, 21 | files | stores and retrieves files |
| Telnet | 23 | a terminal | gives a remote terminal on the server machine |
| Mail (SMTP) | 25 | the email "spool" file | stores mail messages in the spool |
| Web (HTTP) | 80 | files and compute cycles (CGI programs) | retrieves files and runs CGI programs for the client |

A machine that runs a server is also often called "a server". The file `/etc/services` lists the services on a Linux machine. Two lessons follow from "long-running": a server **must not leak** (memory, descriptors), and a server **cannot trust its input**, because anyone who can reach the port can connect.

> **Kid example.** A shop and a customer. The shop (server) opens in the morning and waits all day. The customer (client) walks in, asks for bread (request), the baker gets it from the shelf (handles the request using the resource), hands it over (response), and the customer takes it home (handles the response). Because the shop is open all day for anyone, it must not slowly run out of bags (leak), and it must not believe everything strangers say.

**Remember:**

- IP address → which **host**; port → which **service/program**.
- Well-known ports are only a **convention**: echo 7, FTP 20/21, telnet 23, SMTP 25, HTTP 80.
- The kernel delivers each segment to the right socket by port: **demultiplexing**.
- Nobody listening → **RST** → "connection refused" at once. Firewall dropping → **hang** until timeout.
- Four steps: request, handle, response, handle response. The server waits; the client starts.
- Servers are long-running **daemons**: they must not leak and must not trust input.

### 1.5 What a socket is

**In one sentence:** to the kernel a socket is an **endpoint of communication**; to your program it is just a **file descriptor**, a small number you read from and write to like a file.

The socket interface was **created in the early 1980s** as part of the Berkeley distribution of Unix that contained an early version of the Internet protocols. It:

- provides a **user-level interface** to the network;
- is the **underlying basis for all Internet applications** (every browser, web server and database client ends up in these calls);
- is based on the **client/server** programming model.

The lecture gives the central idea:

- **To the kernel**, a socket is an **endpoint of communication**.
- **To an application**, a socket is a **file descriptor** that lets it read from and write to the network.
- In Unix, **all I/O devices are modelled as files, including networks**. So clients and servers communicate by **reading from and writing to socket descriptors**, with the same `read` and `write` calls used for files.
- The main difference between file I/O and socket I/O is **how the program "opens" the descriptor**.

A **file descriptor** is just a small whole number. Every program starts with 0 (standard input, the keyboard), 1 (standard output, the screen) and 2 (standard error). The first socket a program opens usually gets number 3.

```mermaid
flowchart LR
  P["Your program"] --- D0["0 = keyboard<br/>(standard input)"]
  P --- D1["1 = screen<br/>(standard output)"]
  P --- D2["2 = error messages<br/>(standard error)"]
  P --- D3["3 = your first socket"]
```

> **Kid example.** The kernel is the school bus driver. Only the driver may drive the bus (the network card). You get a ticket number (the descriptor, say 3) and say "please deliver this on ticket 3". The driver does the hard work. Your keyboard, your screen and your socket all have ticket numbers, and you use them in the same way.

The lecture draws the picture: on each host, an **application** talks to a **socket**, which belongs to the **OS + APIs**, which talks through a **network interface** to the Internet. "Sockets make network I/O look like files"; the program **calls system functions** to control and communicate; the **network code handles routing and segmentation**. This is both the convenience and the limitation: you don't choose the route, and you don't control how your bytes are cut into segments, so you **cannot rely on message boundaries**.

```mermaid
flowchart LR
  A1[Application] --> S1[Socket] --> O1["OS + APIs"] --> N1[Network interface] --> I((Internet))
  I --> N2[Network interface] --> O2["OS + APIs"] --> S2[Socket] --> A2[Application]
```

The lecture shows the same API from **Java**: a `Socket` class with input and output streams. Python, Go and Rust wrap the same system calls in the same order. The differences between languages are mostly about error handling and cleanup.

Note that the word "socket" is used in **four senses**: the whole API; the descriptor your program holds; the kernel data structure behind it; and in "socket address", an address-and-port pair.

**Remember:**

- Created in the **early 1980s** at Berkeley; the basis of **all** Internet applications; client/server model.
- Kernel view: **endpoint of communication**. Program view: **file descriptor**.
- In Unix, networks are treated as **files**: you use `read` and `write`. Only **opening** is different.
- Descriptors 0, 1, 2 are already taken; the first socket is usually **3**.
- You cannot choose the route or the segment sizes, so you **cannot rely on message boundaries**.
- "Socket" has four meanings: the API, the descriptor, the kernel structure, and (in "socket address") address + port.

### 1.6 The map of the socket calls

**In one sentence:** the client does `socket` → `connect`, the server does `socket` → `bind` → `listen` → `accept`, and then both just `read`, `write` and `close`.

This is "the one slide from part one worth memorising": the overview of the BSD sockets interface.

- **Client:** `socket` → `connect` → then `write` / `read` → `close`.
- **Server:** `socket` → `bind` → `listen` → `accept` → then `read` / `write` → `close`, and back to `accept` to "await a connection request from the next client".
- The **connection request** travels from the client's `connect` to the server's `accept`.
- When the client closes, the server's `read` returns **EOF** (end of file).
- The helper functions `open_clientfd` (client side: socket + connect) and `open_listenfd` (server side: socket + bind + listen) wrap each side's sequence into one call.

```mermaid
flowchart LR
  c1["CLIENT<br/>socket"] --> c2["connect"] --> c3["write / read"] --> c4["close"]
  s1["SERVER<br/>socket"] --> s2["bind"] --> s3["listen"] --> s4["accept"] --> s5["read / write"] --> s6["close"]
  s6 -->|"next client"| s4
  c2 -.->|"connection request"| s4
```

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  Note over S: socket, bind, listen
  Note over S: accept (waits)
  Note over C: socket
  C->>S: connect (connection request)
  Note over C,S: connection established
  C->>S: write (request)
  S->>C: write (response)
  C->>S: close
  Note over S: read returns 0 (EOF), close
  Note over S: back to accept for the next client
```

The asymmetry to remember: **the client makes two calls before it can talk; the server makes four**, three of which only say "I am willing to receive connections, at this address and port, with a queue this deep".

> **Kid example.** A shop. `socket` = build the shop. `bind` = put the street address on the door. `listen` = open the doors. `accept` = a shop assistant walks over to the next customer while the door stays open for others. The customer's `connect` = walking in. The customer only needs two steps (get ready, walk in); the shop needs four before anyone can be served.

**Remember:**

- Client: **socket, connect**, then write/read, close.
- Server: **socket, bind, listen, accept**, then read/write, close, back to accept.
- `connect` (client) meets `accept` (server).
- When the client closes, the server's `read` returns **0 = EOF**.
- `open_clientfd` = socket + connect; `open_listenfd` = socket + bind + listen.

### 1.7 Where sockets came from: BSD history

**In one sentence:** sockets were born in **4.2BSD in 1983**, and because Berkeley's networking code was **free to share**, the same API spread everywhere.

This part is marked "extra review on your own": the history is interesting but **not examinable**.

- The sockets API **originated with 4.2BSD, released in 1983**.
- A few changes came in **1990 with 4.3BSD Reno**, when the OSI protocols went into the BSD kernel.
- The lecture recommends a talk by **Kirk McKusick**, who shared an office at Berkeley with **Bill Joy**, who led the beginnings of BSD.
- The family tree of BSD releases shows that its descendants are still in use (macOS, iOS, embedded systems, router software).
- The releases from the **Computer Systems Research Group (CSRG)** at Berkeley, 4.2BSD to 4.4BSD, required the receiver to already hold an (expensive) **AT&T Unix source licence**. But **all the networking code** (kernel support and applications) was **written at Berkeley, independently** of the AT&T code.
- So from **1989** Berkeley shipped the **BSD networking releases**, containing all the networking code not covered by the AT&T licence. They were **publicly available**, eventually by **anonymous FTP** to anyone.

```mermaid
flowchart LR
  A["1983<br/>4.2BSD:<br/>sockets are born"] --> B["1989<br/>BSD networking<br/>releases, free to share"] --> C["1990<br/>4.3BSD Reno:<br/>a few changes"] --> D["today<br/>macOS, iOS, routers,<br/>Linux, Windows use the same API"]
```

The lesson: the technical quality of the BSD stack explains why people wanted it; the **licence** explains why they could have it. Commercial Unix vendors, Microsoft and router makers took it, which is why this one API became universal. And because the API made **very few decisions for the programmer** (protocol, addressing, concurrency, buffering), it has aged well.

> **Kid example.** A family recipe that is very good, but locked in a book you must pay a lot for. One cook writes the best part of the recipe again on their own, without copying the locked book, and gives it away for free. Soon every kitchen in town uses his version.

**Remember:**

- Sockets: **4.2BSD, 1983**. Small changes in **4.3BSD Reno, 1990**.
- From **1989**, the networking code was free to share (anonymous FTP), so everyone adopted it.
- This history is **not examinable**.

## 2. A first client and server

**What this chapter is about:** the two smallest complete programs, the daytime client and the daytime server, explained line by line, plus the two big lessons that come with them: TCP is a stream of bytes, and every call must be checked for errors.

### 2.1 The daytime client

**In one sentence:** the daytime client makes a socket, fills in the server's address, connects, and reads in a loop until the server closes the connection.

The **daytime** service (TCP port 13) is very simple: when you connect, the server sends the current date and time as a line of text and closes the connection. The lecture shows the client.

```c
int main(int argc, char **argv) {
    int     sockfd, n;
    char    recvline[MAXLINE + 1];
    struct sockaddr_in servaddr;

    if (argc != 2)  err_quit("usage: a.out <IPaddress>");
    if ( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)  err_sys("socket error");

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port   = htons(13);   /* daytime server */
    if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0)
        err_quit("inet_pton error for %s", argv[1]);

    if (connect(sockfd, (SA *) &servaddr, sizeof(servaddr)) < 0)  err_sys("connect error");

    while ( (n = read(sockfd, recvline, MAXLINE)) > 0) {
        recvline[n] = 0;        /* null terminate */
        if (fputs(recvline, stdout) == EOF)  err_sys("fputs error");
    }
    if (n < 0)  err_sys("read error");
    exit(0);
}
```

Line by line, in plain words:

1. **Declarations.** Reserve a place for the socket descriptor (`sockfd`), a byte count (`n`), a receive buffer (`recvline`) and a **socket address structure** for the server (`servaddr`).
2. **Argument check.** The program must be given the server's IP address when it is started; otherwise it prints how to use it and stops.
3. **`socket(AF_INET, SOCK_STREAM, 0)`** creates a **TCP socket over IPv4**: `AF_INET` = IPv4 address family, `SOCK_STREAM` = reliable byte stream, `0` = "pick the obvious protocol", which is TCP. It returns a **small integer descriptor**. If it returns a negative number, there was an error.
4. **`bzero`** fills the address structure with zeros (so no leftover garbage remains), then the program fills in the **family** (IPv4), the **port** 13 (converted with `htons` to network byte order, [section 4.1](#41-byte-ordering)) and the **IP address** (converted from text like "192.0.2.5" to binary by `inet_pton`, [section 4.4](#44-converting-addresses-between-text-and-binary)).
5. **`connect`** starts the TCP three-way handshake with the server. When it returns successfully, the connection is established.
6. **The read loop.** Call `read` again and again while it returns a positive number of bytes. Each time, add a zero byte at the end (C's way of marking the end of text) and print the text. When `read` returns **0**, the server has closed the connection, and the loop ends. A negative value means an error.

```mermaid
flowchart TB
  A["socket(AF_INET, SOCK_STREAM, 0)<br/>get a TCP socket, say number 3"] --> B["fill servaddr:<br/>IPv4, port 13, server IP"]
  B --> C["connect<br/>three-way handshake"]
  C --> D["read"]
  D -->|"more than 0 bytes"| P["print them"] -->|"read again"| D
  D -->|"0: server closed"| E["exit"]
  D -->|"negative: error"| F["print error, stop"]
```

> **Kid example.** `socket` is buying a phone. Filling in `servaddr` is writing the friend's number on a note (country code, number, extension). `connect` is dialling. The read loop is listening and writing down what the friend says, again and again, until the friend hangs up.

The lecture adds:

- The call to `socket` **can fail**. Why? Answer: the process may have reached its **limit of open descriptors**, the kernel may be **out of memory**, or the address family or type may be **unsupported**.
- We will meet the word "socket" in several meanings: the **sockets API**, the **function named `socket`**, and **the TCP socket**, which is a TCP endpoint.

At the moment `socket` returns, the socket exists but is **connected to nothing**: no address, no peer. Everything useful happens in the calls after it.

**Remember:**

- Daytime = **TCP port 13**; the server sends the time and closes.
- Client steps: **socket → fill address (family, `htons(13)`, `inet_pton`) → connect → read loop**.
- `read` > 0: data; `read` = 0: the other side closed (EOF); `read` < 0: error.
- `socket` can fail: too many open descriptors, out of memory, unsupported family or type.
- Right after `socket`, the socket is connected to **nothing**.

### 2.2 TCP is a byte stream

**In one sentence:** TCP delivers a stream of bytes with **no message boundaries**, so you must always `read` in a loop.

The lecture explains why the client reads **in a loop**.

TCP is a **byte-stream protocol with no record boundaries**. The server's reply is 26 bytes:

`Mon May 26 20:58:40 2003\r\n` (`\r` is carriage return and `\n` is linefeed, together "end of line").

These 26 bytes can arrive in many ways:

- normally as **one TCP segment** with all 26 bytes,
- but also as **26 segments of 1 byte each**, or anything in between.

So you must **always code the read in a loop**, and stop when `read` returns **0** (the other end closed) or a **negative value** (an error). Ending the data by closing the connection is also how **HTTP 1.0** worked.

```mermaid
flowchart LR
  W["server: one write<br/>of 26 bytes"] --> TCP((TCP))
  TCP --> R1["read 1: 10 bytes"]
  TCP --> R2["read 2: 16 bytes"]
  TCP --> R3["read 3: returns 0<br/>(server closed)"]
```

> **Kid example.** Pouring a litre of juice into a pipe. At the other end you fill cups. You may fill three small cups or one big one. The pipe doesn't know that your juice was "one bottle". If the other person needs to know where one bottle ends, you must tell them, for example by saying the size first.

**TCP itself gives no record markers.** If an application needs to mark the ends of records, it must do so itself. There are three common ways:

| Method | Example in the lecture | Weakness |
|---|---|---|
| A **delimiter** at the end of each record | **SMTP** ends each record with carriage return + linefeed (`\r\n`) | the delimiter must be "escaped" if it can appear inside the data |
| A **length prefix** before each record | **Sun RPC** and **DNS** put a binary counter with the record length in front of each record | you must know the length before you start sending |
| **Closing the connection** | the daytime server, HTTP 1.0 | only one message per connection |

```mermaid
flowchart LR
  a0["DELIMITER<br/>(SMTP)"] --> a1["HELLO\r\n"] --> a2["HOW ARE YOU\r\n"]
  b0["LENGTH PREFIX<br/>(DNS, Sun RPC)"] --> b1["5 | HELLO"] --> b2["11 | HOW ARE YOU"]
  c0["CLOSE THE CONNECTION<br/>(daytime, HTTP 1.0)"] --> c1["Mon May 26 ...\r\n"] --> c2["FIN: that's all"]
```

**Remember:**

- TCP is a **byte stream**: no record boundaries. 26 bytes may come as 1 segment, 26 segments, or anything in between.
- **Always read in a loop**; stop on 0 (closed) or negative (error).
- Three ways to mark records: **delimiter** (SMTP, `\r\n`), **length prefix** (Sun RPC, DNS), **close the connection** (daytime, HTTP 1.0).

### 2.3 Error handling with wrapper functions

**In one sentence:** every call can fail, so Stevens wraps each call in a capitalised **wrapper** (`Socket`, `Bind` …) that checks the result and stops the program on error.

**It is essential to check every function call for an error return.** To keep the code readable, Stevens uses **wrapper functions**: the same name with a **capital letter**, which calls the real function, checks the result, and stops the program with a message if there was an error.

```c
int Socket(int family, int type, int protocol) {
    int n;
    if ( (n = socket(family, type, protocol)) < 0)
        err_sys("socket error");
    return (n);
}
```

In words: call the real `socket`; if the result is negative, print "socket error" and stop; otherwise hand back the descriptor. You will see `Socket`, `Bind`, `Listen`, `Accept`, `Write`, `Close` … throughout the lecture.

```mermaid
flowchart LR
  A["Socket(...)<br/>capital S"] --> B["calls the real<br/>socket(...)"]
  B --> C{result < 0 ?}
  C -->|yes| D["print 'socket error'<br/>and stop the program"]
  C -->|no| E["return the descriptor"]
```

A caution: "check and abort" is right for teaching code and small tools, but **wrong for a server**, which must **not stop just because one client did something strange** ([section 7.1](#71-denial-of-service-and-robust-servers)).

> **Kid example.** A helper who opens every parcel for you and checks it. If something inside is broken, the helper shouts "broken!" and stops everything. That is fine when you are practising at home. But a shop cannot close for the whole day just because one customer brought a broken toy back.

**Remember:**

- **Check every call** for an error return.
- Wrapper = same name with a **capital letter**; it calls the real function and stops on error.
- Good for teaching and small tools; **wrong for real servers**, which must keep running.

### 2.4 The daytime server

**In one sentence:** the server makes a socket, **binds** it to port 13 on all its addresses, **listens**, and then forever **accepts** one client, writes the time to it, and closes that connection.

The lecture shows the server.

```c
int main(int argc, char **argv) {
    int     listenfd, connfd;
    struct sockaddr_in servaddr;
    char    buff[MAXLINE];
    time_t  ticks;

    listenfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family      = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port        = htons(13);   /* daytime server */

    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
    Listen(listenfd, LISTENQ);

    for ( ; ; ) {
        connfd = Accept(listenfd, (SA *) NULL, NULL);
        ticks = time(NULL);
        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks));
        Write(connfd, buff, strlen(buff));
        Close(connfd);
    }
}
```

In plain words:

1. **Two descriptors**, where the client had one: `listenfd` (the **listening** socket) and `connfd` (the **connected** socket for one client).
2. Create a TCP socket, zero the address structure, and fill it with family IPv4, address **`INADDR_ANY`** (the wildcard: "any of my addresses"), and port 13. The **server names its own address**, because clients must know in advance where to find it; the client never did this.
3. **`Bind`** attaches the socket to that address and port.
4. **`Listen`** turns it into a listening socket and sets the size of the waiting queue (`LISTENQ`).
5. **The endless loop** (`for ( ; ; )` means "repeat forever"):
   - **`Accept`** sleeps until a client has connected, then returns a **new descriptor** `connfd` for that client.
   - Get the current time (`time`), turn it into text (`ctime`), and put it with `\r\n` into `buff`.
   - **`Write`** the text to **`connfd`** (not to the listening socket).
   - **`Close`** `connfd`, then go back to `Accept`.

"Nine lines. That is a complete TCP server." Two things to fix in your mind (notes): the **listening descriptor is never written to**; it only produces connected descriptors. And the server closes `connfd` but **never closes `listenfd`**: the connection ends, the service continues.

```mermaid
flowchart TB
  L["listenfd<br/>(listening socket, port 13)"] -->|"accept gives client 1"| C1["connfd for client 1<br/>write time, close"]
  L -->|"accept gives client 2"| C2["connfd for client 2<br/>write time, close"]
  L -->|"accept gives client 3"| C3["connfd for client 3<br/>write time, close"]
```

> **Kid example.** A hotel receptionist (listening descriptor) stands at the front desk all day. For each guest, the receptionist hands over a room key (connected descriptor). The guest uses the room, not the front desk. When the guest leaves, the key is returned, but the front desk stays open for the next guest.

**What `accept` really does.** The server process is **put to sleep** in `accept`, waiting for connections. TCP sets up connections with the **three-way handshake**; when the handshake completes, `accept` **returns**, and its return value is a new descriptor, the **connected descriptor**, used to talk to the new client. The order matters: **the kernel completes the handshake first**, even while the program is busy elsewhere; `accept` just **hands over** a connection that the kernel has already established.

**Technical clarification.** The lecture's server code writes `bzeros(&servaddr, …)`. The real function is called **`bzero`** (or the standard `memset(&servaddr, 0, sizeof(servaddr))`); `bzeros` does not exist and would not compile.

**Remember:**

- Server steps: **socket → fill address (`INADDR_ANY`, port 13) → bind → listen → loop { accept, write, close }**.
- Two descriptors: **listening** (`listenfd`, never written to, never closed) and **connected** (`connfd`, one per client).
- `INADDR_ANY` = "any of my addresses".
- The **kernel** finishes the handshake; `accept` only hands over the ready connection.
- The function is `bzero`, not `bzeros`.

### 2.5 Iterative servers

**In one sentence:** an **iterative** server serves one client completely before the next, so other clients must wait even if the kernel has already connected them.

**The daytime server handles only one client at a time.** If several connections arrive at about the same time, the **kernel queues them**, up to some limit, and `accept` returns them **one at a time**. The daytime server is fast, so nobody notices. But if serving each client took longer, we would need to **overlap** the service of one client with another. A server that serves one client after another is called an **iterative server**.

The lecture draws the timeline:

```mermaid
sequenceDiagram
  participant C1 as Client 1
  participant S as Server
  participant C2 as Client 2
  C1->>S: connect
  Note over S: accept returns for client 1
  C1->>S: request
  S->>C1: reply, then close
  C2->>S: connect (kernel completes the handshake)
  Note over C2: client 2 waits, no data flows yet
  Note over S: only now accept returns for client 2
  C2->>S: request
  S->>C2: reply, then close
```

Client 2's connection may already be established by the kernel, but no data flows until the server comes back to `accept`. That waiting gap grows with the time each client takes and with the number of clients waiting. **Every concurrency technique later in the block is an attempt to remove this gap.**

> **Kid example.** A shop with one cashier who serves each customer from "hello" to "goodbye" before even looking at the queue. Fine if everyone buys one sweet; terrible if someone wants to chat for ten minutes. The other customers are already inside the shop (connected), but nobody serves them.

**Remember:**

- Iterative server = **one client at a time**.
- Waiting connections are **queued by the kernel** and `accept` returns them one by one.
- A waiting client may be connected, but **no data flows** until the server calls `accept` again.
- All the concurrency ideas later (fork, threads, `select`) try to remove this waiting gap.

## 3. TCP connection establishment, termination and ports

**What this chapter is about:** what happens inside TCP when you call `connect`, `accept` and `close`: the three-way handshake to open, the four segments to close, the states a connection passes through, why the closer waits in TIME_WAIT, and how port numbers are shared out.

### 3.1 The three-way handshake

**In one sentence:** opening a TCP connection takes three segments, SYN, SYN+ACK and ACK, so that **each side knows its own starting number arrived**.

To understand `connect`, `accept` and `close`, you must understand how TCP connections are **established and terminated**, and TCP's **state transition diagram**.

- The **server** prepares to accept a connection by calling `socket`, `bind` and `listen`. This is a **passive open**.
- The **client** calls `connect`. This is an **active open**. The client's TCP sends a **SYN** ("synchronize") segment, which tells the server the client's **initial sequence number** for the data it will send. The SYN carries **no data**: only an IP header, a TCP header and possibly TCP options.
- The **server** must **acknowledge (ACK)** the client's SYN and must also send **its own SYN** with its initial sequence number. It sends **both in one segment** (SYN + ACK).
- The **client** must **acknowledge** the server's SYN.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  Note over S: socket, bind, listen (passive open)<br/>accept (blocks)
  Note over C: socket<br/>connect (blocks), active open
  C->>S: SYN J
  S->>C: SYN K, ACK J+1
  Note over C: connect returns
  C->>S: ACK K+1
  Note over S: accept returns<br/>read (blocks)
```

Here J and K are the two initial sequence numbers. Why **three** segments?

- Not four: the server's ACK and its own SYN can travel **together**, saving a round trip.
- Not two: **each side must know that its own sequence number arrived**. The client learns this from the server's ACK; the server learns it from the client's final ACK.

> **Kid example.** Phoning a friend. You: "Hello, can you hear me?" (SYN). Friend: "Yes, I hear you. Can you hear me?" (SYN + ACK, two things in one sentence). You: "Yes!" (ACK). Now both of you know that both directions work, and you can talk.

The cost (notes): **no data can flow until one round-trip time has passed**. On a path with a 100 ms round trip, that is a tenth of a second spent only on agreeing. This is why HTTP keeps connections open (keep-alive) and why QUIC combines its handshake with encryption set-up.

**Remember:**

- Server: `socket`, `bind`, `listen` = **passive open**. Client: `connect` = **active open**.
- Three segments: **SYN J**, **SYN K + ACK J+1**, **ACK K+1**. The SYN carries **no data**.
- Three, not four: ACK and SYN travel together. Three, not two: each side must hear that **its own** number arrived.
- `connect` returns after the second segment; `accept` returns after the third.
- Cost: one round-trip time before any data.

### 3.2 Closing a connection and the half-close

**In one sentence:** closing takes four segments (FIN, ACK, FIN, ACK) because each **direction** is closed separately, and `read` returning 0 simply means "the other side has finished sending".

The lecture shows that closing takes **four segments**:

1. One process calls `close` first: the **active close**. Its TCP sends a **FIN**: "I have finished sending data".
2. The other end performs the **passive close**: its TCP **acknowledges the FIN**, and the arrival of the FIN is passed to the application as an **end-of-file** (`read` returns 0), because no more data will come on this connection.
3. **Later**, the process that received the end-of-file **closes its socket**, so its TCP sends its own **FIN**.
4. The TCP at the end that did the active close **acknowledges** that FIN.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  Note over C: close (active close)
  C->>S: FIN M
  Note over S: passive close<br/>read returns 0
  S->>C: ACK M+1
  Note over C,S: half-close, server may still send data
  Note over S: close
  S->>C: FIN N
  C->>S: ACK N+1
```

**Half-close.** Between steps 2 and 3, data **can still flow** from the passive-closing end to the active-closing end. A TCP connection is really **two independent byte streams**, one per direction, and each can end without the other. This is a real feature: a client can say "I have finished sending my request, now send me the answer". The `shutdown` call does exactly this ([section 6.6](#66-the-batch-mode-bug-and-shutdown)).

```mermaid
flowchart LR
  C[Client] -->|"pipe 1: closed by the first FIN"| S[Server]
  S -->|"pipe 2: still open until the second FIN"| C
```

So **a `read` that returns 0 is not an error and not an empty message.** It means "the peer has finished sending".

> **Kid example.** Ending a phone call politely. You: "I have nothing more to say." Friend: "OK." (Your friend can still tell you one more thing, and you still listen.) Friend: "Bye then." You: "OK, bye."

**Remember:**

- Four segments: **FIN, ACK, FIN, ACK**.
- **Active close** = the side that closes first. **Passive close** = the side that receives the first FIN; its `read` returns **0 (EOF)**.
- A connection is **two independent streams**; one can close while the other still carries data (**half-close**, done with `shutdown`).
- `read` = 0 is **not** an error; it means "no more data will come".

### 3.3 The TCP state transition diagram

**In one sentence:** a TCP connection is always in one named **state** (LISTEN, ESTABLISHED, TIME_WAIT …), and every socket call either causes a state change or reports one.

A TCP connection is always in one **state**. The API calls either **drive** a state change or **report** one:

- An application that does an **active open** in the **CLOSED** state makes TCP send a **SYN**, and the new state is **SYN_SENT**. If TCP then receives a **SYN with an ACK**, it sends an **ACK**, and the new state is **ESTABLISHED**. (So `connect` blocks in SYN_SENT and returns in ESTABLISHED.)
- The two arrows **out of ESTABLISHED** are about termination:
  - If the application calls **`close` before receiving a FIN** (active close), the state becomes **FIN_WAIT_1**.
  - If the application **receives a FIN** while ESTABLISHED (passive close), the state becomes **CLOSE_WAIT**.
- **Simultaneous open**: both ends send SYNs at about the same time and the SYNs cross in the network.
- **Simultaneous close**: both ends send FINs at the same time.

```mermaid
stateDiagram-v2
  [*] --> CLOSED
  CLOSED --> LISTEN: listen (passive open)
  CLOSED --> SYN_SENT: connect, send SYN
  LISTEN --> SYN_RCVD: recv SYN, send SYN+ACK
  SYN_SENT --> SYN_RCVD: recv SYN, send SYN+ACK (simultaneous open)
  SYN_SENT --> ESTABLISHED: recv SYN+ACK, send ACK
  SYN_RCVD --> ESTABLISHED: recv ACK
  ESTABLISHED --> FIN_WAIT_1: close, send FIN
  ESTABLISHED --> CLOSE_WAIT: recv FIN, send ACK
  CLOSE_WAIT --> LAST_ACK: close, send FIN
  LAST_ACK --> CLOSED: recv ACK
  FIN_WAIT_1 --> FIN_WAIT_2: recv ACK
  FIN_WAIT_1 --> CLOSING: recv FIN, send ACK (simultaneous close)
  FIN_WAIT_2 --> TIME_WAIT: recv FIN, send ACK
  CLOSING --> TIME_WAIT: recv ACK
  TIME_WAIT --> CLOSED: wait 2 MSL
```

The left path (FIN_WAIT_1, FIN_WAIT_2, TIME_WAIT) is the **active closer**; the right path (CLOSE_WAIT, LAST_ACK) is the **passive closer**. Note that the difference matters: **the end that closes first must sit in TIME_WAIT afterwards**. Which end that is depends on your application protocol. A web server that closes after each response is the active closer on every connection, which historically caused scaling problems.

The big diagram is easier when you follow just one side at a time:

```mermaid
flowchart TB
  a1["ACTIVE CLOSER<br/>(calls close first)<br/>ESTABLISHED"] -->|"close, send FIN"| a2[FIN_WAIT_1] -->|"recv ACK"| a3[FIN_WAIT_2] -->|"recv FIN, send ACK"| a4[TIME_WAIT] -->|"wait 2 MSL"| a5[CLOSED]
  p1["PASSIVE CLOSER<br/>(receives the FIN)<br/>ESTABLISHED"] -->|"recv FIN, send ACK"| p2[CLOSE_WAIT] -->|"close, send FIN"| p3[LAST_ACK] -->|"recv ACK"| p4[CLOSED]
```

> **Kid example.** The states are like the steps of leaving a party. ESTABLISHED: you are chatting. FIN_WAIT: you said "I'm going" and wait for an answer. TIME_WAIT: you are already at the door, but you wait a moment in case your friend shouts one more "bye!". CLOSE_WAIT is your friend's state: they heard you are leaving and finish what they were saying before they say goodbye too.

**Additional context.** The full diagram on the original slide (Stevens, figure 2.4) also has a direct arrow FIN_WAIT_1 → TIME_WAIT when a FIN and the ACK of our FIN arrive together, and the timeout paths back to CLOSED. The drawing above keeps the main arrows.

**Watching the packets.** **`netstat`** displays network connections, routing tables and interface statistics. "Try it!" While your client and server talk you will see `ESTABLISHED`; afterwards `TIME_WAIT`; on the server's waiting socket `LISTEN`. On Linux, **`ss`** is the modern replacement.

**Remember:**

- `connect`: CLOSED → **SYN_SENT** → **ESTABLISHED**. Server: CLOSED → **LISTEN**.
- Active closer: ESTABLISHED → **FIN_WAIT_1** → **FIN_WAIT_2** → **TIME_WAIT** → CLOSED.
- Passive closer: ESTABLISHED → **CLOSE_WAIT** → **LAST_ACK** → CLOSED.
- **Simultaneous open** (SYNs cross) and **simultaneous close** (FINs cross) also exist.
- **The side that closes first ends in TIME_WAIT.**
- See the states with `netstat` (or `ss` on Linux).

### 3.4 TIME_WAIT

**In one sentence:** the side that closes first waits **2 × MSL** in TIME_WAIT, so it can resend a lost final ACK and so that old delayed packets die before the same connection is used again.

- The end that does the **active close** goes through the **TIME_WAIT** state.
- It stays there for **2 × MSL** (twice the maximum segment lifetime).
- **MSL** is the longest time any IP datagram can live in the network. It is **bounded** because every datagram carries an **8-bit hop limit** (TTL), so it cannot circulate forever.
- The recommended MSL (RFC 1122) is **2 minutes**; BSD uses **30 seconds**. So TIME_WAIT lasts **between 1 and 4 minutes**.
- Lost packets are usually the result of **routing problems** (for example a routing loop). Once the problem is fixed, a packet that was stuck in the loop may still be delivered, but this happens **within MSL**.

```math
\text{TIME\_WAIT} = 2 \times \text{MSL}: \quad 2 \times 30\text{ s} = 1\text{ min (BSD)}, \qquad 2 \times 2\text{ min} = 4\text{ min (RFC 1122)}
```

The lecture gives the **two reasons** TIME_WAIT exists:

1. **To implement TCP's full-duplex connection termination reliably.**
2. **To allow old duplicate segments to expire in the network.**

**Reason 1: reliable termination.** Suppose the **final ACK is lost**. The other side (the passive closer) will **resend its final FIN**. If the active closer had already gone to CLOSED, it would have no record of the connection, so it would answer with an **RST** ("unexpected FIN"), and the other side would report an **error even though all data was received correctly**. So the active closer must **keep state information** to be able to **resend the final ACK**. That is also why it is **the active closer that stays in TIME_WAIT**: it is the end that might have to retransmit the final ACK. "If TCP terminates both directions of data flow cleanly for a connection, then it must correctly handle the loss of any of these four segments."

```mermaid
sequenceDiagram
  participant A as Active closer
  participant P as Passive closer
  A->>P: FIN
  P->>A: ACK
  P->>A: FIN
  A--xP: final ACK (lost)
  Note over A: in TIME_WAIT, still remembers the connection
  P->>A: FIN (resent after timeout)
  A->>P: ACK again, all good
```

> **Kid example.** After saying "bye" on a walkie-talkie, you keep it switched on for one more minute. If your friend didn't hear your last "OK", they will say "bye?" again, and you can answer. If you had already switched off, your friend would think something went wrong.

**Reason 2: expiring old duplicates.** Take a connection `(12.106.32.254, 1500, 206.168.112.219, 21)`. It closes. Later a new connection is set up between **the same IP addresses and the same ports**: a new **incarnation** of the old connection. Nothing in a packet header says which incarnation it belongs to. An old, delayed segment from the first incarnation could arrive and, **if its sequence number happens to fit**, be **mistaken for data of the new connection**. TCP prevents this: **a new incarnation of a connection that is in TIME_WAIT cannot be started**. Because TIME_WAIT lasts **2 × MSL**, there is **one MSL for a packet in one direction to die** and **another MSL for its reply** to die. So when a new connection is established, all old duplicates from previous incarnations are **guaranteed to have expired**.

```mermaid
flowchart LR
  A["old connection closes<br/>(same 4 values)"] --> B["TIME_WAIT: 2 × MSL<br/>MSL for a stray packet to die,<br/>MSL for its reply to die"] --> C["new incarnation<br/>(same 4 values) may start:<br/>no old packets left"]
```

> **Kid example.** You move out of an apartment, and a new family moves in. If they move in the next day, your old letters still arrive and they might read them as theirs. So the landlord waits until all your old post has surely stopped coming before the new family gets the same address.

† The failure case, stated directly: if an endpoint were **not** in TIME_WAIT when a delayed segment arrives, "that segment is mistaken for part of the second connection (when it has appropriate sequence numbers)". The protection is "temporal rather than structural": TCP does not detect the stale segment; it **waits until the segment cannot exist any more**.

† The **design consequences**. TIME_WAIT is a **resource**: each connection in it holds a kernel table entry and reserves a four-tuple for about a minute on the machine that closed first. A busy server closing many short connections can hold tens of thousands of them. This shaped protocols (**HTTP keep-alive**, **connection pooling**, **persistent connections**) and the socket API: the option **`SO_REUSEADDR`** lets a server bind a port that still has connections in TIME_WAIT (you will need it in the labs, otherwise a restarted server fails with "address already in use"). The general lesson: a mechanism designed for correctness when connections were few and long can become a bottleneck when usage changes.

**Technical clarification.** The lecture says the TIME_WAIT material is "hidden in the current deck". In fact only the parts marked † above are hidden; the rest of this section is visible.

**Remember:**

- The **active closer** waits in TIME_WAIT for **2 × MSL**.
- MSL: RFC 1122 says **2 minutes**, BSD uses **30 seconds**, so TIME_WAIT is **1 to 4 minutes**. MSL is bounded by the **8-bit hop limit (TTL)**.
- Reason 1: **resend the final ACK** if it was lost (otherwise the peer gets an RST and reports an error).
- Reason 2: let **old duplicate segments die** before a new **incarnation** with the same four values starts.
- † TIME_WAIT costs resources; `SO_REUSEADDR` lets a restarted server bind its port ("address already in use" otherwise).

### 3.5 Port number ranges and ephemeral ports

**In one sentence:** port numbers 0 to 65535 are split into **well-known** (0–1023), **registered** (1024–49151) and **dynamic/private** (49152–65535), and clients normally get a short-lived **ephemeral** port from the kernel.

The lecture divides the 16-bit port numbers (0 to 65535) into three ranges. The figure in the lecture (from Stevens) gives the numbers:

| Range | Ports | Who controls it |
|---|---|---|
| **Well-known** | 0 – 1023 | assigned by **IANA** (Internet Assigned Numbers Authority) |
| **Registered** | 1024 – 49151 | IANA only **lists** their uses, it does not control them |
| **Dynamic or private** | 49152 – 65535 | **not controlled**: the **ephemeral ports** |

```mermaid
block-beta
  columns 3
  A["0 – 1023<br/>well-known<br/>(IANA assigns)"] B["1024 – 49151<br/>registered<br/>(IANA only lists)"] C["49152 – 65535<br/>dynamic / private<br/>(ephemeral)"]
```

The figure also shows older conventions: BSD **reserved** ports 1–1023 (only privileged programs may bind them), traditional BSD **ephemeral** ports 1024–5000, BSD servers (non-privileged) 5001–65535, the `rresvport` range 513–1023, and Solaris ephemeral ports 32768–65535. None of this is enforced by TCP; it is a registry plus agreement.

**Ephemeral ports.** Ephemeral means **short-lived**. They are used for the **client end** of a client-server connection to a well-known port on a server. They may also be used to free up a server's well-known listening port by moving the service connection elsewhere. The allocation is **temporary**, valid only for the session; afterwards the port can be reused. The client does not choose its port: it calls `connect` without `bind`, and the kernel picks an unused one. Also, there are about **16 000** ports in the usual range, so one client machine cannot hold more than about that many connections **to the same server address and port**, because each four-tuple must be unique.

> **Kid example.** At a big shop, the famous departments have fixed numbers everybody knows: "desk 80 is the information desk" (well-known). When you arrive as a customer, you pull a queue ticket like "number 51234" (ephemeral). Your ticket is only yours while you are being served; afterwards somebody else can get the same number.

**Technical clarification.** The lecture's figure starts the well-known range at 1; officially it is **0 to 1023**.

**Remember:**

- Ports are **16-bit**: 0 to 65535.
- **Well-known 0–1023** (IANA assigns), **registered 1024–49151** (IANA lists), **dynamic/private 49152–65535** (ephemeral).
- Older conventions: BSD reserved 1–1023, BSD ephemeral 1024–5000, Solaris ephemeral 32768–65535. Not enforced by TCP.
- **Ephemeral = short-lived**, picked by the kernel for the client when it calls `connect` without `bind`.
- About **16 000** ephemeral ports → at most about that many connections from one client host to the **same** server address and port.

### 3.6 Many connections on one port

**In one sentence:** many clients can use the same server port at once, because TCP picks the right socket by looking at **all four** values of the connection, not just the port.

The lecture shows how a **concurrent server** can serve many clients on **one** port. The server host is **multihomed**: it has two IP addresses, `12.106.32.254` and `192.168.42.1`. It does a passive open on the well-known port **21** (FTP).

1. **Listening.** The server's socket pair is written `{*:21, *:*}`. The first `*` means "any local address of this host" (the wildcard); `*:*` means "any client address and port". The listening socket is **not a connection**; it is a **pattern** that arriving connections are matched against.
2. **First client.** A client at `206.168.112.219` connects from ephemeral port **1500**. The server accepts it and **forks** (makes a copy of itself); the **child** serves this client. The **connected socket** uses the **same local port 21** as the listening socket. On the multihomed server, the **local address is filled in** (`12.106.32.254`) once the connection is established. So the child's socket pair is `{12.106.32.254:21, 206.168.112.219:1500}`. The server host must distinguish the listening socket from the connected socket.
3. **Second client.** Another client process on the same client host connects. Its TCP picks another unused ephemeral port, **1501**. On the server, a second child gets `{12.106.32.254:21, 206.168.112.219:1501}`. The two connections are **distinct**.

```mermaid
flowchart LR
  L["listening socket<br/>∗:21, ∗:∗"] -.->|fork| S1["child 1<br/>12.106.32.254:21,<br/>206.168.112.219:1500"]
  L -.->|fork| S2["child 2<br/>12.106.32.254:21,<br/>206.168.112.219:1501"]
  S1 <-->|"connection 1"| C1["client 1<br/>206.168.112.219 : 1500"]
  S2 <-->|"connection 2"| C2["client 2<br/>206.168.112.219 : 1501"]
```

**The rule to remember:** TCP **cannot demultiplex** incoming segments by looking only at the destination port. It must look at **all four values of the socket pair** to decide which endpoint gets an arriving segment. A segment matches the **most specific** socket that fits, so an established connection always wins over the wildcard listener. (UDP, by contrast, delivers by destination port alone, because a UDP socket has no peer.)

```mermaid
flowchart TB
  A["segment arrives:<br/>from 206.168.112.219:1501<br/>to 12.106.32.254:21"] --> Q{"Is there a socket with<br/>exactly these 4 values?"}
  Q -->|yes| B["deliver to child 2<br/>(most specific match)"]
  Q -->|no, and it is a SYN| C["deliver to the listening<br/>socket ∗:21, ∗:∗"]
```

> **Kid example.** One school phone number (port 21) with many lines. The receptionist (listening socket) answers every new call and passes it to a free teacher (child). The phone system tells the calls apart by who is calling from which phone, not by the school number, which is the same for all.

**Remember:**

- Listening socket `{*:21, *:*}` is a **pattern**, not a connection.
- Each accepted client gets its **own connected socket** on the **same port 21**; on a multihomed host the local address is filled in.
- Two clients on the same host differ by their **ephemeral port** (1500, 1501).
- **TCP demultiplexes on all four values**; the most specific match wins. **UDP** uses only the destination port.

## 4. Byte order, addresses and address structures

**What this chapter is about:** how numbers and addresses are stored in memory and on the wire: the byte-order problem, how names become IP addresses, the C structures that hold a socket address, and how to convert between text like "192.168.1.5" and binary.

### 4.1 Byte ordering

**In one sentence:** computers may store the bytes of a number in different orders, so before sending you convert ports and addresses to the network's **big-endian** order with `htons`/`htonl`, and back with `ntohs`/`ntohl`.

This part is marked "review on your own".

**The problem.** A 16-bit number is made of **2 bytes**, and there are **two ways to store them** in memory:

| Order | Byte at the lower address A | Byte at address A + 1 | Used by |
|---|---|---|---|
| **Little-endian** | low-order byte | high-order byte | Intel/AMD PCs, most ARM systems |
| **Big-endian** | high-order byte | low-order byte | the **network** |

Example: port 80 is `0x0050` in hexadecimal. Big-endian memory holds `00 50`; little-endian memory holds `50 00`. If a little-endian machine sent its memory as it is, a big-endian reader would see `0x5000` = 20480. Wrong port!

```mermaid
block-beta
  columns 4
  h1["Big-endian"] b1["address A: 00"] b2["address A+1: 50"] r1["= 0x0050 = 80 ✔"]
  h2["Little-endian"] l1["address A: 50"] l2["address A+1: 00"] r2["read as big-endian:<br/>0x5000 = 20480 ✘"]
```

> **Kid example.** Writing a date. Sweden writes "2026-10-20" (big part first); the US writes "10/20/2026". If two people don't agree on the order, the same digits mean a different day. `htons` means "rewrite the date in the agreed order before sending".

**Finding your host's order.** There is no single standard, and machines of both kinds exist. The lecture's test program stores the number `0x0102` and then looks at its two bytes one by one:

```c
union { short s; char c[sizeof(short)]; } un;
un.s = 0x0102;
if (un.c[0] == 1 && un.c[1] == 2)      printf("big-endian\n");
else if (un.c[0] == 2 && un.c[1] == 1) printf("little-endian\n");
else                                   printf("unknown\n");
```

A `union` lets the same two bytes of memory be read either as one number (`s`) or as two separate bytes (`c[0]`, `c[1]`). If the first byte is 1, the "big" part came first: big-endian.

**Where it matters.** Network protocols must fix a **network byte order**, so that the sending and receiving protocol stacks agree in which order the bytes of multi-byte fields (addresses, port numbers) are sent. **IP uses big-endian.** In theory an implementation could keep socket address fields in host order and convert when building headers, "but, not :(": in practice the socket address structures already hold these fields **in network byte order**, so **you** must convert.

**The conversion functions.** The network is big-endian; the host may be either. The functions work on 16-bit ("short") and 32-bit ("long") values:

| Function | Read it as | Use it for |
|---|---|---|
| `htons()` | host to network, short | port numbers, before sending |
| `htonl()` | host to network, long | IPv4 addresses, before sending |
| `ntohs()` | network to host, short | port numbers, after receiving |
| `ntohl()` | network to host, long | IPv4 addresses, after receiving |

```mermaid
flowchart LR
  H["your computer's order<br/>(host order)"] -->|"htons / htonl<br/>before sending"| N["big-endian<br/>(network order)"]
  N -->|"ntohs / ntohl<br/>after receiving"| H
```

On a big-endian machine these functions do nothing, which is exactly why forgetting them is dangerous: the program works on one kind of machine and silently gives wrong numbers on another. You will also see many "structure casts" in socket code ([section 4.3](#43-socket-address-structures)).

**Remember:**

- **Little-endian** (Intel/AMD, most ARM): low byte first. **Big-endian**: high byte first. **The network is big-endian.**
- Port 80 = `0x0050`; sent without conversion from a little-endian PC it would be read as 20480.
- `htons`/`htonl` before sending; `ntohs`/`ntohl` after receiving. "s" = 16-bit (ports), "l" = 32-bit (IPv4 addresses).
- Socket address structures already hold values **in network byte order**, so **you** convert.

### 4.2 IP addresses and names

**In one sentence:** hosts have 32-bit IP addresses stored in **network byte order**, names map to addresses through DNS, and the modern call to look them up is **`getaddrinfo`**.

**The programmer's view** has three mappings:

1. **Hosts** are mapped to a set of **32-bit IP addresses**, for example `128.2.203.179`.
2. The set of IP addresses is mapped to **Internet domain names**: `128.2.203.179` is `www.cs.cmu.edu`.
3. A **process** on one host can communicate with a process on another host over a **connection**.

None of these is one-to-one: a host may have several addresses, a name may have several addresses, and many connections may exist between the same pair of addresses.

```mermaid
flowchart LR
  N["name<br/>www.cs.cmu.edu"] <-->|DNS| A["IP address<br/>128.2.203.179"]
  A --> H["host<br/>(one of its adapters)"]
  H --> P["process"] <-->|"connection"| P2["process on<br/>another host"]
```

**IP addresses in memory.** A 32-bit IPv4 address is stored in a structure:

```c
struct in_addr {
    unsigned int s_addr;   /* network byte order (big-endian) */
};
```

IP addresses are **always stored in memory in network byte order**. The same is true for **any integer** carried in a packet header, for example the **port number**. Hence the four conversion functions `htonl`, `htons`, `ntohl`, `ntohs`.

**DNS.** DNS maps between names and addresses in a **worldwide distributed database**.

- **`getaddrinfo()`** is the modern resolver call. **Use this.** It is **thread-safe** (unlike `gethostbyname()`), returns an **ordered list of addresses** (try each in turn), and is **protocol-independent**: one call serves IPv4 and IPv6.
- **`getnameinfo()`** does the reverse direction (address to name).
- **Legacy:** `gethostbyname()`, `gethostbyaddr()` and `struct hostent` are **IPv4-only and not thread-safe**. You will meet them in older code and in the labs.

> **Kid example.** DNS is the phone book: you know your friend's name, and the phone book gives you the number. `getaddrinfo` is a modern phone book that may give you **several** numbers (home, mobile) in a good order to try, and that works for both old and new kinds of numbers (IPv4 and IPv6).

**Remember:**

- Host → IP addresses → names; none of these is one-to-one.
- `struct in_addr` holds a 32-bit IPv4 address, always in **network byte order** (ports too).
- Use **`getaddrinfo`** (thread-safe, list of addresses, IPv4 + IPv6). Reverse: **`getnameinfo`**.
- Legacy `gethostbyname`, `gethostbyaddr`, `struct hostent`: IPv4-only, not thread-safe.

### 4.3 Socket address structures

**In one sentence:** you fill in an IPv4-specific structure `sockaddr_in` (family, port, address) and pass a pointer to it, **cast** to the generic `sockaddr`, to `connect`, `bind` and `accept`.

**Two structures.** The functions `connect`, `bind` and `accept` take a pointer to a **generic** socket address. Programs fill in an **Internet-specific** one and **cast** (re-label) its pointer to the generic type.

```c
struct sockaddr {              /* generic: for connect, bind, accept */
    unsigned short sa_family;  /* protocol family */
    char           sa_data[14];/* address data */
};

struct sockaddr_in {           /* Internet (IPv4) specific */
    unsigned short sin_family; /* address family, always AF_INET */
    unsigned short sin_port;   /* port number, network byte order */
    struct in_addr sin_addr;   /* IP address, network byte order */
    unsigned char  sin_zero[8];/* padding to the size of struct sockaddr */
};
```

- A socket address structure is **always passed by reference** (as a pointer) to socket functions.
- You **must cast** `(sockaddr_in *)` to `(sockaddr *)` for `connect`, `bind` and `accept`.
- Why the generic type at all? It is "**necessary only because C did not have generic (`void *`) pointers when the sockets interface was designed**". The API supports many address families of different sizes, and in 1983 C the only way to say "any of these" was one generic struct plus casts.

> **Kid example.** A form at the post office. The general form just says "type of address" plus a blank box. For a Swedish address you fill in a special Swedish form (street, postcode, town), but you hand it in at the counter that only accepts "general forms". The cast is like writing "general form" on top of your Swedish form: same paper, new label. The clerk reads the "type of address" line first and then knows how to read the rest.

**The length field.** Some systems have an extra first field **`sin_len`** (the length of the structure, 16). It simplifies the handling of **variable-length** address structures inside the kernel. You **never need to set or read it**, unless you work with routing sockets. You only work with `sin_family`, `sin_addr` and `sin_port`. **`sin_zero` is unused**, but **we always set the entire structure to 0 before filling it in**.

**Self-definition.** Socket address structures are **self-defining**: they always **begin with a field (the "family")** that says which address family the structure holds. Newer implementations that support variable-length structures also put a **length** field at the beginning. So the kernel can look at the first field of any structure and know how to read the rest.

**The full declarations:**

```c
struct in_addr {
    in_addr_t s_addr;          /* 32-bit IPv4 address, network byte ordered */
};
struct sockaddr_in {
    uint8_t        sin_len;    /* length of structure (16) */
    sa_family_t    sin_family; /* AF_INET */
    in_port_t      sin_port;   /* 16-bit TCP or UDP port, network byte ordered */
    struct in_addr sin_addr;   /* 32-bit IPv4 address, network byte ordered */
    char           sin_zero[8];/* unused */
};
```

Of five fields, only two carry real information (port and address); the rest make the types, byte order and casting line up. The IPv6 version is a different, larger structure called `sockaddr_in6`.

```mermaid
block-beta
  columns 5
  a["sin_len<br/>16 (some systems)"] b["sin_family<br/>AF_INET"] c["sin_port<br/>htons(13)"] d["sin_addr<br/>IPv4 address"] e["sin_zero<br/>unused, all 0"]
```

```mermaid
flowchart LR
  P["your program fills in<br/>sockaddr_in<br/>family, port, address"] -->|"cast the pointer to<br/>(sockaddr *)"| F["connect / bind / accept"]
  F -->|"kernel reads the<br/>family field first"| K[knows it is IPv4]
```

**Remember:**

- `sockaddr` = **generic**; `sockaddr_in` = **IPv4**; `sockaddr_in6` = **IPv6**.
- Always passed **by reference** (a pointer), and **cast** to `(sockaddr *)` for `connect`, `bind`, `accept`.
- The generic type exists only because 1983 C had no `void *`.
- You use `sin_family`, `sin_port`, `sin_addr`; **zero the whole structure first**; `sin_len` (if present) you never touch.
- Structures are **self-defining**: the **family** field comes first.

### 4.4 Converting addresses between text and binary

**In one sentence:** use **`inet_pton`** (text → binary) and **`inet_ntop`** (binary → text), which work for both IPv4 and IPv6, instead of the old IPv4-only functions.

Humans write addresses as text (`"192.168.1.5"` or `"2001:db8::1"`); socket address structures store them as **binary numbers in network byte order**.

**The old functions: `inet_aton`, `inet_addr`, `inet_ntoa`.** They convert an IPv4 dotted-decimal string to and from a 32-bit address. They are **IPv4 only** and are used a lot in existing code. Avoid writing them: `inet_addr` returns, on error, a value that is also a valid address (255.255.255.255, the broadcast address), and `inet_ntoa` returns a pointer to one shared ("static") buffer, so it is **not thread-safe** and a second call overwrites the first.

**The new functions: `inet_pton` and `inet_ntop`.** The letters mean **p = presentation** (a string) and **n = numeric** (binary). They handle **both IPv4 and IPv6**.

```c
int inet_pton(int family, const char *strptr, void *addrptr);
    /* Returns: 1 if OK, 0 if input not a valid presentation format, -1 on error */
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
    /* Returns: pointer to result if OK, NULL on error */
```

```mermaid
flowchart LR
  T["text (presentation)<br/>'192.168.1.5'"] -->|inet_pton| B["binary (numeric)<br/>4 bytes, network order"]
  B -->|inet_ntop| T
```

Three improvements: an explicit **family** argument (so IPv6 works), **three distinct results** for `inet_pton` (good, bad string, error), and the result is written into **your own buffer** (no shared static storage).

> **Kid example.** `inet_pton` is like turning "twelve" written in words into the digits "12" so a calculator can use it; `inet_ntop` turns "12" back into words so a person can read it. The old tool had one shared whiteboard for its answers, so if two children asked at once, the second answer wiped out the first. The new tool writes the answer in **your own** notebook.

**`sock_ntop`** is a helper that prints any socket address. It looks at the **family field** (`sa->sa_family`): for `AF_INET` it casts to `sockaddr_in`, calls `inet_ntop(AF_INET, …)`, and if the port is not zero, adds `:port` (after `ntohs`). The IPv6 case is the same with `sockaddr_in6`. The caller never needs to know which family it was: this is **protocol-independent** code, made possible by self-defining structures.

**Protocol-independent clients:**

- Use **`sockaddr_storage`**, which is **large enough for any address family**, instead of `sockaddr_in`.
- **`getaddrinfo` returns a list**: walk it and **connect to the first address that works**.
- **One code path serves IPv4 and IPv6**, with no conditional compilation.

Why it matters: IPv6 now carries the majority of traffic, so a client written only for `sockaddr_in` cannot reach a growing part of the Internet.

```mermaid
flowchart TB
  N["host name<br/>www.example.com"] --> G[getaddrinfo]
  G --> L["list: IPv6 address, IPv4 address, ..."]
  L --> T{"new socket,<br/>connect to the next address"}
  T -->|works| OK[use this connection]
  T -->|fails| X[close that socket] --> T
```

**The echo server identifies its client.** After `accept`, the server knows the client's address and can look up its name:

```c
hp = Gethostbyaddr((const char *)&clientaddr.sin_addr.s_addr,
                   sizeof(clientaddr.sin_addr.s_addr), AF_INET);
haddrp = inet_ntoa(clientaddr.sin_addr);
printf("server connected to %s (%s)\n", hp->h_name, haddrp);
```

In words: do a **reverse DNS lookup** of the client's address (`Gethostbyaddr`), turn the address into text (`inet_ntoa`), and print both. Two cautions: reverse lookups are **slow** (a network round trip per connection), and the name is **not trustworthy** for authentication, because the owner of an address block controls its reverse names. This code uses the legacy calls; `getnameinfo` is the modern one.

**Remember:**

- Old, IPv4-only: `inet_aton`, `inet_addr` (error value = 255.255.255.255, a valid address!), `inet_ntoa` (shared static buffer, not thread-safe).
- New: **`inet_pton`** (p = presentation/text → n = numeric/binary; returns 1 OK, 0 bad text, −1 error) and **`inet_ntop`**. Both IPv4 and IPv6.
- `sock_ntop` reads the **family** field and prints any address: protocol-independent.
- Protocol-independent clients: `sockaddr_storage` + walk the `getaddrinfo` list, new socket per attempt.
- Reverse lookups are **slow** and **not trustworthy** for authentication.

## 5. Robust I/O and the elementary socket calls

**What this chapter is about:** how to read and write safely on a byte stream (`readn`, `writen`, `readline`), and then each basic call once more, in detail: `socket`, `connect`, `bind`, `listen`, `accept` and `close`, with their errors and the state changes they cause.

### 5.1 Short reads and writes: readn, writen and readline

**In one sentence:** a socket `read` or `write` may move **fewer bytes than you asked for**, which is normal, so helper functions loop until the job is really done.

**How TCP sockets differ from normal file I/O.** It is **not an error** for `read` or `write` on a socket to transfer **fewer bytes than requested**, because buffer limits in the kernel may be reached. You simply call `read` or `write` again. With `read` this can always happen on a stream socket; with `write` it is normally seen only on non-blocking sockets. Still, the examples always call **`writen`** instead of `write`, in case the implementation returns a short count.

**`readn`: read exactly n bytes.**

```c
ssize_t readn(int fd, void *vptr, size_t n) {
    size_t  nleft = n;  ssize_t nread;  char *ptr = vptr;
    while (nleft > 0) {
        if ( (nread = read(fd, ptr, nleft)) < 0) {
            if (errno == EINTR) nread = 0;   /* interrupted: call read() again */
            else return (-1);
        } else if (nread == 0)
            break;                           /* EOF */
        nleft -= nread;
        ptr   += nread;
    }
    return (n - nleft);                      /* return >= 0 */
}
```

In words: keep a count of bytes still missing. While some are missing, call `read`. If it fails because a **signal interrupted it** (`EINTR`), that is not a real error: count it as 0 bytes and try again. If it returns 0, the peer closed: stop early. Otherwise move forward in the buffer and reduce the count. At the end, return how many bytes were really read.

```mermaid
flowchart TB
  S["nleft = n<br/>(bytes still missing)"] --> Q{nleft > 0 ?}
  Q -->|no| R["return n - nleft"]
  Q -->|yes| RD["read(fd, ptr, nleft)"]
  RD -->|"< 0 and EINTR"| Q
  RD -->|"< 0, other error"| E["return -1"]
  RD -->|"= 0: EOF"| R
  RD -->|"> 0: got some bytes"| M["nleft -= nread<br/>ptr += nread"] --> Q
```

> **Kid example.** Waiting for a 10-piece puzzle that arrives in envelopes. You don't start the puzzle when the first envelope comes; you keep opening envelopes (`readn`) until you have all 10 pieces, or until the post office says "that's all" (EOF). If the doorbell interrupts you (EINTR), you just go back and continue opening.

**`writen`: write exactly n bytes.** The mirror image: loop while bytes remain; if `write` was interrupted (`EINTR`), try again; on another error return −1; otherwise advance. A short write happens when the socket's **send buffer** is full (slow peer, closed window, congestion): the notes call it "backpressure made visible".

**`readline`: read one line.** The simple version reads **one byte at a time** until it finds `\n` or reaches the maximum length, handling `EINTR` and EOF. The lecture labels it "**PAINFULLY SLOW VERSION, example only**": it calls `read` (a system call) **once for every byte**. It is tempting to use the standard C I/O library (stdio, which buffers), but **stdio's buffering creates its own problems**: the **state of the stdio buffers is not exposed**, so data can sit hidden in the library's buffer, and other code (for example `select`) cannot see it.

**Report protocol violations.** Network programs should follow the protocol, but to be safe, **check for unexpected network traffic** too. Protocol violations should be **reported as errors**, so that bugs are noticed and fixed and **malicious attempts are detected**, and applications should **recover and keep working** if possible. With stdio there is **no way to tell if unexpected data is held in its buffers**.

**A better readline: exposing the internal buffer.** The improved version keeps its **own** buffer and makes its state visible:

```c
static int   read_cnt;           /* bytes left in the buffer */
static char *read_ptr;           /* next byte to hand out */
static char  read_buf[MAXLINE];  /* the buffer */

static ssize_t my_read(int fd, char *ptr) {
    if (read_cnt <= 0) {
      again:
        if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
            if (errno == EINTR) goto again;
            return (-1);
        } else if (read_cnt == 0)
            return (0);
        read_ptr = read_buf;
    }
    read_cnt--;
    *ptr = *read_ptr++;
    return (1);
}

ssize_t readlinebuf(void **vptrptr) {   /* lets the caller see leftover data */
    if (read_cnt)
        *vptrptr = read_ptr;
    return (read_cnt);
}
```

In words: `my_read` hands out **one byte at a time**, but it only calls the real `read` when its buffer is **empty**, and then it reads a **whole buffer-full** at once. So there is one system call per buffer instead of one per byte. `readline` is the same loop as before, with `my_read` in place of `read`. `readlinebuf` lets the caller **check whether more data was received beyond one line**. Also, the `static` buffer is shared by the whole program, so this version is **not thread-safe** and cannot serve two sockets at once.

```mermaid
flowchart LR
  K["kernel socket buffer"] -->|"one read() fills<br/>the whole read_buf"| B["read_buf<br/>(read_cnt bytes left)"]
  B -->|"my_read hands out<br/>1 byte at a time"| L["readline builds<br/>the line"]
  B -.->|"readlinebuf:<br/>'is anything left?'"| C["caller can see<br/>leftover data"]
```

> **Kid example.** The slow `readline` walks to the fridge for **every single grape**. The better one carries the whole bowl of grapes to the table once and eats them one by one from the bowl. And it keeps the bowl on the table where others can see whether grapes are still left, instead of hiding them in a closed box (stdio).

**Summary.** TCP sockets give a **byte stream**: there are **no record markers**, and a `read` can return **less than asked for, which is not an error**. `readn`, `writen` and `readline` help with this. However, **network programs should be written to act on buffers rather than lines**.

**Remember:**

- A **short** read or write is **not an error**; call again.
- `readn` / `writen` loop until **exactly n bytes** are done; `EINTR` → try again; `read` = 0 → EOF, stop.
- A short write = full **send buffer** = "backpressure made visible".
- Simple `readline` = one system call **per byte** ("painfully slow"). stdio hides its buffer, so `select` cannot see the data.
- Better `readline` keeps its own **visible** buffer (`readlinebuf`), one system call per buffer; but it is **not thread-safe**.
- **Report protocol violations** as errors, and recover if possible.
- Write programs that act on **buffers**, not lines.

### 5.2 The socket function

**In one sentence:** `socket` creates an unconnected socket of the kind you ask for and gives you a descriptor; it contacts nobody.

The lecture revisits each elementary call, now with its state change. The lecture points back to the state diagram.

**`socket`.** To do network I/O, the first thing a process must do is call `socket`, saying which kind of communication it wants:

```c
int socket(int family, int type, int protocol);
```

- `family`: `AF_INET` (IPv4), `AF_INET6` (IPv6), and others.
- `type`: `SOCK_STREAM` (reliable byte stream) or `SOCK_DGRAM` (datagrams).
- `protocol`: usually `0` = the obvious choice (TCP for `SOCK_STREAM`, UDP for `SOCK_DGRAM`).

It creates **no connection, reserves no port and contacts nothing**; it only allocates a kernel structure and gives you a descriptor.

**Return value and errors.** It returns a **non-negative integer: the file descriptor**. Errors:

- **No permission** to create a socket of this type or protocol;
- **No support** for this address family or protocol;
- **Process file table overflow** (this program has too many open descriptors);
- the **system limit** on the total number of open files has been reached;
- **Insufficient memory**: the socket cannot be created until resources are freed.

**State.** **CLOSED** is the state a socket starts in when `socket` creates it. From there, `connect` leads towards SYN_SENT (a client), while `bind` + `listen` lead to LISTEN (a server).

```mermaid
flowchart LR
  S["socket()<br/>state: CLOSED"] -->|"connect (client)"| A[SYN_SENT]
  S -->|"bind + listen (server)"| B[LISTEN]
```

> **Kid example.** `socket` is like buying a new phone in a shop. You now own a phone (the descriptor), but it is not calling anyone, it has no contacts, and nobody knows its number yet.

**Remember:**

- `socket(family, type, protocol)`: e.g. `AF_INET`, `SOCK_STREAM`, `0` → TCP over IPv4.
- Returns a **non-negative descriptor**; contacts nobody, reserves no port.
- Errors: no permission, no support, too many descriptors (process or system), not enough memory.
- New socket is in state **CLOSED**.

### 5.3 The connect function

**In one sentence:** `connect` makes the client's TCP do the three-way handshake and returns only when the connection is up or has clearly failed (timeout, refused, unreachable).

**`connect`** is how a TCP **client** establishes a connection with its server:

```c
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```

- The client **does not have to call `bind`** first: the kernel **chooses an ephemeral port and the source IP address** if necessary.
- For TCP, `connect` **starts the three-way handshake** and **returns only when the connection is established, or an error occurs**.

**Three ways `connect` can fail:**

| What happens | What the client sees |
|---|---|
| **No response** to the SYN | 4.4BSD sends one SYN when `connect` is called, **another 6 s later and another 24 s later**; if no response arrives within **75 s** in total, a **timeout** error is returned. |
| The server answers the SYN with an **RST** | **"connection refused"**, at once. It means **no process is waiting for connections** on that port; the server program is probably not running. |
| A router answers with **ICMP "destination unreachable"** | A **soft error**: the kernel **saves the message but keeps sending SYNs**. If there is still no response after the fixed time (75 s), the saved error is returned: "**host or network unreachable**". The ICMP error may be **transient** (a routing problem that will be fixed), which is why it is not treated as final at once. |

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server (silent)
  Note over C: time 0 s
  C->>S: SYN
  Note over C: 6 s later
  C->>S: SYN again
  Note over C: 24 s after that
  C->>S: SYN again
  Note over C: 75 s in total: give up, timeout error
```

> **Kid example.** Ringing a doorbell. Nobody comes after three rings and a long wait: timeout. Someone shouts "nobody by that name lives here!": refused. A neighbour says "that road is closed": unreachable, but you keep trying a bit in case it reopens.

The lecture lists when TCP sends an **RST** ("something is wrong"):

- a **SYN arrives for a port with no listening server**;
- TCP wants to **abort** an existing connection;
- TCP receives a segment for a connection that **does not exist**.

**State.** `connect` moves the socket from **CLOSED to SYN_SENT**, and on success to **ESTABLISHED**. On failure the **socket is no longer usable and must be closed**; you **cannot call `connect` again** on it. To retry: close, create a new socket, connect again. (This matters when walking a `getaddrinfo` list: each attempt needs a fresh socket.)

```mermaid
flowchart LR
  A[CLOSED] -->|"connect: send SYN"| B[SYN_SENT]
  B -->|"SYN+ACK arrives, send ACK"| C[ESTABLISHED]
  B -->|"RST: connection refused"| D[unusable: close it]
  B -->|"no answer for 75 s: timeout"| D
```

**Technical clarification.** The lecture once writes "STABLISHED"; it means **ESTABLISHED**.

**Remember:**

- Client need not call `bind`: the kernel picks the **ephemeral port** and **source IP**.
- `connect` = three-way handshake; returns when **established** or on **error**.
- No answer: SYNs at 0, +6 s, +24 s; give up after **75 s** → **timeout**.
- **RST** → "**connection refused**" at once (nobody listening).
- **ICMP destination unreachable** → **soft error**: keep trying; after 75 s report "host or network unreachable".
- RST is sent for: SYN to a port with no listener, aborting a connection, segment for a non-existent connection.
- After a failed `connect`, **close the socket and make a new one**.

### 5.4 The bind function

**In one sentence:** `bind` gives a socket its **own** local address and port; servers use it for their well-known port, clients usually let the kernel choose.

**`bind`** assigns a **local protocol address** to a socket: a 32-bit IPv4 or 128-bit IPv6 address together with a 16-bit TCP or UDP port.

```c
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

- **Servers bind their well-known port** when they start, so clients know where to find them.
- If a socket is not bound, the kernel **chooses an ephemeral port** when `connect` or `listen` is called. That is normal for TCP **clients**, rare for TCP **servers**.

**Binding an IP address.** A process can bind a **specific IP address**, which **must belong to an interface of this host**.

- For a **TCP client**, this sets the **source IP address** of its datagrams. Clients normally don't; the kernel chooses the source address based on the outgoing interface.
- For a **TCP server**, it **restricts the socket to connections destined to that IP address only**. If the server does not bind an address, the kernel uses the **destination address of the client's SYN** as the server's source address.

**What the kernel chooses:**

| IP address given | Port given | Result |
|---|---|---|
| wildcard | 0 | kernel chooses IP address and port |
| wildcard | nonzero | kernel chooses IP address, process specifies port |
| local IP address | 0 | process specifies IP address, kernel chooses port |
| local IP address | nonzero | process specifies IP address and port |

With port 0, the kernel picks an ephemeral port **when `bind` is called**. With the wildcard address, the kernel **does not choose the local IP address until the socket is connected** (then it uses the address the connection arrived on).

```mermaid
flowchart TB
  Q1{"IP address<br/>given?"} -->|"wildcard (INADDR_ANY)"| K1["kernel picks the IP<br/>later, when connected"]
  Q1 -->|"a local IP"| P1["that IP is used"]
  Q2{"port given?"} -->|"0"| K2["kernel picks an<br/>ephemeral port at bind"]
  Q2 -->|"nonzero, e.g. 13"| P2["that port is used"]
```

> **Kid example.** `bind` is like putting a name plate on your apartment door. A shop (server) must put up its name plate, otherwise nobody finds it. A visitor (client) does not need one; the doorkeeper gives them a temporary visitor badge. If the name plate "Apartment 13" is still in use by the previous tenant (TIME_WAIT), you get "address already in use".

**Errors:**

- the address is **protected** and the user is not the superuser (ports below 1024 need privileges);
- the address is **already in use** (often a previous connection in TIME_WAIT; fix with `SO_REUSEADDR`);
- `sockfd` is not a valid descriptor;
- the socket is **already bound**.

**Remember:**

- `bind` sets the **local** address + port (IPv4 32-bit or IPv6 128-bit address, 16-bit port).
- **Servers** bind their well-known port; **clients** usually don't (kernel picks an ephemeral port at `connect`/`listen`).
- Binding a specific IP: client → sets the **source** IP; server → only accepts connections **to that IP**.
- Wildcard + port 0 → kernel picks both. Port 0 → ephemeral port chosen **at `bind`**. Wildcard → IP chosen **when connected**.
- Errors: protected port (< 1024, need superuser), **address already in use** (`SO_REUSEADDR`), bad descriptor, already bound.

### 5.5 The listen function and its two queues

**In one sentence:** `listen` turns a socket into a **waiting** server socket, and the kernel then keeps two queues for it: half-finished handshakes and finished connections waiting for `accept`.

**`listen`** is called **only by a TCP server**:

```c
int listen(int sockfd, int backlog);
```

- It converts an unconnected socket, which by default is assumed to be an **active** socket (one that will call `connect`), into a **passive** socket: the kernel should **accept incoming connection requests** to it.
- It moves the socket from **CLOSED to LISTEN**.
- `backlog` says the **maximum number of connections the kernel should queue** for this socket.
- It must be called **before `accept`**, normally after `socket` and `bind`.

Before `listen`, a SYN to that port gets an RST. After `listen`, it gets a SYN+ACK, **whether or not the program has called `accept` yet**.

**The two queues.** For each listening socket the kernel keeps:

1. an **incomplete connection queue** (state **SYN_RCVD**): one entry for each SYN that has arrived, while the server waits for the handshake to complete;
2. a **completed connection queue** (state **ESTABLISHED**): one entry for each client whose handshake has completed.

```mermaid
flowchart LR
  SYN[client SYN arrives] --> IQ["incomplete queue<br/>SYN_RCVD<br/>(kernel sends SYN+ACK)"]
  IQ -->|"client's ACK arrives:<br/>handshake complete"| CQ["completed queue<br/>ESTABLISHED"]
  CQ -->|"accept takes the<br/>first entry"| P[server program]
  IQ -.->|"no ACK within 75 s:<br/>entry removed"| X[dropped]
```

> **Kid example.** A doctor's waiting room with two areas. At the front door, people who said "hello" but haven't shown their health card yet wait in the hallway (incomplete queue). Once they have shown it, they sit in the waiting room (completed queue). The doctor (`accept`) always calls the first person from the waiting room, never from the hallway. If someone in the hallway never shows a card, after a while they are sent away.

Once an entry is created on the incomplete queue, the **listening socket's parameters are copied** to the new connection; the **server process is not involved**. So socket options set on the listening socket apply to every connection made from it.

**The backlog:**

- Historically, `backlog` was the maximum for the **sum of both queues**.
- There is **no formal definition** of what it means; Berkeley-derived systems multiply it by **1.5**.
- Entries stay incomplete until the **third segment** of the handshake arrives, or until the entry **times out (75 s)**.
- **5** was the maximum supported by 4.2BSD: enough when servers handled hundreds of connections **a day**, but busy servers handle **millions a day**. Newer kernels support larger values, and administrators can change the maximum.
- "What value should the application specify, since 5 is often inadequate?" Answer: **set it generously** (a few hundred is normal); modern systems cap it anyway.

**A `Listen` wrapper that can be tuned:**

```c
void Listen(int fd, int backlog) {
    char *ptr;   /* can override 2nd argument with environment variable */
    if ( (ptr = getenv("LISTENQ")) != NULL)
        backlog = atoi(ptr);
    if (listen(fd, backlog) < 0)
        err_sys("listen error");
}
```

In words: if an **environment variable** called `LISTENQ` is set (a setting you can give from the shell), use its value as the backlog; then call `listen` and stop on error. So the backlog can be tuned **without recompiling**. Errors of `listen`: another socket is already listening on the same port; `sockfd` is not a valid socket; the socket type does not support `listen`.

**Additional context.** The two queues are what a **SYN flood** attacks: an attacker sends many SYNs from fake addresses and never completes the handshake, so the incomplete queue fills up and real clients are turned away. **SYN cookies** are the standard defence (see the TCP guide, [04-TCP.md, section 5.3](04-TCP.md#53-related-quiz-topics-not-in-the-lecture)).

**Remember:**

- `listen(sockfd, backlog)`: only servers; **active → passive** socket; CLOSED → **LISTEN**; call before `accept`.
- After `listen`, the kernel answers SYNs with SYN+ACK even **before** the program calls `accept`.
- Two queues: **incomplete** (SYN_RCVD) and **completed** (ESTABLISHED); `accept` takes from the **completed** queue.
- Incomplete entries time out after **75 s**. Options on the listening socket are **copied** to new connections.
- Backlog: no formal definition, Berkeley × **1.5**, 4.2BSD max **5**; today **set it generously**.
- The `Listen` wrapper reads the **`LISTENQ`** environment variable.
- **SYN flood** fills the incomplete queue; **SYN cookies** defend.

### 5.6 The accept function

**In one sentence:** `accept` takes the next finished connection from the queue (or sleeps until there is one) and gives the server a **brand-new descriptor** for talking to that client.

**`accept`** returns the **next completed connection** from the **front of the completed connection queue**. If that queue is empty, the process is **put to sleep** (by default).

```c
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```

- `cliaddr` and `addrlen` return the **client's address**.
- `addrlen` is a **value-result argument**: **before** the call, it holds the size of the structure `cliaddr` points to; **after** the call, it holds the number of bytes the kernel actually stored there.

> **Kid example of a value-result argument.** You give a friend an empty box and a note saying "this box holds 16 marbles" (value in). Your friend puts marbles in the box and changes the note to "I put in 16" (result out). The same note travels both ways.

**A brand-new descriptor.** `accept` returns a **new descriptor created by the kernel**, which refers to the TCP connection with this client.

- The **first argument** of `accept` is the **listening socket**; the **return value** is the **connected socket**.
- A server normally creates **only one listening socket**, which exists for the **lifetime of the server**.
- The kernel creates **one connected socket for each accepted client** (each completed handshake). When the server has finished with that client, it **closes the connected socket**.

**`accept` illustrated.** `accept` blocks waiting for a connection request. It returns a connected descriptor (`connfd`) with the **same properties** as the listening descriptor (`listenfd`). It returns when the connection is created and ready for I/O. **All I/O with the client goes through the connected socket.** `accept` also fills in the client's IP address.

```c
int listenfd;                 /* listening descriptor */
int connfd;                   /* connected descriptor */
struct sockaddr_in clientaddr;
int clientlen;
clientlen = sizeof(clientaddr);
connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
```

The three panels of the lecture's figure:

```mermaid
sequenceDiagram
  participant C as Client (clientfd)
  participant S as Server
  Note over S: 1. server blocks in accept<br/>on listenfd (3)
  C->>S: 2. client calls connect,<br/>blocks, connection request
  Note over C,S: 3. accept returns connfd (4),<br/>connect returns,<br/>clientfd and connfd are connected
  Note over S: listenfd (3) keeps listening
```

**Connected vs listening descriptors:**

| | Listening descriptor | Connected descriptor |
|---|---|---|
| What it is | the end point for **client connection requests** | the end point of **the connection** between client and server |
| How many | created **once** | a **new one** each time a request is accepted |
| Lifetime | the **whole life of the server** | only as long as it takes to **serve that client** |

**Why the distinction?** It allows **concurrent servers** that talk over many client connections at once. For example, each time a request arrives, the server **forks a child** to handle it and goes straight back to `accept`.

```mermaid
flowchart LR
  L["listenfd = 3<br/>(one, for the server's whole life)"] -->|accept| A["connfd = 4<br/>client A"]
  L -->|accept| B["connfd = 5<br/>client B"]
  L -->|accept| C["connfd = 6<br/>client C"]
```

> **Kid example.** A bakery with a ticket machine at the entrance (listening descriptor). The machine stays there all day. Every customer who comes in gets a **new** paper ticket with their own number (connected descriptor). The baker talks to each customer using their ticket, and throws the ticket away when that customer leaves. The machine itself is never thrown away.

**State.** The connection that `accept` returns is **already ESTABLISHED**. The listening socket stays in **LISTEN** and is not changed.

**Errors:** the descriptor is invalid; **the connection has been aborted** (a client completed the handshake and then reset the connection before `accept`; the right reaction is to ignore it and call `accept` again); the socket is not listening or `addrlen` is invalid; the **per-process** or **system** limit on open descriptors is reached; the socket is not of type `SOCK_STREAM`; not enough free memory (often the **socket buffer limits**, not the system memory); a protocol error; **firewall rules forbid the connection**.

**Remember:**

- `accept` takes the **first** entry of the **completed** queue; sleeps if it is empty.
- `addrlen` is a **value-result** argument: size in, bytes used out.
- Argument = **listening** socket (one, lives as long as the server); return value = **new connected** socket (one per client, closed after serving).
- The returned connection is already **ESTABLISHED**; the listener stays in **LISTEN**.
- "Connection aborted" error: just call `accept` again.
- The distinction makes **concurrent servers** possible (fork a child per client).

### 5.7 The close function

**In one sentence:** `close` marks the socket closed and returns at once; TCP then sends any queued data and the FIN in the background.

**`close`** closes a socket and terminates the TCP connection:

```c
int close(int sockfd);
```

- It **marks the socket as closed and returns immediately**.
- The descriptor **can no longer be used** to read or write.
- TCP will **try to send any data already queued** to the other end, and **after that** the normal termination sequence (FIN, ACK, …) takes place.
- The **`SO_LINGER`** socket option can change this default behaviour.

Also: because `close` returns **at once**, a successful `close` tells you **nothing about whether your data arrived**. If you need certainty, the application must send its own acknowledgement. And `close` ends **both** directions; `shutdown` can end just one ([section 6.6](#66-the-batch-mode-bug-and-shutdown)).

```mermaid
sequenceDiagram
  participant P as Program
  participant K as Kernel TCP
  participant O as Other end
  P->>K: close(sockfd)
  K-->>P: returns at once
  K->>O: remaining queued data
  K->>O: FIN
  Note over P: the program does not know<br/>if the data arrived
```

> **Kid example.** Dropping a letter in the post box. As soon as it falls in, you walk away (close returns). The post office will deliver it later. But dropping it in does not prove that your friend got it; if you need to be sure, your friend must write back.

**Remember:**

- `close` returns **immediately**; the descriptor cannot be used any more.
- TCP still sends queued data, then FIN. `SO_LINGER` can change this.
- A successful `close` does **not** prove the data arrived.
- `close` ends **both** directions; `shutdown` can end **one**.

## 6. The echo service, I/O models and select

**What this chapter is about:** a richer example (the echo client and server), the problem of a program that must watch **two things at once**, the different ways of waiting for input (I/O models), the `select` function that solves it, and two bugs that `select` and `shutdown` fix.

### 6.1 The echo client and server

**In one sentence:** the echo server sends back whatever it receives; the simple echo client works, but it cannot watch the keyboard and the socket at the same time.

The lecture builds a richer pair: the **echo** server sends back whatever the client sends; the client reads lines from the keyboard.

**In use:**

```text
hive> echoclient hive 5000
Please enter msg: 123
Echo from server: 123
Please enter msg: 456789
Echo from server: 456789

hive> echoserver 5000
server established connection with HIVE (128.252.21.14)
server received 4 bytes: 123
server received 7 bytes: 456789
```

(4 bytes for "123" because the newline is counted too; 7 bytes for "456789".)

**The echo server** sets up exactly like the daytime server (port 7 this time), then:

```c
for ( ; ; ) {
    clilen = sizeof(cliaddr);
    connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
    str_echo(connfd);          /* the whole conversation with this client */
    Close(connfd);
}

void str_echo(int sockfd) {
    ssize_t n;  char buf[MAXLINE];
  again:
    while ( (n = read(sockfd, buf, MAXLINE)) > 0)
        Writen(sockfd, buf, n);
    if (n < 0 && errno == EINTR) goto again;
    else if (n < 0) err_sys("str_echo: read error");
}
```

In words: accept a client, then in `str_echo` **read** whatever arrives and **write back exactly `n` bytes** (the number actually read) with `Writen`, until `read` returns 0 (the client closed). If `read` was interrupted by a signal, try again. Then close and accept the next client. This is still an **iterative** server, and now the problem is bigger: `str_echo` runs **until the client closes**, which may be minutes, while other clients wait.

> **Kid example.** The echo server is a parrot. Whatever you say, it says back to you, word for word. But this parrot can only listen to one child at a time, and it keeps listening to that child until the child walks away.

**The echo client:**

```c
sockfd = Socket(AF_INET, SOCK_STREAM, 0);
bzero(&servaddr, sizeof(servaddr));
servaddr.sin_family = AF_INET;  servaddr.sin_port = htons(SERV_PORT);
Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));
str_cli(stdin, sockfd);         /* do it all */

void str_cli(FILE *fp, int sockfd) {
    char sendline[MAXLINE], recvline[MAXLINE];
    while (Fgets(sendline, MAXLINE, fp) != NULL) {
        Writen(sockfd, sendline, strlen(sendline));
        if (Readline(sockfd, recvline, MAXLINE) == 0)
            err_quit("str_cli: server terminated prematurely");
        Fputs(recvline, stdout);
    }
}
```

In words: connect, then repeat: read a line from the keyboard (`Fgets`), send it (`Writen`), read the echoed line back (`Readline`), print it (`Fputs`).

```mermaid
flowchart LR
  K["Fgets<br/>line from keyboard"] --> W["Writen<br/>send to server"] --> R["Readline<br/>echo comes back"] --> P["Fputs<br/>print it"] --> K
```

**The problem.** The client has **two inputs at the same time**: the keyboard and the TCP socket. Suppose the client is **blocked in `fgets`** (waiting for the user to type) and the **server process is killed**. The server's TCP correctly sends a **FIN** to the client, but the client is **busy waiting on the keyboard**, so it **never sees the EOF until it reads from the socket**, possibly much later. One program, two things to watch, and no way to watch both.

```mermaid
sequenceDiagram
  participant U as User keyboard
  participant C as Echo client
  participant S as Server
  Note over C: blocked in fgets, waiting for the user
  S->>C: FIN (server was killed)
  Note over C: does not notice, still in fgets
  U->>C: types a line, much later
  C->>S: write fails or gets RST
  Note over C: only now discovers the server is gone
```

> **Kid example.** You are waiting by the front door for a parcel and can't hear the phone ringing in the kitchen. You need a way to wait for "whichever happens first".

**Remember:**

- Echo server: read `n` bytes, write back exactly `n` bytes with `Writen`, until `read` returns 0.
- "123" is **4 bytes** (the newline counts).
- The simple echo server is still **iterative**: one client for its whole conversation.
- The simple echo client blocks in `fgets` and **misses the server's FIN**: it has two inputs and watches only one.

### 6.2 I/O models

**In one sentence:** a program can wait for data by **blocking**, by **checking again and again** (non-blocking), or by asking the kernel to **tell it which of many descriptors is ready** (I/O multiplexing with `select`).

**I/O multiplexing.** We need a way for the process to **tell the kernel** that it wants to be **notified when one or more I/O conditions are ready**: input is ready to be read, or a descriptor can take more output. This is **I/O multiplexing**, provided by **`select`**. Instead of choosing one descriptor and blocking on it, the program gives the kernel a **set** and blocks on the set.

It is typically used when:

- a **client** handles **multiple descriptors** (for example the keyboard and a socket), or **multiple sockets**;
- a **TCP server** handles both a **listening socket and its connected sockets**;
- a server handles **both TCP and UDP**;
- a server handles **multiple services** and perhaps multiple protocols.

I/O multiplexing is **not limited to network programming**.

**The five I/O models:**

1. **Blocking I/O**,
2. **Non-blocking I/O**,
3. **I/O multiplexing** (`select` and `poll`),
4. signal-driven I/O (**not in this course**),
5. asynchronous I/O (**not in this course**).

A read has **two waiting phases**: (1) **waiting for data** to arrive, and (2) **copying the data** from the kernel into the program's buffer.

- **Blocking.** **By default, all sockets are blocking.** The program calls `read`; if there is no data, the process sleeps; when data arrives it is copied and `read` returns. The process waits through both phases.
- **Non-blocking.** You can set a socket to be non-blocking, which tells the kernel: "when an I/O operation I request cannot be completed without putting the process to sleep, **do not put the process to sleep, but return an error instead**" (the error is `EWOULDBLOCK` / `EAGAIN`). Calling it again and again ("polling") wastes processor time.
- **I/O multiplexing.** **Call `select` and block there**, instead of blocking in the actual I/O call. When `select` returns, one of the descriptors is ready, and then you call `read`.

**Comparison:**

| | Phase 1: wait for data | Phase 2: copy data from kernel to program |
|---|---|---|
| Blocking | blocked in `read` | blocked in `read` |
| Non-blocking | "check, check, check …" (each call returns at once) | blocked in `read` |
| I/O multiplexing | blocked in `select`, until "ready" | blocked in `read` |

"First phase handled differently, second phase handled the same (blocked in the call to `recvfrom`)." Also, only **asynchronous I/O** (not taught here) avoids blocking in both phases; this is the model of Linux's io_uring ([section 7.4](#74-c10k-and-c10m)).

```mermaid
sequenceDiagram
  participant P as Program
  participant K as Kernel
  Note over P,K: Blocking
  P->>K: read
  Note over K: wait for data, then copy
  K-->>P: data
  Note over P,K: Non-blocking
  P->>K: read
  K-->>P: EWOULDBLOCK (no data yet)
  P->>K: read again
  K-->>P: data
  Note over P,K: I/O multiplexing
  P->>K: select on many descriptors
  K-->>P: descriptor 4 is ready
  P->>K: read on 4
  K-->>P: data
```

> **Kid example.** A teacher with 30 students. Blocking: stand next to one student until they answer. Non-blocking: keep running from desk to desk asking "done yet?". Multiplexing (`select`): say "raise your hand when ready", sit down, and help whoever raises a hand.

Multiplexing costs **two system calls** (select, then read) where blocking needs one, so it is worse for a single descriptor. It pays off as soon as there are **several** descriptors to watch with one thread.

**Technical clarification.** The lecture's explanation describes "all five models side by side", but the figure in the lecture shows only **three** columns (blocking, non-blocking, I/O multiplexing). The full figure in Stevens (figure 6.6) has all five.

**Remember:**

- Five models: **blocking**, **non-blocking**, **I/O multiplexing** (`select`, `poll`), signal-driven and asynchronous (the last two not in this course).
- Two phases: **wait for data**, then **copy data**. The models differ only in phase 1; phase 2 is always blocked in the read call.
- **All sockets are blocking by default.** Non-blocking returns `EWOULDBLOCK`/`EAGAIN` instead of sleeping; polling wastes CPU.
- Multiplexing = block in **`select`**, then `read` the ready descriptor. Two system calls, but one thread can watch many descriptors.
- Only asynchronous I/O (io_uring) avoids blocking in both phases.

### 6.3 The select function

**In one sentence:** `select` takes sets of descriptors and a timeout, sleeps until at least one descriptor is ready (or the time is up), and leaves **only the ready ones** marked in the sets.

**`select`:**

```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
```

Example: call `select` and tell the kernel to return only when

- any descriptor in **{1, 4, 5}** is ready for **reading**, or
- any descriptor in **{2, 7}** is ready for **writing**, or
- any descriptor in **{1, 4}** has an **exception condition** pending, or
- **10.2 seconds** have passed.

**What the arguments mean.** `select` tells the kernel **which descriptors to test** (reading, writing, exception) and **how long to wait**. It is **not restricted to sockets**: any file descriptor can be tested (which is why the keyboard and a socket can be watched together). The timeout is a structure with seconds and microseconds:

```c
struct timeval {
    long tv_sec;    /* seconds */
    long tv_usec;   /* microseconds */
};
```

**Three ways to use the timeout:**

| Timeout | Behaviour |
|---|---|
| **null pointer** | **wait forever**: return only when a descriptor is ready |
| a **positive time** | **wait up to that time**: return when ready, or when the time is up |
| **zero** | **don't wait at all**: check the descriptors and return immediately (a poll) |

**Descriptor sets.** An `fd_set` is a row of bits, one per descriptor number. Four macros handle it:

| Macro | Meaning |
|---|---|
| `FD_ZERO(&set)` | clear all bits |
| `FD_SET(fd, &set)` | turn on the bit for `fd` |
| `FD_CLR(fd, &set)` | turn off the bit for `fd` |
| `FD_ISSET(fd, &set)` | is the bit for `fd` on? |

```c
fd_set rset;
FD_ZERO(&rset);      /* initialize the set: all bits off */
FD_SET(1, &rset);    /* turn on bit for fd 1 */
FD_SET(4, &rset);    /* turn on bit for fd 4 */
FD_SET(5, &rset);    /* turn on bit for fd 5 */
```

```mermaid
block-beta
  columns 8
  d0["fd 0<br/>off"] d1["fd 1<br/>ON"] d2["fd 2<br/>off"] d3["fd 3<br/>off"] d4["fd 4<br/>ON"] d5["fd 5<br/>ON"] d6["fd 6<br/>off"] d7["fd 7<br/>off"]
```

The picture is `rset` after the code above: a row of light switches, one per descriptor number, with switches 1, 4 and 5 turned on.

**Important: `select` modifies the descriptor sets** you pass. On return, only the bits of **ready** descriptors are still on. So you must **rebuild the sets before every call**.

```mermaid
flowchart LR
  B["before select:<br/>bits 3, 4, 5, 6 on<br/>(watch these)"] --> SEL[select]
  SEL --> A["after select:<br/>only bits 4 and 6 on<br/>(these are ready)"]
  A -->|"rebuild the set<br/>before the next call"| B
```

> **Kid example.** You give the lunch lady a list: "tell me when Anna, Bo or Cleo's food is ready". She crosses out everyone whose food is **not** ready and gives the list back. Now only "Bo" is left on it. If you want to ask again, you must write a **new** list with all three names, because the old one has been crossed out.

**Return value.** It is the **total number of ready bits** across all sets. **`maxfdp1`** is the **number of descriptors to test**: the **highest descriptor number plus one** (the kernel checks descriptors 0, 1, …, maxfdp1 − 1).

**When is a descriptor ready?**

| Condition | Readable? | Writable? | Exception? |
|---|---|---|---|
| Data to read | ✓ | | |
| Read half of the connection closed (FIN received) | ✓ | | |
| New connection ready for a listening socket | ✓ | | |
| Space available for writing | | ✓ | |
| Write half of the connection closed | | ✓ | |
| Pending error | ✓ | ✓ | |
| TCP out-of-band data | | | ✓ |

"Ready" means **the next call will not block**, not that it will succeed. A socket whose peer has closed is **readable** (and `read` returns 0); a listening socket is **readable** when `accept` has a connection to hand over.

**Technical clarification.** The first bullet of the `select` overview, "TCP client establishes a connection with its server", is copied from the `connect` material by mistake; it has nothing to do with `select`.

**Remember:**

- `select(maxfdp1, readset, writeset, exceptset, timeout)`; works on **any** descriptor, not only sockets.
- Timeout: **NULL** = wait forever; **positive** = wait up to that time; **zero** = don't wait (poll).
- `fd_set` = row of bits; `FD_ZERO`, `FD_SET`, `FD_CLR`, `FD_ISSET`.
- `select` **overwrites** the sets: **rebuild them before every call**.
- `maxfdp1` = **highest descriptor + 1**; return value = number of ready bits.
- Readable: data, **FIN received**, **new connection on a listening socket**, pending error. "Ready" = will not block, not "will succeed".

### 6.4 The echo client with select

**In one sentence:** the new echo client waits in `select` on **both** the keyboard and the socket, so it reacts at once to whichever has something, including a server that has died.

**What makes the socket readable:**

- the peer sends **data**: the socket becomes readable and `read` returns **more than 0** (the number of bytes);
- the peer sends a **FIN** (the peer process ended): readable, and `read` returns **0 (EOF)**;
- the peer sends an **RST** (the peer host crashed and rebooted): readable, `read` returns **−1**, and `errno` holds the error code.

So after `select` says "ready", you must **call `read` and look at the result**.

```mermaid
flowchart LR
  S["select says:<br/>socket readable"] --> R[read]
  R -->|"> 0"| D["data: use it"]
  R -->|"= 0"| F["FIN: peer process ended"]
  R -->|"-1"| E["RST: peer host crashed<br/>and rebooted; check errno"]
```

**The rewritten `str_cli`:**

```c
void str_cli(FILE *fp, int sockfd) {
    int maxfdp1;  fd_set rset;  char sendline[MAXLINE], recvline[MAXLINE];
    FD_ZERO(&rset);
    for ( ; ; ) {
        FD_SET(fileno(fp), &rset);  FD_SET(sockfd, &rset);
        maxfdp1 = max(fileno(fp), sockfd) + 1;
        Select(maxfdp1, &rset, NULL, NULL, NULL);
        if (FD_ISSET(sockfd, &rset)) {          /* socket is readable */
            if (Readline(sockfd, recvline, MAXLINE) == 0)
                err_quit("str_cli: server terminated prematurely");
            Fputs(recvline, stdout);
        }
        if (FD_ISSET(fileno(fp), &rset)) {      /* input is readable */
            if (Fgets(sendline, MAXLINE, fp) == NULL) return;   /* all done */
            Writen(sockfd, sendline, strlen(sendline));
        }
    }
}
```

In words: forever, put **both** the keyboard descriptor (`fileno(fp)`) and the socket into the read set, compute the highest descriptor plus one, and **wait in `select`** until one of them is ready. If the **socket** is ready, read a line from the server (if it returns 0, the server has gone: say so and stop) and print it. If the **keyboard** is ready, read a line from the user (if there is none, we are done) and send it.

**What changed.** The **same four I/O functions** are used (`fgets`, `writen`, `readline`, `fputs`), but the **order of flow** changed: instead of being **driven by `fgets`**, the function is now **driven by `select`**. "We have added greatly to the robustness of our client with only a few additional lines of code." Now, if the server dies while the user is not typing, the client notices **at once**.

```mermaid
flowchart TB
  SEL["select on {keyboard, socket}"] --> Q1{socket ready?}
  Q1 -->|yes| R["readline from socket<br/>0 means server gone"] --> P[print it]
  Q1 -->|no| Q2
  P --> Q2{keyboard ready?}
  Q2 -->|yes| K["fgets a line<br/>none means done"] --> W[writen to socket]
  Q2 -->|no| SEL
  W --> SEL
```

> **Kid example.** Now you sit in the hallway between the front door and the kitchen, where you can hear **both** the doorbell and the phone. Whichever rings first, you answer it. If the phone rings to say "the parcel shop has closed", you know at once.

**Remember:**

- Socket readable: `read` > 0 = **data**; = 0 = **FIN** (peer process ended); −1 = **RST** (peer host crashed and rebooted), see `errno`.
- New `str_cli`: watch **keyboard + socket** with `select`; the flow is now **driven by `select`**, not by `fgets`.
- Same four I/O functions, a few more lines, much more **robust**: a dead server is noticed at once.

### 6.5 Batch input: stop-and-wait is slow

**In one sentence:** sending one line and waiting for its echo before sending the next means each line costs one round-trip time, so 2 000 lines over a 175 ms path take about 350 seconds no matter how fast the link is.

The original client works in **stop-and-wait** mode: it sends one line and **waits for the reply** before sending the next. Each line takes **one RTT** (round-trip time) plus the server's processing time (about 0 for an echo server). That is fine for a human typing, but how long does it take to echo a whole file?

**A worked example:**

- `ping` (average of 30 measurements) gives an **RTT of 175 ms** for 84-byte IP datagrams.
- Take the first **2 000 lines** of a **98 349-byte** file: on average **49 bytes per line**.
- Add the **IP header (20 bytes)** and **TCP header (20 bytes)**: the average segment is about **89 bytes**, nearly the size of the ping packets.
- Estimate: 2 000 × 0.175 s = **350 seconds**. The **measured** time was about **354 seconds**.

| Quantity | Value |
|---|---|
| Lines | 2 000 |
| Bytes per line | 98 349 ÷ 2 000 ≈ 49 |
| Segment size | 49 + 20 + 20 ≈ 89 bytes |
| RTT | 0.175 s |
| Predicted time | 2 000 × 0.175 = 350 s |
| Measured time | ≈ 354 s |

```math
T \approx \text{lines} \times \text{RTT} = 2000 \times 0.175\ \text{s} = 350\ \text{s}
```

So with stop-and-wait, **the speed depends only on the RTT**, not on the link's bandwidth. Nearly half of each packet is header, too.

Because there is a **delay** between sending a packet and its arrival, and because the pipe is **full-duplex** (both directions at once), the stop-and-wait client uses **only a small fraction of the pipe's capacity**. A batched client keeps sending while replies flow back.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  Note over C,S: stop-and-wait: one line per RTT
  C->>S: line 1
  S->>C: echo 1
  C->>S: line 2
  S->>C: echo 2
  Note over C,S: batch mode: keep the pipe full
  C->>S: line 3
  C->>S: line 4
  S->>C: echo 3
  C->>S: line 5
  S->>C: echo 4
```

> **Kid example.** Carrying 2 000 buckets of water from a well, but you must walk all the way back before filling the next one. A long, wide road doesn't help; only a shorter road would. Batch mode is a chain of people passing buckets: many buckets are on the road at once.

**Remember:**

- Stop-and-wait: **one line per RTT**.
- Example: RTT **175 ms**, **2 000** lines, about **49** bytes per line, about **89**-byte segments (20 IP + 20 TCP header) → predicted **350 s**, measured ≈ **354 s**.
- Speed depends on **RTT only**, not bandwidth; the full-duplex pipe is mostly empty. Batch mode keeps it full.

### 6.6 The batch-mode bug and shutdown

**In one sentence:** when the input file ends, the client must not close at once (echoes are still on the way); instead it calls **`shutdown(SHUT_WR)`** to send a FIN and keeps reading until the server's FIN arrives.

**The bug.** We can run the client in **batch mode** by redirecting input and output (for example `echoclient host < file1 > file2`). For an echo protocol, the output file should be **identical** to the input file. But **the output file is always smaller**!

Suppose the input has **9 lines**, and the last one is sent at time 8. The client **cannot close the connection after writing this request**, because there are **still other requests and replies in the pipe**. The cause is how the client handles **EOF on input**: `str_cli` returns to `main`, which ends the program. But in batch mode, **EOF on input does not mean we have finished reading from the socket**: there may still be requests on their way to the server, or replies on their way back.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  C->>S: lines 7, 8, 9
  Note over C: input file ends (EOF)<br/>old client exits and closes
  S--xC: echoes of 7, 8, 9 still on the way, lost
```

> **Kid example.** You throw 9 balls one after another to a friend, who throws each one back. After throwing the 9th ball you walk home at once. But balls 7, 8 and 9 are still flying back to you, and nobody is there to catch them.

**The solution:** send a **FIN** to the server, saying "we have finished sending data", but **leave the socket open for reading**. This is TCP's **half-close** ([section 3.2](#32-closing-a-connection-and-the-half-close)).

**`shutdown` vs `close`:**

- `close` **decrements the descriptor's reference count** and closes the socket only when the count reaches **0** (several descriptors can refer to one socket, for example after `fork`).
- `shutdown` starts TCP's normal termination **regardless of the reference count**.
- `close` terminates **both directions** (reading and writing). Since TCP is full-duplex, there are times we want to tell the other end "I have finished sending", even though it may still have data for us.

**The modes:**

```c
int shutdown(int sockfd, int howto);
```

| `howto` | Effect |
|---|---|
| `SHUT_RD` | the **read** half of the connection is closed |
| `SHUT_WR` | the **write** half is closed (a FIN is sent): the **half-close** |
| `SHUT_RDWR` | both halves are closed |

**The final `str_cli`:**

```c
void str_cli(FILE *fp, int sockfd) {
    int maxfdp1, stdineof;  fd_set rset;  char buf[MAXLINE];  int n;
    stdineof = 0;
    FD_ZERO(&rset);
    for ( ; ; ) {
        if (stdineof == 0) FD_SET(fileno(fp), &rset);
        FD_SET(sockfd, &rset);
        maxfdp1 = max(fileno(fp), sockfd) + 1;
        Select(maxfdp1, &rset, NULL, NULL, NULL);
        if (FD_ISSET(sockfd, &rset)) {                   /* socket is readable */
            if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
                if (stdineof == 1) return;               /* normal termination */
                else err_quit("str_cli: server terminated prematurely");
            }
            Write(fileno(stdout), buf, n);
        }
        if (FD_ISSET(fileno(fp), &rset)) {               /* input is readable */
            if ( (n = Read(fileno(fp), buf, MAXLINE)) == 0) {
                stdineof = 1;
                Shutdown(sockfd, SHUT_WR);               /* send FIN */
                FD_CLR(fileno(fp), &rset);
                continue;
            }
            Writen(sockfd, buf, n);
        }
    }
}
```

In words: a flag `stdineof` remembers whether the input has ended. While it has not, watch both input and socket. When input ends: set the flag, **send a FIN with `shutdown(SHUT_WR)`**, stop watching the input, and **keep going**. Now only the socket is watched; replies keep arriving and are printed. When the **socket** returns 0 **and** the input had already ended, everything is done: **normal termination**. If the socket returns 0 while input has not ended, the server died early. Note that this version reads and writes **buffers** with `Read`/`Write` instead of lines with `fgets`/`readline`.

```mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  C->>S: last lines of input
  Note over C: input EOF, shutdown(SHUT_WR)
  C->>S: FIN
  S->>C: remaining echoes still arrive
  Note over S: read returns 0, server closes
  S->>C: FIN
  Note over C: read returns 0 and stdineof = 1, normal end
```

> **Kid example.** After the 9th ball you shout "that was my last ball!" (shutdown sends a FIN), but you **stay** and keep catching. Only when your friend shouts "and that was my last one!" (the server's FIN) do you go home.

**Summary.** The default is **blocking I/O**, also the most used. The most used function for I/O multiplexing is **`select`**: we tell it which descriptors interest us (reading, writing, exceptions), the maximum time to wait, and the maximum descriptor number plus one. Running the echo client in batch mode with `select` showed that even when the user's input has ended, **data can still be in the pipe** to or from the server; handling this needs **`shutdown`** and TCP's **half-close**. "The **dangers of mixing stdio buffering** (as well as our own readline buffering) **with `select`**" led to versions of the echo client and server that **operate on buffers instead of lines**.

**Additional context.** Why is mixing stdio with `select` dangerous? `fgets` may read **several lines at once** from the kernel into stdio's own hidden buffer and give you only the first. `select` only looks at the **kernel's** descriptor, which is now empty, so it does not report the lines still waiting inside stdio, and they may be ignored until more input arrives. That is why the final version uses `Read` on `fileno(fp)` directly.

**Remember:**

- Batch-mode bug: output file **smaller** than input, because the client exits on input EOF while echoes are still in the pipe.
- Fix: **`shutdown(sockfd, SHUT_WR)`** sends a FIN but keeps reading (**half-close**).
- `close`: decrements the **reference count**, closes **both** directions. `shutdown`: ignores the reference count, can close **one** direction.
- Modes: `SHUT_RD`, `SHUT_WR` (half-close), `SHUT_RDWR`.
- Final client: flag `stdineof`; socket returns 0 with `stdineof = 1` → normal end; with `stdineof = 0` → server died early.
- Don't mix **stdio buffering** with `select`; work on **buffers**, not lines.

### 6.7 An echo server built on select

**In one sentence:** one server process can serve many clients without `fork` by keeping a list of connected descriptors and using `select` to handle whichever socket (listening or connected) is ready.

The lecture builds a server that serves **many clients with one process and no `fork`**.

**Before any client.** The server has **one listening descriptor** and keeps **one read set**. It was started in the foreground, so descriptors **0, 1 and 2** are standard input, output and error; the first free descriptor, **3**, is the listening socket. An integer array `client[]` will hold the connected sockets; all entries start at **−1** (empty). The first argument to `select` will be **4** (highest descriptor 3, plus one).

**First client.** It connects: the **listening descriptor becomes readable**, and the server calls `accept`. The new connected descriptor is **4**. The server must **remember it in `client[]`** and **add it to the descriptor set**.

**Second client.** Later a second client connects and gets descriptor **5**, which must be remembered too.

**First client leaves.** It sends a **FIN**, which makes descriptor 4 **readable**. The server reads, `read` returns **0**, so it **closes** the socket and updates its data: `client[0]` is set back to −1 and bit 4 in the set is turned off.

```mermaid
flowchart LR
  A["start<br/>listen fd 3<br/>client[] all -1<br/>watch {3}"] -->|client 1 connects| B["accept gives fd 4<br/>client[0] = 4<br/>watch {3, 4}"]
  B -->|client 2 connects| C["accept gives fd 5<br/>client[1] = 5<br/>watch {3, 4, 5}"]
  C -->|client 1 sends FIN| D["read on 4 returns 0<br/>close 4, client[0] = -1<br/>watch {3, 5}"]
```

**Bookkeeping.** As clients arrive, the server stores each connected descriptor in the **first free entry** of `client[]` (the first −1) and adds it to the read set. **`maxi`** is the highest index of `client[]` in use; **`maxfd` + 1** is the current first argument to `select`. The number of clients is limited by the **smaller of FD_SETSIZE (1024)** and the maximum number of descriptors the process may open.

**The code:**

```c
int main(int argc, char **argv) {
    int i, maxi, maxfd, listenfd, connfd, sockfd;
    int nready, client[FD_SETSIZE];
    ssize_t n;  fd_set rset, allset;  char buf[MAXLINE];
    socklen_t clilen;  struct sockaddr_in cliaddr, servaddr;

    listenfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);
    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
    Listen(listenfd, LISTENQ);

    maxfd = listenfd;                                  /* initialize */
    maxi = -1;                                         /* index into client[] array */
    for (i = 0; i < FD_SETSIZE; i++) client[i] = -1;   /* -1 indicates available entry */
    FD_ZERO(&allset);
    FD_SET(listenfd, &allset);

    for ( ; ; ) {
        rset = allset;                                 /* structure assignment */
        nready = Select(maxfd + 1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(listenfd, &rset)) {               /* new client connection */
            clilen = sizeof(cliaddr);
            connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
            for (i = 0; i < FD_SETSIZE; i++)
                if (client[i] < 0) { client[i] = connfd; break; }   /* save descriptor */
            if (i == FD_SETSIZE) err_quit("too many clients");
            FD_SET(connfd, &allset);                   /* add new descriptor to set */
            if (connfd > maxfd) maxfd = connfd;        /* for select */
            if (i > maxi) maxi = i;                    /* max index in client[] array */
            if (--nready <= 0) continue;               /* no more readable descriptors */
        }

        for (i = 0; i <= maxi; i++) {                  /* check all clients for data */
            if ( (sockfd = client[i]) < 0) continue;
            if (FD_ISSET(sockfd, &rset)) {
                if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {          /* client closed */
                    Close(sockfd);  FD_CLR(sockfd, &allset);  client[i] = -1;
                } else
                    Writen(sockfd, buf, n);
                if (--nready <= 0) break;              /* no more readable descriptors */
            }
        }
    }
}
```

In plain words:

1. Set up the listening socket as before.
2. Keep **two sets**: **`allset`**, the master list of everything we care about, and **`rset`**, a working copy that `select` is allowed to overwrite. At the top of each loop, `rset = allset` restores the full list in one step.
3. Wait in `select` until something is ready. `nready` = how many descriptors are ready.
4. If the **listening socket** is ready: `accept` the new client, put its descriptor in the first free place of `client[]` (stop with "too many clients" if there is none), add it to `allset`, and update `maxfd` and `maxi`. If that was the only ready descriptor, go back to `select`.
5. Then look through `client[]`. For each client whose descriptor is ready: `read`. If it returns **0**, the client has closed: **close** it, remove it from `allset`, and mark the slot free. Otherwise **echo** the bytes back with `Writen`. Stop early once all `nready` ready descriptors have been handled.

```mermaid
flowchart TB
  T["rset = allset<br/>(fresh copy of the master list)"] --> S["nready = select(...)"]
  S --> L{listening socket<br/>ready?}
  L -->|yes| A["accept, store in client[],<br/>add to allset"] --> C
  L -->|no| C["for each client[i]"]
  C --> R{ready?}
  R -->|"read = 0"| X["close, FD_CLR,<br/>client[i] = -1"] --> N{"more clients<br/>to check?"}
  R -->|"read > 0"| E["Writen the bytes back"] --> N
  R -->|"not ready"| N
  N -->|yes| C
  N -->|"no (or nready used up)"| T
```

> **Kid example.** One waiter for many tables. The waiter keeps a list of occupied tables (`allset`). Each round, the waiter looks at which tables have a hand up (`select`), takes new guests to a free table (accept), brings food to tables that asked (echo), and clears tables whose guests have left (close).

Three things to notice. The program handles "too many clients" by **dying**, which is fine for teaching but not for a real server (it should refuse the one client). `maxfd` is **never lowered** when the highest descriptor closes, so the kernel keeps scanning more than needed. And three **linear scans** happen per event (the kernel over the descriptors, the program over `client[]`, and over the set), harmless at 10 clients but a problem at 10 000 ([section 7.4](#74-c10k-and-c10m)).

**Remember:**

- One process, no `fork`: listening fd **3**, `client[]` array (−1 = free), `select` on all.
- Two sets: **`allset`** (master) and **`rset`** (working copy, `rset = allset` each loop).
- Listening socket readable → **accept**, store in first free slot. Client readable → `read`: 0 → **close** and free the slot; > 0 → **echo**.
- `maxfd` + 1 is the first argument; `maxi` = highest used index; `nready` lets the loop stop early.
- Limit: smaller of **FD_SETSIZE (1024)** and the process's descriptor limit.
- Weak points: dies on "too many clients", `maxfd` never lowered, three **linear scans** per event.

## 7. Robust and scalable servers

**What this chapter is about:** how a single bad (or slow) client can freeze a server and how to prevent it, tools for finding bugs, the main ways to build a server that serves many clients, how servers grew from a thousand to ten million connections, and libraries that sit on top of sockets.

### 7.1 Denial of service and robust servers

**In one sentence:** a server must **never block waiting for one client**, so it reads only what is available and keeps a separate buffer and state for each connection.

**The one-byte attack.** What happens if a **malicious client connects, sends one byte (not a newline), and goes to sleep**? The server calls `read` (inside a read-a-line function), gets the single byte, and then **blocks in the next `read`**, waiting for the rest of the line from this client. The server is now **blocked by this one client** and serves **nobody else**, neither new connections nor existing clients' data, until the malicious client sends a newline or terminates.

```mermaid
sequenceDiagram
  participant M as Bad client
  participant S as Server
  participant G as Good client
  M->>S: one byte, no newline
  Note over S: readline gets 1 byte,<br/>then blocks waiting for newline
  G->>S: data
  Note over G: nobody answers
  Note over S: stuck until the bad client<br/>sends a newline or leaves
```

**The rule:** **a server must never block in a function call related to a single client**, because it might deny service to all other clients. Solutions:

- **Non-blocking I/O**;
- **Each client served by a separate thread** (or process);
- **A timeout** on I/O operations.

**Attack 1: too many connections.** A `select` server is **limited to FD_SETSIZE − 4** connections (typically **1020**): 3 descriptors for standard input/output/error and 1 for the listening socket. "Defenses?" Limit what one client may use (connections per source address, idle timeouts) and use `epoll`/`kqueue`, which have no such ceiling.

**Attack 2: partial lines.** A client gets the server's attention by sending a **partial line**; the server **blocks until the line is completed**. The timeline: client 1 connects; the user types "Hello world\n" but the client sends only "Hello world"; `select` detects input and the server calls `readlineb`; the server **blocks waiting for "\n" from client 1**, while client 2 waits.

**A flaky client** does the same **without any bad intent**:

```c
while (Fgets(buf, MAXLINE, stdin) != NULL) {
    Rio_writen(clientfd, buf, strlen(buf)-1);   /* send the line without its newline */
    Fgets(buf, MAXLINE, stdin);                 /* read & ignore line */
    Rio_writen(clientfd, "\n", 1);              /* newline only after the user types again */
    Rio_readlineb(&rio, buf, MAXLINE);
    Fputs(buf, stdout);
}
```

It sends everything **up to the newline**, and sends the newline only when the user types **another** line. Meanwhile, the server blocks. The server **cannot tell hostility from hesitation**; the effect is the same.

**A robust server:**

- **Break reading a line into multiple partial reads**;
- every time a connection is selected, **read as much as is available** (never more);
- **Construct the line in a separate buffer for each connection**;
- this requires the plain Unix **`read(fd, buf, maxn)`**, which reads **as many bytes as are available, up to `maxn`**.

> **Kid example.** A teacher listening to many children. If one child says "I want to tell you something…" and then goes quiet, a bad teacher stands waiting forever. A good teacher writes down what the child said so far in that child's notebook, helps others, and comes back when the child speaks again.

**Just to complicate things.** If a client sends **x bytes in one `write`**, it is **not guaranteed** that the server receives all x bytes in **one `read`**. For example: the client writes "Hello world\n"; `select` wakes the server, which calls `read`; `read` returns "Hell"; a later `read` returns "o w"; another returns "orld\n". **Solution:** keep a **buffer for each client** and **process it only when a whole message has arrived**. What counts as "a message", and how you know it is complete, depends on the application (for example a newline, or a length prefix).

```mermaid
flowchart LR
  R1["read: 'Hell'"] --> B["client 1's buffer:<br/>'Hell' → 'Hello w' → 'Hello world\n'"]
  R2["read: 'o w'"] --> B
  R3["read: 'orld\n'"] --> B
  B -->|"newline seen:<br/>whole message"| P[process the line]
```

**Conceptual model:** keep a **state machine for each connection**.

| | First version | Second (robust) version |
|---|---|---|
| State of a connection | just the identity of `connfd` | `connfd` + **the partial line** + **count of characters** |
| What `select` decides | which state machine to update | which state machine to update |
| Work per event | process the **entire line** | process **as much of the line as is available** |

**Design issue:** the **granularity** of the state machine must be chosen so that **the server never blocks**.

```mermaid
stateDiagram-v2
  [*] --> Empty
  Empty --> Partial: some bytes arrive, no newline yet
  Partial --> Partial: more bytes, still no newline
  Partial --> Complete: newline arrives
  Empty --> Complete: whole line arrives at once
  Complete --> Empty: echo the line, clear buffer
  Partial --> [*]: client closes or times out
```

**Remember:**

- **One-byte attack**: send one byte without newline → the server blocks in `readline` and serves nobody.
- Rule: **never block on a single client**. Fixes: non-blocking I/O, thread/process per client, timeouts.
- `select` server limit: **FD_SETSIZE − 4** (typically **1020**).
- A **flaky** client causes the same harm as a malicious one.
- Robust server: read **only what is available** with plain `read(fd, buf, maxn)`, keep a **buffer per connection**, process only **complete messages**.
- One `write` of x bytes may arrive in **several reads**.
- Model: a **state machine per connection** (`connfd` + partial line + count), fine-grained enough to **never block**.

### 7.2 Debugging and testing tools

**In one sentence:** program defensively, use **GDB** to step through and inspect a program, and use **telnet** or **netcat** to talk to a text-based server by hand.

**Debugging advice.** Tools make programming simpler: **GDB** helps with debugging; **DDD** is a graphical version of GDB. And:

- **Program defensively**: check error returns and buffer sizes;
- **Build programs in a modular way**;
- **Print sensible error messages**;
- you may want to do **better than the built-in wrapper functions** (which just stop the program).

**Using GDB:**

- **Prepare:** compile with **`-g`** (keep the full symbol table) and **without optimisation** (`-O`, `-O2` …), otherwise the debugger shows confusing lines.
- **Run directly:** `gdb progname`, then `run args`; shell-style redirection works (`run < infile > /dev/null`).
- **Post-mortem:** `gdb progname core` examines the program's state at the moment it crashed.
- **Help:** `help`, `help <topic>`, `help <command>`.
- **Breakpoints:** the program runs until it hits one; `cont` continues. `break foo.c:4` (line 4 of foo.c), `break 16` (line 16 of the current file), `break main` (entry to `main`). CTRL-C (SIGINT) stops a hanging program (sometimes).
- **Stepping:** `step N` runs N source lines (default 1); `next` is like `step` but treats a function call as one line. Hint: avoid "mega-expressions" like `foo(bar(tmp = baz(), tmp2 = baz2()))`, which are hard to step through.
- **Looking at state:** `backtrace` (`bt`) shows the stack of function calls (move with `up` and `down`); `bt full` also prints local variables; `print EXP` prints a value; `x/<count><format><size> ADDR` examines memory. Formats: o (octal), x (hex), d (decimal), u (unsigned decimal), t (binary), f (float), a (address), i (instruction), c (char), s (string). Sizes: b (byte), h (halfword), w (word), g (giant, 8 bytes).
- **More:** the official GDB homepage and a GDB primer (links in the lecture).

```mermaid
flowchart LR
  A["compile with -g,<br/>no -O"] --> B["gdb progname<br/>run args"]
  B --> C["break main<br/>break foo.c:4"]
  C --> D["step / next<br/>through the code"]
  D --> E["print EXP, bt,<br/>x/ ADDR"]
  B -.->|"after a crash"| F["gdb progname core"]
```

> **Kid example.** GDB is like watching a film of your program in slow motion. A breakpoint is the pause button at a chosen scene. `step` moves forward one picture at a time. `print` is zooming in to read what is written on a sign in the picture. `backtrace` shows the list of scenes that led to this moment.

Also: also compile with **`-Wall`** (all warnings); and when a socket call fails, print **`errno`** and the **descriptor number** (many bugs are "wrong descriptor").

**Testing servers with telnet.** `telnet <host> <portnumber>` opens a connection to a server on that host and port, and lets you type to it. It is invaluable for testing servers that send **ASCII text**: the echo server, web servers, mail servers. It separates "is my server wrong?" from "is my client wrong?". Also, **`nc` (netcat)** does the same job on systems without telnet, and that neither works for binary or encrypted protocols (use `openssl s_client` for TLS).

**Remember:**

- Program defensively, modularly, with sensible error messages.
- GDB: compile with **`-g`** and **no optimisation**; `run`, `break`, `cont`, `step`, `next`, `bt`, `print`, `x`; `gdb progname core` after a crash. DDD = graphical GDB.
- Compile with **`-Wall`**; on errors print `errno` and the descriptor number.
- **telnet host port** (or **nc**) lets you type to a **text** server by hand; not for binary or TLS (`openssl s_client`).

### 7.3 Kinds of servers

**In one sentence:** an **iterative** server makes everyone wait for one slow client; a **concurrent** server overlaps clients using **processes**, **threads** or **I/O multiplexing** (event-based).

The lecture compares server designs.

**Iterative servers** process **one request at a time** (the timeline of [section 2.5](#25-iterative-servers)).

**Their flaw.** Client 1 connects and the server blocks waiting for its data. The **user of client 1 goes out to lunch**, so client 1 blocks waiting for the user to type. **Client 2 blocks** waiting to complete its connection request **"until after lunch!"**. **Solution: concurrent servers**, which use **multiple concurrent flows** to serve multiple clients at the same time.

**Concurrent servers** handle multiple requests **concurrently**: client 2's conversation starts at once, overlapping client 1's, and the server never blocks waiting for one user. Note that concurrency does **not** make the total work smaller; it changes **who waits**.

**Mechanisms for concurrent flows:**

| Mechanism | Who interleaves the flows | Memory | Notes |
|---|---|---|---|
| **1. Processes** (`fork`) | the **kernel**, automatically | each flow has its **own private address space** | strong isolation; expensive to create and switch |
| **2. I/O multiplexing** (`select`) | the **programmer**, manually | all flows **share one address space** | popular for high-performance servers |
| **3. Threads** | the kernel | shared address space | cheaper than processes, no isolation |

```mermaid
flowchart TB
  C[Concurrent server] --> P["1. Processes (fork)<br/>one copy of the server per client<br/>kernel switches, private memory"]
  C --> M["2. I/O multiplexing (select)<br/>one process, event loop<br/>programmer switches, shared memory"]
  C --> T["3. Threads<br/>one thread per client<br/>kernel switches, shared memory"]
```

> **Kid example.** A restaurant can hire one waiter per table (processes or threads: simple but costly), or have one very organised waiter who keeps a list and serves whoever raises a hand (event-based: cheap but the waiter must never stand frozen at one table).

**Event-based servers with I/O multiplexing:** keep a **pool of connected descriptors**, and repeat forever: use `select` to block until (a) a **new connection request** arrives on the listening descriptor, or (b) **new data** arrives on an existing connected descriptor. If (a), **add** the new connection to the pool. If (b), **read** the available data; **close** the connection on EOF and remove it from the pool.

**Pros and cons of event-based designs:**

| + | − |
|---|---|
| **one logical control flow** | **significantly more complex to code** than process- or thread-based designs |
| can **single-step with a debugger** | can be **vulnerable to denial-of-service** attacks (if any handler blocks) |
| **no process or thread control overhead** | `select` does not scale |
| design of choice for **high-performance web servers and search engines** | |

The lecture adds: modern equivalents are **epoll (Linux)** and **kqueue (BSD)**; "select does not scale. See C10K."

**Remember:**

- Iterative server: client 2 waits "**until after lunch**" of client 1.
- Concurrent server: **overlaps** clients; total work is the same, but **who waits** changes.
- Three mechanisms: **processes** (kernel interleaves, private memory, isolation, costly), **I/O multiplexing** (programmer interleaves, shared memory), **threads** (kernel interleaves, shared memory, cheaper, no isolation).
- Event-based: + one control flow, debuggable, no process/thread overhead, used by fast web servers; − harder to code, vulnerable to DoS if a handler blocks, `select` doesn't scale.

### 7.4 C10K and C10M

**In one sentence:** to serve ten thousand connections (C10K, 1999) servers moved from `select` to **epoll/kqueue/IOCP**; to serve ten million (C10M, 2013) they reduce or bypass the kernel with **io_uring**, **XDP** and **DPDK**.

**C10K: why `select` ran out.** In **1999, Dan Kegel** asked: can **one server hold ten thousand concurrent connections**?

- `select` has a **hard descriptor limit** (FD_SETSIZE, typically 1024) and **rescans the whole set on every call**.
- Its cost grows with the number of **connections watched**, not the number **active**.
- **Thread-per-connection fails differently**: each thread needs **stack memory**, and the system spends time **context switching**.
- **The answers:** **epoll** (Linux), **kqueue** (BSD), **IOCP** (Windows), and the **nginx** web server built on them. With these, you register interest once and the kernel tells you only what changed.

> **Kid example.** `select` is a teacher who, every minute, walks past **every** desk in a hall of 10 000 students to see who has a hand up, even if only two do. `epoll` is a teacher with a buzzer system: students press a button, and the teacher's screen shows **only** the students who pressed it.

**C10M: the kernel becomes the bottleneck.** In **2013, Robert Graham**: at **ten million** connections "the kernel is the problem, not the solution". **Per-packet system calls, interrupts, copies and lost cache locality** dominate. Responses:

| Technique | Idea |
|---|---|
| **io_uring** | keep the kernel, but **fewer system calls**: shared ring buffers, batching, zero-copy send |
| **XDP / AF_XDP** | run **verified eBPF** programs **in the network driver**, before the normal network stack |
| **DPDK** and **user-space TCP stacks** (mTCP, F-Stack, TAS, Junction) | **leave the kernel entirely**: the program drives the network card itself |

**What kernel bypass costs:**

- **DPDK burns a whole CPU core at 100 %**, constantly **polling** whether traffic arrives or not.
- It needs **huge pages, core pinning, NIC-specific drivers** and real operational expertise.
- Userspace gets **direct hardware access**; **IOMMU and VFIO reduce but do not remove** the risk.
- **For most workloads, the modern kernel** (eBPF, XDP, io_uring, multi-queue network cards) **is enough**.
- Source: Awamoto and Honda, "Opening Up Kernel-Bypass TCP Stacks", USENIX ATC 2025.

```mermaid
flowchart LR
  A["select<br/>~1 000 connections<br/>scans everything"] --> B["epoll / kqueue / IOCP<br/>C10K: 10 000+<br/>only ready events"]
  B --> C["io_uring, XDP<br/>fewer syscalls,<br/>work in the driver"]
  C --> D["DPDK, user-space TCP<br/>C10M: bypass the kernel<br/>costs a polling core"]
```

**Technical clarification.** The lecture says "The SDN lecture taught them [XDP and eBPF] from the switch side". The SDN lecture in this course does not mention XDP or eBPF (see [06-SDN.md](06-SDN.md)); treat that sentence as a leftover from another version of the course.

**Remember:**

- **C10K** (Dan Kegel, **1999**): `select` has a limit (**1024**) and rescans everything; threads cost stack memory and context switches. Answer: **epoll, kqueue, IOCP, nginx**.
- **C10M** (Robert Graham, **2013**): "the kernel is the problem". Answers: **io_uring** (fewer syscalls), **XDP/AF_XDP** (eBPF in the driver), **DPDK** / user-space TCP (bypass the kernel).
- DPDK costs a **CPU core at 100 %**, huge pages, core pinning, special drivers, security risk. Usually the **modern kernel is enough**.

### 7.5 Above the socket: messaging libraries

**In one sentence:** libraries such as **ZeroMQ** send **whole messages** and handle reconnection for you, but you no longer see the connection, its backpressure or its failures.

**Message-oriented middleware.** "We spent nine slides on `readn`, `writen` and `readline`, because TCP is a byte stream." **ZeroMQ** delivers **whole messages, atomically**: the **framing problem disappears**.

- It has a **socket-like API**, but with **patterns instead of connections**: **REQ/REP** (request-reply), **PUB/SUB** (publish-subscribe), **PUSH/PULL** (pipelines). One ZeroMQ socket may talk to many peers.
- It is **brokerless** (no server in the middle). **Automatic reconnection and queueing** come for free.
- The price: "You stop writing plumbing, **and stop seeing the connection**."

```mermaid
flowchart LR
  q1["REQ / REP<br/>asker"] -->|request| q2[answerer]
  q2 -->|reply| q1
  p1["PUB / SUB<br/>publisher"] --> s1[subscriber 1]
  p1 --> s2[subscriber 2]
  u1["PUSH / PULL<br/>step 1"] --> u2[step 2] --> u3[step 3]
```

> **Kid example.** Plain TCP is like a water hose: you must decide yourself where one glass of water ends. ZeroMQ is like a parcel service: it always delivers **whole parcels**, and if the road is closed it waits and tries again for you. But you no longer see the truck or the road, so when something is slow, it is harder to know why.

**The messaging landscape:**

| System | Aim |
|---|---|
| **ZeroMQ** (2007) → **nanomsg** (2012) → **NNG** | one brokerless, socket-like family |
| **NATS** | cloud-native publish-subscribe and request-reply |
| **Aeron** | when **latency dominates**, as in finance |
| **Zenoh** | robotics and edge: data in motion and at rest |
| **Kafka** | a **durable, replayable log**: "a different thing entirely, **not a transport**" |

"Each hides connection visibility, backpressure and failure modes. **Know what you traded.**"

**Remember:**

- **ZeroMQ** delivers **whole messages** atomically: no framing problem.
- Patterns: **REQ/REP**, **PUB/SUB**, **PUSH/PULL**; **brokerless**; automatic reconnection and queueing.
- Price: you **stop seeing the connection** (backpressure, failures).
- Family: ZeroMQ (2007) → nanomsg (2012) → NNG; also NATS, Aeron (latency), Zenoh (robotics/edge), Kafka (a durable log, **not a transport**).

## 8. Summary and review

**What this chapter is about:** the whole block on one page, checklists to test yourself, old exam questions with answers, and quiz topics that the lectures only touch on.

### 8.1 Summary

**In one sentence:** a socket is a file descriptor for the network; know the client and server call sequences, the handshake, TIME_WAIT, ports, byte order, and why servers use `select` (today epoll/kqueue) instead of blocking on one client.

**Concepts:**

- **Socket = file descriptor** → one model for files and network I/O.
- **TCP handshake and termination** → reliable set-up and tear-down.
- **TIME_WAIT** → prevents reuse while old packets may still arrive.
- **Ports** → identify services and let many connections share a host.
- **Blocking vs multiplexing** → decides responsiveness and scalability.
- **MSL** (maximum segment lifetime) → governs TIME_WAIT; wrong values break correctness.
- **Byte order conversion** → interoperability between different hosts.

**Practice:**

- **Client:** `socket` → `connect`. **Server:** `socket` → `bind` → `listen` → `accept` (binds the service to a port).
- **Daytime server** → minimal working TCP example.
- **Echo client/server evolution:** blocking version (easy but fragile) → robust I/O wrappers (`Readline`, `Writen`, full transfers) → `select`-based server (many clients at once) → `shutdown` (signal EOF while still reading).
- **Error-handling wrappers** → standard return-code checks.
- Today: **epoll / kqueue** rather than `select`.

```mermaid
flowchart LR
  D["Daytime server<br/>one message,<br/>iterative"] --> E["Echo server<br/>a conversation,<br/>blocking hurts"]
  E --> S["Echo with select<br/>many clients, one thread,<br/>needs shutdown"]
  S --> R["Robust server<br/>partial reads,<br/>per-client buffers"]
  R --> M["Modern<br/>epoll / kqueue"]
```

**Pitfalls:**

| Pitfall | Consequence |
|---|---|
| assuming TCP preserves message boundaries | must loop on `read` |
| blocking I/O in servers | hangs with multiple clients |
| misconfigured socket options (e.g. `SO_LINGER`, wrong MSL) | unstable or stuck connections |
| ignoring TIME_WAIT | mysterious "address already in use" errors |
| forgetting byte-order conversion | corrupted values between architectures |
| not freeing resources or closing descriptors | leaks and resource exhaustion |

**Best practices:** always **loop on `read`**; use **robust I/O wrappers**; **check return codes** consistently; **prefer multiplexing**; configure socket options carefully; **respect TIME_WAIT**; **convert byte order**; **free resources promptly**; **prefer epoll or kqueue over `select`**; **resolve with `getaddrinfo()` and store in `sockaddr_storage`** (protocol-independent by default).

> **Kid example.** The whole block as one story about a shop. The shop gets an address and opens its door (`socket`, `bind`, `listen`). Customers walk in after a polite three-step greeting (handshake). The shopkeeper hands each customer their own ticket (`accept`). Because the post cuts letters into pieces, the shopkeeper keeps a separate notebook for each customer (buffers). One clever shopkeeper watches all customers at once and helps whoever raises a hand (`select`, epoll). At closing time everyone says goodbye in four steps, and the one who said goodbye first waits a moment at the door (TIME_WAIT).

**Technical clarification.** Two summary lines are imprecise. (1) The summary says TIME_WAIT "prevents reuse of **ports**". More exactly, it prevents reuse of the same **four-tuple** (both addresses and both ports), as [section 3.4](#34-time_wait) explains; a server can still accept new connections on the same port during TIME_WAIT. (2) The summary lists "MSL" next to socket options such as `SO_LINGER`. MSL is **not** a per-socket option a program can configure; it is a fixed value inside the TCP implementation (Linux, for example, uses a fixed 60-second TIME_WAIT).

**Remember:**

- Concepts: socket = fd; handshake and termination; TIME_WAIT; ports; blocking vs multiplexing; MSL; byte order.
- Evolution: daytime → echo (blocking) → robust wrappers → `select` server → `shutdown` → epoll/kqueue.
- Six pitfalls and their consequences (table above); best practices are the opposite of each pitfall.
- TIME_WAIT blocks the same **four-tuple**, not the whole port; MSL is **not** a socket option.

### 8.2 Self-assessment checklists †

**In one sentence:** these three lists say what you should be able to do from memory after each part.

The three hidden self-assessment slides list what you should be able to do without looking.

† **After part I:**

- explain what a socket is in terms of **endpoints and file descriptors**;
- name the **four values** that identify a TCP connection;
- write the **client** sequence (socket, connect, read in a loop, close) and the **server** sequence (socket, bind, listen, accept, write, close, loop);
- say what the **three-way handshake** achieves and why it takes **three** segments;
- say what a **`read` returning zero** means.

† **After part II:**

- trace a connection from **CLOSED to ESTABLISHED** and through **both termination paths**, naming the **active closer**;
- state the **two independent reasons** for TIME_WAIT and why it lasts **2 × MSL**;
- explain why a **listening socket and a connected socket can share a local port**;
- say why a `read` may return **fewer bytes than requested**, and write the loop that handles it.

† **After part III:**

- write `socket`, `connect`, `bind`, `listen`, `accept` and `close` from memory, with the **state change** each causes;
- explain the difference between a **listening** and a **connected** descriptor;
- write a **`select` loop** with a master set and a working set;
- say why **"readable" can mean end-of-file**, and what happens if you don't check;
- explain the **batch-mode bug** and why **`shutdown`** fixes it.

**Remember:**

- If you can do every item in these three lists without looking, you know this block.

### 8.3 Old exam questions on sockets

**In one sentence:** old exams ask true/false tricks (a lock or a timer from sockets), "find the bug in a `select` server", TIME_WAIT questions, and a self-stabilizing overlay built from sockets.

**Additional context.** These come from the course's old exams. Where no official answer exists, the answer is built from the lectures.

**October 2021, 1.m (true or false):** "Using the socket API (`socket`, `bind`, `listen`, `accept`, `connect`, `select`, `close`, `shutdown`) it is possible to implement a **lock** (mutex)." Official answer: **True**. For example, only one socket at a time can **`bind`** a given local address and port: a process "takes the lock" by binding an agreed port; others get "address in use" and retry; **`close`** releases it. (Another way: a lock server that accepts one client at a time.)

> **Kid example.** A lock from sockets is like a single toilet with one key on a hook. Whoever takes the key (binds the port) may go in; others find the hook empty ("address in use") and wait. Hanging the key back (close) lets the next person go.

**October 2021, 1.n (true or false):** "Using the socket API it is possible to implement a **timer**." Official answer: **True**. `select` has a **timeout**: called with no descriptors (or none that become ready) and a timeout of, say, 10.2 seconds, it returns after that time.

**2012–2015, "find the bug in a `select` server"** (several exams). A server calls `select`; if the listening socket is ready it calls `accept` and then immediately `recv` to read a 1-byte client ID. **The flaw:** `select` only said that a **new connection** can be accepted, not that the new client has **sent anything**. The new socket is blocking, so **`recv` blocks**; a slow or malicious client that sends nothing **freezes the whole server** (the one-byte attack of [section 7.1](#71-denial-of-service-and-robust-servers)). **Fix:** add the new socket to the read set and read the ID only when `select` reports it readable. A variant loops `recv` until a 24-byte message is complete: `select` only promised *some* data, so the loop blocks and starves other sockets. **Fix:** a per-socket buffer, and go back to `select`.

```mermaid
flowchart LR
  A["select: listening<br/>socket ready"] --> B[accept]
  B --> X["WRONG: recv at once<br/>blocks if the client<br/>sends nothing"]
  B --> OK["RIGHT: add the new socket<br/>to the read set, recv only<br/>when select says readable"]
```

**2013-10 / 2014-08, Q9a and Q9b.** "Someone suggests shortening TIME_WAIT. What could happen?" Both jobs of TIME_WAIT are put at risk: a **lost final ACK** could no longer be repeated (the peer gets an RST and an error), and **old duplicate segments** could be accepted by a **new incarnation** with the same four-tuple. "How many simultaneous connections are possible, and on what does it depend?" A connection is identified by the **four-tuple**, so many can exist at once; in practice the limit is the **16-bit port space** (for one client towards one server address and port) and the operating system's **resources** (descriptors, memory).

**January 2025, question 2a (3 points):** build and maintain a **complete overlay** (every node in a known set S connected to every other) in a **self-stabilizing** way, using a listening descriptor, connection descriptors, I/O multiplexing, `select`, `connect` and `accept`. The model answer: each node has one **listening** socket and a table `conn[j]` of connection descriptors. To avoid duplicates, the **smaller ID calls `connect`**, the larger ID calls **`accept`**. A **do-forever loop around `select` with a timeout** (never a NULL timeout, or repairs could stop) watches all descriptors; each round it closes `conn[j]` if `read` returned 0 or an error, or if nothing was heard for T seconds; it **re-`connect`s** every empty slot; it accepts new connections only from valid, smaller IDs; and it sends **heartbeats**. Any corrupted entry is either a dead descriptor (closed within T) or a wrong one (closed at once), and every missing connection is rebuilt in the next round: **convergence**; once all pairs are connected and heartbeats flow, nothing changes: **closure**. Traps: a blocking `read` or `connect` on one peer, both sides calling `connect`, and forgetting that TCP is a byte stream (messages need framing).

```mermaid
flowchart TB
  L["do forever"] --> S["select on listening + all conn[j],<br/>WITH a timeout"]
  S --> C1["close conn[j] if read gave 0 or error,<br/>or nothing heard for T seconds"]
  C1 --> C2["for each empty conn[j] with j > my ID:<br/>connect (smaller ID connects)"]
  C2 --> C3["accept only from valid, smaller IDs"]
  C3 --> C4["send heartbeats"]
  C4 --> L
```

**2012-10 Q3:** the client/server call diagram of [section 1.6](#16-the-map-of-the-socket-calls), with boxes to fill in (server: socket, bind, listen, accept; client: socket, connect; then read/write; close; `read` returns 0; back to accept), and the three exchanges: **three-way handshake**, **data**, **connection termination**.

**Remember:**

- Sockets can build a **lock** (bind an agreed port) and a **timer** (`select` with a timeout): both **True**.
- `select` said "**can accept**", not "**has data**": never `recv` right after `accept` without `select`.
- Shortening TIME_WAIT endangers **both** of its jobs.
- Self-stabilizing overlay: smaller ID connects, `select` **with a timeout**, close dead/wrong descriptors, reconnect empty slots, heartbeats → **convergence** and **closure**.

### 8.4 Related quiz topics not in the lectures

**In one sentence:** the quiz also asks about error codes, `fork`-based servers, `poll`, `epoll` (level- vs edge-triggered) and socket options, explained briefly here.

**Additional context.** The course's Socket API quiz (80 questions, based on Stevens' book) asks about several things the lecture only mentions or skips. Short explanations:

**Why the socket layer is in the kernel.** Only the kernel may touch the network card. If the socket API lived entirely in user programs, every program would need **privileged access to device drivers**.

**Errors you should recognise.**

| Situation | Result |
|---|---|
| Two sockets `bind` the same address and port (no special option) | the second `bind` fails with **EADDRINUSE** |
| `send`/`recv` on a TCP socket that never called `connect` | fails with **ENOTCONN** ("not connected"); a TCP socket never turns into UDP |
| `listen` without `bind` | succeeds, but the kernel picks a **random port**, so no client knows where to connect |
| Non-blocking `connect` | returns −1 with **EINPROGRESS**: the handshake has started, not finished. Wait until `select` marks the socket **writable**, then check the result with `getsockopt(SO_ERROR)`. |
| A signal arrives during `accept`, `select` or `read` | the call returns −1 with **EINTR**; simply call it again |
| Client's final handshake ACK is lost | the client is ESTABLISHED, but the server stays in **SYN_RCVD** and resends SYN+ACK |
| Client closes after the handshake but before `accept` | `accept` still returns the connection; the first `read` returns 0 |
| Both sides `close` at the same time | both go FIN_WAIT_1 → **CLOSING** → **TIME_WAIT** (both are active closers) |

**`readn` can block forever** if the peer sends **fewer than n bytes and keeps the connection open**.

**`getaddrinfo` hints.** Setting `hints.ai_family = AF_INET6` asks for **IPv6 results only**; `AF_UNSPEC` asks for both. (On Linux an IPv6 socket bound to `::` may still accept IPv4 clients as "IPv4-mapped" addresses unless the option `IPV6_V6ONLY` is set.)

**`inet_pton` and "300.1.1.1".** The quiz's answer is that it "only validates the string format, not the numeric range". Real modern implementations (for example on Linux) **do** reject 300.1.1.1 (return 0). The lesson either way: **check the return value**.

**Concurrency with `fork`.** The parent accepts, **forks a child** for the client, **closes its copy of the connected socket**, and returns to `accept`; the child closes its copy of the listening socket. When a child ends it becomes a **zombie** until the parent collects its exit status with `wait`/`waitpid`. The kernel tells the parent with the signal **SIGCHLD**, so the server installs a SIGCHLD handler that calls `waitpid` in a loop. Processes give **isolation** but cost **memory and CPU**.

```mermaid
sequenceDiagram
  participant P as Parent
  participant C as Child
  Note over P: accept gives connfd
  P->>C: fork (child gets copies of both sockets)
  Note over P: close connfd, back to accept
  Note over C: close listenfd, serve the client, exit
  C-->>P: SIGCHLD
  Note over P: waitpid removes the zombie
```

> **Kid example.** `fork` is like a shopkeeper who, for every new customer, makes a magic copy of themself. The copy serves the customer while the original goes back to the door. When a copy finishes, it leaves a little note ("I'm done", SIGCHLD); the original must pick up the note (`waitpid`), or the copy stays behind as a "zombie" taking up space.

**More `select` rules.** Rebuild the set before **every** call. When a client disconnects, **`close` it and `FD_CLR` it**; a closed descriptor left in the set makes `select` fail with **EBADF**. A socket can be readable and writable at once, so check both sets. **"Writable" only means there is space in the local send buffer**; the next `write` can still fail (for example after a reset). FD_SETSIZE is fixed at compile time, and the kernel scans every bit: **O(n)**.

**`poll`.** Works like `select` but takes an **array of `struct pollfd`** {fd, events wanted, events that happened}. **No FD_SETSIZE limit**, and no bit sets to rebuild, but still **O(n)**. Return value 0 = **timeout**. `POLLHUP` and `POLLERR` can be reported even if you didn't ask: close and clean up. After closing a socket, **update the array** (remove the entry or set its fd to −1); a stale entry shows up as `POLLNVAL`.

**`epoll` (Linux).** Register each descriptor **once** with `epoll_ctl(EPOLL_CTL_ADD)`; `epoll_wait` returns **only the ready ones**. Remove a descriptor with `EPOLL_CTL_DEL` **before** closing it, because the number may soon be reused by a new socket.

| Mode | Behaviour | Danger |
|---|---|---|
| **Level-triggered** (default) | reports "ready" **again and again** while data is waiting | repeated wake-ups |
| **Edge-triggered** (`EPOLLET`) | reports **once**, when new data arrives | you **must read (or write) in a loop until `EAGAIN`**, or leftover data never triggers a new event and the program stalls |

> **Kid example of edge-triggered epoll.** Level-triggered is a doorbell that keeps ringing while someone stands at the door. Edge-triggered rings once when people arrive; if you don't let everyone in, the ones still waiting won't ring again.

`epoll` cannot watch regular files, so a redirected stdin needs special handling.

**Socket options (`setsockopt`).**

| Option | What it does | Trade-off |
|---|---|---|
| `SO_REUSEADDR` | a restarted server can bind its port while old connections are in TIME_WAIT | in theory an old duplicate could reach the new connection |
| `SO_LINGER`, time 0 | `close` **throws away unsent data and sends RST** (abort) instead of FIN | fast, but data is lost and there is no TIME_WAIT |
| `SO_LINGER`, time > 0 | `close` **blocks** until unsent data is sent and acknowledged, or the time runs out | safer delivery, but `close` can stall a busy server |
| `SO_KEEPALIVE` | after a long idle time (default 2 hours) sends small probes to check the peer is alive | off by default: extra traffic, and a few lost probes on a bad network can wrongly declare a live peer dead |
| `TCP_NODELAY` | turns off **Nagle's algorithm**, which collects small writes into one segment | small writes go out at once (good for games, typing), but many tiny packets each carry 40 bytes of headers |

**Remember:**

- Error codes: **EADDRINUSE** (port taken), **ENOTCONN** (not connected), **EINPROGRESS** (non-blocking connect started), **EINTR** (interrupted, retry), **EBADF** (closed fd still in the set).
- `fork` server: parent closes `connfd`, child closes `listenfd`; reap zombies with **SIGCHLD** + `waitpid`.
- `poll`: array of `pollfd`, no FD_SETSIZE limit, still O(n). `epoll`: register once, get only ready fds; **edge-triggered** → read until `EAGAIN`.
- Options: `SO_REUSEADDR`, `SO_LINGER` (0 = abort with RST), `SO_KEEPALIVE` (2 hours), `TCP_NODELAY` (turns off Nagle).

## Revision notes and references

Technical clarifications, where the lecture material is corrected or made precise, appear in sections [2.4](#24-the-daytime-server), [3.4](#34-time_wait), [3.5](#35-port-number-ranges-and-ephemeral-ports), [5.3](#53-the-connect-function), [6.2](#62-io-models), [6.3](#63-the-select-function), [7.4](#74-c10k-and-c10m), [8.1](#81-summary). Where the slides and the course book differ, this guide follows the book and says so at that point.

- W. R. Stevens, B. Fenner and A. M. Rudoff, *UNIX Network Programming, Volume 1: The Sockets Networking API*, 3rd edition: the source of the lecture's code.
- Linux manual pages [socket(2)](https://man7.org/linux/man-pages/man2/socket.2.html), [select(2)](https://man7.org/linux/man-pages/man2/select.2.html), [shutdown(2)](https://man7.org/linux/man-pages/man2/shutdown.2.html) and [epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html): current API semantics.
- [RFC 9293 — TCP](https://www.rfc-editor.org/rfc/rfc9293): connection states, orderly close and TIME_WAIT.
- D. Kegel, ["The C10K problem"](http://www.kegel.com/c10k.html): the classic overview of serving many clients.
