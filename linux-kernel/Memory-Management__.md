**Linux Kernel Memory Management**
---

## **1. Memory Overview in Linux**

Linux manages memory as **virtual memory**, not directly physical memory. Every process sees its own **virtual address space**. The kernel translates this virtual memory to physical memory using the **MMU (Memory Management Unit)** and **page tables**.

Key concepts:

* **Virtual Memory (VM):** Each process thinks it has a large contiguous memory space.
* **Physical Memory:** The actual RAM.
* **Page:** The smallest unit of memory managed by Linux, usually **4 KB** (default x86). Can be larger for hugepages (2 MB, 1 GB).
* **Page Frame:** The physical page in RAM.
* **Page Table:** Maps virtual addresses to physical addresses.
* **Memory Zones:** The kernel categorizes memory for allocation purposes:

  * `ZONE_DMA` – For DMA-capable devices (e.g., <16 MB).
  * `ZONE_NORMAL` – Regular kernel allocations.
  * `ZONE_HIGHMEM` – High memory (on 32-bit systems, above 4 GB).

---

## **2. Address Spaces**

Linux separates memory into **user space** and **kernel space**:

* **User Space:** Virtual addresses accessible to a process (typically 0x00000000 – 0x7FFFFFFF on 32-bit).
* **Kernel Space:** Virtual addresses reserved for the kernel (0xC0000000 – 0xFFFFFFFF on 32-bit). On 64-bit, kernel space is much larger.

The kernel uses **privileged instructions** and **page table entries** with protection bits to prevent user processes from accessing kernel memory.

---

## **3. Page Management**

Linux manages memory in pages. This is the backbone of VM:

1. **Page Allocation:**

   * Kernel maintains free page lists per zone.
   * Allocation functions:

     * `alloc_pages()`: Allocate a contiguous set of pages.
     * `kmalloc()`: Kernel equivalent of `malloc()`. Can allocate smaller memory chunks.

2. **Page Flags:**
   Each page frame has flags like:

   * `PG_locked` – Page is locked in memory.
   * `PG_dirty` – Page has been modified.
   * `PG_active` – Recently used page.
   * `PG_referenced` – Recently accessed.

3. **Buddy System:**
   Kernel uses **buddy allocator** for physical memory:

   * Memory is divided into blocks of size `2^n` pages.
   * When a block is allocated, if it’s too large, it’s split into two buddies.
   * When freed, buddies are merged back.

---

## **4. Virtual Memory Areas (VMA)**

Every process keeps track of memory mappings using **VMAs**:

* A **VMA** is a contiguous region of virtual memory with the same properties (permissions, backing file, etc.).
* `/proc/<pid>/maps` shows all VMAs for a process.
* Operations like `mmap`, `brk`, or shared memory creation create or modify VMAs.
* Each VMA points to a **struct mm_struct** (memory descriptor for the process).

---

## **5. Page Tables**

Linux uses **multi-level page tables**:

* 32-bit x86: 2-level (Page Directory → Page Table → Page Frame)
* 64-bit x86_64: 4-level (PGD → PUD → PMD → PTE → Page Frame)
* PTEs contain:

  * Physical page frame number
  * Permission bits (RWX, user/kernel)
  * Present bit
  * Dirty/accessed bits

**TLB (Translation Lookaside Buffer):** CPU caches recent page table entries to avoid frequent memory walks.

---

## **6. Memory Allocation APIs in Kernel**

* **kmalloc(size, flags)**: Allocates physically contiguous memory, suitable for small allocations.
* **vmalloc(size)**: Allocates virtually contiguous memory, not necessarily physically contiguous.
* **alloc_pages(gfp_mask, order)**: Low-level page allocation.

**GFP Flags:** Control allocation behavior:

* `GFP_KERNEL` – Can sleep, normal kernel allocation.
* `GFP_ATOMIC` – Cannot sleep, used in interrupt context.
* `GFP_HIGHMEM` – Allocate from high memory.

---

## **7. Caching and Slab Allocator**

Linux uses **slab allocators** for small objects:

* **SLAB, SLUB, SLOB** – Kernel object allocators.
* Advantages:

  * Reduces fragmentation
  * Fast allocation of small objects
  * Maintains caches for frequently used kernel objects (e.g., `task_struct`)

Example: `kmem_cache_create()` creates a cache for a specific object type.

---

## **8. Page Replacement & Swapping**

When physical memory runs low:

1. Kernel uses **page replacement algorithms** (like **LRU**) to free memory.
2. Pages are moved to **swap** space if needed.
3. `kswapd` kernel thread is responsible for freeing pages in the background.

**Memory pressure metrics:**

* `free`, `used`, `cached`, `buffers` in `/proc/meminfo`.
* `vmstat` shows active/inactive pages, swap in/out.

---

## **9. Memory Mapped Files**

* Files can be mapped into memory using `mmap()`.
* Useful for:

  * Lazy loading large files
  * Shared memory between processes
* Kernel keeps track of **struct address_space** and **struct page** for file-backed pages.

---

## **10. NUMA & High-Performance Systems**

* On NUMA (Non-Uniform Memory Access) systems, memory is grouped by nodes.
* Kernel tries to allocate memory **close to the CPU** that requests it.
* NUMA-aware allocators: `numa_alloc_onnode()`.

---

## **11. Kernel Data Structures**

Key structs in memory management:

* `struct page` – Represents a physical page.
* `struct vm_area_struct` – Represents a VMA.
* `struct mm_struct` – Represents the memory layout of a process.
* `struct address_space` – File-backed mapping.

---

## **12. Common Memory-Related Issues**

* **Too many open files / file descriptors** – Each file descriptor can consume a small amount of kernel memory (`struct file`, `struct inode`).
* **OOM Killer** – Triggered when memory is exhausted.
* **Memory leaks in kernel modules** – Can occur if `kmalloc` allocations are not freed.
* **Fragmentation** – Both virtual and physical memory fragmentation can cause allocation failures.

---

If you want, I can **draw a full diagram of Linux memory management**, showing how **process virtual memory → VMA → page tables → physical memory → slab allocator → swap** all tie together. This helps visualize the entire flow.

Do you want me to do that next?
