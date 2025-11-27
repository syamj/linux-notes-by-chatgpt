
---

# ✅ **What is the OS Page Cache?**

The **page cache** is a **kernel memory region** used to cache file data and filesystem-backed pages to speed up future file reads/writes.

### 🔹 **Purpose**

* Reduce disk I/O (super slow compared to RAM)
* Speed up file access
* Coalesce writes before sending to disk
* Keep frequently accessed file pages in RAM

### 🔹 **How it works (simple)**

* When a process reads a file, the kernel checks the **page cache** first.

  * If data exists → **cache hit** → served from RAM
  * If not → **cache miss** → data read from disk and stored into page cache
* When a process writes:

  * Data is written **into the page cache first** (dirty pages)
  * Flushed to disk **asynchronously** via the kernel writeback mechanism (`pdflush` / `bdflush` / `flush-* threads`)

### 🔹 **What goes into page cache?**

* Regular files
* Executable binaries
* Shared libraries
* mmap’ed files
* Directory entries (partially)

You can monitor it:

```bash
cat /proc/meminfo | grep -E "Cached|Buffers"
```
---