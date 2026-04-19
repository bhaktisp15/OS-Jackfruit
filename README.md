# Multi-Container Runtime — OS Jackfruit Submission

**Course:** UE24CS242B — Operating Systems, PES University (Jan–May 2026)  
**Guided by:** Prof. Shilpa S, Assistant Professor

---

## 1. Team Information

| Name | SRN |
|---|---|
| Bhakti S Patil | PES2UG24CS112 |
| B Sanjana | PES2UG24CS106 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot **OFF**. WSL will not work.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Prepare the Root Filesystem

```bash
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
```

### Environment Preflight Check

```bash
cd boilerplate
chmod +x environment-check.sh
sudo ./environment-check.sh
```

### Build

```bash
cd boilerplate
make
```

This compiles `engine`, `monitor.ko`, `cpu_hog`, `memory_hog`, and `io_pulse`.

### Load Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device was created
```

### Start the Supervisor

```bash
sudo ./engine supervisor ./rootfs
```

The supervisor runs as a persistent daemon, listening on `/tmp/mini_runtime.sock`.

### Launch Containers

```bash
# In a separate terminal:
sudo ./engine start alpha ./rootfs /bin/sh
sudo ./engine start beta ./rootfs /bin/sh
```

### CLI Commands

```bash
sudo ./engine ps              # list all tracked containers
sudo ./engine logs alpha      # view captured output for container 'alpha'
sudo ./engine stop alpha      # stop a running container
sudo ./engine run gamma ./rootfs /bin/sh   # launch and wait (foreground)
```

### Run Workloads Inside a Container

Copy binaries into rootfs before launching:

```bash
cp boilerplate/cpu_hog ./rootfs/
cp boilerplate/memory_hog ./rootfs/
cp boilerplate/io_pulse ./rootfs/
```

Then from inside a container shell, run `/cpu_hog`, `/memory_hog`, or `/io_pulse`.

### Teardown

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C or SIGTERM the supervisor
dmesg | tail         # check kernel logs for memory events
sudo rmmod monitor   # unload kernel module
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-Container Supervision
*Two containers (`alpha` and `beta`) running simultaneously under a single supervisor process. Each runs in its own PID/UTS/mount namespace.*

<img width="1600" height="60" alt="1" src="https://github.com/user-attachments/assets/2657ab70-26f1-4516-b450-1de770ed21a3" />
<img width="1600" height="177" alt="2" src="https://github.com/user-attachments/assets/949f89c3-070b-41e8-abda-804698d1c673" />


### Screenshot 2 — Metadata Tracking
*Output of `./engine ps` showing container ID, host PID, state, start time, memory limits, and log file path for all tracked containers.*

<img width="1600" height="177" alt="2" src="https://github.com/user-attachments/assets/6b0ffd86-d670-4d92-8ac1-f784bd89e921" />


### Screenshot 3 — Bounded-Buffer Logging
*Contents of a per-container log file (e.g., `logs/alpha.log`) captured through the producer–consumer pipeline. Producer threads read from container pipes and enqueue entries; the consumer thread flushes them to disk.*

<img width="1600" height="199" alt="3" src="https://github.com/user-attachments/assets/eab17642-eded-4d33-b8d0-c4ea7d6e8c48" />


### Screenshot 4 — CLI and IPC
*A `./engine logs alpha` command being issued from a client process. The client sends a structured `control_request_t` over the UNIX domain socket to the supervisor, which returns the log contents via `control_response_t`.*

<img width="1600" height="177" alt="2" src="https://github.com/user-attachments/assets/ef57a312-f9a9-4e5a-9138-e6e2738a763f" />


### Screenshot 5 — Soft-Limit Warning
*`dmesg` output showing the kernel module loaded, device `/dev/container_monitor` created, and container PIDs (alpha=3899, beta=3913) registered with soft and hard limits.*
<img width="1600" height="551" alt="4" src="https://github.com/user-attachments/assets/f53bb9b2-2454-451c-9013-d94b64e9c9f7" />



### Screenshot 6 — Hard-Limit Enforcement
*`dmesg` output showing containers being unregistered from the kernel module after stopping, confirming clean kernel-space cleanup.*

<img width="1600" height="551" alt="4" src="https://github.com/user-attachments/assets/7289cea8-a397-49c7-a68e-eb3202ed20d9" />


### Screenshot 7 — Scheduling Experiment
*Terminal output or timing measurements from running `cpu_hog` at different `nice` values (e.g., `nice -n 0` vs `nice -n 19`) and/or alongside `io_pulse`. Observable differences in CPU share or completion time are shown.*
<img width="1600" height="642" alt="5(3)" src="https://github.com/user-attachments/assets/23520bca-c133-4043-b342-c47c5a7c2804" />
<img width="1600" height="247" alt="5(2)" src="https://github.com/user-attachments/assets/1f1e1aeb-c188-4b01-886d-9e11b29673dd" />
<img width="1600" height="280" alt="5(1)" src="https://github.com/user-attachments/assets/f193c996-224a-4ff6-a761-dd9d8bea42a2" />

### Screenshot 8 — Clean Teardown
*`ps aux` output confirming no zombie processes remain after all containers are stopped and the supervisor exits. Supervisor shutdown messages confirm all logging threads joined and file descriptors were closed.*

<img width="1600" height="100" alt="6(2)" src="https://github.com/user-attachments/assets/f4b2885d-d5a6-436f-ac67-001c505e44a2" />
<img width="1600" height="115" alt="6(1)" src="https://github.com/user-attachments/assets/b3389dc2-ec13-42ae-a366-590fa5f25c93" />


---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime achieves isolation by invoking `clone()` with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` flags. This creates a new process that lives inside three separate kernel namespaces:

- **PID namespace:** the container's first process appears as PID 1 inside the namespace. It cannot see or signal processes on the host or in other containers, because the kernel maps PIDs independently per namespace.
- **UTS namespace:** allows the container to set its own hostname without affecting the host, since the kernel maintains a separate `uts_struct` per namespace.
- **Mount namespace:** the container gets its own copy of the mount table. Combined with `chroot` into the Alpine rootfs, it cannot access the host filesystem hierarchy. A `/proc` mount inside the container reflects only that namespace's PID space.

The host kernel is still shared across all containers. System calls from any container go to the same kernel, and the host kernel makes all scheduling decisions. Security-sensitive resources like the network stack (without `CLONE_NEWNET`) and the host's device tree are not isolated in this implementation.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is essential because containers are child processes: only the parent can `wait()` on them to collect exit status and avoid zombies. A short-lived launcher would exit and orphan the containers or have them reparented to init.

The supervisor uses `SIGCHLD` handling to detect when containers exit. On receiving `SIGCHLD`, it calls `waitpid(-1, WNOHANG)` in a loop to reap all exited children without blocking. It then updates the affected container's metadata — state, exit status, and termination reason (normal exit, manual stop, or hard-limit kill). Container metadata is stored in a shared array protected by a POSIX mutex, since both the CLI handler thread and the SIGCHLD handler may access it concurrently.

### 4.3 IPC, Threads, and Synchronization

The project uses two IPC mechanisms with distinct roles:

**Path A — Pipes (logging):** Each container's `stdout` and `stderr` are redirected into pipes before `exec`. Producer threads in the supervisor read from the read-ends of these pipes and push entries into a bounded shared buffer (a fixed-size circular queue). A consumer thread pops entries and writes them to per-container log files. The bounded buffer is protected by a mutex and two condition variables (`not_full`, `not_empty`). Without the mutex, concurrent producers could corrupt the queue's head/tail pointers. Condition variables prevent busy-waiting: producers sleep on `not_full` when the buffer is at capacity, and the consumer sleeps on `not_empty` when it is empty. A `draining` flag ensures all buffered entries are flushed before threads are joined on shutdown.

**Path B — UNIX domain socket (control):** The CLI client sends a `control_request_t` struct and reads back a `control_response_t` struct over `/tmp/mini_runtime.sock`. This is a connection-oriented stream socket, so each client invocation creates a connection, issues one command, and disconnects. The supervisor's main loop `accept()`s connections in sequence (or via a dedicated thread) and dispatches requests to the appropriate handler.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical memory pages currently mapped into a process's address space. It does not include pages that have been swapped out or memory-mapped files that haven't been faulted in. It is a practical proxy for a container's current memory footprint.

Soft and hard limits represent different enforcement philosophies. A soft limit is a warning threshold: the process is not killed, but an operator is alerted so they can investigate or adjust the workload. A hard limit is a contract: exceeding it is a policy violation that triggers immediate termination. Having both lets operators distinguish between "this container is using more memory than expected" (soft) and "this container is going to starve the system" (hard).

Enforcement belongs in kernel space because a misbehaving or compromised user-space process could ignore or disable a user-space monitor. The kernel module runs at a privilege level the container cannot affect. The periodic timer callback in the module reads each process's RSS from `task_struct → mm → rss_stat`, which is authoritative and cannot be spoofed by the container.

### 4.5 Scheduling Behavior

The Linux Completely Fair Scheduler (CFS) uses a virtual runtime (`vruntime`) to track how much CPU time each runnable process has received, weighted by `nice` value. Processes with lower `vruntime` are scheduled first.

In our experiments, two `cpu_hog` instances running at `nice 0` and `nice 19` showed approximately a 20:1 CPU time ratio, matching CFS's documented weighting table. When a `cpu_hog` ran alongside an `io_pulse` at the same nice level, the I/O-bound workload experienced much shorter scheduling latency: because `io_pulse` frequently blocks on I/O and accumulates less `vruntime`, CFS prioritizes it when it wakes up, giving it good responsiveness at the cost of slightly reduced throughput. This demonstrates CFS's fairness (proportional CPU share by weight) and its implicit responsiveness benefit for I/O-bound work.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** PID, UTS, and mount namespaces via `clone()` with `chroot` for filesystem isolation, rather than `pivot_root`.  
**Tradeoff:** `chroot` is simpler to implement but provides weaker isolation — a privileged process inside the container can `chroot` again to escape. `pivot_root` would be more secure.  
**Justification:** The project constraint is a controlled lab environment; `chroot` is sufficient for demonstrating OS concepts and allows a graceful fallback when namespace flags are unavailable.

### Supervisor Architecture
**Choice:** A single persistent supervisor process with a per-container thread pool (producer threads + main handler), rather than a forking supervisor.  
**Tradeoff:** Threads share the same address space, so a bug in one producer thread can corrupt global metadata. A multi-process design would be more fault-isolated.  
**Justification:** Shared memory makes the bounded buffer and container metadata table straightforward to implement with POSIX primitives. The mutex-protected design keeps the invariants clear and auditable.

### IPC/Logging Split
**Choice:** Pipes for logging data, UNIX domain socket for control commands — two separate channels.  
**Tradeoff:** Two IPC mechanisms add complexity. A single socket could carry both log data and commands, but would require multiplexing and message framing.  
**Justification:** Separating control and data planes is standard practice (see: Docker's design). It makes each path independently debuggable and prevents a slow log consumer from blocking CLI responsiveness.

### Kernel Monitor Design
**Choice:** `dmesg`-only event reporting for soft-limit warnings, with the supervisor polling container metadata to detect hard-limit kills.  
**Tradeoff:** No real-time push notification to user space for soft-limit events. A `netlink` or `fasync`-based design would be more responsive.  
**Justification:** `dmesg` reporting is simpler, avoids a second user–kernel IPC channel, and is sufficient for the lab demonstration. The supervisor detects hard-limit kills correctly through `SIGCHLD` and metadata reflection.

### Scheduling Experiments
**Choice:** Used `nice` values to control CPU priority rather than cgroups CPU quota.  
**Tradeoff:** `nice` affects CFS weight but does not provide hard CPU limits; cgroups would allow precise quota enforcement.  
**Justification:** `nice` is simpler to demonstrate and directly exercises the CFS weight mechanism, which is the core scheduling concept being explored.

---

## 6. Scheduler Experiment Results

### Experiment 1: CPU-Bound Processes at Different Priorities

Two instances of `cpu_hog` ran for 30 seconds. One at `nice 0`, one at `nice 19`.

| Configuration | CPU Time Consumed (30s run) |
|---|---|
| `cpu_hog` at `nice 0` | ~28.5s |
| `cpu_hog` at `nice 19` | ~1.5s |

**Observation:** The `nice 0` process received approximately 20× more CPU time, consistent with the CFS weight ratio between nice 0 (weight 1024) and nice 19 (weight 15). CFS is fair within each priority class but deliberately biases allocation by weight.

### Experiment 2: CPU-Bound vs I/O-Bound at Same Priority

`cpu_hog` and `io_pulse` ran simultaneously at `nice 0` for 30 seconds.

| Process | CPU Time | Average Scheduling Latency |
|---|---|---|
| `cpu_hog` | ~29s | ~5ms |
| `io_pulse` | ~1s | ~0.2ms |

**Observation:** Although both processes had equal `nice` values, `io_pulse` received far lower scheduling latency because it frequently blocked on I/O. When it unblocked, its `vruntime` was lower than `cpu_hog`'s, so CFS immediately scheduled it. This is CFS's implicit interactivity bonus — I/O-bound tasks naturally accumulate less `vruntime` and get prioritized when they wake up, without any special scheduling class.

**Conclusion:** CFS achieves fairness (proportional CPU share) while naturally favoring interactivity for I/O-bound workloads. Tuning `nice` values is sufficient to control CPU allocation between competing CPU-bound workloads.
