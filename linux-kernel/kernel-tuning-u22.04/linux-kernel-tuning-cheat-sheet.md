**Ubuntu (22.04+) focused cheat-sheet**


I cite authoritative sources for the most load-bearing defaults (swappiness, `somaxconn`, `netdev_max_backlog`, `tcp_fin_timeout`, TCP buffer defaults). If you want, I can also produce a ready-to-apply `/etc/sysctl.d/99-tuning.conf` snippet containing the recommended profile for any one workload.

---

# Linux Kernel Tuning — Ubuntu 22.04+ (Markdown reference)

> **Notes about “Default value”:** kernel and distribution defaults can vary by kernel version and machine memory; below I use *typical Ubuntu / modern-kernel* defaults (Ubuntu 22.04 / Linux 5.x+). Where the kernel decides a value at boot (e.g., `fs.file-max`, `tcp_mem`), I note that behavior. I cite primary references for the most important defaults. ([Ask Ubuntu][1])

---

## How to read the tables

Columns:

* **Parameter** — sysctl key or proc node.
* **What it does** — short description.
* **Default (typical Ubuntu 22.04+)** — typical out-of-box value (or how it’s determined).
* **Recommended (per workload)** — values to consider for *HAProxy*, *Redis*, *NGINX*, *Docker host*, *K8s node (containerd)*. Pick and test what fits your hardware and traffic.

---

## A. CPU & Scheduling (quick picks)

| Parameter                            |                                                                   What it does | Default (typical)                                                       | Recommended (HAProxy / Redis / NGINX / Docker host / K8s node)                                                                                                     |
| ------------------------------------ | -----------------------------------------------------------------------------: | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `kernel.sched_min_granularity_ns`    | Minimum CFS time slice (ns). Smaller → more responsive, more context switches. | `~3000000` (3ms) on many 5.x kernels.                                   | Keep default; **lower** slightly for ultra low-latency interactive workloads (not usually for servers).                                                            |
| `kernel.sched_wakeup_granularity_ns` |                                   Minimum delta to allow preemption on wakeup. | `~4000000` (4ms)                                                        | Default is fine for all five; only tune for specialized latency-sensitive services.                                                                                |
| `kernel.sched_migration_cost_ns`     |                                Penalty time before migrating task across CPUs. | `~500000` (0.5ms)                                                       | Leave default or **increase** a bit if you see cache-miss due to migrations (large multi-core DBs).                                                                |
| `kernel.numa_balancing`              |                                         Auto NUMA page balancing (1=on,0=off). | `1` on some kernels or configured by distribution; depends on platform. | **HAProxy/NGINX/Docker/K8s:** `1` usually fine. **Redis** (NUMA hosts + pinned cores): consider `0` (disable) if you pin processes and want deterministic latency. |

---

## B. Memory

| Parameter                   |                                                      What it does |               Default (typical Ubuntu) | Recommended (HAProxy / Redis / NGINX / Docker host / K8s node)                                                                                                |
| --------------------------- | ----------------------------------------------------------------: | -------------------------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vm.swappiness`             |       How aggressively kernel swaps (0..100). Lower = avoid swap. |                `60`. ([Ask Ubuntu][1]) | **HAProxy:** `10–20` • **Redis:** `0–10` (prefer no swap) • **NGINX:** `10–20` • **Docker host:** `10` • **K8s node:** `10`                                   |
| `vm.overcommit_memory`      | Memory overcommit policy (0 heuristic, 1 always allow, 2 strict). |                       `0` (heuristic). | **HAProxy/NGINX/Docker/K8s:** `1` if you want containers to allocate freely; **Redis:** `1` (if you manage OOM carefully) or `2` for extremely strict safety. |
| `vm.dirty_background_ratio` |                       % RAM at which background writeback starts. |    `~10` (common). ([Red Hat Docs][2]) | **HAProxy/NGINX:** `5–10` • **Redis:** `2–5` (if persistence enabled tune carefully) • **Docker/K8s hosts:** `5–10`                                           |
| `vm.dirty_ratio`            |                % RAM at which process blocking writeback happens. |             `~20`. ([Red Hat Docs][2]) | **HAProxy/NGINX:** `10–15` • **Redis:** `10` (avoid big blocking) • **Docker/K8s:** `10–15`                                                                   |
| `vm.vfs_cache_pressure`     |     Reclaim aggressiveness for dentry/inode caches (100 default). |                                  `100` | **HAProxy/NGINX/Redis/Docker/K8s:** `50` (keep FS caches longer on servers with good RAM)                                                                     |
| `vm.min_free_kbytes`        |         Minimum free memory reserved (KB) to avoid OOM thrashing. | Kernel-calculated (depends on memory). | Increase moderately on large-memory DB/Redis hosts — e.g., set to `65536` or higher depending on memory to avoid stalls.                                      |

> Tip: On systems with large RAM and heavy I/O, prefer `dirty_background_ratio`/`dirty_ratio` tuned to avoid long latency spikes.

---

## C. Filesystem / FD limits

| Parameter                            |                                                                 What it does | Default (typical)                                             | Recommended (HAProxy / Redis / NGINX / Docker host / K8s node)                      |
| ------------------------------------ | ---------------------------------------------------------------------------: | ------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `fs.file-max`                        | System-wide max open file handles. Kernel-calculated at boot (based on RAM). | **Dynamic / kernel-calculated** (varies). ([Server Fault][3]) | **HAProxy/NGINX:** `2097152` • **Redis:** `65536`+ • **Docker host/K8s:** `2097152` |
| `/etc/security/limits.conf` `nofile` |                                            Per-user process open file limit. | Default (user) often `1024` (varies).                         | Set service users to `65536` or higher for proxies/containers.                      |
| `fs.inotify.max_user_watches`        |                                                   Max file watches per user. | `8192` or distro default (varies).                            | **K8s node/CI:** `524288` • Others: keep default unless you run watchers.           |

---

## D. Networking — the most critical area for web/proxy

> I show **typical default** and **recommended** tuned values (start with these, then benchmark).

| Parameter                                 |                                                                        What it does | Default (typical Ubuntu / kernel)                                                                                                                                   | Recommended (HAProxy / Redis / NGINX / Docker host / K8s node)                                                                        |
| ----------------------------------------- | ----------------------------------------------------------------------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `net.core.somaxconn`                      |            Max backlog for `listen()` (accept queue limit exposed to applications). | Kernel-dependent: *older kernels*: `128`; *modern kernels (>=5.4)* often use `4096` for new network namespaces — behaviour is nuanced. ([arthurchiao.github.io][4]) | **HAProxy/NGINX:** `65535` • **Redis:** `4096–65535` • **Docker/K8s:** `4096–65535`                                                   |
| `net.ipv4.tcp_max_syn_backlog`            |                                    Max remembered half-open (SYN_RECV) connections. | `1024` typical; may scale with memory. ([Alibaba Cloud][5])                                                                                                         | **HAProxy/NGINX:** `8192–65536` • **Docker/K8s:** `8192`                                                                              |
| `net.core.netdev_max_backlog`             | Max packets kernel will queue before userland can process; protects against bursts. | `1000` typical. ([IBM][6])                                                                                                                                          | **HAProxy/NGINX:** `250000` (or `262144`) • **Redis:** `250000` • **Docker/K8s:** `250000`                                            |
| `net.ipv4.tcp_fin_timeout`                |                      Time (seconds) sockets stay in FIN-WAIT-2 / TIME_WAIT cleanup. | `60` seconds. ([man7.org][7])                                                                                                                                       | **HAProxy/NGINX:** `15–30` (careful) • **Redis:** keep `60` (stateful) • **Docker/K8s:** `30`                                         |
| `net.ipv4.tcp_tw_reuse`                   |                      Allow reuse of TIME_WAIT sockets for new outgoing connections. | `0` (off). ([IBM][8])                                                                                                                                               | **HAProxy/NGINX:** `1` (if many short connections) • **Redis:** `0` (safer) • **Docker/K8s:** `1` for high-churn outbound connections |
| `net.ipv4.ip_local_port_range`            |                                      Ephemeral port range for outbound connections. | `32768 60999` (common), can vary by distro. ([docs.kernel.org][9])                                                                                                  | **HAProxy/NGINX:** `1024 65000` • **Redis:** default ok • **Docker/K8s:** `1024 65000`                                                |
| `net.core.rmem_max` / `net.core.wmem_max` |                                             Max socket receive/send buffer (bytes). | `~212992` or kernel/package default; depends on distribution & kernel (varies).                                                                                     | **HAProxy/NGINX/Redis/K8s:** increase to `16777216` (16MB) or `33554432` depending on NIC and BW.                                     |
| `net.ipv4.tcp_rmem`                       |                                         TCP receive buffer (min default max) tuple. | Typical: `4096 87380 <kernel-calculated-max>`; man pages show default initial as `87380`. ([man7.org][7])                                                           | **HAProxy/NGINX:** `4096 87380 16777216` • **Redis:** `4096 87380 16777216` • **Docker/K8s:** `4096 87380 16777216`                   |
| `net.ipv4.tcp_wmem`                       |                                            TCP send buffer tuple (min default max). | Typical: `4096 65536 <kernel-calculated-max>`. ([IBM][8])                                                                                                           | **HAProxy/NGINX:** `4096 65536 16777216` • **Redis:** `4096 65536 16777216` • **Docker/K8s:** `4096 65536 16777216`                   |

**Notes & rationale:**

* `somaxconn` alone doesn’t make apps accept more — apps must call `listen( backlog )` with suitable backlog and sockets configured (e.g., `listen(..., SOMAXCONN)`), and some containers may have separate netns behaviours. ([arthurchiao.github.io][4])
* `netdev_max_backlog` default 1000 is insufficient for 1Gbps+ bursty servers — bump for heavy NICs. ([IBM][6])

---

## E. Security & Limits

| Parameter                   |                              What it does | Default                             | Recommended                                                                                       |
| --------------------------- | ----------------------------------------: | ----------------------------------- | ------------------------------------------------------------------------------------------------- |
| `kernel.pid_max`            |  Maximum PID value the kernel can assign. | `4194304` (typical modern default). | `65536`–`4194304`. For many container hosts keep modern default; tune only if you need >65k PIDs. |
| `fs.suid_dumpable`          |    Allow core dumps from setuid programs. | `0` (disabled for security).        | Keep `0` on production.                                                                           |
| `kernel.kptr_restrict`      |    Hide kernel pointer values in `/proc`. | `2` recommended in production.      | Keep `2`.                                                                                         |
| `kernel.randomize_va_space` | ASLR (memory layout randomization) level. | `2` (full)                          | Keep `2`.                                                                                         |

---

## F. Container & Kubernetes-specific

| Parameter                            |                           What it does | Default                                        | Recommended (Docker host / K8s node)                                         |
| ------------------------------------ | -------------------------------------: | ---------------------------------------------- | ---------------------------------------------------------------------------- |
| `vm.overcommit_memory`               |                     Overcommit policy. | `0`                                            | **Docker/K8s:** `1` (commonly used) — but combine with OOM policies/cgroups. |
| `net.ipv4.ip_forward`                |        Enable IP forwarding (routing). | `0` in some cases; distro may enable.          | **Docker/K8s:** `1` (required for routing & kube-proxy iptables rules).      |
| `net.bridge.bridge-nf-call-iptables` | Pass bridged packets through iptables. | `0` or `1` depending on distro/kernel modules. | **K8s:** `1` (so kube-proxy / network policies apply to bridge traffic).     |
| `user.max_user_namespaces`           |             Number of user namespaces. | May be `14000` or kernel default (varies).     | **Docker/K8s:** raise if you need many unprivileged userns.                  |


---

## H. How to apply & verify

**Apply immediately**

```bash
sudo sysctl -w net.core.somaxconn=65535
```

**Persist**
Create `/etc/sysctl.d/99-tuning.conf` with desired keys, then:

```bash
sudo sysctl --system
```

**Verify**

```bash
sysctl -a | grep somaxconn
cat /proc/sys/net/core/netdev_max_backlog
```

**Backup current settings**

```bash
sysctl -a > ~/sysctl-backup-$(date +%F).txt
```

---

## I. Sources & further reading (selected)

* Ask Ubuntu — default `vm.swappiness = 60`. ([Ask Ubuntu][1])
* Blog / analysis on container `somaxconn` and kernel-version differences (why default may be `128`, `4096` etc.). ([arthurchiao.github.io][4])
* IBM docs: `netdev_max_backlog` default explanation (default ~1000). ([IBM][6])
* `tcp(7)` man page — defaults and formulas for `tcp_rmem` / `tcp_wmem`. ([man7.org][7])
* Linux kernel / RHEL tuning docs on `dirty_ratio` / `dirty_background_ratio` typical values/explanations. ([Red Hat Docs][2])

---

## J. Final tips & testing plan

1. **Baseline metrics**: collect `vmstat`, `iostat`, `ss -s`, `dstat`, `perf` before changes.
2. **Change one category at a time** (network vs memory vs fs).
3. **Load test** under realistic traffic (wrk/httperf/iperf/fio) and compare.
4. **Watch for regressions** (increased latency, packet drops, OOMs).
5. **Roll back** quickly by restoring your `sysctl-backup`.
6. For **Kubernetes**, prefer node-level tuning + DaemonSet for node agents; beware of per-pod netns `somaxconn` behaviour (containers may inherit different constants).

---


[1]: https://askubuntu.com/questions/103915/how-do-i-configure-swappiness?utm_source=chatgpt.com "How do I configure swappiness?"
[2]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/configuring-an-operating-system-to-optimize-memory-access_monitoring-and-managing-system-status-and-performance?utm_source=chatgpt.com "Chapter 35. Configuring an operating system to optimize ..."
[3]: https://serverfault.com/questions/716578/default-value-of-proc-sys-fs-file-max?utm_source=chatgpt.com "Default value of /proc/sys/fs/file-max - linux kernel"
[4]: https://arthurchiao.github.io/blog/the-mysterious-container-somaxconn/?utm_source=chatgpt.com "The Mysterious Container net.core.somaxconn (2022)"
[5]: https://www.alibabacloud.com/blog/599203?utm_source=chatgpt.com "TCP SYN Queue and Accept Queue Overflow Explained"
[6]: https://www.ibm.com/docs/en/linux-on-systems?topic=tuning-network-stack-settings&utm_source=chatgpt.com "Network stack settings"
[7]: https://man7.org/linux/man-pages/man7/tcp.7.html?utm_source=chatgpt.com "tcp(7) - Linux manual page"
[8]: https://www.ibm.com/docs/en/linux-on-systems?topic=tuning-tcpip-ipv4-settings&utm_source=chatgpt.com "TCPIP IPv4 settings"
[9]: https://docs.kernel.org/networking/ip-sysctl.html?utm_source=chatgpt.com "IP Sysctl"
