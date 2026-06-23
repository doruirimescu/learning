# Lesson 8 — Memory Deep-Dive: Virtual, Resident, Shared, Swap, and the Page Cache

> Three earlier lessons left memory promissory notes this one pays off. [Lesson 7](07-processes-and-monitoring.md) showed you `top`'s **VIRT / RES / SHR** columns and admitted VIRT was "partly fictional" without explaining *why*. [Lesson 6](06-dmesg.md) watched the kernel's **OOM killer** execute a process to survive memory exhaustion. [Lesson 3](03-disk-usage-and-resource-monitoring.md) called memory one of the four resources and warned that `buff/cache` "isn't lost" and that heavy **swapping** precedes an OOM kill. Now we open the box. By the end you'll understand *what those columns actually mean*, why "free memory" is the wrong number to watch, how swap and the page cache really work, and — the practical payoff — **how to read mounting memory pressure early enough to act before the OOM killer fires.**

---

## 0. The central illusion: every process thinks it owns all the memory

Everything about Linux memory flows from one audacious trick, so we build it first. Without it, the rest is arbitrary; with it, the rest is obvious.

When a process runs, it sees what *looks like* a vast, private, contiguous stretch of memory addresses all to itself — starting at address zero and running up to an enormous ceiling — as though it were the only thing on the machine and owned all of RAM (and then some). **This is a lie the kernel tells every process, deliberately and beautifully.** It's called **virtual memory**, and that private fictional space is the process's **virtual address space.**

Reality is quite different: there's one modest pool of **physical RAM** shared by hundreds of processes, and no single process comes close to owning it all. The magic is a translation layer — a hardware unit in the CPU called the **MMU** (Memory Management Unit), directed by tables the kernel maintains — that, on *every single memory access*, converts the *virtual* address the process used into the *physical* address where the data actually lives (or notices it isn't in RAM at all and fetches it).

The analogy that will carry this whole lesson — a deliberate extension of the "workbench/desk space" image from Lessons 3 and 7:

> **Physical RAM is a small, fast desk. Virtual memory is the illusion, maintained by a tireless assistant (the kernel + MMU), that each worker has an enormous private desk.** In truth the real desk is tiny and shared. The assistant keeps only the papers you're *actively using* on the real desk, stashes the rest in a filing cabinet (disk) or overflow room (swap), and maintains an **index** translating "the drawer you *think* a paper is in" to "where it physically is right now." The worker never knows the difference — they just ask for a paper by its imagined location, and it appears.

Why go to all this trouble? Three payoffs that justify the entire machine:

- **Isolation & safety** — because each process has its *own* private map, process A literally *cannot* name an address in process B's memory. The wall between user-space programs (Lesson 6) is built largely from virtual memory. One crashing program can't scribble on another's data.
- **The illusion of abundance** — a process can be promised far more memory than physically exists, because most promises are never fully cashed in (Section 2).
- **Flexibility** — the same physical page of RAM can appear at different virtual addresses in different processes (sharing, Section 3), or be backed by a file on disk (the page cache, Section 5), or be temporarily parked on disk (swap, Section 6). All invisible to the process.

Hold this image. Every term below — resident, shared, swap, page cache, page fault — is just a detail of *how the assistant manages the gap between the tiny real desk and the vast imagined one.*

---

## 1. Pages, page faults, and the laziness that makes it all work

The assistant doesn't shuffle papers one word at a time — that'd be hopeless overhead. Memory is managed in fixed-size chunks called **pages**, almost always **4 KB** each. RAM is divided into page-sized **frames**; virtual address spaces into pages; and the kernel's **page tables** are the index mapping each virtual page to either a physical frame, a location on disk, or *nothing yet*. (The MMU caches recent translations in a small fast table called the **TLB**, but you rarely need to think about that.)

### Demand paging: nothing is real until you touch it

Here's the laziness that makes the "illusion of abundance" actually work, and it surprises people: **when a process asks for memory, it doesn't get physical RAM — it gets a *promise*** (an entry in the page table marked "this page exists but isn't backed by anything yet"). Physical RAM is assigned only at the moment the process *first actually touches* (reads or writes) that page. This is **demand paging**: pages are materialized on demand, not on request.

So a program can `malloc` a gigabyte and the system shrugs — no RAM moves. Only as the program writes into that gigabyte, page by page, does real memory get consumed. The worker can *claim* a huge desk; the assistant only clears actual desk space for the sheets the worker physically picks up.

### Page faults: the assistant springs into action

When a process touches a page that *isn't* currently sitting in a physical frame, the MMU can't translate the address, so it raises a **page fault** — an interrupt that hands control to the kernel to go "make this page real." Page faults come in two flavors, and the distinction is *the* key to memory performance:

- **Minor (soft) fault** — the page can be satisfied *without touching disk*: it's a brand-new blank page (just hand over a zeroed frame), or the data is already in RAM (e.g. in the page cache, Section 5) and just needs mapping into this process. Fast — microseconds. Happens constantly and harmlessly.
- **Major (hard) fault** — the page's data must be **read in from disk**: from the program's file on disk, a memory-mapped file, or (the painful one) from **swap**. Slow — *thousands* of times slower than a minor fault, because it waits on storage I/O. **A storm of major faults is the signature of a memory-pressured system thrashing**, and you'll learn to spot it in Section 8.

> **Why this matters for everything downstream:** "using memory" really means "having pages resident in physical frames." Memory *pressure* is the contest over a finite number of frames. *Reclaim* (Section 7) is the kernel taking frames back. *Swap* is parking a frame's contents on disk. *OOM* is what happens when there are no frames to give and nothing left to reclaim. Every concept is about the flow of pages in and out of those physical frames.

---

## 2. VIRT vs. RES — finally explained (and why VIRT is "fictional")

Now Lesson 7's mystery columns make sense. Both measure a process's memory, but they measure *completely different things*, and confusing them causes endless false alarms.

**VIRT (virtual size)** is the total size of the process's *virtual address space* — **everything it has mapped or been promised**, whether or not any of it is actually in RAM. This includes:

- memory it allocated but hasn't touched yet (demand-paged promises, Section 1),
- the program's code and every shared library mapped in (even pages never executed),
- memory-mapped files (which may be entirely on disk),
- and more.

This is why **VIRT is often absurdly large and largely meaningless as a "how much RAM is this using" number** — it's the size of the *imagined desk*, including every drawer the worker laid claim to but never opened. A process showing 20 GB VIRT on an 8 GB machine isn't a bug or a leak; most of that is promises and shared mappings, not real RAM.

**RES (resident set size, RSS)** is the part that's **actually resident in physical RAM right now** — the pages currently sitting in real frames. This is the honest "how much real memory is this process occupying" number, the sheets actually spread on the real desk. **When you hunt a memory hog, RES is the number you watch** (and `top`'s `M` key sorts by it). It can only ever be as large as physical RAM.

The relationship: **RES ≤ VIRT, always**, and usually RES is *far* smaller — because most of the virtual space is unmaterialized promises and on-disk backing. Watching VIRT to gauge memory use is like judging how crowded a desk is by counting the drawers the worker *might* someday use. Watch RES.

> **The overcommit insight:** because of demand paging, Linux happily lets the *sum of all processes' VIRT* exceed physical RAM many times over — this is **memory overcommit**, and it's deliberate. It works because processes almost never touch everything they map. The gamble usually pays (most promises go uncashed) — but when too many promises *are* cashed simultaneously and real frames run out, that's precisely the cliff the OOM killer guards (Section 9). Overcommit is why a Linux box runs fine "over budget" right up until, suddenly, it doesn't.

---

## 3. SHR and the double-counting trap (why summing RSS lies)

The third column, **SHR (shared)**, is the part of a process's resident memory that is **shareable with other processes** — and it exposes a subtle measurement problem worth understanding.

Many pages in RAM are *physically shared* between processes. The classic case: **shared libraries.** Dozens of programs all use the C standard library (`libc`); rather than load a separate copy into each, the kernel loads it into physical RAM *once* and maps that *same* physical page into every process's virtual address space (each at whatever virtual address it likes — the page-table indirection from Section 1 makes this trivial). One physical copy, many virtual references. Likewise, when a process `fork`s (Lesson 7), parent and child initially *share* all their pages via **copy-on-write** — the shared page is only duplicated if one of them writes to it.

This creates a **counting trap**: a shared page is counted in the RES of *every* process that maps it. So if you naïvely **sum RES across all processes, you massively overcount** — you count that one physical copy of `libc` dozens of times. Ten cooks all consulting the *same* communal reference book on the shared shelf; if each reports "I'm using one reference book," the total claims ten books exist when there's physically one.

The honest per-process metric is **PSS (Proportional Set Size)**: it counts a process's *private* pages fully, but divides each *shared* page's size by the number of processes sharing it. Sum PSS across processes and you get a figure that actually adds up to real RAM. The tool that shows PSS:

```
smem -tk
```

`smem` reports USS (unique/private), PSS (proportional), and RSS per process, sorted, with a sane total. When you need to answer "*really*, how is my RAM divided among processes?" — especially with many instances of the same program — `smem`'s PSS is the truthful answer where summed RSS lies. (It usually needs installing.)

> **The takeaway:** RES is honest *per process* but dishonest *in aggregate* because of sharing. This is why "I added up everyone's memory and it exceeds my RAM" is a non-paradox — you double-counted the shared pages. Per-process: watch RES. System-wide truth: think PSS, and trust the system-level gauges (Section 8) over hand-summed process columns.

---

## 4. The great dichotomy: anonymous vs. file-backed memory

This is the conceptual hinge of the whole lesson — the distinction that explains *why swap exists, why the page cache can be dropped for free, and how the kernel decides what to reclaim.* Every page in RAM is one of two kinds, based on a single question: **"if the kernel needs this frame back, where can the data go?"**

**File-backed pages** have a home on disk already — they're a copy of something in a file. This includes program code (backed by the executable), memory-mapped files, and the page cache (Section 5). The key property: **if the kernel needs the frame, a file-backed page that's unmodified (clean) can simply be *dropped* — discarded instantly, for free** — because an identical copy is safe on disk and can be re-read later if needed. (If it's been *modified* — "dirty" — the kernel must first *write it back* to its file, then it can drop it.) These pages don't need swap; their backing store is the original file.

**Anonymous pages** have *no* file backing — they're a process's working data: its heap (what `malloc` returns), its stack, runtime computation. The data exists *only* in RAM; there's no file to re-read it from. So the key property: **if the kernel needs an anonymous page's frame, it can't just drop it — there'd be no way to ever get the data back.** Its *only* options are to keep it in RAM, or to write it somewhere it can be retrieved — and that "somewhere" is **swap.** Anonymous memory is the *reason swap exists.*

The analogy snaps it into place:

- **File-backed pages** = photocopies of documents whose originals are filed in the cabinet. Need the desk space? Toss the photocopy — the original's safe; re-copy it later if needed. (If you scribbled notes on the copy first, file those notes before tossing.)
- **Anonymous pages** = the worker's *own handwritten notes* with no copy anywhere. Toss them and the work is *gone forever.* To free that desk space without losing the work, you must first stash the notes in the overflow room (swap).

> **Why this is the hinge:** the kernel's reclaim strategy (Section 7) and the entire role of swap follow directly. Clean file-backed pages are *cheap* to reclaim (drop). Dirty file-backed pages are *medium* (write back, then drop). Anonymous pages are *expensive* (must swap out). And the page cache (next) is just "lots of clean file-backed pages the kernel keeps around because it has the room" — which is exactly why it can be surrendered instantly and shouldn't scare you.

---

## 5. The page cache: why "free" memory is the wrong number

Now the single most misunderstood thing about Linux memory — the source of a thousand "something is eating all my RAM!" panics that are completely unfounded.

**Linux uses otherwise-idle RAM as a cache of file data.** Every time a file is read from disk, the kernel keeps those pages in RAM (file-backed, clean — Section 4) in case they're needed again, because reading from RAM is ~100,000× faster than from disk. Over time, this **page cache** grows to fill *most* of your "free" memory. This is not waste — **it is the kernel doing its job.** Empty RAM is wasted RAM; RAM full of cached file data is RAM earning its keep, and *every page of it can be instantly surrendered* the moment a process needs real memory (drop the clean photocopies).

This is why the naïve reading of memory is *wrong*. Run `free -h` and you see:

```
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.2Gi       0.5Gi       180Mi        10Gi        10Gi
```

The beginner panics: "only 0.5 GiB free!" **That's the wrong column.** That 10 GiB of `buff/cache` is page cache — available the instant anything needs it. The number that actually matters is the last one:

- **`free`** — RAM that is *completely unused* right now. On a healthy, busy Linux system this is *supposed* to be small, because idle RAM has been put to work as cache. **Low `free` is normal and good.**
- **`buff/cache`** — RAM used for the page cache (and buffers). Reclaimable on demand. *Not* a problem.
- **`available`** — **the number that matters.** The kernel's own estimate of how much memory is *actually available to start new programs without swapping* — i.e. truly free RAM *plus* the page cache it can reclaim. **This is your real "how much memory do I have left" figure.** Watch `available`, ignore `free`.

> **Burn this in:** *"free" memory being near zero is not a problem — it's the design.* The page cache expands to use the slack and contracts the instant it's needed. The myth "Linux ate my RAM" comes entirely from reading `free` instead of `available`. The authoritative source is `MemAvailable` in `/proc/meminfo`; `free -h`'s `available` column is just that value made readable.

---

## 6. Swap: the pressure valve (and what's *actually* bad)

Swap is the most maligned, misunderstood feature in this lesson. Let's rehabilitate it precisely.

**Swap is disk space the kernel can use to park anonymous pages (Section 4) that don't fit in RAM.** When physical frames run short, the kernel can take a process's least-recently-used anonymous pages, **write them out to swap** (the overflow room down the hall), and reuse those frames for something more urgent. If the process later touches those pages, it takes a **major page fault** (Section 1) to read them back in — slow, but the data wasn't lost. Swap can be a dedicated partition or a swap file; modern Ubuntu often uses a swap file, and many systems use **zram** (a *compressed* swap that lives in RAM itself — trading CPU to effectively fit more) or **zswap** (a compressed RAM cache in front of disk swap).

The crucial reframing: **swap existing and being lightly used is healthy, not a problem.** It lets the kernel evict genuinely cold anonymous pages (stuff allocated long ago and never touched again) to make room for hot data and more page cache. A few hundred MB sitting in swap, untouched, is the system being *smart* — parking junk it isn't using.

**What's actually bad is *thrashing*** — sustained, heavy swapping *in and out* continuously, where the working set genuinely doesn't fit in RAM, so the kernel is constantly evicting pages it immediately needs back. Now every access is a major fault waiting on disk; the CPU sits idle (high `wa`, Lessons 3/7) while the disk screams; throughput collapses. The machine feels frozen though "nothing" is using CPU. **The signal isn't "swap is used" — it's "swap is *actively moving*"**, which you read as `si`/`so` in `vmstat` (Section 8). Cold pages resting in swap = fine. Pages stampeding through swap = emergency.

The knob worth knowing:

- **`vm.swappiness`** (0–100, default ~60) — the kernel's *relative eagerness* to reclaim by swapping anonymous pages versus dropping page cache. **Lower** = "prefer to shrink the file cache before touching swap"; **higher** = "more willing to swap out idle anonymous pages." It's a *balance dial between the two reclaim sources from Section 4*, not an on/off switch. Tuning it (e.g. lower on a latency-sensitive database, higher on a cache-light batch box) is a real lever, but understand it as bias, not prohibition.

> **The mental correction:** swap is a *pressure-relief valve*, not a failure. Its presence buys the kernel room to maneuver and often *delays* or *prevents* an OOM kill. The disease isn't swap usage; it's swap *churn*. Judge swap by its flow rate, not its fill level.

---

## 7. Reclaim: how the kernel finds memory under pressure

Tie Sections 4–6 together with the mechanism that uses them. When free frames run low, the kernel doesn't wait for zero — it proactively **reclaims** memory, walking through the cheapest options first. This is the assistant clearing desk space *before* the worker is completely stuck.

The escalating reclaim toolkit, cheapest to most disruptive:

1. **Drop clean page-cache pages** (Section 5) — nearly free, no I/O. The first and preferred move: surrender cached file data nobody urgently needs.
2. **Write back dirty file-backed pages, then drop them** — costs a disk write, but recoverable. (A background thread does this continuously even without pressure, so dirty pages don't pile up.)
3. **Swap out anonymous pages** (Section 6) — costs a disk write to swap; only way to reclaim anonymous memory. Governed by `swappiness` relative to option 1.

There are two *modes* in which this happens, and the difference is what you *feel*:

- **Background reclaim (`kswapd`)** — a kernel thread that wakes when free memory dips below a low watermark and reclaims *gently in the background*, ahead of demand. Invisible; the system stays responsive. This is the healthy steady state under moderate pressure.
- **Direct reclaim** — when memory is needed *faster than `kswapd` can supply it*, the allocating process is **forced to do reclaim work itself, stalling mid-allocation** until it frees enough. *This* is when users feel the lurch — latency spikes, momentary freezes. Frequent direct reclaim means pressure has outrun the background machinery: a serious warning sign, and one the PSI metric (Section 8) was invented to expose.

If reclaim *still* can't find enough — cache is shrunk to the bone, swap is full or absent, and a process *demands* a frame that cannot be produced — the kernel has exhausted every option. That's the cliff edge, and over it lies the OOM killer.

---

## 8. Reading memory pressure *before* the OOM killer fires

This is the practical heart of the lesson — the skill you came for. The OOM killer (Lesson 6) is the *failure* endpoint; the goal is to **see the pressure rising and act while you still can.** Memory pressure escalates through recognizable stages, each with a gauge. Learn to read the ladder.

### The escalation ladder (what's happening, and what to watch)

1. **Healthy** — plenty of `available`, page cache large, swap quiet. Watch: `available` in `free -h`.
2. **Cache shrinking** — `available` falling, `buff/cache` being squeezed as the page cache gives ground to growing process memory. Still fine, but trending.
3. **Swapping begins** — anonymous pages start moving to swap. Watch: `si`/`so` in `vmstat` going nonzero and *staying* nonzero.
4. **Thrashing / direct reclaim** — sustained heavy `si`/`so`, high `wa`, processes stalling. Watch: **PSI** rising (below). This is the red zone.
5. **OOM kill** — reclaim fails; kernel kills a process. Watch: `dmesg`/`journalctl -k` (Lesson 6).

### The gauges, in order of usefulness

**`free -h`** — your quick pulse. **Watch `available`, not `free`** (Section 5). Falling `available` with rising swap usage is pressure building.

**`/proc/meminfo`** — the authoritative source behind `free`. The key fields: **`MemAvailable`** (the real headroom), `SwapFree`, `Dirty` (pages awaiting writeback), and the breakdown of `Active`/`Inactive` (the kernel's hot/cold page lists that guide reclaim).

**`vmstat 2`** — the *flow* gauge, and the classic thrash detector. Refreshing every 2s, the columns that matter:

- **`si` / `so`** — swap-**in** and swap-**out**, in KB/s. **This is the thrashing alarm.** Occasional blips are fine; *sustained* nonzero `si`/`so` means the kernel is actively churning pages through swap — the machine is in trouble *right now*, often before users have even finished complaining.
- **`free` / `buff` / `cache`** — the same memory split, trending over time.
- **`b`** — processes blocked (often on I/O, including swap), and **`wa`** (I/O wait) climbing alongside `si`/`so` confirms swap is the bottleneck.

**PSI — Pressure Stall Information** (the modern, best early-warning signal):

```
cat /proc/pressure/memory
```

This is the gauge worth knowing about, because it measures *exactly the thing you care about*: **how much time tasks are stalled waiting for memory.** It reports `some` (at least one task stalled) and `full` (everything stalled) as percentages over the last 10, 60, and 300 seconds. Unlike "how much swap is used" (a level), PSI measures *pain* (a rate of stalling) — so it rises the moment direct reclaim (Section 7) starts hurting, often *before* an OOM kill becomes inevitable. **Rising `some avg10` for memory is your early-warning klaxon**: the system is spending real time blocked on memory, act now. (The same `/proc/pressure/` exists for `cpu` and `io` — PSI is a general pressure framework, memory being its most valuable use.)

**`top` / `htop`** — sort by memory (`M`, Lesson 7) to find *which* process is driving the growth, so you can act on the cause (restart it, fix the leak, add a cgroup limit).

**`sar -r` / history tools** — for *after the fact*: `sar` (from `sysstat`) records memory metrics over time, letting you see the slope leading up to an incident — "memory climbed steadily from 2pm until the OOM at 4:13."

> **The actionable instinct:** don't wait for the OOM message — by then you've already lost a process. Watch **`available`** trending down (`free`/`meminfo`), catch **sustained `si`/`so`** (`vmstat`), and treat **rising memory PSI** as the alarm that pressure has turned into *stalling*. Each is an earlier, gentler warning than the last. Then use `top -M`/`smem` to find the culprit and intervene — restart the offender, tune the workload, or impose a cgroup limit — *before* the kernel makes the choice for you.

---

## 9. The OOM killer, revisited with full context

Lesson 6 introduced the OOM killer as "the kernel kills a process to survive." Now you can see *exactly* when and why it fires, and how to influence it.

It fires only at the cliff edge from Section 7: a process demands a page, **reclaim has exhausted every option** (cache squeezed, swap full or absent), and the allocation *cannot* be satisfied. Rather than let the whole system deadlock, the kernel picks a victim and kills it to free its entire resident set at once. The choice isn't random — each process has an **`oom_score`** (roughly, the more memory it uses, the higher its score, the more likely it's chosen — killing a big hog frees the most), tunable per-process via **`oom_score_adj`** (−1000 to +1000): lower it to *protect* a critical process (your database) and raise it to make a process the *preferred* sacrifice. You'll find the post-mortem in `dmesg`/`journalctl -k`: `Out of memory: Killed process 1234 (mysqld)` plus a table of who was using what.

The far better tool than after-the-fact scoring is **containment via cgroups** — and here the whole series closes a loop. Recall from [Lesson 4](04-systemd-services.md) that **a service *is* a cgroup**, and cgroups can impose hard memory limits:

- **`MemoryMax`** (cgroup v2 `memory.max`) — a hard ceiling; the cgroup can't exceed it. If it tries, reclaim is forced *within that cgroup*, and if that fails, the OOM killer fires **scoped to just that cgroup** — killing the offending service's process instead of letting it take down the whole machine. Blast-radius containment.
- **`MemoryHigh`** (`memory.high`) — a *soft* limit: the kernel throttles and aggressively reclaims the cgroup above this point *without* killing, applying back-pressure to slow a leak gracefully. A gentler tripwire before the hard `Max`.
- **`memory.pressure`** — per-cgroup PSI (Section 8), so you can read pressure *per service*, not just system-wide.

Set on a known-greedy service (`sudo systemctl edit myapp` → `[Service]` → `MemoryMax=2G`, recalling Lesson 4's drop-in overrides), these turn "a memory leak somewhere takes down the entire box at 4am" into "the leaky service is contained, throttled, and at worst restarted in isolation while everything else stays up."

> **The full arc:** overcommit (Section 2) lets the system promise more than it has; reclaim (Section 7) manages the gap; PSI (Section 8) warns when the gap turns to pain; and the OOM killer is the last resort when the gap becomes a chasm. cgroup limits let you *bound the chasm* to one service. You now understand the entire pipeline from `malloc` to the kill — and, more importantly, where to step in along the way.

---

## 10. The command cheatsheet

Grouped by intent.

### Quick memory pulse

| Command | What it tells you |
|---------|-------------------|
| `free -h` | RAM & swap; **watch `available`, not `free`** |
| `cat /proc/meminfo` | Authoritative source: `MemAvailable`, `SwapFree`, `Dirty`, Active/Inactive |
| `uptime` | Not memory, but load context alongside it |

### Watch pressure / flow over time

| Command | What it tells you |
|---------|-------------------|
| `vmstat 2` | **`si`/`so` = the thrashing alarm**; plus free/cache/blocked trends |
| `cat /proc/pressure/memory` | **PSI — the early-warning klaxon** (`some`/`full`, avg10/60/300) |
| `sar -r 1` | Memory metrics (history if logging enabled) |
| `watch -n2 free -h` | Poor-man's live `available` trend |

### Find which process is using memory

| Command | What it tells you |
|---------|-------------------|
| `top` then `M` | Live, sorted by resident memory (Lesson 7) |
| `htop` (sort by MEM) | Friendlier version |
| `smem -tk` | **PSS** — honest per-process share, no double-counting shared pages |
| `ps aux --sort=-%mem \| head` | Snapshot of top memory users |

### Inspect one process in detail

| Command | What it tells you |
|---------|-------------------|
| `cat /proc/1234/status` | VmSize (VIRT), VmRSS (RES), swap used, etc. |
| `cat /proc/1234/smaps_rollup` | Per-process PSS/USS rollup |
| `pmap -x 1234` | Memory map: what regions, file-backed vs anonymous |

### Swap & tuning

| Command | What it does |
|---------|-------------|
| `swapon --show` | What swap devices/files exist and their use |
| `cat /proc/sys/vm/swappiness` | Current swappiness bias |
| `sudo sysctl vm.swappiness=10` | Bias reclaim away from swap (temporary) |
| `dmesg \| grep -i "out of memory"` | OOM-kill post-mortems (Lesson 6) |

### Contain a service (cgroup limits)

| Command | What it does |
|---------|-------------|
| `systemctl show svc -p MemoryCurrent,MemoryMax` | A service's current vs. max memory |
| `sudo systemctl edit svc` → `MemoryMax=2G` | Impose a hard ceiling (Lesson 4 drop-in) |
| `cat /sys/fs/cgroup/.../memory.pressure` | Per-cgroup PSI |

---

## 11. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **virtual memory** | The per-process illusion of a vast private address space | Each worker's imagined giant desk |
| **physical RAM** | The actual, small, shared fast memory | The real, tiny shared desk |
| **MMU / page table** | Hardware+kernel that translates virtual→physical | The assistant and their index |
| **page** | Fixed 4 KB unit of memory management | A standard sheet/folder |
| **demand paging** | Pages become real only when first touched | Clearing desk space only for sheets picked up |
| **page fault (minor/major)** | Touching an unmapped page; minor=RAM, major=disk | Asking for a sheet that's on the desk vs. in storage |
| **VIRT** | Total virtual space mapped (mostly promises) | Drawers claimed, including unopened ones |
| **RES / RSS** | Pages actually resident in RAM — *the honest number* | Sheets actually on the desk |
| **SHR / shared** | Resident pages shareable across processes | The communal reference book |
| **PSS** | Per-process memory counting shared pages proportionally | Your fair share of the communal book |
| **overcommit** | Promising more virtual memory than physical RAM | Letting workers claim more desk than exists |
| **anonymous memory** | Working data with no file backing (heap/stack) | Handwritten notes with no copy |
| **file-backed memory** | Pages copied from a file on disk | Photocopies of filed originals |
| **page cache** | Idle RAM used to cache file data; instantly reclaimable | Photocopies kept on spare desk space |
| **`free` vs `available`** | Unused RAM vs. *truly usable* RAM (incl. reclaimable cache) | Empty desk vs. desk you can clear on demand |
| **swap** | Disk space to park anonymous pages under pressure | The overflow storage room down the hall |
| **thrashing** | Sustained heavy swap in/out; the real disease | Endlessly shuttling papers to/from overflow |
| **swappiness** | Bias dial: swap anon vs. drop cache | Which to clear first when the desk fills |
| **reclaim (kswapd / direct)** | Freeing frames in background vs. stalling the process | Tidying ahead of time vs. stopping to tidy mid-task |
| **PSI** | % of time tasks stall waiting for a resource | A pain meter, not a fill gauge |
| **OOM killer** | Kernel kills a process when reclaim fails | Firing a worker to free a whole desk |
| **`oom_score_adj`** | Per-process bias for who gets killed | Marking who's first/last to be let go |
| **`MemoryMax` (cgroup)** | Hard per-service memory ceiling | A fixed desk allotment per team |

---

## 12. Where to go next

You now understand memory from the `malloc` promise to the OOM kill, and how to read pressure at every stage. Natural continuations:

- **cgroups v2 deep-dive** — the unified resource-control hierarchy behind `MemoryMax`/`MemoryHigh`/`memory.pressure` (and CPU/IO limits), tying Lessons 4, 7, and 8 together into one mechanism.
- **`/proc` and `/sys` as the source of truth** — every tool here just reads these; learning to read them directly (`/proc/meminfo`, `/proc/<pid>/smaps`, `/sys/fs/cgroup/...`) makes you independent of any particular tool.
- **`sysctl` and VM tuning** — `vm.swappiness`, `vm.dirty_ratio`, overcommit policy (`vm.overcommit_memory`): the knobs that change reclaim and overcommit behavior, now that you know what they govern.
- **Performance tooling** — `perf`, `bpftrace`, and PSI-driven tools (like `oomd`/`systemd-oomd`, which act on PSI to kill *gracefully* before the kernel's blunt OOM killer must).

The instinct, now at the deepest layer of the resource stack: **memory is a flow of pages between a small real desk and a vast imagined one — so don't watch how *full* it looks, watch how hard the kernel is *working to keep up*, and step in before that work turns into a kill.**
