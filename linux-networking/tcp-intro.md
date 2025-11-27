
---

## 1. What is TCP?

**TCP = Transmission Control Protocol**

* **Layer:** Transport layer (Layer 4 in OSI).
* **Purpose:** Provide a **reliable, ordered, error-checked, full-duplex byte stream** between two endpoints (client and server).
* **Runs over:** IP (usually IPv4 or IPv6) → that’s why we say **TCP/IP**.
* **Endpoint identity:** `(source IP, source port, dest IP, dest port, protocol)`.

Key properties:

1. **Connection-oriented**

   * There is a **setup phase** (3-way handshake) before data.
   * Both sides maintain connection state (sequence numbers, windows, timers, etc.).

2. **Reliable**

   * Every byte sent is either:

     * Acknowledged by the receiver, or
     * Retransmitted by the sender if lost.
   * Uses sequence numbers + acknowledgements + retransmission timers.

3. **Ordered**

   * Data is delivered to the application in the **same order** it was sent.
   * If segments arrive out of order, TCP buffers and reorders them.

4. **Full-duplex**

   * Both sides can send data independently at the same time over the same connection.

5. **Flow control**

   * TCP ensures it doesn’t overwhelm the receiver by using a **receive window** advertised by the receiver.

6. **Congestion control**

   * TCP tries not to overwhelm the **network** using algorithms like slow start, congestion avoidance, etc.
   * It interprets packet loss / delay as signs of congestion and backs off.

---

## 2. TCP Segment – Important Fields

Each TCP packet (technically called a **segment**) has a header like:

* **Source Port (16 bits)** – e.g., 50012 (ephemeral client port)
* **Destination Port (16 bits)** – e.g., 80/443 for HTTP/HTTPS
* **Sequence Number (32 bits)** – index of first byte in this segment.
* **Acknowledgment Number (32 bits)** – “I have received everything up to byte N-1; next I expect byte N.”
* **Flags (bits):**

  * **SYN** – start connection
  * **ACK** – acknowledgment field is valid
  * **FIN** – finish (close) one direction of the connection
  * **RST** – reset (abort) connection
  * **PSH, URG** – push / urgent (rarely important for typical apps)
* **Window Size** – how many bytes the receiver is willing to accept (flow control).
* **Checksum** – detect corruption.
* **Options** – e.g., MSS (Max Segment Size), Window Scale, Timestamps, SACK.

---

## 3. How a TCP connection is established (3-way handshake)

Let’s take a concrete scenario:

* Client: `C` at IP `10.0.0.10`
* Server: `S` at IP `203.0.113.5`, listening on **port 80** (HTTP)

### Step 0 – Server gets ready (listen)

On the **server**:

1. Application (e.g., nginx) calls:

   * `socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)` → create TCP socket.
   * `bind(203.0.113.5, 80)`
   * `listen()` → put socket in **LISTEN** state.
2. Kernel now listens on `(203.0.113.5, 80)` for incoming TCP connections, maintaining a **SYN backlog queue**.

### Step 1 – Client send SYN

On the **client**:

1. App (e.g., browser) calls `connect("example.com", 80)`.
2. OS resolves `example.com` → `203.0.113.5` via DNS.
3. TCP stack picks a random **ephemeral port**, say `50012`.
4. Client sends:

   * `src = 10.0.0.10:50012`
   * `dst = 203.0.113.5:80`
   * Flags: **SYN = 1**
   * `Seq = x` (Initial Sequence Number, say `100000`)

   Client enters **SYN-SENT** state.

### Step 2 – Server replies with SYN+ACK

Server receives SYN:

1. It creates a **half-open connection entry** in the backlog (not yet visible to app).
2. It chooses its own initial sequence number `y` (say `300000`).
3. It responds:

   * `src = 203.0.113.5:80`
   * `dst = 10.0.0.10:50012`
   * Flags: **SYN = 1, ACK = 1**
   * `Seq = y`
   * `Ack = x + 1` (acknowledging client’s SYN)

   Server enters **SYN-RECEIVED** state.

### Step 3 – Client sends final ACK

Client receives SYN+ACK:

1. Verifies ACK = `x + 1`.

2. Sends:

   * `src = 10.0.0.10:50012`
   * `dst = 203.0.113.5:80`
   * Flags: **ACK = 1**
   * `Seq = x + 1` (no data yet)
   * `Ack = y + 1`

3. Client enters **ESTABLISHED** state.

4. Server receives this ACK, moves connection from backlog to main connection table, enters **ESTABLISHED**, and returns from `accept()` to the app with a new socket for that client.

Now **both sides are in ESTABLISHED**. A TCP connection is fully formed:

`(10.0.0.10:50012) <--> (203.0.113.5:80)`

---

## 4. How data transfer works (sequence, ACK, sliding window)

Now suppose client sends an HTTP request:

```http
GET / HTTP/1.1
Host: example.com

```

Assume this is 100 bytes for simplicity.

### Initial sequence numbers

From handshake:

* Client ISN: `x` → first usable byte seq is `x+1`
* Server ISN: `y` → first usable byte seq is `y+1`

### Client sends data

Client sends segment:

* `Seq = x + 1`
* `Ack = y + 1`  (still acknowledging server’s SYN)
* Payload length = 100 bytes.

This means:

* Bytes `[x+1 ... x+100]` are in this segment.
* Next byte from client would be `x+101`.

### Server ACKs data

Server receives it, passes the 100 bytes up to the HTTP server process, then sends an ACK:

* `Ack = x + 1 + 100 = x + 101`
* This says: “I have successfully received **up to** byte `x+100`; I’m now expecting `x+101`.”

### Sliding window & flow control

TCP uses a **sliding window**:

* Receiver advertises a **window size** `rwnd` (receive window), e.g., 64 KB.
* Sender is allowed to have **unacknowledged data** in flight up to `rwnd`.
* As ACKs come, the “window” slides forward and the sender can push more data.

So the sender can send multiple segments back-to-back:

* Segment 1: bytes 1–1460
* Segment 2: bytes 1461–2920
* ...

Even if ACK for segment 1 hasn’t arrived yet, as long as total bytes in flight ≤ `rwnd` (and also congestion window, `cwnd`).

### Reliability & retransmission

If a segment is **lost**:

* Sender has a **retransmission timer** for data.
* If timer expires without ACK, it **retransmits** that segment.
* Or faster: using **duplicate ACKs** / **fast retransmit** (if receiver keeps ACKing the same old sequence, indicating a missing segment).

Receiver can also advertise **zero window** (rwnd=0) to say “I’m full, stop sending” and later send a **window update** when ready.

### Congestion control (surface-level)

Aside from receiver window (`rwnd`), TCP also has **congestion window `cwnd`**:

* Starts small (slow start), grows quickly, then more cautiously.
* On loss, shrinks `cwnd` to reduce pressure on network.
* Effective sending window = `min(rwnd, cwnd)`.

---

## 5. How a TCP connection is closed

Closing is typically **4 steps** (half-close each direction):

1. **Client → FIN**

   * Client app calls `close()`.
   * Client TCP sends segment with `FIN=1, ACK=1`.
   * Client enters **FIN-WAIT-1**.

2. **Server ACKs FIN**

   * Server TCP sends `ACK` for FIN.
   * Client enters **FIN-WAIT-2** (client can’t send anymore, but can receive).
   * Server is now in **CLOSE-WAIT** (server app still may send remaining data).

3. **Server sends its own FIN**

   * When server app calls `close()`, TCP sends `FIN`.
   * Server goes to **LAST-ACK**.

4. **Client ACKs server’s FIN**

   * Client sends ACK.
   * Client moves to **TIME-WAIT** (waits ~2×MSL to avoid old delayed segments interfering).
   * Server receives ACK and goes to **CLOSED**.
   * After TIME-WAIT, client also goes to **CLOSED**.

There are other cases:

* **RST**: reset connection immediately (e.g., error, no such port, etc.).
* **Simultaneous close/open**: less common but defined in state machine.

---

## 6. Full client → server flow (step-by-step example)

Let’s walk through entire path when a browser connects to a web server via TCP:

### 6.1 Application layer decisions

**Client side (browser):**

1. User types `https://example.com/` in browser.
2. Browser:

   * Parses URL → scheme = HTTPS → use **port 443**.
   * DNS lookup: `example.com` → `203.0.113.5`.
   * Calls OS: `connect(203.0.113.5:443)` → ask TCP to establish a connection.

**Server side (web server):**

1. Web server process (nginx/apache) started earlier:

   * `socket()`, `bind(0.0.0.0:443)`, `listen()`.
   * Possibly configured behind a load balancer, but from TCP view it’s just an endpoint.

### 6.2 TCP 3-way handshake

1. Client → Server: **SYN**

   * src: `client_ip:ephemeral_port`
   * dst: `server_ip:443`
   * seq = `x`, SYN=1

2. Server → Client: **SYN + ACK**

   * src: `server_ip:443`
   * dst: `client_ip:ephemeral_port`
   * seq = `y`, ack = `x+1`, SYN=1, ACK=1

3. Client → Server: **ACK**

   * seq = `x+1`, ack = `y+1`, ACK=1

Now connection is **ESTABLISHED**.

### 6.3 Data exchange (e.g., HTTP over TCP)

1. **TLS handshake** happens next (because HTTPS) → but at TCP level this is just data bytes.
2. Once TLS is done, browser sends HTTP request over TCP:

   * `GET / HTTP/1.1`, headers, etc.
   * Might be multiple TCP segments, depending on size and MSS.
3. Server’s TCP stack receives, reorders if needed, verifies checksum, removes TCP headers, passes raw byte stream up to TLS, then HTTP server.
4. HTTP server generates response, writes to socket:

   * OS splits into TCP segments, assigns sequence numbers, sends.
5. Client TCP receives, ACKs segments, passes up to app, browser renders.

### 6.4 Connection close

After response:

* Either client or server can decide to close:

  * App calls `close()` on socket.
  * TCP sends FIN, goes through 4-step close we described.
* In HTTP/1.1 with keep-alive, multiple requests may happen over the same TCP connection before closing.

---

## TL;DR in one sentence

**TCP is a connection-oriented, reliable, ordered byte-stream protocol that uses a 3-way handshake to establish a connection, sequence numbers + ACKs + windows for reliable and controlled data transfer, and a 4-step FIN/ACK process to close the connection.**

---