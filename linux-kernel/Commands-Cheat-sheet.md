**Commands-Cheat-sheet** 


Most edit actions require `root`.

# Quick index

* View static CPU info
* Monitor CPU / process in real time
* Per-process inspection
* Change scheduling/priority/affinity
* CPU frequency / governor controls
* cgroups / systemd resource controls
* Kernel scheduler & sysctl tuning
* Limits and resource caps
* CPU profiling & tracing
* Container / VM CPU controls
* Useful `/proc` and `/sys` locations

---

# View static CPU info

* `lscpu` — basic CPU topology (sockets, cores, threads).
  Example: `lscpu`
* `nproc` — number of processing units.
  Example: `nproc --all`
* `cat /proc/cpuinfo` — per-CPU details (model, flags).
  Example: `cat /proc/cpuinfo | grep -E 'model name|cpu MHz|processor'`
* `dmidecode -t processor` — BIOS/CPU vendor info (root).

---

# Monitor CPU & processes (real-time)

* `top` — realtime processes + CPU % (interactive).
  Example: `top` (press `1` to show per-CPU)
* `htop` — improved top (interactive; use `F6` sort, `F3` filter).
  Example: `htop`
* `glances` — multi-metric overview (if installed).
* `mpstat` (sysstat) — per-CPU usage.
  Example: `mpstat -P ALL 1`
* `vmstat` — CPU, memory, IO summary.
  Example: `vmstat 1`
* `iostat` (sysstat) — CPU and disk I/O.
  Example: `iostat -c 1`
* `sar` (sysstat) — historical/system activity (requires data collection).
  Example: `sar -u 1 5`
* `pidstat` (sysstat) — per-PID CPU stats over time.
  Example: `pidstat -u 1`
* `watch` — run a command periodically.
  Example: `watch -n 1 'ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 15'`

---

# Per-process inspection

* `ps` — snapshot of processes.
  Example: `ps -eo pid,ppid,uid,cmd,%cpu,%mem --sort=-%cpu | head`
* `top -p <pid>` — monitor specific PID.
* `cat /proc/<pid>/status` — process status & limits.
  Example: `cat /proc/1234/status`
* `cat /proc/<pid>/sched` or `/proc/<pid>/schedstat` — scheduler info for PID.
* `pmap <pid>` — memory map.
* `strace -p <pid>` — syscall tracing (not CPU usage but useful for hang analysis).
* `perf top -p <pid>` — CPU hotspots for a process (requires perf).

---

# Scheduling & priority (change)

* `nice` — start a process with a niceness value.
  Example: `nice -n 10 mycommand`
* `renice` — change niceness of running PID.
  Example: `renice -n 5 -p 1234`
* `chrt` — set/get real-time scheduling policy / priority.
  Example: `chrt -f 99 myrealtimeprocess` (SCHED_FIFO)
* `schedtool` — more scheduler tweaks (if installed).
* `taskset` — set CPU affinity (bitmask or CPU list).
  Example: `taskset -c 0,2 mycommand` or `taskset -cp 0-3 1234`
* `cpulimit` — limit CPU usage of a process by % (userspace).
  Example: `cpulimit -l 30 -p 1234`

---

# CPU frequency & governor controls

* `cpufreq-info` / `cpufreq-set` (cpufrequtils) — view/set governor / frequencies.
  Example: `cpufreq-info` ; `cpufreq-set -g performance`
* `cpupower` (kernel-tools) — show/set frequency policy.
  Example: `cpupower frequency-info`
* `/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` — read/write governor.
  Example: `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`
  Set: `echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor` (root)

---

# cgroups / systemd resource controls (modern)

* `systemd-cgls` — show control groups tree.
* `systemd-cgtop` — top-like view for cgroups.
* `systemctl show <service> -p CPUQuota,CPUShares` — inspect service resource settings.
  Example: `systemctl show nginx -p CPUQuota`
* `systemctl set-property <unit> CPUQuota=50%` — set CPU quota for a service (runtime).
  Example: `sudo systemctl set-property my.service CPUQuota=20%`
* `cgcreate`, `cgset`, `cgexec` — classic cgroup v1 tools (if available).
  Example: `cgcreate -g cpu:/mygroup; cgset -r cpu.shares=512 mygroup; cgexec -g cpu:mygroup ./app`
* For cgroup v2, use `systemd-run --slice=` or write cgroup v2 files under `/sys/fs/cgroup/`.

---

# Kernel scheduler & sysctl tuning

* `sysctl` — view/change kernel tunables.
  Example: `sysctl -a | grep sched` ; `sysctl -w kernel.sched_migration_cost_ns=5000000`
* Persistent: edit `/etc/sysctl.conf` or `/etc/sysctl.d/*.conf`.
* Useful scheduler-related knobs (examples): `kernel.sched_autogroup_enabled`, `kernel.sched_migration_cost_ns`, `kernel.sched_latency_ns` — check availability on your kernel.
* Kernel boot params (GRUB) for huge changes: `isolcpus=`, `nohz_full=`, `rcu_nocbs=` — require GRUB edit and reboot.

---

# Limits and per-user/process caps

* `/etc/security/limits.conf` — set `nofile`, `nproc` per user (PAM).
  Example: `username soft nproc 4096`
* `ulimit` (bash builtin) — show/set at shell level.
  Example: `ulimit -n` ; `ulimit -u 4096`
* `prlimit` — show/set resource limits on an existing process.
  Example: `prlimit --pid 1234 --as=1G`

---

# CPU profiling & tracing

* `perf` — profiling (top/record/report).
  Examples: `perf top`, `perf record -p 1234 -o perf.data ; perf report`
* `perf stat` — high-level performance counters.
  Example: `perf stat -e cycles,instructions mycommand`
* `ftrace` / `trace-cmd` / `kernelshark` — kernel tracing (advanced).
* `bcc` tools (e.g., `execsnoop`, `tcptracer`, `offcpureport`) if installed.
* `heatmap` / Flamegraphs — use `perf` + `FlameGraph` scripts for hotspots.

---

# Container / VM CPU controls

* Docker: `docker run --cpus=1.5 --cpuset-cpus=0,1 ...` ; `docker update --cpus=0.5 <container>`
* Kubernetes: use `resources.requests.cpu` and `resources.limits.cpu` in pod spec; `kubectl top pods` to monitor.
* For VMs (KVM/QEMU): vCPU pinning via `virsh vcpupin`, CPU quota via libvirt.

---

# Useful `/proc` and `/sys` paths (view/edit)

* `/proc/stat` — overall CPU counters (user, system, idle...).
  Example: `cat /proc/stat`
* `/proc/<pid>/stat`, `/proc/<pid>/status` — per-pid stats.
* `/proc/sys/kernel/` — many sysctl knobs (read/write via sysctl or echo).
* `/sys/devices/system/cpu/` — CPU online, topology, possible, cpufreq, isolation.
  E.g. `/sys/devices/system/cpu/online`, `/sys/devices/system/cpu/isolated`
* `/sys/fs/cgroup/` (v1) or `/sys/fs/cgroup/unified/` (v2) — cgroup settings.

---

# Commands to edit persistently (where to put changes)

* Sysctl: `echo ... > /proc/sys/...` (immediate) + `/etc/sysctl.d/99-my.conf` (persist).
* CPU governor: write to `/sys/devices/system/cpu/cpu*/cpufreq/...` (immediate). Use a boot script or systemd service to persist across reboots (or cpupower service).
* systemd resource settings: `systemctl set-property` (runtime) and then edit unit drop-in in `/etc/systemd/system/<unit>.service.d/override.conf` for persistence.
* `/etc/security/limits.conf` for `ulimit` persistence per user.
* GRUB kernel params: edit `/etc/default/grub` then `update-grub` (and reboot).

---

# Handy smaller tools & commands

* `uptime` — load averages.
* `dstat` — combined resource stats (if installed).
* `ss` / `netstat` — network sockets (sometimes processes blocked on IO).
* `iotop` — per-process IO usage (may reveal IO-bound processes consuming CPU wait).
* `kill -SIGSTOP/PID` / `kill -SIGCONT` — pause/resume a process for quick testing.

---

# Examples (copy-ready)

* Show top CPU consumers (one-liner):
  `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 15`
* Pin process 1234 to CPU 0 and 1:
  `taskset -cp 0,1 1234`
* Lower priority of process 1234:
  `renice +10 -p 1234`
* Limit systemd service to 20% CPU:
  `sudo systemctl set-property my.service CPUQuota=20%`
* View per-CPU usage every second:
  `mpstat -P ALL 1`
* Check CPU topology:
  `lscpu && cat /proc/cpuinfo | grep 'model name' -m1`

---

# Permissions & cautions

* Most editing requires `root`. Changing scheduler and kernel parameters can destabilize systems — test on dev/staging first.
* For production, prefer systemd/cgroups limits or container orchestration resource limits over ad-hoc `nice` changes.
* Persistent changes should be stored in proper config files (`/etc/sysctl.d/`, systemd drop-ins, grub) — not only via echo to `/proc`.

---
