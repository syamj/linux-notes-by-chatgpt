
---

# 1. TCP STATE MACHINE — HIGH-LEVEL

TCP uses a **finite state machine (FSM)** defined in RFC 793 + later extensions.

The core idea:

* A TCP connection is not just “open” or “closed”.
* It transitions through **intermediate states** based on:

  * SYN / SYN-ACK / ACK
  * FIN / FIN-ACK / ACK
  * RST

These states help TCP maintain correctness and reliability.

---

# 2. **TCP STATES: Full List with Deep Explanation**

| **State**                              | **Meaning**                                                                                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **CLOSED**                             | No connection exists. Initial state of TCP endpoints.                                                                                    |
| **LISTEN**                             | Server is waiting for incoming SYN packets. Only servers enter LISTEN.                                                                   |
| **SYN-SENT**                           | Client has sent SYN and is waiting for SYN+ACK.                                                                                          |
| **SYN-RECEIVED**                       | Server received a SYN, sent SYN+ACK, awaiting ACK from client. Also seen in simultaneous open.                                           |
| **ESTABLISHED**                        | Data transfer state. Connection fully open. Both sides can send and receive.                                                             |
| **FIN-WAIT-1**                         | One side has sent FIN (start of close) and is waiting for ACK or FIN-ACK.                                                                |
| **FIN-WAIT-2**                         | FIN was acknowledged; now waiting for the other side's FIN.                                                                              |
| **CLOSE-WAIT**                         | Received FIN from peer; application must close its side next.                                                                            |
| **LAST-ACK**                           | Have sent FIN after receiving FIN; waiting for final ACK.                                                                                |
| **CLOSING**                            | Rare: both sides send FIN simultaneously; waiting for ACK.                                                                               |
| **TIME-WAIT**                          | After sending final ACK for peer’s FIN, endpoint waits 2 * MSL (default ~120s) to ensure late packets do not corrupt future connections. |
| **(Deleted in newer RFCs) DELETE-TCB** | (Implementation detail) Cleanup.                                                                                                         |

---

# 3. STATE TRANSITIONS

### **3.1 GRACEFUL CONNECTION ESTABLISHMENT (3-Way Handshake)**

### Client

1. **CLOSED → SYN-SENT**
   After sending SYN.
2. **SYN-SENT → ESTABLISHED**
   After receiving SYN+ACK and sending final ACK.

### Server

1. **CLOSED → LISTEN**
2. **LISTEN → SYN-RECEIVED**
   Received SYN, replied SYN+ACK.
3. **SYN-RECEIVED → ESTABLISHED**
   Received final ACK.

---

# 4. GRACEFUL CONNECTION TERMINATION (4-Way Close)

Let’s assume **client initiates close**:

### **CLIENT STATES**

1. ESTABLISHED → **FIN-WAIT-1**
   Client sends FIN.
2. FIN-WAIT-1 → **FIN-WAIT-2**
   Client receives ACK for its FIN.
3. FIN-WAIT-2 → **TIME-WAIT**
   Client receives FIN from server and sends ACK.
4. TIME-WAIT → **CLOSED**
   After waiting 2MSL.

### **SERVER STATES**

1. ESTABLISHED → **CLOSE-WAIT**
   Server receives client’s FIN, sends ACK.
2. CLOSE-WAIT → **LAST-ACK**
   Application calls close(); server sends FIN.
3. LAST-ACK → **CLOSED**
   Receives ACK for its FIN.

---

# 5. ABRUPT TERMINATION (RST)

An endpoint may send **RST** when:

* Application closes socket with `SO_LINGER=0`
* Bad packet is received (wrong seq, wrong port)
* Connecting to a closed port
* Timeout or internal abort

### Effects:

* **RST immediately kills the connection**.
* No FIN handshake.
* No TIME_WAIT (unless delivered in closing stage).

Examples:

### Client → RST

Server transitions:

* ESTABLISHED → CLOSED
  No LAST-ACK / no FIN.

### Server → RST

Client transitions:

* ESTABLISHED → CLOSED
  No FIN-WAIT / no TIME-WAIT.

Abrupt, dirty shutdown.

---

# 6. DEEP DIVE: SYN BACKLOG (Listen Queue)

When server is in **LISTEN**, it maintains two queues:

---

## **Queue 1: SYN Queue (a.k.a. “half-open” backlog)**

Stores connections in **SYN-RECEIVED** state.

Flows into this queue:

* Server received **SYN**
* Server replied **SYN+ACK**
* Now waiting for final ACK

If this queue fills up:

* New SYNs are dropped → connection attempts fail.
* Protection mechanism → **SYN cookies** (stateless SYN handling).

---

## **Queue 2: Accept Queue (“completed” queue)**

Connections move here **only after server gets final ACK** and enters **ESTABLISHED**.

This is the queue `accept()` pulls from.

---

## SYN Flood Example

Attacker sends millions of SYN packets:

* Fill the server's SYN backlog
* Server wastes memory waiting for final ACK
* Legit clients cannot connect

Mitigation:

* SYN Cookies
* Lower SYN-RETRANS
* Increase backlog

---

# 7. TIME_WAIT — THE MOST IMPORTANT TCP STATE

### TIME_WAIT belongs to **the side that sent the final ACK**.

Usually:

* **Client** when client initiates close.
* **Server** when server initiates close.

---

## WHY does TIME_WAIT exist?

### **Reason 1 — Retransmitted FIN Handling**

If server retransmits its FIN (because ACK was lost), client must still be alive to ACK it.

### **Reason 2 — Prevent port reuse collision**

Imagine:

* old connection `(A:5000 → B:80)`
* later a new connection reuses `A:5000`

If late segments from old connection arrive, they would corrupt the new connection unless TIME_WAIT is enforced.

---

## Duration

Default: **2 × Maximum Segment Lifetime (MSL)**
Usually **60 sec × 2 = 120 sec**.

Some OSes use:

* Linux: **TIME_WAIT = 60s**
* BSD: 120s

---

## Problems caused by TIME_WAIT

1. **SOCK Exhaustion** (common on load balancers)
2. **Port exhaustion** for NAT gateways
3. **Slow reconnect after close**

Mitigation:

* `net.ipv4.tcp_tw_reuse = 1`
* `net.ipv4.tcp_tw_recycle` (deprecated)
* Use `SO_REUSEPORT`
* Long-lived connections (HTTP/2, keep-alive)

---

# 8. COMPREHENSIVE TABLE OF STATES (Client + Server)

---

## **A. Normal Connection (3-way handshake)**

| Step | Client State | Server State | Event                                 |
| ---- | ------------ | ------------ | ------------------------------------- |
| 1    | CLOSED       | LISTEN       | Server ready                          |
| 2    | SYN-SENT     | LISTEN       | Client sends SYN                      |
| 3    | SYN-SENT     | SYN-RECEIVED | Server receives SYN and sends SYN+ACK |
| 4    | ESTABLISHED  | SYN-RECEIVED | Client sends final ACK                |
| 5    | ESTABLISHED  | ESTABLISHED  | Server receives ACK                   |

---

## **B. Graceful Close (Client initiates)**

| Step | Client State | Server State | Event                           |
| ---- | ------------ | ------------ | ------------------------------- |
| 1    | ESTABLISHED  | ESTABLISHED  | Client sends FIN                |
| 2    | FIN-WAIT-1   | CLOSE-WAIT   | Server receives FIN, sends ACK  |
| 3    | FIN-WAIT-2   | CLOSE-WAIT   | Client receives ACK             |
| 4    | FIN-WAIT-2   | LAST-ACK     | Server app closes → FIN sent    |
| 5    | TIME-WAIT    | LAST-ACK     | Client receives FIN → sends ACK |
| 6    | TIME-WAIT    | CLOSED       | Server gets ACK                 |
| 7    | CLOSED       | CLOSED       | Client TIME-WAIT expires        |

---

## **C. Graceful Close (Server initiates)**

| Step        | Client State | Server State            |
| ----------- | ------------ | ----------------------- |
| ESTABLISHED | ESTABLISHED  | Server sends FIN        |
| CLOSE-WAIT  | FIN-WAIT-1   | Client sends ACK        |
| CLOSE-WAIT  | FIN-WAIT-1   | Client app closes → FIN |
| LAST-ACK    | FIN-WAIT-2   |                         |
| CLOSED      | TIME-WAIT    |                         |
| CLOSED      | CLOSED       |                         |

---

## **D. Simultaneous Close (rare)**

| Step        | Client      | Server           |
| ----------- | ----------- | ---------------- |
| ESTABLISHED | ESTABLISHED |                  |
| FIN-WAIT-1  | FIN-WAIT-1  | Both send FIN    |
| CLOSING     | CLOSING     | Both waiting ACK |
| TIME-WAIT   | TIME-WAIT   |                  |
| CLOSED      | CLOSED      |                  |

---

## **E. Abrupt Close (RST)**

### Client receives RST from server:

| Client               | Server               |
| -------------------- | -------------------- |
| ESTABLISHED → CLOSED | ESTABLISHED → CLOSED |

### Client sends RST:

| Client               | Server               |
| -------------------- | -------------------- |
| ESTABLISHED → CLOSED | ESTABLISHED → CLOSED |

---

# 9. Visual Summary (Best Mental Model)

```
         ESTABLISHED
         /           \
  FIN → /             \ ← FIN
       v               v
 FIN-WAIT-1       CLOSE-WAIT
       v               |
       v               | close()
 FIN-WAIT-2            v
       v           LAST-ACK → CLOSED
       v
   TIME-WAIT → CLOSED
```

---
