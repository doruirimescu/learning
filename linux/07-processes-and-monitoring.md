# Lesson 7 — Processes and Process Monitoring

> The previous three lessons all circled the same invisible center. [Lesson 4](04-systemd-services.md) managed **services** — but a service *is* just one or more processes that systemd supervises. [Lesson 5](05-journalctl.md) read what processes **logged**. [Lesson 6](06-dmesg.md) watched the kernel **kill** a process (the OOM killer). In every case the real actor was the **process** — and we never stopped to define it. This lesson zooms all the way into that atom: what a process actually *is*, how to watch the living population of them on your machine (mostly with `top`), and how to *control* them. Once you understand processes, the earlier lessons snap into a single coherent picture.

---

## 0. The mental model: program vs. process

Start with the distinction that everything else depends on, because conflating these two words is the root of most process confusion.

A **program** is a *passive* thing: a file sitting on disk — `/usr/bin/python3`, `/usr/sbin/sshd`, your compiled app. It's instructions at rest. It does nothing. It consumes no CPU and no memory beyond the disk space it occupies. It's inert.

A **process** is a program *brought to life* — a running, active *instance* of that program, loaded into memory and executing, consuming CPU time, holding memory, owning open files and network connections. It's the program *happening.*

The analogy I'll use throughout this lesson — and it's worth holding onto because it makes the resource concepts intuitive: **a program is a recipe in a cookbook; a process is a cook actively making that dish in a kitchen.** The recipe is just paper — it makes nothing, costs nothing to "run." The cook is the live activity: occupying a stove, using counter space, taking time, producing a result. And crucially:

- **One recipe, many cooks.** Ten people can cook the same recipe at once, independently. Likewise, one program can have *many* simultaneous processes — open three `python3` scripts and there are three distinct Python processes, all running the same program file, each with its own separate memory and state. The recipe is shared and read-only; each cooking is its own activity.
- **The kitchen has finite resources.** A fixed number of stoves (CPU cores) and a fixed amount of counter space (RAM). The entire drama of process monitoring — load, memory pressure, the OOM killer — is the story of many cooks contending for limited stoves and counters.

> **Bridging to Lesson 4:** a *service* is simply one or more processes that systemd launches and supervises (puts in a cgroup, restarts, logs). "Process" is the general atom; "service" is the special case of "a process systemd babysits." Everything you learn here applies to *all* processes — services, your shell, your editor, background daemons, and the commands you type.

---

## 1. The anatomy of a process: what every process carries

A process isn't just "running code." The kernel wraps each one in a bundle of bookkeeping — its identity and its possessions. Knowing these fields is exactly what lets you *read* `top` and `ps` output later, so let's meet them as concepts first, not columns.

**A PID — the process ID.** A unique number the kernel assigns when the process is born. It's the process's name tag for the rest of its life. When you want to act on a specific process (inspect it, kill it), the PID is how you name it. (You met the most famous one in Lesson 4: **PID 1**, systemd, the ancestor of all.)

**A PPID — the parent process ID.** Here's a foundational truth: **every process is created by another process.** A process doesn't appear from nowhere — an existing process *forks* (makes a copy of itself) and then typically *replaces the copy with a new program* (this fork-then-exec dance is how Linux launches everything). The creator is the **parent**; the new one is the **child**. So every process records *who made it* (its PPID), and the whole system forms a **tree** rooted at PID 1. Your shell launched the command you just ran; systemd launched your login session; the kernel launched systemd. It's parents all the way up to PID 1.

> **Why the tree matters:** it explains "what launched this?" (Lesson 4's first question) at the most literal level — follow the PPID chain upward. It also explains *orphans* and *zombies*, which we'll get to, and it's exactly the structure cgroups (Lesson 4) box up.

**An owner — the user (UID) it runs as.** Every process runs *as* some user, and inherits that user's permissions. A process owned by `root` can do nearly anything; one owned by `www-data` is constrained. This is the `User=` from Lesson 4's unit files, made real at runtime — and it's central to the security question "should this process be able to do that?"

**Memory — and the surprising way it's measured.** A process holds memory, but there are *two* numbers that matter and people constantly confuse them:

- **Virtual memory (VIRT)** — the total address space the process has *asked for* or mapped, including memory it isn't actually using and shared libraries it shares with others. This number is often huge and somewhat fictional — like the total square footage a cook has *reserved* across the whole building, much of which is empty or shared hallway.
- **Resident memory (RES or RSS)** — the *physical RAM the process is actually occupying right now.* This is the number you usually care about for "how much memory is this really using" — the actual counter space the cook is physically standing at. When you hunt a memory hog, **RES is the honest number.**

**CPU time and state.** The kernel tracks how much CPU time a process has consumed and what it's currently *doing* — running, waiting, sleeping. That "what it's doing" is the **process state**, and it's important enough for its own section.

**An environment and open files.** Each process carries its environment variables (Lesson 4's environment inheritance, now you see *where* it lives — attached to the process) and a table of open file descriptors (files, sockets, pipes it's using).

---

## 2. Process states: running, sleeping, zombie, and the scary D

When you watch processes, you'll see a one-letter **state** column, and reading it is genuinely diagnostic — so understand what the letters mean rather than skimming past them. At any instant, a process is in one of a few states:

- **R — Running (or runnable).** Either actively using a CPU right now, or sitting in the run-queue ready to run the moment a core frees up. The cook is actively cooking, or standing at the ready waiting for a stove.
- **S — Interruptible sleep.** The overwhelmingly common state. The process is *waiting* for something — a keypress, a network packet, a timer — and is happily idle until it arrives. **Most processes on a healthy system are asleep most of the time**, which surprises beginners: a system with 300 processes isn't 300 things frantically computing; it's mostly 300 cooks waiting for an order. They wake instantly when needed.
- **D — Uninterruptible sleep.** This one matters. The process is stuck waiting on something it *cannot* be woken from — almost always **disk or hardware I/O.** It can't even be killed until the I/O completes. A *few* processes briefly in D is normal; *many* processes stuck in D, or one stuck for a long time, is a classic sign of **failing or overloaded storage** — and it ties directly to Lesson 6: those disk I/O errors in `dmesg` are often *why* processes are wedged in D. When a machine feels frozen but the CPU is idle, suspect D-state processes blocked on sick hardware.
- **T — Stopped.** The process has been *paused* (suspended), typically by you pressing `Ctrl-Z` or by a `STOP` signal. It's frozen mid-task, holding its place, until told to continue. The cook has been told "freeze" and stands motionless until "resume."
- **Z — Zombie.** The most misunderstood state, and actually harmless when transient. A zombie is a process that has **finished running but whose parent hasn't yet collected its exit status.** It's dead — using no CPU, no real memory — but its entry lingers in the process table because the parent needs to "reap" it (read how it ended). Think of it as a finished cook who's left the kitchen but whose timecard is still clipped to the board until the manager files it. A few momentary zombies are normal. *Piles* of zombies that don't clear mean a **buggy parent process that isn't reaping its children** — the bug is in the parent, not the zombies.

> **Orphans vs. zombies (a common mix-up):** an *orphan* is a live process whose parent *died first* — it's perfectly healthy, it just lost its parent. The kernel handles this gracefully: orphans are **re-parented to PID 1** (systemd), which becomes their adoptive guardian and will reap them when they finish. A *zombie* is the opposite — already dead, waiting to be reaped. Orphan = alive, lost its parent. Zombie = dead, not yet collected.

---

## 3. The snapshot vs. the live feed: `ps` and `top`

There are two fundamentally different ways to look at processes, and the distinction mirrors the `journalctl` (review) vs `journalctl -f` (follow) split from Lesson 5.

**`ps` is a snapshot** — a single still photograph of which processes existed *at the instant you ran it*. It prints once and returns your prompt. Perfect for "list the processes matching X right now" or feeding into scripts. The most useful incantation to know is:

```
ps aux
```

which lists **every process on the system** (`a` = all users, `u` = user-friendly columns with owner/CPU/memory, `x` = include processes not attached to a terminal — i.e. background daemons). It's the photograph of the entire process population, one line each. You'll often pair it with `grep` to find something specific (though `pgrep`, in the cheatsheet, is cleaner). The columns it shows — USER, PID, %CPU, %MEM, VSZ (virtual), RSS (resident), STAT (state), COMMAND — are exactly the anatomy from Sections 1–2, now you can read them.

**`top` is the live feed** — a continuously refreshing dashboard that updates every few seconds, showing you the process population *changing in real time*, sorted so the heaviest resource users float to the top (hence the name). This is the head chef standing in the kitchen doorway watching every cook at once, the busiest ones front and center. It's the tool you reach for when you ask the live questions: *"what is eating my CPU right now? what's chewing through memory? why is this machine slow this very moment?"*

Because `top` is the workhorse the question asked about, the next two sections dissect its display in detail.

---

## 4. Reading `top`, part 1: the summary header

Run `top` and the screen splits into two regions: a **summary header** at the top (the system-wide vitals) and a **process table** below (the per-process detail). Beginners stare only at the table and miss that the *header* often holds the actual answer. Let's read the header line by line, because each line is a distinct system-health signal.

**Line 1 — uptime and load average.** Looks like:

```
top - 14:32:08 up 3 days,  2:11,  2 users,  load average: 0.42, 0.55, 0.61
```

The headline concept here is **load average**, and it's the most misunderstood number in all of Linux, so let's get it right. The three numbers are the load averaged over the last **1, 5, and 15 minutes**. **Load is not a percentage and not "CPU usage."** It's roughly *the average number of processes that were either running or waiting in line to run* (plus, on Linux, those stuck in D-state I/O waits).

The bridge-and-tollbooth analogy makes it click: imagine the CPU is **a single tollbooth**, and load is *how many cars are at the booth on average* — being served plus queued behind.

- On a **1-core** machine: load `1.0` means the booth is exactly saturated — one car always being served, none waiting. Load `0.5` means it's busy half the time. Load `2.0` means for every car being served, another is *waiting* — the system is overloaded, things feel sluggish.
- The catch: you must **compare load to your number of cores.** A load of `4.0` is a *disaster* on a 1-core machine (4× oversubscribed) but *perfectly healthy* on an 8-core machine (8 booths, only 4 cars — half idle). So always read load relative to core count (which you can get from `nproc`).
- The three numbers tell a *trend*: if 1-min ≫ 15-min, load is *spiking right now*; if 1-min ≪ 15-min, a storm is *passing*. It's a little weather report for system pressure.

**Line 2 — tasks.** A census of the process population by state: `total`, `running`, `sleeping`, `stopped`, `zombie`. Now that you know Section 2's states, this line is instantly meaningful — and a nonzero `zombie` count or a surprising `stopped` count is a flag worth a second look.

**Line 3 — CPU breakdown.** A percentage breakdown of where CPU time is going, and the components are genuinely diagnostic:

- **`us` (user)** — time running normal user-space process code. "Cooks actually cooking." High `us` = your programs are genuinely busy computing.
- **`sy` (system)** — time spent *in the kernel* on behalf of processes (system calls, Lesson 6's kernel work). High `sy` can mean heavy I/O or lots of system-call overhead.
- **`id` (idle)** — doing nothing. High idle = spare capacity.
- **`wa` (I/O wait)** — **the important one.** Time the CPU sat idle *because it was waiting for disk/hardware I/O to finish.* High `wa` is the smoking gun for "the machine is slow but the CPU isn't busy" — it's *blocked on storage*, the same story as D-state processes and Lesson 6's disk errors. When users say "it's slow" and `us` is low but `wa` is high, your problem is the disk, not the CPU.
- **`ni`, `hi`, `si`, `st`** — niced user time, hardware/software interrupts, and "steal" (time a VM lost to its hypervisor). Minor day-to-day, but `st` being high on a cloud VM means a noisy neighbor is stealing your CPU.

**Lines 4–5 — memory and swap.** Physical RAM and swap, broken into `total`, `free`, `used`, and `buff/cache`. The one piece of wisdom that saves endless panic: **`buff/cache` memory is not "lost."** Linux deliberately uses otherwise-idle RAM to cache disk data for speed, and it will *instantly* give that memory back to any process that needs it. So "free" looking tiny is *not* a problem by itself — the number to watch is the `available` figure (and the swap line). A system isn't memory-starved until it's actively and heavily *swapping* (shoving memory out to disk), which is slow and is the prelude to Lesson 6's OOM killer firing.

> **The header is the triage.** Before you even look at individual processes, the header has often already told you the *category* of problem: high load + high `us` = CPU-bound work; low load + high `wa` = disk/I/O problem; shrinking available memory + rising swap = memory pressure heading toward an OOM kill. Read the header *first*.

---

## 5. Reading `top`, part 2: the process table and interactive keys

Below the header is the live, sorted table — one row per process. The columns are the Section 1 anatomy made visible:

| Column | Meaning (from Section 1–2) |
|--------|----------------------------|
| **PID** | The process's unique ID |
| **USER** | Who owns it |
| **PR / NI** | Scheduling priority and "niceness" (Section 6 below) |
| **VIRT** | Virtual memory — the big, partly-fictional reservation |
| **RES** | Resident memory — **the real RAM in use; the honest number** |
| **SHR** | Shared memory (shareable with other processes) |
| **S** | State — R, S, D, T, Z (Section 2) |
| **%CPU** | Share of a *single core's* worth of CPU (so this can exceed 100% for multi-threaded processes across cores!) |
| **%MEM** | RES as a percentage of total RAM |
| **TIME+** | Total CPU time consumed over the process's life |
| **COMMAND** | What program it's running |

A subtlety worth flagging: **`%CPU` is per-core.** A process using four cores fully shows `400%`. This stuns people the first time — it's not a bug, it's the metric being honest that the process is using four cores' worth of compute.

### `top` is interactive — and that's the whole point

The mistake beginners make is treating `top` as a passive wall of numbers. It's actually a **command center**: while it's running, single keypresses reshape it and even act on processes. The essential ones:

- **`q`** — quit. (Your escape hatch.)
- **`P`** — sort by `%CPU` (the default; the **P** is a mnemonic for Processor). "Who's burning CPU?"
- **`M`** — sort by `%MEM` / RES. "Who's hogging memory?" The single most useful keystroke when hunting a memory leak.
- **`T`** — sort by cumulative TIME. "Who's consumed the most CPU over their whole life?"
- **`k`** — **kill** a process: it prompts for a PID and a signal (Section 7). Acting on the problem without leaving the tool.
- **`r`** — **renice** a process: change its priority on the fly (Section 6).
- **`1`** — toggle showing *each CPU core separately* vs. one combined line. Essential for seeing whether work is spread across cores or pinned to one.
- **`H`** — toggle showing individual *threads* rather than whole processes.
- **`c`** — toggle showing the full command line (with arguments and path) vs. just the program name — great for telling apart ten `python3` processes by *which script* each runs.
- **`u`** — filter to a single user's processes.
- **`f`** — the field manager: choose which columns to show and which to sort by.
- **`W`** — save your current layout as the default so `top` opens the way you like next time.

> **The reframe:** `top` isn't a report you read, it's a **live instrument you play.** Launch it, press `M` to find the memory hog, press `c` to see exactly which script it is, press `k` to deal with it — all without typing a single other command. That fluency is the difference between staring at numbers and diagnosing a problem.

### A word on `htop` (and friends)

`top` is **always installed**, everywhere, which is why it's the one to learn first — you'll never be on a machine without it. But you'll quickly hear about **`htop`**, a friendlier alternative (often needs installing) with color, per-core bar graphs, mouse support, scrolling, and easier killing/sorting via labeled function keys. Once you understand `top`'s *concepts*, `htop` is just a nicer skin over the same ideas — learn `top` for universality, reach for `htop` for comfort. (Newer tools like `btop` go further still with prettier graphs.) The concepts transfer completely; only the controls differ.

---

## 6. Controlling priority: `nice` and `renice`

Not all cooks deserve equal stove time. Linux lets you bias the scheduler's attention toward or away from a process using **niceness** — and the name is a genuinely helpful mnemonic, if slightly backwards.

Niceness is a number from **−20 (least nice) to +19 (most nice)**, and the trick to remembering its direction: **"nice" means nice *to other processes*.** A *high* niceness (+19) means "I'm very considerate, I'll yield my CPU time to others" — so that process runs at *low* priority. A *low/negative* niceness (−20) means "I'm not nice, I demand the CPU" — *high* priority. So **higher nice value = lower priority.** Backwards, but memorable once you frame it as politeness.

- **`nice -n 10 some-command`** — *start* a command with a specified niceness (here +10, a considerate background job that won't disturb interactive work).
- **`renice -n 5 -p 1234`** — *change* the niceness of an already-running process (PID 1234). Or do it live in `top` with the `r` key.
- Going *negative* (raising priority, e.g. `-n -5`) requires **root**, because letting any user crank their process to top priority would let them starve everyone else. Lowering your own priority (being nicer) is always allowed.

The everyday use: you're about to run a heavy batch job — a big compile, a video encode, a backup — on a machine you also want to keep using. Launch it with `nice` so it soaks up *spare* CPU but politely steps aside the instant you need the machine to feel responsive. It's "run this, but don't let it ruin my day."

> **Niceness ≠ a hard limit.** It's a *hint* that biases the scheduler's choices, not a cap. For actual hard ceilings ("this service may use at most 50% of one core"), you want **cgroup resource limits** (`CPUQuota`, `MemoryMax`) — which, recall from Lesson 4, exist precisely because a service *is* a cgroup. Niceness tunes *competition*; cgroups enforce *limits*.

---

## 7. Talking to processes: signals, `kill`, and friends

How do you actually *tell a running process to do something* — stop, reload, quit? You don't call it on the phone; you send it a **signal.** This is one of the most elegant and misunderstood corners of Linux, so let's frame it properly.

A **signal** is a tiny, predefined message the kernel delivers to a process — a tap on the shoulder with a specific meaning. The process can (for most signals) decide how to react: handle it gracefully, ignore it, or let the default behavior happen. The command to send one is, somewhat menacingly, called **`kill`** — a historical misnomer, because *most signals don't kill anything.* `kill` really means "send a signal," and killing is just the *default* one. The signals you must know:

- **`SIGTERM` (15) — "please wrap up and exit."** The *polite, default* signal sent by plain `kill 1234`. It asks the process to shut down *gracefully* — finish the current request, flush data to disk, clean up, then exit. **This is almost always the right first choice**, because it gives the process a chance to leave cleanly. Telling a cook "finish your current dish and then clock out."
- **`SIGKILL` (9) — "die NOW."** The nuclear option, `kill -9 1234`. This signal **cannot be caught, handled, or ignored** — the kernel terminates the process instantly, no cleanup, no save, no goodbye. Use it *only* when `SIGTERM` failed and a process is truly stuck, because it can leave corrupt files, orphaned resources, or a half-written database. The cultural reflex to "just `kill -9` it" is a bad habit: **try `SIGTERM` first, reach for `SIGKILL` only when the process ignores it.** (Note: a process wedged in D-state, Section 2, won't even die to `SIGKILL` until its I/O unblocks — another sign the real problem is the hardware.)
- **`SIGHUP` (1) — "hang up / reload."** Originally "the terminal hung up," now conventionally repurposed by many daemons to mean **"re-read your configuration without restarting."** It's how you reload many services' configs without downtime. (systemd's `reload` often maps to this.)
- **`SIGSTOP` / `SIGCONT` — pause and resume.** `SIGSTOP` freezes a process (the T state from Section 2); `SIGCONT` thaws it. This is what `Ctrl-Z` does in your shell, and what job control (next section) builds on.
- **`SIGINT` (2) — "interrupt."** What `Ctrl-C` sends: a polite "stop what you're doing." Most interactive programs treat it as "quit now."

The commands that send signals:

- **`kill 1234`** — send `SIGTERM` (the default) to PID 1234. `kill -9 1234` or `kill -KILL 1234` sends `SIGKILL`; `kill -HUP 1234` sends `SIGHUP`; and so on.
- **`pkill firefox`** / **`killall firefox`** — send a signal to processes *by name* instead of PID, so you don't have to look the number up first. (`pkill` matches patterns; `killall` matches exact names — minor differences, both spare you the PID lookup.)
- **`pgrep firefox`** — the *non*-destructive partner: just *find* the PIDs matching a name, printing them without sending anything. The safe way to ask "is this running, and what's its PID?" before you act.

> **The discipline:** identify with `pgrep`/`top`, then signal with `SIGTERM` first and escalate to `SIGKILL` only if ignored. Reaching straight for `kill -9` is the process-management equivalent of the destructive `dmesg --clear` from Lesson 6 — occasionally necessary, but a reflex worth distrusting because of what it can damage.

---

## 8. Foreground, background, and outliving your terminal

A last practical cluster, because it answers real daily questions: *"how do I run something in the background?"* and *"why did my long-running command die when I closed the terminal?"*

When you run a command in your shell, it runs in the **foreground** — it occupies your terminal, and you wait for it to finish. Linux lets you manage this:

- **`Ctrl-Z`** — suspend the foreground process (sends `SIGSTOP`; it goes to T state, paused).
- **`bg`** — resume that suspended process *in the background*, so it keeps running while giving you your prompt back.
- **`somecommand &`** — start a command in the background from the outset (the trailing `&`). The shell hands you back your prompt immediately and the command runs alongside.
- **`jobs`** — list the background/suspended jobs of *this shell*.
- **`fg`** — pull a background job back into the foreground.

But there's a trap that bites everyone once. By default, processes you launch are **children of your shell**, and when you close the terminal (or your SSH session drops), the shell sends `SIGHUP` to its children — and many of them die. So your hours-long job, started casually with `&`, **vanishes the moment your connection drops.** Two cures:

- **`nohup somecommand &`** — `nohup` ("no hangup") makes the command *immune to `SIGHUP`*, so it survives the terminal closing. Output goes to a `nohup.out` file since there's no terminal to print to.
- **`disown`** — detaches an already-running background job from the shell's job table so it won't receive the hangup.

> **The modern answer, tying back to Lesson 4:** for anything you genuinely want to *keep running* — a server, a long-lived task — the *right* tool isn't `nohup`, it's a **systemd service** (or a `systemd-run` transient unit, or a terminal multiplexer like `tmux`). Why? Because systemd gives you everything `nohup` can't: automatic restart, logging into the journal (Lesson 5), resource limits, and supervision. `nohup` is the quick hack; a service is the durable solution. This is the full circle — we started Lesson 4 with services and now you see they're the *grown-up* way to keep a process alive, as opposed to these shell-level tricks for casual one-offs.

---

## 9. The command cheatsheet

The reference you asked for. Grouped by intent — *what you're trying to do* — because that's how you'll search your memory under pressure. Everything here is explained in the lesson above; this is the lookup table.

### Look at processes (snapshot)

| Command | What it does |
|---------|-------------|
| `ps aux` | Every process on the system, with owner/CPU/memory/state columns |
| `ps aux \| grep nginx` | Snapshot filtered to a name (quick and dirty) |
| `pgrep -a nginx` | Find PIDs by name, cleanly (`-a` also shows the command line) |
| `ps -ef --forest` | All processes drawn as the parent/child **tree** (see the PPID hierarchy) |
| `pstree -p` | The process tree as an ASCII diagram with PIDs |
| `ps -o pid,ppid,user,stat,rss,comm -p 1234` | Custom columns for one specific PID |

### Watch processes (live)

| Command | What it does |
|---------|-------------|
| `top` | The live, refreshing dashboard (always installed) |
| `htop` | Friendlier live monitor — color, per-core bars, easier keys (often needs install) |
| `btop` | Even richer graphical monitor (needs install) |
| `watch -n 2 'ps aux --sort=-%mem \| head'` | Re-run any snapshot command every 2s for a poor-man's live view |

### Key interactive keys *inside* `top`

| Key | Action |
|-----|--------|
| `q` | Quit |
| `P` | Sort by CPU |
| `M` | Sort by memory (RES) |
| `T` | Sort by cumulative CPU time |
| `k` | Kill a process (prompts for PID + signal) |
| `r` | Renice (change priority) |
| `1` | Toggle per-core CPU lines |
| `H` | Toggle threads vs. processes |
| `c` | Toggle full command line |
| `u` | Filter by user |
| `f` | Choose/sort columns |
| `W` | Save current layout as default |

### Control / signal processes

| Command | What it does |
|---------|-------------|
| `kill 1234` | Send **SIGTERM** (polite "wrap up and exit") — *the right first choice* |
| `kill -9 1234` | Send **SIGKILL** (force-kill, no cleanup) — *last resort only* |
| `kill -HUP 1234` | Send **SIGHUP** (often "reload config") |
| `pkill firefox` | Signal processes by name pattern |
| `killall firefox` | Signal all processes with that exact name |
| `kill -l` | List all signal names and numbers |

### Priority & resources

| Command | What it does |
|---------|-------------|
| `nice -n 10 cmd` | Start `cmd` at low priority (considerate background work) |
| `renice -n 5 -p 1234` | Change a running process's niceness (root needed to raise priority) |
| `nproc` | How many CPU cores you have (to interpret load average) |
| `free -h` | Memory and swap usage, human-readable |
| `uptime` | Load average without opening `top` |

### Background / job control

| Command | What it does |
|---------|-------------|
| `cmd &` | Run in background from the start |
| `Ctrl-Z` | Suspend the foreground process |
| `bg` / `fg` | Resume in background / bring to foreground |
| `jobs` | List this shell's background jobs |
| `nohup cmd &` | Run immune to terminal-close (SIGHUP) |
| `disown` | Detach a running job from the shell |

### Inspect a process deeply

| Command | What it does |
|---------|-------------|
| `ls -l /proc/1234/` | The kernel's live directory of *everything* about PID 1234 |
| `cat /proc/1234/status` | Human-readable state, memory, owner of a process |
| `cat /proc/1234/environ \| tr '\0' '\n'` | The exact environment it inherited (Lesson 4!) |
| `sudo lsof -p 1234` | Every file/socket the process has open |
| `sudo ss -tulpn` | Listening ports and the processes behind them (from Lesson 4) |

---

## 10. The three questions, at the process layer

The throughline of every lesson, applied to the atom:

**"What launched this?"** — the literal, lowest-level answer: follow the **PPID chain** up the process tree (`pstree`, `ps --forest`) to PID 1. Every process names its parent; the tree *is* the answer to "what launched this," and it bottoms out at systemd, closing the loop with Lesson 4.

**"Why did it stop?"** — a process stops because it *exited* (finished or errored — its exit code), or because it *received a signal* (Section 7) — possibly a `SIGTERM` from an admin, a `SIGKILL`, or, when there's no other explanation, the kernel's **OOM killer** from Lesson 6. The process layer is where "it just disappeared" gets its answer: *something signaled it, or it exited* — and `top`/journal/`dmesg` together tell you which.

**"What environment did it inherit?"** — now fully concrete: the environment isn't an abstraction, it's **attached to the process** and readable at `/proc/<pid>/environ`. The owner, the working directory, the open files, the variables — all of it lives on the process and is inspectable. Lesson 4 told you a service inherits a declared environment; here you can *read the actual inherited environment of any running process.*

> **The stack, now complete bottom to top:** **processes** are the atoms (this lesson) → **services** are processes systemd supervises (Lesson 4) → their **history** is in the journal (Lesson 5) → and the **kernel** beneath them all narrates hardware reality in `dmesg` (Lesson 6). `top` is your live window on the atoms; the other tools are the layers built around them. Master the atom, and the molecule makes sense.

---

## 11. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **program** | Passive instructions in a file on disk | A recipe in a cookbook |
| **process** | A running, live instance of a program | A cook making that recipe |
| **PID** | A process's unique ID number | Its name tag |
| **PPID** | The parent's PID — who created it | The cook who trained this one |
| **process tree** | The whole parent→child hierarchy, rooted at PID 1 | The kitchen's chain of command |
| **fork / exec** | Copy yourself, then become a new program | How every new cook is hired |
| **orphan** | A live process whose parent died (re-parented to PID 1) | A cook whose manager left; reassigned |
| **zombie** | A finished process not yet reaped by its parent | A clocked-out cook whose timecard lingers |
| **state (R/S/D/T/Z)** | What the process is doing right now | Cooking / waiting / I/O-stuck / paused / done |
| **D-state** | Uninterruptible sleep — blocked on I/O | A cook frozen waiting on a stuck oven |
| **VIRT** | Total virtual memory reserved (partly fictional) | Total floor space reserved |
| **RES / RSS** | Real physical RAM in use — *the honest number* | Counter space physically occupied |
| **load average** | Avg processes running + waiting (1/5/15 min) | Cars at a tollbooth; compare to # of booths |
| **`%CPU` > 100%** | A process using more than one core | One cook working several stoves at once |
| **niceness** | Priority hint, −20 (greedy) to +19 (polite) | How willing a cook is to yield a stove |
| **signal** | A predefined message sent to a process | A tap on the shoulder with a meaning |
| **SIGTERM** | Polite "wrap up and exit" — the right default | "Finish your dish and clock out" |
| **SIGKILL** | Force-kill, uncatchable, no cleanup — last resort | "Drop everything and leave NOW" |
| **SIGHUP** | Often "reload config without restarting" | "Re-read your orders" |
| **`top`** | The live, interactive process dashboard | The chef watching the whole kitchen |
| **`nohup`** | Run a process immune to terminal close | A cook who stays after the manager leaves |

---

## 12. Where to go next

You now understand the running atom and how to watch and control it. Natural directions:

- **`/proc` and `/sys` in depth** (also Lesson 6's next-step) — now extra motivated, since you've seen `/proc/<pid>/` is the source of truth every process tool reads from. Understanding `/proc` is understanding where `ps` and `top` *get their numbers.*
- **Writing your own systemd service** — take the "keep a process alive" thread from Section 8 to its proper conclusion: turn a `nohup`-style hack into a real supervised service with logging and restart.
- **cgroups & resource limits** — the hard-limit counterpart to niceness (Section 6): capping CPU and memory per service, and how the OOM killer chooses its victim.
- **Memory deep-dive** — virtual vs resident vs shared, swap, the page cache, and reading memory pressure *before* the OOM killer fires.

The instinct, now at the most fundamental layer: **everything running on a Linux box is a process — so when something's wrong, find the process, read its vitals, and decide what signal it needs.**
