# Linux Lessons

A personal, concept-first course on how Linux actually works. Each lesson leads with mental models and analogies, explains the *why* behind every command, and ends with a glossary and pointers onward — no dry command dumps.

## Naming convention

Lessons are markdown files named `NN-topic.md`, where:

- **`NN`** is a two-digit ordering number (`01`, `02`, ...), so the folder reads top-to-bottom in learning order.
- **`topic`** is a short kebab-case subject (e.g. `systemd-services`).

Example: `04-systemd-services.md`.

## Lessons

### 02 — [The Linux Filesystem: One Tree, Many Disks](02-filesystem-and-mounts.md)

The most foundational lesson: Linux is **one unified tree** from `/` (no drive letters), and separate disks are grafted in by **mounting**. Builds the full stack — the **FHS** layout (`/etc`, `/var`, `/home`, `/dev`, `/proc`...), "everything is a file" and device naming (`/dev/nvme0n1p5` decoded), **partitions** (where they come from — the partition table & the installer), **filesystems** (ext4/vfat/tmpfs) and formatting, then the keystone: **mounting & mount points** (the "door to another building" analogy), persistence via **`/etc/fstab` and UUIDs**, and **virtual/RAM filesystems** (`tmpfs`, `proc`, `efivarfs`). Payoff: a **line-by-line decode of a real `df -h`** — sorting every row into real-disk / RAM / virtual / network — plus the **boot story** of how the tree assembles (firmware → kernel → systemd). Read this first; Lesson 3 measures what this one explains.

### 03 — [Disk Usage & Resource Monitoring](03-disk-usage-and-resource-monitoring.md)

A foundational orientation: a machine has exactly **four finite resources** — CPU, memory, disk, network — and most trouble is one of them being exhausted. Centers on the trickiest, **disk**: the single unified tree and **mount points** (`lsblk`), capacity (`df -h`, the deleted-but-open-files gotcha), hunting space hogs (`du` drill-down, `ncdu`), the **inode** second-way-to-run-out (`df -i`), a "disk is full" emergency playbook, and **capacity vs. throughput** (`iostat`/`iotop` and `top`'s `wa`). Closes with a one-gauge-per-resource map for CPU/memory/network and a grouped cheatsheet. Reads first; the systemd cluster builds on it.

### 04 — [systemd Services](04-systemd-services.md)

The system & service manager, end to end. Organized around three questions you should be able to answer about any process: *what launched this, why did it stop, and what environment did it inherit?*

Covers: PID 1 and the manager model, units vs services vs targets, cgroups, `enabled` vs `active`, failure & restart loops, unit-file anatomy (`ExecStart` / `ExecStartPre` / `Type`), targets, drop-in overrides & `daemon-reload`, environment inheritance, and user services. Closes with a practical trio:

- **Ubuntu's common & critical services** — a field guide to the regulars you expect to find.
- **Inspecting what's running** — surveying your own machine and what to look for.
- **Spotting a suspicious service** — recognizing a stranger by how it breaks the normal patterns.

### 05 — [journalctl & the systemd Journal](05-journalctl.md)

The system-wide, structured log and how to query it — the history behind `systemctl`'s "why did it stop?" Built on the idea that logs are *structured records*, not text lines, and on four axes of narrowing: **which boot** (`-b`, `-b -1`), **which service** (`-u`), **how severe** (`-p`, priority ranges), and **what time window** (`--since`/`--until`) — plus the live-follow mode (`-f`). Closes with where the journal lives (persistent vs volatile), journalctl vs `/var/log` text files, and the three questions revisited with the dimension of time.

### 06 — [dmesg & the Kernel Ring Buffer](06-dmesg.md)

One floor below services and the journal: the **kernel** itself. Built on the **kernel space vs user space** wall and on *why* the kernel needs its own fixed-size, self-overwriting **ring buffer** (the early-boot chicken-and-egg problem). Covers reading it (`sudo dmesg`, and why `dmesg_restrict` gates it), making it legible (`-H`, `-T` and the seconds-since-boot timestamp mystery), filtering by the same 0–7 severity scale (`--level`, `-x`), the live feed (`-w`), and the high-value events to recognize — hardware appear/disappear, disk/IO errors, the **OOM killer**, oops/panics. Resolves the buffer's ephemerality via `journalctl -k`, and revisits the three questions at the kernel layer.

### 07 — [Processes & Process Monitoring](07-processes-and-monitoring.md)

The **atom** beneath Lessons 4–6: a service is processes, the journal logs processes, the OOM killer kills processes. Built on **program vs. process** (recipe vs. cook), process anatomy (PID/PPID, the tree, UID, VIRT vs RES), and the states (R/S/**D**/T/Z, with orphans vs zombies). Centers on `top` — reading its **summary header** (load average as a tollbooth queue, the `wa` I/O-wait tell, memory & `buff/cache`) and its **process table + interactive keys** as a live instrument, not a passive report. Then control: **niceness** (`nice`/`renice`), **signals** (SIGTERM vs SIGKILL vs SIGHUP, `kill`/`pkill`/`pgrep`), and job control (`bg`/`fg`/`nohup`). Ends with a grouped **command cheatsheet** and the three questions at the process layer.

### 08 — [Memory Deep-Dive](08-memory-deep-dive.md)

Pays off the memory promises from Lessons 3, 6, and 7. Built on the central illusion of **virtual memory** (every process's imagined private desk vs. the small shared real one), **pages / demand paging / page faults** (minor vs. major), and finally explains **VIRT vs RES vs SHR** + overcommit and the **PSS** double-counting trap (`smem`). The conceptual hinge — **anonymous vs. file-backed** memory — explains the **page cache** (why `available`, not `free`, is the number that matters) and **swap** (a pressure valve; *thrashing*, not usage, is the disease). Covers **reclaim** (`kswapd` vs. direct), then the payoff: **reading memory pressure early** via `free`/`MemAvailable`, `vmstat` `si`/`so`, and **PSI** (`/proc/pressure/memory`) — *before* the OOM killer fires. Closes by revisiting the OOM killer with `oom_score_adj` and **cgroup limits** (`MemoryMax`/`MemoryHigh`, Lesson 4), a cheatsheet, and a glossary.

### 09 — [IPC: How Processes Talk (Pipes & Sockets)](09-ipc-pipes-and-sockets.md)

The communication layer beneath the system. Built on **process isolation** (Lesson 7: each process an island the kernel keeps walled off) and the question it forces: *if processes can't touch each other's memory, how do they cooperate?* The answer — the **kernel as post office**, lending channels along a few axes (data shape, who/where, named?, direction, copy-vs-share). Tours the family: **signals** (payload-free notification), **pipes** (the shell's `\|`, the IPC you already use daily), **FIFOs** (named pipes), **Unix domain sockets** (the two-way local workhorse — addressed by a `/run/...` path, the substrate D-Bus is built on), **network sockets** (the same API across machines), and **shared memory + semaphores** (fastest, self-coordinated). Unifies them under **"everything is a file descriptor"** (Lessons 2 & 7 paying off — a process's fd table *is* its switchboard), then shows how to *see* it all live: `lsof -U`, `ss -x`/`-tulpn`, `ls -l` socket (`s`) and FIFO (`p`) types, and `strace` to watch the syscalls.

### 10 — [D-Bus: the System's Switchboard](10-dbus.md)

Why a raw socket isn't enough once *dozens* of services must talk: no directory, no common language, no broadcast. **D-Bus** is the structured **message bus** that fixes all three — one switchboard daemon on a Unix socket (the Lesson 9 anchor) that everyone plugs into. Covers the **system vs session** buses (privilege & lifetime), the four-level **addressing scheme** (bus name → object path → interface → member, read as *company → department → catalog → action*), the four **message types** (method call/return/error + broadcast **signals** — and the crucial *D-Bus signal ≠ Unix signal* warning), **properties & introspection** (the self-describing bus), **service activation** (the same lazy-start as systemd socket units), and **security via bus policy + polkit** (that password prompt *is* a D-Bus conversation). Ends by revealing how much you already rely on it — `systemctl`, logind/power, NetworkManager, notifications, drive-mounting, and **Flatpak portals** — all driven with `busctl` (`list`/`introspect`/`call`/`monitor`).

## Packaging (sub-course)

Debian/Ubuntu packaging — `dpkg`, `apt`, repositories, and building `.deb`s yourself — lives in its own folder: **[packaging/](packaging/)**. It's a self-contained topic with its own toolchain, but it builds on the disk and systemd lessons here (and links back).

- **[01 — Package Management: `dpkg`, `apt`, `apt-get`, `aptitude`](packaging/01-package-management.md)** — the two-layer supply-chain model (`dpkg` the dock worker vs `apt` the logistics manager), the two-phase install, maintainer scripts, repositories & sources, the `dpkg -l` state codes, and the broken-upgrade recovery + safe-upgrade pattern.
- **[02 — Building Your Own `.deb`](packaging/02-building-your-own-deb.md)** — hand-assemble a real systemd-service package (`control`, `conffiles`, all four maintainer scripts), watch the unpack→configure phases live, then deliberately break `postinst` to reproduce and recover the `iF` state.
- **[03 — Snap & Flatpak](packaging/03-snap-and-flatpak.md)** — the sealed-container worlds beside `apt`: self-contained, sandboxed, auto-updating. Snap (squashfs `loop` mounts, single store, channels, confinement) vs Flatpak (remotes, runtimes, portals), why Firefox is installed two ways, and how to tell which copy you're running.

## Suggested next lessons

These build directly on the foundations laid above:

- **Writing your own service from scratch** — build, enable, and debug a real unit file end to end (then watch it with `journalctl -fu`).
- **Timers** — the modern, systemd-native replacement for cron, on the same unit/dependency machinery.
- **Resource control via cgroups** — `MemoryMax`, `CPUQuota`, and capping what a service is allowed to consume.
