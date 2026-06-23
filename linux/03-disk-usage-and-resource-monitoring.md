# Lesson 3 — Disk Usage and Resource Monitoring

> This is a *foundational* lesson — it sits earlier than the systemd cluster ([Lesson 4](04-systemd-services.md) onward) on purpose, because before you can manage services, logs, or the kernel, you need to answer the most basic operational question of all: **"Is this machine healthy, and if not, which resource is running out?"** Most Linux trouble — slowness, crashes, services that won't start, the dreaded "No space left on device" — is ultimately *some resource being exhausted.* This lesson teaches you to find out which one, with a deep focus on the resource that causes the most sudden, total outages and is the least intuitive: **disk.**

---

## 0. The mental model: a machine has exactly four resources

Here's the single most useful framework for diagnosing *any* "the server is unhealthy" situation. Strip away the details and a computer offers just **four finite resources** that work can contend for. Almost every performance or failure problem is one of these four being exhausted:

1. **CPU** — compute time. Are the processors saturated with work?
2. **Memory (RAM)** — working space. Is there room to hold what's running?
3. **Disk** — *two* distinct things, and conflating them causes endless confusion: **capacity** (is there *space* to store data?) and **throughput/latency** (how *fast* can data move on and off the disk?).
4. **Network** — how fast data moves to and from other machines.

Think of these as the **four utilities of a workshop**: electrical power (CPU), workbench space (memory), the storage warehouse (disk), and the loading dock to the outside world (network). When the workshop grinds to a halt, a good foreman doesn't panic — they methodically check each utility: *Are we out of power? Out of bench space? Is the warehouse full? Is the loading dock jammed?* **Diagnosis is just figuring out which of the four is the bottleneck**, and each has its own gauge.

> **Why this framing first:** beginners debug by guessing. Experts triage by *elimination* across these four. The rest of the Linux monitoring toolkit is just "the gauge for resource N" — and once you know there are only four gauges to check, the dozens of commands stop being a jumble and become a map. This lesson covers disk in depth (the trickiest) and maps the tools for the other three (which [Lesson 7](07-processes-and-monitoring.md) covers in detail).

We start with disk, because "disk full" is uniquely brutal: when a disk fills, programs can't write, databases corrupt, logs stop, and services fail to start — *all at once*, often with cryptic errors. And disk is the resource people understand the least, because of an architectural idea you must grasp first.

---

## 1. The architectural foundation: one tree, many disks

Before any monitoring command makes sense, you need the Linux model of storage, which is profoundly different from Windows and is the source of most beginner confusion.

**On Windows**, each disk gets a letter — `C:`, `D:`, `E:` — and you navigate separate, parallel worlds. **On Linux, there is exactly one tree.** Everything — every file, every disk — hangs off a single root directory called `/`. There are no drive letters. So how do multiple physical disks fit into *one* tree? Through an idea called **mounting**: a disk (or a partition of one) is *attached* to the tree at a specific directory, called its **mount point**, and from then on, walking into that directory means walking onto that disk.

The analogy: imagine a single grand house with one front door (`/`). Some rooms are part of the original structure, but other rooms are actually **separate warehouses connected through doorways.** When you walk through the doorway labeled `/home`, you might be stepping into an entirely different physical building — but to you, walking the house, it's seamless: just another room off the hallway. **A mount point is a doorway where a storage device is grafted into the unified tree.** You could have your operating system on one disk mounted at `/`, a second disk mounted at `/home` for user files, and a USB stick that appears at `/media/usb` when you plug it in — all stitched into one navigable tree.

This is why there are *two layers* you'll monitor, and keeping them straight is half the battle:

- **Block devices** — the raw physical (or virtual) storage hardware and its partitions: `/dev/sda`, `/dev/sda1`, `/dev/nvme0n1p2`. These are the *warehouses themselves*, before they're connected to anything.
- **Filesystems & mount points** — a block device gets *formatted* with a **filesystem** (a scheme for organizing files into directories — `ext4`, `xfs`, `btrfs` are common ones), and then *mounted* at a point in the tree. This is the *doorway and the shelving system inside the warehouse.*

The command that shows you this whole structure at a glance:

```
lsblk
```

`lsblk` ("list block devices") draws your storage as a **tree**: each physical disk, its partitions nested beneath it, their sizes, and crucially *where each is mounted* in the filesystem. It's the map of "what physical storage exists and how it's wired into the directory tree." **Always start a disk investigation here** — it answers "what disks do I even have, and where do they live in the tree?" before you ask "how full are they?"

> **Hold onto this distinction:** a *block device* is the hardware (the warehouse); a *filesystem mounted at a mount point* is how that hardware appears as directories in the one big tree (the connected, shelved, walkable room). The next two commands — `df` and `du` — operate on this model, and confusing the layers is the #1 source of disk confusion.

---

## 2. `df` — how full is each filesystem? (disk *free*)

```
df -h
```

`df` stands for **"disk free,"** and it answers the capacity question at the *filesystem* level: **for each mounted filesystem, how big is it, how much is used, and how much remains?** The `-h` flag (which you'll add to nearly every storage command) means **"human-readable"** — show sizes as `15G` and `230M` instead of raw counts of 1K blocks. Always use `-h`; the raw numbers are a needless math exercise.

Reading the output, column by column, because each answers a different question:

- **Filesystem** — *which* device/filesystem this row describes (e.g. `/dev/sda1`).
- **Size** — its total capacity.
- **Used** — how much is consumed.
- **Avail** — how much is free *and actually usable*. (Note: `Used + Avail` is usually slightly *less* than `Size`, because filesystems reserve a small slice for root and overhead. Normal, not a bug.)
- **Use%** — the percentage full. **This is the number your eye should land on**, the fuel gauge.
- **Mounted on** — *where in the tree* this filesystem is attached (its mount point from Section 1). This is how you connect "the disk is full" to "...which means the directory `/var` is the one in trouble."

The key conceptual point: **`df` reports per-*filesystem*, not per-folder.** It thinks in terms of the mounted warehouses from Section 1. So if `/home` is a separate disk, `df` shows it as its own row with its own Use%, independent of `/`. This is exactly why `lsblk` comes first — `df`'s rows only make sense once you know the mount layout.

> **The fuel gauge habit:** `df -h` is the *first* command to run when anything misbehaves on a server, because a full disk masquerades as a hundred unrelated bugs. A filesystem at `100%` (or even `95%`+) is very often the real, hidden cause of "the app is throwing weird errors." Check the gauge before chasing ghosts.

### A famous gotcha: "df says full, but I deleted the files!"

A classic mystery worth knowing now: you delete a huge log file, but `df` *still* shows the disk full. Why? Because in Linux, a file's space isn't reclaimed until **every process that has it open lets go.** If a running program (say, a service still writing to that "deleted" log) holds the file open, the space stays consumed — the file is gone from the directory but still alive to that process. The fix is to *restart or signal the process holding it* (you'll learn to find it with `lsof` in Lesson 7). The deeper lesson: **disk space is tied to open files, not just directory entries** — a direct consequence of how processes (Lesson 7) and files relate.

---

## 3. `du` — what's *eating* the space? (disk *usage*)

`df` tells you a filesystem is 95% full. The natural next question — *"full of **what**?"* — is what `du` answers. `du` stands for **"disk usage,"** and where `df` reports per-filesystem capacity, **`du` measures how much space a specific directory (and everything beneath it) consumes.** They're complementary halves: `df` is the gauge, `du` is the investigation.

The two forms you'll actually use:

```
du -sh /var/log
```

`-s` = **summary** (one total for the whole path, not a line per file), `-h` = human-readable. This says "how big is `/var/log`, all in?" — a single clean number. Perfect for "how much is this one directory costing me?"

```
du -h --max-depth=1 /var | sort -rh | head
```

This is the **"hunt the space hog" incantation**, and it's worth dissecting because you'll use it constantly:

- `du -h --max-depth=1 /var` — show the size of `/var` itself *and each of its immediate subdirectories* (depth 1), but don't drown you in every nested file.
- `sort -rh` — sort **r**everse (biggest first), respecting **h**uman-readable sizes (so `2G` correctly sorts above `500M`).
- `head` — just the top several.

The result: an instant ranked list of *which subdirectory is the biggest*. Then you `cd` into the winner and run it again, drilling down level by level — like opening the biggest box, finding the biggest box inside *it*, and repeating until you reach the actual culprit. This **drill-down** is the fundamental manual technique for "what filled my disk?"

> **The `df` → `du` workflow, internalized:** `df -h` tells you *which filesystem* is full → you note its mount point → you `du` your way *down* from that mount point, following the biggest subdirectory at each step, until you find the offending files. Gauge, then hunt. This two-step is one of the most-used routines in all of Linux administration.

A subtle but important caveat: `df` and `du` can *disagree* on totals, and now you know why — the deleted-but-open-files gotcha from Section 2 means `df` counts space that `du` (which only walks the visible directory tree) can't see. When they disagree by a lot, suspect a large deleted-but-still-open file.

---

## 4. `ncdu` — the interactive way to hunt

The `du | sort | head` drill-down is powerful but tedious to repeat by hand. **`ncdu`** ("NCurses Disk Usage") is the tool that automates the whole hunt into an interactive explorer — and it's the one to reach for in practice.

```
ncdu /
```

It scans the tree once, then gives you a **navigable, ranked, full-screen view**: directories listed biggest-first, with bars showing relative size. You move with arrow keys, press Enter to *descend* into a directory (which re-ranks by what's inside it), and back out again — doing the Section 3 drill-down *visually and instantly*, without retyping commands. You can even delete files right inside it. It's like a file explorer sorted entirely by "what's taking up the most room," which is exactly the lens you want when hunting a space hog.

`ncdu` usually needs installing (`sudo apt install ncdu` on Ubuntu), so it won't always be present on a fresh server — which is precisely why you learn the `du` incantation in Section 3 first (it's *always* available), and treat `ncdu` as the comfortable upgrade when you can install it. Same relationship as `top` (universal) vs `htop` (nicer) from Lesson 7.

> **The practical reality:** for "my disk filled up and I need to find why *right now*," `ncdu /` (or `ncdu` pointed at the full filesystem's mount point) is the fastest path from "something's eating space" to "*there* it is." It's the disk equivalent of `top` — your live, interactive instrument for the storage resource.

---

## 5. The sneaky second way to run out: inodes

Here's a failure mode that baffles people because the obvious gauge says everything's fine: **`df -h` shows plenty of free space, yet writes fail with "No space left on device."** How can the disk be full *and* empty at once? The answer is **inodes**, and understanding them deepens your model of how filesystems actually work.

When a filesystem is created, it builds a fixed-size table of **inodes** — one **inode per file or directory**, each holding that file's *metadata* (its size, owner, permissions, timestamps, and pointers to where its data physically lives). The filename you see is just a label pointing to an inode; the inode is the file's "real" identity. The crucial fact: **the number of inodes is fixed when the filesystem is formatted.** So a filesystem can run out of storage in *two independent ways*:

- **Out of space** — the data blocks are full (normal "disk full"). A few enormous files do this.
- **Out of inodes** — you've created so many *files* that the inode table is exhausted, *even if each file is tiny and there's gigabytes of block space free.* Millions of little files do this.

The analogy: the warehouse has both **shelf space** (data blocks) and a **fixed set of numbered index cards** in the catalog (inodes), one card per item stored. You can run out of *index cards* while shelves sit half-empty — if you're storing millions of tiny items, you exhaust the catalog before the shelves. With no free card, you can't file a new item, no matter how much shelf room remains.

The gauge for this is `df`'s lesser-known sibling flag:

```
df -i
```

`-i` shows the **inode** usage instead of block usage — same layout (total, used, available, use%), but counting *inodes* rather than *bytes*. **When `df -h` looks fine but writes fail, `df -i` is the very next thing to check.** The usual culprits: a directory drowning in millions of tiny session files, cache fragments, or mail spool files. (`du` with inode-counting, or `ncdu` which shows item counts, then helps you find *where* the file explosion is.)

> **Why this matters for your mental model:** "disk full" isn't one condition, it's two — *space* and *inodes* — because a filesystem manages two separate finite things. Knowing both gauges exist (`df -h` and `df -i`) is what separates "I'm stumped, there's clearly free space" from "ah, we're out of inodes." It's a small piece of knowledge that saves hours exactly once, memorably.

---

## 6. The "disk is full" emergency playbook

Let's make this concrete, because a full disk is one of the most common real incidents you'll face, and there's a standard set of usual suspects. When `df -h` shows a filesystem near 100%, these are the places to look first — the directories that *grow* and commonly run away:

- **Logs in `/var/log`** — services log endlessly, and a misbehaving service can write gigabytes of errors fast. This connects straight to [Lesson 5](05-journalctl.md): the **systemd journal** itself lives here (under `/var/log/journal`) and, if not size-capped, can balloon. The fix — `journalctl --vacuum-size=200M` to trim it — is something you'll appreciate having learned both lessons for. A runaway log is the single most common cause of a full server disk.
- **`/tmp` and `/var/tmp`** — temporary files that programs forgot to clean up. `/tmp` is sometimes a RAM-backed filesystem (so filling it eats *memory*, tying disk to resource #2!), sometimes real disk.
- **Package caches** — `/var/cache/apt` on Ubuntu accumulates downloaded `.deb` packages; `apt clean` reclaims it.
- **Container/Docker storage** — `/var/lib/docker` is notorious for hoarding old images, stopped containers, and volumes; it can quietly consume tens of gigabytes.
- **A user's home directory** — downloads, forgotten datasets, caches in `~/.cache`.
- **Deleted-but-open files** — the Section 2 gotcha: if `du` can't find the space `df` says is used, hunt for a large deleted file still held open by a process.

The disciplined response, putting the whole lesson together:

1. **`df -h`** — confirm *which* filesystem is full (and `df -i` if space looks fine but writes fail).
2. **`lsblk`** — sanity-check the disk/mount layout if anything's surprising.
3. **`ncdu /`** (or `du --max-depth=1` drill-down from the full mount point) — find *what's* eating it.
4. **Reclaim** — delete/rotate the offending files, vacuum the journal, clean caches — and if `df`/`du` disagree, restart the process holding a deleted file.

> **The mindset:** a full disk is rarely mysterious once you have the routine. It's almost always logs, caches, temp files, or containers. Knowing the usual suspects means you spend seconds checking the likely places rather than minutes scanning blindly.

---

## 7. Capacity vs. throughput: monitoring disk *speed*, not just space

Remember from Section 0 that "disk" is really *two* resources. Everything above was about **capacity** (is there room?). The other half is **throughput and latency** (how *fast* can data move?) — and a disk can be killing your performance while being nearly empty. A machine can feel frozen not because the disk is *full* but because it's *saturated* — every program waiting in line to read or write.

You actually met the symptom already in Lesson 7: the **`wa` (I/O wait)** percentage in `top`, and processes stuck in **D-state** (uninterruptible sleep on I/O). High `wa` with low CPU usage is the signature of a **disk-throughput** bottleneck — the CPU is idle *because it's waiting for a slow disk.* That's your first hint that the problem is disk-speed, not disk-space. The dedicated gauges:

```
iostat -xz 2
```

`iostat` reports **per-device I/O statistics**, refreshing every 2 seconds. The columns to care about: **`%util`** (how busy the device is — near 100% means saturated, the disk is the bottleneck) and **`await`** (average time requests wait — high values mean a slow or overwhelmed disk). It's the "how hard is each disk working, and is it keeping up?" gauge. (It's part of the `sysstat` package, often needs installing.)

```
sudo iotop
```

`iotop` is **`top` for disk I/O**: a live, sorted view of *which processes* are reading and writing the most *right now*. Where `iostat` says "the disk is saturated," `iotop` says "...and *this process* is the one hammering it." It's the natural pairing — find the saturation, then find the culprit, exactly mirroring the `df`→`du` and `top`→(per-process) patterns.

> **The capacity/throughput split, cemented:** `df`/`du`/`ncdu` answer **"is there room?"** (and a full disk breaks things outright). `iostat`/`iotop` and `top`'s `wa` answer **"is the disk fast enough?"** (and a saturated disk makes everything *slow*). Two different problems, two different toolsets, both filed under "disk." Always know which one you're chasing.

---

## 8. The other three gauges: a resource-monitoring map

Disk was our deep dive; here's the orientation map for the *other* three resources, so you have one gauge per utility. Most are covered in depth in [Lesson 7](07-processes-and-monitoring.md) — this is the index that ties the four together.

**CPU — "are the processors saturated?"**

- `uptime` — instant **load average** (running + waiting processes; compare to core count). The fastest CPU-pressure glance.
- `top` / `htop` — live per-process CPU use and the `us`/`sy`/`id`/`wa` breakdown (Lesson 7).
- `mpstat -P ALL 2` — per-core utilization, to see if work is balanced or pinned to one core.

**Memory — "is there working space?"**

- `free -h` — total/used/free/available RAM and swap, human-readable. Remember Lesson 7's wisdom: **`buff/cache` isn't "lost"**, watch the `available` figure and whether **swap** is being used (heavy swapping = memory pressure → the OOM killer from Lesson 6 looms).
- `top` / `htop` — sort by memory (`M`) to find the hog.

**Network — "how fast is data moving in/out?"**

- `ss -tulpn` — listening ports and which process owns each (you met this in Lesson 4). The "what's even using the network?" gauge.
- `iftop` / `nethogs` — live bandwidth use, by connection (`iftop`) or **by process** (`nethogs` — the "which program is eating my bandwidth?" tool). Both usually need installing.
- `ip -s link` — per-interface traffic counters and errors.

**All-in-one dashboards** (one screen, several gauges at once):

- `vmstat 2` — the classic combined snapshot: CPU, memory, swap, and I/O in one refreshing line. A great "everything at a glance" pulse.
- `glances` — a modern, friendly full-screen dashboard covering all four resources together (needs installing). When you want *one* tool to eyeball overall health, this is it.

> **The unifying takeaway:** there are only four resources, so there are only four questions. For each, you now have a *quick glance* tool and a *find-the-culprit* tool. Monitoring isn't memorizing fifty commands — it's knowing which of four gauges to read, and then drilling from "which resource" to "which process or directory."

---

## 9. The command cheatsheet

Grouped by intent — *what you're trying to find out* — since that's how you'll reach for them under pressure.

### See your storage layout

| Command | What it does |
|---------|-------------|
| `lsblk` | Tree of physical disks → partitions → mount points (**start here**) |
| `lsblk -f` | Same, plus filesystem type and labels |
| `mount \| column -t` | Every mounted filesystem and its options |
| `findmnt` | Mounts shown as a readable tree |

### How full is the disk? (capacity)

| Command | What it does |
|---------|-------------|
| `df -h` | **The fuel gauge** — used/free per filesystem, human-readable |
| `df -i` | Same, but for **inodes** (check when space is free but writes fail) |
| `df -h /var` | Capacity of just the filesystem holding a given path |

### What's eating the space? (investigation)

| Command | What it does |
|---------|-------------|
| `du -sh /path` | Total size of one directory tree |
| `du -h --max-depth=1 /var \| sort -rh \| head` | Rank immediate subdirectories biggest-first (**drill-down hunt**) |
| `ncdu /` | Interactive, navigable, ranked disk-usage explorer (needs install) |
| `du -ah /path \| sort -rh \| head` | Rank individual *files* biggest-first |
| `find /path -size +500M` | Find files larger than a threshold |

### How fast is the disk? (throughput)

| Command | What it does |
|---------|-------------|
| `iostat -xz 2` | Per-device I/O: watch `%util` (saturation) and `await` (latency) |
| `sudo iotop` | Live, per-process disk read/write — *who* is hammering the disk |
| `top` → watch `wa%` | High I/O-wait + low CPU = disk-throughput bottleneck (Lesson 7) |

### The other three resources (quick glances)

| Command | Resource | What it tells you |
|---------|----------|-------------------|
| `uptime` | CPU | Load average at a glance |
| `top` / `htop` | CPU + memory | Live per-process use (Lesson 7) |
| `free -h` | Memory | RAM & swap; watch `available` and swap use |
| `ss -tulpn` | Network | Listening ports + owning processes |
| `nethogs` | Network | Bandwidth **by process** (needs install) |
| `vmstat 2` | All | Combined CPU/mem/swap/IO pulse |
| `glances` | All | Full-screen all-resource dashboard (needs install) |

### Reclaim space (common fixes)

| Command | What it frees |
|---------|--------------|
| `journalctl --vacuum-size=200M` | Trims the systemd journal (Lesson 5) |
| `sudo apt clean` | Clears downloaded package cache |
| `docker system prune` | Removes unused Docker images/containers/volumes |
| `sudo lsof +L1` | Finds deleted-but-still-open files holding space |

---

## 10. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **the four resources** | CPU, memory, disk, network — the things work contends for | The four workshop utilities |
| **block device** | Raw storage hardware/partition (`/dev/sda1`) | A warehouse before it's connected |
| **filesystem** | The scheme organizing a device into files/dirs (`ext4`) | The shelving system inside |
| **mount point** | The directory where a device is grafted into the one tree | A doorway connecting a warehouse to the house |
| **the single tree** | All storage hangs off one root `/` (no drive letters) | One house, many connected rooms |
| **`lsblk`** | Lists the disk → partition → mount tree | The map of all warehouses |
| **`df` (disk free)** | Used/free capacity *per filesystem* | The warehouse fuel gauge |
| **`du` (disk usage)** | Space consumed by a *directory tree* | Measuring what's in one room |
| **`ncdu`** | Interactive ranked disk-usage explorer | A file browser sorted by bulk |
| **inode** | Fixed-count metadata record, one per file | A numbered catalog index card |
| **out of inodes** | Too many files, even with free space | Out of index cards, shelves half-empty |
| **deleted-but-open file** | Space not freed because a process holds it open | An item "removed" but still in someone's cart |
| **capacity vs throughput** | Disk *room* vs disk *speed* — two different problems | A full warehouse vs a jammed loading dock |
| **`iostat` / `iotop`** | Per-device / per-process disk speed gauges | How hard each dock works / who's hauling |
| **`wa` (I/O wait)** | CPU idle *because* it's waiting on disk | Workers standing idle, waiting on the dock |
| **`free -h`** | RAM and swap usage | Workbench space gauge |
| **load average** | Processes running + waiting (compare to cores) | Cars queued at the tollbooth |
| **`vmstat` / `glances`** | All-resource combined dashboards | The whole control panel at once |

---

## 11. Where to go next

With the four-resources map and the disk deep-dive in hand, the rest of the series builds naturally:

- **[Lesson 7 — Processes & Process Monitoring](07-processes-and-monitoring.md)** — the deep dive on the CPU and memory gauges sketched in Section 8, plus how to act on the process you find eating a resource (signals, priority).
- **[Lesson 4 — systemd Services](04-systemd-services.md)** — once you can spot a service eating disk or CPU, you'll want to manage it: start, stop, limit, and inspect it.
- **[Lesson 5 — journalctl](05-journalctl.md)** — the logs that so often *fill* the disk (Section 6), and how to read and cap them.
- **Filesystem & partitioning deeper** — creating filesystems, resizing partitions, LVM (logical volumes that span disks), and `/etc/fstab` (the config that decides what gets mounted where at boot — the *persistent* version of Section 1's mounting).

The instinct to carry into every other lesson: **when a Linux machine misbehaves, don't guess — check the four gauges, find which resource is exhausted, then drill from "which resource" down to "which process or directory."**
