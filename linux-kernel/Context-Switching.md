
---

##  What is Context Switching?

**Context Switching** is the process by which the CPU switches from executing one process (or thread) to another.

Since a single CPU core can execute only **one instruction stream at a time**, the illusion of multitasking is created by rapidly switching between processes.
Each process gets a small time slice of the CPU — known as a **CPU quantum** — and when that time ends (or it’s blocked for I/O), the kernel **saves** the current process’s state and **restores** another’s.

---

## ⚙️ What Does “Context” Mean?

The **context** of a process/thread is everything the CPU needs to resume its execution *exactly where it left off.*

That includes:

| Category            | What It Contains                                                                                                              | Purpose                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **CPU registers**   | General-purpose registers (RAX, RBX, RCX, RDX, etc.), instruction pointer (RIP), stack pointer (RSP), flags register (RFLAGS) | Restores CPU computation state                |
| **Memory mapping**  | Page tables, virtual memory mappings                                                                                          | Ensures correct address space                 |
| **Kernel stack**    | Each process/thread has its own kernel stack                                                                                  | For kernel mode execution                     |
| **Scheduling info** | Priority, state (READY/RUNNING/SLEEPING), CPU affinity                                                                        | Helps the scheduler pick the next task        |
| **FPU/SIMD state**  | Floating point, vector registers (XMM/YMM/ZMM)                                                                                | Saved/restored only when needed (lazy saving) |

All of this together forms the **process context**.

---

## 🧩 Types of Context Switches

1. **Process context switch**

   * Happens between two processes.
   * Requires switching **address space**, **page tables**, and **kernel stack**.
   * Expensive (flushes TLB).

2. **Thread context switch (within same process)**

   * Threads share address space.
   * Only CPU registers, stack pointers, and some kernel state are switched.
   * Cheaper than process switch.

3. **Interrupt context switch**

   * CPU switches from a running process to handle an **interrupt** (hardware or software).
   * Kernel saves minimal state, executes ISR (Interrupt Service Routine), then resumes.
   * Doesn’t involve scheduler (unless ISR wakes up a process).

---

## 🔁 When Does Context Switching Occur?

1. **Voluntary (Cooperative):**

   * Process calls a blocking syscall like `read()`, `sleep()`, `wait()`, or yields CPU (`sched_yield()`).
   * Kernel marks it as waiting and switches to another runnable process.

2. **Involuntary (Preemptive):**

   * Process exceeds its CPU quantum.
   * Higher-priority process becomes ready.
   * Hardware timer interrupt (e.g., from **scheduler tick**) triggers scheduler.
   * Kernel preempts current process and switches context.

---

## 🧮 The Steps in a Context Switch (Kernel’s Perspective)

Let’s take an example in Linux:

1. **Timer interrupt fires (e.g., every 1ms):**

   * CPU traps into kernel mode.
   * Current process’s registers are saved to its **kernel stack**.

2. **Kernel runs the scheduler (`__schedule()`):**

   * Chooses the next runnable process (based on scheduling policy).

3. **Kernel saves the old process context:**

   * Updates `task_struct` of old process (`prev`).
   * Saves registers, FPU state, etc.

4. **Kernel loads new process context:**

   * Updates `cr3` register (page table base) → switch address space.
   * Loads new kernel stack pointer.
   * Restores CPU registers and instruction pointer from `next` process’s context.

5. **Returns to user mode:**

   * Uses `iretq` instruction → resumes new process as if it was never interrupted.

---

## 🧬 Where This Happens in Linux Code

Linux kernel (simplified flow):

```
schedule()
  └── context_switch()
        ├── switch_mm()        // switch virtual memory space
        ├── switch_to(prev, next) // low-level CPU register switch
        └── finish_task_switch()
```

`switch_to()` is **architecture-dependent** (assembly code), e.g., in x86 it saves and restores registers and the stack pointer.

---

## 🧩 Data Structures Involved

* **`task_struct`** – represents each process/thread.

  * Holds CPU state, priority, scheduling info, kernel stack pointer, etc.
* **`mm_struct`** – holds the memory mappings of a process.
* **`thread_struct`** – architecture-specific CPU context.

  * e.g., general registers, flags, instruction pointer.

---

## 🧱 Cost of Context Switching

Context switching is **not free** — it causes:

1. **CPU overhead**

   * Saving/restoring registers and kernel stack.

2. **TLB flush**

   * Switching address space invalidates TLB entries (unless same address space).

3. **Cache pollution**

   * New process brings new data/code → old cache lines evicted.

4. **Kernel time**

   * Scheduler decisions, system calls.

🔹 Typical cost: ~1–5 µs for a thread switch on modern CPUs
🔹 Much higher for process switches (depends on TLB/cache behavior)

---

## 🔍 How to Observe Context Switching in Linux

### 1. `vmstat`

```bash
vmstat 1
```

Look at `cs` column → number of context switches per second.

### 2. `pidstat -w`

Shows voluntary (`nvcsw`) and involuntary (`nivcsw`) switches per process:

```bash
pidstat -w 1
```

### 3. `/proc/<pid>/status`

```
cat /proc/1234/status | grep ctxt
```

Shows:

* `voluntary_ctxt_switches`
* `nonvoluntary_ctxt_switches`

### 4. `perf sched record/report`

To profile scheduling activity and visualize switches.

---

## ⚡ Optimizing Context Switching

1. **Use fewer threads** → thread pools instead of 1 thread per request.
2. **Avoid blocking I/O** → use async I/O or event-driven models.
3. **Pin tasks to cores (CPU affinity)** → avoid unnecessary migrations.
4. **Tune scheduler (CFS parameters)** for your workload.
5. **Batch work** → reduce wakeups.

---
