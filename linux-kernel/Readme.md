Linux Kernel
---

### 🧠 **Definition**

The **Linux kernel** is the **core part of the Linux operating system**.
It’s a **program that controls everything in your computer** — how hardware and software interact.

In simple terms:

> The Linux kernel is like the “brain” of the operating system — it sits between your applications and your hardware.

---

### ⚙️ **What the Kernel Does**

The kernel handles 4 key responsibilities:

1. **Process Management**

   * Decides which processes (programs) run, when, and for how long.
   * Handles multitasking and process scheduling.
   * Example: Running Chrome, Spotify, and Terminal at the same time — the kernel ensures each gets CPU time.

2. **Memory Management**

   * Allocates and frees RAM for programs.
   * Uses techniques like **paging** and **virtual memory**.
   * Example: If RAM is full, it swaps data to disk (swap space).

3. **Device Management**

   * Talks to hardware like keyboards, disks, and network cards using **device drivers**.
   * Example: When you plug in a USB device, the kernel detects and mounts it.

4. **File System Management**

   * Controls how data is stored and retrieved.
   * Supports many filesystems: `ext4`, `xfs`, `btrfs`, etc.

---

### 🧩 **Kernel vs Operating System**

| Component                 | Description                                              |
| ------------------------- | -------------------------------------------------------- |
| **Kernel**                | The core managing CPU, memory, and devices.              |
| **Operating System (OS)** | Kernel **+ utilities + user space programs**.            |
| **Example**               | Ubuntu = Linux kernel + GNU tools + desktop environment. |

So when people say **“Linux OS”**, they mean the **Linux kernel + userland tools** like `bash`, `systemd`, etc.

---

### 🧱 **Types of Kernels**

Linux uses a **monolithic kernel** (all major components run in kernel space).
But it can dynamically load modules (drivers) — so it’s **modular**.

| Kernel Type | Example           | Description                                       |
| ----------- | ----------------- | ------------------------------------------------- |
| Monolithic  | Linux             | All core functions in one large kernel.           |
| Microkernel | Minix, QNX        | Minimal core, with drivers running in user space. |
| Hybrid      | Windows NT, macOS | Mix of both.                                      |

---

### 🔢 **Kernel Version Example**

You can check your current kernel version with:

```bash
uname -r
```

Example output:

```
6.8.0-35-generic
```

* `6.8.0` → major/minor version
* `35` → patch level
* `generic` → build type

---

### 🧠 Summary

| Concept             | Description                                     |
| ------------------- | ----------------------------------------------- |
| **What**            | The core software that connects apps ↔ hardware |
| **Responsible for** | CPU, memory, devices, files                     |
| **Lives in**        | `/boot/vmlinuz-*`                               |
| **Loaded by**       | Bootloader (GRUB) during system startup         |
| **Developed by**    | Open-source community (led by Linus Torvalds)   |

---

