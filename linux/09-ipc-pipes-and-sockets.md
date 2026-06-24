# Lesson 9 — IPC: How Processes Talk (Pipes & Sockets)

> [Lesson 7](07-processes-and-monitoring.md) taught you that a process is an **island**: it has its own memory (Lesson 8), its own open-file table, its own identity. The kernel works *hard* to keep processes isolated — one process flatly **cannot** reach into another's memory. That isolation is the bedrock of a stable system. But it raises an obvious question that the whole rest of Linux quietly depends on: **if processes are walled off from each other, how do they cooperate?** How does your shell hand `grep`'s output to `sort`? How does `systemctl` ask PID 1 to restart a service? How does a "low battery" notification get from a power daemon to your screen? The answer is **Inter-Process Communication (IPC)** — the family of channels the *kernel* lends to processes so they can pass data and signals across the walls. This lesson is the foundation for the next: D-Bus ([Lesson 10](10-dbus.md)) is built directly on the most important mechanism here — the **socket** — and you can't really understand a "message bus" until you understand the pipe it's bolted onto.

---

## 0. The mental model: islands, and the kernel as the post office

Picture every running process as an **island**. By the kernel's design, no island can wade over to another and read its memory — that's *process isolation*, and it's why one crashing program doesn't corrupt the others (Lesson 7). Each island is sovereign.

So how do two islands exchange a message? They can't touch directly — but they share one thing: **they all sit in the same ocean, and the ocean is the kernel.** Every IPC mechanism works the same fundamental way: **a process asks the kernel to create a channel, and the kernel ferries the data** from one process to the other. The processes never touch; the kernel is always the intermediary.

```
   Process A                  KERNEL  (the post office)                  Process B
   ─────────                  ──────────────────────                     ─────────
   write("hi") ──────────────▶  [ a channel it manages ]  ──────────────▶ read() → "hi"
   (island)                     copies bytes across the wall              (island)
```

The kernel is a **post office** sitting between sovereign islands. A process can't mail a letter by swimming it over; it hands the letter to the post office, which delivers it. *Every* form of IPC is just a different **class of postal service** the kernel offers:

- a **fire alarm** that conveys "something happened!" but carries no letter (signals),
- a **one-way pneumatic tube** between two adjacent rooms (pipes),
- a **named mail slot** anyone can drop into (FIFOs),
- a **two-way telephone line** between any two parties (sockets),
- a **shared whiteboard** both parties scribble on directly (shared memory).

> **Why this framing first:** there are half a dozen IPC mechanisms and beginners experience them as an unrelated jumble (`|`, `mkfifo`, `socket()`, `shmget`...). They're not unrelated — they're **the same idea (kernel-mediated channels) tuned for different needs.** Once you see that, the question stops being "what are all these things?" and becomes "*which postal service fits this job?*" — and that's a decision along a small set of axes, which is the next section.

---

## 1. The map: the axes that distinguish every IPC mechanism

Before the mechanisms themselves, here's the *map* that organizes them. Every IPC channel is a point in a space defined by a few questions. Learn the questions and each mechanism becomes "the one that answers them *this* way":

1. **How much data, and what shape?** A bare *notification* (no payload)? A continuous *byte stream*? Or discrete *messages* with boundaries?
2. **Who can use it — and where are they?** Only *related* processes (a parent and its children)? *Any* process on the machine? Or processes on *different machines* across a network?
3. **How are the endpoints named/found?** Is the channel *anonymous* (you must inherit it from a relative) or does it have a *name/address* (so strangers can find it)?
4. **One-way or two-way?** Simplex (A→B only) or duplex (both directions)?
5. **How fast, and at what cost?** Does the kernel *copy* the data (safe, a little slower) or do the processes *share memory directly* (fastest, but you must coordinate access yourself)?

Here's the whole lesson previewed as a table — skim it now, and it'll make sense in full by the end:

| Mechanism | Data shape | Who can talk | Named? | Direction | Note |
|-----------|-----------|--------------|--------|-----------|------|
| **Signal** | notification only (a number) | any process (with permission) | by PID | one-way | the "fire alarm" |
| **Pipe** (anonymous) | byte stream | **related** processes only | no | one-way | the shell's `\|` |
| **FIFO** (named pipe) | byte stream | any local process | yes (a path) | one-way | a named mail slot |
| **Unix domain socket** | stream *or* messages | any **local** process | yes (a path) | **two-way** | **the workhorse — D-Bus uses this** |
| **Network socket** | stream (TCP) / messages (UDP) | across **machines** | yes (IP:port) | two-way | same API, over the wire |
| **Shared memory** | raw memory region | any local process | yes (a key/name) | two-way | fastest; needs a semaphore to coordinate |

> **The one row to tattoo on your brain** is the Unix domain socket: *two-way, between any local processes, addressed by a filesystem path.* It's the general-purpose local telephone line, and it's the substrate beneath D-Bus, X11, Docker, the systemd control channel, database connections, and more. Everything else in this lesson is either *simpler* than it (signals, pipes) or a *variation* on it (network sockets, which are the same thing over a network).

---

## 2. Signals — the fire alarm (notification, no payload)

You already met signals in [Lesson 7](07-processes-and-monitoring.md): `SIGTERM`, `SIGKILL`, `SIGHUP`, sent with `kill`/`pkill`. Now place them on the IPC map: a **signal is the most minimal IPC there is** — it carries *no data*, just a single number meaning "this kind of event happened." The kernel interrupts the target process and runs its handler for that signal.

It's the **fire alarm** of IPC: when the bell rings, everyone knows *what* happened (fire!) but the bell can't carry a paragraph of detail. `SIGTERM` says "please shut down" — nothing more. The richness is in the *agreed meaning* of each number, not in any payload.

```
kill -TERM 1234       # send "please terminate" to PID 1234
kill -HUP  1234       # SIGHUP — by convention, "reload your config"
```

Signals are addressed **by PID** (you must know who you're paging), they're **one-way** (no reply — the receiver can't "answer" the signal), and they're **asynchronous** (they can arrive at any moment, mid-instruction). They're perfect for "wake up / stop / reload" control and useless for "here is 4 KB of structured data." When you need to send *content*, you reach for the next mechanisms.

> **The placement:** signals are IPC stripped to the absolute minimum — pure notification. Keep them in the family, because "send a signal" and "send a message over a socket" are two ends of the same spectrum: *how much can the channel carry?* Signals carry one bit of meaning; sockets carry anything.

---

## 3. Pipes — the `|` you use every day

Here's the beautiful part: **you have been doing IPC since your very first shell command.** Every time you write

```
ps aux | grep firefox | sort -k3 -rn
```

those `|` characters are **pipes** — the kernel wiring one process's output directly into the next one's input. A pipe is an **anonymous, one-way byte stream** between processes, and it's the most-used IPC mechanism on the planet.

### How the `|` actually works

Recall from Lesson 7 that every process is born with three standard file descriptors: **stdin (0)**, **stdout (1)**, **stderr (2)**. Normally stdout goes to your terminal. When the shell sees `A | B`, it asks the kernel for a pipe — a tube with two ends — then **rewires** `A`'s stdout into the *write* end and `B`'s stdin into the *read* end. Now everything `A` prints flows straight into `B`, never touching the terminal. The processes don't know or care; each just reads stdin and writes stdout as always.

```
   ps aux   ──stdout──▶│  pipe  │──▶ stdin   grep firefox  ──stdout──▶│ pipe │──▶ stdin  sort
            (write end)            (read end)
```

The analogy: a **one-way pneumatic tube** between two adjacent rooms. Room A drops papers in; they whoosh to room B. It only goes one way, and the tube has no label on the outside — which leads to the catch.

### The catch: "anonymous" means relatives only

An anonymous pipe has **no name**, so the only way a process can get an end of it is to **inherit it** from the process that created it. That's why pipes work between a shell and its children (the shell forks all the commands in the pipeline and hands each the right ends) — they're all **related**. A random unrelated process across the system *cannot* find or join an anonymous pipe; there's nothing to address. That limitation is exactly what the next two mechanisms remove.

> **The reframe:** the humble `|` is not "shell syntax" — it's a request to the kernel for an IPC channel, plus some file-descriptor rewiring. Internalizing this dissolves the line between "using the shell" and "doing systems programming": they're the same kernel facility. Pipes also explain a performance truth — a pipeline streams (B starts processing A's first bytes before A finishes), it doesn't buffer the whole output, because the tube has limited capacity and the kernel makes a fast writer *wait* when the tube is full (backpressure).

---

## 4. Named pipes (FIFOs) — giving the tube an address

The pipe's one weakness is anonymity: only relatives can connect. A **named pipe**, or **FIFO**, fixes that by giving the tube a **name in the filesystem** — so *any* process that knows the path can connect, related or not.

```
mkfifo /tmp/mytube          # create a named pipe at this path
# terminal 1:
cat /tmp/mytube             # blocks, waiting to READ from the tube
# terminal 2:
echo "hello" > /tmp/mytube  # WRITE into the tube → "hello" appears in terminal 1
```

It behaves exactly like the `|` pipe — one-way byte stream, the kernel ferrying bytes — but now it's a **named mail slot in a public hallway**: anyone who knows where the slot is can drop mail in, and whoever's assigned to that slot reads it. The name decouples the two ends in *time and lineage*: the reader and writer need not be relatives, and need not even start at the same moment.

Notice `mkfifo` creates something that shows up in `ls` — because of Lesson 2's "**everything is a file**." A FIFO *is* a special file. Run `ls -l /tmp/mytube` and the type character (the first column) is **`p`** (for *pipe*), not `-`. Hold that thought; it's about to become a theme.

> **The placement:** a FIFO is "a pipe that grew a name." It's the smallest possible step from the private `|` toward *addressable* communication between strangers — and "addressable between strangers" is precisely what real services need. But FIFOs are still *one-way* and still just a *raw byte stream*. For two-way conversation between arbitrary local processes — the thing daemons actually need — we want the socket.

---

## 5. Unix domain sockets — the local telephone line (the one that matters most)

This is the **most important mechanism in the lesson**, because it's the foundation D-Bus is built on and the way most local daemons communicate. A **Unix domain socket** is a **two-way (duplex)** communication channel between **any two processes on the same machine**, addressed by a **filesystem path**. If a pipe is a one-way tube, a Unix socket is a **full telephone line**: both parties can speak and listen at once, and either can call a known number to start the conversation.

### The client–server shape

Sockets formalize a pattern you'll see everywhere: **one side listens, the other connects.** A *server* process:

1. **creates** a socket,
2. **binds** it to an address (for Unix sockets, a path like `/run/docker.sock`),
3. **listens** for incoming connections, and
4. **accepts** each connecting client, getting a private two-way channel per client.

A *client* process creates a socket and **connects** to that known address. Once connected, both sides `read()` and `write()` freely, in both directions. That's the entire model behind Docker (`/run/docker.sock`), the systemd private bus, database sockets (`/run/postgresql/.s.PGSQL.5432`), the X server, and — next lesson — the D-Bus daemon.

```
   SERVER                                          CLIENT
   socket() ─ bind("/run/foo.sock") ─ listen()
        │                                          socket()
        └──────── accept() ◀───── connect("/run/foo.sock") ──┘
                      │                                 │
                      └─────── two-way byte channel ────┘
                              both read() and write()
```

### The address is a path — and it's a *file* (the `s` type)

A Unix socket's "phone number" is a path on disk, conventionally under **`/run`** — and now [Lesson 2](02-filesystem-and-mounts.md) pays off twice: `/run` is the *runtime state* directory recreated fresh each boot (exactly where transient sockets belong), and the socket itself is a **special file**. List one and the `ls -l` type character is **`s`** (for *socket*):

```
ls -l /run/docker.sock
srw-rw---- 1 root docker 0 Jun 24 09:00 /run/docker.sock
└─ 's' = this is a socket
```

(There's also an **abstract namespace** variant whose name isn't on the filesystem at all — it shows up with a leading `@` in tools — but the path-based kind is the common one to picture.)

### Why sockets, not FIFOs, for real services

Three upgrades over a FIFO make sockets the daemon's choice:

- **Two-way** — a request gets a reply over the *same* connection (a FIFO would need two FIFOs and careful coordination).
- **Connection-oriented** — the server gets a *separate, private channel per client* via `accept()`, so it can serve many clients at once without their bytes intermingling.
- **Message-or-stream + metadata** — Unix sockets can preserve **message boundaries** (datagram mode) and can even pass special payloads like *file descriptors* and the **peer's verified credentials** (uid/pid) across the socket. That credential-passing is huge for security: a server can ask the kernel "who is *really* on the other end of this socket?" and trust the answer. (D-Bus and polkit lean on exactly this — Lesson 10.)

> **The keystone:** the Unix domain socket is the general-purpose local IPC channel — two-way, addressable by a `/run/...` path, one private channel per client, able to carry both data and identity. Almost every "daemon you talk to locally" exposes a Unix socket. **D-Bus is, at its core, a well-known daemon listening on a Unix socket at `/run/dbus/system_bus_socket`** — everything Lesson 10 adds (message format, addressing, routing) sits *on top of* this one mechanism. Get sockets, and the bus is just structure layered over a phone line.

---

## 6. Network sockets — the same idea, across machines

Here's a payoff for very little extra effort: **a network socket is the same API as a Unix socket, with a different kind of address.** Swap the filesystem path for an **IP address + port number**, and the identical `socket → bind → listen → accept` / `connect` dance now connects processes on *different machines* over the network. Same `read()`/`write()`, same client–server shape — the kernel just routes the bytes over the network instead of across local memory.

This is why "sockets" is *the* unifying abstraction of Linux communication: **local IPC and network communication are the same mechanism**, distinguished only by the address family (`AF_UNIX` = a path; `AF_INET` = an IP:port). You already glimpsed the network side in [Lesson 4](04-systemd-services.md):

```
ss -tulpn      # every LISTENING network socket + the process behind it
```

That's the network-socket census — every `bind()`+`listen()` on an IP:port, mapped to its owning service. (Full networking — TCP, ports, firewalls — is a future lesson; here the point is just that it's *the same socket idea* you already understand, pointed at the wire.)

> **The placement:** Unix sockets (local, by path) and network sockets (remote, by IP:port) are siblings — one API, two address families. Learn the shape once and you understand both local daemon communication *and* network services. D-Bus uses the *local* (Unix) kind; a web server uses the *network* kind; the mental model is identical.

---

## 7. Shared memory & friends — the fastest channel (and its catch)

One family remains, at the opposite end of the spectrum from signals. With every mechanism so far, the kernel **copies** data from one process to the other — safe, but it costs a copy. For maximum speed, **shared memory** lets two processes map the *same physical region of RAM* into both their address spaces, so one writes and the other instantly sees it **with no copy and no kernel round-trip.** It's the **shared whiteboard**: both parties scribble on the *same* board, so there's nothing to deliver.

That speed comes with a catch the other mechanisms didn't have: **no built-in turn-taking.** With a pipe or socket, the kernel naturally orders things; with a shared whiteboard, if both write at once you get gibberish (a *race condition* — Lesson 7's territory). So shared memory is almost always paired with a **semaphore** — a kernel-managed counter that acts as a *talking stick*, ensuring only one party scribbles at a time. Together with **message queues** (the kernel holds a queue of discrete labeled messages), these three — **shared memory, semaphores, message queues** — are the classic "System V / POSIX IPC" trio for high-performance local communication (databases, in-memory caches, audio servers).

You can see them on a running system:

```
ipcs           # list System V shared memory segments, semaphores, and message queues
```

> **The placement:** shared memory is the "go fast, steer yourself" option — no copy, but *you* own the synchronization. Most software never reaches for it directly; it's the engine room of performance-critical programs. For ordinary "let these two programs talk," pipes and sockets win because the kernel handles the hard parts (ordering, buffering, backpressure) for you.

---

## 8. The unifying lens: it's all file descriptors

Step back and notice the thread running through almost everything above. A pipe end, a FIFO, a Unix socket, a network socket — to the process, **every one of them is a file descriptor you `read()` and `write()`.** This is [Lesson 2](02-filesystem-and-mounts.md)'s "**everything is a file**" delivering its biggest dividend, and [Lesson 7](07-processes-and-monitoring.md)'s "open-file table" revealing its real purpose:

> **A process's file-descriptor table is its switchboard to the outside world.** Some of those descriptors point at real files on disk; others are pipes to sibling processes; others are sockets to local daemons or remote machines. The process uses the *same* `read`/`write` verbs on all of them. The kernel hides "is this a disk file, a pipe, or a TCP connection?" behind one uniform interface.

That's why `lsof` ("**l**i**s**t **o**pen **f**iles") is your master IPC inspection tool from Lesson 7 — its "files" include every pipe and socket. When you list a process's open descriptors, you're literally reading its **list of communication channels**: which files it's reading, which pipes connect it to relatives, which sockets connect it to daemons and the network. The fd table *is* the map of who a process can talk to.

---

## 9. Seeing IPC on your own system

Concepts stick when you watch them live. Here's how to *observe* every mechanism above on a running machine:

**Find a process's channels (the switchboard view):**

```
ls -l /proc/1234/fd        # raw: every fd of PID 1234 — pipes show as 'pipe:[...]', sockets as 'socket:[...]'
sudo lsof -p 1234          # friendly: every open file, pipe, and socket for that process
```

**Census the Unix-domain sockets (local daemons):**

```
ss -xl                     # listening Unix sockets (the local "phone numbers" daemons expose)
sudo lsof -U               # every Unix socket + which processes hold each end
ls -l /run/*.sock          # the socket files themselves — note the 's' type character
```

**Census the network sockets (Lesson 4 callback):**

```
ss -tulpn                  # listening TCP/UDP ports + owning process
```

**Watch a conversation happen (the "aha" moment):**

```
strace -e trace=read,write,socket,connect -p 1234   # see the actual IPC syscalls a process makes, live
```

`strace` is the X-ray: attach it to a process and watch the literal `connect("/run/dbus/system_bus_socket")`, `write(...)`, `read(...)` calls stream past as it talks to other processes. There is no better way to *feel* that IPC is real and constant.

**See the System V IPC objects:**

```
ipcs                       # shared memory segments, semaphores, message queues in use
```

> **The habit:** when you wonder "how is *this* program talking to *that* one?", the answer is always one of these channels — and `lsof -p <pid>` / `ss -x` / `strace` will *show* you which. "What is this process connected to?" is now an answerable question, not a mystery.

---

## 10. The command cheatsheet

Grouped by what you're trying to do.

### Create / use IPC channels

| Command | What it does |
|---------|-------------|
| `cmd1 \| cmd2` | Anonymous pipe: wire cmd1's stdout into cmd2's stdin |
| `mkfifo /path` | Create a named pipe (FIFO) any process can open by path |
| `kill -SIG pid` / `pkill name` | Send a signal (the minimal, payload-free IPC) |

### Inspect channels & endpoints

| Command | What it shows |
|---------|--------------|
| `sudo lsof -p PID` | Every open file/pipe/socket for a process (its switchboard) |
| `ls -l /proc/PID/fd` | Raw fd table — `pipe:[...]`, `socket:[...]` entries |
| `sudo lsof -U` | All Unix-domain sockets and who holds each end |
| `ss -xl` | Listening Unix sockets (local daemon "phone numbers") |
| `ss -tulpn` | Listening network sockets (TCP/UDP) + owning process |
| `ls -l /run/*.sock` | Socket files on disk (type char `s`); FIFOs show type `p` |
| `ipcs` | System V shared memory, semaphores, message queues |

### Watch IPC happen

| Command | What it does |
|---------|-------------|
| `strace -e trace=network,read,write -p PID` | Live trace of a process's IPC syscalls |
| `strace -f -e trace=signal -p PID` | Watch signals delivered to a process |

---

## 11. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **IPC** | Inter-process communication — kernel-mediated channels between processes | Postal services between islands |
| **process isolation** | The kernel rule that no process can read another's memory | Sovereign islands that can't wade over |
| **signal** | Payload-free notification addressed by PID | A fire alarm — event, no detail |
| **pipe (anonymous)** | One-way byte stream between *related* processes; the shell's `\|` | A one-way pneumatic tube between adjacent rooms |
| **FIFO (named pipe)** | A pipe with a filesystem name, so strangers can connect | A named mail slot in a public hallway |
| **Unix domain socket** | Two-way local channel between any processes, addressed by a path | A telephone line with a known local number |
| **client–server** | One side `listen`s at an address; others `connect` to it | A switchboard: callers dial a number that's waiting |
| **network socket** | Same socket API, addressed by IP:port, across machines | The same phone line, long-distance |
| **address family** | What kind of address a socket uses (`AF_UNIX` path / `AF_INET` IP:port) | Local extension vs. full phone number |
| **shared memory** | Two processes mapping the same RAM — no copy, fastest | A shared whiteboard both write on |
| **semaphore** | Kernel counter coordinating access to a shared resource | A talking stick — only the holder may write |
| **message queue** | Kernel-held queue of discrete labeled messages | A pigeonhole rack of sorted notes |
| **file descriptor** | A process's numbered handle to a file/pipe/socket | A line on the process's switchboard |
| **`/run`** | Tmpfs for runtime state where sockets live; fresh each boot | The hallway where today's mail slots hang |
| **`lsof`** | Lists a process's open files — including pipes & sockets | Reading a process's switchboard |
| **`ss`** | Socket statistics — lists Unix & network sockets | The phone directory of open lines |

---

## 12. Where to go next

You now have the communication primitives the rest of the system is built from. The direct sequels:

- **[Lesson 10 — D-Bus](10-dbus.md)** — the immediate payoff. A raw socket lets *two* processes talk, but a desktop or system has *dozens* of services that all need to reach each other — and point-to-point sockets don't scale (no discovery, no standard message format, no broadcast). D-Bus is a **structured message bus** layered on top of a single Unix socket: one switchboard everyone connects to, with addressing and a common language. Everything in that lesson stands on the socket you just learned.
- **Networking (future)** — TCP/IP, ports, DNS, and firewalls: the network-socket half of this lesson, expanded into its own world. `ss -tulpn` is the bridge.
- **[Lesson 4 — systemd Services](04-systemd-services.md)** — revisit **socket units** (`.socket`): systemd can hold a listening socket *itself* and start your service only when the first client connects (socket activation). Now you know what that socket *is*.
- **[Lesson 7 — Processes](07-processes-and-monitoring.md)** — re-read the open-files and signals sections; they read differently once you see fds as a switchboard and signals as minimal IPC.

The instinct to carry forward: **when two programs cooperate, a kernel-mediated channel connects them — a pipe, a socket, a signal, or shared memory.** "How does X talk to Y?" is always answerable: find the channel (`lsof`, `ss`, `strace`), name its type, and the rest of the behavior follows from the kind of postal service it is.
