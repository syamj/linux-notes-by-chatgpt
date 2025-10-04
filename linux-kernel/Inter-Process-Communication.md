**Inter-Process Communication (IPC)** 
---

## 🧠 **What is IPC?**

**Inter-Process Communication (IPC)** refers to the mechanisms that allow **processes to exchange data and coordinate their actions**.

Since each process runs in its **own isolated memory space**, they can’t directly read or write each other’s memory — IPC provides controlled methods to share or transfer information safely.

---

## 🧩 **Why IPC is needed**

1. **Data Exchange** – Processes often need to share information (e.g., client-server communication).
2. **Event Notification** – One process might need to notify another when some event happens (e.g., data ready).
3. **Resource Sharing** – For synchronization when accessing shared resources.
4. **Modular System Design** – Break a monolithic app into smaller communicating processes.

---

## ⚙️ **IPC Mechanisms (Linux)**

Linux provides multiple IPC mechanisms, each with different use cases, speed, and complexity:

| Mechanism          | Communication Type | Direction      | Persistence                   | Kernel/Filesystem involvement |
| ------------------ | ------------------ | -------------- | ----------------------------- | ----------------------------- |
| **Pipes / FIFOs**  | Stream (byte)      | Unidirectional | Temporary / Persistent (FIFO) | Kernel                        |
| **Message Queues** | Message-based      | Bidirectional  | Persistent (until removed)    | Kernel                        |
| **Shared Memory**  | Memory-based       | Bidirectional  | Persistent (until detached)   | Kernel (setup only)           |
| **Semaphores**     | Synchronization    | N/A            | Persistent                    | Kernel                        |
| **Sockets**        | Stream / Datagram  | Bidirectional  | Persistent                    | Kernel (network stack)        |
| **Signals**        | Event Notification | Unidirectional | Temporary                     | Kernel                        |

---

## 🧵 1. **Pipes and FIFOs**

### **Unnamed Pipe**

* Used between **parent and child** processes.
* Created using `pipe()`.
* Data flows **one way** — one writes, another reads.

```c
int fd[2];
pipe(fd);
write(fd[1], "Hello", 5);
read(fd[0], buf, 5);
```

### **Named Pipe (FIFO)**

* Exists in the filesystem (`mkfifo /tmp/myfifo`).
* Any process can open it for communication.
* Example:

  ```bash
  echo "Hi" > /tmp/myfifo
  cat /tmp/myfifo
  ```

---

## 📨 2. **Message Queues**

* Communication through messages with **IDs** and **priorities**.
* Persistent in the kernel until deleted.
* System V or POSIX variants exist.

**System V Example (C):**

```c
key_t key = ftok("progfile", 65);
int msgid = msgget(key, 0666 | IPC_CREAT);
msgsnd(msgid, &msg, sizeof(msg), 0);
msgrcv(msgid, &msg, sizeof(msg), 0, 0);
```

Useful when multiple processes communicate asynchronously.

---

## 💾 3. **Shared Memory**

* Fastest IPC since processes directly access shared memory.
* Set up using:

  * **System V**: `shmget()`, `shmat()`, `shmdt()`
  * **POSIX**: `shm_open()`, `mmap()`

Since multiple processes access the same region, **synchronization** (via semaphores or mutexes) is critical.

---

## ⏱️ 4. **Semaphores**

* Used for **synchronization**, not for transferring data.
* Can control access to shared resources (like database locks).
* System V semaphores use `semget()`, `semop()`, etc.

Example use case:

* One process writes to shared memory.
* Another reads only when a semaphore signals "data ready".

---

## 🌐 5. **Sockets**

* Most versatile IPC mechanism.
* Works for **local** (UNIX domain sockets) and **network** (TCP/UDP) communication.
* Allows bidirectional communication between unrelated processes.

Example:

* Web server process communicates with a client via a socket connection.
* Also used for **microservices** or **client-server** architectures.

```bash
# UNIX domain socket example
server -> /tmp/socketfile <- client
```

---

## 🚨 6. **Signals**

* Lightweight notification mechanism.
* Used to inform a process that an event occurred (e.g., `SIGINT`, `SIGTERM`, or user-defined).
* Example: `kill -SIGUSR1 <pid>`

You can catch signals in code using `signal()` or `sigaction()`.

---

## 🧮 7. **Memory-Mapped Files (mmap)**

* A file is mapped into memory, and processes can access it like a shared memory region.
* Used for high-speed communication and persistence.

```c
int fd = open("file.txt", O_RDWR);
char *data = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```

---

## ⚖️ **Comparison Summary**

| IPC Type      | Speed         | Complexity | Synchronization Needed | Scope            |
| ------------- | ------------- | ---------- | ---------------------- | ---------------- |
| Pipe/FIFO     | Medium        | Simple     | No                     | Local            |
| Message Queue | Medium        | Moderate   | No                     | Local            |
| Shared Memory | Fastest       | Moderate   | Yes                    | Local            |
| Sockets       | Slower        | High       | Optional               | Local or Network |
| Signals       | Very Fast     | Simple     | No                     | Local            |
| Semaphores    | N/A (control) | Moderate   | N/A                    | Local            |

---

## 🔍 **Real-world Use Cases**

| Use Case                                            | IPC Type                   |
| --------------------------------------------------- | -------------------------- |
| Parent-child process communication                  | Pipe                       |
| Logging daemon collecting app logs                  | Message Queue              |
| Shared cache between multiple worker processes      | Shared Memory + Semaphores |
| Web server communicating with backend               | UNIX/TCP Socket            |
| Event-based notification system                     | Signals                    |
| File-backed shared data between unrelated processes | mmap                       |

---

Would you like me to explain **how the Linux kernel internally manages IPC (like message queues, semaphores, and shared memory via the `ipc_namespace` and kernel objects)** next? That’s where it gets really deep — down to kernel data structures and syscalls.
