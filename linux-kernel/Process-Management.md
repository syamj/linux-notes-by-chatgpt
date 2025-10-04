## Process Management by Kernel
 
 Let’s walk through the entire process lifecycle in Linux — from birth → execution → death — including special cases like errors, signals, and OOM kills.

 
Example:


Let’s trace exactly what happens, step by step, when you run a command like `ls` or when a service is started by `systemd`.

We’ll go from **user → shell → kernel → process creation**.

---

##  Big Picture

At a high level:

```
You (user)
 ↓
Shell or systemd (already running process)
 ↓
fork()   ← inside user-space
 ↓
Kernel (creates child process)
 ↓
execve() ← inside user-space, loads new program
 ↓
New program runs (ls, nginx, etc.)
```

So — **it’s not the kernel directly deciding to create processes.**
It’s always an **existing process** (like your shell or `systemd`) that calls `fork()` or `clone()` — and the **kernel performs the actual duplication** internally.

---

## 🧩 Case 1: You Run `ls` in a Terminal

### Step-by-Step:

1. **You type:**

   ```bash
   ls -l
   ```

2. **Shell reads your input**

   * Your shell (e.g., `bash`, `zsh`, `fish`) is a user-space process that’s *always running* on your terminal.
   * It parses your command line.

3. **Shell calls `fork()`**

   * `bash` uses the `fork()` system call to create a child process.
   * Now there are two almost identical processes:

     * **Parent** → the original shell
     * **Child** → a clone created via `fork()`

   At this point, both are running the same code (`bash`), but the child will soon become something else.

4. **Child calls `execve()`**

   * The child process replaces itself with the new program (`/bin/ls`) using:

     ```c
     execve("/bin/ls", ["ls", "-l"], envp);
     ```
   * This system call asks the **kernel** to:

     * Load the binary (`ls`) from disk.
     * Set up memory, stack, environment.
     * Start execution at the ELF entry point.

5. **Kernel sets up the process**

   * Creates a new `task_struct`.
   * Maps ELF sections into memory.
   * Initializes file descriptors (stdin, stdout, stderr from the parent).
   * Marks it `TASK_RUNNING`.

6. **Scheduler runs `ls`**

   * The process now runs, outputs to your terminal.

7. **When `ls` finishes**

   * It calls `exit()`.
   * The kernel changes its state → `ZOMBIE`.
   * Sends `SIGCHLD` to parent (your shell).

8. **Parent shell calls `waitpid()`**

   * Collects the exit code.
   * Kernel cleans up the child process completely.

✅ Then your prompt returns.

---

## 🧩 Case 2: Starting a Service via `systemd`

Now, let’s look at a **non-interactive** example — like when you run:

```bash
sudo systemctl start nginx
```

### Step-by-Step:

1. **You talk to `systemctl`**

   * `systemctl` is just a CLI client.
   * It uses **D-Bus** (an IPC mechanism) to send a message to the **`systemd` daemon**.

2. **`systemd` receives the request**

   * `systemd` runs as PID 1 — the very first user-space process.
   * It’s responsible for managing all services.

3. **`systemd` decides to start the service**

   * It looks at the service unit file (e.g., `/lib/systemd/system/nginx.service`).
   * Prepares the environment, user, working directory, etc.

4. **`systemd` calls `fork()`**

   * It forks a child process using `fork()` (or more commonly, `clone()` — a newer syscall that underlies `fork()`).
   * This child becomes the future `nginx` master process.

5. **Child calls `execve()`**

   * The child replaces itself with `/usr/sbin/nginx` using `execve()`.

6. **Kernel creates and schedules the process**

   * The kernel allocates PID, sets up memory, file descriptors, etc.

7. **`systemd` tracks it**

   * The parent (systemd) keeps the PID to monitor the service.
   * If it crashes or exits, `systemd` knows and can restart it based on the unit config (`Restart=on-failure` etc.).

---

## 🧩 Who Actually Performs `fork()`?

| Scenario                                        | Who Calls `fork()`    | What Happens Next                                 |
| ----------------------------------------------- | --------------------- | ------------------------------------------------- |
| You run `ls`                                    | `bash` or `zsh`       | Calls `fork()` then `execve("ls")`                |
| You run a background job                        | `bash`                | Calls `fork()` but doesn’t wait immediately       |
| A service starts                                | `systemd`             | Calls `fork()` or `clone()`, then `execve()`      |
| You SSH into a system                           | `sshd`                | Parent `sshd` forks a new child per login         |
| A web server spawns workers                     | `nginx` or `apache`   | Master forks multiple worker processes            |
| A container runtime (Docker) starts a container | `containerd` / `runc` | Calls `clone()` to isolate namespaces and cgroups |

So the **pattern is always:**

> A user-space parent process → calls `fork()` or `clone()` → kernel duplicates → child `execve()` new program.

---

## 🔧 Under the Hood in the Kernel

When user-space calls `fork()`, it’s actually invoking a **syscall**:

```c
clone(flags, child_stack, pid_t *ptid, pid_t *ctid, struct pt_regs *regs)
```

Internally, the kernel executes:

```c
do_fork() → copy_process() → alloc_pid() → wake_up_new_task()
```

Each new process gets:

* Its own `task_struct`
* New PID
* Shared or copied resources (depending on flags)

---

These system calls can be traces uding `strace` command.

 `strace` is one of the **most powerful tools in a Linux engineer’s toolbox**, especially for debugging **what’s actually happening between user-space and the kernel**.

Let’s go step by step — you’ll learn **what `strace` is**, **what it can show**, and **how to use it effectively** with examples.

---

## 🧠 What Is `strace`?

`strace` stands for **System Call Trace**.

It intercepts and records all **system calls** made by a process — and the **signals** received by that process.

Think of it like:

> “A microscope that shows exactly how a process talks to the Linux kernel.”

---

## ⚙️ Why Use `strace`?

You can use `strace` for **debugging**, **troubleshooting**, and **learning**.

| Use Case                               | Example                                                        |
| -------------------------------------- | -------------------------------------------------------------- |
| 🧩 **Debug program startup issues**    | Why won’t my binary start? Missing shared library? Wrong path? |
| 🗂️ **Find missing files/configs**     | Which files does this program try to read/write?               |
| 🔒 **Permission or path errors**       | "Permission denied" — which file caused it?                    |
| 🧠 **Understand system behavior**      | What syscalls does a command make when it runs?                |
| 🐛 **Trace crashes or hangs**          | Where is the process getting stuck?                            |
| 🚀 **Performance debugging**           | See delays in system calls or I/O waits                        |
| 🧰 **Reverse engineering / education** | Learn how `fork()`, `execve()`, etc. work in reality           |

---

## 🔍 How `strace` Works

When you run:

```bash
strace ls
```

`strace`:

1. Attaches itself to the process using the `ptrace()` system call.
2. Intercepts every **system call** (like `open()`, `read()`, `write()`, `execve()`, etc.).
3. Logs each call, its arguments, and return values.

It’s like watching a live conversation between your program and the Linux kernel.

---

## 🧪 Example 1: Trace a Simple Command

```bash
strace ls
```

Sample output (shortened):

```
execve("/bin/ls", ["ls"], 0x7ffec3b2b840 /* 50 vars */) = 0
brk(NULL)                               = 0x563b47ee6000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
...
write(1, "Desktop  Documents  Downloads\n", 31) = 31
exit_group(0)                           = ?
```

### 🔹 What you can see:

* `execve()` — starting the binary `/bin/ls`
* `openat()` — opening shared libraries and files
* `write(1, ...)` — writing to stdout (your terminal)
* `exit_group(0)` — process exits successfully

So you literally see:
**fork → execve → open → read/write → exit** — the full process lifecycle in system call terms.

---

## 🧪 Example 2: Trace What a Shell Does When Running `ls`

Let’s see how `bash` calls `fork()` and `execve()`.

```bash
strace -f bash -c "ls"
```

The `-f` flag tells `strace` to **follow child processes** (important because `bash` forks a child for `ls`).

Sample snippet:

```
[pid  1020] clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4d9ef25a10) = 1021
[pid  1021] execve("/bin/ls", ["ls"], 0x7ffd9a0a32a0 /* 47 vars */) = 0
[pid  1021] write(1, "Desktop\nDocuments\nDownloads\n", 29) = 29
[pid  1021] exit_group(0) = ?
```

🔹 You can see:

* **PID 1020 (bash)** → calls `clone()` (underlying syscall for `fork()`)
* **PID 1021 (child)** → calls `execve("/bin/ls")`
* Output printed → process exits.

This is the real evidence of the lifecycle we discussed earlier!

---

## 🧪 Example 3: Trace a Running Process

You can attach to an existing process:

```bash
sudo strace -p <PID>
```

Example:

```bash
sudo strace -p 1234
```

This lets you see what syscalls a running service is making (e.g., `nginx`, `mysqld`).

Press `Ctrl+C` to detach.

---

## 🧪 Example 4: Log to a File (for Long Traces)

```bash
strace -o trace.log -f nginx -t -T
```

Options explained:

* `-o trace.log` → write output to a file
* `-f` → follow child processes
* `-t` → timestamp each line
* `-T` → show time spent in each syscall

Now you can open `trace.log` and analyze it with `less`, `grep`, or even `awk`.

---

## 🧩 Common Options Summary

| Option               | Description                                                 |
| -------------------- | ----------------------------------------------------------- |
| `-f`                 | Follow child processes (important for `bash`, `systemd`)    |
| `-p <pid>`           | Attach to existing process                                  |
| `-o <file>`          | Write output to file                                        |
| `-e trace=<syscall>` | Filter specific syscalls (e.g., `-e trace=open,read,write`) |
| `-e open,read,write` | Only show file-related syscalls                             |
| `-T`                 | Show time spent in each syscall                             |
| `-tt`                | Add precise timestamps                                      |
| `-c`                 | Show syscall summary with counts and timing                 |

Example:

```bash
strace -c ls
```

Output:

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 70.00    0.000154           5        31           read
 20.00    0.000044           4        11           openat
 10.00    0.000022           4         5           close
------ ----------- ----------- --------- --------- ----------------
100.00    0.000220                    47           total
```

This gives you a **performance profile** of system calls.

---

## 🧰 Example: Debug “File Not Found” Error

Say your app fails with:

```
Error: configuration file not found
```

Run:

```bash
strace -e open,openat ./myapp
```

You’ll see something like:

```
openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
```

Now you know *exactly* what path it was trying to open — and that it didn’t exist.

---

## 🧩 Key Concept: strace Shows the Kernel Boundary

User-space → Kernel-space transitions are **only possible through system calls**.
`strace` shows every one of them.

It’s how you can *see inside the black box* of process behavior without touching its source code.

---

## 🧠 TL;DR Summary

| Concept             | Meaning                             |
| ------------------- | ----------------------------------- |
| **strace**          | Traces system calls & signals       |
| **fork → execve**   | Always visible when commands start  |
| **Debug uses**      | Missing files, permissions, hangs   |
| **Attach to PID**   | See what a running process is doing |
| **Filter syscalls** | Focus on specific operations        |
| **Timing mode**     | Analyze performance bottlenecks     |

---



## Scheduling (Who Gets CPU Time)

The kernel can’t run all processes at once (unless you have infinite CPUs 😉).
So it uses **scheduling algorithms** to decide who runs next.

Linux has several scheduling classes:

- CFS (Completely Fair Scheduler) → used for normal tasks.
- It ensures fair CPU distribution by tracking “virtual runtime.”

- Real-Time Schedulers (SCHED_FIFO, SCHED_RR) → for time-critical tasks.
- Example: Audio playback or robotics.

- Idle & Batch Scheduling → for background or low-priority tasks.

Each process gets a PID and a priority (nice value) which the scheduler uses.

You can check them using:
```
ps -eo pid,comm,pri,ni,stat
```



## Deep dive on CPU Scheduling!

**CPU scheduling** is the kernel’s mechanism to decide:

> “Which runnable process gets the CPU, and for how long?”

Linux is a **preemptive multitasking OS**, meaning multiple processes can run “concurrently” by rapidly switching CPU context between them.

---

## 🏗️ Key Concepts

1. **Process State**
   Linux tracks each process with a `task_struct` that contains a **state field**:

   * `TASK_RUNNING` → ready to run or currently running
   * `TASK_INTERRUPTIBLE` → waiting for an event (e.g., I/O)
   * `TASK_UNINTERRUPTIBLE` → waiting for I/O, cannot be interrupted
   * `TASK_STOPPED` → stopped via signal (e.g., SIGSTOP)
   * `TASK_ZOMBIE` → finished, waiting for parent to reap

2. **Runqueue**

   * Each CPU has a **runqueue** (list of runnable processes).
   * Scheduler picks the next process from this queue.

3. **Context Switch**

   * Switching CPU from one process to another:

     1. Save current CPU registers & state
     2. Load registers & state of new process
     3. Update `task_struct` pointers
   * Very fast but non-zero cost.

---

## 🧩 Linux Scheduling Classes

Linux supports **different scheduling classes** for different workloads:

| Class                               | Purpose                       | Typical Use Case                   |
| ----------------------------------- | ----------------------------- | ---------------------------------- |
| **CFS (Completely Fair Scheduler)** | General-purpose, time-sharing | User processes, GUI, CLI           |
| **Real-time (RT) FIFO / RR)**       | Strict priority, predictable  | Audio, video, robotics             |
| **Idle**                            | Low-priority background tasks | Background cleanup, kernel threads |

* **CFS** is default for normal processes.
* **Real-time classes** are used when a process sets `SCHED_FIFO` or `SCHED_RR`.

---

## 🏃‍♂️ CFS (Completely Fair Scheduler)

CFS is the **main Linux scheduler** since 2.6.23.

**Key ideas:**

1. **Virtual Runtime (`vruntime`)**

   * Each process tracks how much CPU time it has used.
   * Scheduler tries to give each process **fair share of CPU**, i.e., smaller `vruntime` → next to run.

2. **Red-Black Tree**

   * CFS stores runnable processes in a **red-black tree**, sorted by `vruntime`.
   * Leftmost node → process that has received the least CPU → selected next.

3. **Time Slice**

   * Instead of fixed time slices, CFS uses **proportional fair time slices**.
   * CPU share = `(weight of process / total weight)` × CPU period.

4. **Preemption**

   * If a new process arrives with smaller `vruntime`, it can **preempt the current running process** immediately.

---

### ⚡ Example

Suppose 3 processes P1, P2, P3 have different priorities:

| Process | Weight (priority) | CPU usage |
| ------- | ----------------- | --------- |
| P1      | 1024              | 10ms      |
| P2      | 512               | 5ms       |
| P3      | 2048              | 20ms      |

CFS ensures **each process gets CPU proportional to its weight** — not strictly round-robin.

---

## 🏎️ Real-Time Scheduling

1. **SCHED_FIFO**

   * First-in-first-out.
   * Highest priority RT process runs until it **blocks, exits, or is preempted by higher priority**.
   * No time slicing.

2. **SCHED_RR**

   * Round-robin.
   * Same priority RT processes take **fixed time slices** in order.

* RT processes always **preempt normal CFS processes**, so they are guaranteed CPU.

---

## 🏗️ Scheduler Data Structures

1. **`task_struct`**

   * Stores all info about a process: state, priority, runqueue pointer, vruntime, CPU affinity.
2. **Runqueue (`rq`)**

   * Per CPU, contains:

     * Number of running tasks
     * Total weight
     * Red-black tree of runnable tasks
3. **CFS Red-Black Tree**

   * Stores only **TASK_RUNNING** processes.

---

## 🔄 Scheduling Flow (High Level)

1. **Interrupt or Timer Tick** occurs → kernel calls scheduler.
2. Scheduler selects **next process to run** based on class:

   * RT first, then CFS.
3. **Check preemption**:

   * If current process has exceeded its fair share, or a higher priority process arrived → context switch.
4. **Context Switch**:

   * Save current process state
   * Load next process state
   * Update CPU registers and task pointers
5. CPU runs next process until next tick or preemption.

---

## 🧠 Priority & Niceness

* **Nice values** (-20 to +19) affect CFS weights:

  * Lower nice → higher weight → more CPU
  * Higher nice → lower weight → less CPU
* Real-time priorities (0-99) override CFS completely.

---

## 🔧 Commands to Observe Scheduling

1. `top` / `htop`

   * Shows running processes, CPU %, nice value.

2. `ps -eo pid,comm,ni,pri,rtprio,stat`

   * `ni` = nice, `pri` = priority, `rtprio` = real-time priority.

3. `chrt`

   * View/set real-time scheduling policies.

4. `/proc/<pid>/sched`

   * Shows detailed per-process scheduling info:

     ```
     pid 1234
     se.sum_exec_runtime : 123.45
     se.vruntime        : 12.34
     ```

---

## 🏁 Special Scenarios

1. **I/O-bound vs CPU-bound**

   * I/O-bound processes sleep frequently → get **more frequent CPU** when they wake (CFS favors fairness).
2. **Forked processes**

   * Inherit scheduling class & priority from parent.
3. **CPU affinity**

   * Process can be pinned to a specific CPU.
4. **Load balancing**

   * In multi-core systems, kernel moves tasks across CPUs for fairness.

---

## 🔑 Summary

* Linux is **preemptive multitasking**, multiple processes share CPU.
* **CFS** is default for normal processes → ensures fair CPU usage based on weights.
* **Real-time scheduling** (FIFO/RR) preempts normal processes for predictability.
* Scheduler works with **task_struct, runqueues, vruntime, red-black trees**.
* Priority, nice, CPU affinity, and load balancing influence scheduling.
* All of this happens **transparently to user-space**, but you can monitor via `/proc`, `top`, `htop`, `strace`.

---


Now we’ll connect **process states, scheduling, and kernel events** to show a **full lifecycle from creation to termination**, including sleep, wake-ups, and preemption.

---

## 1️⃣ Process States Recap

Every Linux process lives in one of several states (`task_struct->state`):

| State                  | Meaning                                                        |
| ---------------------- | -------------------------------------------------------------- |
| `TASK_RUNNING`         | Process is ready or currently running on CPU                   |
| `TASK_INTERRUPTIBLE`   | Waiting for an event (like I/O), can be interrupted by signals |
| `TASK_UNINTERRUPTIBLE` | Waiting for I/O, cannot be interrupted (usually disk waits)    |
| `TASK_STOPPED`         | Stopped by signal (e.g., SIGSTOP)                              |
| `TASK_ZOMBIE`          | Process finished execution but parent hasn’t reaped it yet     |

---

## 2️⃣ Process Lifecycle + Scheduling

### Step 1: **Process Creation**

* `fork()` / `clone()` creates a new `task_struct`.
* Child inherits:

  * FD table
  * Scheduling policy (CFS or RT)
  * Nice/priority
* Initially: `state = TASK_RUNNING`
* Added to CPU **runqueue** of assigned CPU.

---

### Step 2: **Waiting for CPU**

* If CPU is busy:

  * Process sits in **runqueue**.
  * Scheduler selects the next process based on **priority / vruntime**.
* When selected → process is dispatched → `TASK_RUNNING` → executes on CPU.

---

### Step 3: **Performing Work**

* Process executes instructions:

  * **CPU-bound:** keeps consuming time slice.
  * **I/O-bound:** calls `read()`, `write()`, or waits for network/socket.
* **I/O calls** → kernel may put process to sleep:

  * Example: `read()` on empty pipe → `TASK_INTERRUPTIBLE`.

---

### Step 4: **Sleeping / Waiting**

* Process sleeps while waiting for **event** (I/O completion, timer, signal).
* Scheduler removes it from CPU, picks next runnable process.
* Two main types:

  1. `TASK_INTERRUPTIBLE` → wakes up on signal.
  2. `TASK_UNINTERRUPTIBLE` → wakes up only when I/O completes.

---

### Step 5: **Wake-up**

* When event occurs:

  * Disk finishes read
  * Socket receives data
  * Timer expires
* Kernel calls `wake_up()` → changes state back to `TASK_RUNNING`.
* Process is **re-added to runqueue**, waits for CPU.

---

### Step 6: **Preemption**

* Kernel preempts running process if:

  * A higher priority process becomes runnable (RT or CFS with lower `vruntime`)
  * Time slice of current process expires
* `TASK_RUNNING` → preempted → saved to runqueue → next process scheduled.

---

### Step 7: **Handling Signals**

* If process is in `TASK_INTERRUPTIBLE`:

  * Signal (e.g., `SIGTERM`, `SIGKILL`) → process wakes up.
* Kernel sets pending signal → scheduler may run signal handler.
* If `SIGKILL` → process terminated immediately.

---

### Step 8: **Exiting Process**

* When process calls `exit()`:

  * State becomes `TASK_DEAD` (or `TASK_ZOMBIE` temporarily)
  * FDs are closed
  * Memory and kernel resources released
  * Kernel notifies parent (`SIGCHLD`)
* Parent calls `wait()` / `waitpid()` → reaps the child → removes zombie.

---

### Step 9: **Special Case: OOM Kill**

* If system runs out of memory:

  * Kernel OOM killer selects a process (usually largest memory consumer)
  * Sends `SIGKILL` → process dies immediately
  * All FDs and resources cleaned up
* Scheduler removes process from runqueue.

---

## 3️⃣ Timeline of Events

| Event                  | State                                | Scheduler Role                                  |
| ---------------------- | ------------------------------------ | ----------------------------------------------- |
| `fork()`               | TASK_RUNNING                         | Adds to runqueue                                |
| Waiting for I/O        | TASK_INTERRUPTIBLE / UNINTERRUPTIBLE | Removed from CPU, other processes scheduled     |
| Event occurs           | TASK_RUNNING                         | Added to runqueue, CPU scheduled when available |
| Time slice expires     | TASK_RUNNING                         | Preempted, added back to runqueue               |
| Signal arrives         | TASK_INTERRUPTIBLE                   | Wakes up, signal handler executed               |
| Process calls `exit()` | TASK_ZOMBIE → TASK_DEAD              | Removed from runqueue, resources cleaned        |

---

## 4️⃣ How This Looks in Practice

Imagine running `cat /etc/passwd`:

1. `bash` forks → new process `cat` → `TASK_RUNNING`
2. Scheduler picks `cat` → executes
3. `cat` calls `read(fd, buf, size)` → FD on file → may sleep if file not cached → `TASK_INTERRUPTIBLE`
4. Disk returns data → `wake_up()` → `TASK_RUNNING` → scheduled → continues reading
5. Writes output via `write()` → FD 1 (stdout)
6. EOF → `exit()` → `TASK_ZOMBIE` → parent `bash` reaps → process fully terminated

**During this time, scheduler may preempt `cat`** if other processes are runnable.

---

## 5️⃣ Key Kernel Functions Involved

| Function                | Purpose                                     |
| ----------------------- | ------------------------------------------- |
| `schedule()`            | Main scheduler function, picks next task    |
| `wake_up()`             | Wakes up a sleeping process                 |
| `try_to_wake_up()`      | Checks conditions and sets process state    |
| `context_switch()`      | Saves/restores CPU registers and task state |
| `exit_notify()`         | Notifies parent after exit                  |
| `release_task_struct()` | Cleans up resources                         |

---

### 🔑 Takeaways

* A process **lives in different states**, not just “running.”
* Scheduler moves processes **on and off CPU** based on priority, time slice, and events.
* **I/O, signals, timers, and preemption** are the main triggers for state changes.
* **Termination** can be normal (`exit()`) or forced (OOM, SIGKILL).
* All of this is handled in kernel — **user sees only that process runs, sleeps, and exits**.

---


## 🧮  Process Communication and Signals

During its life, a process interacts with others and with the kernel.

### 🔹 Signals (Software Interrupts)

Signals let the kernel or other processes **control** a process’s behavior.

| Signal    | Meaning              | Default Action   |
| --------- | -------------------- | ---------------- |
| `SIGTERM` | Terminate gracefully | Exit             |
| `SIGKILL` | Kill immediately     | Exit             |
| `SIGSTOP` | Pause process        | Stop             |
| `SIGCONT` | Resume process       | Continue         |
| `SIGSEGV` | Segmentation fault   | Core dump        |
| `SIGINT`  | Ctrl+C from terminal | Exit             |
| `SIGCHLD` | Child exited         | Ignore or handle |

The process can **catch** or **ignore** most signals, except `SIGKILL` and `SIGSTOP`.

Example:

```c
signal(SIGTERM, cleanup_handler);
```

When a process receives a signal:

* It’s added to its pending signal mask.
* At the next scheduling point, kernel delivers it.
* The process runs its signal handler or performs the default action.

---

##  Process Errors and Crashes

### 🔹 When the process does something invalid:

Examples:

* Segmentation fault (`SIGSEGV`)
* Bus error (`SIGBUS`)
* Illegal instruction (`SIGILL`)
* Floating point exception (`SIGFPE`)

Then:

1. Kernel sends the corresponding signal (e.g., `SIGSEGV`).
2. Process can handle it via `signal()` or `sigaction()`.
3. If not handled → kernel kills process and may create a **core dump**.

Core dumps are stored at `/proc/sys/kernel/core_pattern`.

Example:

```
Segmentation fault (core dumped)
```

---

## Process Termination

A process can end in several ways:

### 🔹 Normal Termination

* Calls `exit()` or returns from `main()`.
* The kernel:

  1. Releases memory.
  2. Closes file descriptors.
  3. Changes state → **ZOMBIE**.
  4. Notifies parent via `SIGCHLD`.

The parent must call `wait()` or `waitpid()` to clean it up — this removes it from the process table.

If parent doesn’t reap it → it remains a **zombie**.

---

### 🔹 Forced Termination

* Another process (like the shell) or kernel sends:

  ```bash
  kill -9 <pid>    # SIGKILL
  ```
* The kernel **immediately deallocates** all resources.
* No signal handler or cleanup code runs.
* Cannot be caught or ignored.

---

### 🔹 Orphaned Processes

If parent dies before child:

* The child is **adopted by `init`/`systemd` (PID 1)**.
* PID 1 reaps it when it exits → ensures no permanent zombies.

---

##  Out-of-Memory (OOM) Killer

When the system runs out of memory:

* The kernel’s **OOM killer** triggers.
* It scans all processes and kills one or more to free memory.

### 🔹 Selection Criteria

OOM killer calculates a score based on:

* Memory usage (`RSS`)
* Priority (`oom_score_adj`)
* Process type (system-critical processes are less likely to die)

You can view scores:

```bash
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_adj
```

If selected:

* Kernel sends `SIGKILL` to that process.
* Logs appear in `dmesg`:

  ```
  Out of memory: Kill process 1234 (chrome) score 980 or sacrifice child
  Killed process 1234 (chrome) total-vm:1G, anon-rss:512M
  ```

---

## Other Termination Scenarios

| Scenario                               | Description                                                |
| -------------------------------------- | ---------------------------------------------------------- |
| **Kernel panic / crash**               | If kernel itself fails, all processes die.                 |
| **User logout / terminal close**       | Sends `SIGHUP` to child processes.                         |
| **Cgroup or container limit exceeded** | Kernel kills process inside cgroup (e.g., in Docker).      |
| **Timeout or watchdog**                | Supervisor process kills it after a timeout.               |
| **System shutdown**                    | `systemd` sends `SIGTERM` then `SIGKILL` to all processes. |

---

##  Process Death and Cleanup

When a process finally dies:

1. It enters **ZOMBIE** state (only `task_struct` remains).
2. Parent receives `SIGCHLD`.
3. Parent calls `wait()`:

   * Reads child’s exit code.
   * Kernel removes `task_struct`.
4. PID can now be reused by future processes.

---

## 🔍 Lifecycle Summary Diagram (Conceptually)

```
        ┌──────────┐
        │ fork()   │
        └────┬─────┘
             │
        ┌────▼─────┐
        │ exec()   │
        └────┬─────┘
             │
        ┌────▼────┐
        │ Running │◄──────────┐
        └────┬────┘           │
             │                 │
        ┌────▼────┐        ┌───▼──────┐
        │ Sleeping│        │ Stopped  │
        └────┬────┘        └───┬──────┘
             │                 │
        ┌────▼────┐        ┌───▼──────┐
        │ Zombie  │        │ SIGKILL  │
        └────┬────┘        └───┬──────┘
             │                 │
        ┌────▼─────────────────▼───┐
        │        Dead / Reaped     │
        └──────────────────────────┘
```

---

## 🧠 TL;DR

| Stage        | What Happens                                   |
| ------------ | ---------------------------------------------- |
| **Creation** | `fork()` → new process, `exec()` → new program |
| **Running**  | Scheduled by kernel (CFS)                      |
| **Waiting**  | I/O, sleep, or signals                         |
| **Error**    | Kernel sends signal (e.g., SIGSEGV)            |
| **Killed**   | Via signal, OOM, or system event               |
| **Zombie**   | Waiting for parent cleanup                     |
| **Reaped**   | Completely removed from process table          |

---
