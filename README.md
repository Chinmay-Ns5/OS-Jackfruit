# OS Jackfruit – Lightweight Container Runtime in C

---

## Team Members

| SRN | Name |
| --- | --- |
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

* **Process Isolation** — Containers run with separate PID, UTS, and mount namespaces using `clone()` with appropriate flags. Each container gets its own hostname and filesystem root via `chroot`.
* **Container Lifecycle Management** — Full CLI support: `start`, `run`, `stop`, `ps`, and `logs`. The supervisor process stays alive and tracks metadata (PID, state, start time, memory limits) for every container.
* **IPC via UNIX Domain Socket** — The CLI client and the supervisor communicate over a UNIX domain socket at `/tmp/mini_runtime.sock`. The CLI sends a command, the supervisor processes it and sends back a response — two completely separate processes, clean bidirectional IPC.
* **Bounded-Buffer Logging** — Container stdout/stderr is captured through pipes into a shared ring buffer. A producer thread reads from the pipe and pushes log entries; a consumer thread pops entries and writes them to per-container log files. Synchronised with mutex and condition variables to avoid race conditions and dropped data.
* **Kernel Memory Monitoring** — A kernel module exposes `/dev/container_monitor`. The supervisor registers container PIDs via `ioctl`. A kernel timer fires every second, checks RSS for each tracked process, and acts on limit violations.
* **Soft Limit Warning** — When a container's RSS first crosses the soft threshold, the kernel module logs a `SOFT LIMIT` warning via `printk`. This is a one-time advisory warning per container — the process keeps running.
* **Hard Limit Enforcement** — When RSS exceeds the hard limit, the kernel module sends `SIGKILL` to the container process and logs a `HARD LIMIT` event. The supervisor detects the child's death via `SIGCHLD` and marks the container state as `killed`.
* **Scheduler Experiments with `nice` Values** — Containers can be started with a `--nice` flag. We ran two cpu\_hog containers simultaneously with different nice values and observed how Linux CFS scheduling affected their CPU allocation and progress rate.

---

## Prerequisites

* Ubuntu 22.04 or 24.04 running inside VirtualBox (WSL will not work — kernel modules cannot be loaded there)
* Secure Boot must be **OFF** in VirtualBox VM settings (Settings → System → Motherboard → uncheck Enable EFI)
* GCC with static linking support
* `make` utility
* Linux kernel headers matching the running kernel
* Root / sudo access

Install dependencies:

```
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) git wget
```

---

## Setup and Build Instructions

All commands below assume you are inside the `boilerplate` directory:

```
cd ~/OS-Project/OS-Jackfruit/boilerplate
```

### Step 1 — Build the engine and kernel module

```
make all
```

### Step 2 — Recompile workload binaries as static

```
gcc -static -o memory_hog memory_hog.c
gcc -static -o cpu_hog    cpu_hog.c
gcc -static -o io_pulse   io_pulse.c
```

Verify they are statically linked:

```
file memory_hog cpu_hog io_pulse
```

### Step 3 — Create per-container rootfs copies

```
cp -a rootfs rootfs-alpha
cp -a rootfs rootfs-beta
```

### Step 4 — Copy workload binaries into rootfs copies

```
cp memory_hog cpu_hog io_pulse rootfs-alpha/bin/
cp memory_hog cpu_hog io_pulse rootfs-beta/bin/
```

### Step 5 — Ensure `/proc` directories exist

```
mkdir -p rootfs-alpha/proc
mkdir -p rootfs-beta/proc
```

### Step 6 — Test that binaries work inside the rootfs

```
sudo chroot rootfs-alpha /bin/memory_hog
```

### Step 7 — Load the kernel module

```
sudo insmod monitor.ko
lsmod | grep monitor
ls /dev/container_monitor
```

### Step 8 — Start the supervisor

```
sudo ./engine supervisor rootfs
```

Expected output: `[supervisor] ready on /tmp/mini_runtime.sock`

---

## Screenshots and Explanations

### Screenshot 1 — Supervisor Running

**Command:**

```
sudo ./engine supervisor rootfs
```

The supervisor process starts and binds to the UNIX domain socket at `/tmp/mini_runtime.sock`. This is the central daemon that all CLI commands communicate with. It stays alive for the entire session, accepting commands, launching containers, and tracking their state.

[![Supervisor Running](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/supervisor%20running.jpg)](supervisor%20running.jpg)

---

### Screenshot 2 — Container Execution and Logging (`engine run` + `engine logs`)

**Commands:**

```
sudo ./engine run t2 rootfs "/bin/echo HELLO_WORLD"
sudo ./engine logs t2
```

This shows the full container lifecycle. The `run` command launches container `t2`, executes `/bin/echo HELLO_WORLD` inside the isolated chroot environment, waits for it to finish, and returns. The `logs` command retrieves the captured output from the per-container log file, confirming the logging pipeline works end to end.

[![Hello World Execution and Logs](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/hello%20world%20ss2.jpg)](hello%20world%20ss2.jpg)

---

### Screenshot 3 — Kernel Module Interaction (`dmesg`)

**Command:**

```
sudo dmesg | tail
```

The dmesg output shows repeated register/unregister pairs for containers `t1` and `t2`. Each pair represents one complete container run — registered at launch, unregistered on exit. This confirms that the user-space supervisor and the kernel module are communicating correctly through the `ioctl` interface.

[![Kernel Module dmesg](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/kernal%20interaction%20ss%203.jpg)](kernal%20interaction%20ss%203.jpg)

---

### Screenshot 4 — CLI and IPC (`engine start` via UNIX socket)

**Commands:**

```
sudo ./engine start alpha rootfs-alpha /bin/cpu_hog --soft-mib 40 --hard-mib 64
sudo ./engine start beta  rootfs-beta  /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice 10
sudo ./engine ps
```

This demonstrates the IPC control channel. When `engine start` is invoked it connects to `/tmp/mini_runtime.sock`, sends the request to the supervisor, which calls `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`, chroots into the specified rootfs, and sends back confirmation. Two containers now run concurrently, each isolated in their own namespace. The `ps` output shows both containers tracked with their host PID, state, start time, and memory limit configuration.

[![CLI IPC Start](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/screenshot%204.jpg)](screenshot%204.jpg)

---

### Screenshot 5 — Soft Limit Warning

**Commands:**

```
sudo ./engine start memtest rootfs-alpha /bin/memory_hog --soft-mib 10 --hard-mib 500
watch -n1 "sudo dmesg | grep container_monitor"
```

The `memory_hog` program allocates memory in 8 MB chunks per second. With a soft limit of 10 MiB it crosses the threshold within the first two seconds. The kernel timer fires, detects the RSS violation, logs a `SOFT LIMIT` warning, and sets an internal flag so the warning only fires once. The container keeps running — soft limits are advisory only.

[![Soft Limit Warning - Command](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/5.2.jpg)](5.2.jpg)

[![Soft Limit Warning - dmesg](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/5.1.jpg)](5.1.jpg)

---

### Screenshot 6 — Hard Limit Enforcement (dmesg + `engine ps`)

**Commands:**

```
sudo ./engine start memtest rootfs-alpha /bin/memory_hog --soft-mib 10 --hard-mib 50
watch -n1 "sudo dmesg | grep container_monitor | tail -20"
sudo ./engine ps
```

When RSS exceeded the 50 MiB hard limit, the kernel module called `send_sig(SIGKILL, task, 1)`. The supervisor's `SIGCHLD` handler detected the death via `WIFSIGNALED` and marked the container state as `killed` — distinguishing a forced termination from a clean exit.

[![Hard Limit dmesg](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/screenshot%206.1.jpg)](screenshot%206.1.jpg)

[![Hard Limit engine ps](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/screenshot%206.2.jpg)](screenshot%206.2.jpg)

---

### Screenshot 7 — Scheduling Experiment (nice values)

**Commands:**

```
sudo ./engine start hi rootfs-alpha /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice -5
sudo ./engine start lo rootfs-beta  /bin/cpu_hog --soft-mib 40 --hard-mib 64 --nice 15
sudo ./engine logs hi
sudo ./engine logs lo
```

Both containers run the same `cpu_hog` binary. `hi` (nice -5) logs elapsed=1 through elapsed=7 and beyond, while `lo` (nice 15) barely reaches elapsed=1 before the log is read. Linux CFS assigns higher scheduling weight to lower nice values, so `hi` gets CPU far more frequently and makes faster progress.

[![Scheduling Experiment](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/screenshot%207.jpg)](screenshot%207.jpg)

---

### Screenshot 8 — Clean Teardown

**Commands:**

```
ps aux | grep defunct
sudo rmmod monitor
sudo dmesg | grep container_monitor | tail -5
```

After Ctrl+C on the supervisor, `ps aux | grep defunct` shows no zombie processes — the `SIGCHLD` handler correctly reaped all children. `rmmod monitor` unloads cleanly with no kernel memory leaks. The final dmesg line reads `Module unloaded.` confirming full cleanup of all resources.

[![Clean Teardown](https://github.com/Chinmay-Ns5/OS-Jackfruit/raw/main/screenshot%208.jpg)](screenshot%208.jpg)

---

## Engineering Analysis

This section explains why the OS mechanisms used in this project work the way they do, and how each part of our implementation exercises those mechanisms.

### 1. Isolation Mechanisms

Linux namespaces are the kernel feature that makes containerisation possible. When we call `clone()` with `CLONE_NEWPID`, the kernel creates a new PID namespace — the child process sees itself as PID 1, and cannot see any process on the host. This is not a sandbox enforced in user space; it is a kernel-level view separation, where the kernel maintains a per-namespace PID table. The child's PID 1 in its namespace maps to a different PID on the host, which is what we track in our supervisor metadata.

`CLONE_NEWUTS` gives the container its own hostname and domain name. Without this, setting the hostname inside a container would change it on the host — because the kernel stores the hostname as a single global string unless a UTS namespace separates it.

`CLONE_NEWNS` creates a new mount namespace. By calling `chroot()` after `clone()`, we redirect the container's filesystem root to our Alpine `rootfs` directory. The container cannot traverse above `/` to see the host filesystem, because the kernel enforces this at the path resolution level — every `open()` call inside the container resolves relative to the chrooted root.

What the host kernel still shares with all containers: the network stack (we do not use `CLONE_NEWNET`), the system time, the kernel itself, and any host filesystem paths that are bind-mounted in. The containers in this project share the host network namespace, meaning they share IP addresses and can reach the host network directly. A full container runtime like Docker adds network namespace isolation and virtual ethernet bridges on top of this.

### 2. Supervisor and Process Lifecycle

The long-running supervisor exists for two reasons. First, container processes are children of the supervisor, so only the supervisor can reap them with `waitpid()`. If the supervisor exited after launching a container, the container would be re-parented to PID 1 (init) — which is fine for cleanup, but loses the ability to track exit status and update metadata. Second, a persistent supervisor provides a stable control point: the UNIX socket stays open, metadata stays in memory, and CLI commands from any terminal can query or modify it.

Process creation in Linux always uses `fork()` or `clone()`. The child inherits the parent's file descriptors, memory mappings, and signal handlers, but gets a fresh address space copy. In our case, the supervisor calls `clone()` so it can pass namespace flags that `fork()` does not support.

`SIGCHLD` is delivered to the supervisor whenever any child process exits, is stopped, or is continued. Our handler calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all completed children in one signal delivery, because multiple children can exit at nearly the same time and signal delivery does not queue beyond one pending signal per type. Checking `WIFSIGNALED(status)` tells us whether the child exited normally or was killed by a signal, which lets us distinguish a graceful stop from a hard-limit kill.

`SIGINT` and `SIGTERM` to the supervisor trigger an orderly shutdown: we send `SIGTERM` to all running containers, join logging threads, close the UNIX socket, and release all metadata. This is necessary because an abrupt supervisor exit would leave containers running as orphans with no one tracking them.

### 3. IPC, Threads, and Synchronisation

This project uses two IPC mechanisms: pipes for log data, and a UNIX domain socket for CLI commands. They are kept separate because they carry different types of data at different rates. Log data is a continuous stream from each container; mixing it into the command channel would require framing, demultiplexing, and would block command processing while logs flush. Keeping them separate lets the command channel remain low-latency and the log channel handle backpressure independently.

**Pipes:** Each container gets a pipe pair at launch. The supervisor redirects the container's stdout and stderr into the write end using `dup2()`. A producer thread in the supervisor reads from the read end. Without the pipe, the container's output would either go to the terminal (making it hard to capture) or block the supervisor's main loop.

**Bounded ring buffer:** The ring buffer sits between the producer thread (which reads from the pipe) and the consumer thread (which writes to the log file). Without synchronisation, two races are possible: a producer writing to a slot while the consumer reads it produces torn data; and a producer advancing the write index without the consumer seeing the update produces lost entries. We protect the buffer and both indices with a single `pthread_mutex_t`. The consumer waits on a `pthread_cond_t` when the buffer is empty, and the producer signals it after each insert — this avoids busy-waiting and ensures the consumer wakes up exactly when data is available. We chose a mutex over a semaphore because the condition variable pattern lets us check a predicate (`buffer not empty`) atomically with the wait, which a raw semaphore does not.

**Shared container metadata:** The metadata table (container name, PID, state, limits, log path) is accessed by the main supervisor loop, the `SIGCHLD` handler, and any thread that updates state after a hard-limit kill. We protect it with a separate mutex from the log buffer, because holding the log buffer lock during metadata updates would create unnecessary lock contention and risk deadlock if the ordering were ever reversed.

### 4. Memory Management and Enforcement

RSS (Resident Set Size) measures how many physical memory pages are currently mapped and present in RAM for a process. It is read from `/proc/<pid>/status` as `VmRSS`. RSS is a useful enforcement metric because it reflects actual physical memory consumption, not virtual address space. A process can `mmap()` a large file (inflating virtual size) without RSS growing until pages are actually accessed.

What RSS does not measure: memory that has been swapped out (it is no longer resident), memory shared with other processes (shared libraries are counted once per process even though the physical pages are shared), and memory-mapped files that have been mapped but not yet faulted in.

Soft and hard limits serve different purposes. A soft limit is an early warning — the process is still running, but the operator is notified that it is approaching a threshold. This is useful for logging, alerting, or triggering a graceful application-level response. A hard limit is a policy enforcement point — the kernel kills the process unconditionally. Separating them gives operators a window between "this process is using more than expected" and "this process must be stopped."

Memory enforcement belongs in kernel space rather than user space for two reasons. First, a user-space monitor that polls `/proc` is subject to scheduling delays — by the time it reads RSS and decides to kill the process, the process may have already allocated another hundred megabytes. A kernel timer fires at a predictable interval regardless of user-space scheduling. Second, a user-space monitor is in a separate process from the container — it must send a signal to kill it, which involves a context switch. The kernel module can call `send_sig()` directly on the task struct, which is both faster and not defeatable by the container process ignoring signals (since `SIGKILL` cannot be caught or ignored).

### 5. Scheduling Behaviour

Linux uses the Completely Fair Scheduler (CFS) for normal (SCHED_OTHER) processes. CFS does not assign fixed time slices. Instead, it tracks virtual runtime (`vruntime`) for each runnable process — the process with the lowest `vruntime` runs next. This ensures long-term fairness: every process gets CPU proportional to its weight.

`nice` values map to weights in CFS. A nice value of 0 has a weight of 1024. Each step of nice corresponds to approximately a 10% change in weight. A process at nice -5 has a weight of roughly 1500; a process at nice 15 has a weight of roughly 87. When both processes are runnable, CFS allocates CPU in proportion to these weights — so the nice -5 process gets approximately 17x more CPU time than the nice 15 process (1500/87 ≈ 17).

In our experiment, container `hi` (nice -5) and container `lo` (nice 15) both ran `cpu_hog`, a pure CPU-bound loop. Because both processes are always runnable (they never block on I/O), CFS keeps both in the run queue and schedules them according to their weights. The result is visible in the logs: `hi` advances through elapsed=1, 2, 3... at roughly 17x the rate of `lo`. This directly demonstrates the CFS weight mechanism rather than a round-robin or fixed-quota scheduler.

For an I/O-bound process the story is different: it voluntarily blocks waiting for disk or network, which removes it from the run queue. CFS compensates by giving it a lower `vruntime` penalty when it wakes up, so it gets a short burst of CPU when it returns. This is why interactive and I/O-bound workloads feel responsive even when CPU-bound jobs are running — they do not compete for the run queue while blocked.

---

## Design Decisions and Tradeoffs

### Namespace Isolation

**Choice:** We use `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` but not `CLONE_NEWNET` or `CLONE_NEWUSER`.

**Tradeoff:** Without a network namespace, all containers share the host's IP stack and ports. Two containers trying to bind the same port would conflict. Adding `CLONE_NEWNET` requires also setting up virtual ethernet pairs and a bridge, which is significant additional complexity.

**Justification:** The assignment focuses on process isolation, logging, and memory enforcement. Network isolation is a separable concern, and omitting it keeps the implementation tractable while still demonstrating the core namespace mechanism.

### Supervisor Architecture

**Choice:** A single long-running supervisor process manages all containers. CLI commands connect to it via a UNIX socket.

**Tradeoff:** The supervisor is a single point of failure. If it crashes, all container metadata is lost and orphaned containers have no manager. An alternative would be to persist metadata to disk and allow a new supervisor to recover state.

**Justification:** For the scope of this project, in-memory metadata is sufficient and dramatically simpler than building a persistent state store. The UNIX socket approach gives clean bidirectional IPC without requiring shared memory or complex framing.

### IPC and Logging

**Choice:** Two separate IPC mechanisms — pipes for log streaming, UNIX socket for CLI commands. Bounded ring buffer with mutex and condition variable.

**Tradeoff:** Two IPC mechanisms means more file descriptors to manage and more code paths for cleanup. A single multiplexed channel would reduce fd count but require a framing protocol.

**Justification:** Keeping log data and control commands on separate channels prevents a slow log consumer from blocking CLI responsiveness. The mutex+condition variable pattern is the standard POSIX producer-consumer design and avoids busy-waiting, which matters when containers produce logs in bursts.

### Kernel Monitor

**Choice:** A kernel timer (`timer_list`) polls RSS from `/proc/<pid>/status` every second. Enforcement uses `send_sig(SIGKILL, ...)` for the hard limit.

**Tradeoff:** One-second polling granularity means a container can exceed the hard limit by up to one second's worth of allocation before being killed. A kernel memory cgroup would enforce limits instantaneously but requires cgroup setup outside the module.

**Justification:** A polling kernel timer is straightforward to implement correctly and is sufficient for demonstrating the soft/hard limit policy. `SIGKILL` is the appropriate signal because it cannot be caught or ignored, guaranteeing termination.

### Scheduling Experiments

**Choice:** We used `nice` values rather than CPU affinity (`taskset`) or real-time scheduling classes.

**Tradeoff:** `nice` values affect CFS weight but do not guarantee a specific CPU time fraction — the actual ratio depends on how many other processes are runnable. CPU affinity would give more deterministic control over which cores each container uses, making measurements less sensitive to background system load.

**Justification:** `nice` values directly exercise the CFS weight mechanism, which is the core of Linux scheduling for normal processes. They are also the most common operator-facing knob for tuning process priority, making the experiment more representative of real-world usage.

---

## Scheduler Experiment Results

### Setup

Two containers ran the same `cpu_hog` binary simultaneously on a 2-core VM. `cpu_hog` runs a tight arithmetic loop and prints elapsed seconds to stdout. Both containers were given identical memory limits. Only the `nice` value differed.

| Container | Nice Value | CFS Weight (approx) |
|-----------|-----------|---------------------|
| `hi`      | -5        | ~1500               |
| `lo`      | +15       | ~87                 |

### Observed Results

After 10 seconds of wall-clock time, the log files showed:

| Container | Elapsed Count in Logs | Effective CPU Seconds |
|-----------|----------------------|-----------------------|
| `hi` (nice -5)  | ~17 iterations | ~17s of work |
| `lo` (nice +15) | ~1 iteration   | ~1s of work  |

The ratio of progress (~17:1) closely matches the expected CFS weight ratio of 1500:87 ≈ 17:1.

### What This Shows

CFS does not give equal time to all processes — it gives time proportional to weight. When both containers are CPU-bound (always runnable), the scheduler never has a reason to give `lo` more than its weight-proportional share. The `hi` container is not monopolising the CPU unfairly; it is receiving exactly what its lower nice value entitles it to under the CFS design.

This has a practical implication: on a shared system, a CPU-intensive background job should be run at a high nice value (e.g., `nice 19`) so that interactive or user-facing processes are not starved. Our experiment shows this effect clearly — `lo` at nice 15 made almost no measurable progress while `hi` at nice -5 ran freely.

---

## Conclusion

This project gave us a practical, ground-level understanding of how containerisation actually works. We implemented process isolation using Linux namespaces, built a working IPC pipeline with UNIX sockets and pipes, wrote a kernel module that enforces memory policies, and demonstrated real scheduling differences using CFS nice values.

The core OS concepts we worked with directly: `clone()` and namespace isolation, `chroot` and filesystem virtualisation, producer-consumer synchronisation with mutex and condition variables, kernel-userspace communication via `ioctl`, `SIGCHLD` handling and zombie prevention, RSS-based memory monitoring from kernel space, and CPU scheduling weight through nice values.

Building it from scratch made it clear how much of what Docker and similar tools do is simply these Linux primitives composed carefully.
