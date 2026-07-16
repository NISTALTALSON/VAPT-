# DNS — Deep Dive (Mastery Level)
### Companion Document to "The Complete Networking Guide"

---

## Table of Contents
1. What Is DNS? (The Big Picture)
2. Why DNS Exists
3. The DNS Hierarchy (Namespace Structure)
4. Types of DNS Servers
5. The Full DNS Resolution Process (Step-by-Step)
6. Recursive vs Iterative Queries
7. DNS Record Types — Full Mastery
   - A Record
   - AAAA Record
   - CNAME Record
   - MX Record
   - TXT Record
   - NS Record
   - PTR Record
   - SOA Record
   - SRV Record
   - CAA Record
   - Other/Legacy Records
8. DNS Caching & TTL
9. DNS Zones & Zone Files
10. DNS Ports & Protocol Behavior (TCP vs UDP)
11. DNS Security (DNSSEC, Spoofing, Poisoning)
12. Common DNS Tools & Commands
13. Real-World Example: Full DNS Lookup Walkthrough
14. Quick-Reference Cheat Sheet

---

## 1. What Is DNS? (The Big Picture)

**DNS (Domain Name System)** is the system that translates human-readable domain names (like `google.com`) into machine-readable IP addresses (like `142.250.premium.x` or `2607:f8b0::200e`) that computers actually use to locate and communicate with each other.

It is often called **"the phonebook of the internet"** — you know a business by its name, but the phone network needs the actual number to connect the call. DNS does exactly that for websites and services.

Without DNS, you'd have to memorize and type raw IP addresses for every website you wanted to visit — completely impractical at internet scale.

---

## 2. Why DNS Exists

- **Human memory limitation:** Nobody can memorize millions of numeric IP addresses. Names are far easier to remember.
- **Flexibility:** A company can change its server's underlying IP address without anyone needing to know — they just update the DNS record, and the domain name stays the same.
- **Load distribution:** A single domain name can point to multiple IP addresses (multiple servers), enabling load balancing and redundancy.
- **Abstraction:** DNS decouples the *identity* of a service (its name) from its *location* (its IP address) — a core principle in scalable network design.

---

## 3. The DNS Hierarchy (Namespace Structure)

DNS is organized as an **inverted tree**, structured from most general (top) to most specific (bottom):

```
                    . (Root)
                     |
        -----------------------------
        |            |              |
      .com          .org           .net    ← Top-Level Domains (TLDs)
        |
    example.com                             ← Second-Level Domain
        |
   -------------
   |           |
 www.example.com   mail.example.com          ← Subdomains
```

### Breaking down `mail.example.com.`
Reading right to left:
| Part | Name | Example |
|---|---|---|
| `.` | Root domain | Invisible, implied at the very end |
| `com` | Top-Level Domain (TLD) | Managed by a registry (Verisign for .com) |
| `example` | Second-Level Domain (SLD) | The actual registered domain name |
| `mail` | Subdomain / Hostname | A specific service/host under the domain |

### Types of TLDs
| Type | Examples | Notes |
|---|---|---|
| **gTLD** (Generic) | .com, .org, .net, .info | Open for general registration |
| **ccTLD** (Country Code) | .uk, .jp, .in, .de | Assigned to specific countries |
| **sTLD** (Sponsored) | .gov, .edu, .mil | Restricted to specific entities |
| **New gTLDs** | .app, .dev, .io, .tech | Newer, often industry/purpose-specific |

---

## 4. Types of DNS Servers

DNS resolution relies on several categories of servers, each with a distinct role:

| Server Type | Role |
|---|---|
| **DNS Resolver (Recursive Resolver)** | The "middleman" — receives your query and does all the legwork of tracking down the answer on your behalf (e.g., your ISP's DNS, or public resolvers like Google `8.8.8.8` or Cloudflare `1.1.1.1`) |
| **Root Server** | The starting point of DNS resolution — knows where to find the TLD servers. There are 13 logical root server clusters worldwide (labeled A–M), globally distributed via anycast. |
| **TLD Server** | Knows which authoritative servers are responsible for domains under a specific TLD (e.g., all `.com` domains) |
| **Authoritative Name Server** | The final word — holds the actual DNS records for a specific domain and gives the definitive answer |

**Key distinction:** A recursive resolver doesn't necessarily *know* the answer — it *hunts down* the answer by querying other servers on your behalf. An authoritative server *does* know the answer directly, because it's the source of truth for that domain's records.

---

## 5. The Full DNS Resolution Process (Step-by-Step)

**Scenario:** You type `www.example.com` into your browser.

```
Your Device
     |
     | 1. Check local cache (browser, OS) — if cached, DONE.
     v
Recursive Resolver (e.g., ISP or 8.8.8.8)
     |
     | 2. Check its own cache — if cached, return answer, DONE.
     |    If not cached, begin recursive lookup:
     v
Root Server (.)
     |
     | 3. "I don't know the IP, but here's who handles .com"
     v
TLD Server (.com)
     |
     | 4. "I don't know the IP, but here's the authoritative
     |     server for example.com"
     v
Authoritative Name Server (example.com)
     |
     | 5. "Here is the actual A record: 93.184.216.34"
     v
Recursive Resolver
     |
     | 6. Caches the answer (per its TTL) and returns it to you
     v
Your Device
     |
     | 7. Now connects directly to 93.184.216.34 (browser opens TCP
     |    connection to that IP, e.g., via HTTPS)
```

This entire process usually takes **milliseconds** and is largely invisible to the user — but every single web request, email send, and app connection depends on it.

---

## 6. Recursive vs Iterative Queries

| Query Type | Description |
|---|---|
| **Recursive Query** | The client asks the resolver, and expects a *final answer* — the resolver does all the work of chasing down the answer across multiple servers and returns just the result. (This is what your device does to its resolver.) |
| **Iterative Query** | The resolver asks each server (root, TLD, authoritative) and each one responds with "I don't know, but ask this other server" — the resolver iterates through referrals until it gets the final answer. (This is what the resolver does to root/TLD/authoritative servers.) |

**In short:** Your device makes a *recursive* request to its resolver ("just give me the answer"), and the resolver itself performs a series of *iterative* queries to actually go get that answer.

---

## 7. DNS Record Types — Full Mastery

DNS records are entries stored on authoritative name servers that define how a domain behaves — where it points, how mail should be routed, ownership verification, and more.

### A Record (Address Record)
- Maps a domain name to an **IPv4 address**.
- The most fundamental and common DNS record type.
```
example.com.       A       93.184.216.34
```
- Used any time a device needs to reach a domain over IPv4.

### AAAA Record ("Quad-A" Record)
- Maps a domain name to an **IPv6 address**.
- Functionally identical purpose to an A record, just for the 128-bit IPv6 address space instead of the 32-bit IPv4 space.
- Named "AAAA" because an IPv6 address (128 bits) is four times the size of an IPv4 address (32 bits) — A×4 = AAAA.
```
example.com.       AAAA    2606:2800:220:1:248:1893:25c8:1946
```
- A domain can (and increasingly does) have **both** an A and an AAAA record, so devices can connect via whichever IP version they support — this is called "dual-stack."

### CNAME Record (Canonical Name)
- Creates an **alias** — points one domain name to *another domain name* (not directly to an IP).
- The DNS resolver then has to resolve that target domain's own A/AAAA record to get the actual IP.
```
www.example.com.   CNAME   example.com.
blog.example.com.  CNAME   example.github.io.
```
- Extremely common for pointing subdomains to third-party services (e.g., pointing `shop.yourdomain.com` to a Shopify-hosted domain) without needing to know or update the underlying IP.
- **Important rule:** A CNAME record **cannot coexist** with any other record type on the same exact name (e.g., you can't have both a CNAME and an MX record on `www.example.com`). The root domain (the "apex," e.g., `example.com` itself) also generally cannot have a CNAME record — this is why services often use an "ALIAS" or "ANAME" record (a non-standard workaround) at the root instead.

### MX Record (Mail Exchange)
- Specifies which mail server(s) are responsible for **receiving email** on behalf of the domain.
- Includes a **priority/preference value** — lower numbers = higher priority. If the top-priority server is unreachable, mail servers try the next lowest number.
```
example.com.   MX   10   mail1.example.com.
example.com.   MX   20   mail2.example.com.
```
- In this example, mail is attempted to `mail1` first (priority 10); if that fails, `mail2` (priority 20) is tried as a backup.
- MX records point to a **hostname**, which must in turn resolve via its own A/AAAA record — MX records cannot point directly to an IP address.

### TXT Record (Text Record)
- Stores arbitrary **text data** attached to a domain — originally meant for human-readable notes, but now heavily used for machine-readable verification and email security.
- Extremely common modern uses:
  - **SPF (Sender Policy Framework):** Lists which mail servers are authorized to send email on behalf of the domain (anti-spoofing)
    ```
    example.com.  TXT  "v=spf1 include:_spf.google.com ~all"
    ```
  - **DKIM (DomainKeys Identified Mail):** Publishes a public key used to verify that an email's content wasn't tampered with in transit
  - **DMARC:** Tells receiving mail servers what to do if SPF/DKIM checks fail (reject, quarantine, or allow)
  - **Domain ownership verification:** Services like Google Workspace, AWS, or SSL certificate providers ask you to add a specific TXT record to prove you control the domain.

### NS Record (Name Server)
- Specifies which servers are the **authoritative name servers** for the domain — essentially declaring "these servers hold the real DNS records for this zone."
```
example.com.   NS   ns1.exampledns.com.
example.com.   NS   ns2.exampledns.com.
```
- These are the servers a TLD server refers your resolver to during the iterative lookup process.

### PTR Record (Pointer Record) — Reverse DNS
- Does the **opposite** of an A record: maps an **IP address back to a domain name**.
- Used for reverse lookups — e.g., mail servers often check PTR records to verify a sending server's legitimacy (anti-spam measure); a missing/mismatched PTR record is a red flag.
- Lives in a special reverse-lookup zone using `in-addr.arpa` (IPv4) or `ip6.arpa` (IPv6).
```
34.216.184.93.in-addr.arpa.   PTR   example.com.
```
(Note: the IP is written in reverse octet order for this zone.)

### SOA Record (Start of Authority)
- Every DNS zone has exactly **one** SOA record — it defines core administrative information about the zone itself.
- Contains: the primary name server, the admin's email, a serial number (used to track zone updates for replication), and timing values (refresh, retry, expire, minimum TTL) that control how secondary/backup DNS servers sync with the primary.
```
example.com.  SOA  ns1.exampledns.com. admin.example.com. (
                    2024031501  ; Serial
                    3600        ; Refresh
                    900         ; Retry
                    1209600     ; Expire
                    86400 )     ; Minimum TTL
```

### SRV Record (Service Record)
- Specifies the **location (hostname + port)** of specific services, beyond just plain web/mail traffic — commonly used for things like VoIP (SIP), Microsoft Active Directory, and messaging services (XMPP).
- Format includes priority, weight, port, and target.
```
_sip._tcp.example.com.   SRV   10  60  5060  sipserver.example.com.
```

### CAA Record (Certification Authority Authorization)
- Specifies **which Certificate Authorities (CAs)** are permitted to issue SSL/TLS certificates for the domain — a security control that prevents unauthorized CAs from issuing rogue certificates for your domain.
```
example.com.   CAA   0   issue   "letsencrypt.org"
```

### Other / Legacy Records (Good to Know)
| Record | Purpose |
|---|---|
| **NAPTR** | Naming Authority Pointer — used in telephony/VoIP number-to-service translation |
| **DNSKEY** | Publishes public keys used for DNSSEC validation |
| **DS** | Delegation Signer — links a child zone's DNSSEC key to the parent zone, forming a chain of trust |
| **RRSIG** | A DNSSEC digital signature over a set of records, proving they haven't been tampered with |
| **HINFO** | Host Information — legacy, rarely used, described host hardware/OS |

---

## 8. DNS Caching & TTL

To avoid hammering authoritative servers with repeated identical queries, DNS answers are **cached** at multiple levels: your browser, your OS, your router, and your recursive resolver.

### TTL (Time To Live)
- Every DNS record has a TTL value (in seconds) set by the domain's administrator, specifying **how long** that answer may be cached before it must be re-queried.
- Example: `TTL 3600` = cached answer is valid for 1 hour before resolvers should ask again.
- **Low TTL** (e.g., 300 seconds): Useful when you're planning to change a record soon (e.g., during a server migration) — changes propagate faster, at the cost of more frequent lookups.
- **High TTL** (e.g., 86400 seconds/24 hours): Reduces load and lookup latency, but means changes take longer to propagate globally, since old cached answers linger.

### Why "DNS propagation" takes time
When you update a DNS record, it doesn't change instantly everywhere — every resolver that had cached the old value keeps serving it until that record's TTL expires. This is why DNS changes can appear to take anywhere from minutes to 24–48 hours to fully "propagate" worldwide.

---

## 9. DNS Zones & Zone Files

A **DNS zone** is a distinct portion of the DNS namespace managed by a specific organization/administrator — essentially, the collection of all DNS records for a domain (and its subdomains, unless delegated further).

A **zone file** is the actual text file (stored on the authoritative name server) containing all the records for that zone — A, AAAA, MX, TXT, NS, SOA, etc. — in a defined syntax.

**Zone Transfer:** The process of copying zone data from a primary (master) DNS server to secondary (slave) servers, keeping them synchronized. Uses TCP (not UDP) due to the potentially large amount of data. AXFR = full zone transfer; IXFR = incremental (only changes since last sync).

---

## 10. DNS Ports & Protocol Behavior (TCP vs UDP)

- **Port 53** is used for both.
- **UDP** is used for the vast majority of ordinary DNS queries — small, fast, connectionless, ideal for quick request/response lookups.
- **TCP** is used when:
  - The response exceeds 512 bytes (traditional DNS UDP size limit) — common with DNSSEC-signed responses, or records with lots of data
  - Performing a **zone transfer** between DNS servers (needs TCP's reliability for large data transfers)
  - Some modern configurations increasingly use **EDNS0** to allow larger UDP responses, reducing the need to fall back to TCP.

---

## 11. DNS Security (DNSSEC, Spoofing, Poisoning)

### The core vulnerability
Traditional DNS was designed without built-in authentication — a resolver has no inherent way to verify that a response actually came from the legitimate authoritative server and wasn't tampered with in transit. This opens the door to attacks.

### DNS Spoofing / Cache Poisoning
An attacker tricks a DNS resolver into caching a **false** record — for example, making `bank.com` resolve to the attacker's malicious IP instead of the real one. Victims querying that poisoned resolver get silently redirected to a fake site, often used for phishing/credential theft.

### DNSSEC (DNS Security Extensions)
- Adds **cryptographic signatures** to DNS records, allowing resolvers to verify that a response is authentic and hasn't been altered.
- Uses a **chain of trust**: the root zone signs the TLD zones, TLD zones sign the domain's zone, forming an unbroken chain of cryptographic verification from the root down to the specific record.
- Key record types involved: **DNSKEY** (public keys), **RRSIG** (signatures), **DS** (links parent/child zone trust).
- DNSSEC protects against spoofing/tampering but does **not** encrypt DNS traffic (that's a separate concern — see below).

### DNS Encryption: DoH & DoT
Standard DNS queries are sent in **plaintext**, meaning anyone monitoring network traffic (ISP, attacker on the same network) can see exactly what domains you're looking up.
- **DoT (DNS over TLS):** Encrypts DNS queries using TLS, typically over port 853.
- **DoH (DNS over HTTPS):** Encrypts DNS queries by tunneling them inside regular HTTPS traffic (port 443), making DNS queries indistinguishable from normal web traffic — increasingly built into modern browsers by default.

---

## 12. Common DNS Tools & Commands

| Command | Purpose |
|---|---|
| `nslookup example.com` | Basic DNS lookup (Windows/Linux/Mac) |
| `nslookup -type=MX example.com` | Query a specific record type |
| `dig example.com` | Detailed DNS lookup tool (Linux/Mac), shows full response |
| `dig example.com MX` | Query specific record type with dig |
| `dig +trace example.com` | Shows the entire resolution path from root → TLD → authoritative |
| `dig -x 93.184.216.34` | Reverse DNS lookup (PTR record) |
| `host example.com` | Simple, quick DNS lookup |
| `ipconfig /flushdns` (Windows) | Clears the local DNS resolver cache |
| `systemd-resolve --flush-caches` (Linux) | Clears local DNS cache on some Linux systems |

---

## 13. Real-World Example: Full DNS Lookup Walkthrough

**Command:** `dig example.com MX`

**Conceptual response breakdown:**
```
;; QUESTION SECTION:
;example.com.          IN   MX

;; ANSWER SECTION:
example.com.    3600   IN   MX   10   mail1.example.com.
example.com.    3600   IN   MX   20   mail2.example.com.

;; AUTHORITY SECTION:
example.com.    3600   IN   NS   ns1.exampledns.com.
example.com.    3600   IN   NS   ns2.exampledns.com.
```

**What this tells us:**
- The domain has two mail servers, `mail1` (higher priority, 10) and `mail2` (backup, priority 20)
- These records are cached for `3600` seconds (1 hour) before needing re-verification
- The authoritative name servers for the domain are `ns1` and `ns2.exampledns.com`

---

## 14. Quick-Reference Cheat Sheet

```
CORE RECORD TYPES
  A       →  Domain → IPv4 address
  AAAA    →  Domain → IPv6 address
  CNAME   →  Domain → another domain (alias)
  MX      →  Domain → mail server(s), with priority
  TXT     →  Arbitrary text (SPF, DKIM, DMARC, verification)
  NS      →  Domain → authoritative name servers
  PTR     →  IP → domain (reverse lookup)
  SOA     →  Zone's administrative info (one per zone)
  SRV     →  Host+port for specific services (VoIP, AD, etc.)
  CAA     →  Which CAs may issue SSL certs for the domain

RESOLUTION ORDER
  Local cache → Recursive Resolver → Root → TLD → Authoritative

QUERY TYPES
  Recursive  → Client asks resolver for the FINAL answer
  Iterative  → Resolver asks each server, follows referrals

PROTOCOL
  Port 53 | UDP for normal queries | TCP for large responses/zone transfers

SECURITY
  DNSSEC → Authenticates records (prevents spoofing/poisoning)
  DoH/DoT → Encrypts DNS traffic (prevents eavesdropping)
```

**Key exam/interview fact to remember:** A CNAME record can never coexist with other records (like MX) on the same name, and typically cannot be used at the domain's root/apex — this trips up beginners constantly when setting up custom domains with third-party services.
