UDP **does not establish or close a connection at all** — but **UDP sockets have a lifecycle** in the OS, and applications often *simulate* connection-like behavior.


---

# 🧠 UDP Socket Lifecycle (Full Internal Flow)

Let's break it into 3 stages:

## 1️⃣ Socket Creation

## 2️⃣ "Connection" (optional)

## 3️⃣ Data Transfer

## 4️⃣ Socket Closure

---

# 1️⃣ **Socket Creation**

Application calls:

### **Server:**

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);
bind(fd, ...);
```

### **Client:**

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);
```

Linux creates an entry in the kernel with:

* local port (after bind or sendto)
* local IP (optional)
* receive queue
* send queue
* socket buffer sizes
* file descriptor

📌 **No network message exchanged. Nothing goes on the wire.**

---

# 2️⃣ **"Connection" Phase in UDP (Optional but Important)**

UDP supports `connect()` but it does **not** establish a connection.

### Example:

```c
connect(fd, server_ip, server_port);
```

What actually happens internally?

### ✔ Kernel sets:

* Default destination IP
* Default destination port
* Enables filtering: only packets from that IP:port are delivered
* Allows using `send()` instead of `sendto()`

### ❌ Kernel does NOT send:

* No SYN
* No handshake
* No state created on the server

### ✔ It’s more like:

**“I plan to talk only to that IP:port. Please autofil it.”**

---

# 3️⃣ **Data Transfer (The Real Part)**

## When client sends:

`sendto(fd, data, ..., dest_ip, dest_port)`
or
`send(fd, data, ...)` after `connect()`

### Kernel builds a UDP datagram:

| Layer      | Content                              |
| ---------- | ------------------------------------ |
| UDP Header | src port, dst port, length, checksum |
| IP Header  | src IP, dst IP, protocol=17          |
| Data       | user payload                         |

### Then OS sends it to NIC → router → network.

📌 **No ACK from receiver. No retransmit. No flow control.**

---

# 🧠 On the Receiver (Server) Side

Server does:

1. Kernel receives the packet into NIC RX ring.

2. NIC DMA copies to kernel memory.

3. IP layer verifies checksum & routing.

4. UDP layer checks destination port:

   * If there is a bound socket → deliver to that socket’s **receive buffer**
   * If not → send ICMP Port Unreachable (optional)

5. `recvfrom()` returns the datagram.

---

# 🔥 IMPORTANT: UDP Has No Stream — Every packet independent

* No ordering
* No merging
* No splitting
* Every datagram matches exactly one send → one receive

This is why streaming protocols (RTP, QUIC) add sequencing numbers.

---

# 4️⃣ **Closure (Socket Close) – Simple & Instant**

Application calls:

```c
close(fd);
```

Kernel frees:

* buffer queues
* file descriptor
* socket structure

📌 **No FIN packets. No TIME_WAIT. No lingering.**

Unlike TCP which needs:

* FIN/ACK exchange
* TIME_WAIT state
  UDP just deletes the socket.

---

# 🧩 Full UDP Socket Lifecycle Summary

### **Client and Server**

```
socket() -----------> creates socket structure in kernel
bind() --------------> assigns local port/IP
connect() (optional) -> sets default remote IP/port (no packets sent)
sendto()/send() -----> packets transmitted (no ACKs)
recvfrom()/recv() ---> packets delivered from RX buffer
close() -------------> socket destroyed (no packets sent)
```

---


# ⭐ UDP DOES NOT HAVE CONNECTIONS

Unlike TCP (which has a full state machine: SYN → ESTABLISHED → FIN → TIME_WAIT), UDP has:

* ❌ No handshake
* ❌ No connection state
* ❌ No FIN/ACK
* ❌ No sequence numbers
* ❌ No retransmissions
* ❌ No teardown

From the protocol standpoint, a UDP “connection” is simply:
👉 **A sender writing datagrams to an IP + port**
👉 **A receiver listening on a port**

**But the OS socket API *does* allow things like `connect()` on UDP.**
This does **NOT** create a connection in the network — it only binds the socket to a default remote IP/port.

We’ll explain that too.

---

# 🌟 Why People Think UDP Has “Connections”

Some protocols over UDP implement their own connection semantics:

* QUIC (HTTP/3)
* RTP/RTCP
* WireGuard
* DNS-over-QUIC
* Custom gaming protocols

They add:

* Handshake
* Reliability
* Retransmission
* Encryption
* Stream multiplexing

**But these are above UDP, not part of UDP.**

---

# ⚙️ What About NAT Keepalive?

Because UDP has no connection, NAT devices **delete UDP mappings quickly** (30–120 seconds).
So apps send:

* Empty packets
* Heartbeats
* STUN keepalives

This simulates “connections” to keep NAT state alive.

---

# ⭐ Deep Internals: How Linux Stores UDP Sockets

Linux stores UDP sockets in the **UDP hash table**, using:

* Local port
* Local IP
* Remote IP (if connected)

Packet delivery lookup is O(1).
Kernel maintains:

* per-socket RX queue
* socket buffers (sk_buff)
* memory pressure monitoring

No connection state machine like TCP’s huge set of states.

---

# 🔥 Final Summary — UDP Lifecycle in One Line

✔ **Creation:** socket(), bind() → kernel allocates socket
✔ **"Connect":** sets default remote address (no network activity)
✔ **Data:** send datagram → receive datagram (no guarantees)
✔ **Close:** socket removed instantly (no FIN, no state transitions)

---
