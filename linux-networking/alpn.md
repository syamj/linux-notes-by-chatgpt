
---

# ✅ 1. What is ALPN?

**Application-Layer Protocol Negotiation (ALPN)** is a TLS extension used to negotiate which application-layer protocol will run **inside the TLS tunnel**.

It lets the client say:

```
I support: ["http/1.1", "h2"]
```

And the server responds:

```
Chosen protocol: "h2"
```

### ALPN is used to negotiate:

* HTTP/1.1
* HTTP/2
* gRPC (which runs on HTTP/2)
* Redis-over-TLS
* MQTT-over-TLS
* Postgres-over-TLS
* etc.

**ALPN is not for HTTP/3** (explained later).

---

# 🔵 2. Where ALPN sits inside the TLS handshake?

ALPN is inside:

* **ClientHello** (client proposes protocols)
* **EncryptedExtensions** (TLS 1.3) or **ServerHello** (TLS 1.2) (server chooses)

### TLS 1.2 ALPN:

```
ClientHello:
    Extension: ALPN (["http/1.1", "h2"])
ServerHello:
    Extension: ALPN ("h2")       ← server picks protocol
```

### TLS 1.3 ALPN:

```
ClientHello:
    Supported extensions:
        ALPN: ["h2", "http/1.1"]

Server → Client:
    EncryptedExtensions:
        ALPN: "h2"               ← server response is encrypted
```

### Important:

**ALPN negotiation completes *before* encrypted application data starts.**

So the connection is already decided to be HTTP/1.1 or HTTP/2 before any request hits the server.

---

# 🔥 3. How HTTP/2 is negotiated with ALPN

The browser sends in ClientHello:

```
ALPN: ["h2", "http/1.1"]
```

The server replies with:

```
ALPN: "h2"
```

When both agree:

* HTTP/2 begins **inside** the TLS session
* Stream frames, HPACK header compression, multiplexing, etc.

If server does not support HTTP/2:

* Server returns `"http/1.1"`
* Client uses HTTP/1.1

If server supports HTTP/2 but does NOT advertise ALPN:

* Browsers will **not** use HTTP/2
  (they require ALPN or h2 is rejected)

---

# 🔥 4. Why HTTP/3 cannot be negotiated via ALPN inside TLS

HTTP/3 **does not use TCP**, therefore it does **not** use TLS handshake directly.

Instead:

* HTTP/3 runs over **QUIC**
* QUIC runs over **UDP**
* QUIC has **TLS 1.3 built inside it, not above it**

So ALPN can’t be used in normal TLS exchange because **TLS itself is not running over TCP for HTTP/3**.

### Where does ALPN live for HTTP/3?

Inside the **QUIC TLS handshake**, not the TCP handshake.

---

# 🔵 5. HTTP/3 ALPN negotiation (over QUIC)

HTTP/3 uses QUIC Initial packets that contain a **TLS 1.3 ClientHello** inside them.
Inside this, the client sends ALPN:

```
ALPN: ["h3", "h2", "http/1.1"]
```

Server replies with:

```
ALPN: "h3"
```

This differs from HTTP/2:

* HTTP/2 ALPN is in **TLS-over-TCP**
* HTTP/3 ALPN is in **TLS-over-QUIC-over-UDP**

---

# 🧠 6. Putting it all together:

## HTTP/1.1 – negotiated by ALPN

```
TCP → TLS → ALPN → "http/1.1"
```

## HTTP/2 – negotiated by ALPN

```
TCP → TLS → ALPN → "h2"
```

## HTTP/3 – negotiated by ALPN *inside QUIC*, not TCP

```
UDP → QUIC → (TLS 1.3 handshake inside QUIC) → ALPN → "h3"
```

HTTP/3 **never uses TCP**.

---

# 📡 7. Full packet-level flow (Side-by-Side)

## 🔹 HTTPS HTTP/2 (TLS 1.3 over TCP)

```
TCP: SYN → SYN+ACK → ACK
TLS: ClientHello (ALPN: ["h2", "http/1.1"])
TLS: ServerHello
TLS: EncryptedExtensions (ALPN: "h2")
TLS: Finished
TLS: Application Data (HTTP/2 frames)
```

## 🔹 HTTPS HTTP/3 (QUIC over UDP)

```
UDP: QUIC Initial
      ClientHello (ALPN: ["h3", "h2", "http/1.1"])

UDP: QUIC Initial
      ServerHello
      EncryptedExtensions (ALPN: "h3")
      Finished

QUIC: STREAM Frames (HTTP/3 frames)
```

---

# 🛰 8. Wireshark: How ALPN looks

### HTTP/2 over TLS:

```
Handshake Protocol: ClientHello
    Extension: application_layer_protocol_negotiation
        Protocols: h2, http/1.1

Handshake Protocol: EncryptedExtensions
    Extension: application_layer_protocol_negotiation
        Protocol: h2
```

### HTTP/3 over QUIC:

```
QUIC Initial Packet
    TLSv1.3 Record Layer
        Handshake: ClientHello
            ALPN: h3
```

---

# ⭐ Final Summary

### **TLS ALPN decides between HTTP/1.1 and HTTP/2.**

* Sent in ClientHello
* Server responds with chosen protocol
* Negotiated before encrypted traffic

### **HTTP/3 does NOT use TCP or TLS-over-TCP.**

* Uses QUIC over UDP
* QUIC embeds TLS 1.3 handshake
* ALPN ("h3") happens inside QUIC TLS, not in TCP TLS

### Side-by-Side:

| Protocol | Transport | Security Layer       | ALPN Location |
| -------- | --------- | -------------------- | ------------- |
| HTTP/1.1 | TCP       | TLS                  | ALPN in TLS   |
| HTTP/2   | TCP       | TLS                  | ALPN in TLS   |
| HTTP/3   | UDP       | QUIC (TLS1.3 inside) | ALPN in QUIC  |

---

