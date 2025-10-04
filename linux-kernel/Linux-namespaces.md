**Linux namespaces** 

---

## 🧠 What Is a Namespace?

A **namespace** in Linux is a kernel-level mechanism that **isolates system resources** for a group of processes.
Each namespace provides a **separate view** of a particular system resource (like process IDs, network interfaces, or mount points).

So —

> Two processes in different namespaces can think they are the only processes in the system, even though they share the same kernel.

---

## 🧩 Types of Linux Namespaces

As of today, Linux supports **seven major namespaces** (and one extension):

| Namespace             | Flag              | Isolates / Virtualizes                  | Example                                                  |
| --------------------- | ----------------- | --------------------------------------- | -------------------------------------------------------- |
| **Mount**             | `CLONE_NEWNS`     | File system mount points                | Each container can have its own root filesystem.         |
| **PID**               | `CLONE_NEWPID`    | Process IDs                             | Processes inside see their own PID 1.                    |
| **Network**           | `CLONE_NEWNET`    | Network interfaces, routing tables, IPs | Containers get their own virtual NIC, IP stack.          |
| **IPC**               | `CLONE_NEWIPC`    | System V IPC, POSIX message queues      | Shared memory and semaphores are isolated.               |
| **UTS**               | `CLONE_NEWUTS`    | Hostname and domain name                | Containers can have their own hostname.                  |
| **User**              | `CLONE_NEWUSER`   | User and group IDs                      | Allows root inside container to map to non-root outside. |
| **Cgroup**            | `CLONE_NEWCGROUP` | Cgroup root hierarchy                   | Isolates resource control groups per container.          |
| *(Optional)* **Time** | `CLONE_NEWTIME`   | System and monotonic clocks             | Used to fake or shift time inside a container.           |

---

## 🧱 How Namespaces Work Together

When you start a container:

* Docker (or `runc`) creates a new set of namespaces for it:

  * New mount namespace (isolated filesystem)
  * New PID namespace (own process tree)
  * New network namespace (virtual NIC)
  * New UTS namespace (container hostname)
  * New user namespace (map UID/GID)
* Then it **`chroot`s or pivot_root** into the container’s root filesystem.
* Finally, it starts a process inside that namespace.

Each namespace isolates one aspect — together they provide a **container illusion**.

---

## 🔧 Real-World Use Cases

### 1. **Containers (Docker, Podman, Kubernetes)**

The most common use case.

* Each container gets isolated filesystem, PID, network stack, etc.
* User namespace enables non-root containers (security).
* Example: Docker uses all of them under the hood.

---

### 2. **Sandboxing / Security Isolation**

Used by:

* Chrome/Firefox sandboxed renderers
* systemd services with `PrivateNetwork=yes`, `ProtectSystem=full`
* Snap, Flatpak — to confine desktop apps.

---

### 3. **Multi-network Testing / Simulations**

You can create multiple **network namespaces** for testing networking setups without VMs:

```bash
ip netns add ns1
ip netns add ns2
ip link add veth1 type veth peer name veth2
ip link set veth1 netns ns1
ip link set veth2 netns ns2
```

Now `ns1` and `ns2` have independent network stacks — great for experimenting with routing, iptables, or service meshes.

---

### 4. **PID Namespace for Process Containment**

You can run a process so it doesn’t see any other system process:

```bash
unshare --pid --fork bash
```

Now inside, if you run `ps -ef`, you’ll see only your shell — perfect for lightweight isolation.

---

### 5. **Custom Hostname (UTS Namespace)**

```bash
unshare --uts bash
hostname container1
```

Outside, your system hostname remains unchanged.

---

### 6. **User Namespace for Privilege Separation**

Allows mapping root inside → unprivileged user outside:

```bash
unshare --user --map-root-user bash
```

Inside the shell, you’re root — but you can’t harm the host.

That’s how **rootless containers** work.

---

### 7. **Mount Namespace for Chroot on Steroids**

`chroot` alone only changes the root directory; processes can still affect global mounts.
`unshare --mount` fully isolates mount operations.
Used in system installers and LiveCD environments.

---

## ⚙️ Tools to Explore Namespaces

| Tool                  | Purpose                                              |
| --------------------- | ---------------------------------------------------- |
| `unshare`             | Create a new namespace and run a command there       |
| `nsenter`             | Enter an existing namespace (e.g., into a container) |
| `ip netns`            | Manage network namespaces                            |
| `lsns`                | List all current namespaces                          |
| `cat /proc/<pid>/ns/` | Inspect namespaces of a process                      |

---

## 🧩 Example: A Minimal Container by Hand

```bash
mkdir rootfs
debootstrap --variant=minbase focal rootfs  # or use busybox

unshare --mount --uts --ipc --net --pid --fork --user --map-root-user \
  chroot rootfs /bin/bash
```

You now have a shell with its own PID tree, network stack, and hostname — effectively a container without Docker.

---

## 💡 Summary

| Namespace | Isolates              | Used In                     |
| --------- | --------------------- | --------------------------- |
| Mount     | File system           | Chroot, Containers          |
| PID       | Processes             | Containers, Sandboxes       |
| Network   | Network stack         | Containers, Network testing |
| IPC       | Shared memory, queues | Containers                  |
| UTS       | Hostname              | Containers                  |
| User      | UID/GID               | Rootless containers         |
| Cgroup    | Resource accounting   | Docker, Systemd             |
| Time      | Clock                 | Time virtualization         |

---
