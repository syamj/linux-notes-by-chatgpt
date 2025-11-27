After the **TCP 3-way handshake** completes and both sides are in the **ESTABLISHED** state, **TLS begins**.
TLS sits *above TCP*, so everything in TLS is carried inside TCP segments as encrypted or plaintext records.

Let’s walk step-by-step through:

1. **Where TLS sits in the OSI model**
2. **What exactly happens after TCP handshake**
3. **TLS 1.2 handshake (classic)**
4. **TLS 1.3 handshake (modern, faster, default everywhere now)**
5. **What the data looks like on the wire**
6. **How encryption keys are derived**
7. **How application data (like HTTP) flows after TLS handshake**

This will give you a full network engineer + DevOps level understanding.

---

# 🔹 1. TLS Layering (Where it sits)

```
Application Layer (HTTP, gRPC, MySQL protocol)
---------------------------
TLS (outside OSI, sits between L4 & L7)
---------------------------
TCP (Layer 4)
IP  (Layer 3)
Ethernet (Layer 2)
```

TCP provides:

* reliable delivery
* ordering
* retransmissions
* segmentation

Then TLS provides:

* confidentiality
* integrity
* authentication
* optional client auth

---

# 🔹 2. What happens **immediately** after TCP handshake

Right after TCP is ESTABLISHED:

* Client: "I want to speak TLS"
* Client sends: **ClientHello** (first TLS packet)
* Server replies with **ServerHello**
* Then both negotiate algorithms, send certificates, do key exchange.
* When TLS handshake is done, both sides switch to **encrypted application data**.

---

# 🔹 3. Deep-Dive: TLS 1.2 Handshake (Legacy but still used)

TLS 1.2 handshake is longer and slower (2 RTTs).
Here is the exact sequence:

### Step 1 — Client → Server: **ClientHello**

Contains:

* TLS version supported
* Cipher suites supported
* Random number (ClientRandom)
* Extensions (SNI = hostname, ALPN = h2/http1.1, etc.)

### Step 2 — Server → Client: **ServerHello**

Contains:

* Chosen cipher suite
* Random number (ServerRandom)
* Chosen TLS version

### Step 3 — Server → Client: **Certificate**

Server sends its certificate chain:

* leaf cert
* intermediate
* root (optional)

### Step 4 — Server → Client: **ServerKeyExchange**

(used in ECDHE/DHE ciphers for forward secrecy)

Server sends:

* ephemeral key (ECDHE)
* signature

### Step 5 — Server → Client: **ServerHelloDone**

Indicates server is done with its part.

### Step 6 — Client → Server: **ClientKeyExchange**

Client generates:

* a PreMasterSecret (random bytes)
* encrypts it using server’s public key
* sends it

### Step 7 — Both derive keys:

Using:

* PreMasterSecret
* ClientRandom
* ServerRandom

They compute:

* master secret
* session keys
* MAC keys
* encryption keys

### Step 8 — Client sends:

* **ChangeCipherSpec**
* **Finished** (encrypted)

### Step 9 — Server sends:

* **ChangeCipherSpec**
* **Finished** (encrypted)

🔐 == TLS handshake complete ==
Everything after this point is encrypted.

---

# 🔹 4. TLS 1.3 (Modern) — Faster and Better

TLS 1.3 radically simplified the handshake.
Only **1 RTT** needed.

### TLS 1.3 packet flow:

### Step 1 — Client → Server: **ClientHello + KeyShare**

Client sends:

* cipher suites
* supported groups (curves)
* SNI
* ALPN
* **KeyShare (ECDHE pub key)**

### Step 2 — Server → Client:

* **ServerHello**
* **KeyShare**
* Encrypted Extensions
* Certificate
* CertificateVerify
* Finished

At this point, the server already has the shared key.

### Step 3 — Client → Server:

* Finished (encrypted)

Connection established.

### Performance advantage:

| Feature                 | TLS 1.2         | TLS 1.3                 |
| ----------------------- | --------------- | ----------------------- |
| Round trips             | 2 RTT           | 1 RTT                   |
| Algorithms              | Many, some weak | Only strong ciphers     |
| Perfect Forward Secrecy | Optional        | Always                  |
| Session resumption      | Complex         | 0-RTT/1-RTT fast resume |

---

# 🔹 5. What does the handshake look like on the wire?

### After TCP ESTABLISHED, wireshark or tcpdump shows:

```
ClientHello
ServerHello
Certificate
ServerKeyExchange
ServerHelloDone
ClientKeyExchange
ChangeCipherSpec
Finished
ChangeCipherSpec
Finished
```

or for TLS 1.3:

```
ClientHello
ServerHello, EncryptedExtensions, Certificate, CertificateVerify, Finished
Finished
```

Before “Finished”, TLS packets are plaintext.

After “Finished”, everything is encrypted:

```
Application Data (encrypted)
Application Data (encrypted)
Application Data (encrypted)
```

---

# 🔹 6. How keys are generated (deep but simple)

### For ECDHE (TLS 1.2 & 1.3):

Both sides have:

* private key: `a`, `b`
* public key: `A = a*G`, `B = b*G`

They exchange `A` and `B`.

Both compute the same shared secret:

```
S = a * B = b * A
```

Then through HKDF (HMAC-based key derivation) the following are derived:

* traffic keys
* MAC keys
* forward secrecy keys

TLS 1.3 uses a hierarchical HKDF:

```
early_secret → handshake_secret → master_secret → traffic keys
```

---

# 🔹 7. After TLS handshake → encrypted application data

Once handshake is done:

* Application protocols run over TLS, not raw TCP.
* Example: HTTP/1.1, HTTP/2, gRPC, MySQL, Redis-over-TLS.

All packets now look like:

```
TLS Application Data
TLS Application Data
TLS Application Data
```

Inside these encrypted records are:

* HTTP requests
* HTTP responses
* JSON, protobuf, etc.

TCP still provides:

* retransmission
* ordering
* segmentation

TLS only sees a **byte stream**, not packets.

---

# 🔹 Full Flow: TCP + TLS handshake together

### Complete sequence:

```
TCP: SYN → SYN+ACK → ACK      (TCP is now ESTABLISHED)

TLS: ClientHello
TLS: ServerHello
TLS: Certificate
TLS: ServerKeyExchange
TLS: ServerHelloDone
TLS: ClientKeyExchange
TLS: ChangeCipherSpec
TLS: Finished
TLS: ChangeCipherSpec
TLS: Finished               (TLS is now secure)

HTTP: GET /   (encrypted now)
HTTP: 200 OK  (encrypted)
```

---