
---

# 🧵 **Linux Caches (Complete List + What They Store)**

Linux uses several types of caches to optimize performance.
Below is a **complete SRE-grade list**.

---

# 1️⃣ **Page Cache**

* **Caches file contents**
* Used for read/write IO to filesystems
* Backed by RAM
* Evicted via LRU

📌 **Most important cache in Linux**
📌 Biggest part of “free memory” usage

---

# 2️⃣ **Buffer Cache**

* Caches *block device metadata* (file system metadata)
* Example:

  * superblocks
  * inodes
  * block group descriptors

In `/proc/meminfo`, called **Buffers**.

---

# 3️⃣ **Dentry (Directory Entry) Cache**

* Caches directory path lookups → improves `open()`, `stat()`, `ls` performance
* Maps:
  `path -> inode`

List:

```bash
cat /proc/slabinfo | grep dentry
```

---

# 4️⃣ **Inode Cache**

* Caches inodes of recently accessed files
* Improves file open/read/write operations
* Prevents repeated disk metadata fetch

List:

```bash
cat /proc/slabinfo | grep inode
```

---

# 5️⃣ **Slab Cache**

**Slab allocator** is a memory management system.

It caches:

* kernel objects
* inodes
* dentries
* network structures
* VFS structures
* file descriptors
* task structs
* socket structures

See slab usage:

```bash
slabtop
```

---

# 6️⃣ **Swap Cache**

* Memory pages that are swapped out go through **swap cache** first.
* Prevents repeated re-read from swap device.

Purpose:

* Avoid writing pages to swap if they might be referenced again soon
* Avoid reading swapped pages back twice

---

# 7️⃣ **VFS (Virtual File System) Cache**

Umbrella term for dentries + inode caches + file structure caches.

Used to speed up:

* `open()`, `stat()`, `read()`, `write()` calls
* path traversal

---

# 8️⃣ **Network-related Caches**

Linux networking subsystem maintains multiple caches:

### 🔹 Route Cache (older kernels)

* Cached routes for packets
* Mostly replaced by fib_trie now

### 🔹 ARP Cache

* IP ↔ MAC mapping
* Shown with:

```bash
ip neigh
```

### 🔹 TCP Socket Cache

* TCP structs (tcp_sock)
* LISTEN socket info
* Reused socket buffers

### 🔹 Netfilter Connection Tracking Cache (conntrack)

* Tracks flows across NAT/firewall
* Used in Kubernetes nodes heavily

View:

```bash
conntrack -L
```

---

# 9️⃣ **CPU Caches (Hardware-level)**

Not OS-controlled but relevant:

* L1 Instruction & Data cache
* L2 cache
* L3 shared cache

Very fast → measured in nanoseconds
Essential for system performance but not a Linux-specific mechanism.

---

# 🔟 **Read-Ahead Cache**

* Used by block layer
* Predictively loads next blocks into RAM
* Great for sequential I/O (e.g., big file reads, DB scans)

Check:

```bash
blockdev --getra /dev/sda
```

---

# 1️⃣1️⃣ **Writeback Cache**

* Temporary holding area for dirty pages
* Kernel flushes using `flush-<device>` threads

Tune thresholds:

```bash
vm.dirty_background_ratio
vm.dirty_ratio
```

---

# 1️⃣2️⃣ **Filesystem-specific Caches**

Examples:

### 🔹 ext4

* extent cache
* journaling cache

### 🔹 XFS

* XFS inode cache
* metadata cache
* log buffer cache

### 🔹 btrfs

* extent map cache
* checksum cache

---

# 1️⃣3️⃣ **DNS Cache**

Linux does **NOT** have a native DNS cache in the kernel.
But systemd-resolved / nscd provide userspace DNS caching.

Check:

```bash
systemd-resolve --statistics
```

---

# 1️⃣4️⃣ **Application-level caches**

Not part of kernel, but still important:

* Redis
* Memcached
* JVM heap cache
* Database buffer pools (Postgres shared_buffers, MySQL buffer pool)

---

# 🧩 Summary Table

| Cache              | What It Stores        | Purpose                    |
| ------------------ | --------------------- | -------------------------- |
| Page Cache         | File data pages       | Speed up file reads/writes |
| Buffer Cache       | Block device metadata | Faster FS ops              |
| Dentry Cache       | Path lookup info      | Speed up `open()`          |
| Inode Cache        | File metadata         | Faster file ops            |
| Slab Cache         | Kernel objects        | Avoid frequent alloc/free  |
| Swap Cache         | Swapped pages         | Avoid double I/O           |
| VFS Cache          | Dentries + inodes     | Improve file operations    |
| Network Caches     | ARP, TCP, conntrack   | Faster packet routing      |
| Read-Ahead Cache   | Next disk blocks      | Faster sequential reads    |
| Writeback Cache    | Dirty pages           | Improve write performance  |
| FS-specific Caches | FS metadata           | FS performance             |
| DNS Cache          | Hostname → IP         | Reduce DNS lookups         |

---

