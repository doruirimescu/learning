# Lesson 1 — Computer Architecture: The Machine You're Actually Racing On

> [The README](README.md) made one promise above all others: in HFT, **latency is the product**, and every nanosecond on the path from "a price-change packet hits your network card" to "your order packet leaves it" is the whole game. This lesson is where that promise meets the metal. You cannot remove a nanosecond you don't understand, and *almost all* of the nanoseconds you'll fight over in software live inside the CPU and its memory system. So before NICs, before Linux tuning, before a single line of C++, we build an honest mental model of the machine. By the end you'll understand why the modern CPU is *nothing like* the simple "read an instruction, do it, repeat" device most programmers imagine — and why that gap is exactly where HFT performance is won and lost.
>
> This lesson assumes **no prior hardware knowledge**. We build every concept from the ground up. It's long because it's foundational: every later lesson leans on it.

---

## 0. First, the units: what a nanosecond actually is

We're going to throw around words like "cycle," "nanosecond," and "cache miss." Let's pin them down so they mean something concrete, because in HFT these aren't abstractions — they're money.

A **nanosecond (ns)** is one billionth of a second. There are a billion of them in a second. Light — the fastest thing in the universe — travels about **30 centimeters (one foot) in a nanosecond.** Hold that image: in the time it takes light to cross your desk, that's *one* of the units we're counting. HFT systems care about *tens* of nanoseconds.

A modern CPU has a **clock** — a metronome ticking billions of times per second. Each tick is a **cycle**. A "3 GHz" CPU ticks **3 billion times per second**, so one cycle lasts about **one-third of a nanosecond (~0.33 ns)**. The CPU does its work in lockstep with these ticks. When we say something "costs 200 cycles," on a 3 GHz machine that's about **66 nanoseconds** of wall-clock time. Throughout this lesson I'll give costs in both cycles and nanoseconds, because cycles are how hardware people think and nanoseconds are how the latency race is scored.

A few more building blocks:

- A **bit** is a single 0 or 1. A **byte** is 8 bits. Memory and caches are measured in bytes (KB = thousand, MB = million, GB = billion bytes).
- A **register** is a tiny storage slot *inside* the CPU core itself — it holds one value (typically 8 bytes on a 64-bit machine) that the CPU can use *instantly*, with zero delay. A core has only a few dozen of these. They are the fastest storage that exists. Everything the CPU actually computes on must be in a register at the moment of computation.
- An **instruction** is one elementary thing the CPU can do: "add these two registers," "load a value from memory into a register," "jump to a different part of the program if this register is zero." Your C++ program, after compilation (Lesson 2), is ultimately a long list of these.

That's the whole vocabulary we need to start. Now the single most important idea in the lesson.

---

## 1. The central surprise: computing is nearly free; *moving data* is the expense

Most programmers carry an unspoken mental model of the CPU that goes like this: "the computer reads my instructions one at a time and does them; to make my program faster, I should make it do *fewer* things." This model is **wrong in a way that matters enormously for HFT**, and replacing it is the purpose of this lesson.

Here is the truth, and it surprises nearly everyone the first time:

> **A modern CPU core can perform arithmetic faster than it can fetch the numbers to do arithmetic *on*.** Adding, multiplying, comparing — these take a cycle or less. But *getting a value from main memory* into a register where the CPU can touch it takes **hundreds of cycles.** The computation is the cheap part. The *data movement* is the expensive part, often by a factor of a hundred or more.

Let that sink in, because it inverts the naive model. The bottleneck in modern high-performance code is almost never "the CPU is doing too much math." It's "the CPU is sitting idle, *waiting* for data to arrive from memory." A core stalled waiting on memory is a race car idling at a red light — the engine's power is irrelevant while it waits.

The mental model that will carry this entire lesson:

> **The CPU is a tiny, impossibly fast workshop, fed by a long and slow supply chain.** Inside the workshop, a master craftsman (the execution units) can assemble parts the instant they're handed to him — assembly is practically free. But the parts live in warehouses of increasing distance. The closest shelf (registers) is at his fingertips. A nearby cabinet (L1 cache) is a step away. A bigger storeroom down the hall (L2/L3 cache) is a short walk. And the giant central warehouse (main memory / RAM) is *across town.* The craftsman's speed is almost never the limit. The limit is **how often he has to wait for a part to be carried in from across town** — and the entire art of high-performance computing is *keeping the parts he needs close to his hands.*

Everything else in this lesson — caches, cache lines, prefetching, data layout, NUMA — is a detail of *how the supply chain works and how you keep it from stalling the craftsman.* If you remember only one thing, remember this: **performance is a data-movement problem, not a computation problem.**

---

## 2. The memory hierarchy: why there are warehouses at all

Why not just make all memory as fast as registers? Physics and money. Fast memory is expensive, physically large per byte, and must sit *very close* to the core (signals travel at finite speed — remember, ~30 cm per nanosecond). You can have a *tiny* amount of blazing-fast storage, or a *huge* amount of slow storage, but not a huge amount of blazing-fast storage. So engineers build a **hierarchy**: a series of stores, each one bigger but slower than the last, arranged so the CPU checks the fastest one first.

Here is the hierarchy on a typical modern server CPU. The exact numbers vary by chip, but the *ratios* are what matter, and they're remarkably stable across hardware:

| Level | Typical size | Typical latency (cycles) | Approx. time @ 3 GHz | Analogy |
|-------|-------------|--------------------------|----------------------|---------|
| **Register** | ~dozens of values | 0 (instant) | 0 ns | In the craftsman's hand |
| **L1 cache** | 32–64 KB per core | ~4 cycles | ~1–1.5 ns | The cabinet at arm's reach |
| **L2 cache** | 256 KB–2 MB per core | ~12–15 cycles | ~4–5 ns | The storeroom down the hall |
| **L3 cache** | 8–64 MB, shared by all cores | ~40 cycles | ~15 ns | The building's basement archive |
| **Main memory (RAM)** | 64–512 GB | ~200–300 cycles | ~70–100 ns | The warehouse across town |
| **SSD / NVMe** | terabytes | — | ~10,000–100,000 ns | Another city |
| **Network / spinning disk** | — | — | millions of ns+ | Another continent |

**L1, L2, and L3 are the "caches"** — small, fast memories that sit between the core and main memory, holding *copies* of data the core has recently used (or is likely to use soon). "L" stands for "level"; L1 is closest and fastest, L3 farthest and largest. The plan is simple and beautiful: **when the core needs a value, it checks L1 first. If it's there (a "cache hit"), great — a couple of nanoseconds. If not (a "cache miss"), it checks L2, then L3, then finally trudges all the way to main memory** — and the deeper it has to go, the more cycles the core spends *waiting.*

### The human-scale analogy that makes the ratios visceral

The raw numbers (1 ns vs. 100 ns) are easy to wave away — they're all "tiny." So scale them up to human time. Pretend **one CPU cycle takes one second** instead of a third of a nanosecond. Now the hierarchy feels like this:

- **Register access:** instant — it's already in your hand.
- **L1 cache hit (~4 cycles):** ~4 seconds — you reach into a nearby drawer.
- **L2 cache hit (~12 cycles):** ~12 seconds — you walk to a shelf across the room.
- **L3 cache hit (~40 cycles):** ~40 seconds — you go down to the basement.
- **Main memory (~200+ cycles):** **~3–4 minutes** — you drive across town to the warehouse.
- **SSD read:** **a day or more.**
- **Network round-trip across the world:** **years.**

Suddenly it's obvious. A cache miss to main memory isn't "a bit slower" — at human scale, it's the difference between *grabbing something off your desk* and *driving across town and back.* If your trading code's hot path takes a few of those "drives across town" per market-data message, you've already lost the race to a competitor whose data was sitting in L1. **This table is the reason the rest of the lesson exists.** Every optimization is an effort to turn "drives across town" into "reaches into the drawer."

---

## 3. The cache line: memory moves in 64-byte chunks, never single bytes

Now a fact that quietly governs how you must lay out *every* piece of data in a fast system. When the CPU fetches something from memory, **it does not fetch the single byte (or single 8-byte number) you asked for.** It always fetches a fixed-size, aligned block called a **cache line** — almost universally **64 bytes** on modern hardware. That whole 64-byte line is carried up through the hierarchy and parked in the cache together.

Why? Two reasons. First, the overhead of a memory fetch is mostly in the *trip*, not the *amount* — once you've driven across town, you may as well fill the truck. Second, programs overwhelmingly use data near data they just used (more on this in the next section), so grabbing the neighbors is usually a smart bet.

> **The analogy: you never grab one sheet of paper from the warehouse — you grab the whole folder it's filed in.** Ask for one document and the supply chain hands you the entire 64-byte folder containing it and its neighbors. If the *next* thing you need is also in that folder, it's already on your desk — free. If your next need is in a *different* folder across town, you pay the full trip again.

This single fact has enormous consequences, and they're the bridge from "how the machine works" to "how you must write code":

- **If your data is packed tightly and you use it in order**, then one expensive fetch (one folder) serves you many useful values before you need another trip. You amortize the cost. This is *fast.*
- **If your data is scattered**, so that each value you need lives in a *different* 64-byte line, then every single access is a fresh trip across town, and you fetch 64 bytes to use only 8 of them — wasting 7/8 of every truck. This is *catastrophically slow*, even though you're "doing the same amount of work."

Two programs that compute the *exact same result* can differ in speed by 10× or more purely based on whether their data accesses stay within already-fetched cache lines. **In HFT, this is not a micro-optimization — it is the difference between a viable strategy and a dead one.** We'll make it concrete in Section 5.

---

## 4. Locality and the prefetcher: why predictable access is rewarded

The cache bets that grabbing neighbors pays off. That bet rests on a real, observed property of nearly all programs called **locality of reference**, which comes in two flavors:

- **Spatial locality:** if you access a piece of data, you're likely to access *nearby* data soon. (You read array element 5; you'll probably read element 6 next.) The cache line exploits this directly — it grabbed element 6's neighborhood when you touched element 5.
- **Temporal locality:** if you access a piece of data, you're likely to access *the same data again* soon. (A loop counter, a frequently-read configuration value.) The cache exploits this by *keeping* recently-used lines around rather than discarding them.

On top of these, the CPU has a marvelous piece of machinery called the **hardware prefetcher.** It watches your memory-access *pattern* and, when it detects a predictable rhythm — say, you're marching through memory at a steady stride (element 0, 1, 2, 3...) — it **starts fetching the upcoming cache lines *before you ask for them.*** By the time the craftsman reaches for the next folder, it's already been carried in. The expensive trip across town happened *in the background*, overlapped with useful work, so the core never had to wait.

> **The prefetcher is an assistant who watches you work and, noticing you're going through files in order, runs ahead to fetch the next ones before you ask.** Predictable, sequential, steady-stride access patterns make this assistant brilliantly effective — the data is always waiting for you. **Random, unpredictable access defeats the assistant entirely** — he can't guess where you'll jump next, so you wait for every fetch.

This is *the* reason a recurring rule in high-performance code is "**access memory sequentially and predictably.**" It's not superstition. It's that sequential access (a) keeps you inside already-fetched cache lines (spatial locality) and (b) lets the prefetcher hide the cost of future fetches. Random access does the opposite on both counts. **Marching through a contiguous array is the single most cache-friendly thing you can do; chasing pointers around the heap is among the worst.** Hold that thought — it's about to become a concrete design rule.

---

## 5. The payoff: data layout, or why a linked list can ruin you

Now we cash in everything from Sections 2–4 into the most practical, money-making idea in this lesson: **how you arrange your data in memory often matters more than what algorithm you run on it.** Let's make it vivid with the two most important contrasts.

### Contiguous arrays vs. pointer-chasing (the linked-list trap)

Imagine you have a million numbers to add up. Two ways to store them:

1. **A contiguous array:** all million numbers sitting back-to-back in one solid block of memory. To sum them, you march from start to finish. Each cache line you fetch contains 16 numbers (64 bytes ÷ 4 bytes each), all of which you immediately use. The prefetcher sees your steady march and runs ahead. **You almost never wait.** This is close to the theoretical best the machine can do.

2. **A linked list:** each number lives in its own little separately-allocated node, and each node holds a *pointer* (a memory address) to the next one. The nodes are scattered all over the heap, wherever the allocator happened to place them. To sum them, you follow pointer → pointer → pointer. Each "next" lands you at a random address, almost certainly in a *different* cache line that isn't in cache and that the prefetcher couldn't predict (it can't guess a pointer's value ahead of time). **Every single step is a fresh trip across town** — a cache miss, ~100 ns of the core sitting idle. A million of them.

The two programs compute the identical sum. The contiguous version can be **10× to 50× faster** — not because it does less arithmetic (it does exactly the same amount), but because the linked list spends nearly all its time *waiting for memory* while the array spends nearly all its time *working.* 

> **This is why HFT code is built on flat, contiguous arrays and is deeply suspicious of pointer-based structures** (`std::list`, `std::map`, tree-of-nodes, graphs of separately-allocated objects). It's not dogma — it's the cache-line and prefetcher facts of Sections 3–4 made into a design rule. The hot path *marches through contiguous memory* whenever it possibly can. We'll see the C++ specifics in Lesson 7, but the *reason* lives here, in the architecture.

### Array-of-Structs vs. Struct-of-Arrays

A subtler, equally important version. Say each "thing" in your system has several fields — imagine a market-data record with a price, a quantity, a timestamp, and a symbol ID. You have a million of them, and your hot path needs to scan **only the prices** (say, to find the best one). Two layouts:

**Array of Structs (AoS)** — the natural, object-oriented way. Each record is a struct holding all its fields together, and you have an array of these structs:

```
[price0 qty0 time0 sym0][price1 qty1 time1 sym1][price2 qty2 time2 sym2]...
```

When you scan just the prices, watch what the cache line does. You fetch a 64-byte line to read `price0` — but that line is *filled with* `qty0`, `time0`, `sym0` and maybe the start of record 1, none of which you want right now. You used maybe 4 of the 64 bytes you hauled across town. **You're wasting ~90% of your memory bandwidth** and touching far more cache lines than necessary.

**Struct of Arrays (SoA)** — the data-oriented way. Store each field in its own contiguous array:

```
prices:  [price0 price1 price2 price3 ...]   ← all packed together
qtys:    [qty0   qty1   qty2   qty3   ...]
times:   [time0  time1  time2  time3  ...]
syms:    [sym0   sym1   sym2   sym3   ...]
```

Now when you scan prices, every byte of every cache line you fetch *is a price you want.* One fetch gives you 16 prices. The prefetcher locks onto the steady march. You touch the minimum possible number of cache lines and waste nothing. For a price-scan hot path, SoA can be **several times faster** than AoS, again with identical arithmetic.

> **The lesson: lay out data according to how the hot path *accesses* it, not according to how you conceptually *group* it.** Objects-bundling-their-fields is comfortable for human reasoning but often wrong for the cache. This idea — called **data-oriented design** — is foundational to HFT C++ (Lesson 8), and now you know *why* it works: it's the cache line (Section 3) and the prefetcher (Section 4) speaking. **The cache doesn't care about your object model. It cares about which 64-byte folders you touch, and in what order.**

---

## 6. Inside the core: the CPU does not do one instruction at a time

We've spent five sections on the *memory* side of the supply chain because that's where most HFT latency hides. Now we step inside the workshop itself — the core — because how it executes instructions is just as far from the naive model, and it explains the *other* big latency source: branches.

The naive model says the CPU does: fetch instruction → do it → fetch next → do it, one at a time, in order. **Every part of that is false on a modern CPU.** Here's what really happens.

### Pipelining: the assembly line

Executing one instruction isn't a single action — it's several stages: **fetch** the instruction from memory, **decode** what it means, **execute** the operation, and **write back** the result (real CPUs have 15–20 finer stages, but four is enough for the idea). 

The naive CPU would do all four stages for instruction 1, *then* all four for instruction 2, and so on — like a laundromat where you wash, dry, and fold one load completely before starting the next, leaving the washer idle while you dry. Wasteful. Instead, the CPU **pipelines**: the moment instruction 1 leaves the "fetch" stage and moves to "decode," instruction 2 enters "fetch." Like a smart laundromat running the washer, dryer, and folding table *simultaneously* on different loads. At full tilt, a four-stage pipeline can finish one instruction every cycle even though each instruction takes four cycles to traverse — because four are always in flight at once.

> **The pipeline is an assembly line. Instructions don't wait politely in single file; they're processed in overlapping stages, many in flight at once.** This is great for throughput — until something disrupts the line, at which point the cost of the disruption is large because so much in-flight work may have to be thrown away. (That "something" is usually a branch — Section 7.)

### Superscalar and out-of-order: many lanes, reordered work

Two more layers of cleverness pile on:

- **Superscalar execution:** the core has *multiple* execution units — several adders, multipliers, load/store units — so it can genuinely do **several instructions in the same cycle**, in parallel, if they don't depend on each other. It's not one assembly line; it's *several* running side by side.
- **Out-of-order execution:** the core does *not* slavishly follow your program's written order. It looks ahead at a window of upcoming instructions and executes whichever ones have their inputs ready, *while* others are stalled waiting for data. If instruction 5 is waiting on a value from memory (a cache miss — across-town trip!), the core can run instructions 6, 7, and 8 in the meantime, as long as they don't need instruction 5's result. It then reassembles the results into the correct order so your program's logic is preserved. This is how the core partially *hides* memory latency — by finding other useful work to do during the wait.

The name for "how much independent work the core can find to do in parallel" is **Instruction-Level Parallelism (ILP).** More ILP means the expensive units and the memory waits get overlapped with useful work, and the core stays busy.

### The real cost is the *dependency chain*, not the instruction count

Here's the consequence that overturns the naive optimization instinct. Because the core runs independent instructions in parallel and out of order, **counting instructions is a poor guide to speed.** What actually limits you is the **dependency chain**: a sequence of instructions where each *needs the result of the previous one*, so they *cannot* be parallelized — they must run one after another, each waiting for the last.

Consider: `a = b + c; d = a + e; f = d + g;` — each line needs the previous line's result. Three additions, but they're forced to run *in sequence*; the core can't start the second until the first finishes. Now consider: `a = b + c; d = e + f; g = h + i;` — also three additions, but *independent*. The superscalar core can do all three *in the same cycle.* Same instruction count; three times the speed.

> **The takeaway that guides micro-optimization (Lesson 2): the speed of a hot code sequence is governed by its longest chain of dependent operations, not by how many operations it contains.** Shortening that critical dependency chain — and giving the core independent work to overlap with memory waits — is what actually moves the needle. "Doing fewer instructions" often does nothing if the instructions you removed weren't on the critical path. This is why HFT engineers read assembly and reason about dependencies (Lesson 2) rather than just counting lines of code.

---

## 7. Branch prediction: the CPU gambles on your `if` statements

Now we hit the single most important thing happening *inside* the core for HFT, and it follows directly from the pipeline. The pipeline needs to know *which instruction comes next* so it can keep the assembly line full. Usually that's easy — the next instruction is the one after this one. But what about an `if` statement, a loop condition, a `switch` — a **branch**, where the next instruction depends on a comparison that *hasn't finished computing yet?*

The pipeline can't afford to stop and wait — that would drain the assembly line and waste all those overlapping stages. So the CPU does something audacious: **it guesses.** A dedicated piece of hardware, the **branch predictor**, predicts which way the branch will go (taken or not taken), and the core *speculatively* charges ahead down the predicted path, executing instructions there **before it even knows whether the guess was right.**

> **The branch predictor is a chef who starts cooking the dish he bets you'll order before you've finished speaking.** Most of the time he's right — you're a regular, you always order the same thing — and your food is ready absurdly fast because he started early. The predictor is *startlingly* good at this: for predictable branches (a loop that runs a million times always takes the "keep looping" branch; an error-check that almost never fires), it's right well over 95% of the time, and those branches cost effectively **nothing.**

But what happens when the guess is *wrong* — a **branch misprediction**? The chef has cooked the wrong dish. Everything he prepared must be thrown out, and he starts over on the correct order. In CPU terms: all the speculative work in the pipeline must be discarded ("flushed"), and the pipeline must refill from the correct path. That costs roughly **15–20 cycles** — call it **~5–7 nanoseconds** — every time. On the human scale from Section 2, that's the chef scrapping a half-cooked meal and beginning anew while you wait.

Now connect it to HFT. A few nanoseconds, *occasionally*, sounds harmless. But:

- A branch whose outcome is **predictable** (almost always goes the same way, or follows a simple repeating pattern) is essentially free — the predictor nails it.
- A branch whose outcome is **data-dependent and effectively random** — for example, "is this incoming order a buy or a sell?" where it's genuinely 50/50 and unpredictable — defeats the predictor. It guesses wrong about half the time, and you eat the full mispredict penalty on a huge fraction of those branches, right in your hot path.

> **This is why HFT code is written to be branch-predictable, or branch-*less* entirely.** Engineers restructure logic so the common case is an overwhelmingly-predictable branch (so it's free), and sometimes eliminate unpredictable branches altogether using **branchless techniques** — computing both possibilities and selecting between them with arithmetic or a conditional-move instruction, so there's *no branch for the predictor to miss.* You'll meet the C++ forms of this in Lesson 8 (`[[likely]]`/`[[unlikely]]`, branchless selection). The *reason* they matter is this section: an unpredictable branch in the hot path is a recurring ~5 ns tax, and in a race scored in nanoseconds, that's real.

A crucial subtlety for HFT's obsession with *predictability* (jitter, from Module 0): branch misprediction is a source of *variance*, not just average cost. The same code path can be fast when the branch predictor is "warm" and trained, and slower right after a pattern change — exactly the kind of inconsistency HFT works to eliminate.

---

## 8. Virtual memory and the TLB: the hidden translation tax

So far we've talked as if a memory address directly names a location in RAM. It doesn't, and the indirection has a latency cost you must know about.

For safety and flexibility, modern systems give every program its *own* private, imaginary view of memory called its **virtual address space** — each program thinks it has a vast contiguous memory all to itself, starting at address zero. In reality, those **virtual addresses** are mapped to scattered locations in the *real* physical RAM by the operating system, in chunks called **pages** (traditionally **4 KB** each). (The full story of virtual memory — why it exists, paging, the page cache — is a Linux-level topic; see the memory deep-dive in [`../linux/`](../linux/). Here we care only about its *performance* consequence.)

The consequence: **every single memory access your program makes requires translating a virtual address into a physical one.** If the CPU had to consult the operating system's translation tables (which themselves live in memory!) on every access, that would be ruinous — extra across-town trips for *every* load. So, just as it caches data, the CPU caches *translations* in a special small, fast table called the **TLB (Translation Lookaside Buffer).**

> **The TLB is the index card that tells the craftsman "the folder you call #5 is physically in warehouse aisle 12."** When the translation you need is on a card in the TLB (a **TLB hit**), translation is instant and invisible. When it isn't (a **TLB miss**), the CPU must walk the operating system's full translation tables to find the mapping — several extra memory accesses, tens to hundreds of cycles — *before it can even begin* the actual data fetch you wanted. A TLB miss is a tax paid *on top of* a possible cache miss.

The TLB is small — it holds only a few hundred to a couple thousand translations. Since each ordinary entry covers one 4 KB page, the TLB can only "see" a few megabytes of memory at once before it starts missing. If your hot data structures sprawl across many pages, you'll suffer TLB misses *in addition to* cache misses, and the translation tax becomes a measurable latency source.

The HFT remedy, which you'll configure in Lesson 4 (OS tuning): **huge pages.** Instead of 4 KB pages, the OS can use **2 MB** (or even 1 GB) pages. Now a single TLB entry covers 2 MB instead of 4 KB — **500× more memory per entry** — so the same small TLB can map your entire working set with room to spare, and TLB misses on the hot path nearly vanish.

> **Huge pages are how HFT systems make the translation tax disappear.** It's a direct, deliberate trade: bigger pages mean fewer translations to cache, which means the TLB covers your whole hot dataset, which means no surprise translation stalls in the middle of the race. Remember this section when Lesson 4 tells you to enable huge pages — the *why* lives right here.

---

## 9. Many cores: NUMA and the cost of distance inside the box

Real trading servers have many CPU cores, and often **two physical CPU chips ("sockets")** on one motherboard. This introduces a geography problem you must respect.

When there are two sockets, the system's RAM is physically split: some memory sticks are wired directly to socket 0, and some to socket 1. A core on socket 0 can reach *its own* attached memory quickly. But to read memory attached to socket 1, its request must travel across the **inter-socket link** (a connection bridging the two chips) — a noticeably longer trip, often **1.5–2× the latency** of local access. This arrangement is called **NUMA — Non-Uniform Memory Access** — "non-uniform" precisely because *not all memory is equally far away.* The combination of a socket and its directly-attached memory is a **NUMA node.**

> **The analogy: two kitchens sharing a building.** Each kitchen (socket) has its own pantry (local memory) right beside it. A chef can grab from his own pantry instantly. But if he needs an ingredient from the *other* kitchen's pantry, someone has to walk it over — slower, every time. A well-run operation keeps each chef's ingredients in *his own* pantry.

The HFT consequence is a hard rule you'll implement in Lesson 4: **pin each thread to a specific core, *and* ensure the memory it uses is allocated on that same core's NUMA node.** If your busy-polling thread runs on socket 0 but its data sits in socket 1's memory, every access pays the inter-socket tax — a silent, constant latency penalty for no reason. Worse, if you let the OS freely migrate the thread between sockets, its data is sometimes local and sometimes remote, injecting *jitter* (variance) — the thing HFT hates most. The fix is to lock the thread *and* its memory to one node so every access is local and consistent. (You'll also often see the NIC itself attached to a specific socket's PCIe lanes — Lesson 3 — and you'll want your network-handling thread on *that* socket, so the packet data arrives into local memory.)

---

## 10. Cores fighting over data: cache coherence and false sharing

Multiple cores create a second, subtler problem — and one of the most infamous latency bugs in concurrent code, so we'll take it slowly.

Each core has its *own* private L1 and L2 caches. So what happens when two cores both have a copy of the same data in their caches and one of them *changes* it? If nothing kept them in sync, core B would keep reading a stale value while core A had already updated it — chaos. To prevent this, hardware runs a **cache coherence protocol** (the common one is called MESI, after the four states a cache line can be in). You don't need its internals; you need its *cost*:

> **Cache coherence guarantees all cores agree on the current value of any memory location, but it enforces that agreement through expensive coordination.** When one core writes to a cache line, the protocol must *invalidate* every other core's copy of that line — sending messages between cores and forcing the others to re-fetch the line next time they need it. A cache line being written by one core and read by others "ping-pongs" between caches, and each bounce costs dozens to hundreds of cycles.

That coordination is *necessary* when cores genuinely share data. But here's the trap that bites everyone, called **false sharing**:

Remember that the unit of caching is the **64-byte cache line** (Section 3), not the individual variable. Now suppose two *completely unrelated* variables — say, a counter that core A updates constantly and a different counter that core B updates constantly — happen to sit in the *same 64-byte cache line* (because you declared them next to each other, or they fell adjacent in a struct). The two cores never touch *each other's* variable. Logically there is no sharing at all. **But the hardware tracks coherence per cache line, not per variable.** So every time core A writes its counter, it invalidates the *whole line* in core B's cache — including B's unrelated counter — forcing B to re-fetch. And vice versa. The line ping-pongs furiously between the two cores' caches, and both run dramatically slower, *as if* they were fighting over shared data — when in truth they share *nothing* except an accident of memory layout.

> **The analogy: two chefs at opposite ends of one long cutting board.** Chef A chops onions on the left; Chef B dices carrots on the right. They never touch each other's food. But it's *one board*, and the kitchen's rule is "only one chef may hold the board at a time." So every time A reaches to chop, he must take the whole board — yanking it away from B — and then B must take it back. They spend their day passing the board back and forth instead of cooking, *purely because their unrelated work shares one physical object.* That's false sharing: unrelated data, one cache line, ruinous ping-ponging.

False sharing can slow a concurrent hot path by **5–10×**, and it's invisible in the source code — the variables look independent because they *are* independent. The fix, which you'll write in C++ in Lesson 8, is **padding and alignment**: deliberately space hot, independently-written variables so each lives on its *own* cache line (e.g. `alignas(64)`), giving each chef his own cutting board. The *reason* you'll do that lives here: the coherence protocol works on whole lines, so you keep independently-written data on separate lines.

This section is also why HFT favors a **share-nothing, thread-per-core** design (Lesson 8): rather than many cores contending over shared mutable data (paying coherence costs and risking false sharing), each core owns its own data and threads communicate by passing messages through carefully-designed lock-free queues. Less sharing means less coherence traffic means lower, more predictable latency.

---

## 11. Hyper-threading (SMT) and why HFT often turns it off

One more piece of CPU cleverness, and a real tuning decision that follows from it.

**Simultaneous Multithreading (SMT)** — Intel brands it **Hyper-Threading** — lets a *single physical core* present itself to the operating system as **two logical cores.** The idea: recall that cores often stall waiting on memory (Section 2). During those stalls, the core's execution units sit idle. SMT exploits this by keeping *two* independent streams of instructions ready on one core; when one stream stalls on a memory wait, the core can execute instructions from the *other* stream, filling the idle gaps. For general-purpose servers chasing throughput, this is a nice win — the hardware stays busier.

For HFT, it's usually the **wrong** trade, and firms typically **disable it in the BIOS.** Why?

> **SMT means two logical cores *share* one physical core's real resources** — its execution units, and critically its L1 and L2 caches. The two threads compete. When your latency-critical trading thread is running on a logical core, the *other* logical core on the same physical core can evict your thread's data from the shared L1/L2 cache, steal execution-unit slots, and generally inject **interference and variance** into your carefully-tuned hot path. You traded a little extra throughput for a lot of *unpredictability* — and predictability (Module 0) is what HFT prizes.

So the standard HFT posture is: **disable hyper-threading**, so each physical core is dedicated to one thread with its full, uncontended cache and execution resources. Combined with core isolation and pinning (Lesson 4), this gives your hot thread a private, quiet, consistent core — no roommate evicting its cache, no roommate stealing its execution slots. Throughput drops; latency *consistency* improves; HFT happily takes that deal. This is one of the concrete BIOS-level decisions Lesson 3 (hardware tuning) will revisit.

---

## 12. Putting it together: where the nanoseconds actually go

Let's assemble everything into one concrete picture — a simplified slice of what happens when a market-data packet's price update flows through your code, annotated with the architectural costs you now understand. This is the "tick-to-trade path" from the [README](README.md) viewed through the *architecture* lens (later lessons add the NIC, kernel bypass, and network layers).

Imagine your hot path, on receiving a price, must: look up the instrument's record, update its order book, and run a quick decision. Watch the architecture decide your latency:

1. **The instrument lookup.** If your instrument data is a `std::map` (a pointer-chasing tree, Section 5), each lookup chases pointers through scattered nodes — several cache misses, several across-town trips, *hundreds of nanoseconds*, plus possible TLB misses (Section 8) if the nodes span many pages. If instead it's a flat array indexed by a small instrument ID (contiguous, Section 5), it's one cache line, likely already hot in L1 — *a nanosecond or two.* **Same logical operation, ~100× latency difference, decided purely by data layout.**

2. **The order-book update.** If the book is laid out struct-of-arrays with the hot fields contiguous (Section 5), the update touches a minimal set of cache lines that the prefetcher (Section 4) has likely kept warm. If it's a sprawl of separately-allocated objects, you scatter across the heap and stall.

3. **The decision branch.** "Is this price better than my threshold?" If that branch is predictable (Section 7), free. If your logic has an unpredictable, data-dependent branch in the middle of the hot path, you eat ~5 ns *and* inject jitter every time it mispredicts.

4. **The whole thing, on the right core.** If the thread is pinned to a core on the same NUMA node as its data and the NIC (Section 9), with hyper-threading off (Section 11) so no roommate evicts its cache, and its memory locked into huge pages (Section 8) so no TLB stalls and no page faults — then every one of the accesses above hits warm, local, fast storage *consistently.* If any of those is wrong, you get sporadic across-town trips, coherence ping-pong, or translation stalls — and your latency becomes both *higher* and *less predictable.*

> **The unifying insight:** a market-data message might require only a few dozen actual arithmetic operations — *nanoseconds* of real computing. Whether processing it takes **50 nanoseconds or 5,000 nanoseconds** is determined almost entirely by **how many times the core has to wait for data** — which is determined by cache behavior, data layout, branch predictability, translation, NUMA locality, and core contention. *That* is computer architecture for HFT in one sentence: **your competition isn't out-computing you; whoever stalls the core less, wins.**

---

## 13. How you actually observe this: hardware performance counters

You might fairly ask: "If cache misses and branch mispredicts are invisible in my source code, how do I *know* where they're happening?" The answer, which Lessons 2 and 7 develop, is that the CPU itself **counts** these events in dedicated hardware registers called **performance counters**, and tools can read them.

On Linux, the front-line tool is **`perf`**. It can tell you, for a running program, how many cache misses (at each level), branch mispredictions, TLB misses, and stalled cycles you incurred, and attribute them to specific functions and even lines of code. Specialized tools (Intel VTune, `perf stat`, `perf record`) go deeper. The crucial discipline, which the README's C++ module stresses: **you optimize by *measuring* these counters, never by guessing.** The architecture in this lesson tells you *what* to look for and *why* it costs; the counters tell you *where* it's actually happening in your code.

And — vital for HFT — you measure the **tail, not the average.** A path that averages 1 µs but occasionally spikes to 50 µs (a cache eviction, a TLB miss, a migration, a mispredict storm) is an HFT liability even though its *mean* looks great. You'll report percentiles (p99, p99.9), and you'll hunt the architectural causes of the spikes — every one of which is a concept from this lesson.

---

## 14. Recap: the mental model in one page

Everything in this lesson hangs off a single inversion of the naive picture of a computer:

- **Computing is nearly free; moving data is the expense.** Performance is a data-movement problem. (§1)
- Data lives in a **hierarchy** from registers (instant) to main memory (~100 ns / "across town"). You check the fast levels first; a **cache miss** means a slow trip. (§2)
- Memory moves in **64-byte cache lines**, never single values. Use all 64 bytes you haul, or waste the trip. (§3)
- The **prefetcher** rewards **sequential, predictable** access and is useless against random access. (§4)
- Therefore: **lay out data for how the hot path accesses it** — contiguous arrays over pointer-chasing, struct-of-arrays over array-of-structs. Layout often beats algorithm. (§5)
- The core runs many instructions **in parallel, out of order** (pipelined, superscalar). The real cost is the **dependency chain**, not the instruction count. (§6)
- Branches are **gambled on** by the predictor; predictable branches are free, **unpredictable ones cost ~5 ns and add jitter.** Write branch-predictable or branchless hot paths. (§7)
- Every access is **translated** (virtual→physical); the **TLB** caches translations, and **huge pages** make the TLB cover your whole working set so translation stalls vanish. (§8)
- On multi-socket machines, memory is **NUMA** — local is fast, remote is slow. Pin threads *and* their memory to the same node. (§9)
- Cores keep caches coherent at a cost; **false sharing** (unrelated variables on one cache line) silently ruins concurrent code — fix with **padding/alignment**. (§10)
- **Hyper-threading** shares one core between two threads, injecting interference; HFT usually **disables it** for predictability. (§11)
- Net: a message needs nanoseconds of *computing*; whether it takes 50 ns or 5,000 ns is decided by **how often the core stalls** — and reducing stalls (and their variance) is the whole job. (§12)
- You find the stalls with **hardware performance counters** (`perf`), measuring the **tail**, never guessing. (§13)

If this model is solid, every later lesson clicks into place: the NIC (Lesson 3) is about getting packet data into a warm cache without the kernel; Linux tuning (Lesson 4) is about keeping the core un-stalled and un-interrupted; kernel bypass (Lesson 6) removes across-the-board overhead; and HFT C++ (Lesson 8) is, almost line for line, *this lesson turned into code.*

---

## 15. Glossary

| Term | Meaning |
|------|---------|
| **Cycle** | One tick of the CPU clock. On a 3 GHz CPU, ~0.33 ns. Hardware costs are measured in cycles. |
| **Register** | A storage slot inside the core holding one value; instant access; only a few dozen exist. All computation happens on registers. |
| **Cache (L1/L2/L3)** | Small, fast memories between the core and RAM holding copies of recently/soon-used data. L1 closest & fastest, L3 largest & shared across cores. |
| **Memory hierarchy** | The ordered stack from registers → L1 → L2 → L3 → RAM → SSD, each bigger but slower. |
| **Cache hit / miss** | Whether the data the core needs is (hit) or isn't (miss) in a given cache level. A miss means a slower trip to the next level. |
| **Latency** | Time to get a result; here, the delay before requested data arrives. RAM ≈ 100 ns vs. L1 ≈ 1 ns. |
| **Cache line** | The fixed 64-byte block in which memory always moves through the hierarchy. The fundamental unit of all data-layout reasoning. |
| **Spatial locality** | Tendency to access data near recently-accessed data. Exploited by cache lines. |
| **Temporal locality** | Tendency to re-access the same data soon. Exploited by keeping lines cached. |
| **Prefetcher** | Hardware that detects predictable access patterns and fetches upcoming cache lines early, hiding memory latency. Defeated by random access. |
| **AoS / SoA** | Array-of-Structs (fields bundled per object) vs. Struct-of-Arrays (each field in its own contiguous array). SoA is cache-friendly when scanning one field. |
| **Data-oriented design** | Laying out data by how the hot path accesses it, not by conceptual object grouping. |
| **Pipeline** | The CPU's overlapping instruction-processing stages (fetch/decode/execute/write-back) — an assembly line keeping many instructions in flight. |
| **Superscalar** | A core with multiple execution units, able to run several independent instructions in the same cycle. |
| **Out-of-order execution** | Running ready instructions before earlier stalled ones, then reassembling correct order — hides memory-wait time. |
| **ILP (Instruction-Level Parallelism)** | How much independent work the core can run in parallel. |
| **Dependency chain** | A sequence of instructions each needing the previous one's result; cannot be parallelized; the true limiter of hot-path speed. |
| **Branch** | A point where the next instruction depends on a not-yet-computed condition (`if`, loop, `switch`). |
| **Branch predictor** | Hardware that guesses a branch's outcome so the pipeline can speculatively run ahead. |
| **Branch misprediction** | A wrong guess; speculative work is flushed and the pipeline refills — ~15–20 cycles (~5–7 ns) plus jitter. |
| **Branchless** | Restructuring code to avoid a branch entirely (compute both, select arithmetically), eliminating mispredict risk. |
| **Virtual memory / page** | Each program's private address space, mapped to physical RAM in fixed chunks (pages, usually 4 KB). |
| **TLB (Translation Lookaside Buffer)** | A small cache of virtual→physical address translations. A miss forces a slow table walk before the data fetch. |
| **Huge pages** | Large (2 MB/1 GB) pages so each TLB entry covers far more memory, eliminating TLB stalls on large working sets. |
| **NUMA** | Non-Uniform Memory Access: on multi-socket machines, local memory is faster than memory attached to another socket. |
| **NUMA node** | A socket plus its directly-attached memory. |
| **Cache coherence** | Hardware protocol keeping all cores' cached copies of a memory location consistent — necessary but costly when data is shared/written. |
| **False sharing** | Unrelated variables sharing one cache line, causing ruinous inter-core ping-ponging despite no logical sharing. Fixed by padding/alignment. |
| **SMT / Hyper-Threading** | One physical core presenting as two logical cores that share its resources; boosts throughput but adds latency variance. HFT usually disables it. |
| **Performance counters** | CPU hardware registers counting cache misses, branch mispredicts, TLB misses, etc.; read via tools like `perf` to find stalls. |
| **Jitter** | Variance in latency — the rare slow case. HFT minimizes it as aggressively as average latency. |

---

## 16. Where to go next

You now hold the mental model the rest of the course is built on. The natural next steps:

- **Lesson 2 — Modern Assembly.** See *exactly* what your C++ becomes, and learn to read the compiler's output so you can spot the dependency chains (§6), the branches (§7), and the memory accesses (§3) this lesson described — in the actual instructions. Architecture tells you what costs; assembly shows you where it is.
- **Lesson 3 — Hardware.** Zoom out from the core to the NIC, PCIe, and the box, where the packet's journey begins and ends — and where NUMA (§9) and BIOS tuning (hyper-threading, §11) get decided physically.
- **Lesson 7 — C++ Data Structures.** The first cash-in: *why* contiguous containers (`std::vector`, `std::array`) beat pointer-chasing ones (`std::list`, `std::map`) is exactly the cache-line and prefetcher story of §§3–5, now made concrete container by container.
- **Lesson 8 — C++ for HFT.** The full payoff: every technique there (data-oriented layout, `alignas` against false sharing, branchless code, avoiding pointer-chasing containers) is a direct, line-by-line response to a concept in *this* lesson. When you read it, you'll recognize every rule's architectural reason.
- **Hands-on milestone (from the README):** write two versions of a sum-a-million-numbers program — one over a contiguous array, one over a linked list — time them, and *predict the ratio before you measure.* Then use `perf stat` to watch the cache-miss counts diverge. Feeling that 10×+ gap in your own hands, and seeing it in the counters, makes this entire lesson permanent.

> The deepest takeaway to carry forward: **you are not optimizing instructions; you are choreographing data movement so the core never has to wait.** Everything else is detail.
