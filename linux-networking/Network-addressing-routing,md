

## 🧭 1. **Unicast**

**Definition:**
Communication between **one sender and one specific receiver**.

**Example:**
When your computer sends a request to `www.google.com`, your packet goes to one specific Google server’s IP address (e.g. `142.250.183.4`).

**Use cases:**

* Regular web browsing
* SSH, email
* Point-to-point communication

**How it works:**
Each device has a **unique IP address**, and routers forward the packet to that **exact destination address**.

📡 **Analogy:** Sending a letter to a specific person’s home address.

---

## 🌍 2. **Broadcast**

**Definition:**
One sender sends data to **all devices** in a network (within the same subnet).

**Example:**
When a computer first joins a LAN, it might send an **ARP broadcast**:
“Who has IP 192.168.1.1? Tell me your MAC address.”

**Use cases:**

* ARP requests
* DHCP discovery (find a DHCP server on LAN)

**How it works:**
Broadcast packets go to **all nodes in the broadcast domain** (e.g., all devices in `192.168.1.0/24` network).

📡 **Analogy:** Shouting in a room — everyone hears it, even if not all care.

---

## 👥 3. **Multicast**

**Definition:**
One sender sends packets to **a group of receivers** who have explicitly “joined” a multicast group.

**Example:**
Live streaming or IPTV to multiple subscribers (without sending separate copies to each).

**Use cases:**

* Video/audio conferencing
* Live TV or radio over IP
* Routing protocols (OSPF, RIP use multicast)

**How it works:**

* Uses **special IP ranges** (IPv4: `224.0.0.0 – 239.255.255.255`).
* Receivers **subscribe** to a group (using IGMP).
* Routers duplicate packets only when needed, saving bandwidth.

📡 **Analogy:** A WhatsApp group message — only members of the group receive it.

---

## 🧩 4. **Anycast**

**Definition:**
One sender sends data to **multiple servers that share the same IP address**, but the **network (via BGP)** routes it to the **nearest or best** destination.

**Example:**
CDNs (Cloudflare, Google, Akamai) — when you hit `1.1.1.1` or `8.8.8.8`, your packet goes to the **closest** data center, not a single global machine.

**Use cases:**

* DNS resolvers (Google DNS, Cloudflare DNS)
* CDN edge servers
* Load balancing across multiple locations

**How it works:**

* Multiple servers advertise the same IP prefix via **BGP** from different geographic locations.
* Routers forward traffic to the **shortest AS path** (i.e., closest server).
* If one location fails, BGP automatically routes to the next closest one.

📡 **Analogy:** A toll-free customer care number — you dial one number, but the call connects to the nearest available call center.

---

## ⚖️ Summary Table

| Type          | Direction        | Receivers                | Routing Basis        | Typical Use Case   | Example           |
| ------------- | ---------------- | ------------------------ | -------------------- | ------------------ | ----------------- |
| **Unicast**   | 1 → 1            | Single                   | Destination IP       | Web browsing, SSH  | `192.168.1.5`     |
| **Broadcast** | 1 → All          | All in subnet            | Broadcast domain     | ARP, DHCP          | `255.255.255.255` |
| **Multicast** | 1 → Many (group) | Subscribed members       | Group address (IGMP) | IPTV, conferencing | `224.0.0.1`       |
| **Anycast**   | 1 → Nearest      | One (nearest among many) | BGP shortest path    | CDN, DNS           | `8.8.8.8`         |

---

