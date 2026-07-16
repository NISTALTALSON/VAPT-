# Ports & Protocols — Deep Dive (Mastery Level)
### Companion Document to "The Complete Networking Guide"

---

## Table of Contents
1. What Is a Port? (Fundamentals)
2. Port Ranges & Classification
3. Master Port Reference Table
4. TCP — Deep Dive
5. The TCP 3-Way Handshake (Full Detail)
6. TCP Connection Termination (4-Way FIN)
7. TCP Header Structure & Flags
8. UDP — Deep Dive
9. UDP Communication Process
10. TCP vs UDP — Full Comparison
11. Protocol-by-Protocol Deep Dive
    - HTTP & HTTPS
    - DNS
    - SSH
    - Telnet
    - FTP
    - SMTP
    - POP3
    - IMAP
    - SNMP
    - ICMP
    - NTP
    - RDP
    - DHCP (recap)
12. How a Packet Actually Travels (End-to-End Example)
13. Quick-Reference Cheat Sheet

---

## 1. What Is a Port? (Fundamentals)

An **IP address** identifies *which device* on a network you're talking to. A **port** identifies *which application or service* on that device the data is meant for.

Think of it like an apartment building:
- The **IP address** = the building's street address
- The **port number** = the specific apartment/unit number inside

Without ports, a server wouldn't know whether incoming data was meant for its web server, email server, or file server — since all of that traffic arrives at the same IP address.

Every network connection is uniquely identified by a **socket**, made up of:
```
IP Address + Port Number = Socket
Example: 192.168.1.10:443
```

A full TCP/UDP connection is actually identified by **4 values** (called a "4-tuple" or "socket pair"):
```
Source IP : Source Port  ↔  Destination IP : Destination Port
```
This is how a server can handle thousands of simultaneous connections on the same port (e.g., port 443) — each client has a unique source IP/port combination.

---

## 2. Port Ranges & Classification

Ports are 16-bit numbers, so they range from **0 to 65535** (2^16 = 65536 total ports).

| Range | Name | Description |
|---|---|---|
| 0 – 1023 | **Well-Known Ports** | Reserved for standard, universally recognized services (HTTP, FTP, SSH, DNS). Require admin/root privileges to bind to on most OSes. |
| 1024 – 49151 | **Registered Ports** | Registered with IANA for specific applications/vendors (not as strictly controlled) |
| 49152 – 65535 | **Dynamic/Private/Ephemeral Ports** | Temporarily assigned by the OS to client-side connections; not permanently tied to any service |

**Ephemeral ports in action:** When your browser connects to a website, your OS randomly picks an ephemeral port (e.g., `54321`) as the *source port*, while the destination port is the well-known port of the service (e.g., `443` for HTTPS). This is how your PC can have many browser tabs open to different (or the same) sites simultaneously.

---

## 3. Master Port Reference Table

| Port | Protocol Type | Service | Description |
|---|---|---|---|
| 20 | TCP | FTP (Data) | File Transfer Protocol — actual data transfer |
| 21 | TCP | FTP (Control) | FTP — commands/control channel |
| 22 | TCP | SSH | Secure Shell — encrypted remote access |
| 23 | TCP | Telnet | Unencrypted remote access (insecure, legacy) |
| 25 | TCP | SMTP | Sending email between mail servers |
| 53 | TCP/UDP | DNS | Domain Name resolution (UDP for queries, TCP for zone transfers/large responses) |
| 67 | UDP | DHCP (Server) | DHCP server listens here |
| 68 | UDP | DHCP (Client) | DHCP client listens here |
| 69 | UDP | TFTP | Trivial File Transfer Protocol (simple, no auth) |
| 80 | TCP | HTTP | Unencrypted web traffic |
| 110 | TCP | POP3 | Retrieving email (downloads & often deletes from server) |
| 119 | TCP | NNTP | Network News Transfer Protocol (Usenet) |
| 123 | UDP | NTP | Network Time Protocol — clock synchronization |
| 143 | TCP | IMAP | Retrieving email (syncs, keeps mail on server) |
| 161 | UDP | SNMP | Simple Network Management Protocol (monitoring/queries) |
| 162 | UDP | SNMP Trap | SNMP server receiving alerts/traps from devices |
| 179 | TCP | BGP | Border Gateway Protocol — internet backbone routing |
| 389 | TCP/UDP | LDAP | Lightweight Directory Access Protocol (directory services, e.g., Active Directory) |
| 443 | TCP | HTTPS | Encrypted web traffic (HTTP over TLS/SSL) |
| 445 | TCP | SMB | Windows file/printer sharing |
| 514 | UDP | Syslog | System logging protocol |
| 587 | TCP | SMTP (Submission) | Modern encrypted email submission port (client → server) |
| 636 | TCP | LDAPS | LDAP over SSL |
| 993 | TCP | IMAPS | IMAP over SSL/TLS (encrypted) |
| 995 | TCP | POP3S | POP3 over SSL/TLS (encrypted) |
| 3306 | TCP | MySQL | Database connections |
| 3389 | TCP | RDP | Remote Desktop Protocol (Windows remote access) |
| 5060/5061 | TCP/UDP | SIP | Session Initiation Protocol (VoIP call setup) |
| 8080 | TCP | HTTP-Alt | Common alternate web/proxy port |

**Note on your list:** "smdp" and "iman" and "sm" and "euder" and "rdn" appear to be typos — I've covered them as **SMTP**, **IMAP**, **SNMP**, and **RDP** below, which are the standard networking protocols those likely refer to.

---

## 4. TCP — Deep Dive

**TCP (Transmission Control Protocol)** is a **connection-oriented**, **reliable**, byte-stream protocol. It guarantees that data arrives complete, in order, and error-checked — at the cost of speed/overhead.

### Core Guarantees TCP Provides
1. **Reliability** — lost packets are detected and retransmitted
2. **Ordered delivery** — packets are reassembled in the correct sequence even if they arrive out of order
3. **Error checking** — checksums detect corrupted data
4. **Flow control** — prevents a fast sender from overwhelming a slow receiver (via the "sliding window")
5. **Congestion control** — prevents overwhelming the network itself (via algorithms like slow start, congestion avoidance)

### Why TCP is "connection-oriented"
Before any actual data is sent, TCP performs a **handshake** to establish a virtual connection between both endpoints, and formally **tears it down** when finished. UDP does neither.

---

## 5. The TCP 3-Way Handshake (Full Detail)

This is the single most commonly tested networking concept in interviews and certifications — master it completely.

### The Process
```
Client                          Server
  |------ SYN (seq=x) --------->|     Step 1: Client requests connection
  |<--- SYN-ACK (seq=y,ack=x+1)-|     Step 2: Server acknowledges + requests back
  |------ ACK (ack=y+1) ------->|     Step 3: Client confirms — connection established
```

### Step-by-step breakdown
**Step 1 — SYN:**
The client sends a TCP segment with the **SYN** (synchronize) flag set, and an initial random **sequence number** (e.g., `seq=1000`). This says: "I want to start a connection, and my starting sequence number is 1000."

**Step 2 — SYN-ACK:**
The server responds with **both** the SYN and ACK flags set. It acknowledges the client's sequence number (`ack=1001`, i.e., client's seq+1) and includes its **own** random sequence number (e.g., `seq=5000`). This says: "I acknowledge your request, and here's MY starting sequence number."

**Step 3 — ACK:**
The client responds with just the **ACK** flag set, acknowledging the server's sequence number (`ack=5001`). This says: "Acknowledged — connection is now fully established."

After this handshake, both sides have confirmed each other's sequence numbers, and actual **data transfer begins.**

### Why sequence numbers matter
Every byte sent in a TCP stream is numbered. This lets the receiver:
- Reassemble packets in the correct order even if they arrive out of order
- Detect missing packets (gaps in sequence numbers) and request retransmission
- Detect duplicate packets and discard them

### Real-world analogy
It's like a phone call:
- "Hello, can you hear me?" (SYN)
- "Yes I can hear you, can you hear me?" (SYN-ACK)
- "Yes, I can hear you too." (ACK)
- *Now the actual conversation (data) begins.*

---

## 6. TCP Connection Termination (4-Way FIN)

Just as TCP formally opens a connection, it formally closes one — using a **4-way handshake** (sometimes appears as 3 packets if flags are combined).

```
Client                          Server
  |------ FIN (seq=x) --------->|     Step 1: Client wants to close
  |<--------- ACK --------------|     Step 2: Server acknowledges
  |<--------- FIN --------------|     Step 3: Server also wants to close
  |--------- ACK -------------->|     Step 4: Client acknowledges — closed
```

- Either side can initiate termination.
- Because TCP connections are **full-duplex** (data flows both directions independently), each direction must be closed separately — hence 4 steps instead of 2.
- After sending the final ACK, the client typically enters a **TIME_WAIT** state for a short period, to ensure any delayed/duplicate packets are handled before the connection is fully released.

### RST (Reset)
A TCP **RST** flag abruptly terminates a connection immediately (not gracefully), typically used when:
- A connection error occurs
- A port is closed/service unavailable and a SYN was sent to it
- An application forcibly aborts the connection

---

## 7. TCP Header Structure & Flags

Every TCP segment carries a header with key control fields:

| Field | Purpose |
|---|---|
| Source Port / Destination Port | Identifies sending/receiving application |
| Sequence Number | Tracks byte order for reassembly |
| Acknowledgment Number | Confirms received bytes |
| Window Size | Flow control — how much data receiver can accept before needing an ACK |
| Checksum | Error detection |
| Flags (control bits) | SYN, ACK, FIN, RST, PSH, URG (see below) |

### TCP Flags Explained
| Flag | Meaning |
|---|---|
| **SYN** | Synchronize — initiates a connection |
| **ACK** | Acknowledge — confirms received data/request |
| **FIN** | Finish — gracefully ends a connection |
| **RST** | Reset — abruptly terminates a connection |
| **PSH** | Push — tells receiver to pass data to the application immediately, don't buffer |
| **URG** | Urgent — marks data as high priority |

### Sliding Window (Flow Control)
The receiver tells the sender how much data (in bytes) it's willing to accept before requiring an acknowledgment — this "window size" dynamically adjusts, preventing the sender from flooding a slower receiver's buffer.

### Congestion Control (Brief)
TCP also self-regulates its sending rate to avoid overwhelming the *network* (not just the receiver), using algorithms like:
- **Slow Start** — begins sending slowly, doubles rate as ACKs confirm success
- **Congestion Avoidance** — once a threshold is hit, increases more conservatively
- **Fast Retransmit/Recovery** — quickly resends lost packets without waiting for a full timeout

---

## 8. UDP — Deep Dive

**UDP (User Datagram Protocol)** is a **connectionless**, lightweight protocol. It sends data ("datagrams") without establishing a connection, without guaranteeing delivery, order, or error recovery.

### Why use something "unreliable" on purpose?
Because reliability = overhead = latency. Applications where **speed matters more than perfection** prefer UDP:
- **VoIP calls / video calls** — a dropped packet just causes a tiny glitch; re-sending old audio data would be pointless (you'd rather skip it and keep talking in real time)
- **Live video streaming**
- **Online gaming** — you want the *current* position of a player, not a delayed retransmission of an old one
- **DNS queries** — small, quick request/response, retry is cheap if it fails
- **DHCP**
- **SNMP**
- **NTP**

### UDP Header (much simpler than TCP)
| Field | Purpose |
|---|---|
| Source Port | Sending application |
| Destination Port | Receiving application |
| Length | Size of the UDP datagram |
| Checksum | Basic error detection (optional in IPv4) |

That's it — **only 4 fields**, no sequence numbers, no acknowledgments, no handshake, no flow control.

---

## 9. UDP Communication Process

Unlike TCP's structured handshake, UDP communication is simply:

```
Client -------- Datagram sent -------> Server
(no handshake, no confirmation required, no guaranteed delivery)
```

**Step-by-step:**
1. The sending application hands data to UDP.
2. UDP wraps it with source/destination port info and a checksum — that's the entire "header" — and sends it immediately.
3. There is **no acknowledgment** — the sender doesn't know or care if it arrived.
4. If reliability is needed, **the application itself** must handle it (e.g., DNS will simply re-send a query if it times out waiting for a response; video call apps use their own internal logic for lost frames).

### "Fire and forget"
UDP is often described this way — packets are sent without any confirmation mechanism. This is precisely what makes it fast, but also means:
- Packets can arrive out of order
- Packets can be lost entirely with no automatic retransmission
- Packets can be duplicated

---

## 10. TCP vs UDP — Full Comparison

| Feature | TCP | UDP |
|---|---|---|
| Connection type | Connection-oriented | Connectionless |
| Handshake | 3-way handshake required | None |
| Reliability | Guaranteed delivery, retransmits lost data | No guarantee |
| Ordering | Packets reassembled in correct order | No ordering guarantee |
| Speed | Slower (overhead from handshake/ACKs) | Faster (minimal overhead) |
| Header size | 20+ bytes | 8 bytes |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes | No |
| Error recovery | Yes (retransmission) | No (app must handle it) |
| Use cases | Web browsing, email, file transfer, SSH | Streaming, gaming, VoIP, DNS, DHCP |
| PDU Name | Segment | Datagram |

**Rule of thumb:** If the data *must* arrive completely and correctly (a file, a webpage, an email) → **TCP**. If speed and real-time delivery matter more than perfection (a voice call, a live stream, a fast query) → **UDP**.

---

## 11. Protocol-by-Protocol Deep Dive

### HTTP & HTTPS
- **HTTP (Hypertext Transfer Protocol)** — Port 80, TCP. The foundation of data communication for the web. Stateless (each request is independent), request/response model (GET, POST, PUT, DELETE methods).
- **HTTPS** — Port 443, TCP. HTTP wrapped inside a **TLS/SSL encryption layer**. Before any HTTP data is exchanged, a **TLS handshake** occurs (separate from the TCP handshake) to negotiate encryption keys and verify the server's identity via a digital certificate.
- HTTPS protects against eavesdropping, tampering, and impersonation — essential for anything involving passwords, payments, or private data.

### DNS (Domain Name System)
- Port 53. Uses **UDP** for standard queries (fast, small requests) but falls back to **TCP** for larger responses (like zone transfers between DNS servers, or responses exceeding 512 bytes).
- Resolves domain names (`example.com`) into IP addresses.
- Hierarchical: Root → TLD (.com) → Authoritative server → Answer.

### SSH (Secure Shell)
- Port 22, TCP. Provides **encrypted** remote command-line access to a device — the secure replacement for Telnet.
- Uses public-key cryptography for authentication, and encrypts the entire session (commands, output, everything) so it cannot be intercepted/read in transit.
- Also used for secure file transfer (SFTP) and tunneling other traffic securely.

### Telnet
- Port 23, TCP. Legacy remote access protocol — functionally similar to SSH but sends **everything in plain text**, including usernames and passwords.
- **Insecure and obsolete** for production use — anyone capturing the traffic can read the entire session. Still occasionally used for testing raw connectivity to a specific port.

### FTP (File Transfer Protocol)
- Ports 20 (data) & 21 (control), TCP.
- Port 21 handles commands (login, navigation, file requests); Port 20 (in "active mode") handles the actual file transfer.
- Sends credentials in plain text by default — insecure unless combined with encryption (FTPS or replaced with SFTP, which actually runs over SSH on port 22).

### SMTP (Simple Mail Transfer Protocol)
- Port 25 (server-to-server relay) or Port 587 (modern client submission, typically encrypted), TCP.
- Used to **send** email — both from a mail client to a server, and between mail servers relaying messages to their final destination.
- SMTP does NOT retrieve/read mail — that's the job of POP3 or IMAP.

### POP3 (Post Office Protocol v3)
- Port 110 (or 995 encrypted), TCP.
- Used to **retrieve/download** email from a server to a single device.
- Classic behavior: downloads messages and **deletes them from the server** (though modern clients often configure it to leave a copy).
- Best suited for checking mail from a single device only — doesn't sync across multiple devices well.

### IMAP (Internet Message Access Protocol)
- Port 143 (or 993 encrypted), TCP.
- Also retrieves email, but **keeps mail synchronized on the server** — read/unread status, folders, and messages stay consistent across all devices (phone, laptop, webmail).
- This is why virtually all modern email setups use IMAP over POP3.

### SNMP (Simple Network Management Protocol)
- Port 161 (queries/polling) and 162 (traps/alerts), UDP.
- Used by network administrators to **monitor and manage** network devices (routers, switches, servers) — collects stats like bandwidth usage, uptime, temperature, errors.
- **SNMP Trap:** an unsolicited alert sent *from* a device *to* a monitoring server when something significant happens (e.g., interface down), rather than waiting to be polled.
- Uses "communities" (like passwords) in older versions (v1/v2c — weak security); SNMPv3 adds proper authentication and encryption.

### ICMP (Internet Control Message Protocol)
- No port number — ICMP operates directly at the **Network Layer (Layer 3)**, alongside IP, not on top of TCP/UDP.
- Used for diagnostics and error reporting, **not** for transferring application data.
- Powers tools like **ping** (Echo Request/Echo Reply messages) and **traceroute** (uses ICMP Time Exceeded messages, or UDP depending on OS).
- Common ICMP message types: Echo Request/Reply (ping), Destination Unreachable, Time Exceeded (TTL expired), Redirect.

### NTP (Network Time Protocol)
- Port 123, UDP.
- Synchronizes clocks across devices on a network to a highly accurate common time source (often referencing atomic clocks via a hierarchy of "stratum" levels).
- Critical for logging accuracy, security certificate validation, authentication protocols (like Kerberos), and distributed systems where timing matters.

### RDP (Remote Desktop Protocol)
- Port 3389, TCP.
- Microsoft's protocol for remotely accessing and controlling a Windows machine's full graphical desktop (not just command line, unlike SSH/Telnet).
- Encrypts the session and transmits screen, keyboard, and mouse data between client and host.

### DHCP (recap — see main guide Section 12 for full DORA process)
- Ports 67 (server) / 68 (client), UDP.
- Automatically assigns IP addresses and network configuration to devices joining a network.

---

## 12. How a Packet Actually Travels (End-to-End Example)

**Scenario:** You type `https://example.com` into your browser.

1. **DNS Resolution (UDP, port 53):** Your device asks a DNS resolver "what's the IP for example.com?" and gets back an IP like `93.184.216.34`.
2. **TCP 3-Way Handshake (port 443):** Your device performs SYN → SYN-ACK → ACK with the server to establish a connection.
3. **TLS Handshake:** Encryption keys are negotiated, and the server's SSL certificate is verified (this is what makes it HTTPS, not just HTTP).
4. **HTTP Request:** Your browser sends a `GET /` request over the now-encrypted connection.
5. **Server Response:** The server sends back the webpage data in TCP segments, each tracked via sequence numbers to ensure correct reassembly.
6. **ACKs flow continuously** in both directions confirming receipt of data as the page loads.
7. **Connection Termination:** Once done (or after a timeout), a 4-way FIN handshake closes the connection (or it stays open briefly for further requests via HTTP Keep-Alive).

Throughout this entire process, **ICMP** might fire in the background if something goes wrong (e.g., "Destination Unreachable" if the server is down), and **NTP** ensures your device's clock is accurate enough for the TLS certificate's validity dates to check out correctly.

---

## 13. Quick-Reference Cheat Sheet

```
TCP (reliable, connection-oriented):
  20/21  FTP        22  SSH         23  Telnet      25  SMTP
  80     HTTP        110 POP3       143 IMAP        443 HTTPS
  587    SMTP(sub)   993 IMAPS      995 POP3S        3389 RDP

UDP (fast, connectionless):
  53  DNS (queries)   67/68  DHCP    69  TFTP
  123 NTP             161/162 SNMP

No Port (Layer 3, works with IP directly):
  ICMP  (ping, traceroute, error messages)

TCP 3-Way Handshake:      SYN → SYN-ACK → ACK
TCP 4-Way Termination:    FIN → ACK → FIN → ACK
UDP Communication:        Send → (no confirmation) → Done
```

**Memory tip:** If it needs to be *perfect* (a file, a webpage, an email) → TCP. If it needs to be *fast* and real-time perfection doesn't matter (a call, a stream, a quick lookup) → UDP. If it's just a diagnostic/error message about the network itself → ICMP.
