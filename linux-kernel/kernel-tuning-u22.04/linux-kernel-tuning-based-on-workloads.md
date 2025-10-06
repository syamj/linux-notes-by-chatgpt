Perfect 👍 Let’s go **one by one** — we’ll prepare optimized `tuning.conf` (i.e. `/etc/sysctl.d/tuning.conf`) for **each component**:

* ✅ HAProxy
* ✅ NGINX
* ✅ Docker (with containerd)
* ✅ Kubernetes (with containerd runtime)

Each file will:

* Be **optimized for high throughput + low latency** workloads
* Include **default values as comments** for clarity
* Include **rationale** for tuning

---

## 🧩 1️⃣ HAProxy – `/etc/sysctl.d/99-haproxy-tuning.conf`

```bash
# ==========================
# HAProxy Kernel Tuning
# ==========================

# Allow more connections in the backlog queue
# default: 128
net.core.somaxconn = 65535

# Increase max incoming connection queue size
# default: 4096
net.core.netdev_max_backlog = 16384

# Enable reuse/recycle of TCP connections in TIME_WAIT
# default: 0
net.ipv4.tcp_tw_reuse = 1

# Enable fast recycling of TIME_WAIT sockets
# default: 0
net.ipv4.tcp_tw_recycle = 0   # deprecated in newer kernels, keep off for safety

# Increase ephemeral port range for outgoing connections
# default: 32768 60999
net.ipv4.ip_local_port_range = 1024 65535

# Reduce TCP FIN timeout (for faster connection cleanup)
# default: 60
net.ipv4.tcp_fin_timeout = 15

# Enable TCP SYN cookies to protect against SYN flood
# default: 1
net.ipv4.tcp_syncookies = 1

# Increase memory for TCP buffers
# default: 4096 87380 6291456
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Allow reuse of sockets in TIME_WAIT state for new connections (safe)
# default: 0
net.ipv4.tcp_tw_reuse = 1

# Disable ICMP redirects
# default: 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
```

**Goal:** Maximize concurrent TCP connections, reduce latency for short-lived requests.

---

## 🌐 2️⃣ NGINX – `/etc/sysctl.d/99-nginx-tuning.conf`

```bash
# ==========================
# NGINX Kernel Tuning
# ==========================

# Allow many queued connections before accept() is called
# default: 128
net.core.somaxconn = 65535

# Increase network backlog for high-load situations
# default: 1000
net.core.netdev_max_backlog = 16384

# Optimize TCP parameters for keepalive and reuse
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# Increase max open files (important for many clients)
# default: 1024 (system dependent)
fs.file-max = 2097152

# Increase ephemeral port range
net.ipv4.ip_local_port_range = 1024 65535

# Allow more memory for TCP buffers
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Reduce swap usage
vm.swappiness = 10

# Disable source route verification and redirects
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
```

**Goal:** Handle thousands of concurrent connections efficiently with minimal latency.

---

## 🐳 3️⃣ Docker (using containerd) – `/etc/sysctl.d/99-docker-tuning.conf`

```bash
# ==========================
# Docker / Containerd Kernel Tuning
# ==========================

# Enable IP forwarding for container networking
# default: 0
net.ipv4.ip_forward = 1

# Enable bridge call for iptables
# default: 0
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# Allow large connection backlog
net.core.somaxconn = 65535

# Allow more queued packets before being dropped
net.core.netdev_max_backlog = 16384

# Increase number of open files
fs.file-max = 2097152

# Reduce swappiness (avoid swapping containers)
vm.swappiness = 10

# Allow reuse of TIME_WAIT sockets
net.ipv4.tcp_tw_reuse = 1

# Increase range of ephemeral ports
net.ipv4.ip_local_port_range = 1024 65535

# Enable cgroup memory accounting
# Required for Docker memory limits
vm.overcommit_memory = 1
```

**Goal:** Support high-density container workloads with stable network and memory behavior.

---

## ☸️ 4️⃣ Kubernetes (with containerd) – `/etc/sysctl.d/99-k8s-tuning.conf`

```bash
# ==========================
# Kubernetes Kernel Tuning
# ==========================

# Enable IP forwarding and bridge netfilter
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# Increase max open file descriptors
fs.file-max = 2097152

# Allow enough network backlog and TCP queues
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 16384

# Allow reuse of sockets in TIME_WAIT
net.ipv4.tcp_tw_reuse = 1

# Enable TCP BBR congestion control for better throughput
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Increase ephemeral port range
net.ipv4.ip_local_port_range = 1024 65535

# Optimize TCP memory
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Reduce swap usage (Kubernetes requires swap off)
vm.swappiness = 0

# Enable transparent hugepages defrag off (for stable memory)
vm.nr_hugepages = 0
```

**Goal:** Stability + high performance for containerized workloads and inter-pod communication.

---

### 💡 Apply All Changes

After placing the file (e.g. `/etc/sysctl.d/99-k8s-tuning.conf`):

```bash
sudo sysctl --system
```

---
