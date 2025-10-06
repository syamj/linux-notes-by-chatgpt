Now, let’s cover the **monitoring and verification** part — i.e. how to check that your kernel tunings are actually applied, and how to **interpret** what you see.

We’ll go section-by-section:

---

## 🧠 1️⃣ Basic sysctl Verification

### 🔍 List all sysctl parameters (with values)

```bash
sysctl -a
```

* Shows **every tunable parameter** currently active.
* Use `grep` to filter:

  ```bash
  sysctl -a | grep somaxconn
  ```

### ✅ Verify a specific value

```bash
sysctl net.core.somaxconn
# Output example: net.core.somaxconn = 65535
```

Interpretation:

* If the value matches what you set in `/etc/sysctl.d/...conf`, it means the change is active.

### 🔁 Reload and recheck

```bash
sudo sysctl --system
sudo sysctl -p /etc/sysctl.d/99-haproxy-tuning.conf
```

* Loads all sysctl files again.
* Use `-p <file>` to reload only that file.

---

## 📊 2️⃣ File Descriptor / Connection Limits

### Check open file limits

```bash
ulimit -n
# Output: 1024 (default) or 2097152 (after tuning)
```

Meaning:

* Max number of file descriptors a process can open.
* Should match your tuning (`fs.file-max`) and also be raised in `/etc/security/limits.conf` for systemd services.

### Check system-wide limit

```bash
cat /proc/sys/fs/file-max
```

---

## 🌐 3️⃣ TCP and Network Parameter Checks

### Show TCP memory tuning

```bash
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem
```

Interpretation:

* Three numbers represent **min**, **default**, and **max** buffer sizes.
* If `max` is large (like 16 MB), the system can handle large windows for high throughput.

### Check ephemeral port range

```bash
sysctl net.ipv4.ip_local_port_range
# Example: net.ipv4.ip_local_port_range = 1024 65535
```

* Defines port range for outgoing connections.
* Wider range = fewer “address already in use” errors under load.

### Check connection backlog

```bash
sysctl net.core.somaxconn
sysctl net.core.netdev_max_backlog
```

* `somaxconn`: max length of the queue of pending connections waiting for accept().
* `netdev_max_backlog`: packets queued on the network interface before being processed by kernel.

If HAProxy/NGINX logs show “**listen queue full**” or “**dropping connections**,” increase these.

---

## 🕸️ 4️⃣ IP Forwarding and Bridge Checks (for Docker/K8s)

### Check IP forwarding

```bash
sysctl net.ipv4.ip_forward
```

* Must be `1` for Docker/Kubernetes networking.

### Check bridge netfilter

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

* Must be `1` to allow container traffic through iptables.

---

## 🚦 5️⃣ TCP State and Socket Monitoring

### Check current TCP connections (summary)

```bash
ss -s
```

Example output:

```
Total: 1234 (kernel 0)
TCP:   1024 (estab 900, closed 100, orphaned 0, synrecv 24, timewait 100)
```

Interpretation:

* **estab**: active connections.
* **timewait**: sockets waiting to close (can be reduced with `tcp_fin_timeout` and `tcp_tw_reuse`).

### View listening ports

```bash
ss -lntp
```

Shows:

* Protocol, local address, listening port, process name (e.g. nginx, haproxy).

### Detailed socket stats

```bash
ss -i
```

Shows per-socket info like congestion control (`bbr`/`cubic`), RTT, cwnd (congestion window).

---

## 📶 6️⃣ Network Interface Health

### Check backlog and dropped packets

```bash
netstat -s | grep -E "listen|drop|overflow"
```

or

```bash
cat /proc/net/netstat | grep Tcp
```

Interpretation:

* Drops/overflows = kernel queue too small → increase `somaxconn` or `netdev_max_backlog`.

### Interface stats

```bash
ip -s link
```

Shows RX/TX packets, errors, drops for each NIC.
If drops are increasing → tune network buffers.

---

## 🧮 7️⃣ Memory and Swappiness

### Check swap usage and swappiness

```bash
free -h
sysctl vm.swappiness
```

Interpretation:

* `vm.swappiness=0` → avoid swapping (recommended for K8s).
* `vm.swappiness=10` → mild swapping allowed (for Redis/HAProxy VMs).

### Check hugepages

```bash
cat /proc/meminfo | grep HugePages
```

* For most container workloads, `HugePages_Total` should be 0 unless explicitly enabled.

---

## ⚙️ 8️⃣ Validate TCP Congestion Control

### Check current congestion control

```bash
sysctl net.ipv4.tcp_congestion_control
# e.g. net.ipv4.tcp_congestion_control = bbr
```

### List all available algorithms

```bash
sysctl net.ipv4.tcp_available_congestion_control
```

If you see `bbr cubic reno`, you can switch:

```bash
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

---

## 🧾 9️⃣ Quick Verification Summary Script

You can create a quick status script `/usr/local/bin/sysctl-status`:

```bash
#!/bin/bash
echo "===== SYSCTL SUMMARY ====="
for p in net.core.somaxconn net.core.netdev_max_backlog \
         net.ipv4.ip_forward net.ipv4.tcp_fin_timeout \
         net.ipv4.ip_local_port_range net.ipv4.tcp_tw_reuse \
         fs.file-max vm.swappiness net.ipv4.tcp_congestion_control; do
  echo "$p = $(sysctl -n $p)"
done
```

Make it executable:

```bash
chmod +x /usr/local/bin/sysctl-status
/usr/local/bin/sysctl-status
```

---
