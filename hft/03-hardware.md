# Lesson 3 — Hardware: The Physical Machinery of a Trading Server

> [Lesson 1](01-computer-architecture.md) lived *inside* the CPU core — caches, pipelines, branches. [Lesson 2](02-modern-assembly.md) read the instructions that core executes. Now we zoom all the way out, to the *physical box* sitting in a rack and the *wire* running out of it, because the HFT latency race does not start or end in your code. It starts the instant a packet of market data physically arrives at your **network card**, and it ends the instant your order packet physically departs from it. *Everything* your software does — every cache hit, every instruction, every clever C++ trick from the later lessons — happens in the gap between those two hardware events. So the hardware around the CPU, and above all the network card, disproportionately decides who wins.
>
> This lesson assumes **no hardware-engineering background.** We build up the physical picture piece by piece: how a packet physically arrives, what a NIC really is and does, why specialized cards and FPGAs exist, how the NIC connects to the CPU, why *time itself* is a hardware problem in HFT, and why firms rent space inside the exchange's own building. By the end you'll be able to trace a packet's entire physical journey and name where every nanosecond accrues.

---

## 0. The frame that governs this whole lesson

Before any component, fix the picture that makes the rest matter.

> **The mental model: the NIC is the starting gate *and* the finish line of the race.** Imagine two trading firms watching the same price change. The exchange's computer sends out a packet announcing the new price. The race clock starts the moment that packet's electrical (or light) signal reaches each firm's network card. It stops the moment each firm's *response* — an order packet — leaves its network card heading back. Whoever's order reaches the exchange's matching engine first wins the trade. There is no prize for second place.

Two consequences flow from this and shape everything below:

1. **Software latency is only the *middle* of the race.** The packet has to physically travel from the exchange to you (distance — §7), get from the wire into your CPU's memory (the NIC and PCIe — §1–§5), be processed (Lessons 1–2, 4–7), and then make the whole trip back. The fastest code in the world loses if your cable to the exchange is longer than your competitor's, or if your NIC takes microseconds to hand the packet to software.

2. **The "tick-to-trade" path is a hardware-software pipeline, and you optimize all of it.** HFT engineers obsess over the *whole* journey, not just the C++. This lesson is the hardware half of that journey. Later lessons (4–7) are the software half; kernel bypass (Lesson 6) is specifically about shortening the handoff *between* hardware and software.

Keep the starting-gate/finish-line image in mind. Every component we meet is judged by one question: **how many nanoseconds does it add between the gate and the finish?**

---

## 1. What a network packet physically is, and how it arrives

Let's start from absolute basics, because the NIC only makes sense once you know what it's handling.

When the exchange wants to tell the world "the price of instrument X is now 100.05," it doesn't send a continuous stream — it sends a **packet**: a finite bundle of bytes with a header (addressing and bookkeeping info) and a payload (the actual data). That packet travels as a physical signal — pulses of light through a fiber-optic cable, or electrical signals over copper — across the network to your building, through switches (§6), and finally arrives at your server's **network card.**

> **The analogy: a packet is a letter, and the NIC is your building's mailroom.** The letter arrives at the loading dock as a physical thing (the signal on the wire). Someone has to receive it, check it's addressed to you, open the envelope, and get the contents to the right desk inside the building (your CPU and its memory). The mailroom — the NIC — is where the physical world of signals-on-wires becomes the digital world of bytes-in-memory. How fast and how *predictably* that mailroom operates is the first link in the latency chain.

The signal itself travels astonishingly fast — near the speed of light — but "fast" is relative when you're counting nanoseconds, which is exactly why physical distance becomes a competitive variable (§7). For now, hold the picture: a physical signal arrives at a physical card, and that card's job is to turn it into bytes your software can use.

---

## 2. The NIC: the most important component in the box

The **NIC — Network Interface Card** (also called a network adapter) — is the hardware that connects your server to the network. Every computer has one. But in HFT it's the single most scrutinized, most specialized, most expensive-per-unit component, because — per §0 — it's the gate and the finish line. Let's open it up and see what it actually does, building the vocabulary you'll need for kernel bypass (Lesson 6).

### What the NIC does, step by step

When a packet's signal arrives at the NIC, a sequence of physical events unfolds:

1. **The PHY and MAC.** The NIC's **PHY** (physical layer chip) converts the incoming light/electrical signal into raw bits. The **MAC** (Media Access Control) block assembles those bits into a **frame** (the link-layer packet), checks it's addressed to this machine, and validates it (a checksum catches corruption). This is pure hardware, nanoseconds, before software is involved at all.

2. **DMA into host memory.** Here's the crucial part. The NIC does not hand the packet to the CPU directly. Instead, it uses **DMA — Direct Memory Access** — to *write the packet's bytes straight into the server's main memory (RAM)* by itself, without bothering the CPU. The NIC and the OS pre-arrange a set of memory buffers (a **ring buffer** — see below) where incoming packets get deposited. DMA is the NIC acting as its own delivery driver: it carries the letter from the loading dock and drops it directly onto a designated shelf in memory, no CPU involvement in the carrying.

3. **Notifying the CPU: interrupt vs. polling.** Now the packet is sitting in RAM — but how does the software *find out* it arrived? Two fundamentally different answers, and the choice is central to HFT:

   - **Interrupts (the normal way):** the NIC raises an **interrupt** — an electrical signal that yanks the CPU away from whatever it's doing, forcing it to stop, save its state, run a special handler to process the new packet, and then resume. It's a doorbell: the mailroom rings a bell, and you drop everything to go answer it.
   - **Polling / busy-polling (the HFT way):** the software *doesn't wait to be told.* A CPU core runs a tight loop constantly asking the NIC, "Anything new? Anything new? Anything new?" — checking the ring buffer over and over, as fast as it can. No bell, no interruption: you stand at the mailroom window staring at the shelf, grabbing each letter the instant it lands.

### Why HFT chooses polling — and burns a whole core to do it

This trade-off is so important it defines HFT's relationship with the NIC, so let's reason it out carefully.

**Interrupts are efficient but slow and jittery.** Efficient because the CPU is free to do other work (or sleep, saving power) until a packet actually arrives — great for a normal server handling sporadic traffic. But each interrupt costs real time: the CPU must stop, save its registers, jump to the handler (polluting its caches and pipeline — Lesson 1!), and later restore everything. That's hundreds of nanoseconds to *microseconds* of overhead and, worse, it's *variable* — the delay depends on what the CPU was doing and whether other interrupts are queued. Variable delay is **jitter**, the thing HFT hates most (Module 0).

**Polling is wasteful but fast and predictable.** Wasteful because a CPU core spins at 100% doing nothing but asking "anything yet?" — burning electricity and an entire core even when no packets arrive. But when a packet *does* land, the polling loop notices it almost instantly — no interrupt overhead, no context save/restore, no cache pollution — and the time-to-notice is *consistent* every single time. Low latency *and* low jitter.

> **The HFT verdict: burn the core.** A modern server has many cores; dedicating one (or a few) to spin forever polling the NIC is a trivial price for shaving microseconds and eliminating jitter off the most latency-critical step in the system. So HFT systems **busy-poll**: a pinned, isolated core (you'll set this up in [Lesson 4](README.md)) does nothing but watch the NIC's ring buffer. This single decision ripples through the whole architecture — it's *why* core isolation (Lesson 4) and kernel bypass (Lesson 6) exist. Remember this reasoning; it recurs constantly.

### The ring buffer

One more piece of vocabulary, because it appears everywhere. The NIC and software communicate through a **ring buffer** (or "descriptor ring"): a fixed-size circular array of slots in memory. The NIC fills slots with incoming packets as it receives them (advancing a "head" pointer around the ring), and the software consumes them (advancing a "tail" pointer). It's circular — when you reach the end, you wrap around to the start — so it never needs reallocation. 

> **The analogy: a lazy-Susan of mail slots.** The NIC drops letters into slots going clockwise; you pick them up following behind. As long as you keep up, there's always an empty slot ahead for the next letter. If you fall behind and the NIC catches up to you (the ring is full), incoming packets get **dropped** — a real failure mode in market data (Lesson 5 covers recovering from drops). The ring buffer is the fundamental handshake between network hardware and software, and you'll meet its software cousin — the SPSC ring buffer — in Lesson 8.

---

## 3. Specialized low-latency NICs and kernel-bypass hardware

A normal NIC works fine with the operating system's built-in networking. But recall (from §2, and the whole point of Lesson 6) that going *through the OS* to get a packet is slow. HFT uses **specialized NICs** designed from the ground up to let software talk to the card *directly*, bypassing the operating system's network stack. You don't need to memorize vendors, but you should know the landscape and what makes these cards different.

- **Kernel-bypass-capable cards.** The defining feature: they ship with libraries that let your application read and write packets *directly* to the NIC's ring buffers in user space, skipping the kernel entirely (the full story is Lesson 6). The historically dominant names in HFT are **Solarflare** (now part of AMD, famous for its *Onload* and *ef_vi* kernel-bypass software) and **Mellanox** (now NVIDIA, famous for high-speed cards and RDMA — Lesson 6). When someone says "a Solarflare card," they mean "a NIC built for kernel bypass."

- **Onboard timestamping.** These cards stamp each packet with a precise arrival time *in hardware*, the instant it hits the card — far more accurate than asking software "what time is it now?" after the packet has already wound through the system. This is essential for honest latency measurement (§5 below) and for regulatory time-stamping (Lesson 9).

- **Hardware offloads — a double-edged sword.** NICs can perform certain work *on the card* to relieve the CPU: computing checksums, splitting large data into packets (segmentation offload), distributing incoming packets across multiple CPU cores (receive-side scaling). For a throughput-oriented server these help. But for HFT, some offloads *add latency or jitter* (they batch work, or add a processing step), so HFT often **disables** them to keep the path short and predictable. This is a recurring theme you'll see again in Lesson 5: features built for throughput are frequently the enemy of latency, and tuning means turning them off.

> **The takeaway:** an HFT NIC is chosen and configured for *one* thing — getting a single packet from the wire into your software, and back, with the fewest and most predictable nanoseconds. Throughput, CPU efficiency, and convenience — the things normal NICs optimize — are all secondary or actively unwanted.

---

## 4. FPGAs: putting the logic *in the hardware*

Now the frontier of HFT speed, and the most important hardware concept beyond the NIC. So far the picture is: NIC receives packet → software (on the CPU) decides what to do → NIC sends response. But every step through software — even perfectly tuned, kernel-bypassed, cache-warm software — costs *some* time: hundreds of nanoseconds at least. What if you could make the decision **without the CPU being involved at all**, right there in the network hardware, in *tens* of nanoseconds?

That's what an **FPGA** does. Let's build up what it is, because it's genuinely different from a CPU.

### What an FPGA is

A **CPU** is a *fixed* chip that executes a *program* — a list of instructions (Lesson 2) it reads and runs, one after another (well, cleverly overlapped — Lesson 1, but conceptually sequential). Its hardware never changes; only the software does.

An **FPGA — Field-Programmable Gate Array** — is a chip whose *internal circuitry itself can be reconfigured.* Instead of writing a program that the chip *runs*, you describe a *digital circuit* (using a hardware description language like Verilog or VHDL), and the FPGA physically *becomes* that circuit — wiring up its array of logic gates to implement exactly the logic you specified. 

> **The analogy: software on a CPU is a chef following a recipe step by step; an FPGA is a custom-built machine on the factory floor that does one specific thing in a single motion.** The chef (CPU) is flexible — hand him any recipe and he'll follow it — but he works through the steps one at a time. The custom machine (FPGA) can't do anything *else*, but for the one task it was *built* for, it's incomparably faster, because the task is wired directly into its physical structure. There are no "instructions to fetch and execute" — the data flows through purpose-built circuitry and the answer falls out the other end, massively in parallel.

### Why FPGAs matter in HFT

Because that custom circuit can be wired *directly to the network*, an FPGA can implement the entire decision loop — receive a market-data packet, parse it, check it against a trading rule, and emit an order packet — **in the hardware itself, without the data ever reaching the CPU.** This is called **tick-to-trade in hardware**, and it operates in the realm of **tens of nanoseconds** — an order of magnitude faster than even the best software path. For the most latency-sensitive, simplest strategies (e.g., "if the price crosses this level, fire this order"), nothing in software can compete.

The trade-offs, which is why FPGAs haven't replaced software:

- **They're hard to develop for.** Describing circuits in Verilog/VHDL is a specialized skill, development is slow, and debugging is painful compared to C++.
- **They're inflexible.** Complex, frequently-changing strategy logic is far easier to express and iterate in software. Reconfiguring an FPGA is heavyweight.
- **The common pattern is hybrid.** Many firms put the *simplest, most latency-critical, most stable* logic on the FPGA (fast-path filtering, risk checks, the simplest strategies) and keep the *complex, evolving* logic in C++ on the CPU. The FPGA handles the nanosecond-critical common case; the CPU handles everything else.

> **What you need as a software engineer starting out:** you almost certainly won't write FPGA code at first — that's a distinct discipline. But you *must* understand what FPGAs are *for*, where the software/hardware boundary sits, and that the existence of FPGA competitors sets the latency bar you're measured against. The spectrum runs from "all software on the CPU" → "kernel-bypassed software" (Lesson 6) → "logic in the FPGA" — a continuum of *how close to the wire the decision gets made.* This lesson's job is to make sure you know that spectrum exists and where each option sits.

---

## 5. PCIe and clocks: connecting the NIC, and knowing *when*

Two more hardware topics that quietly cost nanoseconds and that HFT engineers actually tune.

### PCIe: the bus between the NIC and the CPU

The NIC is a card; the CPU and memory are elsewhere on the motherboard. They're connected by **PCIe — Peripheral Component Interconnect Express** — the high-speed bus that links expansion cards (NICs, GPUs, FPGAs) to the CPU and memory. When the NIC DMAs a packet into RAM (§2), it travels over PCIe. PCIe has its own latency (the trip across the bus) and its own topology that matters:

- **Which slot, attached to which CPU.** On a multi-socket server (recall **NUMA** from [Lesson 1 §9](01-computer-architecture.md)), the PCIe slots are physically wired to *specific* CPU sockets. A NIC plugged into a slot wired to socket 0 delivers its packets into socket 0's local memory fastest. If your packet-processing thread runs on socket 1, every packet has to cross the inter-socket link — a silent latency tax. **So you place the NIC, the polling thread, and its memory all on the *same* NUMA node.** This is a concrete, physical reason the NUMA discipline of Lesson 1 and the pinning of Lesson 4 matter: the hardware layout of the box dictates where your software should run.
- **PCIe generation and lanes** affect bandwidth and latency, and HFT cares, but the topology point above is the one to internalize first.

> **The connection across lessons:** Lesson 1 told you *local memory is faster than remote*; this lesson tells you *the NIC physically attaches to one socket's memory*; Lesson 4 will tell you *how to pin your thread there.* Three lessons, one physical fact: keep the packet, the memory, and the code on the same NUMA node.

### Clocks and time synchronization: *when* is a hardware problem

Here's something that surprises newcomers: in HFT, **knowing precisely what time it is — to the nanosecond — is a hard hardware problem that firms spend real money solving.** Why does time matter so much?

- **To measure your own latency (§0, the whole game).** You can't optimize tick-to-trade if you can't measure it precisely. That requires accurately timestamping when a packet arrived and when your order left — and (§3) the best timestamps come from the NIC *hardware*, the instant the packet crosses the card.
- **To order events correctly.** When data arrives from multiple sources, you need a consistent, precise clock to know what happened first.
- **For regulation (Lesson 9).** Rules like Europe's MiFID II legally *mandate* that trading firms synchronize their clocks to within a tiny tolerance of official time and timestamp events accordingly. This isn't optional; it's the law.

The hardware and protocols that solve this:

- **PTP — Precision Time Protocol.** A protocol (with hardware support in the NIC and network switches) that synchronizes clocks across machines to **sub-microsecond, often nanosecond, accuracy** — vastly better than the ordinary internet time protocol (NTP), which is millisecond-grade. PTP works by exchanging precisely hardware-timestamped messages to measure and cancel out network delay.
- **GPS-disciplined clocks and PPS.** Many firms anchor their time to a GPS receiver (GPS satellites carry atomic-clock time) feeding a **PPS — Pulse Per Second** — signal, an extremely precise once-a-second electrical tick that disciplines the local clock so it never drifts.
- **Hardware timestamping in the NIC.** As in §3 — the card stamps each packet's arrival/departure time itself, the moment it crosses the wire, removing all the software delay that would corrupt a software timestamp.

> **The mental model: HFT runs on a shared, ultra-precise sense of time, anchored to the heavens (GPS) and distributed by special protocols (PTP).** Without it, you couldn't measure who won the race, couldn't reconstruct what happened, and couldn't satisfy the regulator. This connects directly to [Lesson 2 §10](02-modern-assembly.md): `rdtsc` gave you cycle-accurate timing *within* one CPU; PTP and hardware timestamps give you accurate, *agreed* time *across* machines and against the outside world.

---

## 6. The network between you and the exchange: switches

Your packet doesn't go straight from the exchange to your NIC — it passes through **network switches**, the devices that forward packets between machines in a network. Switches add latency too, and HFT cares about which kind you use:

- **Store-and-forward switches** receive the *entire* packet, check it for errors, then forward it. Safe, but the whole packet must arrive before any of it leaves — latency proportional to packet size.
- **Cut-through switches** start forwarding a packet *as soon as they've read its destination address* (the first few bytes), before the rest has even arrived. Far lower latency, at the cost of occasionally forwarding a corrupt packet. **HFT uses ultra-low-latency cut-through switches**, where the switch's added delay is measured in *tens of nanoseconds* and is a published, competed-on specification.

> **The point:** even the boxes *between* your server and the exchange are latency-engineered hardware, chosen and measured in nanoseconds. The whole physical path — NIC, cable, switch, cable, exchange — is a latency budget, and HFT firms optimize every line item.

---

## 7. Colocation and the physics of distance

Now the most striking hardware fact in all of HFT, and the one that sounds like a joke until you do the arithmetic: **the physical length of your cable to the exchange is a competitive variable, because the speed of light is finite.**

Recall from [Lesson 1 §0](01-computer-architecture.md): light travels about **30 centimeters (one foot) per nanosecond.** In a fiber-optic cable, signals travel even *slower* — only about two-thirds the speed of light in vacuum — because light is slower in glass. So:

- A signal traveling **300 meters** of fiber takes roughly **1.5 microseconds** one way. That's an eternity in HFT terms — enough time to lose thousands of races.
- If your server is **100 meters farther** from the exchange's matching engine than a competitor's, you are *permanently, physically* about half a microsecond behind on every single message, no matter how fast your code is. You cannot out-program the speed of light.

This drives the defining practice of HFT infrastructure:

### Colocation ("colo")

Exchanges *rent out rack space inside their own data centers*, right next to the matching engine. This is **colocation** — putting your servers physically in the same building as the exchange's computers, sometimes meters away. Every firm that colocates is reduced to the same tiny cable distance, so the exchanges go further: they provide **equal-length cabling** to every customer's rack (deliberately coiling extra cable for closer racks!) so that *no* colocated firm has a distance advantage over another. At that point the race returns to who has the faster NIC, FPGA, and code — which is why everything else in this course matters.

### Between data centers: microwave and lasers

When firms need to send data *between* distant locations — say, between exchanges in different cities (Chicago and New York is the classic example) — distance dominates, and here's a beautiful consequence of §7's physics: **fiber-optic cable is too slow, not because of bandwidth but because light is slower in glass.** So firms build **microwave** and **laser** networks that send signals *through the air* (where they travel at nearly the full speed of light) in a straighter line than cable can run. A microwave link between Chicago and New York beats the best fiber route by milliseconds — and firms pay fortunes for a few feet of antenna height or a slightly straighter path.

> **The mental model: HFT is partly a physics and real-estate business.** A meaningful chunk of the latency game is won or lost *before any software runs*, in decisions about where your server physically sits and how the signal physically travels. As a software engineer you won't run the cabling — but you must understand that your code operates inside a physical latency budget set by the speed of light, and that the firm has already spent heavily to minimize the parts of that budget you *can't* touch. Your job is the part you *can.*

---

## 8. Server and firmware tuning: the BIOS checklist

Finally, the box itself must be configured to be a low-jitter machine, and much of this happens *below* the operating system, in the **BIOS/UEFI firmware** — the low-level software that configures the hardware before the OS even boots. These are real settings HFT engineers own, and each follows directly from concepts you've already learned:

- **Disable CPU power-saving (C-states).** Modern CPUs save power by putting idle cores to *sleep* in progressively deeper "C-states," and waking a sleeping core takes time — *microseconds* of variable delay. For a core that's busy-polling a NIC (§2), you never want it to sleep, and you certainly never want a wake-up delay on the hot path. So HFT **disables deep C-states**, keeping cores always awake and instantly responsive. (You pay in electricity and heat; you gain predictability.)
- **Disable frequency scaling (P-states / "turbo" variability).** CPUs normally vary their clock speed up and down to save power. But a varying clock means varying performance — *jitter* — and it corrupts cycle-based timing (the `rdtsc` caveat from [Lesson 2 §10](02-modern-assembly.md)). HFT pins the CPU at a fixed, maximum frequency so performance is constant and predictable.
- **Disable Hyper-Threading / SMT.** As [Lesson 1 §11](01-computer-architecture.md) explained in detail: SMT makes two logical threads share one physical core's caches and execution units, injecting interference and variance. HFT disables it in the BIOS so each physical core is dedicated, uncontended, and predictable.
- **Configure NUMA and memory settings** so the topology (§5) is exposed correctly to the OS, letting you pin threads and memory to the right node (Lesson 4).
- **Disable unnecessary devices and interrupt sources** that could otherwise fire interrupts and perturb the hot cores.

> **The unifying theme of the checklist: trade efficiency for predictability.** Every default the BIOS ships with — sleep when idle, slow down to save power, share cores for throughput — optimizes for the *average* machine's efficiency. HFT overrides every one of them, accepting higher power draw and lower throughput in exchange for a machine whose performance is *constant and predictable to the nanosecond.* This is the same trade you'll make again in software (Lesson 4): **HFT systematically chooses determinism over efficiency at every layer.**

---

## 9. Putting it together: the physical journey of a packet

Let's assemble the whole lesson into the complete physical path of one market-data update and the order it triggers — the hardware skeleton of the "tick-to-trade" path that the rest of the course fleshes out:

1. **The exchange's matching engine** decides the price changed and emits a packet (§1).
2. The packet travels as light through fiber, the distance minimized because your server is **colocated** in the exchange's building (§7), through an **ultra-low-latency cut-through switch** (§6) that began forwarding it after reading just the address.
3. The signal **arrives at your NIC** — *the race clock starts* (§0). The NIC's PHY/MAC turn the signal into a frame and validate it; the NIC **hardware-timestamps** the arrival (§3, §5).
4. The NIC **DMAs the packet directly into RAM** — specifically into a ring buffer in the memory local to the NUMA node the NIC is attached to (§2, §5), no CPU involvement in the transfer.
5. Your **busy-polling thread**, pinned to an isolated core on *that same NUMA node* (§2, §5, and Lesson 4), running on a CPU kept awake and at fixed frequency by the **BIOS tuning** (§8), notices the new packet almost instantly — no interrupt, no jitter.
6. From here the **software** takes over (Lessons 4–7): parse, decide, risk-check — all in cache-warm, branch-predictable C++ (Lessons 1–2, 7) — having gotten the packet into user space via **kernel bypass** on a specialized NIC (§3, and Lesson 6).
7. The order packet is handed back to the NIC (again bypassing the kernel), **DMA'd out**, and leaves the card — *the race clock stops* (§0) — back through the switch, down the equal-length cable, to the exchange.
8. The whole round trip is **measured precisely** using NIC hardware timestamps and a PTP/GPS-disciplined clock (§5), reported as a latency distribution (the tail, not the mean — Lesson 2 §10).

   *Or* — for the simplest, most latency-critical strategies — steps 4–7 happen *entirely inside an FPGA* wired to the NIC (§4), and the CPU never sees the packet at all: tick-to-trade in tens of nanoseconds.

> **The unifying insight:** the packet spends a great deal of its journey in *hardware and on wires*, not in your code. The hardware decisions — which NIC, plugged into which slot, on which NUMA node, in a colocated rack, behind a cut-through switch, on a BIOS-tuned box, possibly with an FPGA — set the *budget* your software operates within. **You can't win the race in software alone, but you can certainly lose it there** — and you can only reason about your software's share of the latency once you understand the hardware path it lives inside. That's what this lesson gave you.

---

## 10. Recap

- The **NIC is the starting gate and finish line**; software latency is only the *middle* of a physical round trip. Judge every component by the nanoseconds it adds between gate and finish. (§0)
- A **packet** is a bundle of bytes arriving as a physical signal; the **NIC** is the mailroom turning signals into bytes-in-memory. (§1)
- The NIC uses **DMA** to write packets straight into RAM ring buffers, then notifies software via **interrupts** (efficient, slow, jittery) or **polling** (wasteful, fast, predictable). **HFT burns a core to busy-poll** — the decision that drives core isolation and kernel bypass. (§2)
- **Specialized NICs** (Solarflare/AMD, Mellanox/NVIDIA) exist for **kernel bypass** and **hardware timestamping**; some **offloads** help throughput but hurt latency and are disabled. (§3)
- **FPGAs** are reconfigurable chips that *become* a custom circuit, enabling **tick-to-trade in hardware** in tens of nanoseconds. Hard to develop, inflexible — used hybrid: simplest/critical logic in the FPGA, complex logic in C++. Know the software→bypass→FPGA spectrum. (§4)
- **PCIe** connects NIC to CPU and is **NUMA-attached** — keep NIC, memory, and thread on the same node. **Time** is a hardware problem solved by **PTP**, **GPS/PPS**, and **NIC hardware timestamps** — for measurement *and* regulation. (§5)
- **Cut-through switches** forward after reading just the address — tens of nanoseconds vs. store-and-forward. The whole path is a latency budget. (§6)
- **Colocation** puts your server in the exchange's building (with equal-length cabling) because **the speed of light makes distance a competitive variable**; **microwave/laser** links beat fiber between cities because light is slower in glass. HFT is partly a physics/real-estate business. (§7)
- **BIOS tuning** — disable C-states, fix the frequency, disable Hyper-Threading, set up NUMA, cut stray interrupts — systematically **trades efficiency for predictability**, the same trade made at every HFT layer. (§8)
- The full **physical journey** of a packet shows hardware sets the *budget* your software lives within: you can't win in software alone, but you can lose there. (§9)

---

## 11. Glossary

| Term | Meaning |
| --- | --- |
| **NIC** | Network Interface Card — the hardware connecting a server to the network; in HFT, the most scrutinized component (race start/finish). |
| **Packet / frame** | A finite bundle of bytes (header + payload) sent over the network; "frame" is the link-layer form the NIC handles. |
| **PHY / MAC** | NIC blocks that convert signal↔bits (PHY) and assemble/validate frames (MAC). |
| **DMA (Direct Memory Access)** | The NIC writing packets straight into RAM by itself, without using the CPU to carry the data. |
| **Ring buffer / descriptor ring** | A fixed-size circular array of slots where the NIC deposits packets and software consumes them; overflow means dropped packets. |
| **Interrupt** | An electrical signal forcing the CPU to stop and handle an event (e.g., packet arrival). Efficient but adds latency and **jitter**. |
| **Polling / busy-polling** | A core spinning in a loop checking the NIC constantly instead of waiting for an interrupt. Wasteful but fast and predictable — the HFT choice. |
| **Kernel-bypass NIC** | A NIC (e.g., Solarflare, Mellanox) whose libraries let software access its ring buffers directly in user space, skipping the OS (Lesson 6). |
| **Hardware timestamping** | The NIC recording each packet's precise arrival/departure time on the card itself, for accurate measurement and regulation. |
| **Offload** | Work done on the NIC (checksums, segmentation, packet steering) to relieve the CPU; helps throughput but often hurts HFT latency, so frequently disabled. |
| **FPGA** | Field-Programmable Gate Array — a chip whose circuitry is reconfigured to *become* custom logic; enables tick-to-trade in hardware (tens of ns). |
| **Verilog / VHDL** | Hardware description languages used to specify the circuit an FPGA becomes. |
| **Tick-to-trade** | The end-to-end time from receiving a market-data update to sending an order; the number HFT competes on. |
| **PCIe** | The high-speed bus connecting expansion cards (NIC, FPGA) to the CPU and memory; NUMA-attached to specific sockets. |
| **NUMA node** | A CPU socket plus its directly-attached memory and PCIe slots; keep NIC, memory, and thread on the same one. |
| **PTP (Precision Time Protocol)** | A protocol with NIC/switch hardware support that synchronizes clocks across machines to sub-microsecond/nanosecond accuracy. |
| **NTP** | The ordinary internet time protocol — millisecond-grade, too coarse for HFT. |
| **GPS-disciplined clock / PPS** | Anchoring local time to GPS satellites' atomic-clock time via a precise Pulse-Per-Second signal so the clock never drifts. |
| **Switch (cut-through vs. store-and-forward)** | Network forwarding device; cut-through forwards after reading just the address (low latency, HFT choice) vs. store-and-forward (reads whole packet first). |
| **Colocation ("colo")** | Renting rack space inside the exchange's data center to minimize physical distance (and thus latency) to the matching engine; exchanges equalize cable lengths. |
| **Microwave / laser links** | Through-the-air networks between distant sites, faster than fiber because light travels faster in air than in glass. |
| **BIOS/UEFI** | Low-level firmware configuring the hardware before the OS boots; where C-states, frequency, and SMT are turned off for HFT. |
| **C-states / P-states** | CPU power-saving sleep levels (C) and frequency/voltage levels (P); disabled or fixed in HFT to avoid wake-up delays and clock variability (jitter). |

---

## 12. Where to go next

- **Lesson 4 — OS / Linux Fundamentals.** The natural sequel: you now know the box wants a busy-polling thread pinned to an isolated core on the NIC's NUMA node, with a CPU kept awake at fixed frequency. Lesson 4 is *how you actually make Linux do all that* — core isolation, CPU pinning, IRQ affinity, huge pages, and killing every remaining source of jitter the BIOS couldn't.
- **Lesson 5 — Networking.** What's actually *in* the packets the NIC receives (market-data protocols, UDP multicast) and how the standard OS network stack — the thing the specialized NICs of §3 let you bypass — processes them, and at what cost.
- **Lesson 6 — Kernel Bypass.** The direct payoff of §2–§3: the software techniques (DPDK, Onload, AF_XDP, RDMA) that let your busy-polling thread read the NIC's ring buffers *directly*, skipping the OS entirely — the single biggest software latency win, made possible by the hardware in this lesson.
- **Hands-on milestone (from the README):** you can't easily get a colo rack or a Solarflare card to play with, but you *can* explore the time layer — read your machine's clock-source settings, look at how `ethtool` reports your NIC's features and timestamping capabilities, and read about a real exchange's colocation and connectivity offering (most publish them). Seeing a real exchange's published switch latencies and cabling policy makes §6–§7 concrete.

> The through-line from Lessons 1–3: **Lesson 1 was the core, Lesson 2 was its instructions, and Lesson 3 is the world the core lives in.** Together they define the *physical and architectural budget* your software gets to spend. Now we turn to spending it well — starting with making the operating system get out of the way.
