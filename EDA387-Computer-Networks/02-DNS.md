# The Domain Name System (DNS): How Names Become Addresses

**Course:** EDA387/DIT663 Computer Networks, lecture "Domain Name System (DNS)" (Review 02). Part A repeats DNS as taught in the earlier course EDA344: names, delegation, resource records and glue. Part B, "Beyond EDA344", is new: DNSSEC, the root KSK rollover, encrypted DNS, anycast and how DNS fails. Several slides follow the Stanford CS144 lecture by David Mazières.

**Revised:** 21 October 2026

**† Supplementary material:** content that comes only from hidden PowerPoint slides. It may not be part of this year's examinable material.

**How to read this guide.** Every part starts with **In one sentence**, the whole idea in one line. Then comes the explanation in small steps, a **Kid example** (the same idea told with everyday things), pictures, and a **Remember** list with what you must know for the exam. **Technical clarification** marks a place where the slides are wrong or unclear, and gives the correct version. **Additional context** adds something useful that is not on the slides.

---

## Glossary

| Term | Full name | Meaning in simple words |
|---|---|---|
| A record | Address record | Says "this name has this IPv4 address", for example `stanford.edu. A 171.67.216.14`. |
| AA | Authoritative Answer | A flag in the reply that means "I am the official server for this name". |
| AAAA record | "quad-A" record | Says "this name has this IPv6 address". |
| Anycast | — | Many machines in different places use the **same** IP address; the network delivers each packet to the nearest one. |
| ARPA | Address and Routing Parameter Area (originally Advanced Research Projects Agency) | The top-level domain `.arpa`, used for reverse lookups (`in-addr.arpa`, `ip6.arpa`). |
| Authoritative server | — | A name server that holds the **official** records of a zone and answers for it. |
| Cache | — | A short-term memory. A resolver keeps answers it has already found, so it does not have to ask again. |
| Cache poisoning | — | Tricking a resolver into remembering a **false** answer, which it then gives to all its users. |
| CNAME | Canonical Name record | A nickname record: "this name is just another name for that (real) name". |
| Delegation | — | A parent zone hands the job of a child zone to other servers, using NS records. "Ask them, not me." |
| dig | Domain Information Groper | A command-line tool that sends a DNS question and prints the whole reply, part by part. |
| DNS | Domain Name System | The worldwide system that turns names (like www.chalmers.se) into values (like IP addresses). |
| DNSKEY | DNS Key record | DNSSEC record that publishes a zone's public key (its "stamp" for checking signatures). |
| DNSSEC | DNS Security Extensions | Adds digital signatures to DNS records, so a resolver can check an answer is genuine. It does **not** hide anything. |
| DoH | DNS over HTTPS | DNS questions sent inside HTTPS on port 443, looking like normal web traffic (RFC 8484). |
| DoQ | DNS over QUIC | DNS questions sent over the QUIC transport (RFC 9250). |
| DoT | DNS over TLS | DNS questions sent inside an encrypted TLS connection on port 853 (RFC 7858). |
| DS | Delegation Signer record | A record in the **parent** zone holding a fingerprint (hash) of the child zone's key: the parent vouching for the child. |
| EDNS(0) | Extension Mechanisms for DNS, version 0 | An add-on that lets a client say "I can accept UDP replies bigger than 512 bytes" (RFC 6891). |
| FQDN | Fully Qualified Domain Name | A complete name that goes all the way to the root, written with a final dot, for example `www.stanford.edu.` |
| Glue record | — | The address (A or AAAA record) of a name server, stored in the **parent** zone, that breaks a chicken-and-egg loop. |
| KSK | Key Signing Key | The DNSSEC key that signs a zone's set of keys. The root KSK is the starting point of trust for all of DNSSEC. |
| Label | — | One part of a name between dots: `www`, `stanford`, `edu`. At most 63 characters. |
| MX | Mail Exchange record | Says which server(s) receive e-mail for a domain, with a preference number. |
| Name compression | — | Inside one DNS message, a repeated name ending is replaced by a 2-byte pointer to where it already appeared. |
| NOTIFY | — | A message from a primary server to its secondaries: "the zone changed, refresh soon" (RFC 1996). |
| NS | Name Server record | Says which servers are the official (authoritative) servers for a zone. |
| nslookup | — | An older, simpler DNS tool. It hides the Additional section, so it is poor for studying glue. |
| NXDOMAIN | Non-Existent Domain | The reply code meaning "this name does not exist" (RCODE 3). |
| Oblivious DoH | — | DoH sent through a relay, so that nobody learns both **who** asked and **what** was asked. |
| PTR | Pointer record | Goes backwards: from an address (written under `in-addr.arpa` or `ip6.arpa`) to a name. A reverse lookup. |
| QNAME minimisation | Query-name minimisation | A resolver tells each server only the part of the name that server needs (RFC 9156). |
| RCODE | Response Code | 4-bit field in the reply with the result: 0 = no error, 3 = name does not exist, and so on. |
| RD / RA | Recursion Desired / Recursion Available | Flags: the client asks "please do the whole search for me" (RD); the server says "I can do that" (RA). |
| Recursive resolver | — | The server (your ISP's, or a public one like 8.8.8.8) that does the whole search for you and remembers the results. Also called the local DNS server. |
| Referral | — | An answer that says "I don't know, but ask these servers". |
| Resource record (RR) | — | One piece of DNS data, written `name TTL class type rdata`. |
| Root server | — | A server for the very top of the name tree, ".". It knows where every top-level domain is served. |
| RRSIG | Resource Record Signature | DNSSEC record holding a signature over a set of records. |
| SOA | Start of Authority record | The zone's "office information": version number, refresh timers, and how long "does not exist" answers may be remembered. |
| SPF | Sender Policy Framework | A TXT record listing which IP addresses may send e-mail for a domain. |
| SRV | Service record | Says which host and which port offer a given service, for example SIP over TLS. |
| Stub resolver | — | The small DNS client inside your computer or app; it only asks the recursive resolver and waits. |
| TC | Truncated | A flag meaning "the reply did not fit; ask again over TCP". |
| TLD | Top-Level Domain | The last part of a name: `.se`, `.com`, `.edu`, … |
| Trust anchor | — | The one key you trust in advance (for DNSSEC: the root KSK). All other trust is checked starting from it. |
| TTL | Time To Live | How many seconds a record may be kept in a cache before it must be thrown away. |
| TXT | Text record | Holds free text; used for things like SPF and for proving that you own a domain. |
| Zone | — | A part of the name tree run by one organisation, for example `chalmers.se`. |
| ZSK | Zone Signing Key | The DNSSEC key that signs the normal records of a zone; it is itself signed by the KSK. |

## Contents

- [Before you start: the words you need](#before-you-start-the-words-you-need)
- [1. Purpose and design of DNS](#1-purpose-and-design-of-dns)
  - [1.1 What DNS is for](#11-what-dns-is-for)
  - [1.2 Before DNS: the hosts.txt file](#12-before-dns-the-hoststxt-file)
  - [1.3 Goals of DNS and the "good news"](#13-goals-of-dns-and-the-good-news)
- [2. Hierarchy, delegation and resolution](#2-hierarchy-delegation-and-resolution)
  - [2.1 Zones, delegation and root servers](#21-zones-delegation-and-root-servers)
  - [2.2 How one lookup travels: recursive and non-recursive queries](#22-how-one-lookup-travels-recursive-and-non-recursive-queries)
  - [2.3 The DNS protocol: port 53, UDP and TCP](#23-the-dns-protocol-port-53-udp-and-tcp)
- [3. Resource records and DNS messages](#3-resource-records-and-dns-messages)
  - [3.1 Resource records](#31-resource-records)
  - [3.2 Glue records](#32-glue-records)
  - [3.3 Inside a DNS message](#33-inside-a-dns-message)
  - [3.4 Secondary servers and the SOA record](#34-secondary-servers-and-the-soa-record)
  - [3.5 MX records: where e-mail goes](#35-mx-records-where-e-mail-goes)
  - [3.6 CNAME records: aliases](#36-cname-records-aliases)
  - [3.7 Reverse lookups: PTR records](#37-reverse-lookups-ptr-records)
  - [3.8 Using DNS for load balancing](#38-using-dns-for-load-balancing)
  - [3.9 SRV and TXT records](#39-srv-and-txt-records)
  - [3.10 DNS overview: what Part A gives us](#310-dns-overview-what-part-a-gives-us)
- [4. Security and privacy](#4-security-and-privacy)
  - [4.1 What DNS never promised](#41-what-dns-never-promised)
  - [4.2 Cache poisoning and the Kaminsky attack](#42-cache-poisoning-and-the-kaminsky-attack)
  - [4.3 DNSSEC: a chain of trust](#43-dnssec-a-chain-of-trust)
  - [4.4 Signing and validating are different curves](#44-signing-and-validating-are-different-curves)
  - [4.5 The root KSK rollover](#45-the-root-ksk-rollover)
  - [4.6 Sweden as an outlier](#46-sweden-as-an-outlier)
  - [4.7 Encrypting the query: DoT, DoH and DoQ](#47-encrypting-the-query-dot-doh-and-doq)
  - [4.8 Who are you hiding from?](#48-who-are-you-hiding-from)
- [5. Resilience and failure](#5-resilience-and-failure)
  - [5.1 Anycast and root resilience](#51-anycast-and-root-resilience)
  - [5.2 How DNS fails](#52-how-dns-fails)
- [6. Practice and review](#6-practice-and-review)
  - [6.1 Exam skill: reading dig output](#61-exam-skill-reading-dig-output)
  - [6.2 Related quiz topics not in the lecture](#62-related-quiz-topics-not-in-the-lecture)
- [Revision notes and references](#revision-notes-and-references)

## Before you start: the words you need

DNS uses a few basic words again and again. Learn these first and the rest gets much easier.

- **IP address:** the number a computer really uses to send a packet to another computer, like `93.184.216.34`. It is like a phone number.
- **Name:** the easy word humans use instead, like `www.chalmers.se`. It is like a person's name in your contacts.
- **Resolver:** the helper that turns a name into an address for you. Your computer has a tiny one inside (the stub resolver), and it asks a bigger one on the network (the recursive resolver).
- **Name server:** a computer that answers DNS questions about some names.
- **Cache:** short-term memory. Once an answer is found, it is kept for a while (the TTL) so nobody has to search again.

```mermaid
flowchart LR
    U["You type<br/>www.chalmers.se"] --> S["Stub resolver<br/>(inside your laptop)"]
    S -->|"what is the address?"| R["Recursive resolver<br/>(the helper)"]
    R -->|"asks name servers<br/>if not in its cache"| N["Name servers<br/>around the world"]
    R -->|"answer: an IP address"| S
    S --> B["Browser connects<br/>to that address"]
```

> **Kid example.** DNS is the phone book of the Internet. You know your friend's name; the phone book gives you their number. Your phone (the computer) can only dial numbers, never names.

## 1. Purpose and design of DNS

**What this chapter is about:** why we need DNS at all, how people did it before DNS, and what DNS was designed to achieve.

### 1.1 What DNS is for

**In one sentence:** computers send packets to IP addresses, people remember names, and DNS is the system that turns the name into the address.

#### Parsing a URL

When you type an address into a browser, the browser first cuts it into parts. The lecture's example is:

`http://user:pass@google.com:80/docs/index.html?lang=en&sort=asc#section2`

```mermaid
flowchart LR
    A["http<br/>(scheme)"] --> B["user:pass<br/>(userinfo)"] --> C["google.com<br/>(host)"] --> D["80<br/>(port)"]
    D --> E["/docs/index.html<br/>(path)"] --> F["lang=en&sort=asc<br/>(query)"] --> G["section2<br/>(fragment)"]
```

Read the picture from left to right, in the same order as the URL.

- **Scheme** (`http`, `https`, `ftp`, `mailto`, `file`, `data`, `irc`, …): which protocol to use. It is followed by a colon `:`.
- **`//`** (the authority marker) is needed when a host (and possibly userinfo) follows.
- **Userinfo** (`user:pass`): an optional user name and password.
- **Host:port** (`google.com:80`): which machine, and which "door" (port) on it.
- **Path** (`/docs/index.html`): which file or resource on that machine.
- **Query** (`?lang=en&sort=asc`): extra settings sent to the server, as key=value pairs.
- **Fragment** (`#section2`): a place inside the page. The browser uses it; it is **not** sent to the server.

The **host** part is a name, but computers send packets to **IP addresses**. DNS is the system that turns the name into the address.

> **Kid example.** A URL is like a full delivery note: "by bicycle (scheme), to Pizza Palace (host), back door number 80 (port), shelf 3, box 2 (path), extra cheese please (query), open at page 5 (fragment)". The delivery person still needs the street address of Pizza Palace. DNS is the map that gives it.

#### How do we find the address?

The lecture shows the tool `nslookup`:

```
$ nslookup www.example.com
Server:   127.0.0.1
Address:  127.0.0.1#53

Non-authoritative answer:
Name:    www.example.com
Address: 93.184.216.34
Name:    www.example.com
Address: 2606:2800:220:1:248:1893:25c8:1946
```

What each line means:

1. `Server` and `Address`: which DNS server answered (here one on the same machine, `127.0.0.1`, on port 53).
2. **"Non-authoritative answer"**: the answer came from a **cache**, not from the official server that owns `example.com`.
3. The reply gives **one IPv4 address** (`93.184.216.34`) and **one IPv6 address** (`2606:2800:…`).

`nslookup` can do **forward lookups** (name → address), **reverse lookups** (address → name), and ask for record types like A, AAAA, MX, NS and CNAME. Professionals usually prefer `dig`, which shows more.

#### Motivation

- IP addresses are hard to remember, so we need to map **symbolic names to IP addresses**.
- This is done by **library functions and servers**: the function `getaddrinfo()` in your program talks to a name server over UDP (or TCP).
- More generally, DNS maps **symbolic names to values** of many kinds, not only addresses.

The lecture's picture shows a user sending mail to `user@example.com`:

```mermaid
sequenceDiagram
    participant M as Mail program
    participant N as Name server
    participant E as Server 93.184.216.34
    M->>N: where is example.com?
    N-->>M: 93.184.216.34
    M->>E: open a TCP connection over IP
```

**Remember:**
- Computers use IP addresses; people use names. DNS translates.
- Your program asks with `getaddrinfo()`, which talks to a name server over UDP (or TCP).
- "Non-authoritative answer" means "from a cache".
- DNS maps names to many kinds of values, not only addresses.

### 1.2 Before DNS: the hosts.txt file

**In one sentence:** before DNS, the whole Internet used one shared list of names, kept by one person, and that could not grow.

In the early ARPANET, every host was listed in **one file, `hosts.txt`**:

- To add a host, you **e-mailed the global network administrator**.
- The administrator **mailed out a new `hosts.txt` every few days**.
- The file was kept at `SRI-NIC.ARPA`, address `26.0.0.73` (RFC 952).
- Hosts periodically **downloaded updates with FTP**.

Today this would be **completely impractical**:

- The file would be huge (gigabytes).
- What if two people want to add the **same name**?
- Who is **authorized** to change the address of a name?
- People need to change mappings **more often than every few days** (for example dynamic IP addresses).

```mermaid
flowchart TB
    subgraph old["Old way: one central file"]
        direction LR
        admin["one administrator<br/>at SRI-NIC"] --> file["hosts.txt"]
        file -->|"FTP every few days"| h1["host"]
        file -->|"FTP every few days"| h2["host"]
    end
    subgraph new["DNS: many owners"]
        direction LR
        root["root zone"] -->|delegates| se[".se"] -->|delegates| ch["chalmers.se<br/>(run by Chalmers)"]
    end
    old -->|"did not scale"| new
```

> **Kid example.** Imagine the whole country's phone numbers written in **one notebook**, kept by one teacher. To add your number you send the teacher a letter, and every few days the teacher posts a fresh copy of the whole notebook to every home. That works for a village, not for the world. DNS instead gives each town its own phone book, and each town updates its own.

**Technical clarification.** The lecture says "look at your hosts.txt in `C:\Windows\System32\drivers\etc`". On Windows the file in that folder is called **`hosts`** (no `.txt` extension); on Linux and macOS it is `/etc/hosts`. It is the modern leftover of the old idea: entries in it are used before (or instead of) DNS on your own machine.

**Remember:**
- Before DNS: one file `hosts.txt`, one administrator, copied out by FTP.
- It failed because of size, name clashes, who-may-change-what, and slow updates.
- On your own machine the file `hosts` (Windows) or `/etc/hosts` still exists.

### 1.3 Goals of DNS and the "good news"

**In one sentence:** DNS must be huge, owned by many people and hard to break, and this is possible because names are read far more often than they change, so answers can be cached.

#### Goals

1. **Scalability.** DNS must handle a huge number of records, "potentially exponential in name size", because software can make up (synthesize) names on the fly.
2. **Distributed control.** Let people control their own names.
3. **Fault tolerance.** Old software assumed `hosts.txt` was always there, and lookups that fail can make programs fail badly. So DNS must **minimize lookup failures** even when other parts of the network have problems.

#### The good news

Two properties make these goals easier:

- **Read-only or read-mostly database.** People look names up far more often than names change.
- **Loose consistency.** When you add a machine, it is okay if the information takes minutes or hours to spread.

Together these suggest **aggressive caching**: once you have looked up a name, remember the result and do not ask again in the near future. How long you may remember is set by the record's **TTL** (time to live).

> **Kid example.** You ask your teacher for the pizza shop's number and write it on a note that says "valid for one week". For a week you use the note and do not bother the teacher. That note is a cache, and "one week" is the TTL. If the shop changes its number, you might call the old one for a few days, and that is okay (loose consistency).

**Remember:**
- Three goals: **scalability**, **distributed control**, **fault tolerance**.
- Two helpful facts: names are **read much more than changed**, and **small delays in updates are fine**.
- So DNS caches aggressively, for as long as the **TTL** says.

## 2. Hierarchy, delegation and resolution

**What this chapter is about:** how the name tree is cut into zones, how a resolver walks down that tree to find an answer, and which transport (UDP or TCP) DNS uses.

### 2.1 Zones, delegation and root servers

**In one sentence:** the name tree is cut into zones, each run by its own owner, and every parent zone tells you where to find the servers of its children.

#### Zones and delegation

DNS breaks the namespace (the tree of all names) into **zones**:

- `.` (the "root"), `.se`, `chalmers.se`, …
- Zones are **separately administered**, which is called **delegation**.
- **Parent zones tell you how to find the servers for their subdomains.**
- Each zone is served from **several replicated servers** (copies, so one broken server is not a disaster).

The lecture's figure shows the classic name tree: below the root are `edu`, `com`, `gov`, `mil`, `org`, `net`, `uk`, `fr`, …; below `edu` are `princeton`, `mit`, …; below `princeton` are `cs`, `ee`, `physics`; and below `cs` are host names like `ux01` and `ux04`. Boxes drawn around parts of the tree are the zones.

```mermaid
flowchart TD
  root["(root)"]
  subgraph rootzone["root zone"]
    root
  end
  root --> edu["edu"]
  root --> com["com"]
  root --> se["se"]
  subgraph eduzone["edu zone"]
    edu
  end
  subgraph sezone["se zone"]
    se
  end
  edu --> princeton["princeton"]
  edu --> mit["mit"]
  com --> cisco["cisco"]
  se --> chalmers["chalmers"]
  subgraph pz["princeton.edu zone"]
    princeton --> ee["ee"]
    princeton --> physics["physics"]
  end
  princeton --> cs["cs"]
  subgraph csz["cs.princeton.edu zone"]
    cs --> ux01["ux01"]
    cs --> ux04["ux04"]
  end
  subgraph chz["chalmers.se zone"]
    chalmers --> www["www"]
  end
```

Arrows point from a parent name to a child name. Each box is one zone: an organisation that runs its own servers. Notice that `ee` and `physics` stay inside the `princeton.edu` zone, while `cs` has been **delegated** into its own zone.

A name is read **from right to left**, from general to specific: `www.chalmers.se.` means "inside the root, inside `se`, inside `chalmers`, the host `www`". The final dot is the root.

```mermaid
flowchart LR
    A[".<br/>root"] --> B["se<br/>country"] --> C["chalmers<br/>organisation"] --> D["www<br/>one machine"]
```

The picture reads `www.chalmers.se.` backwards: from the most general part (the root) to the most specific part (one machine).

> **Kid example.** Think of a postal address written backwards: "Sweden, Gothenburg, Main Street 5, flat 3". The country post office only knows how to reach Gothenburg; the Gothenburg office only knows the streets; the building manager knows the flats. Nobody needs to know everything. Each one just **delegates** ("passes on the job") to the next level down. A zone is one of those offices, with its own boss.

#### Root servers

- Root (and TLD) servers must be **widely replicated**, because every lookup that is not cached starts at the top.
- For some, operators use tricks like **IP anycast**: the same IP address is announced from many places, and the network sends each query to the nearest copy. (More in [section 5.1](#51-anycast-and-root-resilience).)

**Remember:**
- The name tree is split into **zones**, each run by its own owner (**delegation**).
- A **parent tells you where its children's servers are**.
- Names are read **right to left**; the final dot is the **root**.
- Root and TLD servers are copied many times, some using **anycast**.

### 2.2 How one lookup travels: recursive and non-recursive queries

**In one sentence:** your computer asks one helper "find it for me" (recursive), and the helper then asks root, TLD and the official server one by one "what do you know?" (non-recursive).

There are **two types of query**:

- **Recursive:** "Please find the final answer for me." Applications make recursive queries to their **local DNS server** (the recursive resolver).
- **Non-recursive** (also called iterative): "Tell me what you know, or who to ask next." The local server queries remote servers non-recursively.

The local server **caches results aggressively**. For example, it only needs to contact the root on the **first** query for a name ending in `.umass.edu`; after that it remembers where the `edu` servers are.

The lecture's figure: the host `cis.poly.edu` wants the address of `gaia.cs.umass.edu`.

```mermaid
sequenceDiagram
  participant H as Requesting host cis.poly.edu
  participant L as Local DNS server dns.poly.edu
  participant R as Root DNS server
  participant T as TLD DNS server (edu)
  participant A as Authoritative server dns.cs.umass.edu
  H->>L: 1. recursive query: gaia.cs.umass.edu?
  L->>R: 2. non-recursive query
  R-->>L: 3. referral: ask the edu servers
  L->>T: 4. non-recursive query
  T-->>L: 5. referral: ask dns.cs.umass.edu
  L->>A: 6. non-recursive query
  A-->>L: 7. answer: the IP address
  L-->>H: 8. answer (and the local server caches it)
```

Step by step:

- **Step 1:** your host asks its local server once, and waits for the final answer (**recursive**).
- **Steps 2 and 3:** the local server asks a root server. The root does not know `gaia`, but it says "ask the `edu` servers" (a **referral**).
- **Steps 4 and 5:** the local server asks an `edu` server, which says "ask `dns.cs.umass.edu`".
- **Steps 6 and 7:** the local server asks `dns.cs.umass.edu`, the **authoritative** (official) server, which gives the address.
- **Step 8:** the local server gives the address to your host, and keeps a copy in its cache.

Steps 2, 4 and 6 are **non-recursive**: each server either answers or gives a referral.

> **Kid example.** You ask your mum (the recursive resolver) "Where does Sara live?". Mum calls the school office (root), who says "ask her class teacher" (TLD). The teacher says "ask Sara's parents" (authoritative server). They give the address. Mum tells you. You only asked once; mum did all the calling. And mum writes the address down, so next time she knows it at once (cache).

**Technical clarification.** The figure numbers the arrows 1 to 8, but the lecture's explanation describes only steps 1 to 7 and says the local server returns the answer to the client "(7)". In the figure, step 7 is the authoritative server's reply to the local server and step 8 is the local server's reply to the host.

**Remember:**
- **Recursive** = "find the final answer for me" (host → local server).
- **Non-recursive / iterative** = "tell me what you know or who to ask" (local server → root, TLD, authoritative).
- A **referral** is "I don't know, ask these servers".
- The local server **caches**, so the root is only needed the first time.

### 2.3 The DNS protocol: port 53, UDP and TCP

**In one sentence:** DNS normally sends one small UDP packet each way on port 53, and switches to TCP when the answer is too big.

- DNS uses **TCP or UDP port 53**.
- **Most traffic uses UDP.** It is a lightweight protocol with a **512-byte message limit** for UDP.
- If UDP fails (for example, the **reply is truncated** because it was too big), the client **retries with TCP**.
- TCP is a byte stream with no message boundaries, so over TCP **every message is prefixed with its length** as a 16-bit integer.
- **One bit in the query** (RD, "recursion desired") says whether the query is recursive.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: question over UDP, port 53
    S-->>C: reply too big, so TC bit set (truncated)
    C->>S: same question again over TCP, port 53
    S-->>C: full reply (length first, then the message)
```

> **Kid example.** UDP is like sending a postcard: quick and cheap, but it only has room for a little writing. If the answer does not fit on the postcard, the other side writes "too long, call me" (the TC bit), and you phone them (TCP), where you can talk as long as you need.

**Technical clarification.** The lecture's explanation says "the UDP protocol has a maximum message size of 512 bytes". That limit is a rule of **DNS** (RFC 1035), not of UDP: UDP itself can carry up to about 65,500 bytes of data.

**Additional context.** Since 1999 the extension **EDNS(0)** (now RFC 6891) lets a client say "I can accept bigger UDP replies", for example 1,232 or 4,096 bytes. This matters for DNSSEC, whose signatures make replies much larger than 512 bytes.

**Remember:**
- DNS uses **port 53**, mostly over **UDP**, falling back to **TCP** when a reply is **truncated** (TC bit).
- Over TCP, each message starts with a **16-bit length**.
- The 512-byte limit is a **DNS** rule, not a UDP rule; **EDNS(0)** allows bigger UDP replies.
- The **RD** bit says "recursion desired".

## 3. Resource records and DNS messages

**What this chapter is about:** what DNS data looks like (records), the chicken-and-egg problem that glue records solve, what a DNS message looks like inside, and the many kinds of records: SOA, MX, CNAME, PTR, SRV and TXT.

### 3.1 Resource records

**In one sentence:** every piece of DNS information is one line called a resource record, saying "this name, for this long, has this type of value".

**All DNS information is stored as resource records (RRs):**

`name [TTL] [class] type rdata`

| Field | Meaning | Example |
|---|---|---|
| name | The domain name the record is about | `www.stanford.edu.` |
| TTL | Time to live, in seconds: how long caches may keep the record | `1800` (30 minutes) |
| class | For extensibility; almost always `IN` (value 1), "Internet" | `IN` |
| type | The kind of record | `A`, `NS`, `MX`, … |
| rdata | The resource data; its format depends on the type | `171.67.216.14` |

Brackets mean the field may be left out when writing records by hand.

The two most important types:

- **A**: an Internet (IPv4) address.
- **NS**: a name server for the zone.

Example output of `dig stanford.edu`:

```
stanford.edu.   1800    IN  A   171.67.216.14
stanford.edu.   1800    IN  A   171.67.216.16
stanford.edu.   172800  IN  NS  Argus.stanford.edu.
```

How to read it:

- **Line 1:** "the name `stanford.edu.` has an Internet-class address record with value 171.67.216.14, and you may cache that for 1,800 seconds".
- **Line 2:** a second address for the same name (a name may have several).
- **Line 3:** `Argus.stanford.edu.` is a name server for `stanford.edu`, and that fact may be cached for 172,800 seconds (2 days). NS records usually have long TTLs because they change rarely.

```mermaid
flowchart LR
    N["name<br/>stanford.edu."] --- T["TTL<br/>1800 s"] --- C["class<br/>IN"] --- Y["type<br/>A"] --- D["rdata<br/>171.67.216.14"]
```

> **Kid example.** A resource record is like one line in a contact list: "Grandma (name) — keep this for one month (TTL) — phone (class) — mobile (type) — 070-123 45 67 (data)". Each kind of fact (mobile, home address, e-mail) is a different type, and one person can have several lines.

#### How the local server knows the root servers

The local name server must be configured with a **root cache file** (also called the root hints file), which lists the root name servers and their addresses:

```
.                    3600000  NS  A.ROOT-SERVERS.NET.
A.ROOT-SERVERS.NET.  3600000  A   198.41.0.4
.                    3600000  NS  B.ROOT-SERVERS.NET.
B.ROOT-SERVERS.NET.  3600000  A   128.9.0.107
```

Each line is one record: name, TTL, type, data. The NS lines say "the root zone `.` is served by A.ROOT-SERVERS.NET (and B, …)"; the A lines give their addresses. With this file the resolver can start every lookup from the top.

> **Kid example.** Every new phone comes with one number already saved: the number of directory enquiries. You never had to look it up, so you can always start from there. The root hints file is that pre-saved number.

**Technical clarification.** This file excerpt is old. `198.41.0.4` is still the address of A-root, but B-root has been renumbered more than once since `128.9.0.107` (most recently in 2023). Real resolvers ship an up-to-date copy, published by IANA. The lecture also describes "six columns" as if both lines were one row; in fact each line is a separate record with four fields.

**Remember:**
- A record is `name [TTL] [class] type rdata`; class is almost always `IN`.
- **A** = IPv4 address; **NS** = name server of a zone.
- **TTL** is in seconds; NS records usually have long TTLs.
- A resolver starts from a built-in **root hints** file.

### 3.2 Glue records

**In one sentence:** when a zone's name server has a name **inside** that same zone, the parent must also hand out the server's address (a glue record), or nobody could ever reach it.

#### The chicken-and-egg problem

To look up names ending in `.stanford.edu.`, you ask `Argus.stanford.edu.`. But how do you get the address of `Argus.stanford.edu.`? It is itself a name inside `stanford.edu`, so to find it you would have to ask … `Argus.stanford.edu.`

The lecture acts this out with `example.com`:

1. Client: what is the IP of `example.com`?
2. Local DNS: where can I find it? Maybe via `ns1.example.com`?
3. Local DNS: what is the IP address of `ns1.example.com`?
4. Local DNS: oh, I first need to resolve `example.com`…
5. Local DNS: I think we have a problem.
6. `.com` DNS: **let me store the IP address of `ns1.example.com`!**

**Solution: glue records.** A glue record is an A record for the name server, stored **in the parent zone**. The name servers for `edu.` hold the A record of `Argus.stanford.edu.`; the `.com` servers hold the A record of `ns1.example.com`. The parent hands it out together with the referral, in the **Additional** section of the reply.

```mermaid
flowchart TB
  subgraph loop["Without glue: a loop"]
    direction LR
    q1["need IP of<br/>example.com"] -->|"ask its server"| q2["need IP of<br/>ns1.example.com"]
    q2 -->|"it is inside<br/>example.com"| q1
  end
  subgraph fix["With glue"]
    direction LR
    p["the .com servers"] -->|"referral: NS ns1.example.com<br/>plus glue A record"| r["resolver"]
    r -->|"contacts the server<br/>directly"| n["ns1.example.com"]
  end
  loop ~~~ fix
```

> **Kid example.** The school office says "for questions about class 5B, ask 5B's teacher". But the only list of where 5B's teacher sits hangs inside 5B's own room, and you need the teacher to get in. Stuck! So the office also writes down the teacher's room number for you. That room number, kept at the office (the parent), is the glue.

**When is glue needed?** Only when the name server's name is **inside the zone being delegated** (an "in-bailiwick" server). If the server's name is somewhere else in the tree, the resolver can look it up the normal way, and no glue is needed.

#### Walking the tree by hand with dig

The lecture looks up `www.scs.stanford.edu` with no cache, by asking each server in turn with `dig +norec` (`+norec` turns off the RD bit, so each server only says what it knows):

```
dig +norec www.scs.stanford.edu @a.root-servers.net
dig +norec www.scs.stanford.edu @a.edu-servers.net
dig +norec www.scs.stanford.edu @argus.stanford.edu
dig +norec www.scs.stanford.edu @ns1.fs.net
```

```mermaid
sequenceDiagram
  participant Y as You (dig +norec)
  participant R as a.root-servers.net
  participant E as a.edu-servers.net
  participant S as argus.stanford.edu
  participant F as ns1.fs.net
  Y->>R: www.scs.stanford.edu?
  R-->>Y: referral to the edu servers, with glue
  Y->>E: www.scs.stanford.edu?
  E-->>Y: referral to the stanford.edu servers (Argus...), with glue
  Y->>S: www.scs.stanford.edu?
  S-->>Y: referral to the scs.stanford.edu servers (ns1.fs.net...)
  Y->>F: www.scs.stanford.edu?
  F-->>Y: the answer (A record)
```

This gives the results for `.edu`, `stanford.edu`, `scs.stanford.edu` and finally `www.scs.stanford.edu`. "Where are the glue records?" The lecture's answer is **in the first and second replies**, in the **ADDITIONAL** section. The reason fits the rule above:

- The `edu` servers (`a.edu-servers.net`, …) and `Argus.stanford.edu` are named **inside** the zones being delegated, so the parent must supply their addresses.
- The servers for `scs.stanford.edu` include `ns1.fs.net`, which lives in a completely different part of the tree (`fs.net`), so the resolver can find it by a normal lookup. No glue needed.

#### The Chalmers case

To reach `www.chalmers.se`, a resolver asks the `.se` servers: who serves `chalmers.se`? According to the slide, there are four answers:

- `ns1.chalmers.se`, `ns2.chalmers.se`, `ns3.chalmers.se`: three servers **inside the zone being delegated**. Their addresses live only on servers we cannot reach yet, so **`.se` must carry glue A records** for them.
- `dns.uu.se`, in Uppsala: **needs no glue**. The resolver can find it through the ordinary path `.se → uu.se`.

"One delegation, both sides of the rule."

```mermaid
flowchart TD
  subgraph sezone[".se zone (the parent)"]
    ns["NS records for chalmers.se"]
    glue["glue A records for<br/>ns1, ns2, ns3.chalmers.se"]
  end
  ns --> n1["ns1.chalmers.se"]
  ns --> n2["ns2.chalmers.se"]
  ns --> n3["ns3.chalmers.se"]
  ns --> uu["dns.uu.se"]
  glue -.->|"addresses for"| n1
  glue -.->|"addresses for"| n2
  glue -.->|"addresses for"| n3
  uu -->|"found the normal way<br/>via uu.se: no glue"| uuz["uu.se zone"]
```

Two lessons:

- **Resilience.** Chalmers' own operations documentation explains why one name server is kept off site: so that some data stays reachable when Chalmers' connection to SUNET (the Swedish university network) is down. Their example is e-mail: a sending mail server that gets **no answer at all** may throw the message away, but one that gets an answer keeps retrying. So the availability of the naming layer changes how a service two layers above it fails. `dns.uu.se` is not just a different building; it is a different **autonomous system** (see the routing guide).
- **Caution.** The delegation comes from an undated Chalmers page and may be old. The lecturer is told to check it before teaching. If Chalmers no longer has in-bailiwick servers, the lesson becomes "when is glue *not* needed".

> **Kid example.** Chalmers keeps a spare key to its house at a friend's house in Uppsala. If the road to Chalmers is blocked, people can still get the spare key from Uppsala. And because the friend lives in another town with its own address, you can find the friend without asking Chalmers first (no glue needed).

**Additional context.** The same delegation appears in the October 2018 exam (dig output captured in June 2018): asking `a.ns.se` for `www.chalmers.se` returned no answer, four NS records in the authority section, and the A records of `ns1`, `ns2`, `ns3.chalmers.se` in the additional section. That is exactly a referral with glue. See [section 6.1](#61-exam-skill-reading-dig-output).

#### Verify it yourself

| Command | What it shows |
|---|---|
| `dig +trace www.chalmers.se` | Walks root → `.se` → `chalmers.se`, printing each referral. The glue is in the referral from `.se`. |
| `dig +short NS chalmers.se` | The full set of name servers. Which are inside the zone? Which needs no glue? |
| `dig +short DS chalmers.se` | `.se` is signed with DNSSEC. Does it vouch for a Chalmers key? |
| `dnsviz.net/d/chalmers.se` | A website that draws the whole chain. Works on any operating system. |

- **The glue is the ADDITIONAL section of the referral.** In `dig +trace`, the AUTHORITY section lists the NS names, and the ADDITIONAL section carries their addresses. Without it the resolver would have names and no way to reach them.
- **Windows:** `nslookup` hides the ADDITIONAL section, which is exactly the thing being taught. Use DNSViz or WSL (Linux on Windows) to run `dig`.
- **The DS question has two possible outcomes, and both teach something.** If a DS record comes back, `.se` signs a commitment to a Chalmers key and the trust chain runs root → `se` → `chalmers.se`. If nothing comes back, `.se` is signed but `chalmers.se` is not, so the chain stops at the parent: the "signing versus validating" gap of [section 4.4](#44-signing-and-validating-are-different-curves).

**The pivot to Part B.** Delegation is one mechanism carrying two things: **addresses** (glue, which we have just seen) and **trust** (DS, which comes in Part B). The resolver walks the same path both times.

**Remember:**
- Glue = the **address of a name server**, stored in the **parent** zone.
- Needed only when the server's name is **inside** the zone being delegated (in-bailiwick).
- Glue arrives in the **ADDITIONAL** section of a referral.
- Chalmers: `ns1/2/3.chalmers.se` need glue; `dns.uu.se` does not.
- Delegation carries **addresses** (glue) and **trust** (DS).

### 3.3 Inside a DNS message

**In one sentence:** a DNS message has a 12-byte header and four sections (question, answer, authority, additional), and the same layout is used for questions and replies.

#### The five sections

The structure of a DNS message is defined in **RFC 1035**. **The same format is used for queries and replies.**

```mermaid
block-beta
  columns 2
  h["Header"] h2["ID, flags, and four counts"]
  q["Question"] q2["the question for the name server"]
  a["Answer"] a2["RRs answering the question"]
  au["Authority"] au2["RRs pointing toward an authority"]
  ad["Additional"] ad2["RRs holding additional information"]
```

- A **query** has no RRs in the Answer, Authority or Additional sections.
- A **reply** repeats the question and adds RRs.
- **Authority** allows for delegation: it holds NS records saying "ask these servers".
- **Additional** holds glue and other RRs the client might need next.

> **Kid example.** A DNS message is like a school form. The top box (header) has a form number and some tick boxes. Then: "Your question", "The answer", "Who to ask instead", and "Extra useful information". When you hand in the form, only "Your question" is filled in. When it comes back, the other boxes are filled in too.

#### The header

The header is 12 bytes: six 16-bit rows.

```mermaid
block-beta
  columns 1
  id["ID (16 bits)"]
  fl["Flags (16 bits): QR, Opcode, AA, TC, RD, RA, Z, RCODE"]
  qd["QDCOUNT (16 bits)"]
  an["ANCOUNT (16 bits)"]
  ns["NSCOUNT (16 bits)"]
  ar["ARCOUNT (16 bits)"]
```

The second row is split into eight fields, from the first bit to the last: QR (1 bit), Opcode (4), AA (1), TC (1), RD (1), RA (1), Z (3), RCODE (4). That adds up to 16 bits.

| Field | Bits | Meaning |
|---|---|---|
| ID | 16 | A number chosen by the client; the reply copies it, so the client can match replies to queries. |
| QR | 1 | 0 = query, 1 = response. |
| Opcode | 4 | Kind of query; 0 = standard query. |
| AA | 1 | Authoritative answer: the replying server is in charge of this name. |
| TC | 1 | Truncated: the reply did not fit (retry over TCP). |
| RD | 1 | Recursion desired: set by the client. |
| RA | 1 | Recursion available: set by the server if it is willing to do recursion. |
| Z | 3 | Reserved; must be zero. |
| RCODE | 4 | Error code; 0 = no error. |
| QDCOUNT | 16 | Number of entries in the Question section. |
| ANCOUNT | 16 | Number of RRs in the Answer section. |
| NSCOUNT | 16 | Number of RRs in the Authority section. |
| ARCOUNT | 16 | Number of RRs in the Additional section. |

> **Kid example.** The ID is like a ticket number at a bakery. You take ticket 47, and when "47" is called you know that this bread is yours, even if many people are waiting.

**Additional context.** Common RCODE values: 0 NOERROR, 1 FORMERR (bad query), 2 SERVFAIL (server failed, for example a DNSSEC validation failure), 3 NXDOMAIN (name does not exist), 5 REFUSED. In `dig` output the flags line shows the set bits in lower case, for example `flags: qr aa rd`. Later RFCs gave two of the three Z bits a meaning for DNSSEC (AD, "authenticated data", and CD, "checking disabled").

#### How a resource record is encoded

```mermaid
block-beta
  columns 1
  n["NAME (variable length)"]
  t["TYPE (16 bits)"]
  c["CLASS (16 bits)"]
  ttl["TTL (32 bits)"]
  l["RDLENGTH (16 bits)"]
  d["RDATA (RDLENGTH bytes)"]
```

| Field | Size | Meaning |
|---|---|---|
| NAME | variable | The domain name, encoded as in [How names are encoded](#how-names-are-encoded). |
| TYPE | 16 bits | The record type (A = 1, NS = 2, CNAME = 5, MX = 15, …). |
| CLASS | 16 bits | Usually IN = 1. |
| TTL | 32 bits | Seconds the record may be cached. |
| RDLENGTH | 16 bits | Length of RDATA in bytes. |
| RDATA | RDLENGTH bytes | The data: 4 bytes for an A record, a name for an NS record, and so on. |

#### How names are encoded

- A DNS name is a series of **labels**. `www.stanford.edu.` has 3 labels: `www`, `stanford`, `edu`.
- Labels may contain letters, digits and `-`, but **should not start or end with `-`**.
- A label is at most **63 characters**.
- Each label is encoded as **a length byte followed by the label**: `03 www 08 stanford 03 edu 00`.
- The **last label is always empty** (length 0): that zero byte is the root.

```mermaid
flowchart LR
    a["03<br/>(length)"] --> b["www<br/>(3 bytes)"] --> c["08<br/>(length)"] --> d["stanford<br/>(8 bytes)"] --> e["03<br/>(length)"] --> f["edu<br/>(3 bytes)"] --> g["00<br/>(root: empty label)"]
```

So the whole name takes 1 + 3 + 1 + 8 + 1 + 3 + 1 = **18 bytes**.

> **Kid example.** Imagine writing words on a strip of paper with no dots or spaces, but before each word you write how many letters it has: "3www8stanford3edu0". Reading it, you count: take 3 letters (www), then 8 letters (stanford), then 3 (edu), then 0 means "the end". No dots needed.

- **Names are case-insensitive** (`WWW.Stanford.EDU` is the same name), **but the server must preserve the case of the question in its reply.** The lecture suggests trying `www.sTANford.EDu` and looking at the authority section.

**Additional context.** A whole name may be at most 255 bytes in this encoded form (RFC 1035).

#### Name compression

A DNS message often repeats the same name endings (for example `stanford.edu` in the question, in the answer, in the NS records and in the glue). The case-preservation rule makes this worse, because the server cannot just rewrite everything in one style. So DNS allows **pointer labels that re-use suffixes**:

- A normal label starts with a length byte between 0 and 63. In binary, 63 is `00111111`, so the **top two bits of a normal length byte are always `00`**.
- If the byte is **0xC0 (192) or more**, i.e. its top two bits are `11`, it is not a length. The first two bytes together form a **pointer**: subtract `0xC000` from them, and the result is an **offset** into the message where the rest of the name can be found.

```mermaid
block-beta
  columns 4
  b["1 1"] off["OFFSET (14 bits)"]:3
```

The first two bits are both 1; the remaining 14 bits are the offset (widths roughly to scale).

**Worked example.** The header is 12 bytes, so the question name always begins at byte offset 12 (0x0C). If an answer record has the same name as the question, its NAME field can be just the two bytes `C0 0C`: `0xC00C − 0xC000 = 0x000C = 12`, "the name that starts at byte 12". Two bytes instead of 18 for `www.stanford.edu`. A pointer can also come after some labels, for example `04 mail` followed by a pointer to `stanford.edu` stored elsewhere in the message.

```mermaid
flowchart LR
    Q["byte 12: question name<br/>03 www 08 stanford 03 edu 00"]
    A["answer record NAME:<br/>C0 0C"] -->|"points back to byte 12"| Q
```

> **Kid example.** In a long story you write "Princess Isabella of the Northern Kingdom" once, and after that you just write "(see page 1)". Same meaning, much less writing. The pointer `C0 0C` is "see byte 12".

**Additional context.** A quiz question from 2023–2025 asks for the main purpose of compression: re-using repeated suffixes inside one message, which keeps messages small and avoids truncation.

**Remember:**
- A message = **header (12 bytes)** + Question + Answer + Authority + Additional; same format for query and reply.
- Header flags: **QR, Opcode, AA, TC, RD, RA, Z, RCODE**; the **ID** matches replies to queries.
- A record on the wire: NAME, TYPE, CLASS, TTL (32 bits), RDLENGTH, RDATA.
- Names: length byte + label, ending with `00`; labels ≤ 63 characters; names are case-insensitive but case is preserved.
- **Compression:** a byte starting with bits `11` begins a 2-byte pointer; `C0 0C` = "the name at byte 12".

### 3.4 Secondary servers and the SOA record

**In one sentence:** a zone is copied from one primary server to several secondary servers, and the zone's SOA record says how often they should check for changes.

- **Availability requires geographically dispersed replicas.** For example, Stanford could ask MIT to host a secondary copy of `scs.stanford.edu`.
- **Typical setup:** one master (primary) server and many slave (secondary) servers that copy the zone from it.
- **How often should secondaries sync up?** It is a trade-off:
  - all the time → high overhead;
  - rarely → stale (out-of-date) data.
- DNS **puts this trade-off under the domain owner's control**: fields in the **Start of Authority (SOA)** record control the secondaries' behaviour.
- The primary can **unilaterally change the SOA**.
- To speed up propagation, the primary can also **notify** the secondaries of a change, as a hint to refresh sooner (**NOTIFY, RFC 1996**).

```mermaid
flowchart LR
  P["Primary (master) server"] -->|"NOTIFY: zone changed<br/>(RFC 1996)"| S1["Secondary server 1"]
  P -->|"NOTIFY"| S2["Secondary server 2"]
  S1 -->|"checks the SOA serial<br/>on its refresh timer"| P
  S2 -->|"checks the SOA serial<br/>on its refresh timer"| P
  P -->|"zone transfer if<br/>the serial is newer"| S1
  P -->|"zone transfer if<br/>the serial is newer"| S2
```

The SOA record "states administrative information for a zone" (try `dig stanford.edu soa`), and it also **tells you how long you can cache negative results** (answers saying a name does not exist).

> **Kid example.** The teacher has the master copy of the class list. Two helpers keep copies. Every morning (the refresh timer) the helpers ask "what is the version number of the list?". If the number went up, they copy the new list. If the teacher changes something important, she can also shout "I changed the list!" (NOTIFY) so the helpers check right away.

**Additional context.** The SOA's seven data fields, as listed in the course quiz guide and RFC 1035:

| Field | Meaning |
|---|---|
| **MNAME** | The primary server. |
| **RNAME** | The administrator's e-mail address, with the `@` written as a dot. |
| **SERIAL** | Version number; secondaries copy the zone when it grows. |
| **REFRESH** | How often secondaries check. |
| **RETRY** | How long to wait after a failed check. |
| **EXPIRE** | When a secondary that cannot reach the primary stops answering. |
| **MINIMUM** | Today used for the negative-caching time (RFC 2308). |

**Remember:**
- One **primary**, several **secondaries** in different places, for availability.
- The **SOA** record lets the owner choose the sync trade-off; the **SERIAL** number tells secondaries when to copy.
- **NOTIFY** (RFC 1996) is a hint to refresh sooner.
- The SOA also sets how long "does not exist" answers are cached.

### 3.5 MX records: where e-mail goes

**In one sentence:** an MX record says which server receives the e-mail for a domain, even if the domain itself has no address.

"For historical reasons, mail does not have to use A records directly." An **MX (Mail Exchange)** record names the servers that accept mail for a domain.

The lecture's example: `ping scs.stanford.edu` says there is **no such host** (there is no A record), but you can still mail the CS144 staff at that domain, because `dig scs.stanford.edu mx` returns MX records pointing at real mail servers.

```mermaid
flowchart LR
    S["Sender's mail server"] -->|"asks DNS: MX of<br/>scs.stanford.edu?"| D["DNS answer:<br/>MX 10 mail server A<br/>MX 20 mail server B"]
    D -->|"sender tries the<br/>lowest number first"| A["mail server A"]
    D -.->|"only if A does<br/>not answer"| B["mail server B"]
```

> **Kid example.** A big company's building has no letterbox at the front door. Instead a sign says "post goes to the mail room, back entrance". The MX record is that sign. If there are two mail rooms, the one with the lower number is tried first.

**Additional context.** An MX record is written `name TTL IN MX preference host`, for example `example.com. 3600 IN MX 10 mail.example.com.` A **lower** preference number means "try this server first".

**Additional context (exam warning).** In the October 2024 exam results, the question "Which DNS record type is used for email delivery?" shows "A record" as correct. That key is wrong: the right answer is **MX record**.

**Remember:**
- **MX** = where e-mail for a domain goes.
- A domain can receive mail even with **no A record**.
- **Lower preference number = tried first.**
- E-mail delivery uses **MX**, not A (the 2024 exam key is wrong).

### 3.6 CNAME records: aliases

**In one sentence:** a CNAME says "this name is only a nickname; look up that other (real) name instead".

*(Part B starts here: "Beyond EDA344". Everything from this point on is new compared with the introductory course.)*

A **CNAME** record says a name is an **alias**:

`name [TTL] [IN] CNAME canonical-name`

It works **as if every RR of the canonical name also belonged to the alias**. A program can ask for the canonical (real) name with the **`AI_CANONNAME`** flag to `getaddrinfo()`.

Examples, to save typing:

```
wb.scs.stanford.edu.  CNAME  williamsburg-bridge.scs.stanford.edu.
mb.scs.stanford.edu.  CNAME  manhattan-bridge.scs.stanford.edu.
```

> **Kid example.** A CNAME is a nickname. If your friend Alexander is called "Alex" by everyone, then asking "Where does Alex live?" gives the same answer as asking about Alexander, because "Alex" just points to the real name.

#### The Yahoo example

- `www.yahoo.fr` has one CNAME, `rc.yahoo.com`.
- `www.yahoo.de` has the **same** CNAME, `rc.yahoo.com`.
- So both the French and German names point to the same canonical name, and `rc.yahoo.com` can have A/AAAA records spread across Yahoo's infrastructure.

The full chain, measured with `dig`:

```mermaid
flowchart TD
  fr["www.yahoo.fr"] -->|"CNAME (TTL 300)"| rc["rc.yahoo.com"]
  de["www.yahoo.de"] -->|"CNAME (TTL 1800)"| rc
  rc -->|"CNAME (TTL 300)"| ga["global-accelerator.dns-rc.aws.oath.cloud"]
  ga -->|"CNAME (TTL 900)"| aws["a7de0457831fd11f7.awsglobalaccelerator.com"]
  aws -->|"A (TTL 300)"| ip1["13.248.158.7"]
  aws -->|"A (TTL 300)"| ip2["76.223.84.192"]
```

**Why do this?**

- Instead of updating **50+ DNS zones**, one per country domain, Yahoo only maintains the records of `rc.yahoo.com`.
- Even better, those addresses can be spread across Yahoo's infrastructure with **load balancing and geolocation routing**.
- **CNAME chaining** lets Yahoo hand responsibility to another company's DNS name (AWS, Cloudflare, Akamai) instead of hard-coding IP addresses.
- **Flexibility:** if AWS changes the addresses behind its name, Yahoo does not have to touch its own DNS.

> **Kid example.** Fifty classrooms each put up a sign "for lost property, go to the office". If the lost-property box moves, only the office needs to know; the fifty signs never change. And the office may even say "we now use the town's lost-property service", and that is fine too.

#### CNAME rules †

*(This subsection is hidden material †.)*

- **A CNAME precludes any other RRs for the same name.** You might want `david.com CNAME david.stanford.edu`, but this is **illegal**, because `david.com` is a zone and needs NS records (and an SOA) at that same name.
- The **answer section** of a reply *can* contain the CNAME for the query name **plus** other RRs for the canonical name. That is how the Yahoo chain above arrives in one reply.
- **Don't point MX records at CNAMEs**, because the server then puts no A records in the additional section. (The lecture suggests trying `bad-mx.scs.stanford.edu.`)

**Additional context.** The rule "one name cannot have both a CNAME and other records" is why you cannot put a CNAME at a zone's top (the "apex", like `example.com` itself). Some DNS providers offer non-standard workarounds called ALIAS or ANAME records.

**Remember:**
- **CNAME** = alias (nickname) → canonical (real) name.
- A name with a CNAME may have **no other records**; so no CNAME at a zone apex.
- Many names can point to one canonical name (Yahoo: 50+ country domains → `rc.yahoo.com`).
- CNAME chains can hand the job to another company (AWS).
- Do not point MX records at CNAMEs.

### 3.7 Reverse lookups: PTR records

**In one sentence:** a PTR record goes backwards, from an IP address to a name, by writing the address backwards under `in-addr.arpa`.

**Remember traceroute?** It shows the **names** of routers on a path, which it learns through **reverse lookups**: going from an address back to a name.

- The address `128.30.2.121` is looked up as the name `121.2.30.128.in-addr.arpa`.
- A **PTR record** at that name points to the canonical name.

The lecture's example:

```
tinyos.stanford.edu         -> sing.stanford.edu    (CNAME)
sing.stanford.edu           -> 171.67.76.65         (A)
65.76.67.171.in-addr.arpa   -> sing.stanford.edu    (PTR)
```

The forward and reverse directions agree: the PTR points to the canonical name `sing`, not to the alias `tinyos`.

```mermaid
flowchart LR
    t["tinyos.stanford.edu"] -->|"CNAME"| s["sing.stanford.edu"]
    s -->|"A"| ip["171.67.76.65"]
    r["65.76.67.171.in-addr.arpa"] -->|"PTR"| s
```

> **Kid example.** Normally you look up a name to find a phone number. A reverse phone book works the other way: you see a number on your phone and look up whose it is. PTR records are DNS's reverse phone book.

#### Mapping addresses to names

`name [TTL] [IN] PTR ptrdname`

- `name` must somehow encode the address. How?
- `ptrdname` is the domain name for this address.
- **IPv4 addresses are stored under `in-addr.arpa`**: reverse the four numbers and append `in-addr.arpa`. To look up `171.66.3.9`, ask for `9.3.66.171.in-addr.arpa.`
- **IPv6 addresses are stored under `ip6.arpa`.**

**Why reversed? Delegation!** DNS names get more specific from **right to left** (`www.chalmers.se`), but IP addresses get more specific from **left to right** (`171.66.x.x` is a network, `171.66.3.9` is one host). Reversing the numbers puts the network part on the right, where the general part of a DNS name belongs. Then the whole subtree `66.171.in-addr.arpa` can be **delegated** to whoever owns the network `171.66.0.0/16`.

```mermaid
flowchart TD
  arpa["arpa"] --> inaddr["in-addr.arpa"]
  arpa --> ip6["ip6.arpa"]
  inaddr --> n171["171.in-addr.arpa"]
  n171 --> n66["66.171.in-addr.arpa"]
  subgraph owner["delegated to the owner of 171.66.0.0/16"]
    n66 --> n3["3.66.171.in-addr.arpa"]
    n3 --> n9["9.3.66.171.in-addr.arpa: PTR record"]
  end
```

> **Kid example.** A date like 2026-10-21 goes from big (year) to small (day). A DNS name goes the other way: small on the left, big on the right. To file an IP address in the DNS tree, you simply turn it around so the "big" part (the network) ends up on the right, like a DNS name.

**Historical note.** ARPA (the US Advanced Research Projects Agency) funded the original Internet. The acronym has since been **re-purposed** (RFC 3172) as **Address and Routing Parameter Area**.

**Additional context.** `dig -x 129.16.212.14` is a shortcut that asks for the PTR record of `14.212.16.129.in-addr.arpa` (this appears in a 2015 exam). In `ip6.arpa`, an IPv6 address is reversed **one hexadecimal digit at a time**, so a reverse name has 32 labels.

**Remember:**
- **PTR** = address → name (reverse lookup), used for example by traceroute.
- IPv4: **reverse the four numbers** + `.in-addr.arpa`; IPv6: reverse **each hex digit** + `.ip6.arpa`.
- Reversing makes **delegation** to the network owner possible.
- `dig -x` does a reverse lookup for you.

### 3.8 Using DNS for load balancing

**In one sentence:** give one name several addresses and shuffle their order in every answer, and clients spread themselves over all the servers.

- You can have **multiple RRs of most types for one name**.
  - This is **required** for NS records (for availability).
  - It is **useful** for A records.
  - It is **not legal** for CNAME records (a name can have only one CNAME).
- **Servers rotate the order** in which records are returned.
- `getaddrinfo()` returns a linked list of `addrinfo` structures, and **most apps just use the first address returned**.
- So even if your name server caches the results, **clients are spread among the servers**.
- Try it: run `dig cnn.com` several times and watch the order change.

```mermaid
sequenceDiagram
  participant C1 as Client 1
  participant C2 as Client 2
  participant R as Resolver
  C1->>R: www.example.com?
  R-->>C1: A1, A2, A3 (client 1 uses A1)
  C2->>R: www.example.com?
  R-->>C2: A2, A3, A1 (client 2 uses A2)
```

This is called **round-robin DNS**. It is cheap, but it cannot see whether a server is overloaded or down; a dead address keeps being handed out until someone removes it (and caches expire).

> **Kid example.** A pizza chain has one phone number but four kitchens. The person who answers reads the kitchen list in a different order to each caller, and most callers just go to the first kitchen they hear. So the customers spread out over all four kitchens. But if one kitchen burns down, the list is still read out with it, until someone crosses it off.

**Remember:**
- Several records per name: **required** for NS, **useful** for A, **illegal** for CNAME.
- Servers **rotate** the order; apps use the **first** address → load is spread (**round-robin DNS**).
- Weakness: it does not know if a server is down or overloaded.

### 3.9 SRV and TXT records

**In one sentence:** SRV says "this service runs on this host and port", and TXT lets you store any text, for example a list of servers allowed to send your e-mail.

#### SRV records (service location)

`_service._proto.name [TTL] [IN] SRV prio weight port target`

| Part | Meaning |
|---|---|
| `_service` | The service, for example `_sip` for SIP (voice over IP). |
| `_proto` | `_tcp` or `_udp`. |
| `name` | The domain the record applies to. |
| `prio` | Priority. As with MX records, a **lower number means higher priority**. |
| `weight` | Within one priority, affects the randomization of the order (higher weight = chosen more often). |
| `port` | The TCP or UDP port number (particularly useful for SIP). |
| `target` | The server's name; the client then needs its A record. |

SRV is like a **generalization of MX records to any service**. Example from the lecture: `dig _sip._tls.microsoft.com SRV`. The lecture gives `_sip._tcp.example.com. 3600 IN SRV 10 5 5060 sipserver.example.com.`, meaning: priority 10, weight 5, port 5060, on host `sipserver.example.com`.

```mermaid
flowchart LR
    Q["_sip._tcp.example.com"] -->|"SRV 10 5 5060"| T["sipserver.example.com<br/>port 5060"]
    T -->|"A record"| IP["IP address"]
```

> **Kid example.** An MX record is like a sign "post goes to the mail room". An SRV record is the same kind of sign for **anything**: "phone calls go to room 12, door 5060", "games go to room 7, door 3074".

**Technical clarification.** In `_sip._tls.microsoft.com`, the protocol label is `_tls`, although the lecture says `_proto` is `_tcp` or `_udp`. SIP over TLS is a real convention that uses `_tls` here; the lecture's rule describes the general case.

#### TXT records

`name [TTL] [IN] TXT "text"`

- You can put **arbitrary text** in DNS; the text means whatever you want it to mean.
- This is **great for prototyping new services**, because you **don't need to change the DNS infrastructure**.

**Example: `dig gmail.com txt`.** The answer is an **SPF** record ("Sender Policy Framework", previously "sender permitted from"). Much spam is forged e-mail, so SPF lists the IP addresses that are allowed to send mail from `@gmail.com`:

```
gmail.com. 3600 IN TXT "v=spf1 ip4:216.239.32.0/19 ip4:64.233.160.0/19 ip4:66.249.80.0/20 ~all"
```

A receiving mail server trusts mail from those ranges. The final `all` term says what to do with mail from **any other** address.

> **Kid example.** A TXT record is a sticky note on your front door. You can write anything on it. SPF uses it to say "only the postmen from these three post offices are allowed to deliver letters signed with my name; be suspicious of anyone else".

**Technical clarification.** The lecture's record ends in `~all`, but the slide text says "otherwise (`?all`) it is suspicious (SoftFail: may be marked spam)", and the lecture's explanation shows a version ending in `?all`. In SPF (RFC 7208) these are different: **`~all` = SoftFail** ("probably not allowed; accept but mark as suspicious"), while **`?all` = Neutral** ("no statement"). Other qualifiers: `-all` = Fail (reject), `+all` = Pass. So the lecture's explanation fits `~all`, not `?all`.

**Additional context.** A 2023–2025 multiple-choice question asks for the difference between SRV and TXT: SRV gives the server (host and port) for a specific service; TXT carries extra policy or configuration text, such as SPF rules.

**Remember:**
- **SRV** = host + port for a service; lower priority number first; weight spreads load.
- **TXT** = free text; used for **SPF** and for prototyping new services.
- SPF endings: `~all` SoftFail, `?all` Neutral, `-all` Fail, `+all` Pass.

### 3.10 DNS overview: what Part A gives us

**In one sentence:** DNS is a huge, shared, cached name-to-value system that works well, but it cannot tell a true answer from a false one.

- DNS is a **distributed system for mapping names to values** (for example IP addresses).
- A **read-dominated workload allows caching**.
- The **name structure allows distribution** and independent administration.
- **Caching means bad data can stay a long time.**
- The **standard protocol does not authenticate** that a response comes from the right server; **DNSSEC does**.

The last two points are where Part B begins.

**Remember:**
- Caching makes DNS fast, but also keeps **bad data** around.
- Plain DNS does **not** check who sent an answer; **DNSSEC** adds that.

## 4. Security and privacy

**What this chapter is about:** plain DNS can be lied to and can be read by anyone on the way. This chapter shows the attacks, the fix for lying (DNSSEC, signatures), the fix for reading (encrypted DNS), and why both fixes are spreading slowly.

### 4.1 What DNS never promised

**In one sentence:** the original DNS checks neither that an answer is true (integrity) nor that nobody can read your question (confidentiality).

- The original design guarantees **neither integrity nor confidentiality**.
  - **Integrity** = is the answer true and unchanged?
  - **Confidentiality** = can others read my question?
- **Any answer that arrives with the right transaction ID is believed.**
- **Queries cross the network in plain text**, visible to every hop.
- Both properties were **retrofitted, separately, decades apart**.

The lecture opens with: "We have just spent forty minutes on how DNS finds an answer. Now: how do you know the answer is true?"

DNS was specified in 1983–87 for a network of a few thousand hosts run by people who knew each other. Under those conditions authentication is a cost with no benefit. So the protocol has **no notion of a wrong answer**: a resolver accepts any UDP datagram that arrives on the right port, from the right address, with a matching 16-bit transaction ID and question. There is no signature and no channel security. Every resolver, network and observer on the path sees the full query, which means they see the name of every service each user contacts.

The two gaps were filled by **different mechanisms**, and students mix them up constantly:

| Gap | Fixed by | From about |
|---|---|---|
| Integrity | DNSSEC (signatures) | late 1990s |
| Confidentiality | Encrypted transports (DoT, DoH, DoQ) | 2016 |

```mermaid
flowchart TB
    G["Plain DNS (1983–87)<br/>no integrity, no confidentiality"]
    G -->|"is the answer true?"| I["Integrity gap<br/>fixed by DNSSEC (signatures)<br/>late 1990s"]
    G -->|"can others read my question?"| C["Confidentiality gap<br/>fixed by DoT, DoH, DoQ (encryption)<br/>from 2016"]
```

> **Kid example.** Plain DNS is like asking a question by shouting across the playground and believing whoever shouts back first with your ticket number. Anyone can hear your question (no confidentiality), and anyone can shout a fake answer (no integrity).

**Link back:** this is the same story as BGP in the routing lecture: a trusting design from an era of known operators, secured afterwards by bolting on a public-key infrastructure (RPKI there, DNSSEC here).

**Remember:**
- Plain DNS has **no integrity** and **no confidentiality**.
- An answer is believed if port, address, **16-bit ID** and question match.
- **Integrity → DNSSEC**; **confidentiality → DoT/DoH/DoQ**. Two different fixes.

### 4.2 Cache poisoning and the Kaminsky attack

**In one sentence:** an attacker races the real server with fake answers, and if one fake answer wins, the resolver remembers it and gives it to all its users.

- A resolver **caches whatever answer it accepts, and serves it to everyone.**
- **Kaminsky (2008):** forge answers faster than the legitimate reply arrives.
- Only a **16-bit transaction ID** stood in the way.
- **Source-port randomisation raised the cost. It did not fix the design.**

Caching is what makes DNS scale, and it is also what makes the attack worth doing: **poison one resolver and you have poisoned every user behind it, for the lifetime of the TTL.**

**The mechanics.** The attacker makes the resolver send a query, then races the real authoritative server with forged replies. A forgery wins if it has the right source address, port, transaction ID and question. In 2008 the only real unknown was the 16-bit ID: about 65,000 possibilities.

**Kaminsky's contribution** was not a new forgery technique but a way to **retry forever**. Ask about names that do not exist (`aaa1.bank.com`, `aaa2.bank.com`, …). Each one starts a fresh race, and the forged reply carries an **authority section that redirects the whole zone** (`bank.com`) to the attacker's server. Losing a race costs the attacker nothing, so sooner or later one forgery wins.

```mermaid
sequenceDiagram
  participant X as Attacker
  participant R as Resolver
  participant A as Real server for bank.com
  X->>R: query aaa1.bank.com (a name that does not exist)
  R->>A: query aaa1.bank.com, ID = ?
  X-->>R: many forged replies with guessed IDs
  A-->>R: real reply (too late if a forgery matched)
  Note over R: one match = false delegation cached for every user
  X->>R: query aaa2.bank.com (try again if it lost)
```

The forged replies say "`bank.com` is served by the attacker's server". If one of them has the right ID, the resolver believes it.

> **Kid example.** Cache poisoning is like someone sneaking a false phone number into the class phone list. Everyone who looks at the list afterwards calls the wrong number, until the list is thrown away and written again (the TTL runs out). The prankster only needs to win the race to the list once. Kaminsky's trick: keep asking about new, made-up classmates ("aaa1", "aaa2", …), so every question is a new race, and losing costs nothing.

**The response: source-port randomisation.** The resolver also picks a random source port for each query, which adds about 16 more bits to guess, so roughly 4 billion combinations instead of 65,000.

```mermaid
flowchart LR
    B["Before: guess the ID<br/>16 bits ≈ 65,000 tries"] -->|"add a random<br/>source port (≈ 16 bits)"| A["After: guess ID and port<br/>32 bits ≈ 4 billion tries"]
```

**The point the lecture insists on:** that is a **mitigation, not a fix**. It makes the attack expensive; it does not let a resolver tell a true answer from a false one. Nothing is authenticated. "What would it take to actually verify an answer?" Signatures, which is the next section.

**Remember:**
- Poison one resolver → **every user** behind it gets the false answer, **for the whole TTL**.
- Kaminsky (2008): query **non-existent names** so each try is a new race; the fake reply redirects the **whole zone**.
- 16-bit ID ≈ 65,000 guesses; with **source-port randomisation** ≈ 4 billion.
- That is a **mitigation, not a fix**: nothing is authenticated.

### 4.3 DNSSEC: a chain of trust

**In one sentence:** DNSSEC adds signatures to DNS, and each parent zone vouches for its child's key, so a resolver can check every step from the root down.

- **RRSIG** signs a record set; **DNSKEY** holds the signing (public) key.
- **DS** in the **parent** zone commits to the child's key, so each delegation becomes a link in a chain.
- **Validation walks the same path as resolution**, checking signatures at each step.
- **The chain is only as long as the delegations that are signed.**

| Record | Lives in | What it is |
|---|---|---|
| RRSIG | the zone itself | A signature over a set of records (for example all A records of a name). |
| DNSKEY | the zone itself | The public key that verifies the RRSIGs. |
| DS | the **parent** zone | A hash of the child's key: the parent vouching for the child. |

Delegation already forms a chain from root to TLD to zone. DNSSEC **hangs a cryptographic commitment on each existing link**. A validating resolver walks the same path it was walking anyway, and at each step checks:

1. that the parent's **DS** matches the child's **DNSKEY**, and then
2. that the **DNSKEY** verifies the **RRSIG** over the records.

```mermaid
flowchart TD
  anchor["Root trust anchor (root KSK),<br/>shipped with the resolver"]
  subgraph root["root zone"]
    rk["DNSKEY of root"]
    rds["DS for se<br/>(signed by RRSIG)"]
  end
  subgraph se[".se zone"]
    sk["DNSKEY of se"]
    sds["DS for chalmers.se<br/>(signed by RRSIG)"]
  end
  subgraph ch["chalmers.se zone"]
    ck["DNSKEY of chalmers.se"]
    rec["A record of www.chalmers.se<br/>(signed by RRSIG)"]
  end
  anchor -->|"verifies"| rk
  rk -->|"verifies the RRSIG over"| rds
  rds -->|"hash matches"| sk
  sk -->|"verifies the RRSIG over"| sds
  sds -->|"hash matches"| ck
  ck -->|"verifies the RRSIG over"| rec
```

Read the picture from the top: the root key you already trust checks the root's DS for `.se`; that DS matches `.se`'s key; `.se`'s key checks the DS for `chalmers.se`; and so on down to the address you asked for.

The **only thing you must trust in advance is the root key**: a single trust anchor, shipped with the resolver. Everything else follows from it.

> **Kid example.** DNSSEC is like a chain of signed permission slips. The head teacher (the root) signs a slip saying "this is the real stamp of class 5B's teacher" (the DS record). The class teacher stamps the answers with that stamp (RRSIG). You trust the answer because you can follow the signatures all the way up to the head teacher, whose signature you already know.

**Two misconceptions to avoid:**

1. **DNSSEC does not encrypt.** Every record is still public, just signed.
2. **A signed answer is not automatically a secure answer.** If any link in the chain is unsigned, the chain simply stops, and the answer is **"insecure"** (not checked), rather than **"bad"** (proved false).

**Additional context.** In practice a zone usually has two kinds of key: a **KSK** (key signing key) that signs only the DNSKEY set, and a **ZSK** (zone signing key) that signs all the other records. The DS record in the parent points at the KSK. This split lets the busy ZSK be changed often without involving the parent. The root KSK in [section 4.5](#45-the-root-ksk-rollover) is "the key that signs the key that signs the root zone".

```mermaid
flowchart LR
    DS["DS in the parent"] -->|"points at"| KSK["KSK"]
    KSK -->|"signs"| SET["the DNSKEY set"]
    SET -->|"contains"| ZSK["ZSK"]
    ZSK -->|"signs"| RR["all other records"]
```

**Remember:**
- **RRSIG** = signature; **DNSKEY** = public key; **DS** (in the parent) = hash of the child's key.
- Check at each step: **DS matches DNSKEY**, then **DNSKEY verifies RRSIG**.
- Only the **root key** is trusted in advance (trust anchor).
- DNSSEC **does not encrypt**; an unsigned link gives **"insecure"**, not "bad".
- **KSK** signs the keys; **ZSK** signs the records.

### 4.4 Signing and validating are different curves

**In one sentence:** zone owners decide to sign, resolver operators decide to check, and because neither gains much alone, the two numbers grow at very different speeds.

- **Signing** a zone is the **zone owner's** decision.
- **Validating** an answer is the **resolver operator's** decision.
- **Neither requires the other**, and the two adoption rates differ sharply.
- **Roughly a third of queries are validated; far fewer zones are signed.**

Neither party gets much benefit from acting alone:

- Sign your zone while nobody validates, and you have paid for key management to protect no one.
- Validate while few zones are signed, and most answers come back "insecure": complexity without coverage.

Validation is near a third of queries because **a handful of large public resolvers** flipped one setting. Signed zones are a much smaller share because signing needs effort from **millions of separate zone owners**. One curve moved because a few operators acted; the other has to wait for everybody. (The lecturer is told to re-pull current figures from APNIC Labs and date them.)

```mermaid
flowchart LR
    v1["Validating:<br/>a few big public resolvers"] --> v2["flip one setting"] --> v3["about 1/3 of<br/>queries validated"]
    s1["Signing:<br/>millions of zone owners"] --> s2["each must do<br/>their own work"] --> s3["far fewer<br/>zones signed"]
```

> **Kid example.** Imagine a rule that every lunchbox must have a name sticker (signing), and teachers should check the stickers (validating). Three teachers can decide in one afternoon to start checking. But getting a thousand families to all put stickers on takes years. And nobody bothers much while the other side is not doing it yet.

**Link back — the important bit:** this is the same shape as **RPKI** in the routing lecture: prefixes signed at roughly 67%, enforcement near 12%. Different protocol, different decade, identical structure: **the party who pays and the party who benefits are not the same party.** The lecture says you will see it a third time with BBR (TCP congestion control).

**Remember:**
- **Signing** = zone owner; **validating** = resolver operator; independent decisions.
- About **one third** of queries are validated; **far fewer** zones are signed.
- Same pattern as **RPKI**: the one who pays is not the one who benefits.

### 4.5 The root KSK rollover

**In one sentence:** the one key at the top of DNSSEC is being replaced, and millions of resolvers must learn the new key without anyone being able to tell them directly.

- **11 October 2026:** KSK-2024 becomes the active signing key for the root zone.
- **11 January 2027:** KSK-2017 is revoked.
- ICANN published operator guidance on **11 August 2026**.
- This is happening **during this course**. (The lecture says to verify the current status at iana.org, because dates have moved before.)

The **root KSK** is the trust anchor for the whole DNSSEC hierarchy. Every validating resolver on the planet holds a copy. It signs the key that signs the root zone, and everything else chains from it.

Keys must be rotated: to limit exposure, to prove the procedure works, and to move to stronger parameters. The current key, KSK-2017, has been in place since 2017 and is being replaced by KSK-2024.

**The difficulty:** this is not a software update anyone can push. There is no channel from ICANN to every resolver on Earth. The new key must reach millions of independently run machines with **no central coordination** and **no way to confirm** they got it.

```mermaid
timeline
  title Root KSK rollover dates (from the lecture)
  2017 : KSK-2017 in use
  11 Aug 2026 : ICANN operator guidance
  11 Oct 2026 : KSK-2024 starts signing
  11 Jan 2027 : KSK-2017 revoked
```

> **Kid example.** The head teacher's signature is changing. Every classroom in the world has a copy of the old signature on the wall, to check permission slips. Nobody can visit every classroom. So for a while, the head teacher signs a note **with the old signature** saying "this is my new signature", and pins it up where everyone can see it.

#### RFC 5011: changing a key without a flag day

A **flag day** is a moment when everybody must switch at once. RFC 5011 avoids one.

- Every validating resolver on Earth holds the root trust anchor.
- **No central mechanism exists to push a new one.**
- RFC 5011: the new key is **published alongside the old one, signed by it**, and adopted **after a hold-down period**.
- "Distributed consensus by patience, with no coordinator."

The lecture calls this "the most important slide in the lecture for the second half of the course".

**The problem:** you must update state held by millions of independent machines. You cannot contact them, you do not know who they are, and you cannot tell whether any one of them has updated. And a resolver that ends up with the wrong key **fails closed**: every validated answer breaks.

**The mechanism, step by step:**

1. Publish the new key in the root DNSKEY set **next to the old one, signed by the old one**.
2. Resolvers that poll the root see a new key that validates under the anchor they already trust.
3. They do **not** adopt it at once. They start a **hold-down timer of 30 days**.
4. Only if the new key stays **continuously present and valid for the whole period** do they add it as a trust anchor.
5. Later, the old key is published with the **REVOKE bit** set, signed by itself, and resolvers remove it.

```mermaid
stateDiagram-v2
  [*] --> Unknown: new key not yet seen
  Unknown --> HoldDown: new key appears, validly signed by the trusted old key
  HoldDown --> Unknown: key disappears before 30 days
  HoldDown --> Trusted: present and valid for 30 days
  Trusted --> Revoked: key seen with the REVOKE bit, self-signed
  Revoked --> [*]
```

> **Kid example.** You see the note "this is my new signature" signed with the old signature. You do not believe it straight away. You wait 30 days, checking every day that the note is still there. A prankster could pin up a fake note for a day, but not keep it up for a whole month without the head teacher noticing. After 30 days you accept the new signature.

**Why this matters for the course:** no leader, no synchrony assumption, no acknowledgements. Correctness rests on every participant **eventually observing the same published state** and applying the **same local rule**. Resolvers that were offline catch up when they return, provided they were not away longer than the window. "This is a distributed algorithm, running on the real Internet, right now. In the second half of this course you will learn to reason about exactly this kind of system — and to prove that it converges."

Question to think about: *what happens to a resolver that was switched off for the whole period?* It never saw the new key validated by the old one, so it cannot follow the chain: that is the failure case of the next part.

#### What could go wrong

- A resolver that **misses the update fails closed**: every validated answer breaks.
- Operators **cannot list which resolvers are ready**.
- The **2017 rollover was postponed a full year** over exactly this uncertainty.
- "A correct protocol, delayed by inability to observe global state."

The failure mode is severe and asymmetric. A resolver holding only the old key, once the root is signed by the new one, cannot validate anything. It does not fall back to "insecure"; it **fails closed**, and every user behind it loses DNS entirely.

**The history.** The first root KSK rollover was scheduled for **11 October 2017**. Weeks before, telemetry from **RFC 8145** (in which resolvers report which trust anchors they hold) suggested that a significant number were not ready. ICANN could not tell whether that meant truly unprepared resolvers or a reporting artifact. Unable to distinguish, they **postponed by a full year**. The rollover finally happened in **October 2018**, without major incident.

**The lesson:** the protocol was correct; RFC 5011 worked. What failed was **observability**: the operators could not see the global state well enough to know whether it was safe to go ahead, so they waited. In the second half of the course you meet systems that must converge **without any participant knowing the global state**.

**Remember:**
- Dates: **11 Aug 2026** guidance, **11 Oct 2026** KSK-2024 signs, **11 Jan 2027** KSK-2017 revoked.
- **RFC 5011:** new key published **signed by the old one**, **30-day hold-down**, old key later **REVOKE**d.
- No leader, no acknowledgements: a real distributed algorithm.
- A resolver with only the old key **fails closed** (loses DNS completely).
- The first rollover was **postponed from Oct 2017 to Oct 2018** because nobody could see the global state.

### 4.6 Sweden as an outlier

**In one sentence:** Sweden validates DNSSEC much more than most countries, and the reason is rules and market structure, not technology.

- Scandinavian validation rates are **among the highest measured anywhere**.
- Global validation is near a third of queries.
- **Why here?** Registry policy, a concentrated resolver market, early `.se` adoption.
- **Deployment outcomes are institutional, not just technical.**
- Current figure: `stats.labs.apnic.net/dnssec/SE` (the lecturer is told to pull the live number and date it).

Nothing technical explains Sweden's high rate: the same software and standards exist everywhere. The explanation is institutional:

- The **`.se` registry** was an early and active DNSSEC adopter, at one point offering **discounts for signed domains**.
- The Swedish **resolver market is concentrated** in a few operators, so a handful of decisions move the national figure.
- **Public-sector requirements** pushed adoption in a way market forces did not.

> **Kid example.** Every school in the world can buy the same bicycle helmets. But in one town, the shop gives a discount on helmets, there are only three schools, and the town council says "helmets please". So almost every child there wears one. The helmets are not better; the town is organised differently.

"Who is your resolver right now?" Most people do not know: eduroam, mobile operator or home ISP, and whether their answers are validated differs. The general lesson: deployment of Internet protocols is usually decided by registries, regulators and market structure more than by protocol design. RPKI adoption also varies sharply by region.

**Remember:**
- Sweden's validation rate is among the **highest in the world**.
- Reasons: **`.se` registry** (early, discounts), **few resolver operators**, **public-sector requirements**.
- Deployment is decided by **institutions**, not just technology.

### 4.7 Encrypting the query: DoT, DoH and DoQ

**In one sentence:** DoT, DoH and DoQ put DNS questions inside an encrypted connection, so people on the network path cannot read them.

| Transport | RFC | Port | Key property |
|---|---|---|---|
| **DoT**, DNS over TLS | 7858 | 853 (its own port) | Visible **as DNS** (the port gives it away), but the contents are hidden. |
| **DoH**, DNS over HTTPS | 8484 | 443 (the web's port) | Deliberately **indistinguishable from web traffic**. |
| **DoQ**, DNS over QUIC | 9250 | — | Encryption **without head-of-line blocking**. |

**Integrity and confidentiality are orthogonal** (independent): **DNSSEC does one, these do the other.**

- DNSSEC **signs**; it does not hide.
- These transports **hide**; they do not sign.
- A DoH query to an unsigned zone is **confidential and unauthenticated**.
- A DNSSEC-validated answer over plain UDP is **authenticated and public**.

```mermaid
quadrantChart
  title Two independent protections
  x-axis "Not encrypted" --> "Encrypted (DoT, DoH, DoQ)"
  y-axis "Not signed" --> "Signed and validated (DNSSEC)"
  quadrant-1 "private and authentic"
  quadrant-2 "authentic but public"
  quadrant-3 "plain DNS: neither"
  quadrant-4 "private but unauthenticated"
```

Read the square like a map: going right adds encryption, going up adds signatures. Only the top-right corner has both.

> **Kid example.** DNSSEC is like a postcard with a wax stamp: anyone can still read what is written, but nobody can change it without breaking the stamp. DoT and DoH are like an envelope you cannot see through: nobody on the way can read your question, but the envelope by itself does not prove who wrote the answer inside. You need both to be private **and** sure.

**DoT vs DoH, seen by a network operator.** With DoT, operators can still see that DNS traffic is happening and manage it (a feature for them), but users on networks that block port 853 cannot use it (a limitation). DoH looks like ordinary web traffic, which makes it **censorship-resistant** and makes network operators **unhappy**, for the same reason. "Why would a network operator object to DoH but not DoT?" Visibility and control.

> **Kid example.** DoT is a sealed letter in a bright red "DNS" envelope: the school guard cannot read it, but can see it is a DNS letter and stop all red envelopes. DoH is a sealed letter inside a normal white envelope, the same as millions of other letters (web traffic): the guard cannot even tell which ones are DNS.

**DoQ** uses QUIC. With TCP, one lost packet holds up everything behind it on the connection (head-of-line blocking); QUIC avoids this, which matters when one connection carries many queries at the same time.

**Additional context.** DoQ also uses port 853, but over UDP (QUIC runs on UDP).

**Remember:**
- **DoT**: TLS, port **853**, visible as DNS. **DoH**: HTTPS, port **443**, looks like web traffic. **DoQ**: QUIC (UDP port 853), no head-of-line blocking.
- Encryption = **confidentiality**; DNSSEC = **integrity**. They are **independent**.
- Network operators dislike DoH because they lose **visibility and control**.

### 4.8 Who are you hiding from?

**In one sentence:** encryption hides your questions from the network, but the resolver still sees all of them, so the real question is which resolver you choose to trust.

- Encrypting resolver traffic hides queries **from the network path**.
- It does **not** hide them **from the resolver**, which now sees more, from more users.
- DoH shifted resolution **from ISPs toward a handful of large operators**.
- "A privacy mechanism that concentrates the data it protects."

Encryption protects the query in transit. It does nothing at the endpoint: the resolver must read every name you look up in order to answer it. So the real question is not "is my DNS private?" but **"who am I choosing to trust?"**. Before DoH, that was usually your ISP: legally constrained, local, often uninterested. After DoH, browsers and operating systems began to default to a few large public resolvers: better resourced, often with better privacy policies, but **vastly larger in total**. The data concentrated, not through malice or bad design, but because of **where the encryption ends**. (This is a discussion slide; it has no clean answer.)

```mermaid
flowchart LR
    U["You"] -->|"encrypted:<br/>path cannot read"| R["Resolver<br/>reads everything"]
    R --> N["Name servers"]
```

> **Kid example.** You put your letter in a locked box so the postman cannot read it. But you are sending it to a friend who opens the box and reads everything. The lock protects you from the postman, not from the friend. And if everyone in town sends their letters to the same friend, that friend now knows a lot about everyone.

#### Minimising what anyone learns

A different strategy: rather than encrypting the channel, **give each party less in the first place**.

- **QNAME minimisation (RFC 9156):** send each server only the labels it needs. The root learns the TLD, not the full name.
- **Oblivious DoH:** separate *who is asking* from *what is asked*, using a relay.
- "Privacy by data minimisation rather than by encryption."

```mermaid
sequenceDiagram
  participant R as Resolver
  participant Root as Root server
  participant SE as .se server
  participant EX as example.se server
  Note over R,EX: classic: the full name www.example.se goes to every server
  Note over R,EX: with QNAME minimisation (RFC 9156)
  R->>Root: who serves se?
  Root-->>R: the .se servers
  R->>SE: who serves example.se?
  SE-->>R: the example.se servers
  R->>EX: www.example.se?
  EX-->>R: the answer
```

> **Kid example.** You want to find Sara's house. You do not tell the country office "I am looking for Sara at Main Street 5". You only ask "which way to Gothenburg?". In Gothenburg you only ask "which way to Main Street?". Only the last person learns the full address. That is QNAME minimisation.

```mermaid
flowchart LR
  U["User"] -->|"encrypted query<br/>(relay cannot read it)"| P["Relay:<br/>knows WHO asks"]
  P -->|"forwards, without<br/>the user's address"| T["Target resolver:<br/>knows WHAT is asked"]
  T -->|"encrypted answer"| P
  P -->|"encrypted answer"| U
```

Neither the relay nor the target alone can link the user to the query.

> **Kid example.** Oblivious DoH is like passing a sealed note through a friend. The friend knows the note is from you, but cannot read it. The teacher who reads it does not know who wrote it, because the friend handed it over. Nobody knows both "who" and "what".

**The principle:** encrypting a channel protects data from third parties but leaves the endpoint fully informed. **Minimisation protects data from the endpoint too.** It survives a compromised or merely curious resolver, which encryption does not. "Which would you rather have if you distrusted your resolver?"

**Additional context.** Oblivious DoH is specified in RFC 9230 (an experimental RFC from 2022).

**Remember:**
- Encryption hides queries from the **path**, not from the **resolver**.
- DoH moved queries from ISPs to **a few big resolvers**: data **concentrates**.
- **QNAME minimisation** (RFC 9156): each server learns only what it needs.
- **Oblivious DoH** (RFC 9230): relay knows **who**, target knows **what**, nobody knows both.

## 5. Resilience and failure

**What this chapter is about:** how the root servers survive failures (anycast), and the quiet ways DNS can be wrong without anyone noticing.

### 5.1 Anycast and root resilience

**In one sentence:** each root server name is really hundreds of machines sharing one address, and if one breaks, normal routing quietly sends you to another.

- **Thirteen root server identities; over a thousand physical instances.**
- **BGP anycast:** many sites announce the same prefix; routing picks the nearest.
- **A site failing is a route withdrawal**: queries move elsewhere without clients noticing.
- **Failure is handled by the routing system, not by DNS.**

**Clear up the myth:** there are **not** thirteen root servers. There are thirteen root server **identities**, named A to M, because thirteen addresses was what fitted in a 512-byte UDP response in 1987. Behind those thirteen names are well over a thousand machines worldwide.

**How:** many sites announce the same IP prefix into BGP from different places. Ordinary routing delivers each query to the instance closest in BGP terms. No DNS mechanism is involved: the resolver believes it is talking to one server.

```mermaid
flowchart LR
  subgraph sites["Many instances of one root identity, all announcing the same prefix"]
    s1["Instance in Stockholm"]
    s2["Instance in Frankfurt"]
    s3["Instance in Tokyo"]
  end
  r1["Resolver in Sweden"] -->|"BGP picks the nearest"| s1
  r2["Resolver in Japan"] -->|"BGP picks the nearest"| s3
  s1 -.->|"if it fails: route withdrawn,<br/>BGP reconverges"| s2
```

The dotted arrow shows what happens when the Stockholm instance fails: it withdraws its route, BGP re-converges, and the Swedish resolver's queries flow to the next-nearest instance. There is no failover logic in DNS, no health check and no timeout: **the routing system absorbs the failure**, because the address was never tied to one machine.

> **Kid example.** Anycast is like a chain of shops that all use the same phone number: your call goes to the nearest shop. If one shop burns down, the phone company simply sends the calls to the next nearest shop, and callers never notice.

Links made in the lecture: anycast as an IP-layer mechanism belongs to the IPv6 lecture; this is BGP (whose problems filled the routing lecture) used deliberately for fault tolerance; and anycast is also load balancing, an idea that returns as consistent hashing. This is "the one place in the lecture where DNS handles failure well — and it does so by delegating the problem to a different layer."

**Remember:**
- **13 root identities (A to M)**, over **1,000 machines**; 13 because that fitted in 512 bytes in 1987.
- **Anycast:** same prefix announced from many places; BGP picks the **nearest**.
- A failed site = a **route withdrawal**; **routing**, not DNS, handles the failure.

### 5.2 How DNS fails

**In one sentence:** DNS usually fails quietly: every part follows its rules correctly, and the answer is still wrong.

- **Stale delegation:** the parent points somewhere the child no longer is.
- **Expired signatures:** valid data rejected because time moved on.
- **A poisoned cache** serves wrong answers until the TTL expires, and cannot detect it.
- **No component knows it is wrong. Recovery is external, or it is time.**

| Failure | What happens | Why nobody notices |
|---|---|---|
| Stale delegation | The parent's NS records point at servers the child no longer uses. | Every component behaves correctly: the parent serves what it was configured with, the resolver follows it, the old server answers or does not. Nothing is "broken", and the answer is wrong. |
| Expired signatures | RRSIG records carry validity periods. Let one lapse and validating resolvers reject perfectly good data. | The zone and the key are intact; only the clock moved. Zones have taken themselves offline this way. |
| Poisoned cache | A resolver serves forged data confidently. | It has no way to detect the forgery. Recovery is not repair; it is waiting for the TTL to run out. |

> **Kid example.** Stale delegation: the school office still sends visitors to a classroom that moved last month. Expired signature: your bus pass is real, but the date ran out, so the driver says no. Poisoned cache: the class phone list has a wrong number, and everyone keeps calling it until the list is rewritten. In all three, every person is "following the rules", and still the result is wrong.

**The common thread:** in all three, **no component knows it is wrong**. There is no error, no alarm, no inconsistency that any single participant can see. The system is in a bad state while being **locally correct everywhere**.

```mermaid
flowchart LR
    A["Bad state<br/>(wrong answers)"] -->|"no alarm: every part<br/>is locally correct"| A2["stays in the<br/>bad state"]
    A2 -->|"only outside help<br/>or the TTL running out"| B["Correct state"]
```

**The hand-over to the second half of the course:** "That is now the fifth time. BGP wedgies converged to a state nobody wanted. IPv6 hides its own breakage. TCP settles into equilibria nobody chose. SDN cannot bootstrap in-band control after a fault. And DNS cannot tell that it is lying. Every one is a deployed system, locally correct, that can reach a bad state and cannot leave it unaided. The rest of this course is about systems that can." That is exactly what **self-stabilization** means: a system that returns to correct behaviour from **any** state, without outside help.

**Remember:**
- Three quiet failures: **stale delegation**, **expired signatures**, **poisoned cache**.
- **No component knows it is wrong**; the system is **locally correct everywhere**.
- Recovery needs outside help or time (TTL).
- This motivates **self-stabilization**: recover from any state without help.

## 6. Practice and review

**What this chapter is about:** how to read `dig` output in an exam, and short answers to quiz topics that the lecture does not cover.

### 6.1 Exam skill: reading dig output

**In one sentence:** "no answer + an authority section + extra A records" in a dig reply means a referral with glue.

**Additional context.** Old exams (2012–2015, October 2017, November 2018) ask you to read `dig` output. The 2026 exam pattern makes a long written question unlikely, but the facts behind it appear in multiple-choice questions. The output below is an **illustration**: the layout follows a real `dig` reply, but the addresses are made-up documentation addresses (192.0.2.x).

```
$ dig www.chalmers.se @a.ns.se
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4711
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 3

;; QUESTION SECTION:
;www.chalmers.se.          IN  A

;; AUTHORITY SECTION:
chalmers.se.     86400  IN  NS  ns1.chalmers.se.
chalmers.se.     86400  IN  NS  ns2.chalmers.se.
chalmers.se.     86400  IN  NS  ns3.chalmers.se.
chalmers.se.     86400  IN  NS  dns.uu.se.

;; ADDITIONAL SECTION:
ns1.chalmers.se. 86400  IN  A   192.0.2.1
ns2.chalmers.se. 86400  IN  A   192.0.2.2
ns3.chalmers.se. 86400  IN  A   192.0.2.3

;; SERVER: (the address of a.ns.se)#53
```

How to read it, step by step:

1. **Question:** type `A` for `www.chalmers.se` (dig asks for A when you give no type).
2. **Flags:** `qr` = this is a response; `rd` = the client asked for recursion; there is **no `ra`**, so this server does not do recursion; there is **no `aa`**, so it is **not authoritative** for this name.
3. **ANSWER: 0** but **AUTHORITY: 4**: this is a **referral** ("I don't know, ask these servers").
4. **ADDITIONAL** holds A records only for the three servers **inside** `chalmers.se`: those are the **glue**. `dns.uu.se` has none, because it can be found through `uu.se`.
5. **SERVER** shows which server answered (here `a.ns.se`, a server of the `.se` TLD).

```mermaid
flowchart TD
    Q{"ANSWER section empty?"}
    Q -->|"no"| ANS["a real answer<br/>(check aa: official or cached?)"]
    Q -->|"yes"| AU{"AUTHORITY has NS records?"}
    AU -->|"yes"| AD{"ADDITIONAL has A records<br/>for those NS names?"}
    AD -->|"yes"| RG["referral with glue"]
    AD -->|"no"| R["referral without glue"]
    AU -->|"no"| OTHER["no data or an error:<br/>look at status (RCODE)"]
```

> **Kid example.** Reading dig output is like reading a reply letter. "Answer: (empty)" means "I don't know". "Ask these four people" is the authority section. "Here are the phone numbers of three of them" is the additional section, the glue. The fourth person (in Uppsala) you can find yourself.

A useful rule from the 2018 exam guide: **"No answer + authority section + additional A records" means a referral with glue.** And when you ask `dig ns se.` the answer section already lists the NS records, so the authority section is empty: there is nothing new to put there.

**Remember:**
- Flags: `qr` response, `rd` recursion wanted, `ra` recursion available, `aa` authoritative.
- **ANSWER 0 + AUTHORITY NS + ADDITIONAL A = referral with glue.**
- Glue only for servers **inside** the delegated zone.

### 6.2 Related quiz topics not in the lecture

**Additional context.** The course's pre-lecture DNS quiz (31 questions) asks about a few things that the lecture does not cover directly. Short explanations:

- **Stub resolver.** The small DNS client in your laptop or app. It does no searching itself; it sends one recursive query (RD = 1) to the recursive resolver and waits.
- **TTL trade-off.** A **long TTL** means faster lookups and less load on servers, but a change (new IP address) takes longer to reach everyone. A **short TTL** means changes spread quickly, but more queries. When the TTL expires, the resolver throws the record away and asks again.
- **NXDOMAIN and negative caching.** NXDOMAIN means "this name does not exist". That answer is cached too, for a time taken from the zone's SOA record (RFC 2308), so a typo does not cause a new query every time.
- **Wildcards.** A record for `*.example.com` gives a made-up (synthesized) answer for any name under `example.com` that has no records of its own, such as `unknown.example.com`.
- **DNS redirection vs HTTP redirection.** DNS redirection simply gives a different IP address for the same name; the user does not see it and the URL in the browser stays the same. HTTP redirection happens after you connect: the web server answers "go to this other URL" (status 301 or 302), and the browser's address bar changes.
- **Dynamic DNS (DDNS) with DHCP.** When DHCP gives a device a new IP address, the device's DNS record is updated automatically, so its name keeps pointing at its current address.
- **Amplification attack.** The attacker sends a small query to an open resolver, with the **victim's** address forged as the source. The resolver sends a much larger reply to the victim. Small in, big out: many resolvers together can flood the victim. (It works because DNS over UDP does not check the source address.)
- **Public resolvers and privacy.** A public resolver like 8.8.8.8 sees every name you look up, so it could log and profile you. Encryption (DoT/DoH) hides queries from the network, not from the resolver ([section 4.8](#48-who-are-you-hiding-from)).
- **Exam-style summary of the design.** The property that gives DNS both fault tolerance and administrative scalability is **hierarchical delegation of zones combined with replication on several authoritative servers** (a 2023–2025 multiple-choice question).

```mermaid
sequenceDiagram
    participant X as Attacker
    participant O as Open resolver
    participant V as Victim
    X->>O: small query, forged source = victim's address
    O-->>V: big reply (the victim never asked)
```

> **Kid example.** Amplification is like ordering a hundred pizzas by phone and giving the victim's home address. The phone call was short, but the victim's door is flooded with deliveries they never asked for.

## Revision notes and references

Technical clarifications, where the lecture material is corrected or made precise, appear in sections [1.2](#12-before-dns-the-hoststxt-file), [2.2](#22-how-one-lookup-travels-recursive-and-non-recursive-queries), [2.3](#23-the-dns-protocol-port-53-udp-and-tcp), [3.1](#31-resource-records), [3.9](#39-srv-and-txt-records). Where the slides and the course book differ, this guide follows the book and says so at that point.

- [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034) and [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035): DNS concepts, records and message format.
- [RFC 2782 — SRV records](https://www.rfc-editor.org/rfc/rfc2782): service location records.
- [RFC 4033 — DNSSEC introduction and requirements](https://www.rfc-editor.org/rfc/rfc4033) and [RFC 5011 — automated updates of DNSSEC trust anchors](https://www.rfc-editor.org/rfc/rfc5011): the chain of trust and trust-anchor rollover.
- [RFC 7858 — DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858), [RFC 8484 — DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484) and [RFC 9250 — DNS over QUIC](https://www.rfc-editor.org/rfc/rfc9250): encrypted transports for DNS queries.
