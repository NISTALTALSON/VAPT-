# Routing, NAT & Switching — Deep Dive (Mastery Level)
### Companion Document to "The Complete Networking Guide"

*(Note: You wrote "MACOS" and "WLAN" — based on context, this covers **MAC addresses** and **VLANs**, which are the standard networking topics that pair with switching/ARP. Covered in full below.)*

---

## Table of Contents
**PART A — ROUTING**
1. What Is Routing? (The Big Picture)
2. How a Router Makes Decisions
3. The Routing Table — Deep Dive
4. Static Routing
5. Dynamic Routing
6. Interior vs Exterior Routing Protocols
7. Distance-Vector vs Link-State Routing
8. Major Routing Protocols in Detail (RIP, OSPF, EIGRP, BGP)
9. Administrative Distance & Route Selection
10. NAT (Network Address Translation) — Full Deep Dive
11. Routing + NAT: How Your Home Internet Actually Works

**PART B — SWITCHING**
12. What Is Switching? (The Big Picture)
13. MAC Addresses — Deep Dive
14. How a Switch Actually Works (MAC Address Table)
15. ARP (Address Resolution Protocol) — Full Deep Dive
16. Broadcast Domains vs Collision Domains
17. VLANs — Deep Dive
18. Spanning Tree Protocol (STP) — Preventing Loops
19. Switching + Routing Together: The Full Journey of a Packet
20. Quick-Reference Cheat Sheet

---

# PART A — ROUTING

## 1. What Is Routing? (The Big Picture)

**Routing** is the process of selecting the best path for data to travel from a source device to a destination device **across different networks**. It happens at **Layer 3 (Network Layer)** of the OSI model, and it's what makes the internet — a global network of networks — actually function.

**Key distinction from switching:** Switching moves data *within* a single network/subnet using MAC addresses. Routing moves data *between* different networks/subnets using IP addresses. A router is the device responsible for this decision-making.

---

## 2. How a Router Makes Decisions

Every time a router receives a packet, it performs this basic logic:
1. Look at the packet's **destination IP address**.
2. Check its **routing table** to find the best matching route toward that destination network.
3. Forward the packet out the correct interface toward the next hop (or the final destination if directly connected).
4. If no matching route exists and there's no default route, the packet is **dropped**, and the router typically sends back an ICMP "Destination Unreachable" message.

This process repeats at every router along the path — each one only needs to know the *next hop*, not the entire end-to-end path. This is sometimes called "hop-by-hop" forwarding.

---

## 3. The Routing Table — Deep Dive

A router's routing table lists all known destination networks and how to reach them.

**Example simplified routing table:**
```
Destination        Next Hop         Interface     Metric   Source
0.0.0.0/0           203.0.113.1     Gi0/0         1        Static (default route)
192.168.1.0/24      Directly connected  Gi0/1     0        Connected
10.0.0.0/8          10.1.1.1        Gi0/2         2        OSPF
172.16.0.0/16       10.1.1.1        Gi0/2         5        RIP
```

| Field | Meaning |
|---|---|
| Destination | The target network (in CIDR notation) |
| Next Hop | The IP address of the next router to forward toward |
| Interface | Which physical/logical interface to send the packet out of |
| Metric | A cost value used to pick the best route when multiple exist |
| Source | How the router learned this route (directly connected, static, or a specific dynamic protocol) |

### Longest Prefix Match Rule (Critical Concept)
When a router has multiple matching routes for a destination, it always chooses the **most specific** match — the one with the longest subnet mask (highest CIDR prefix number).

**Example:** Destination is `192.168.1.50`. Router has both:
- `192.168.0.0/16` (less specific)
- `192.168.1.0/24` (more specific)

The router picks the `/24` route because it's a longer, more specific prefix match — even if the `/16` route also technically covers that address.

### The Default Route
`0.0.0.0/0` is the "catch-all" route — used when no more specific route matches. This is typically how a router sends unknown traffic (e.g., general internet traffic) toward your ISP.

---

## 4. Static Routing

**Static routes** are manually configured by a network administrator — they don't change unless someone edits them.

```
ip route 172.16.0.0 255.255.0.0 10.1.1.1
```
(This says: "To reach 172.16.0.0/16, send traffic to next-hop 10.1.1.1")

### Pros
- Full administrator control, predictable behavior
- No protocol overhead (no routing updates being exchanged constantly)
- More secure — no dynamic protocol to potentially be attacked/spoofed
- Simple to configure for small networks with few paths

### Cons
- Doesn't automatically adapt if a link goes down (routes must be manually fixed)
- Doesn't scale well — becomes unmanageable in large, complex networks with many possible paths
- Prone to human error in large configurations

### Common Use Cases
- Small networks with a single path to the internet (e.g., a small office with one ISP connection — a static default route is simplest)
- Stub networks (networks with only one way in/out)
- Specific security-sensitive links where predictable routing is preferred over automatic adaptation

---

## 5. Dynamic Routing

**Dynamic routing** uses routing protocols that let routers **automatically discover** networks and **share route information** with each other, adapting to topology changes (like a failed link) without manual intervention.

### How it generally works
1. Routers running the same protocol exchange information about networks they know about (directly connected or learned from other routers).
2. Each router builds and continuously updates its routing table based on these exchanges.
3. If a link fails, routers detect this and recalculate/reroute automatically (this is called **convergence**).

### Convergence Time
The time it takes for **all** routers in a network to update their tables and agree on the new topology after a change. Faster convergence = less disruption during outages. Link-state protocols (like OSPF) generally converge faster than older distance-vector protocols (like RIP).

---

## 6. Interior vs Exterior Routing Protocols

| Type | Scope | Examples |
|---|---|---|
| **IGP (Interior Gateway Protocol)** | Routes within a single organization's network (an "Autonomous System") | RIP, OSPF, EIGRP |
| **EGP (Exterior Gateway Protocol)** | Routes *between* different organizations/networks on the internet | BGP |

An **Autonomous System (AS)** is a large network or group of networks under one administrative control (e.g., an ISP, a large corporation), each identified by a unique **AS number** used in BGP.

---

## 7. Distance-Vector vs Link-State Routing

### Distance-Vector Protocols (e.g., RIP, EIGRP)
- Each router only knows the **distance (metric)** and **direction (vector)** to a destination — not the actual full topology.
- Routers periodically share their entire routing table with directly connected neighbors ("routing by rumor" — you trust what your neighbor tells you without seeing the full picture yourself).
- Simpler, but slower to converge, and more prone to routing loops (mitigated with mechanisms like split horizon, route poisoning, hold-down timers).

### Link-State Protocols (e.g., OSPF, IS-IS)
- Every router builds a **complete map** of the entire network topology (not just what neighbors report).
- Routers flood "link-state advertisements" describing their directly connected links to the entire network.
- Each router independently runs an algorithm (Dijkstra's Shortest Path First) on this full map to calculate the best routes.
- Faster convergence, more scalable, more resource-intensive (CPU/memory) than distance-vector.

---

## 8. Major Routing Protocols in Detail

### RIP (Routing Information Protocol)
- Distance-vector, uses simple **hop count** as its only metric.
- Maximum of **15 hops** — anything requiring 16+ hops is considered "unreachable" (a built-in scalability limitation).
- Slow convergence, sends full routing table updates every 30 seconds by default.
- Largely considered legacy/outdated for modern networks, but still useful for learning the fundamentals.

### OSPF (Open Shortest Path First)
- Link-state, uses **cost** as its metric (based on interface bandwidth by default — faster links = lower cost = preferred).
- Fast convergence, highly scalable, widely used in enterprise networks.
- Organizes large networks into **areas** (Area 0 is the mandatory "backbone" area) to reduce the size of routing computations and improve scalability.
- Open standard (not vendor-locked, unlike EIGRP historically).

### EIGRP (Enhanced Interior Gateway Routing Protocol)
- Originally Cisco-proprietary (now partially open), often described as a "hybrid" protocol — has characteristics of both distance-vector and link-state.
- Uses a sophisticated composite metric (bandwidth, delay, reliability, load).
- Fast convergence via the **DUAL (Diffusing Update Algorithm)**, which pre-computes backup routes for near-instant failover.

### BGP (Border Gateway Protocol)
- **The protocol that runs the entire internet's backbone routing** — it's how traffic finds its way between different ISPs and massive organizations globally.
- Path-vector protocol — makes decisions based on network **policies** and the **AS-path** (the sequence of Autonomous Systems a route passes through), not just simple metrics like hop count.
- Extremely configurable/policy-driven — ISPs use BGP to control exactly how traffic enters/exits their networks based on business relationships, cost, and preference — not just "shortest path."
- Runs over **TCP port 179** (unusual among routing protocols, which often run directly over IP).
- Two types: **eBGP** (between different Autonomous Systems) and **iBGP** (within the same Autonomous System).

---

## 9. Administrative Distance & Route Selection

When a router learns about the *same* destination network from multiple sources (e.g., both a static route AND an OSPF-learned route), it needs a way to decide which to trust. This is **Administrative Distance (AD)** — lower number = more trusted/preferred.

| Route Source | Default Administrative Distance |
|---|---|
| Directly Connected | 0 |
| Static Route | 1 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| RIP | 120 |
| iBGP | 200 |

**Important distinction:** AD decides *which protocol's route to trust* when multiple sources offer a route to the same destination. **Metric** (hop count, cost, etc.) decides the best path *within* a single protocol when it has multiple possible routes.

---

## 10. NAT (Network Address Translation) — Full Deep Dive

**NAT** translates private IP addresses (used internally on a LAN) into a public IP address (used on the internet), and vice versa — allowing many devices with private, non-routable addresses to share a small number (often just one) of public IP addresses.

### Why NAT Exists
- IPv4 address exhaustion — there aren't enough public IPv4 addresses for every device on Earth to have its own.
- Security by obscurity — internal private IPs aren't directly reachable/visible from the internet, since the outside world only ever sees your router's public IP.

### The Three Types of NAT (Full Detail)

**1. Static NAT**
- A **one-to-one, permanent** mapping between one private IP and one public IP.
- Used when a specific internal device (like a server) needs to be **consistently and predictably reachable** from the internet via the same public IP every time.
```
Private: 192.168.1.10  <-->  Public: 203.0.113.10  (always this exact mapping)
```

**2. Dynamic NAT**
- Maps private IPs to public IPs **from a pool** of available public addresses, assigned on a first-come, first-served basis.
- If you have 50 internal devices but only 20 public IPs in the pool, only 20 devices can access the internet simultaneously — others must wait for an address to free up.
- Less common today than PAT, since it doesn't actually solve address scarcity as efficiently.

**3. PAT (Port Address Translation) / NAT Overload**
- The type used by virtually every home router.
- **Many private IPs share a single public IP**, differentiated from each other using unique **source port numbers**.
- This is how your laptop, phone, and smart TV can all access the internet simultaneously through one public IP address.

**How PAT actually works, step-by-step:**
```
Internal device: 192.168.1.10:51000  →  wants to reach  →  8.8.8.8:443

Router (NAT) rewrites the packet:
Source becomes: 203.0.113.5:61001 (router's public IP + a NEW unique port it assigns)

Router keeps a NAT translation table entry:
  Internal              External                  
  192.168.1.10:51000  ↔  203.0.113.5:61001  ↔  8.8.8.8:443

When the response comes back to 203.0.113.5:61001, the router checks its 
table, sees it belongs to 192.168.1.10:51000, and forwards it back correctly.
```
This is precisely why devices behind NAT generally **cannot be directly reached from the internet unopened** — there's no existing table entry for unsolicited inbound traffic, so the router doesn't know which internal device to forward it to (this is also an incidental security benefit, acting like a basic firewall).

### Port Forwarding
A manual override of PAT behavior — the administrator configures the router to always forward traffic on a specific external port to a specific internal IP/port, allowing external access to something like a home game server or security camera.
```
External: 203.0.113.5:25565  →  Always forward to  →  192.168.1.50:25565
```

### NAT Traversal Challenges
Because NAT hides internal addressing, certain applications (peer-to-peer connections, VoIP, some online games) that need devices behind NAT to be directly reachable by other outside devices require special techniques (STUN, TURN, UPnP, hole punching) to work around NAT's inherent behavior.

---

## 11. Routing + NAT: How Your Home Internet Actually Works

Putting it all together — here's the complete real-world journey:

1. Your laptop (`192.168.1.10`) sends a request to `google.com`.
2. Your laptop checks: is Google's IP within my local subnet? No → send to **default gateway** (your home router, `192.168.1.1`).
3. Your router receives the packet. It performs **routing** — determining that this traffic needs to go out to the internet via its WAN interface.
4. Before sending it out, the router performs **PAT/NAT** — rewriting the source IP from your private `192.168.1.10` to its own public IP, and tracking the translation in its NAT table.
5. The packet travels across the internet — through your ISP's routers, and potentially dozens of other routers (each making independent routing decisions using protocols like **BGP** at the largest scale) — until it reaches Google's servers.
6. Google's response comes back to your router's public IP.
7. Your router checks its NAT table, translates the destination back to your laptop's private IP, and forwards it internally.

Your router is simultaneously acting as a **router** (making Layer 3 forwarding decisions) and performing **NAT** (translating addresses) — two related but distinct functions often bundled into the same home device.

---

# PART B — SWITCHING

## 12. What Is Switching? (The Big Picture)

**Switching** is the process of forwarding data **within the same local network** using **MAC addresses**, operating primarily at **Layer 2 (Data Link Layer)** of the OSI model. A switch connects multiple devices within a LAN and intelligently directs traffic only to the specific device it's intended for — unlike an old-fashioned hub, which blindly broadcasts everything to every port.

---

## 13. MAC Addresses — Deep Dive

A **MAC (Media Access Control) address** is a globally unique, hardware-burned-in identifier assigned to a device's network interface card (NIC) — it identifies a device at Layer 2, within its local network segment.

### Structure
```
00:1A:2B:3C:4D:5E
└──┬───┘└───┬───┘
  OUI      NIC-specific
```
- **48 bits total**, written as 12 hexadecimal digits (in pairs, separated by colons or hyphens).
- **First 24 bits (OUI — Organizationally Unique Identifier):** Identifies the manufacturer (e.g., a specific block assigned to Apple, Cisco, Dell).
- **Last 24 bits:** A unique serial number assigned by that manufacturer, ensuring global uniqueness.

### Key Facts About MAC Addresses
- MAC addresses are meant to be **permanent and globally unique** (unlike IP addresses, which change based on network/location) — though modern OSes increasingly support "MAC randomization" for privacy on Wi-Fi networks.
- MAC addresses have **local significance only** — they're used for communication *within* a local network segment, and are **not routable** across different networks (routers strip/replace Layer 2 info as packets cross network boundaries — this is a key reason IP addressing exists at all).
- **Broadcast MAC address:** `FF:FF:FF:FF:FF:FF` — a frame sent to this address is delivered to *every* device on the local network segment.

### MAC vs IP — The Core Distinction
| | MAC Address | IP Address |
|---|---|---|
| OSI Layer | 2 (Data Link) | 3 (Network) |
| Assigned by | Manufacturer (hardware) | Network administrator/DHCP |
| Scope | Local network segment only | Globally routable |
| Changes? | Rarely (though can be spoofed/randomized) | Often (different network = different IP) |
| Format | 48-bit hex (`00:1A:2B:3C:4D:5E`) | 32-bit (IPv4) or 128-bit (IPv6) |
| Used by | Switches | Routers |

---

## 14. How a Switch Actually Works (MAC Address Table)

A switch maintains a **MAC address table** (also called a CAM table — Content Addressable Memory table) that maps MAC addresses to the specific physical port they're reachable through.

### The Learning Process
1. A frame arrives on a switch port with a source MAC address.
2. The switch **records** that source MAC address alongside the port it arrived on, in its MAC address table (this is how it "learns" the network's layout dynamically — no manual configuration needed).
3. The switch checks the frame's **destination MAC address** against its table:
   - **If found:** Forward the frame *only* out that specific port (efficient — this is called "unicast forwarding").
   - **If NOT found:** **Flood** the frame out *every* port except the one it arrived on (since the switch doesn't yet know where that destination lives) — every connected device sees it, but only the intended recipient responds, and the switch then learns its location from that reply.
   - **If destination is the broadcast address:** Always flood to all ports.

### Example MAC Address Table
```
MAC Address           Port    VLAN
00:1A:2B:3C:4D:5E      1       10
00:1A:2B:3C:4D:6F      2       10
00:1A:2B:3C:4D:7A      3       20
```

This table entries **age out** after a period of inactivity (default often 300 seconds) to keep the table current if devices move or disconnect.

---

## 15. ARP (Address Resolution Protocol) — Full Deep Dive

**ARP** bridges the gap between Layer 3 (IP addresses) and Layer 2 (MAC addresses) — it answers the question: **"I know this device's IP address, but what's its MAC address so I can actually deliver a frame to it on the local network?"**

### Why ARP Is Needed
When your device wants to send data to another device on the **same local network**, it needs to build a Layer 2 frame, which requires a **destination MAC address** — but applications typically only know the destination's IP address. ARP resolves this.

### The ARP Process, Step-by-Step
```
1. Device A wants to send data to 192.168.1.20 (on the same subnet).
2. Device A checks its local ARP cache — is 192.168.1.20's MAC already known?
   - If yes: use it immediately, skip to step 5.
   - If no: continue to step 2.
3. Device A broadcasts an ARP Request to the entire local network:
   "Who has 192.168.1.20? Tell 192.168.1.10 (me)."
   (Sent to broadcast MAC: FF:FF:FF:FF:FF:FF — every device on the LAN receives it)
4. Only the device that actually owns 192.168.1.20 responds directly (unicast) 
   with an ARP Reply: "192.168.1.20 is at MAC 00:1A:2B:3C:4D:6F"
5. Device A stores this mapping in its local ARP cache (for a limited time,
   avoiding repeated ARP requests for the same destination) and now sends
   the actual data frame directly to that MAC address.
```

### ARP and Routing — What Happens for Remote Destinations
If the destination IP is **not** on the local subnet, a device doesn't ARP for the destination's IP directly — instead, it ARPs for its **default gateway's** MAC address, and sends the frame there. The router then handles forwarding it onward (potentially ARPing again on the next network segment to find the next hop).

### ARP Cache
```
IP Address        MAC Address           Type
192.168.1.1        00:1A:2B:3C:4D:01    Dynamic
192.168.1.20       00:1A:2B:3C:4D:6F    Dynamic
```
View it on most systems with: `arp -a`

### ARP Spoofing (Security Concern)
Since ARP has **no built-in authentication**, an attacker can send fake/forged ARP replies claiming to own an IP address they don't (e.g., claiming to be the default gateway) — tricking other devices into sending their traffic to the attacker instead. This is a classic **Man-in-the-Middle** technique on local networks, mitigated by tools like Dynamic ARP Inspection (DAI) on managed switches.

### Gratuitous ARP
A device proactively announces its own IP-to-MAC mapping to the whole network without being asked — often used when a device first boots up (to detect IP conflicts) or during a failover event (e.g., a backup server taking over an IP, announcing "I now own this IP" so switches update their tables immediately).

---

## 16. Broadcast Domains vs Collision Domains

| Concept | Definition | What separates it |
|---|---|---|
| **Collision Domain** | A network segment where devices might transmit simultaneously and their signals "collide" | Every switch **port** is its own collision domain (modern switched networks essentially eliminate collisions entirely with full-duplex links) |
| **Broadcast Domain** | A network segment where a broadcast frame (`FF:FF:FF:FF:FF:FF`) reaches every device | A **router** (or VLAN boundary) separates broadcast domains — switches do NOT, by default, they forward broadcasts everywhere |

**Key fact:** A switch, by default, extends a **single** broadcast domain across all its connected ports (unless VLANs are used to segment it) — but each individual port is its own isolated collision domain. A hub, by contrast, is a single collision domain across all ports (all devices essentially share one wire).

---

## 17. VLANs — Deep Dive

*(Since you mentioned "WLAN" alongside switching/ARP/MAC — VLANs are the concept that actually pairs with this topic, so covered here. WLAN/wireless networking specifically was covered in the main networking guide, Section 17.)*

A **VLAN (Virtual LAN)** logically divides a single physical switch (or group of switches) into multiple separate, isolated broadcast domains — as if they were entirely separate physical networks — without needing separate physical hardware for each.

### Why VLANs Exist
- **Security:** Isolate sensitive traffic (e.g., a Finance VLAN completely separated from a Guest Wi-Fi VLAN, even though they might share the same physical switch).
- **Reduced broadcast traffic:** Smaller broadcast domains mean less unnecessary traffic flooding every device.
- **Logical organization independent of physical location:** Group devices by department/function rather than by which physical switch port they happen to plug into.

### Access Ports vs Trunk Ports
| Port Type | Purpose |
|---|---|
| **Access Port** | Belongs to exactly **one** VLAN — used to connect end devices (PCs, printers, phones). The device connected to it has no awareness it's even on a VLAN — the switch handles that transparently. |
| **Trunk Port** | Carries traffic for **multiple** VLANs simultaneously — used for links *between* switches, or between a switch and a router. Uses **802.1Q tagging** to mark which VLAN each frame belongs to as it crosses the trunk. |

### 802.1Q Tagging
When a frame travels across a trunk link, the switch inserts a small **VLAN tag** (4 bytes) into the Ethernet frame header, containing the VLAN ID (1–4094). This tag is stripped off again before the frame reaches an access port/end device — end devices never actually see or need to understand VLAN tags.

### Inter-VLAN Routing
By design, devices on **different VLANs cannot communicate directly** — even if physically connected to the same switch — because they're in different broadcast domains (effectively different logical networks, each typically needing its own IP subnet). To allow communication *between* VLANs, you need:
- A **router** with a connection to each VLAN (or a single connection using "router-on-a-stick" — a trunk link carrying multiple VLANs, with the router handling inter-VLAN routing via subinterfaces), OR
- A **Layer 3 switch** — a switch with built-in routing capability, which can route between VLANs without needing an external router at all.

### Native VLAN
On a trunk link, the **native VLAN** is the one VLAN whose traffic is sent **untagged** (a legacy compatibility feature) — everything else on the trunk is explicitly tagged. Misconfigured native VLANs (mismatched between two switches) are a common source of subtle network issues and a known security concern (VLAN hopping attacks).

---

## 18. Spanning Tree Protocol (STP) — Preventing Loops

Redundant physical links between switches (added intentionally for fault tolerance/backup paths) create a serious problem: **Layer 2 loops**. Unlike Layer 3 (where IP packets have a TTL that eventually expires them), Ethernet frames have **no TTL** — a loop can cause frames to circulate endlessly, rapidly multiplying and consuming all available bandwidth (a "broadcast storm"), crashing the network.

**STP (Spanning Tree Protocol)** solves this by:
1. Electing a **Root Bridge** (a reference switch that all path calculations are based on).
2. Calculating the best (lowest-cost) path from every other switch to the Root Bridge.
3. **Blocking** redundant links that would otherwise create a loop — those ports go into a "blocking" state (not forwarding traffic) while still being ready to activate instantly if the primary path fails.
4. If the active link fails, STP recalculates and un-blocks a backup port to restore connectivity — providing redundancy *without* creating an active loop.

Modern networks often use faster variants like **RSTP (Rapid Spanning Tree Protocol)** for much quicker convergence after a topology change.

---

## 19. Switching + Routing Together: The Full Journey of a Packet

**Scenario:** PC-A (`192.168.1.10`) on Switch 1 wants to send data to PC-B (`192.168.2.20`) on a completely different subnet, reachable through Router R1.

```
1. PC-A determines PC-B's IP is NOT on its local subnet
   → needs to send via its default gateway (Router R1, 192.168.1.1)

2. PC-A checks its ARP cache for 192.168.1.1's MAC address
   → if not cached, broadcasts an ARP Request; R1 replies with its MAC

3. PC-A builds an Ethernet frame:
   Source MAC = PC-A's MAC | Destination MAC = R1's MAC
   Source IP = 192.168.1.10 | Destination IP = 192.168.2.20
   (Notice: destination MAC is the ROUTER's, but destination IP is
    still PC-B's — this distinction is critical to understand)

4. Switch 1 receives the frame, checks its MAC table, forwards it
   out the port connected to R1 (Layer 2 switching)

5. Router R1 receives the frame, strips the Layer 2 header, examines
   the destination IP (192.168.2.20), checks its ROUTING TABLE,
   determines the correct outgoing interface (Layer 3 routing)

6. R1 needs a NEW Layer 2 frame for the next segment — it ARPs (if not
   cached) for PC-B's MAC address, builds a new frame:
   Source MAC = R1's MAC | Destination MAC = PC-B's MAC
   (IP addresses stay the same throughout — only MAC addresses change
    at each hop)

7. Switch 2 receives this new frame, checks its MAC table, forwards
   it to PC-B's port

8. PC-B receives the frame — communication complete.
```

**The single most important takeaway from this entire walkthrough:** IP addresses stay constant end-to-end throughout the journey, but MAC addresses change at **every single hop** along the path — this is the fundamental interplay between Layer 2 (switching) and Layer 3 (routing) that makes the entire internet function.

---

## 20. Quick-Reference Cheat Sheet

```
ROUTING
  Static Route    → Manually configured, no auto-adaptation
  Dynamic Route    → Auto-discovered/updated via protocols (RIP, OSPF, EIGRP, BGP)
  Distance-Vector  → Trusts neighbor's reported distance (RIP, EIGRP)
  Link-State       → Builds full topology map (OSPF, IS-IS)
  Path-Vector      → Policy-based, uses AS-path (BGP)
  Longest Prefix Match → Router always picks the MOST SPECIFIC matching route
  Administrative Distance → Lower = more trusted (Connected=0, Static=1, OSPF=110, RIP=120)

NAT
  Static NAT   → 1-to-1 permanent mapping
  Dynamic NAT  → Many-to-many from a pool, first-come-first-served
  PAT/Overload → Many-to-ONE, differentiated by port number (used by home routers)
  Port Forwarding → Manual rule: external port → specific internal IP:port

SWITCHING
  MAC Address        → 48-bit hardware address, Layer 2, LOCAL scope only
  Switch learns       → Records source MAC + port on every frame received
  Unknown destination → Switch FLOODS the frame to all ports
  Known destination   → Switch forwards ONLY to the correct port

ARP
  Purpose: IP address  →  MAC address (Layer 3 to Layer 2 bridge)
  Broadcast Request → Unicast Reply
  ARP Cache stores learned mappings temporarily
  Spoofing = fake ARP replies used for Man-in-the-Middle attacks

VLANS
  Access Port → belongs to ONE VLAN, connects end devices
  Trunk Port  → carries MULTIPLE VLANs, uses 802.1Q tagging
  Different VLANs CANNOT talk without a router or Layer 3 switch

STP
  Prevents Layer 2 loops by blocking redundant paths until needed

KEY RULE TO REMEMBER
  IP addresses stay the SAME end-to-end.
  MAC addresses change at EVERY hop along the path.
```
