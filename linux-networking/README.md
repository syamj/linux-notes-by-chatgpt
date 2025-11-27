# Linux Networking — Index (by topic)

## OSI model files
- [osi-model.md](./osi-model.md)  
  - 🧱 OSI Model – The 7 Layers Explained (Application → Physical)  
  - Layer 7 — Application Layer  
  - Layer 6 — Presentation Layer  
  - Layer 5 — Session Layer  
  - Layer 4 — Transport Layer  
  - Layer 3 — Network Layer  
  - Layer 2 — Data Link Layer  
  - Layer 1 — Physical Layer  
  - OSI vs TCP/IP Comparison
- [osi-model-packet-flow.md](./osi-model-packet-flow.md)  
  - Real Packet Flow Example: Browser → Server (HTTP Request)  
  - STEP 1 – Application Layer (Layer 7) → STEP 7 – Physical Layer (Layer 1)  
  - Packet trace example & full step-by-step summary

## TCP
- [tcp-intro.md](./tcp-intro.md)  
  - What is TCP?  
  - TCP Segment — Important Fields  
  - 3-way handshake (connection establishment)  
  - Data transfer: sequence, ACK, sliding window  
  - Connection close flow
- [tcp-states.md](./tcp-states.md)  
  - TCP state machine (full list)  
  - State transitions (establish/terminate/RST)  
  - SYN backlog, TIME_WAIT deep dive  
  - Visual summary & comprehensive tables

## UDP
- [udp-intro.md](./udp-intro.md)  
  - What is UDP? — key characteristics and use cases  
  - UDP header and when to use UDP
- [udp-internals.md](./udp-internals.md)  
  - UDP socket lifecycle (creation → transfer → closure)  
  - Kernel internals, NAT keepalive, Linux UDP storage

## TLS / mTLS
- [tls.md](./tls.md)  
  - TLS layering and where it sits  
  - TLS 1.2 handshake (step-by-step)  
  - TLS 1.3 handshake and packet-level flow  
  - Keys generation and encrypted application data
- [tls-vs-mtls.md](./tls-vs-mtls.md)  
  - TLS vs mTLS — differences and packet-level comparison  
  - mTLS flow, CertificateVerify and when to use mTLS

## Network addressing & routing
- [Network-addressing-routing.md](./Network-addressing-routing.md)  
  - Unicast, Broadcast, Multicast, Anycast  
  - Summary table of addressing modes

## Enabling Multi‑Queue (NIC / driver tuning)
- [Enable-multiqueue-in-an-Ethernet-interface.md](./Enable-multiqueue-in-an-Ethernet-interface.md)  
  - Definition and why it exists  
  - Receive Side Scaling (RSS), NAPI, TX/XPS, conntrack impact  
  - Configuration examples and performance considerations

## ALPN (Application-Layer Protocol Negotiation)
- [alpn.md](./alpn.md)  
  - What is ALPN and where it sits in the TLS handshake  
  - How ALPN negotiates HTTP/1.1 vs HTTP/2 (and HTTP/3 / QUIC notes)  
  - Packet-level flow examples and Wireshark notes

## HTTP (versions & behavior)
- [HTTP-versions.md](./HTTP-versions.md)  
  - HTTP/1.1 — classic connection-oriented flow  
  - HTTP/2 — binary multiplexing and performance tradeoffs  
  - HTTP/3 — QUIC over UDP, advantages & challenges  
  - Side-by-side comparison and flow examples

## Cilium (eBPF networking & security)
- [Understanding-Cilium.md](./Understanding-Cilium.md)  
  - What is Cilium and core advantages as a CNI  
  - eBPF fundamentals and where Cilium hooks into the kernel  
  - Pod-to-Pod, Service lookup, DNAT, load balancing datapath steps  
  - Observability (Hubble), eBPF maps and example datapath

## Other useful flows & diagnostics
- [DNS-Lookup-Flow.md](./DNS-Lookup-Flow.md)  
  - dig lookup flow, kernel/user-space interactions, HTTP request flow example
- (Already indexed above) [Enable-multiqueue-in-an-Ethernet-interface.md](./Enable-multiqueue-in-an-Ethernet-interface.md)
- (Already indexed above) [udp-internals.md](./udp-internals.md)

---

