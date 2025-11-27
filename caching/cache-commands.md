Below is a **complete, practical list of commands** to view **every major Linux cache live** — page cache, buffers, slab, dentries, inodes, swap cache, CPU caches, ARP cache, route cache, conntrack, filesystem metadata, and more.
You can run these commands directly on any Linux system (physical or virtual). No special tools needed beyond standard Linux utilities.

---

# ✅ **1. View Page Cache + Buffer Cache (File Data Cache)**

### Memory summary

```bash
free -h
```

Look at:

* **buff/cache** → page cache + buffer cache

### Detailed page cache stats

```bash
cat /proc/meminfo | egrep "Cached|Buffers|Dirty|Writeback"
```

Useful fields:

* `Cached:` → Page cache
* `Buffers:` → Buffer cache
* `Dirty:` → Pages waiting to be flushed
* `Writeback:` → Pages currently being flushed

---

# ✅ **2. View Slab Cache (Kernel Object Cache)**

### Live, interactive view (best)

```bash
slabtop
```

Key entries:

* **dentry** (dentry cache)
* **inode_cache** (inode cache)
* **kmalloc-* pools**
* **filp** (file descriptors)
* **task_struct**

### Raw slab stats

```bash
cat /proc/slabinfo
```

---

# ✅ **3. View Dentry Cache specifically**

```bash
cat /proc/slabinfo | grep -i dentry
```

---

# ✅ **4. View Inode Cache**

```bash
cat /proc/slabinfo | grep -i inode
```

---

# ✅ **5. View Swap Cache (what's currently swapped out)**

### To see pages in swap cache:

```bash
grep -E "SwapCached" /proc/meminfo
```

### To see actual swapped-out pages (per process):

```bash
cat /proc/<pid>/smaps | grep Swap
```

---

# ✅ **6. View Read-Ahead Cache Settings**

(Not a “cache” but controls how much data OS prefills)

```bash
sudo blockdev --getra /dev/sda
```

For all disks:

```bash
lsblk -o NAME,RA
```

---

# ✅ **7. View Writeback Cache (Dirty Pages)**

### Dirty + writeback pages:

```bash
cat /proc/meminfo | grep -E "Dirty|Writeback"
```

### Kernel writeback activity:

```bash
cat /proc/vmstat | grep -E "dirty|writeback"
```

---

# ✅ **8. View VFS Cache usage (all file system caches)**

Count dentries:

```bash
grep -i dentry /proc/slabinfo
```

Count inodes:

```bash
grep -i inode /proc/slabinfo
```

FS pressure:

```bash
cat /proc/sys/fs/dentry-state
```

---

# ✅ **9. View CPU Caches (Hardware L1/L2/L3)**

### CPU cache hierarchy

```bash
lscpu | grep -i 'cache'
```

### Detailed per-core cache info

```bash
ls -l /sys/devices/system/cpu/cpu0/cache
```

View e.g. L1/L2 sizes:

```bash
cat /sys/devices/system/cpu/cpu0/cache/index0/size
cat /sys/devices/system/cpu/cpu0/cache/index1/size
cat /sys/devices/system/cpu/cpu0/cache/index2/size
cat /sys/devices/system/cpu/cpu0/cache/index3/size
```

---

# ✅ **10. View ARP Cache (Neighbor table)**

```bash
ip neigh
```

Or detailed:

```bash
arp -n
```

---

# ✅ **11. View Route Cache (Linux FIB)**

Modern kernels:

```bash
ip route show cache
```

Full table:

```bash
ip route
```

Older kernels (pre-3.x used a route cache):

```bash
cat /proc/net/rt_cache
```

---

# ✅ **12. View Connection Tracking Cache (conntrack)**

(This is huge on Kubernetes nodes)

Installed via `conntrack-tools`:

```bash
sudo conntrack -L
```

Stats:

```bash
sudo conntrack -S
```

Table:

```bash
cat /proc/net/nf_conntrack
```

---

# ✅ **13. View DNS Cache**

If using systemd-resolved:

```bash
systemd-resolve --statistics
```

Clear it:

```bash
sudo systemd-resolve --flush-caches
```

---

# ✅ **14. View Filesystem-specific Caches**

### ext4

```bash
cat /proc/fs/ext4/<device>/mb_groups
```

### XFS

```bash
xfs_info /
xfs_db -c 'inode 0' -c 'print'
```

---

# ✅ **15. View Bcache (SSD/HDD caching layer)**

Installed systems:

```bash
cat /sys/fs/bcache/*/stats_total/*
```

---
# 🧹 **How to Clear Linux Caches — Commands & Safety Guide**
Below is a **complete, safe, SRE-grade guide** on how to **clear every major Linux cache** — **what commands to use, what they actually do, and when you SHOULD NOT clear them**.
---

# 🚨 **General Warning**

Clearing caches **forces Linux to re-fetch data from disk/CPU/memory**, so:

* Apps may slow down temporarily
* Disk I/O spikes
* Latency increases
* OS will quickly rebuild caches anyway

**Do NOT clear caches on production unless you have a specific reason.**

---

# ✅ **1. Clear Page Cache (File Data Cache)**

### 🔹 Clear only page cache (recommended safest)

```bash
sudo sync
echo 1 | sudo tee /proc/sys/vm/drop_caches
```

### 🔹 What it does

* Drops file data pages (cached file contents)
* Keeps slab cache (kernel objects) intact

### 🔹 When safe

* Testing disk I/O performance
* After copying large files
* Debugging caching issues
* Benchmarking applications

### 🔹 When NOT to do it

* On production systems
* On database servers
* On Kubernetes nodes handling traffic

---

# ✅ **2. Clear Dentry Cache + Inode Cache (Path + metadata cache)**

```bash
sudo sync
echo 2 | sudo tee /proc/sys/vm/drop_caches
```

### 🔹 What it does

* Clears:

  * Directory entry cache (dentries)
  * Inode cache (metadata)
* Page cache remains intact

### 🔹 Safe use cases

* Filesystem benchmarks
* After massive file deletions

### ❌ NOT safe when

* High inode workloads (web servers, DBs, Kubernetes nodes)
* It will slow down path resolution & file operations

---

# ✅ **3. Clear Page Cache + Dentry + Inode (full VFS cache reset)**

```bash
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

### 🔹 What it does

* Flushes EVERYTHING (except slab kernel objects)
* Forces disk access for next operations

### ❌ **Avoid on any real environment** unless:

* You're doing FS-level benchmarking
* Troubleshooting heavy FS caching issues

---

# ❄️ **Understanding `sync`**

`sync` flushes all dirty buffers to disk.
You ALWAYS run it before dropping caches to avoid data loss.

---

# ✅ **4. Clear Slab Cache (Kernel Objects)**

You cannot directly delete slab memory with `drop_caches`.
But you can shrink caches:

```bash
echo 2 | sudo tee /sys/kernel/slab/*/shrink
```

### Slab objects include:

* dentries
* inodes
* task_struct
* file descriptors
* socket objects

### ❌ HIGH RISK

Shrinking slab caches too aggressively can:

* Stall kernel operations
* Slow down network stack
* Slow down process creation / file opens

Only use for:

* Slab debugging
* Memory leak debugging
* Extreme OOM situations

---

# ✅ **5. Clear Swap Cache**

### Swap cache = pages in RAM that are also in swap.

Clear by forcing swap off + on:

```bash
sudo swapoff -a
sudo swapon -a
```

### ❌ HIGH RISK

* Causes paging storms
* FREEZES the system if swap is heavily used
* Do NOT use on production nodes

Safe only on:

* Test VMs
* After memory leak analysis
* Systems with almost no active swap usage

---

# ✅ **6. Clear ARP Cache**

```bash
sudo ip -s -s neigh flush all
```

### Safe when:

* Network changes
* Wrong MAC entries
* Failover events

### NOT safe when:

* Clearing under load → temporary traffic drops for some flows

---

# ✅ **7. Clear Route Cache / FIB cache**

Modern Linux doesn’t use route cache heavily, but to flush:

```bash
sudo ip route flush cache
```

Safe to do:

* Yes, mostly safe
* Causes small routing recalculations

---

# ✅ **8. Clear DNS Cache**

If using systemd-resolved:

```bash
sudo systemd-resolve --flush-caches
```

If using dnsmasq:

```bash
sudo systemctl restart dnsmasq
```

Safe: Yes
Effect: Minor delay on first lookup

---

# ✅ **9. Clear conntrack Cache (NAT/Firewall state table)**

⚠️ **HIGH RISK — drops all active connections**

```bash
sudo conntrack -F
```

### Safe only during:

* Firewall misconfigurations
* IP failovers
* NAT stuck flows

NOT safe:

* On Kubernetes nodes
* On production gateways
* Any place with live traffic

---

# 🧹 **10. Combined Script — Clear All Safe Caches**

This clears ONLY safe caches (page cache + dentries + inodes):

```bash
#!/bin/bash
sync
echo 3 > /proc/sys/vm/drop_caches
```

---

# ❌ **What you should NEVER clear on production**

| Cache                | Why                               |
| -------------------- | --------------------------------- |
| Slab                 | Kernel instability, slow syscalls |
| Swap cache (swapoff) | Can hang system                   |
| Full VFS drop        | Will degrade performance          |
| Conntrack            | Drops live network connections    |
| Page cache           | Large performance drop            |

---

# 🧩 Summary table- Clear cache

| Cache Type    | How to Clear                     | Safe?       |
| ------------- | -------------------------------- | ----------- |
| Page Cache    | `echo 1 > drop_caches`           | ⚠️ Medium   |
| Dentry/Inode  | `echo 2 > drop_caches`           | ⚠️ Medium   |
| All VFS Cache | `echo 3 > drop_caches`           | ❌ No        |
| Slab Cache    | `*/shrink`                       | ❌ No        |
| Swap Cache    | `swapoff -a`                     | ❌ No        |
| ARP Cache     | `ip neigh flush`                 | ✔ Usually   |
| Route Cache   | `ip route flush cache`           | ✔ Yes       |
| DNS Cache     | `systemd-resolve --flush-caches` | ✔ Yes       |
| Conntrack     | `conntrack -F`                   | ❌ Dangerous |


---

# 🧩 **Quick Summary Table — Commands**

| Cache Type       | Command                           |                  |
| ---------------- | --------------------------------- | ---------------- |
| Page Cache       | `free -h`, `/proc/meminfo`        |                  |
| Buffer Cache     | `/proc/meminfo`                   |                  |
| Slab Cache       | `slabtop`, `/proc/slabinfo`       |                  |
| Dentry Cache     | `grep dentry /proc/slabinfo`      |                  |
| Inode Cache      | `grep inode /proc/slabinfo`       |                  |
| Swap Cache       | `/proc/meminfo                    | grep SwapCached` |
| Read-Ahead Cache | `blockdev --getra`                |                  |
| Writeback Cache  | `grep Dirty /proc/meminfo`        |                  |
| VFS Cache        | `/proc/sys/fs/dentry-state`       |                  |
| CPU Cache        | `lscpu`, `/sys/devices/.../cache` |                  |
| ARP Cache        | `ip neigh`                        |                  |
| Route Cache      | `ip route show cache`             |                  |
| Conntrack Cache  | `conntrack -L`                    |                  |
| DNS Cache        | `systemd-resolve --statistics`    |                  |

---
