# OS Jackfruit – Lightweight Container Runtime in C

---

## Team Members

| SRN | Name |
|-----|------|
| PES2UG24AM047 | Chinmay N S |
| PES2UG24AM036 | Ayush Gaikwad |

---

## Project Overview

OS Jackfruit is a lightweight Linux container runtime built from scratch in C. The idea was to understand how tools like Docker work under the hood by actually implementing the core pieces ourselves — process isolation, logging, IPC, and memory enforcement.

Each "container" in this project is a Linux process created using `clone()` with namespace flags (`CLONE_NEWPID`, `CLONE_NEWUTS`, `CLONE_NEWNS`). This gives the process its own isolated PID tree, hostname, and filesystem view. The container is then `chroot`'d into a minimal root filesystem so it cannot see the host's directories.

Alongside the user-space runtime, we built a Linux Kernel Module (`monitor.c`) that tracks container PIDs, polls their RSS (Resident Set Size) every second, logs a warning when memory crosses a soft threshold, and sends `SIGKILL` when the hard limit is exceeded.

The entire project runs on Ubuntu 24.04 inside VirtualBox.

---

## Features Implemented

- **Process Isolation** — Containers run with separate PID, UTS, and mount namespaces using `clone()` with appropriate flags. Each container gets its own hostname and filesystem root via `chroot`.
- **Container Lifecycle Management** — Full CLI support: `start`, `run`, `stop`, `ps`, and `logs`. The supervisor process stays alive and tracks metadata (PID, state, start time, memory limits) for every container.
- **IPC via UNIX Domain Socket** — The CLI client and the supervisor communicate over a UNIX domain socket at `/tmp/mini_runtime.sock`. The CLI sends a command, the supervisor processes it and sends back a response — two completely separate processes, clean bidirectional IPC.
- **Bounded-Buffer Logging** — Container stdout/stderr is captured through pipes into a shared ring buffer. A producer thread reads from the pipe and pushes log entries; a consumer thread pops entries and writes them to per-container log files. Synchronised with mutex and condition variables to avoid race conditions and dropped data.
- **Kernel Memory Monitoring** — A kernel module exposes `/dev/container_monitor`. The supervisor registers container PIDs via `ioctl`. A kernel timer fires every second, checks RSS for each tracked process, and acts on limit violations.
- **Soft Limit Warning** — When a container's RSS first crosses the soft threshold, the kernel module logs a `SOFT LIMIT` warning via `printk`. This is a one-time advisory warning per container — the process keeps running.
- **Hard Limit Enforcement** — When RSS exceeds the hard limit, the kernel module sends `SIGKILL` to the container process and logs a `HARD LIMIT` event. The supervisor detects the child's death via `SIGCHLD` and marks the container state as `killed`.
- **Scheduler Experiments with `nice` Values** — Containers can be started with a `--nice` flag. We ran two cpu_hog containers simultaneously with different nice values and observed how Linux CFS scheduling affected their CPU allocation and progress rate.

---

## Prerequisites

- Ubuntu 22.04 or 24.04 running inside VirtualBox (WSL will not work — kernel modules cannot be loaded there)
- Secure Boot must be **OFF** in VirtualBox VM settings (Settings → System → Motherboard → uncheck Enable EFI)
- GCC with static linking support
- `make` utility
- Linux kernel headers matching the running kernel
- Root / sudo access

Install dependencies:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) git wget
```

---

## Setup and Build Instructions

All commands below assume you are inside the `boilerplate` directory:

```bash
cd ~/OS-Project/OS-Jackfruit/boilerplate
```

### Step 1 — Build the engine and kernel module

```bash
make all
```

This compiles `engine` (the user-space runtime), `monitor.ko` (the kernel module), and the workload binaries.

### Step 2 — Recompile workload binaries as static

The workload binaries need to be statically linked so they run inside a minimal rootfs without needing shared libraries from the host.

```bash
gcc -static -o memory_hog memory_hog.c
gcc -static -o cpu_hog    cpu_hog.c
gcc -static -o io_pulse   io_pulse.c
```

Verify they are statically linked (output must say `statically linked`):

```bash
file memory_hog cpu_hog io_pulse
```

### Step 3 — Create per-container rootfs copies

Two separate rootfs copies are needed so that two containers can run simultaneously without sharing the same filesystem and causing conflicts.

```bash
cp -a rootfs rootfs-alpha
cp -a rootfs rootfs-beta
```

### Step 4 — Copy workload binaries into rootfs copies

```bash
cp memory_hog cpu_hog io_pulse rootfs-alpha/bin/
cp memory_hog cpu_hog io_pulse rootfs-beta/bin/
```

### Step 5 — Ensure `/proc` directories exist

The container mounts `/proc` inside the chroot so that process-related tools work correctly inside the container. The directory must exist beforehand.

```bash
mkdir -p rootfs-alpha/proc
mkdir -p rootfs-beta/proc
```

### Step 6 — Test that binaries work inside the rootfs

Before running any container, verify manually that a binary executes inside the chroot:

```bash
sudo chroot rootfs-alpha /bin/memory_hog
```

You should see output like `allocation=1 chunk=8MB total=8MB`. Press Ctrl+C to stop. If this works, containers will work correctly too.

### Step 7 — Load the kernel module

```bash
sudo insmod monitor.ko
```

Verify it loaded and the device exists:

```bash
lsmod | grep monitor
ls /dev/container_monitor
```

### Step 8 — Start the supervisor

The supervisor is a long-running daemon that accepts CLI commands over the UNIX socket. Run this in a dedicated terminal and leave it open for the entire session.

```bash
sudo ./engine supervisor rootfs
```

Expected output: `[supervisor] ready on /tmp/mini_runtime.sock`

---

## Screenshots and Explanations

### Screenshot 1 — Supervisor Running

**Command:**
```bash
sudo ./engine supervisor rootfs
```

The supervisor process starts and binds to the UNIX domain socket at `/tmp/mini_runtime.sock`. This is the central daemon that all CLI commands communicate with. It stays alive for the entire session, accepting commands, launching containers, and tracking their state. The "ready" message confirms the control socket was created successfully and the supervisor is listening for incoming connections from CLI clients.

---

### Screenshot 2 — Container Execution and Logging (`engine run` + `engine logs`)

**Commands:**
```bash
sudo ./engine run t2 rootfs "/bin/echo HELLO_WORLD"
sudo ./engine logs t2
```

This shows the full container lifecycle. The `run` command launches container `t2`, executes `/bin/echo HELLO_WORLD` inside the isolated chroot environment, waits for it to finish, and returns. The `logs` command retrieves the captured output — `HELLO_WORLD` — from the per-container log file.

This confirms the logging pipeline works end to end. The container's stdout was piped into the supervisor, passed through the bounded-buffer producer-consumer system, written to a log file, and retrieved correctly on demand. It also shows that the container ran in a properly isolated namespace where the command found and executed its binary.

---

### Screenshot 3 — Kernel Module Interaction (`dmesg`)

**Command:**
```bash
sudo dmesg | tail
```

Every time a container starts, the supervisor opens `/dev/container_monitor` and calls `ioctl(MONITOR_REGISTER)` with the container's PID and memory limits. When the container exits, it calls `ioctl(MONITOR_UNREGISTER)`. The kernel module logs both events via `printk`.

The dmesg output shows repeated register/unregister pairs for containers `t1` (PIDs 5176, 5198, 5212, 5230) and then `t2` (PID 5239). Each pair represents one complete container run — registered at launch, unregistered on exit. This confirms that the user-space supervisor and the kernel module are communicating correctly through the `ioctl` interface, which is the kernel-userspace IPC path for memory monitoring.

---

### Screenshot 4 — CLI and IPC (`engine start` via UNIX socket)

**Commands:**
```bash
sudo ./engine start alpha rootfs-alpha /bin/cpu_hog --soft-mib 40 --hard-mib 64
sudo ./engine start beta  rootfs-beta  /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice 10
```

This demonstrates the IPC control channel. When `engine start` is invoked, it runs as a short-lived client process. It connects to `/tmp/mini_runtime.sock`, serialises the start request, and sends it over the UNIX domain socket to the supervisor.

The supervisor receives the request, calls `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` to create an isolated container process, chroots it into the specified rootfs, registers the PID with the kernel module, and sends back the confirmation message. The terminal shows `container 'alpha' started` and `container 'beta' started` — these are the supervisor's responses arriving back through the socket.

Two containers are now running concurrently in the background, each isolated in their own namespace with their own rootfs copy.

---

### Screenshot 5 — Soft Limit Warning

**Commands:**
```bash
sudo ./engine start memtest rootfs-alpha /bin/memory_hog --soft-mib 10 --hard-mib 500
watch -n1 "sudo dmesg | grep container_monitor"
```

The `memory_hog` program allocates memory in 8 MB chunks, one per second. With a soft limit of 10 MiB, it crosses the threshold within the first two seconds of running.

The dmesg output shows:
```
[container_monitor] Registering container=memtest pid=8158 soft=10485760 hard=524288000
[container_monitor] SOFT LIMIT container=memtest pid=8158 rss=17379328 limit=10485760
```

The kernel timer fired, `get_rss_bytes()` returned 17,379,328 bytes (~16.5 MiB) for PID 8158, which exceeds the soft limit of 10,485,760 bytes (10 MiB). The module printed the warning and set an internal `soft_warned` flag so the same container does not generate repeated warnings. The container keeps running — soft limits are advisory, not enforced.

This shows the kernel module correctly performing periodic RSS polling and threshold detection entirely in kernel space, without any involvement from the user-space supervisor.

---

### Screenshot 6 — Hard Limit Enforcement (dmesg + `engine ps`)

**Commands:**
```bash
sudo ./engine start memtest rootfs-alpha /bin/memory_hog --soft-mib 10 --hard-mib 50
watch -n1 "sudo dmesg | grep container_monitor | tail -20"
sudo ./engine ps
```

The dmesg output (first part) shows the complete sequence for container `memtest` with PID 6395:

```
Registering container=memtest pid=6395 soft=10485760 hard=52428800
SOFT LIMIT container=memtest pid=6395 rss=17379328 limit=10485760
HARD LIMIT container=memtest pid=6395 rss=59322368 limit=52428800
Unregister request container=memtest pid=6395
```

When RSS reached 59 MB — exceeding the 50 MiB hard limit — the kernel module called `send_sig(SIGKILL, task, 1)`. The container was terminated immediately. The supervisor's `SIGCHLD` handler fired, called `waitpid()` to reap the child, detected via `WIFSIGNALED` that it was killed by a signal, and updated the container's state.

`engine ps` (second part) confirms the result:
```
ID       PID    STATE    STARTED
memtest  6395   killed   2026-04-06T18:24:28
```

The `killed` state (as opposed to `exited`) specifically means the process was terminated by a signal it did not initiate itself. This distinction in the metadata is important — it tells you whether the container exited cleanly or was forcefully stopped.

---

### Screenshot 7 — Scheduling Experiment (nice values)

**Commands:**
```bash
sudo ./engine start hi rootfs-alpha /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice -5
sudo ./engine start lo rootfs-beta  /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice 15
sudo ./engine logs hi
sudo ./engine logs lo
```

Both containers run the exact same `cpu_hog` binary — a tight CPU loop that prints its progress once per second. The only difference between them is priority: `hi` runs at nice -5 and `lo` runs at nice 15.

The logs tell the story clearly:

- `hi` (nice -5): shows `elapsed=1` through `elapsed=7`, then jumps to `elapsed=23`, and completes with `done duration=10`.
- `lo` (nice 15): shows only `elapsed=1` and then `done duration=10` immediately — it barely made it to the first checkpoint before the log was read.

Linux's CFS (Completely Fair Scheduler) assigns weights based on nice values. A nice of -5 gives roughly 3× the scheduling weight of nice 0, while nice 15 gives significantly less. The scheduler tracks a virtual runtime per process and always picks the runnable task with the lowest virtual runtime. Since `hi`'s virtual runtime advances more slowly relative to real time (because of its higher weight), it gets selected more frequently and makes faster progress.

The gap in the `hi` log (jumps from elapsed=7 to elapsed=23) is interesting — it shows a period where the log was not being read but the container kept running. It is a logging snapshot, not a measure of pauses in execution. The `lo` container getting only `elapsed=1` before completion shows it was severely time-limited by the scheduler.

---

### Screenshot 8 — Clean Teardown

**Commands:**
```bash
ps aux | grep defunct
sudo rmmod monitor
sudo dmesg | grep container_monitor | tail -5
```

After pressing Ctrl+C on the supervisor (which triggered `[supervisor] shutting down...`), we verified clean resource cleanup across both user space and kernel space.

`ps aux | grep defunct` — only the grep process itself appears in the output. No actual container processes are left as zombies. This confirms the supervisor's `SIGCHLD` handler correctly called `waitpid()` for all children before the supervisor exited, preventing any zombie accumulation.

`sudo rmmod monitor` — the kernel module unloads cleanly with no errors, which means no memory was leaked in kernel space (all list nodes were freed in `monitor_exit()`).

`dmesg | grep container_monitor | tail -5` — shows the `hi` and `lo` containers being registered and unregistered in order, followed by `Module unloaded.` as the final line. Nothing was left tracked in the kernel's list.

The teardown sequence confirms that all resources — file descriptors, process entries, kernel memory, the UNIX socket, and the character device — were released properly.

---

## Conclusion

This project gave us a practical, ground-level understanding of how containerisation actually works. We implemented process isolation using Linux namespaces, built a working IPC pipeline with UNIX sockets and pipes, wrote a kernel module that enforces memory policies, and demonstrated real scheduling differences using CFS nice values.

The core OS concepts we worked with directly: `clone()` and namespace isolation, `chroot` and filesystem virtualisation, producer-consumer synchronisation with mutex and condition variables, kernel-userspace communication via `ioctl`, `SIGCHLD` handling and zombie prevention, RSS-based memory monitoring from kernel space, and CPU scheduling weight through nice values.

Building it from scratch made it clear how much of what Docker and similar tools do is simply these Linux primitives composed carefully. The project was a good exercise in systems programming where the abstractions are thin and the consequences of getting things wrong show up immediately.
