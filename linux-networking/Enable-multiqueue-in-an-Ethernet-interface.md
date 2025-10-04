**multiqueue** in an Ethernet interface 

## 1. Definition

**Multiqueue NICs** are network interfaces that have **multiple hardware transmit (TX) and receive (RX) queues**.

* Instead of a single queue for all incoming and outgoing packets, the NIC splits traffic into **multiple parallel queues**.
* Each queue can be processed by **a separate CPU core**, improving throughput and reducing CPU bottlenecks.

---

## 2. Why it exists

* Modern servers have **many CPU cores** and **10/25/40/100Gbps NICs**.

* Single queue NICs can only process packets in **one CPU core**, which limits:

  * Maximum throughput
  * Interrupt handling efficiency
  * Packet processing speed

* Multiqueue solves this by **parallelizing packet processing across multiple cores**.

---

## 3. How it works

1. **Receive side (RX queues)**:

   * NIC assigns incoming packets to different RX queues based on:

     * **RSS (Receive Side Scaling) hash**: uses headers like source/destination IP, port, protocol.
     * Each queue has a separate **interrupt / NAPI poll**.
   * Each RX queue is processed by a **different CPU core**.
   * Reduces lock contention and improves cache locality.

2. **Transmit side (TX queues)**:

   * Multiple queues for outgoing packets.
   * Each CPU core can enqueue packets to a separate TX queue.
   * NIC sends packets from these queues independently.

3. **Interrupt Handling**:

   * Each queue can generate its own interrupt or use **MSI-X** to map interrupts to CPU cores.

---

## 4. Linux support

* Modern Linux kernels fully support multiqueue NICs.
* Commands to check:

  ```bash
  ethtool -l eth0        # shows number of RX/TX queues
  cat /sys/class/net/eth0/queues/        # see individual queues
  ```
* Multiqueue works best when combined with:

  * **IRQ affinity / smp_affinity** → bind queues to CPU cores
  * **RSS** → distribute flows evenly across queues
  * **XPS/ RPS** → software TX/RX steering

---

## 5. Benefits

| Feature                 | Benefit                                                          |
| ----------------------- | ---------------------------------------------------------------- |
| Parallel RX/TX queues   | Use multiple CPU cores simultaneously                            |
| RSS hash                | Packets from same flow go to same queue → preserves packet order |
| Reduced lock contention | Each queue handled independently                                 |
| High throughput         | NIC can handle multiple 10/25/40/100Gbps flows efficiently       |

---

## 6. Practical example

* Server with **4 cores** and **10Gb NIC**.
* NIC has **4 RX queues and 4 TX queues**.
* Flow from client A goes to queue 0 → CPU0 processes it.
* Flow from client B goes to queue 1 → CPU1 processes it.
* Result: both flows processed in parallel, higher throughput, lower latency.

---

✅ **In short**:
Multiqueue Ethernet NICs split packet handling into **multiple RX/TX queues**, each of which can be processed independently by different CPU cores. This improves **parallelism, throughput, and CPU efficiency**, especially for servers handling **high-speed or high-volume traffic**.

---

Let’s now connect **multiqueue NICs** with the **Linux networking stack**, including **conntrack, RSS, and packet flow**. This is crucial for high-performance packet processing.

---

## 1. Overview of packet flow in Linux

When a packet arrives at a multiqueue NIC:

```
[NIC RX queue] → [NIC driver / NAPI poll] → [Kernel network stack] → [Socket / conntrack / iptables / routing] → [Application]
```

Now let’s see how **multiqueue and CPU parallelism** fit in.

---

## 2. **Receive Side Scaling (RSS)**

* NIC computes a **hash** on each incoming packet (usually using **5-tuple**: src IP, dst IP, src port, dst port, protocol).
* RSS assigns packets from the **same flow** to the **same RX queue** → preserves packet ordering.
* Each RX queue is assigned to a **CPU core** via **IRQ affinity**.
* Benefits:

  * Multiple flows processed in parallel.
  * Reduces cache misses and lock contention.
  * Packet ordering maintained per flow.

---

## 3. **NAPI (New API) and RX queue processing**

* Each RX queue has a **NAPI context**:

  * NIC raises interrupt → NAPI poll starts processing packets.
  * Kernel dequeues packets from the RX queue.
  * NAPI reduces **interrupt storm** under high traffic.

* Multiple RX queues → multiple cores polling in parallel.

---

## 4. **Connection Tracking (conntrack) impact**

* Conntrack is part of **netfilter** for **stateful packet inspection** and NAT.
* Each packet is looked up in **conntrack table** to determine state (NEW, ESTABLISHED, RELATED).
* Kernel needs **locks** to protect the conntrack table.
* **High packet rate** can become a bottleneck:

  * Multiqueue + RSS ensures packets from different flows go to different CPUs.
  * Each CPU can access different conntrack buckets → less lock contention.
* For large-scale connections, **nf_conntrack hash table can be tuned** to reduce collisions.

---

## 5. **Routing and policy lookups**

* After conntrack, packet hits **routing table / policy routing rules**.
* Kernel does **longest prefix match** to determine egress interface.
* Multiqueue + RPS (Receive Packet Steering) can steer packets to the CPU that will **transmit the reply** → reduces cross-CPU cache misses.

---

## 6. **Transmit Side Queues (TX) and XPS**

* Once packet is ready to send:

  * TX queue is chosen, ideally **bound to the same CPU** that processed the RX packet.
  * XPS (Transmit Packet Steering) ensures CPU → TX queue affinity.
  * Minimizes lock contention and maximizes throughput.

---

## 7. **Software optimizations in Linux**

* **RPS (Receive Packet Steering)**:

  * Allows kernel to **re-distribute packets from RX queue** to different CPUs in software if needed.
* **SOCKMAP / AF_XDP**:

  * Bypass kernel for ultra-low-latency packet processing.
* **IRQ affinity**:

  * Each RX/TX queue interrupt is mapped to a specific CPU.
* **Large conntrack tables & hash tuning**:

  * Needed for thousands/millions of concurrent flows.

---

## 8. **Summary diagram (logical flow)**

```
           Multiqueue NIC
        --------------------
 RX Queue 0 → CPU0 → NAPI → conntrack → routing → TX Queue 0 → CPU0 → NIC
 RX Queue 1 → CPU1 → NAPI → conntrack → routing → TX Queue 1 → CPU1 → NIC
 RX Queue 2 → CPU2 → NAPI → conntrack → routing → TX Queue 2 → CPU2 → NIC
 ...       → ... → ...   → ...       → ...
```

* RSS ensures **flow-to-queue mapping**
* Conntrack works **per-flow**, CPU-local buckets reduce contention
* Routing + TX queue + XPS → CPU-local transmit

---

## ✅ Key Takeaways

1. **Multiqueue NICs**: enable parallel RX/TX for multiple CPU cores.
2. **RSS**: ensures flow-based ordering → prevents packet reordering.
3. **Conntrack**: uses hash tables; parallel CPU access reduces locking bottlenecks.
4. **RPS/XPS**: software steering to improve CPU locality.
5. **High-speed networks** (10G/25G/100G) require all these features to avoid **single-core saturation**.

---
