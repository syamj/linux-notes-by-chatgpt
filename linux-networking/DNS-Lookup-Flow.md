

# 🔍 DNS Lookup Flow (`dig` command)

When you run:

```bash
dig www.google.com
```

Here’s what happens step-by-step in a Linux system:

---

## 🧠 1. User Space Command Execution

* The `dig` command is a user-space utility (part of BIND tools).
* It performs DNS queries manually — bypassing glibc’s resolver library.
* You can specify:

  * A target domain (e.g., `www.google.com`)
  * A DNS server (optional)
  * Record type (e.g., `A`, `AAAA`, `CNAME`, etc.)

---

## ⚙️ 2. DNS Query Construction

* `dig` constructs a **DNS query packet** following RFC 1035.
* Packet includes:

  * Header (ID, flags, recursion desired, etc.)
  * Question section (domain name + record type)
* Uses **UDP** by default (port 53), unless:

  * Response exceeds 512 bytes → fallback to **TCP**
  * Or user specifies `+tcp`

---

## 🧩 3. Kernel Networking Stack (Outgoing Packet Path)

1. **Socket creation:**

   * `dig` calls `socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)`
2. **Binding:** The kernel assigns an ephemeral source port (e.g., 53000)
3. **Routing:**

   * Kernel consults the **routing table** (`ip route show`)
   * Finds the outgoing interface and next-hop (DNS server)
4. **Conntrack (if enabled):**

   * The UDP flow (`src:port` → `dst:53`) is tracked
5. **NIC Transmission:**

   * Packet enters NIC TX queue
   * NIC uses **multi-queue** or RSS (Receive Side Scaling) to distribute load across CPU cores

---

## 🌐 4. DNS Server Communication

* Packet reaches the DNS server (e.g., 8.8.8.8 or `/etc/resolv.conf` entry)
* The server checks:

  * Its **cache**
  * Or performs recursive resolution by contacting root → TLD → authoritative servers
* Response is built and sent back as a **UDP reply**

---

## 📥 5. Kernel Networking Stack (Incoming Packet Path)

1. NIC receives packet → sends interrupt to assigned CPU core
2. Kernel processes it via **netfilter hooks** (PREROUTING, INPUT)
3. Packet matched to socket using the 5-tuple (src/dst IP + ports)
4. Delivered to the user-space socket buffer

---

## 🧾 6. User Space Response Handling

* `dig` reads the socket using `recvfrom()`
* Parses DNS response
* Displays:

  * Question/Answer section
  * TTL, authoritative info, and query time

---

## 📘 7. Supporting Files Used

* `/etc/resolv.conf` → Lists nameservers to query
* `/etc/hosts` → Checked only by glibc resolver (not by `dig`)
* `/etc/nsswitch.conf` → Determines lookup order (also bypassed by `dig`)

---

# 🌍 HTTP Request Flow (`curl https://www.google.com`)

Let’s break down what happens when you run:

```bash
curl https://www.google.com
```

---

## 🧠 1. User Space Command Initialization

* `curl` is a user-space HTTP client.
* It uses **libcurl** underneath.
* It parses the URL and detects protocol → `https`.

---

## 🔍 2. Name Resolution (DNS)

* `curl` invokes glibc resolver (unlike `dig`):

  * Checks `/etc/nsswitch.conf` for lookup order (e.g., `hosts: files dns`)
  * Looks up `/etc/hosts` first
  * Falls back to DNS using nameservers from `/etc/resolv.conf`
* Uses **UDP 53** to query DNS.
* The response contains IP(s) for `www.google.com`.

---

## 🔒 3. TCP Connection Establishment

Once IP is resolved:

1. `curl` calls:

   ```c
   socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
   connect(fd, <google_ip:443>)
   ```
2. **Kernel performs 3-way handshake:**

   * SYN → SYN-ACK → ACK
   * Tracked in the **conntrack table**

---

## 🔐 4. TLS/SSL Handshake

Since protocol is HTTPS:

1. `curl` initiates **TLS handshake** with the server:

   * `ClientHello` → includes supported TLS versions, ciphers, extensions (SNI)
   * `ServerHello` → returns chosen cipher, certificate, etc.
2. **Certificate validation:**

   * Verified against `/etc/ssl/certs/ca-certificates.crt`
3. Once validated, session keys are exchanged (ECDHE).
4. Now, the connection is encrypted and ready for HTTP.

---

## 📤 5. HTTP Request

* `curl` sends:

  ```
  GET / HTTP/1.1
  Host: www.google.com
  User-Agent: curl/8.0
  Accept: */*
  ```
* Encrypted and sent via TCP.

---

## 📥 6. HTTP Response

* Google server responds with an HTTP response:

  ```
  HTTP/2 200
  content-type: text/html
  ```
* Data chunks are decrypted by TLS layer.
* TCP acknowledgments ensure reliable delivery.

---

## ⚙️ 7. Kernel-Level Path

### Outgoing:

1. User-space → system call (`send()`)
2. Kernel → TCP layer (segmentation, retransmission)
3. NIC TX queue (may use multi-queue for parallel transmission)

### Incoming:

1. NIC RX queue → interrupt → NAPI poll
2. Packet reconstructed by TCP stack
3. Data copied to user-space buffer via `recv()`

---

## 🧾 8. Supporting Config Files

| File                     | Purpose                                     |
| ------------------------ | ------------------------------------------- |
| `/etc/nsswitch.conf`     | Defines lookup order (`files dns`)          |
| `/etc/hosts`             | Local hostname resolution                   |
| `/etc/resolv.conf`       | DNS server list                             |
| `/etc/ssl/certs/`        | Trusted CA certificates                     |
| `/proc/net/tcp`          | Shows active TCP sockets                    |
| `/proc/net/nf_conntrack` | Shows connection tracking info (if enabled) |

---

## ⚡ Summary of Key System Interactions

| Layer      | Component                              | Example Functionality               |
| ---------- | -------------------------------------- | ----------------------------------- |
| User Space | `curl`, `glibc`, `openssl`             | Request generation, TLS, DNS        |
| Kernel     | `tcp`, `udp`, `netfilter`, `conntrack` | Packet routing, connection tracking |
| NIC        | Multi-queue, RSS                       | Parallel packet TX/RX               |
| Filesystem | `/etc/*`                               | Resolver and certificate info       |

---
