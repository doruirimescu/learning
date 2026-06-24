# High-Frequency Trading for Software Engineers — A Lesson Plan

> A concept-first roadmap for a software engineer who wants to break into **high-frequency trading (HFT)**. This is the *plan*, not the lessons themselves: an ordered arc of modules, each one explaining **what you need to learn, why it matters in HFT specifically, and the mental model to hold** — so that when you sit down to study (or when these get written out as full lessons), you know exactly what you're building toward and how the pieces interlock.
>
> The guiding principle, borrowed from the rest of this repo: lead with *why* and with mental models, not with command dumps and API lists. In HFT more than anywhere, **you cannot optimize a machine you don't understand.**

---

## 0. The one idea everything hangs on: latency is the product

Before any module, internalize the thing that makes HFT *different* from all other software engineering. In a normal backend, **throughput** is king — you want to serve a million requests, and a request taking 50 ms instead of 5 ms is invisible to users. In HFT, you often process a *modest* number of messages, but the **time from "a packet carrying market data hits your network card" to "your order packet leaves the network card" is the entire game** — and it is measured in **nanoseconds**, not milliseconds.

> **The mental model: HFT is a race, and the track is a wire.** Two firms see the same price change at the same instant. Whoever's order reaches the exchange's matching engine *first* gets the trade; the loser gets nothing, or worse, gets picked off. There is no second place. A single cache miss (~100 ns), one branch mispredict (~15 ns), one trip through the kernel's network stack (~microseconds), one page fault, one lock contention — any of these can be the difference between profit and loss. **Everything in this curriculum is, ultimately, about removing nanoseconds from that path.**

This reframes how you read every later module. When you study CPU caches, you're not learning trivia — you're learning *where your 100 ns went*. When you study the NIC, you're learning how to skip the kernel entirely. The word the industry uses for this — designing software that respects and exploits how the hardware actually behaves — is **mechanical sympathy** (a term borrowed from race-car driving: the driver who understands the engine drives it faster). That phrase is the spine of this whole plan.

A second idea, equally important and often a surprise to newcomers: **in HFT, predictability beats raw speed.** A path that is *usually* 1 µs but *occasionally* 50 µs (a "tail latency" spike, called **jitter**) is often worse than a path that is always 2 µs. The rare slow case is exactly the moment the market is moving and you most need to be fast. So a recurring theme is not just "make it fast" but "make it **deterministic** — eliminate the sources of variance" (allocations, page faults, context switches, cache eviction, GC pauses in other languages). Hold both ideas: **minimize latency, and minimize its variance.**

---

## How to read this plan — the learning arc

The modules are ordered as a *dependency chain*, bottom-up: you understand the machine, then the language that drives it, then the hardware around it, then the OS taming it, then the network connecting it, then the techniques that bypass the slow parts, then the practices that make it a business. You *can* skip around, but each module assumes the ones before it.

| # | Module | One-line purpose |
| --- | --- | --- |
| 1 | **Computer Architecture** | Understand the machine you're racing on — caches, pipelines, why "the CPU is not what you think." |
| 2 | **Modern Assembly** | See what your C++ actually becomes; learn to read the compiler's output and reason about instruction cost. |
| 3 | **Hardware** | NICs, kernel-bypass cards, FPGAs, PCIe, clocks — the physical machinery of a trading server. |
| 4 | **OS / Linux Fundamentals** | Tame the operating system: isolate cores, kill jitter, pin everything, make the box deterministic. |
| 5 | **Networking** | TCP/IP, UDP multicast, market-data protocols — where the data comes from and where latency hides in the stack. |
| 6 | **Kernel Bypass** | Skip the OS network stack entirely — DPDK, Onload, RDMA, AF_XDP — the single biggest latency win. |
| 7 | **C++ Data Structures** | The STL containers — array, vector, list, ordered/unordered map & set: complexity, what each can and can't do, what to use when, and how to make them fast. |
| 8 | **C++ for HFT** | The language of the industry, mapped onto every concept above: cache-friendly code, templates, zero-allocation hot paths, lock-free. |
| 9 | **Industry Practices & Standards** | FIX, market data, exchanges, risk controls, testing, regulation — how it's actually built and run. |

A reasonable study order is exactly 1 → 9. If you're impatient to write code, you can pull the C++ pair (Modules 7–8) forward and learn it *alongside* 1–6, since every C++ technique is a direct response to a hardware fact you'll be learning. But don't start the C++ modules *before* at least Module 1 — you'd be memorizing rules ("avoid `virtual` in the hot path," "don't use `std::map` in the hot loop") without understanding the *why* (the indirect-call branch mispredict, the cache-missing pointer chase) that lets you reason about new cases yourself.

---

## Module 1 — Computer Architecture: the machine you're actually racing on

> 📖 **Written:** [`01-computer-architecture.md`](01-computer-architecture.md) — the full lesson.

**Why this is first.** Every later optimization is a reaction to a fact about the hardware. If you don't know the cache hierarchy, "make it cache-friendly" is a cargo-cult incantation. This module builds the mental model of the modern CPU as it *really* is — wildly parallel, deeply speculative, and dominated by the cost of *moving data*, not the cost of *computing on it*.

> **The mental model: the CPU is a tiny, blazingly-fast workshop fed by a long, slow supply chain.** Doing arithmetic is nearly free; *getting the data to do arithmetic on* is the expense. A modern core can do dozens of operations in the time it takes to fetch one number from main memory. So performance is overwhelmingly a story about **the memory hierarchy** — keeping the data the core needs *close*.

Concepts to master, each with its HFT payoff:

- **The memory hierarchy and cache lines.** Registers → L1 (~1 ns) → L2 (~4 ns) → L3 (~15 ns) → main memory (~100 ns). Memory moves in **64-byte cache lines**, never single bytes. *Payoff:* this single fact dictates how you lay out every data structure — struct-of-arrays vs array-of-structs, padding, and why a linked list is poison and a contiguous array is gold in the hot path.
- **Cache misses, spatial & temporal locality, prefetching.** Why sequential access is an order of magnitude faster than random; how the hardware prefetcher rewards predictable strides. *Payoff:* the core reason HFT code favors flat arrays and avoids pointer-chasing.
- **The instruction pipeline, superscalar execution, and instruction-level parallelism (ILP).** The CPU isn't doing one instruction at a time — it's juggling dozens in flight, out of order. *Payoff:* explains why micro-optimizing instruction counts is often pointless while *dependencies* between instructions (which serialize the pipeline) are what actually cost you.
- **Branch prediction and misprediction.** The CPU *guesses* which way an `if` goes and speculatively runs ahead; a wrong guess costs ~15–20 cycles to unwind. *Payoff:* why HFT code is written to be **branch-predictable** (or branchless), and why unpredictable data-dependent branches in the hot path are a real cost you'll learn to measure and remove.
- **The TLB, virtual memory, and huge pages.** Address translation has its own cache (the TLB); a miss is expensive. *Payoff:* why HFT systems use **huge pages** to cover their working set with fewer TLB entries (links forward to Module 4).
- **NUMA (Non-Uniform Memory Access).** On multi-socket servers, memory attached to *your* socket is faster than memory attached to the other. *Payoff:* why you pin a thread *and* its memory to the same NUMA node, and why crossing nodes is a silent latency tax.
- **Cache coherence, false sharing, and the cost of sharing across cores.** When two cores write variables that happen to live on the *same cache line*, they ping-pong the line between caches even though they never touch the same data — **false sharing**, a classic latency killer. *Payoff:* directly motivates cache-line padding in concurrent data structures (Module 8).
- **Simultaneous multithreading (hyper-threading), and why HFT usually turns it off.** Two logical threads sharing one physical core's resources adds variance. *Payoff:* a real, common BIOS-level tuning decision.

**What mastery looks like:** you can look at a data structure and a hot-path access pattern and *predict* roughly how many cache lines it touches and where the misses are — before measuring.

---

## Module 2 — Modern Assembly: seeing what your code becomes

> 📖 **Written:** [`02-modern-assembly.md`](02-modern-assembly.md) — the full lesson.

**Why now.** You don't write HFT systems in assembly — you write them in C++. But you must be able to **read** the assembly the compiler emits, because that's the ground truth of what the machine does, and because the gap between "what I wrote" and "what got compiled" is where latency hides and where myths die. This module is about *reading and reasoning*, not hand-writing.

> **The mental model: assembly is the receipt for your C++.** You hand the compiler a high-level intent; it hands the CPU a list of instructions. To know whether your "optimization" did anything, you read the receipt. Tools like **Compiler Explorer (godbolt.org)** make this a daily habit, not an arcane skill.

Concepts to master:

- **x86-64 basics: registers, the instruction format, the calling convention (System V ABI).** Enough to follow a function: what's in registers, what's on the stack, how arguments and return values flow. (ARM/AArch64 matters too as it grows in servers, but x86-64 is the HFT default.)
- **Reading compiler output.** Mapping a C++ loop or struct access to its instructions; recognizing when the compiler **inlined**, **vectorized**, **unrolled**, or **elided** something — and when it *didn't* and why (e.g. an aliasing concern blocked it).
- **Instruction latency vs. throughput, and dependency chains.** Each instruction has a latency (cycles until its result is ready) and a throughput (how many can issue per cycle). The real cost of a code sequence is its **critical dependency path**, not its instruction count. *Payoff:* this is *the* lens for micro-optimization, and it ties back to Module 1's pipeline.
- **SIMD / vectorization (SSE, AVX, AVX-512).** One instruction operating on 4/8/16 values at once. *Payoff:* used in HFT for parallel parsing, checksums, bulk comparisons; you'll learn when the compiler auto-vectorizes and when you reach for intrinsics.
- **Memory ordering and atomics at the instruction level.** What `LOCK`-prefixed instructions and fences actually cost; the x86 memory model. *Payoff:* the hardware reality underneath C++'s `std::atomic` and memory-order parameters (Module 8).
- **`rdtsc` / `rdtscp` and cycle-accurate timing.** How HFT code measures itself at nanosecond resolution, and the pitfalls (out-of-order execution, frequency scaling) that make naive timing lie. *Payoff:* you can't optimize what you can't measure, and milliseconds-granularity tools are useless here.

**What mastery looks like:** given a small hot function, you can pull it up in Compiler Explorer, read the emitted assembly, identify the dependency chain and any surprise (a hidden call, a missed vectorization, a spurious load), and explain the per-iteration cost in cycles.

---

## Module 3 — Hardware: the physical machinery of a trading server

> 📖 **Written:** [`03-hardware.md`](03-hardware.md) — the full lesson.

**Why now.** With the CPU understood (Modules 1–2), zoom out to the *box* and the *wire*. HFT runs on specialized, carefully chosen hardware, and the **network card is the single most important component** because it's the first and last thing to touch every packet in the race.

> **The mental model: the NIC is the starting gate and the finish line.** The race clock starts when a market-data packet arrives at your NIC and stops when your order packet departs it. Everything in software lives *between* those two NIC events. So the NIC — and how directly your software can talk to it — disproportionately determines who wins.

Concepts to master:

- **The NIC (Network Interface Card) in depth.** How a packet physically arrives: DMA into host memory, interrupts vs. polling, RX/TX queues, ring buffers. Why **interrupts add latency and jitter** and why HFT prefers **busy-polling** (a core spinning, asking "anything yet? anything yet?"). *Payoff:* this is the foundation for kernel bypass (Module 6).
- **Specialized low-latency NICs.** Cards like Solarflare/Xilinx (now AMD) and Mellanox/NVIDIA that offer kernel-bypass libraries and onboard features. What "**kernel-bypass-capable**" means at the hardware level.
- **FPGAs (Field-Programmable Gate Arrays).** The frontier of HFT speed: putting trading logic *in the network card's hardware* so a decision is made in nanoseconds without the CPU ever being involved (**tick-to-trade in hardware**). You don't need to be an FPGA engineer to start, but you must understand *what they're for* and where the software/hardware boundary sits.
- **PCIe (the bus connecting NIC to CPU).** Bandwidth, latency, and why the *topology* (which PCIe slot, attached to which CPU socket — back to NUMA) matters for shaving nanoseconds.
- **Clocks and time synchronization: PTP (Precision Time Protocol), PPS, hardware timestamping.** HFT lives or dies on accurate, sub-microsecond time — for measuring your own latency, for ordering events, and for regulatory timestamping. *Payoff:* you'll learn why a GPS-disciplined clock and NIC hardware timestamps exist.
- **Colocation and the physics of distance.** Firms rent racks *inside* the exchange's data center ("**colo**") because light travels ~1 foot per nanosecond — literal cable length to the matching engine is a competitive variable. Equal-length cabling, microwave/laser links between data centers (faster than fiber because light is slower in glass). *Payoff:* understanding why the *whole system*, not just code, is latency-engineered.
- **Server tuning at the firmware level.** BIOS settings: disabling power-saving C-states and frequency scaling (you want the CPU always at max, never "waking up"), disabling hyper-threading, configuring the right power profile. *Payoff:* a real checklist HFT engineers own.

**What mastery looks like:** you can describe the full physical journey of a market-data packet from the exchange's matching engine to your trading logic and back, naming every component it crosses and where latency accrues.

---

## Module 4 — OS / Linux Fundamentals: taming the operating system

**Why now.** Linux is the universal HFT operating system, but a *stock* Linux box is built for fairness and throughput across many processes — the **opposite** of what HFT wants. This module is about wrestling the OS into a single-minded, deterministic latency machine. Much of it builds directly on the Linux lessons in [`../linux/`](../linux/) (processes, scheduling, memory, NUMA) — read those first if you haven't.

> **The mental model: by default the OS is a democracy optimizing for the common good; HFT needs a dictatorship optimizing for one process.** Every fairness mechanism — the scheduler time-slicing your thread away, an interrupt stealing your core, the kernel migrating your process to a "less busy" CPU — is a source of *jitter*. This module is the art of telling the OS "leave this core alone; it belongs to me, and only me, forever."

Concepts to master:

- **The scheduler, context switches, and why they're the enemy.** What a context switch costs (direct cost + cache/TLB pollution = the *real* cost). Real-time scheduling classes (`SCHED_FIFO`). *Payoff:* the foundation for "never let my hot thread be preempted."
- **CPU pinning / affinity and core isolation (`isolcpus`, `cpuset`, `taskset`).** Reserving cores so the kernel's general workload never runs on them; pinning your busy-polling thread to one isolated core for its entire life. *Payoff:* the single most important Linux tuning step for determinism.
- **IRQ affinity.** Steering hardware interrupts *away* from your hot cores so a random network or disk interrupt can't preempt your trading thread. *Payoff:* removes a major jitter source.
- **Huge pages.** Backing your big buffers with 2 MB/1 GB pages to slash TLB misses (ties back to Module 1). *Payoff:* fewer, cheaper address translations on the hot path.
- **Memory locking and pre-faulting (`mlock`/`mlockall`).** Forcing your pages to be resident and never swapped, and *touching* all your memory at startup so you never take a **page fault** during trading. *Payoff:* eliminates a catastrophic, unpredictable latency spike.
- **Avoiding allocation and syscalls in the hot path.** Why `malloc`, `mmap`, logging, and any system call are banned from the critical path (they can block, lock, or trap into the kernel). *Payoff:* the design constraint that shapes all HFT C++ (Module 8).
- **NUMA control (`numactl`).** Binding memory allocation to the local node (ties back to Module 1).
- **The kernel's role and why we want less of it.** Recognizing every place the kernel intervenes (syscalls, interrupts, scheduling, the network stack) — which sets up *why* kernel bypass (Module 6) is such a big win.
- **Measuring jitter.** Tools and techniques (`cyclictest`, latency histograms) to *prove* your box is deterministic, not just assume it.

**What mastery looks like:** given a fresh Linux server, you can produce and justify a tuning checklist that turns it into a low-jitter trading host, and you can measure the result.

---

## Module 5 — Networking: where the data comes from and where latency hides

**Why now.** With the box tamed, study the *data flow*. HFT is fundamentally a networking problem: you receive **market data** (prices) over the network and send **orders** over the network. This module is about the protocols and the network stack — and crucially, **how much latency the standard stack adds**, which motivates Module 6.

> **The mental model: two different conversations.** Market data is a *broadcast* — the exchange shouts prices to everyone at once, so it uses **UDP multicast** (fast, connectionless, no retransmission, "fire and forget"). Orders are a *private, reliable* conversation with the exchange — they need guaranteed delivery, so they typically ride **TCP** (or a reliable session protocol). Understanding *why each side uses what it uses* explains the whole architecture.

Concepts to master:

- **The TCP/IP stack, layer by layer, and its latency cost.** What the kernel does to every packet: copies, checksums, protocol processing, the journey from NIC to socket to your `recv()`. *Payoff:* this is the latency that kernel bypass (Module 6) deletes — you must understand it to appreciate what you're removing.
- **UDP vs. TCP for HFT.** Why market data is UDP multicast (low latency, broadcast efficiency, you accept occasional loss) and why orders are reliable/TCP. The trade-offs of "no retransmission."
- **Multicast in depth.** How one exchange feed reaches thousands of subscribers; group management; why HFT firms subscribe to raw multicast feeds rather than slower aggregated ones.
- **Handling packet loss in market data: A/B feed arbitration.** Exchanges send the *same* data on two independent multicast feeds; you listen to both and take whichever packet arrives first / fill gaps from the other. *Payoff:* a real, ubiquitous HFT pattern that also *reduces* latency (first-of-two wins).
- **Sequence numbers, gap detection, and recovery (snapshot/retransmit).** How you know you missed a packet and how you recover the order book without falling behind.
- **NIC offloads and their double edge.** Features like TCP segmentation offload can *help* throughput but *hurt* latency or determinism; HFT often disables them. *Payoff:* another tuning decision with a real rationale.
- **Network topology and switches.** Cut-through vs. store-and-forward switches; switch latency as a measured quantity. *Payoff:* the network *between* you and the exchange is itself latency-engineered.

**What mastery looks like:** you can describe how a price update travels from the exchange to your order book, how you'd detect and recover a dropped multicast packet, and you can point to every microsecond the standard kernel stack adds.

---

## Module 6 — Kernel Bypass: skipping the slow path

**Why now.** This is the payoff of Modules 3–5. You've learned the kernel's network stack adds microseconds of latency and jitter (copies, interrupts, context switches, protocol processing). HFT's answer is radical: **don't use it.** Let your application talk to the NIC *directly*, in user space, bypassing the kernel entirely. This is often the **single largest latency win** in the whole stack.

> **The mental model: the kernel is a helpful but slow middleman, and you fire it.** Normally, a packet arrives → NIC interrupts the kernel → kernel processes it through the stack → copies it → wakes your process → you read it. Kernel bypass cuts the middleman out: the NIC delivers packets straight into your user-space memory, and your busy-polling thread reads them with zero syscalls, zero copies, zero context switches. You trade the kernel's *convenience and safety* for raw *speed and determinism*.

Concepts to master:

- **Why bypass works: the costs you're removing.** Eliminating syscalls (no user/kernel transition), zero-copy (NIC DMAs straight to your buffer), no interrupts (you busy-poll), no scheduler involvement. Tie each saving back to a specific cost from Modules 4–5.
- **The major technologies and what distinguishes them:**
  - **DPDK (Data Plane Development Kit)** — a full user-space packet-processing framework; you implement (or use a library for) the network stack yourself. Maximum control, maximum work.
  - **Solarflare Onload (AMD) / ef_vi** — a kernel-bypass stack that transparently accelerates standard sockets (Onload) or gives a raw low-level API (ef_vi). Popular in HFT for its "just works with sockets" path plus a fast low-level option.
  - **RDMA (Remote Direct Memory Access) / RoCE / InfiniBand** — writing directly into a remote machine's memory with no CPU involvement; used in some HFT and HPC contexts.
  - **AF_XDP / XDP (eBPF)** — Linux's *native* fast-path: process packets in the kernel at the driver level (XDP) or hand them to user space with minimal overhead (AF_XDP). A modern, open option.
- **The cost of bypass: you now own the network stack.** No kernel TCP means *you* (or a library) handle reassembly, retransmission, ARP, etc. Understanding this trade-off is half the lesson.
- **Busy-polling as the execution model.** A dedicated, pinned, isolated core (Module 4) spinning 100% to poll the NIC. Why burning a whole core is *worth it* in HFT, and how it interacts with everything in Module 4.
- **Where the line is drawn.** Software bypass (DPDK/Onload) vs. pushing logic into the NIC/FPGA (Module 3) — a spectrum of "how close to the wire does the decision get made."

**What mastery looks like:** you can explain, microsecond by microsecond, what kernel bypass removes from the packet path, choose between the major technologies for a given constraint, and articulate the engineering cost you take on in exchange.

---

## Module 7 — C++ Data Structures: knowing the containers cold

> 📖 **Written:** [`07-cpp-data-structures.md`](07-cpp-data-structures.md) — the full lesson.

**Why now.** Before you can write *fast* C++ (Module 8), you must know the standard containers cold — what each one *is* underneath, what it can and can't do, how expensive each operation is, and (crucially for HFT) how its memory layout interacts with the cache hierarchy of Module 1. The single most common performance mistake in C++ is reaching for the wrong container, and the second is not knowing how to make the right one fast.

> **The mental model: a container is a contract about *memory layout* and *which operations are cheap*. Choosing one is choosing your cache behavior and your complexity profile at the same time.** The deepest divide is **contiguous** (`array`, `vector` — packed, cache-friendly, prefetcher-loved) vs. **node-based** (`list`, `map`, `set`, and chained hash tables — scattered across the heap, a cache miss per hop). Module 1 already told you which the machine prefers; this module makes it concrete container by container.

Concepts to master:

- **The cost vocabulary: Big-O complexity** — `O(1)`, `O(log n)`, `O(n)`, amortized cost — *and its limits*: Big-O hides the constant factor and cache behavior, so a "worse" complexity is often faster in practice (a linear scan of a `vector` beats a `O(log n)` `std::map` lookup for small/medium sizes). The HFT engineer reasons about *both* the asymptotic class *and* the cache.
- **Contiguous containers: `std::array` (fixed, stack) and `std::vector` (dynamic, heap).** O(1) random access, cache-friendly iteration, amortized-O(1) `push_back`, O(n) middle insert/erase; capacity, `reserve`, and iterator invalidation on reallocation.
- **Node-based sequence: `std::list` / `forward_list`.** O(1) splice/insert-with-iterator but O(n) search, no random access, and a cache miss per node — why HFT almost never uses it.
- **Ordered associative: `std::map` / `std::set`** (balanced binary search trees). O(log n) lookup/insert/erase, sorted iteration, range queries (`lower_bound`/`upper_bound`) — and the impossibilities (can't sort by value, can't mutate a key in place, no random access). The classic order-book structure.
- **Unordered associative: `std::unordered_map` / `unordered_set`** (hash tables). Average O(1) lookup/insert, worst-case O(n); no ordering or range queries; load factor, rehashing, and the hidden pointer-chasing in the standard chaining implementation.
- **The possible-and-impossible matrix** — for every container, what it *can* do, what it *can't*, and *why* the invariant forbids it.
- **How to speed them up** — `reserve` to kill reallocation/rehash spikes, preferring contiguous layouts, **flat (sorted-vector) maps/sets** and **open-addressing hash maps** as cache-friendly replacements, custom allocators / preallocated pools (Module 8), and choosing the container by *access pattern*, not habit.

**What mastery looks like:** given any "store these and do these operations on them" requirement, you pick the container whose complexity *and* cache behavior fit, you can state what it cannot do without being asked, and you know the faster drop-in alternative when the standard one stalls.

---

## Module 8 — C++ for HFT: the language, mapped onto the machine

**Why now (and why it's the biggest module).** C++ is *the* HFT language because it gives you control over memory layout and hardware with no runtime, no garbage collector, and zero-overhead abstractions. But "knowing C++" is not the point — the point is **writing C++ that respects everything in Modules 1–7.** Every idiom here is a direct response to a hardware fact (Modules 1–3) or a container fact (Module 7) you've already learned. That's why this module comes *after* the hardware and the containers: so each rule has a reason, not just an authority.

> **The mental model: in HFT, C++ is a way to *place data and control flow exactly where the hardware wants them*. You're not writing "object-oriented business logic"; you're choreographing cache lines, dependency chains, and branch predictors using a high-level notation.** Mechanical sympathy (Module 0) is the whole game; C++ is just the instrument.

Concepts to master, grouped by the hardware reality they serve:

### Mapping to memory (serves Module 1)

- **Data-oriented design: struct-of-arrays vs. array-of-structs.** Laying out data by *access pattern*, not by *conceptual object*, so the hot path touches contiguous cache lines. *The* foundational HFT C++ skill.
- **Cache-line awareness, `alignas`, and padding to avoid false sharing.** Putting hot fields on their own cache line; aligning structures to 64 bytes.
- **Stack vs. heap; why heap allocation is banned in the hot path.** `new`/`malloc` can lock, fault, and is unpredictable. *Payoff:* leads to the allocation strategies below.
- **Memory pools, arena/bump allocators, and object pools.** Pre-allocating everything at startup (touching it to pre-fault, Module 4) and never allocating during trading. *Payoff:* zero-allocation, deterministic hot path.
- **`std::vector` and contiguous containers vs. node-based containers (`std::list`, `std::map`).** Why pointer-chasing containers are avoided; flat maps, ring buffers, and contiguous alternatives.

### Mapping to control flow (serves Modules 1–2)

- **Branch reduction and `[[likely]]`/`[[unlikely]]`; branchless techniques.** Helping or removing the branch predictor's job in the hot path.
- **`inline`, the cost of function calls, and keeping the hot path in I-cache.** Why hot code is kept small and inlined.
- **The cost of `virtual`: vtables, indirect calls, and mispredicts.** Why runtime polymorphism is often replaced by compile-time polymorphism (next).

### Zero-overhead abstraction (the C++ superpower)

- **Templates and compile-time polymorphism (CRTP, the "curiously recurring template pattern").** Getting polymorphism's flexibility with *no* runtime indirection — the compiler resolves everything, then inlines it. *The* idiom that lets HFT code be both generic and fast.
- **`constexpr` / `consteval` and compile-time computation.** Moving work from run time to compile time — the cheapest nanosecond is one you spent before the market opened.
- **Template metaprogramming and policy-based design (in moderation).** Configuring behavior at compile time with zero runtime cost.
- **Move semantics, RVO, and avoiding copies.** Not paying for data movement you don't need.

### Concurrency (serves Module 1's cache-coherence section)

- **`std::atomic`, memory ordering (`acquire`/`release`/`relaxed`), and what they cost on x86.** Mapping back to Module 2's fences and Module 1's coherence.
- **Lock-free data structures, especially the single-producer/single-consumer (SPSC) ring buffer.** The canonical HFT inter-thread queue; why locks are avoided (blocking, priority inversion, jitter).
- **The thread-per-core model.** Pinned threads (Module 4) communicating via lock-free queues, not shared mutable state behind locks.

### Measurement and discipline

- **Profiling with hardware performance counters (`perf`, VTune, cache-miss/branch-miss counts).** Optimizing by *measurement*, not by *guess* — and measuring the *tail*, not just the mean.
- **Benchmarking pitfalls.** Why microbenchmarks lie (the optimizer deletes your test, warm vs. cold cache, frequency scaling); using Compiler Explorer (Module 2) and proper harnesses.
- **What the compiler does for you: optimization flags, LTO, PGO.** `-O3`, link-time optimization, profile-guided optimization — and reading the assembly (Module 2) to confirm it worked.

**What mastery looks like:** given a hot-path requirement, you choose data layouts and abstractions whose *machine behavior* you can predict and defend, you keep the critical path allocation-free and branch-friendly, and you prove the result with hardware counters — not vibes.

---

## Module 9 — Industry Practices & Standards: how it's actually built and run

**Why last.** All the speed in the world is worthless if you connect to the wrong feed, send a malformed order, blow through a risk limit, or violate a regulation. This module is the *context* that turns a fast program into a trading *system* — and it's what separates someone who can optimize code from someone an HFT firm will actually hire.

> **The mental model: HFT is a business with a hostile, adversarial, real-money environment and a regulator watching. Speed is necessary but not sufficient; correctness, risk control, and operational discipline are what keep the firm alive.** A bug here doesn't fail a unit test — it loses millions in seconds (see: Knight Capital, 2012, ~$440M lost in 45 minutes).

Concepts to master:

- **The FIX protocol (Financial Information eXchange).** The lingua franca for order entry and communication with exchanges/brokers; its tag-value format, sessions, and the low-latency binary variants (FIX/FAST, SBE — Simple Binary Encoding). *Payoff:* the standard you'll actually implement against.
- **Market data feeds and protocols.** Exchange-specific binary protocols (e.g. Nasdaq ITCH/OUCH, CME MDP); the structure of an **order book** and how you build/maintain one from a feed. *Payoff:* the core data structure of any trading system.
- **Exchange and matching-engine mechanics.** How orders match (price-time priority), order types, what "the book" is, and how your fill happens. You can't trade well against a machine you don't understand.
- **Risk controls and pre-trade checks.** Hard limits enforced *before* an order leaves (max order size, position limits, price collars, kill switches). Often the one place a few nanoseconds of latency is *deliberately accepted* for safety. *Payoff:* both an engineering and a regulatory necessity.
- **Determinism, testing, and replay.** Capturing real market data and **replaying** it to test strategies and reproduce bugs deterministically (a packet capture is the ultimate regression test); simulation and backtesting. *Payoff:* how you develop safely against a non-reproducible live market.
- **Latency measurement as a discipline.** End-to-end "tick-to-trade" measurement, hardware timestamping (Module 3), histograms and percentiles (you report p99/p99.9, never just the mean). *Payoff:* the firm competes on this number; you must be able to measure and defend it.
- **Regulation and compliance.** MiFID II (Europe), Reg NMS / SEC rules (US), the clock-synchronization mandates (back to PTP, Module 3), audit trails, market-making obligations, and circuit breakers. *Payoff:* the rules of the game you're legally bound to.
- **System architecture and operations.** Deployment in colo, monitoring without perturbing the hot path, deterministic logging (write to a ring buffer, drain off the hot path), and the separation of the fast "data plane" from the slower "control plane." *Payoff:* how the whole system is structured so the hot path stays pure.

**What mastery looks like:** you can describe the full lifecycle of a trade — from market-data packet, through your strategy and risk checks, to an order on the wire and a fill back — naming the protocols, the controls, and the measurement at each step, and you understand the business and regulatory stakes that make correctness non-negotiable.

---

## Putting it together: the tick-to-trade path as the capstone

Everything in this plan converges on one diagram you should be able to draw and annotate by the end — the **tick-to-trade path**:

> A price-change packet leaves the exchange matching engine → travels the colocated network (Module 3) → arrives at your NIC (Module 3) → is delivered to user space via kernel bypass, no kernel, no copy (Module 6) → your busy-polling thread on an isolated, pinned core (Module 4) reads it → you parse the market-data protocol and update the order book (Modules 5, 9), held in cache-friendly containers (Module 7) → your strategy decides, in cache-resident, branch-friendly, allocation-free C++ (Modules 1, 7, 8) → a pre-trade risk check passes (Module 9) → an order packet is encoded (FIX/SBE, Module 9) → sent straight to the NIC via kernel bypass (Module 6) → onto the wire to the exchange (Module 3) → all measured nanosecond-by-nanosecond with hardware timestamps (Modules 2, 3, 9).

When you can draw that path, name every component, explain *why* each is built the way it is, and point to where each nanosecond goes — you've integrated the whole curriculum. **That diagram is the job.**

---

## Suggested study sequence and milestones

1. **Foundations (Modules 1–2):** Build the CPU mental model and the habit of reading assembly in Compiler Explorer. *Milestone:* predict and then measure the cost of a random-vs-sequential array traversal, and explain the cache-miss difference.
2. **The machine and the OS (Modules 3–4):** Understand the NIC and tame a Linux box. *Milestone:* produce a low-jitter tuning checklist and measure jitter before/after with `cyclictest`.
3. **The network and the bypass (Modules 5–6):** Understand the stack's cost, then learn to skip it. *Milestone:* explain the full kernel network path and exactly what DPDK/Onload removes from it.
4. **The containers (Module 7):** Learn the STL containers' complexity, limits, and cache behavior. *Milestone:* benchmark a `std::map` vs. a sorted `std::vector` (flat map) lookup at several sizes and explain the crossover with cache behavior.
5. **The language (Module 8):** Learn C++ as an instrument for the hardware. *Milestone:* write a single-producer/single-consumer lock-free ring buffer and a zero-allocation order-book update, and profile both with `perf`.
6. **The business (Module 9):** Learn the protocols, risk, and regulation. *Milestone:* parse a sample market-data feed into an order book and encode a FIX order with a pre-trade risk check.

---

## Glossary (quick reference)

| Term | Meaning |
| --- | --- |
| **HFT** | High-Frequency Trading — automated trading where success depends on extremely low latency. |
| **Latency / jitter** | Time from input to output / the *variance* in that time (the rare slow spike). Both must be minimized. |
| **Tick-to-trade** | The end-to-end time from receiving a market-data update ("tick") to sending an order. The number HFT competes on. |
| **Mechanical sympathy** | Writing software that respects how the hardware actually works, to go faster. The spine of this curriculum. |
| **Cache line** | The 64-byte unit in which memory moves through the cache hierarchy. |
| **NUMA** | Non-Uniform Memory Access — memory local to your CPU socket is faster than remote. |
| **False sharing** | Two cores fighting over a cache line because unrelated hot variables share it. |
| **Branch misprediction** | The CPU guessing an `if` wrong and paying ~15–20 cycles to recover. |
| **NIC** | Network Interface Card — the hardware the latency race starts and ends at. |
| **Kernel bypass** | Talking to the NIC directly from user space, skipping the kernel's network stack (DPDK, Onload, AF_XDP, RDMA). |
| **Busy-polling** | A pinned core spinning to check the NIC for packets instead of waiting on interrupts — trades a CPU core for low, predictable latency. |
| **Core isolation / pinning** | Reserving CPU cores so only your thread runs on them, eliminating preemption jitter. |
| **Huge pages** | Large (2 MB/1 GB) memory pages that reduce TLB misses. |
| **FPGA** | Reconfigurable hardware used to make trading decisions in the NIC itself, in nanoseconds. |
| **PTP** | Precision Time Protocol — sub-microsecond clock synchronization, needed for measurement and regulation. |
| **Colocation (colo)** | Renting space inside the exchange's data center to minimize the physical distance (and thus latency) to the matching engine. |
| **UDP multicast** | The broadcast protocol exchanges use to send market data to all subscribers at once. |
| **FIX** | Financial Information eXchange — the standard protocol for order entry; binary variants (FAST, SBE) for low latency. |
| **Order book** | The live set of buy/sell orders at each price; the core data structure built from the market-data feed. |
| **CRTP** | Curiously Recurring Template Pattern — C++ compile-time polymorphism with no runtime indirection. |
| **SPSC ring buffer** | Single-Producer/Single-Consumer lock-free queue — the canonical HFT inter-thread channel. |
| **DPDK / Onload / AF_XDP / RDMA** | The main kernel-bypass technologies (full framework / socket-accelerator / native-Linux / remote-memory). |

---

## Where to go next

Once the plan above is internalized, the natural directions are:

- **Write the full lessons.** Each module here is a candidate for its own deep, concept-first lesson file (`01-computer-architecture.md`, etc.) in the style of the [`../linux/`](../linux/) course.
- **Build a toy stack end-to-end.** A market-data replayer → order book → strategy → simulated exchange, profiled at each stage — the single best way to make all eight modules concrete.
- **Read the canon.** Agner Fog's optimization manuals (architecture & assembly), the Intel/AMD optimization guides, *C++ High Performance* and the CppCon low-latency talks, and the DPDK/Onload documentation.

> The fastest path into HFT engineering is not memorizing any one of these modules — it's understanding how they *connect*, because the job is precisely to reason across all of them at once, in nanoseconds, on the tick-to-trade path.
