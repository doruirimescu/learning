# Lesson 2 — Modern Assembly: Reading What Your Code Actually Becomes

> [Lesson 1](01-computer-architecture.md) built the mental model of the machine: computing is nearly free, *moving data* is the expense, and the core stalls on cache misses, branch mispredicts, and dependency chains. But that lesson described costs in the abstract. This lesson makes them *visible*. When you compile a line of C++, the compiler translates it into a list of the CPU's native instructions — **assembly** — and that list is the *ground truth* of what the machine will do. The gap between "what I wrote" and "what actually runs" is enormous, full of surprises, and is exactly where HFT latency hides and where performance myths go to die.
>
> Crucially: **you will almost never *write* assembly in an HFT job. You will constantly *read* it.** This lesson is about reading and reasoning — turning the abstract costs of Lesson 1 into things you can see in the actual instructions, confirm with your own eyes, and reason about in cycles. It assumes **no prior assembly knowledge.** We build it from nothing.

---

## 0. Why an HFT engineer reads assembly at all

Most programmers go their whole careers without reading a line of assembly, and they're fine. Why is it non-negotiable in HFT?

Because in HFT you make claims like "this version is faster because it avoids a branch" or "the compiler vectorized this loop" or "this abstraction is zero-overhead." **Every one of those claims is either true or false at the level of the emitted instructions — and the only way to *know* is to look.** The compiler is a sophisticated, aggressive, and occasionally surprising translator. It might:

- Turn your three lines of code into a single instruction (it saw through your logic).
- Turn your one elegant line into forty instructions (an abstraction leaked).
- Silently *fail* to apply the optimization you were counting on (something blocked it).
- Do exactly what you hoped — but you'd never know without checking.

> **The mental model: assembly is the receipt for your C++.** You hand the compiler a high-level *intent*; it hands the CPU an itemized list of operations. When you want to know whether your "optimization" actually changed anything — or whether the machine is doing what you think — **you read the receipt.** A surprising amount of HFT performance work is exactly this: change the C++, read the new receipt, confirm the instructions improved, measure to be sure.

There's a second reason, which connects straight to Lesson 1: the architectural costs that dominate HFT latency — dependency chains (§6 of Lesson 1), branches (§7), memory accesses (§3) — are *properties of the instruction stream*. You reasoned about them abstractly last lesson; here you learn to *find them in the assembly* and estimate their cost in cycles. Architecture tells you what's expensive; assembly shows you *where it is in your actual program.*

---

## 1. What assembly actually is: the layers from source to silicon

Let's establish exactly where assembly sits, because the terms get muddled.

The CPU does not understand C++. It does not even understand assembly. The only thing it physically executes is **machine code** — raw bytes, numbers that the hardware decodes directly into actions. Machine code is unreadable to humans (it's just hex like `48 01 f0`). 

**Assembly language** is a *thin, human-readable layer directly on top of machine code* — a one-to-(almost)-one textual representation of those raw bytes. Where machine code says `48 01 f0`, assembly says `add rax, rsi` ("add the register named rsi into the register named rax"). Each assembly instruction (a **mnemonic** like `add`, `mov`, `jmp`) corresponds to a specific machine-code instruction. Assembly is just machine code you can read.

Here's the full pipeline from your source to the running CPU:

```
your_code.cpp  ──(compiler: gcc/clang)──►  assembly (.s)  ──(assembler)──►  machine code (.o)  ──(linker)──►  executable  ──►  CPU executes
   C++                                       human-readable      raw bytes        program
```

The step that matters for us is the **first arrow**: the compiler translating C++ into assembly. *That's* where all the cleverness (and all the surprises) happen — inlining, optimization, vectorization, register allocation. The assembler step afterward is a nearly-mechanical text-to-bytes conversion. So when we say "read the assembly," we mean: look at what the *compiler* produced, because that reveals every decision it made about your code.

> **The key realization:** there is a real, inspectable artifact sitting between your C++ and the silicon, and it tells you the truth. The whole rest of this lesson is learning to read it.

One more term: **ISA (Instruction Set Architecture)** — the *vocabulary* of instructions a particular CPU family understands. The dominant ISA in HFT (and servers generally) is **x86-64** (also called AMD64 or x64), made by Intel and AMD. ARM's **AArch64** is growing in servers and matters increasingly, but x86-64 is the default and what we'll use throughout. The ISA defines what registers exist, what instructions exist, and what they do — it's the contract between the compiler and the chip.

---

## 2. The CPU's vocabulary: registers

Lesson 1 introduced **registers** as the tiny, instant-access storage slots *inside* the core where all computation happens. Now we name them, because reading assembly is largely tracking which values live in which registers.

x86-64 has **16 general-purpose registers**, each holding 64 bits (8 bytes). Their names are a historical accident worth knowing:

| 64-bit name | Conventional/typical use | Notes |
|-------------|--------------------------|-------|
| `rax` | Return values; accumulator | The "a" register |
| `rbx` | General | Callee-saved (see §4) |
| `rcx` | 4th function argument; loop counts | The "c" register |
| `rdx` | 3rd function argument | The "d" register |
| `rsi` | 2nd function argument; source pointer | "source index" |
| `rdi` | 1st function argument; dest pointer | "destination index" |
| `rbp` | Frame/base pointer | Callee-saved |
| `rsp` | **Stack pointer** — always points at the top of the stack | Special, never general |
| `r8`–`r15` | General (5th–6th args in r8/r9) | The "new" registers added with 64-bit |

Two quirks you'll see constantly:

- **Sub-register names.** Each register can be accessed at smaller widths under different names. `rax` is the full 64 bits; `eax` is its low **32 bits**; `ax` the low 16; `al` the low 8. So when you see `eax` in the assembly, it's just the 32-bit slice of `rax`. Why does this matter? Because a 32-bit `int` in C++ lives in `eax`, while a 64-bit `long` or a pointer lives in `rax` — the width in the assembly tells you the type's size. (Subtle rule: writing to a 32-bit register like `eax` *zeroes the upper 32 bits* of `rax`. You'll see compilers exploit this.)
- **Special registers.** `rsp` always points to the top of the **stack** (§4). `rip` is the **instruction pointer** — it holds the address of the next instruction to execute; jumps and branches work by changing `rip`. `rflags` holds status bits set by operations (e.g., "was the last result zero?", "did it overflow?") — branches read these (§7).

There are also the **SIMD registers** — `xmm0`–`xmm15` (128-bit), `ymm0`–`ymm15` (256-bit), and `zmm0`–`zmm31` (512-bit) — used for vector math and floating point. We'll meet them properly in §8.

> **The mental model: registers are the craftsman's hands and the few tools on his bench.** There are only sixteen of them. *Everything* the CPU computes on must first be loaded into a register; results are computed in registers and then stored back to memory. A huge part of the compiler's job (**register allocation**) is the puzzle of juggling your program's many variables through only sixteen slots — and when it runs out, it has to "spill" a value back to memory (the stack) and reload it later, which you'll see and recognize as a small cost.

---

## 3. The anatomy of an instruction

An assembly instruction is a **mnemonic** (the operation) followed by its **operands** (what it operates on). The simplest pattern:

```
add  rax, rsi        ; rax = rax + rsi
```

`add` is the operation; `rax` and `rsi` are the operands. Read it as "add rsi into rax." Operands can be: a **register** (`rax`), an **immediate** (a literal constant, like `5`), or a **memory location** (an address, written in brackets).

### A small dictionary of instructions you'll see everywhere

You don't need hundreds of instructions to read hot-path code. A couple dozen cover most of it:

| Instruction | Meaning |
|-------------|---------|
| `mov dst, src` | Copy `src` into `dst` (the workhorse — "move," really "copy") |
| `add` / `sub` | Add / subtract |
| `imul` / `mul` | Integer multiply (signed / unsigned) |
| `inc` / `dec` | Increment / decrement by 1 |
| `lea dst, [expr]` | "Load Effective Address" — compute an address expression and put the *result* in `dst` (often used as a fast arithmetic trick, not just for addresses) |
| `cmp a, b` | Compare (subtract `b` from `a`, set flags, discard result) |
| `test a, b` | Bitwise-AND `a` and `b`, set flags, discard result (often `test rax, rax` to check "is rax zero?") |
| `jmp label` | Unconditional jump |
| `je`/`jne`/`jl`/`jg`/`jle`/`jge` | Conditional jumps ("jump if equal / not-equal / less / greater / ..."), reading the flags set by a preceding `cmp`/`test` — **these are branches** (§7) |
| `call label` | Call a function (push return address, jump) |
| `ret` | Return from a function |
| `push` / `pop` | Push/pop a value onto/off the stack |
| `xor rax, rax` | Bitwise-XOR rax with itself = **set rax to zero** (the idiomatic, fastest way to zero a register — you'll see it constantly) |

### Reading memory operands (the addressing modes)

A memory operand in brackets can contain an *address calculation*, and this is where the connection to Lesson 1's data layout becomes visible. The general form is `[base + index*scale + displacement]`:

```
mov eax, [rdi]                 ; load the 32-bit value at the address in rdi
mov eax, [rdi + 4]             ; load from address rdi+4
mov eax, [rdi + rsi*4]         ; load from rdi + rsi*4  — this is array[i]!
```

That last one is exactly how `array[i]` for an `int` array compiles: `rdi` holds the array's base address, `rsi` holds the index `i`, and `*4` is because each `int` is 4 bytes. **When you see `[base + index*scale]`, you're looking at an array access** — and recalling Lesson 1, you can now ask: is that access sequential (cache-friendly) or scattered (cache-missing)? The assembly shows you the access pattern directly.

### A note on syntax: Intel vs. AT&T

You'll encounter two notations for x86 assembly. **Intel syntax** (used by Compiler Explorer's default, Windows tools, and Intel's manuals) writes `mov dst, src` — destination first. **AT&T syntax** (the Linux/GNU default, what you get from `gcc -S` or `objdump`) writes `movl %src, %dst` — *source* first, with `%` on registers and `$` on immediates. Same machine code, different text. **This lesson uses Intel syntax because it's more readable**, but know that on a Linux box the default tools emit AT&T, so you'll either mentally reverse the operand order or pass `-masm=intel` / `objdump -M intel`. Don't let the reversed order trip you up — it's the number-one beginner confusion.

---

## 4. Reading a function: the stack and the calling convention

To read real code you must understand two things working together: the **stack** (where local data and the call chain live) and the **calling convention** (the agreed rules for how functions pass arguments and return values). Without these, function-related assembly is gibberish; with them, it reads like prose.

### The stack

The **stack** is a region of memory used for function calls. It grows *downward* (toward lower addresses — a historical convention), and `rsp` always points at its current top. When a function needs scratch space for local variables, it moves `rsp` down to "allocate" room; when it returns, it moves `rsp` back up to "free" it. Pushing and popping the call chain happens here too.

> **The mental model: the stack is a stack of plates, one per active function call.** When function A calls B, a new plate (B's **stack frame** — its local variables and bookkeeping) is placed on top. When B returns, its plate is removed and we're back to A's. The stack pointer `rsp` is your finger marking the top plate. This is why infinite recursion "blows the stack" — you keep adding plates until you run out of room.

### The calling convention (System V AMD64 ABI)

When function A calls function B, *how* does B receive its arguments and return its result? There must be a shared agreement, or compiled code from different places couldn't interoperate. On Linux x86-64, that agreement is the **System V AMD64 ABI** (ABI = Application Binary Interface). The rules you need:

- **Integer/pointer arguments** go in registers, in this order: **`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`.** A 7th argument and beyond go on the stack.
- **Floating-point arguments** go in `xmm0`–`xmm7`.
- **The return value** comes back in **`rax`** (and `rdx` for 128-bit returns; `xmm0` for floats).
- **Caller-saved vs. callee-saved registers.** Some registers (`rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`) may be freely clobbered by the called function — if the *caller* wanted to keep a value in one, it must save it first ("caller-saved"). Others (`rbx`, `rbp`, `r12`–`r15`) must be preserved by the called function — if B wants to use `rbx`, it must save and restore it ("callee-saved"). This is just a division of responsibility so both sides don't redundantly save everything.

Now a tiny function becomes completely readable. Here's `int add(int a, int b) { return a + b; }` compiled with optimization:

```
add:
    lea eax, [rdi + rsi]    ; eax = rdi + rsi   (a is in edi, b in esi; result in eax)
    ret                     ; return (value is in eax/rax, per the ABI)
```

You can read every byte of intent: argument `a` arrived in `edi` (the 32-bit slice of `rdi`, because it's an `int`), `b` in `esi`, the compiler used the `lea` trick to add them into `eax` in one instruction, and `ret` hands `eax` back as the return value. **Two instructions. No stack frame at all** — the function was simple enough that the compiler never touched the stack. That's the kind of thing you confirm by reading the receipt.

### Prologue and epilogue

More complex functions that need stack space begin with a **prologue** and end with an **epilogue** — bookkeeping you'll learn to skim past:

```
push rbp            ; prologue: save caller's frame pointer
mov  rbp, rsp       ; set up our frame pointer
sub  rsp, 32        ; allocate 32 bytes of local stack space
... function body ...
leave               ; epilogue: undo the frame (mov rsp,rbp; pop rbp)
ret
```

When you read assembly, **mentally skip the prologue/epilogue** and focus on the body — that's where the work (and the cost) is. But notice their existence tells you something: a function with a big `sub rsp, N` is using a lot of stack (maybe large locals, or the compiler spilled registers). A function with *no* prologue is a "leaf" function the compiler kept entirely in registers — cheaper.

> **Why this matters for HFT:** function calls aren't free. Each `call`/`ret`, each argument shuffle, each prologue/epilogue costs instructions and can disrupt the pipeline. This is precisely *why* HFT leans on **inlining** (§6) — eliminating the call entirely by pasting the function's body into the caller. When you read assembly and see that a function call *vanished* (the callee's instructions appear inline with no `call`), that's the optimizer doing exactly what you want.

---

## 5. Compiler Explorer: your daily instrument

The tool that makes all of this a practical daily habit rather than an arcane chore is **Compiler Explorer**, at **godbolt.org** (named after its creator, Matt Godbolt). It's a website (and can be self-hosted) with a brilliant premise: type C++ on the left, see the **assembly the compiler produces on the right, instantly, with the source lines color-matched to the instructions they generated.**

This single tool changes how you work. You no longer *wonder* whether the compiler inlined your function, vectorized your loop, or saw through your abstraction — you type it in, glance right, and *know.* You can switch compilers (gcc, clang, MSVC), versions, and flags on the fly and watch the assembly change. For an HFT engineer it becomes muscle memory: *write candidate code → read its receipt → compare alternatives.*

### Optimization levels: the single most important setting

The compiler's behavior is dominated by its **optimization level**, a flag you must always be conscious of:

- **`-O0`** (no optimization, the default): the compiler translates your code *literally and naively*, storing every variable to the stack, optimizing nothing. The assembly is verbose, slow, and **misleading** — it does *not* represent how your release build performs. **Never judge performance from `-O0` output.** It's useful only for debugging.
- **`-O2`**: the standard production optimization level — aggressive inlining, dead-code elimination, register allocation, loop optimizations. This is what most real software ships with.
- **`-O3`**: `-O2` plus more aggressive loop transformations and **auto-vectorization** (§8). Common in HFT and numerical code.
- **`-march=native`** (or a specific target like `-march=skylake-avx512`): tells the compiler it may use *all* instructions available on the target CPU — including the newest SIMD (AVX2, AVX-512). Without it, the compiler conservatively avoids newer instructions for portability. **HFT builds target the exact CPU they'll run on**, so this is standard, and it dramatically changes what you'll see (e.g., whether AVX appears).
- **Link-Time Optimization (`-flto`)** and **Profile-Guided Optimization (`-fprofile-...` / PGO):** optimizations that work across translation units (LTO) or use data from a real run to guide decisions like which branches are hot (PGO). Both matter in HFT and both change the receipt.

> **The discipline this builds:** every performance claim gets checked at the optimization level you actually ship. "It's faster" means nothing without "...at `-O2`/`-O3` with `-march=native`, here's the assembly diff." Compiler Explorer makes that check take ten seconds, so there's no excuse to guess.

---

## 6. What the optimizer does — and how to recognize when it didn't

The compiler at `-O2`/`-O3` is not a literal translator; it's an aggressive optimizer that often transforms your code beyond recognition. Recognizing its major moves — and spotting when one you expected *failed to happen* — is the core skill of reading HFT assembly.

The big transformations:

- **Inlining.** The body of a called function is pasted directly into the caller, eliminating the `call`/`ret`, the argument shuffling, and the prologue/epilogue — *and*, more importantly, exposing the inlined code to further optimization in its new context. **Inlining is the single most important optimization** because it's the gateway to all the others. *Recognize it:* the callee's instructions appear inline, and there's no `call` to it. *When it fails:* the function was too big, was compiled separately (no LTO), was virtual (§ vtables below), or was marked to prevent it — and you'll see a `call` remain, with its overhead.
- **Constant folding & propagation.** Computations on known constants are done *at compile time*. `int x = 3 * 4;` becomes just `12` in the assembly — the multiply never runs. (This is the seed of `constexpr`, Lesson 8: doing work before the program even starts.)
- **Dead-code elimination.** Code whose result is never used is deleted entirely. *This is a benchmarking hazard:* if you write a microbenchmark that computes something but never uses the result, the optimizer **deletes your entire benchmark** and you "measure" an empty loop (more in §10).
- **Loop unrolling.** A loop body is duplicated several times per iteration so the loop overhead (counter increment, condition check, branch) is paid less often, and to expose more independent work to the superscalar core (Lesson 1 §6). *Recognize it:* you see the same operation repeated several times inside one loop iteration, with the index advancing by 4 or 8.
- **Auto-vectorization.** The compiler rewrites a scalar loop to use SIMD instructions processing multiple elements at once (§8). *Recognize it:* `xmm`/`ymm`/`zmm` registers and packed instructions (`paddd`, `vaddps`...) appear where you wrote a plain loop.
- **Strength reduction & peephole tricks.** Replacing expensive operations with cheap equivalents: multiply-by-constant becomes shifts and `lea`s; divide-by-constant becomes a multiply by a magic number. Seeing a divide you wrote turn into a multiply is normal and good.
- **Register allocation.** Keeping hot variables in registers across a whole loop instead of reloading them from memory. *When it struggles:* you see "spills" — values stored to the stack and reloaded — a sign of register pressure.

### When the optimizer *fails* — and why it's often your fault (fixably)

The most valuable thing reading assembly teaches you is to catch optimizations that *didn't* happen. Common blockers:

- **Pointer aliasing.** If the compiler can't prove two pointers don't overlap in memory, it must assume a write through one might change what the other sees — so it can't keep values in registers or reorder/vectorize safely. It conservatively reloads from memory. This is why `__restrict` (a promise that pointers don't alias) can dramatically improve generated code, and why you'll see "unnecessary" reloads when aliasing can't be ruled out.
- **`volatile`.** Tells the compiler a variable may change outside the program's control, forbidding it from caching the value in a register or eliminating accesses. Necessary for memory-mapped hardware, but it *defeats optimization* — never use `volatile` for threading (that's what atomics are for, §9).
- **Function-call barriers.** A call to a function the compiler can't see into (not inlined) is an *optimization barrier*: the compiler must assume the call could read or write any memory, so it flushes assumptions across the call. This is another reason inlining matters so much.
- **Floating-point strictness.** By default the compiler won't reorder floating-point math (it would change rounding), which blocks some vectorization. `-ffast-math` relaxes this (with correctness caveats).

> **The HFT habit:** when you expected the compiler to do something — inline this, vectorize that, keep this in a register — and the code is slower than you hoped, *read the assembly to see what it actually did.* Nine times out of ten you'll find a specific, fixable blocker (an aliasing problem, a missed inline, a stray `volatile`), not a mysterious slowdown. The receipt diagnoses the problem.

### A concrete cost: `virtual` functions

Lesson 1 mentioned that runtime polymorphism is costly; the assembly shows you exactly why. A normal function call is `call some_address` — the target is known at compile time, the branch predictor handles it trivially, and it can be inlined. A **`virtual` function call** instead must look up the target at runtime through the object's **vtable** (a table of function pointers): load the vtable pointer from the object, load the function pointer from the vtable, then `call` *that*. In assembly:

```
mov  rax, [rdi]          ; load vtable pointer from the object
call [rax + 16]          ; indirect call through the vtable (offset 16 = 3rd virtual fn)
```

That `call [rax + 16]` is an **indirect call** — the target isn't known until runtime, so it (a) **can't be inlined** (the compiler doesn't know what it's calling, so it can't optimize across it), and (b) is an **indirect branch** that the predictor can mispredict (Lesson 1 §7), costing ~5 ns when it does. *This* is why HFT replaces runtime polymorphism with compile-time polymorphism (templates/CRTP, Lesson 8): to turn that unpredictable, un-inlinable indirect call back into a direct, inlinable one. Now you can *see* the difference in the receipt.

---

## 7. The real cost model: latency, throughput, and dependency chains

Reading *which* instructions ran is half the job. The other half is estimating *how much they cost* — and here the naive "count the instructions" instinct fails badly, exactly as Lesson 1 warned. Let's build the real cost model.

Every instruction has **two** different cost numbers, and confusing them is a classic mistake:

- **Latency:** how many cycles from when an instruction starts until its *result is available* for a dependent instruction to use. (An `add` has ~1 cycle latency; a `mul` ~3; a division 20+; a load that hits L1 ~4–5; a load that misses to RAM ~200+.)
- **Throughput:** how many of that instruction the core can *start* per cycle, given enough independent work. Because the core is superscalar (Lesson 1 §6) with multiple execution units, it can have many instructions of the same type in flight. Throughput is often expressed as instructions-per-cycle or its reciprocal (cycles-per-instruction). A core might sustain *two* multiplies started per cycle even though each takes 3 cycles to finish — because they overlap.

> **The analogy: latency is how long one cake takes to bake; throughput is how many cakes the bakery can have in the oven at once.** A cake takes an hour to bake (latency), but with ten ovens the bakery finishes ten cakes per hour (throughput). If your ten cakes are *independent*, you exploit all the ovens — high throughput. But if each cake can only be started once the *previous one finishes* (you need the first as an ingredient for the second), you're stuck baking them one at a time, and your speed is governed entirely by the one-hour *latency*, ten ovens or not.

That second case is the **dependency chain** from Lesson 1 §6, now made precise. The speed of a code sequence is bounded by **either** its total instruction count divided by throughput (if there's lots of independent work) **or** the latency of its longest dependent chain (if the work is serial) — *whichever is worse.* In HFT hot paths, you're very often **latency-bound**: stuck waiting for each result to feed the next.

### Seeing it concretely

Consider summing an array two ways. The naive loop:

```cpp
int sum = 0;
for (int i = 0; i < n; ++i) sum += a[i];
```

Every `sum += a[i]` depends on the previous value of `sum`. So even though the core *could* do several adds per cycle (throughput), it can't — each add must wait for the previous one's result (latency). This loop is **latency-bound on the dependency chain through `sum`**: one add per ~1 cycle of add-latency, no overlap. 

Now break the chain with **multiple accumulators** (a transformation `-O3` often does for you, called unrolling with reduction):

```cpp
int s0=0, s1=0, s2=0, s3=0;
for (int i = 0; i < n; i += 4) {
    s0 += a[i];   s1 += a[i+1];   s2 += a[i+2];   s3 += a[i+3];
}
int sum = s0 + s1 + s2 + s3;
```

Now the four running sums are *independent* — `s0 += ...` doesn't wait for `s1 += ...` — so the superscalar core does them in parallel, exploiting throughput. Same number of additions; potentially **3–4× faster**, purely by shortening the dependency chain. **This is the single most important micro-optimization pattern, and you can see it directly in the assembly** (four independent accumulator registers vs. one).

### Where the cost numbers come from

You don't memorize instruction costs — you look them up. The canonical references are **Agner Fog's instruction tables** (free, exhaustive latency/throughput data per instruction per CPU), the **Intel Intrinsics Guide**, and **uops.info**. There's even a tool, **LLVM-MCA** (Machine Code Analyzer, built into Compiler Explorer), that takes a snippet of assembly and *simulates* the CPU's pipeline to estimate cycles, show you the bottleneck, and tell you whether you're latency- or throughput-bound. For HFT micro-optimization, pasting a hot loop into LLVM-MCA and reading its analysis is a standard move.

A deeper detail you'll encounter: modern x86 CPUs don't execute your assembly instructions directly — they first **decode each instruction into one or more micro-operations (µops, "you-ops")**, which are what actually execute on the various **ports** (execution units). One `add reg, [mem]` instruction might decode into a load µop and an add µop. This is why instruction-counting is doubly misleading: the real currency is µops flowing through ports. You don't need to master this to start, but know the vocabulary — when a colleague says "that loop is port-5 bound," they mean a particular execution unit is the bottleneck, visible in tools like LLVM-MCA.

---

## 8. SIMD: doing many operations with one instruction

So far every instruction operated on single values — one add, one number. But the CPU also has instructions that operate on *many* values at once, and they're a major HFT tool. This is **SIMD: Single Instruction, Multiple Data.**

The idea: those wide SIMD registers from §2 (`xmm` = 128-bit, `ymm` = 256-bit, `zmm` = 512-bit) don't hold one number — they hold a *pack* of several. A 256-bit `ymm` register holds **eight** 32-bit integers, or four 64-bit doubles, side by side. A single SIMD instruction like `vpaddd ymm0, ymm1, ymm2` adds **eight pairs of integers simultaneously**, in roughly the time one scalar add would take. Eight operations, one instruction.

> **The analogy: scalar instructions are a clerk stamping one envelope at a time; SIMD is a stamping machine that presses eight envelopes in a single stroke.** Same motion, eight times the work — *if* your envelopes are lined up neatly in a row (contiguous in memory — back to Lesson 1's data layout!) so the machine can grab eight at once.

The families, by width and age:

- **SSE** (128-bit, `xmm`): the old baseline, on every x86-64 CPU. 4 floats or 4 int32s at a time.
- **AVX / AVX2** (256-bit, `ymm`): 8 floats / 8 int32s at a time. Standard on modern servers.
- **AVX-512** (512-bit, `zmm`): 16 floats / 16 int32s at a time, plus powerful new features (masking, etc.). Available on many server CPUs.

### How you get SIMD into your code

Three ways, in increasing order of control and effort:

1. **Auto-vectorization.** At `-O3 -march=native`, the compiler *automatically* rewrites suitable scalar loops into SIMD. This is the easy win — but it's fragile (aliasing, branches inside the loop, non-contiguous access, or floating-point strictness can all silently prevent it). *You confirm it happened by reading the assembly* — seeing `ymm` registers and packed ops where you wrote a plain loop. If they're absent, vectorization failed and you investigate why.
2. **Intrinsics.** Special functions (like `_mm256_add_ps`) that map almost one-to-one to SIMD instructions, letting you write vector code in C++ explicitly while the compiler still handles register allocation. HFT uses these when auto-vectorization won't cooperate and the loop is hot enough to justify hand-writing.
3. **Hand-written assembly.** Rare; reserved for the most extreme cases.

### HFT uses (and a critical caveat)

In HFT, SIMD shows up in **parsing market-data messages** (comparing many bytes at once to find delimiters or validate fields), **checksums/hashing**, **bulk comparisons** (scanning many prices), and numerical strategy computation. 

The critical caveat, especially for AVX-512: **using the widest SIMD can cause the CPU to *reduce its clock frequency*** ("downclocking") because those instructions draw more power and generate more heat. If you fire off a few AVX-512 instructions but the *rest* of your hot path is scalar, the frequency drop can make your overall code **slower**, not faster — you paid a clock-speed tax for a brief vector burst. This is a real, measured HFT gotcha: wide SIMD is a win only when you use it heavily enough to amortize the downclock, and sometimes AVX2 (256-bit) is deliberately chosen over AVX-512 to avoid it. As always: measure, and read the receipt.

---

## 9. Atomics and memory ordering at the instruction level

Lesson 1 §10 explained cache coherence — how cores keep their cached copies of memory consistent, at a cost. Now we see what that costs *in instructions*, because this is where C++'s `std::atomic` and multithreading meet the metal. This section is dense; the goal is to make you able to *recognize* atomic operations and memory barriers in assembly and know they aren't free.

### The x86 memory model: strong, but not free

When multiple cores read and write shared memory, a deep question arises: in what *order* do one core's writes become visible to another core? Different CPUs make different guarantees. x86-64 has a relatively **strong memory model** (called TSO — Total Store Order): roughly, all cores see writes in a consistent order, and the hardware does *not* reorder loads after loads or stores after stores in surprising ways. (ARM is weaker — it reorders more aggressively, requiring more explicit barriers. This is one reason porting lock-free code from x86 to ARM is treacherous.) Strong, however, does **not** mean free.

### Atomic read-modify-write: the `LOCK` prefix

Consider incrementing a shared counter from multiple threads: `counter++`. In plain code that's three steps — load, add, store — and two threads can interleave and lose an update. To make it **atomic** (indivisible), x86 has the **`LOCK` prefix**, which makes a read-modify-write instruction happen as one uninterruptible unit, with exclusive ownership of the cache line for its duration. `std::atomic<int> c; c.fetch_add(1);` compiles to:

```
lock add dword ptr [rip + c], 1      ; atomically add 1 to the counter in memory
```

That `lock` prefix is the whole story. It forces the core to take exclusive ownership of the cache line (coordinating with all other cores via the coherence protocol, Lesson 1 §10) and prevents reordering around it. **A `lock`-prefixed instruction costs on the order of tens of cycles (~10–20+ ns)** even when uncontended — *far* more than the ~1-cycle plain `add` it resembles — because of that mandatory cross-core coordination. When the line *is* contended (multiple cores hammering the same counter), it's dramatically worse, as the line ping-pongs between cores. **This is why HFT minimizes shared atomic state and prefers single-producer/single-consumer designs (Lesson 8): every `lock` is a coherence tax you can see right there in the assembly.**

You'll also see `xchg` (atomic exchange — implicitly locked) and `cmpxchg` (compare-and-swap, the primitive under most lock-free algorithms — "atomically, if memory still equals X, set it to Y").

### Memory fences

Sometimes you don't need an atomic operation, just an *ordering guarantee* — "make sure all my writes before this point are visible before any after it." That's a **memory fence/barrier**: `mfence` (full barrier), `lfence` (load barrier), `sfence` (store barrier). These force the core to wait until pending memory operations complete, defeating some of the reordering and overlap that normally hides latency — so they, too, have a cost you pay deliberately.

### What `std::atomic` memory orders compile to

C++'s `std::atomic` lets you specify a **memory order** per operation (`relaxed`, `acquire`, `release`, `seq_cst`), and the beautiful payoff of reading assembly is seeing what each *costs* on x86:

- On x86's strong model, plain atomic **loads and stores** with `acquire`/`release` often compile to **ordinary `mov` instructions** — essentially free! — because the hardware already provides the needed ordering.
- But `seq_cst` (sequential consistency, the *default* and strongest) **stores** require a fence or a `lock`-prefixed instruction — costly.
- Read-modify-write operations (`fetch_add` etc.) always need the `lock` prefix regardless.

> **The HFT lesson hiding in here:** the *default* `std::atomic` operations use `seq_cst`, the most expensive ordering. By choosing the *weakest correct* memory order (`relaxed` or `acquire`/`release`) for each operation, you can turn an expensive fenced/locked instruction into a plain `mov` on x86 — a real latency win. But getting the ordering right is famously subtle, and **the only way to confirm you actually saved the instruction is to read the assembly.** This is one of the most concrete places where reading the receipt directly translates to nanoseconds. (The C++ side of this is Lesson 8.)

---

## 10. Measuring at nanosecond resolution: `rdtsc` and its traps

You cannot optimize what you cannot measure, and HFT optimizations live at the scale of *single-digit nanoseconds* — far below what ordinary timing functions (which have microsecond-or-worse resolution and overhead) can see. The tool for cycle-accurate, in-program timing is a special CPU instruction: **`rdtsc`** — "Read Time-Stamp Counter."

The CPU maintains a **Time-Stamp Counter (TSC)**, a register that increments at a constant rate (on modern CPUs, at a fixed reference frequency regardless of the core's actual clock speed — "invariant TSC"). `rdtsc` reads it into registers. Bracket a piece of code with two `rdtsc` reads, subtract, and you get the elapsed cycles — convertible to nanoseconds. A minimal reader:

```cpp
static inline uint64_t rdtsc() {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a"(lo), "=d"(hi));   // result split across edx:eax
    return (uint64_t(hi) << 32) | lo;
}
```

This is *the* timing primitive of HFT latency measurement (and connects to the hardware timestamping of Lesson 3). But it is riddled with traps that will give you confidently wrong numbers if you don't respect them:

- **Out-of-order execution corrupts your measurement (Lesson 1 §6).** The core may execute your `rdtsc` *before* the code you meant to time, or the timed code may spill past the second `rdtsc` — because the core reorders freely. To get a meaningful boundary you must *serialize*: use **`rdtscp`** (a variant that waits for prior instructions to finish) and/or an `lfence`/`cpuid` barrier around the reads. Without serialization, you're timing a blur.
- **The measurement overhead is comparable to what you're measuring.** Reading the TSC itself costs cycles. When timing something that takes only a few nanoseconds, the `rdtsc` overhead is a large fraction of the result. You must measure the overhead of an empty measurement and subtract it, and you generally time a *batch* of iterations, not one.
- **Core migration breaks it.** If the OS moves your thread to a different core mid-measurement (Lesson 4 explains why and how to prevent it), you may read two different cores' counters and get garbage (even negative durations). You must **pin the thread** (Lesson 4) for `rdtsc` timing to be valid.
- **Frequency scaling vs. invariant TSC.** The TSC ticks at a constant reference rate, but the core's *actual* clock may vary (turbo, power saving). So TSC cycles don't map cleanly to "work cycles" unless you've fixed the frequency (Lesson 3/4 BIOS tuning). For converting TSC to nanoseconds you calibrate against a known clock once at startup.

> **And the deadliest benchmarking trap of all (from §6): the optimizer deletes code whose result is unused.** If you time a computation but never use its output, dead-code elimination removes the entire computation and you measure nothing (often a suspiciously round, tiny number). The defenses are to *consume* the result (feed it to a volatile sink, or use a compiler-specific "do not optimize away" helper like the ones in Google Benchmark), and — you guessed it — **read the assembly to confirm your benchmark's work is actually present in the receipt.** A microbenchmark you haven't disassembled is a microbenchmark you can't trust.

This is also why HFT reports the **distribution** of timings (median, p99, p99.9, max), never just an average — the whole point (Module 0) is the *tail*, and a single mean hides the rare slow spikes that lose races.

---

## 11. Putting it together: reading a hot function end-to-end

Let's integrate everything. Suppose your hot path has this innocent-looking function, meant to find whether any price in a small contiguous array exceeds a threshold:

```cpp
bool any_above(const float* prices, int n, float threshold) {
    for (int i = 0; i < n; ++i)
        if (prices[i] > threshold) return true;
    return false;
}
```

Reading its `-O3 -march=native` receipt, here's the kind of thing you'd look for and what each tells you, drawing on the whole lesson:

- **Arguments** (§4): `prices` arrives in `rdi`, `n` in `esi`, `threshold` in `xmm0` (float → SIMD register per the ABI). Confirmed at a glance.
- **The memory access** (§3, Lesson 1): you'd see `[rdi + rax*4]`-style addressing — the `*4` confirming 4-byte floats, and the steadily-incrementing index confirming the **sequential, prefetcher-friendly access** Lesson 1 wanted. Good.
- **Vectorization** (§8): at `-O3 -march=native` the compiler may **vectorize** this — loading 8 prices into a `ymm` register, comparing all 8 against a broadcasted threshold in one instruction (`vcmpps`), and checking if any matched. If you see `ymm` registers, it vectorized; the loop processes 8 elements per iteration. If you only see scalar `xmm` `comiss` comparisons, it *didn't* vectorize — and you'd ask why (maybe the early `return` defeated it; a branchless "did any exceed" reduction might vectorize better).
- **The branch** (§7): the `prices[i] > threshold` comparison plus conditional jump is a **branch**. If the answer is usually "no" (prices rarely exceed threshold), it's predictable and cheap. If it's a coin-flip, you're eating mispredicts in the hot path — visible as the `cmp`/`ja` pair, and a candidate for branchless rework.
- **The cost model** (§7): is this loop latency-bound or throughput-bound? The comparisons are independent across iterations (no value carries forward except the early-exit), so it *can* be throughput-bound and fast — paste it into LLVM-MCA to confirm and find the bottleneck port.
- **The early return** (§4): `return true` becomes `mov eax, 1; ret`; `return false` becomes `xor eax, eax; ret` (§3's zeroing idiom). The boolean is in `al`/`eax` per the ABI.

In a few seconds of reading, you've confirmed the access pattern is cache-friendly, learned whether it vectorized, identified the branch as a predictability risk, and formed a hypothesis about the bottleneck — *all before running anything.* That is the entire point of this lesson: **the assembly turns Lesson 1's abstract costs into concrete, visible, checkable facts about your actual program.**

---

## 12. Recap

- You won't *write* assembly in HFT; you'll *read* it constantly. **Assembly is the receipt for your C++** — the ground truth of what the machine does. (§0)
- Source → compiler → **assembly** → machine code. The compiler step is where all the cleverness and surprises live; the receipt reveals every decision. (§1)
- The CPU computes in **16 general-purpose registers** (`rax`, `rdi`, ...), with sub-register names (`eax`) indicating data width. Special: `rsp` (stack top), `rip` (next instruction), `rflags`. (§2)
- Instructions are **mnemonic + operands**; `[base + index*scale]` memory operands are array accesses you can read directly. Intel syntax is dst-first; AT&T (Linux default) is src-first — don't get reversed. (§3)
- Functions use the **stack** and the **System V ABI** (args in `rdi, rsi, rdx, rcx, r8, r9`; return in `rax`). Skim prologue/epilogue; focus on the body. Function calls cost — which is why **inlining** matters. (§4)
- **Compiler Explorer (godbolt.org)** makes reading the receipt a ten-second habit. Always judge at `-O2`/`-O3 -march=native`, **never `-O0`.** (§5)
- The optimizer **inlines, folds constants, eliminates dead code, unrolls, vectorizes, reduces strength**. Learn to spot when an expected optimization *didn't* happen — usually a fixable blocker (**aliasing**, `volatile`, an un-inlined call). `virtual` calls become un-inlinable, mispredictable **indirect calls** through a vtable. (§6)
- The real cost model: **latency** (time to a result) vs. **throughput** (how many in flight). Hot paths are often **latency-bound on a dependency chain**; breaking the chain (multiple accumulators) is the key micro-optimization. Look up costs in **Agner Fog's tables**; simulate with **LLVM-MCA**. (§7)
- **SIMD** (SSE/AVX/AVX-512, the `xmm`/`ymm`/`zmm` registers) does many operations per instruction; get it via **auto-vectorization** (confirm in the receipt) or **intrinsics**. Beware AVX-512 **downclocking**. (§8)
- **Atomics** compile to **`lock`-prefixed** instructions costing tens of cycles via cache-coherence coordination; **memory fences** cost too. On x86's strong model, weak memory orders can become plain `mov`s — a real win you confirm by reading the receipt. (§9)
- Measure at nanosecond scale with **`rdtsc`/`rdtscp`**, respecting its traps: out-of-order execution (serialize!), measurement overhead, core migration (pin the thread!), frequency scaling, and the **dead-code-elimination benchmark trap**. Report the **tail**, not the mean. (§10)

---

## 13. Glossary

| Term | Meaning |
|------|---------|
| **Assembly** | Human-readable, near-one-to-one textual form of the CPU's native machine code. |
| **Machine code** | The raw bytes the CPU actually executes; what assembly assembles into. |
| **ISA** | Instruction Set Architecture — the instruction vocabulary of a CPU family. x86-64 (Intel/AMD) is the HFT default; AArch64 (ARM) is rising. |
| **Mnemonic** | The name of an instruction (`mov`, `add`, `jmp`). |
| **Operand** | What an instruction acts on: a register, an immediate (constant), or a memory location `[...]`. |
| **Register (general-purpose)** | One of 16 64-bit slots (`rax`, `rdi`, ...) where computation happens; `eax` etc. are narrower slices. |
| **`rsp` / `rip` / `rflags`** | Stack pointer / instruction pointer (next instruction) / status flags read by conditional jumps. |
| **Addressing mode** | How a memory operand computes an address; `[base + index*scale + disp]` is the array-access form. |
| **Intel vs. AT&T syntax** | Two notations: Intel `mov dst, src` (dst first); AT&T `mov %src, %dst` (src first, Linux default). |
| **Stack** | Downward-growing memory region for call frames and locals; `rsp` marks its top. |
| **Stack frame** | One function call's slice of the stack (locals + bookkeeping). |
| **Calling convention / ABI** | The rules for passing args and returning values. System V AMD64: args in `rdi, rsi, rdx, rcx, r8, r9`; return in `rax`. |
| **Caller/callee-saved** | Which side is responsible for preserving a register across a call. |
| **Prologue / epilogue** | Setup/teardown of a stack frame at a function's start/end; skim past them. |
| **Compiler Explorer (godbolt)** | Web tool showing the assembly your C++ compiles to, source-matched and instant. |
| **Optimization level** | `-O0` (none, misleading), `-O2` (production), `-O3` (+ vectorization); `-march=native` enables target-specific (AVX) instructions. |
| **Inlining** | Pasting a callee's body into the caller, removing call overhead and enabling further optimization. The gateway optimization. |
| **Dead-code elimination** | Removing code whose result is unused — the classic microbenchmark-destroying trap. |
| **Loop unrolling** | Duplicating a loop body per iteration to cut loop overhead and expose parallelism. |
| **Auto-vectorization** | Compiler turning a scalar loop into SIMD automatically. |
| **Aliasing** | Inability to prove two pointers don't overlap, forcing conservative (slower) code; `__restrict` relaxes it. |
| **Vtable / indirect call** | Runtime function-pointer table behind `virtual`; produces an un-inlinable, mispredictable indirect call. |
| **Latency (of an instruction)** | Cycles until its result is usable by a dependent instruction. |
| **Throughput** | How many of an instruction can be started per cycle, given independent work. |
| **Dependency chain** | A serial sequence where each instruction needs the prior's result; the true limiter of latency-bound code. |
| **µop (micro-op) / port** | The internal sub-operations instructions decode into / the execution units they run on. |
| **LLVM-MCA** | Tool that simulates the pipeline for a snippet and reports cycles and bottlenecks. |
| **Agner Fog tables** | The canonical free reference for per-instruction latency/throughput. |
| **SIMD** | Single Instruction, Multiple Data — one instruction on a pack of values, via `xmm`/`ymm`/`zmm`. |
| **SSE / AVX / AVX2 / AVX-512** | SIMD generations: 128 / 256 / 256 / 512-bit wide. |
| **Intrinsics** | C++ functions mapping ~one-to-one to SIMD instructions for explicit vectorization. |
| **Downclocking** | CPU lowering its frequency under heavy (esp. AVX-512) load — can make wide SIMD a net loss. |
| **`LOCK` prefix** | Makes a read-modify-write atomic via exclusive cache-line ownership; costs tens of cycles. Underlies `std::atomic` RMW ops. |
| **Memory fence (`mfence`/`lfence`/`sfence`)** | Forces memory-operation ordering; defeats some latency-hiding, so it has a cost. |
| **Memory model (TSO)** | x86's relatively strong ordering guarantees; lets weak `std::atomic` orders compile to plain `mov`s. ARM is weaker. |
| **TSC / `rdtsc` / `rdtscp`** | The Time-Stamp Counter and the instructions reading it for cycle-accurate timing (`rdtscp` serializes). |

---

## 14. Where to go next

- **Lesson 3 — Hardware.** Leave the core and meet the NIC, PCIe, clocks, and the physical server. The `rdtsc` timing of §10 generalizes into **hardware timestamping**; the BIOS frequency settings that make TSC trustworthy get decided there.
- **Lesson 8 — C++ for HFT.** The destination. Every assembly-level fact here becomes a C++ rule there: *why* `virtual` is avoided (§6's vtable), *why* `__restrict` and avoiding aliasing matters (§6), *how* `std::atomic` memory orders map to `lock`/fence/`mov` (§9), *how* to write loops that vectorize (§8) and break dependency chains (§7). You'll read the receipt for your own C++ and recognize all of it.
- **Tooling habit to build now:** open **godbolt.org**, paste small functions, and play. Toggle `-O0` → `-O2` → `-O3 -march=native` and *watch the assembly transform.* Write the two array-sum versions from §7 and confirm the multiple-accumulator one breaks the dependency chain. Paste a hot loop into **LLVM-MCA** and read its bottleneck analysis. The skill of this lesson is built by doing, not reading — and the feedback loop is seconds long.

> The through-line from Lessons 1 and 2: **architecture tells you what costs; assembly shows you where the cost is in your actual code.** Together they turn "make it fast" from guesswork into a measurable, visible engineering discipline — which is exactly what the rest of this course will lean on.
