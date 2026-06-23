# Lesson 4 — systemd Services

> The three questions this lesson teaches you to answer about *any* running (or dead) program on a modern Linux box:
>
> 1. **What launched this?**
> 2. **Why did it stop?**
> 3. **What environment did it inherit?**
>
> If you can answer those three confidently, you understand systemd well enough to debug production. Everything below builds toward that.

---

## 0. The mental model: who's in charge of a Linux machine?

Before a single command, let's build the picture, because systemd only makes sense once you see the hole it fills.

When a Linux machine boots, the kernel starts up, gets the hardware talking, and then does one last thing before it steps back: it starts **exactly one** user-space process and hands it the magic number **PID 1** (Process ID 1). PID 1 is the **ancestor of every other process on the system**. Nothing runs that isn't, somewhere up its family tree, a child of PID 1.

Think of PID 1 as the **building manager of an apartment complex**. The kernel built the building and installed the manager. From then on, the manager is responsible for:

- Turning the lights on in every apartment (starting services at boot).
- Noticing when a tenant moves out unexpectedly and deciding whether to find a new one (restarting crashed services).
- Keeping a directory of who lives where and whether they're home (tracking service state).
- Deciding the *order* things happen — you can't turn on the apartment's network jack before the building's main network switch is powered.

On most modern Linux distributions (Ubuntu, Debian, Fedora, RHEL, Arch, SUSE...), **the program holding PID 1 is `systemd`**. It is the building manager.

> **Why this matters for your three questions:** "What launched this?" almost always traces back to systemd. "Why did it stop?" is a question systemd recorded the answer to. "What environment did it inherit?" is determined by how systemd was configured to launch it. systemd is not just *a* tool here — it's *the* authority. Learning to interrogate it is the whole game.

### A tiny bit of history (so the name makes sense)

The old way (called **SysV init**, or "init scripts") was a pile of shell scripts that ran one after another at boot, like a morning checklist read top to bottom. It worked, but it was slow (everything was sequential) and dumb (a script had no real idea whether the thing it started actually stayed up).

systemd replaced that with something closer to a **dependency-aware orchestrator**: it knows that service B needs service A and the network, so it can start independent things *in parallel* and the dependent things *in the right order*. The "d" is the Unix convention for a **daemon** — a background program that runs without a person sitting at it ("daemon" = a helpful background spirit, the original 1960s MIT coinage, nothing sinister).

So **systemd = "system daemon"**: the background manager of the whole system.

---

## 1. The vocabulary: units, services, and the things systemd manages

systemd doesn't think in terms of "scripts." It thinks in terms of **units**. A unit is *"one thing systemd knows how to manage."* That's deliberately abstract, because systemd manages more than just programs.

Picture the building manager's filing cabinet. Every folder in it is a **unit**, and the folders come in different colored tabs depending on what kind of thing they describe:

| Unit type | File extension | What it represents | Apartment-building analogy |
|-----------|---------------|--------------------|----------------------------|
| **service** | `.service` | A program to run/keep running | A tenant living in an apartment |
| **socket** | `.socket` | A network port or pipe that, when poked, starts a service | A doorbell that summons a tenant when rung |
| **target** | `.target` | A named *grouping* of units — a milestone | "The whole 3rd floor is ready" |
| **timer** | `.timer` | A schedule that triggers a service (cron's replacement) | A recurring calendar reminder |
| **mount** | `.mount` | A filesystem that should be mounted | A storage locker that should be unlocked |
| **device** | `.device` | A piece of hardware systemd tracks | A physical appliance |

For this lesson we care overwhelmingly about **`.service`** units (the tenants) and a little about **`.target`** units (the milestones). But it helps enormously to know that when you type `systemctl`, the "ctl" stands for **control**, and you are the manager's assistant issuing instructions about units.

> **Key reframe:** You will hear people say "service." Internally systemd says "unit." A service *is* a unit — specifically, a unit of type service. When a command says `list-units`, it'll show you services *and* sockets *and* targets, because they're all units. Don't be surprised when you ask about one thing and see its whole family.

---

## 2. Looking at a service: `systemctl status`

This is the command you will run more than any other, so we'll spend real time here. The instinct most people have is to learn the typing and move on. Resist that — the *output* is where the learning is.

```
systemctl status ssh --no-pager -l
```

Let's decode every piece, because each word is doing a job:

- **`systemctl`** — "I'm talking to the system manager."
- **`status`** — "Tell me everything about the current state of..."
- **`ssh`** — "...this unit." (You can write `ssh` or `ssh.service`; systemd assumes `.service` if you don't say otherwise. If you mean a different type, you *must* spell it out, e.g. `ssh.socket`.)
- **`--no-pager`** — Normally systemd pipes long output into a *pager* (a scroll-one-screen-at-a-time viewer like `less`) that takes over your terminal and waits for you to press `q`. `--no-pager` says "just dump it all to the screen and give me my prompt back." This is the single most quality-of-life flag for reading output in scripts, in logs, or when you just want to glance and move on.
- **`-l`** — "long": don't truncate long lines with `...`. Show me the full command paths and messages. Truncated output is where bugs hide.

### Reading the output, line by line

A typical `status` block looks like this (annotated):

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-06-23 09:14:02 UTC; 3h 1min ago
       Docs: man:sshd(8)
   Main PID: 812 (sshd)
      Tasks: 1 (limit: 4567)
     Memory: 5.1M
        CPU: 240ms
     CGroup: /system.slice/ssh.service
             └─812 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

Now the meaning:

**The colored dot (`●`)** is a traffic light. Green/white dot = happy. Red dot = failed. A hollow/white circle = inactive (dead, but not in an error sense). Your eye should learn to land here first.

**`Loaded:`** answers *"Does systemd even know about this unit, and where did it read its definition from?"* This line has three treasures packed into it:
  - `loaded` — systemd successfully parsed a unit file. (If this said `not-found`, you'd have a typo or a missing service.)
  - `/lib/systemd/system/ssh.service` — **the exact file** that defines this service. This is gold: when you want to know "what does this thing actually do," that path is your answer. Memorize the habit of reading it.
  - `enabled; preset: enabled` — whether it's set to start **at boot**. Hold this thought — the difference between "enabled" and "active" is so important it gets its own section (Section 4). For now: this word is about *boot*, not *now*.

**`Active:`** answers your question #2 in advance — *the current run-state and how long it's been that way.* `active (running)` means the program is alive right now. The timestamp and "3h 1min ago" tell you *when it entered this state* — incredibly useful: "oh, it restarted 3 hours ago, right when the deploy happened."

**`Main PID:`** answers question #1 partially — this is the process ID of the *main* process systemd is babysitting, plus its name. If you ever want to attach a debugger or check it in `top`, here's your number.

**`CGroup:`** is the quiet hero of the whole output, and most tutorials skip it. We'll give it its own section (Section 3) because it's the architectural heart of how systemd actually controls things.

**Below CGroup**, indented with tree characters (`└─`), are the **actual processes** that belong to this service. Not just the main one — *every* child it spawned. This is how you catch a service that quietly forked 40 worker processes.

> **Practice the reading, not the typing.** Run `status` on something and force yourself to narrate each line out loud: "loaded from this file, enabled at boot, active since this time, main process is this PID, and these are its children." Do that five times on five different services and you'll never be afraid of the output again.

---

## 3. The architectural keystone: cgroups (control groups)

Here is the concept that separates people who *use* systemd from people who *understand* it. It also quietly answers a question you didn't know you had: **"How does systemd know which processes belong to a service, even ones it didn't directly launch?"**

### The problem cgroups solve

In the old world, "a service" was a fuzzy idea. You'd start a program; it would fork child processes; those would fork grandchildren; some would intentionally **detach** themselves to escape their parent (a classic daemon trick). When you tried to *stop* the service, you'd kill the main process — and the orphaned children would keep running, leaking memory and holding ports. It was like trying to evict a tenant who'd secretly sublet to five other people who all had their own keys. You evict the one on the lease and the place is still full.

### The cgroup solution

The Linux **kernel** (not systemd — this is a kernel feature systemd *uses*) provides **control groups**, or **cgroups**: a way to put processes into a labeled box such that **every child, grandchild, and sneaky detached descendant is automatically a member of the same box**, and *cannot* escape it.

systemd's brilliant move: **it puts each service in its own cgroup.** The cgroup *is* the service, definitionally. So:

- "Which processes belong to `ssh.service`?" → "Everyone in the `ssh.service` cgroup." No guessing.
- "Stop the service completely" → "Kill everyone in the box." No orphans survive. This is why `systemctl stop` is reliable where old `kill` was not.
- "How much memory/CPU is this service using?" → "Sum it across the box." This is exactly the `Memory:` and `CPU:` lines in the status output.

Back to the analogy: a cgroup is like giving each apartment a **single master door that locks everyone inside it out at once**, no matter how many people the tenant snuck in. The manager doesn't need a list of every occupant — they just lock the one door.

The `CGroup:` line in `systemctl status` is literally showing you the contents of that box. Now you know why it's the most architecturally honest line in the output: it's not a guess about what belongs to the service, it's the *definition* of the service.

> **Why this is worth your attention:** Most "the service won't die" or "I stopped it but the port is still in use" mysteries dissolve the moment you think in cgroups. The unit of life and death in systemd is the *box*, not the *process*.

---

## 4. The single most misunderstood distinction: `enabled` vs `active`

If you remember one conceptual thing from this lesson, make it this. People conflate these constantly and it leads to "but I started it, why is it gone after reboot?!" confusion.

These two words answer **two completely different questions about time:**

| Term | Question it answers | Time frame | Analogy |
|------|--------------------|-----------|---------|
| **active** | Is it running *right now*? | The present moment | Is the tenant *home* right now? |
| **enabled** | Will it start *automatically at boot*? | Every future boot | Does the tenant have a *lease* that auto-renews? |

These are **independent.** All four combinations are real and meaningful:

- **active + enabled** — Running now, and will come back after a reboot. (The normal state of a service you depend on, like `ssh`.)
- **active + disabled** — Running now, but a reboot will *not* bring it back. (You started it by hand for a one-off; it'll vanish on reboot. A common "gotcha.")
- **inactive + enabled** — Not running now, but *will* start at next boot. (E.g. you stopped it temporarily for maintenance but left the lease intact.)
- **inactive + disabled** — Not running and won't come back. (Fully parked.)

### The two commands map cleanly onto the two concepts

This is the payoff — once you see the table above, the commands stop being arbitrary:

**Commands about the PRESENT (active):**
```
sudo systemctl start ssh      # make it active now
sudo systemctl stop ssh       # make it inactive now
sudo systemctl restart ssh    # stop then start (a clean bounce)
```

**Commands about the FUTURE / BOOT (enabled):**
```
sudo systemctl enable ssh     # add the auto-start-at-boot lease
sudo systemctl disable ssh    # tear up the lease
```

> **The classic trap, stated plainly:** `start` does **not** survive a reboot. `enable` does **not** start it right now. If you want both — "run it now *and* every boot from now on" — you either run both commands, or use the shortcut: `sudo systemctl enable --now ssh`. The `--now` flag means "and also do the present-tense version." It's the single most useful flag people don't know exists.

### Why does `enable` need `sudo` but `status` doesn't?

`status` is *reading* — anyone can look at the directory. `start`/`stop`/`enable`/`disable` are *changing the system's behavior* — that's the building manager's authority, and you need administrator rights (`sudo` = "do this as superuser") to wield it. The permission boundary maps onto "looking" vs "changing." This is a recurring Linux theme.

### How `enable` actually works under the hood (and why it demystifies a lot)

When you `enable` a service, systemd doesn't flip a hidden switch in a database. It does something refreshingly physical: **it creates a symbolic link** (a "symlink" — a pointer file) from the service into a directory that represents "things to start at boot." `disable` deletes that link. That's the entire mechanism.

This is why enabling is durable across reboots (the link is a real file on disk) and why it's separate from running (creating a pointer doesn't launch anything). The "lease" in the analogy is a literal piece of paper in a literal folder. We'll see *which* folder when we talk about targets (Section 9).

---

## 5. When things go wrong: `--failed` and reading failure

Your question #2 — "why did it stop?" — gets its dedicated tooling here.

```
systemctl --failed
```

This shows you **only the units that are in a failed state** — a triage list. Think of it as walking into the building and asking "just tell me which apartments have a problem right now," instead of inspecting all 200. On a healthy machine this list is empty, and that emptiness is itself information: "nothing is broken at the systemd level."

What does **failed** actually mean? A service is `failed` when it exited in a way systemd considers *abnormal* — it crashed, returned an error code, timed out on startup, or hit a restart limit (more on that in Section 7). It's different from `inactive`, which is the calm "not running, on purpose" state. **Failed = died badly. Inactive = resting peacefully.**

When you find a failed service, the investigation flow is:

1. `systemctl status the-service --no-pager -l` — the status block for a *failed* service is far richer than for a healthy one. The `Active:` line will say `failed`, show the **exit code** or signal that killed it, and the bottom of the output includes the **last few log lines** the service produced before dying. That snippet alone solves a huge fraction of cases.

2. If the snippet isn't enough, you go to the full logs with `journalctl -u the-service` (the journal is systemd's logging system — its own deep topic, but `-u` means "unit" and shows you everything that service ever logged). The status block is the headline; the journal is the full story.

> **Conceptual point:** systemd *remembers* why a service died even after it's dead. The old world lost that information — a crashed daemon left you guessing. systemd captures the exit code, the signal, the timing, and the final log output, and holds onto it. "Why did it stop?" is answerable *because systemd treats death as data worth recording.*

---

## 6. Surveying the whole system: the `list-*` commands

Three commands, easily confused, that answer three different "show me everything" questions. The trick to keeping them straight is to ask **"a list of *what*, exactly?"**

### `systemctl list-units --all`

A list of **units systemd currently knows about in memory**, including (`--all`) the inactive ones it would otherwise hide. This is a snapshot of the *running system's* awareness — the manager's mental roster of everyone, present or absent. Without `--all`, it politely hides the inactive ones to reduce noise; *with* `--all`, you see the complete picture including the empty apartments.

Use it when you want to know: *"What is the state of everything right now?"*

### `systemctl list-unit-files`

A list of **unit files installed on disk**, and their *enablement* state (enabled/disabled/static/etc.). Note the subtle but crucial difference from the previous command: this is about **files that exist**, not **units that are loaded/running**. 

The analogy: `list-units` is the manager walking the building seeing who's home; `list-unit-files` is the manager reading the filing cabinet of *every lease document that exists*, whether or not that tenant ever moved in.

Use it when you want to know: *"What services *could* run on this machine, and which are set to start at boot?"*

> **The distinction is the lesson.** A unit *file* can exist on disk without being loaded into memory; a unit can be loaded without its file being enabled. "On disk" vs "in memory" vs "enabled at boot" are three separate axes. These two commands let you inspect different axes. If you ever feel lost, ask which axis you're looking at.

You'll also notice a state called **`static`** in `list-unit-files`. A static unit *can't* be enabled or disabled because it has no install instructions — it exists only to be pulled in *by other units* as a dependency, never started on its own. Like a utility closet: nobody "lives" there, but other apartments depend on it.

---

## 7. Restart loops: when "self-healing" becomes a heartbeat of failure

This deserves its own section because it's a real production trap and it beautifully illustrates a systemd design tension.

One of systemd's best features is **automatic restart**: if a service is configured with a restart policy and it crashes, systemd brings it right back. Self-healing infrastructure — wonderful. The building manager notices a tenant collapsed and immediately calls a replacement.

But consider a service that crashes *because of a permanent problem* — a bad config file, a missing database, a port already taken. It starts, fails instantly, restarts, fails instantly, restarts... This is a **restart loop** (or "crash loop"). The manager keeps installing new tenants into an apartment with no floor; each one falls through immediately, and the manager just keeps sending more.

systemd protects against this with **rate limiting**: it tracks how many times a service has restarted within a time window (e.g. "5 starts in 10 seconds"). If the service exceeds that, systemd gives up and parks it in the **`failed`** state with a message like *"start request repeated too quickly."* That message is a *diagnosis*, not a bug: it means "this thing cannot stay up, and I've stopped wasting effort." The right response is never "raise the limit" — it's "go find out *why* it can't start," which loops you back to `status` and `journalctl`.

> **How this informs your three questions:** A service that's `failed` with "repeated too quickly" answers #2 ("why did it stop?") with "it never successfully *started* — it's been thrashing." That reframes your investigation from "what killed a healthy service?" to "what prevents this from ever coming up?" Different question, different fixes.

---

## 8. What's *inside* a service: `systemctl cat` and the anatomy of a unit file

Now we open the lease document itself. Everything systemd does for a service is declared in its **unit file** — a plain text config file. Two commands let you read it:

```
systemctl cat ssh
```

`cat` shows you the **unit file(s) as they exist on disk**, including the file path as a comment header, *and* any overrides layered on top (we'll get to overrides in Section 10). It's the honest "show me the source." Think of `systemctl cat` as *"read me the lease and all its amendments, exactly as written."*

```
systemctl show ssh
```

`show` is different and worth contrasting: it dumps the **fully-resolved, computed properties** of the unit — every single setting, including the hundreds of defaults you never wrote, expressed as `Key=Value` pairs. Where `cat` shows what a *human wrote*, `show` shows what systemd *actually concluded* after applying all defaults and overrides. `cat` is the recipe; `show` is the full nutritional label with every trace ingredient listed.

When debugging "is this setting actually in effect?", `show` is the source of truth, because it reflects the *resolved* value. When asking "what did someone intend here?", `cat` is clearer because it's only the human-authored parts.

### Anatomy of a `.service` file

A service unit file is divided into **sections** marked by `[BracketHeaders]`. Here's an annotated example:

```ini
[Unit]
Description=My Web App
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env
ExecStartPre=/usr/local/bin/myapp-migrate
ExecStart=/usr/local/bin/myapp --port 8080
Restart=on-failure
RestartSec=5
User=myapp

[Install]
WantedBy=multi-user.target
```

Let's read it as three answers to three questions:

**`[Unit]` — "How do I relate to the rest of the system?"** This section is about *identity and ordering*. 
  - `Description` is the human label you see in `status`.
  - `After=` is **ordering only**: "don't start me until these are started." It does *not* mean "I require them" — purely sequence. Like "wait until the elevator is on before I move in," but if there's no elevator you'll still move in.
  - `Requires=` is a **hard dependency**: "if this other unit fails, I should fail too." This is the actual "I cannot live without" relationship.
  - The pairing `After=` + `Requires=` together means "I need it *and* it must come first" — a common, sensible combo. Keeping the two ideas (need vs order) separate is a recurring systemd subtlety.

**`[Service]` — "How do I actually run?"** This is the meat:
  - `Type=` tells systemd *how to tell when I've successfully started*. `simple` (the default) means "the moment you run my ExecStart command, consider me started." Other types like `forking` or `notify` exist for programs that background themselves or actively signal readiness. This is how systemd avoids the old problem of "I think it started but it actually crashed half a second later."
  - `ExecStart=` — **the command that *is* the service.** This is the literal answer to your question #1: "what launched this?" The full path and arguments are right here. This is why `systemctl cat` is your go-to for "what does this thing even run?"
  - `ExecStartPre=` — a command to run **before** `ExecStart`, every time, that must succeed first. In the example it runs a database migration before launching the app. If `ExecStartPre` fails, `ExecStart` never runs — a built-in pre-flight check. (There's a matching `ExecStartPost` for after, and `ExecStop` for shutdown.) Think of it as "before the tenant moves in, the cleaners must finish — and if they can't, the move-in is cancelled."
  - `Restart=on-failure` and `RestartSec=5` — the restart policy from Section 7. "If I die badly, wait 5 seconds and bring me back." (The `RestartSec` delay is itself a gentle anti-crash-loop measure.)
  - `User=myapp` — **the identity the process runs as.** Hugely important for security and for the next section: a service running as `myapp` inherits `myapp`'s permissions and a fairly bare environment, *not* yours.

**`[Install]` — "What happens when someone enables me?"** This section is *only* consulted by `enable`/`disable`. 
  - `WantedBy=multi-user.target` is the instruction: "when enabled, create my boot-time symlink in the `multi-user.target` group." Remember Section 4 — `enable` makes a symlink; *this line* tells it *where*. A unit with no `[Install]` section is one of those `static` units that can't be enabled. The lease analogy completes here: `[Install]` is the clause that says which floor's roster to add you to.

> **This file is the Rosetta Stone for all three questions.** `ExecStart` = what launched it. `Restart`/`Type` = how it behaves when it stops. `Environment`/`EnvironmentFile`/`User` = what environment it inherited. When in doubt, `systemctl cat` the service and *read the file.*

---

## 9. Targets: milestones, not programs

We've mentioned `multi-user.target`. Let's properly understand targets, because they're how systemd organizes the *whole boot* and they answer "what does 'the system is ready' even mean?"

A **target** is a **named grouping of units** — a synchronization point, a milestone. It runs no program of its own. It exists to be a label that means "all the things associated with this milestone are now up."

The analogy: a target is like a stage in a building's morning opening checklist. *"Lobby ready"* isn't a task — it's a milestone that means the doors, lights, and front desk are all up. *"Open for business"* is a higher milestone that depends on *"lobby ready"* plus more.

The important targets:

- **`multi-user.target`** — "the system is fully up as a multi-user, networked, command-line machine." This is the milestone most servers boot to, and it's why server services say `WantedBy=multi-user.target` — "start me as part of the system being properly up." The name is historical: "multi-user" (as opposed to single-user maintenance mode) meant a fully operational system where many people could log in.
- **`graphical.target`** — everything in `multi-user.target` *plus* the graphical desktop. A milestone built *on top of* another milestone (it `Requires` multi-user). This layering is the dependency engine in action.
- **`network.target`** — the milestone "networking is configured," which is why so many services say `After=network.target`.

Targets are the **modern replacement for the old "runlevels"** (numbered system states 0–6 from the SysV era). Where a runlevel was an opaque number, a target is a named, composable milestone with explicit dependencies. If you ever hear "runlevel 3," its spiritual successor is `multi-user.target`; "runlevel 5" is `graphical.target`.

> **Why targets close the loop on boot:** "Enabling a service" (Section 4) means "link me into a target." "Booting" means "reach a target." So enabling `WantedBy=multi-user.target` literally means "when the system works toward the multi-user milestone, pull me along." Boot is just systemd resolving the dependency tree to reach the default target. The pieces connect.

---

## 10. Overrides and drop-ins: changing a service *without* editing its file

This is one of systemd's most elegant ideas, and it solves a problem you'll hit constantly: **"I want to change one setting on a service whose file is owned by the system package manager."**

### The problem

The `ssh.service` file lives in `/lib/systemd/system/` — that's **package-manager territory.** When SSH gets an update, the package manager overwrites that file. So if you'd edited it directly, your change gets blown away on the next update. It's like writing your house rules directly onto the building's master blueprint — the next renovation prints a fresh blueprint and your notes are gone.

### The solution: drop-ins (layered overrides)

systemd lets you place **additional, small config files** that **layer on top of** the original without touching it. These are called **drop-ins**. The original file is the base layer; your drop-in is a transparent sticky note on top that says "...except change *this* one setting." When systemd computes the final config, it reads the base, then applies the drop-ins on top.

The analogy: the master blueprint stays pristine. You add a **separate amendment sheet** clipped to it: "Apartment 4 — additional rule: quiet hours after 10pm." Renovations reprint the blueprint, but your amendment sheet stays clipped on. (And `systemctl cat`, helpfully, shows you the blueprint *and* all clipped amendments together — that's the override-aware behavior mentioned in Section 8.)

### The command that does it right

```
sudo systemctl edit mysql
```

`systemctl edit` is the *correct, safe* way to override. It doesn't open the original file — it opens a **fresh, empty drop-in file** for you (conventionally stored at `/etc/systemd/system/mysql.service.d/override.conf`). You write *only the settings you want to change*, in the same `[Section]` `Key=Value` format. systemd handles placing the file in the right directory. You never touch, and never risk, the original.

Note the two directories at play, and why they matter:
- `/lib/systemd/system/` (sometimes `/usr/lib/...`) — **the vendor/package's files.** Don't edit these.
- `/etc/systemd/system/` — **the administrator's files** (you). Drop-ins and your custom units live here, and `/etc` always wins over `/lib`. The convention "`/etc` is for *your* local configuration, vendor defaults live elsewhere" is a Linux-wide principle, not just systemd — you'll see it everywhere.

### The step everyone forgets: `daemon-reload`

```
sudo systemctl daemon-reload
```

Here's a subtlety that trips up *everyone* at least once. systemd reads unit files into memory and works from that in-memory copy for speed (remember `list-units` = "in memory" vs `list-unit-files` = "on disk"). When you change a file on disk — whether via a drop-in or by hand — **the in-memory copy is now stale.** systemd is still managing the old version it read earlier.

`daemon-reload` tells systemd: **"re-read all the unit files from disk and rebuild your in-memory picture."** It's the manager re-reading the updated filing cabinet. Until you do this, your edits exist on disk but systemd is ignoring them.

> **Important nuance:** `daemon-reload` reloads the *configuration*, but it does **not** restart the *service*. The running process is still the old one, running with the old settings, oblivious. To make a config change take effect on the actual process, you usually need both: `daemon-reload` (so systemd knows the new config) **and then** `restart` (so the process relaunches under it). Two distinct steps for two distinct things — reloading the *plan* vs relaunching the *worker*. (`systemctl edit` is kind enough to run `daemon-reload` for you automatically, but a hand-edited file needs you to run it yourself — a frequent "why isn't my change working?!" moment.)

---

## 11. Environment inheritance — your question #3, head-on

This is the most slippery of the three questions and the one that produces the most "but it works when *I* run it!" frustration. Let's untangle it carefully, because the confusion is almost always about *whose* environment a service gets.

### The core misconception

When *you* log into a shell and run a program, that program inherits **your** environment: your `$PATH`, your `$HOME`, the variables your `.bashrc` set, your locale, everything your login session accumulated. It's natural to assume a systemd service gets the same.

**It does not.** A systemd service is launched by **PID 1**, not by your shell. It inherits **systemd's** environment, which is deliberately *minimal and clean* — a near-empty room. It does **not** read your `.bashrc`, does **not** see variables you exported in your terminal, and often has a stripped-down `$PATH`. 

The analogy: when you run a program yourself, it moves into *your* fully-furnished apartment with all your stuff. When systemd runs a service, it moves into a *bare, freshly-cleaned unit* with only the essentials the building provides. People debug for hours because they assumed the service had their furniture.

This is **by design and it's a good design** — services should be reproducible and not depend on whatever happened to be in some admin's shell. But it means **you must declare a service's environment explicitly**, which is exactly what these unit-file directives do:

- `Environment=NODE_ENV=production` — set a variable inline, right in the unit file. Simple, visible.
- `EnvironmentFile=/etc/myapp/env` — read variables from a separate file. Good for secrets or long lists you don't want in the unit file, and lets you change them without editing the unit.

So when your service can't find a command ("`node: not found`") even though *you* can run `node` fine, the answer is almost always: **the service's `$PATH` doesn't include where `node` lives, because it didn't inherit your shell's `$PATH`.** The fix is to declare it — full paths in `ExecStart`, or an explicit `Environment=PATH=...`.

> **This is the whole reason question #3 exists.** "What environment did it inherit?" has a surprising answer — "almost nothing, plus whatever the unit file declares" — and that surprise is the source of a huge class of bugs. Internalize: *a service's environment is what the unit file says it is, not what your shell has.*

### Inspecting and proving it

`systemctl show ssh` (from Section 8) includes the resolved `Environment=` properties — that's how you *verify* what a service will actually see, rather than guessing. When in doubt, `show` tells the truth.

---

## 12. User services: a second, personal systemd just for you

Everything so far has been about the **system instance** of systemd — the PID 1 building manager that runs services as root or as service accounts, machine-wide, starting at boot before anyone logs in.

But there's a second flavor: the **user instance.** When *you* log in, systemd starts a *personal* systemd process **just for your user account**, running as *you*. It manages services that:

- run **as you**, with your permissions,
- start when *you* log in (not at system boot),
- are perfect for *your* personal long-running tools — a sync daemon, a personal web service, a dev environment — things that shouldn't need root and shouldn't run when you're not logged in.

The analogy: the system instance is the **building manager** for the whole complex. The user instance is **your personal household assistant** inside your own apartment — it can only act within your space, with your authority, and it clocks in when you arrive.

You talk to it by adding `--user` to the same commands:

```
systemctl --user status
systemctl --user start my-personal-thing
systemctl --user enable my-personal-thing
```

Notice two things that follow directly from "it's *yours*":

1. **No `sudo` needed.** You're managing *your own* services with *your own* authority. There's no system-wide permission boundary to cross — it's your apartment, you have the keys. (Contrast Section 4, where system services needed `sudo` because they affect everyone.)
2. **The unit files live in *your* home directory** (`~/.config/systemd/user/`), not in system directories. Your leases, your filing cabinet.

### The environment commands — and why they exist here specifically

The user instance has its **own environment block**, separate from any single shell, that all your user services inherit. systemd gives you commands to inspect and shape it:

```
systemctl --user show-environment
```
*"Show me the environment block my user services will inherit."* This is question #3, scoped to your personal services. It's the user-instance analog of the inspection you'd do with `show` on a system service.

```
systemctl --user set-environment VAR=value
systemctl --user unset-environment VAR
```
*Set* or *remove* a variable in that shared user environment. 

Why does this matter more for user services than system ones? Because user services are exactly where you'd want to share something dynamic across several of your personal tools — say a `DISPLAY` for GUI apps, an API token for a few of your daemons, or a path to a project directory. `set-environment` injects it into the user instance's block so **every** subsequently-started user service sees it, without editing each unit file. It's like leaving a note on your household assistant's desk that applies to every chore they start from now on.

> **Connecting it back:** The fact that user services have a *separate, inspectable, mutable* environment is the same lesson as Section 11 — services inherit a managed environment, not your shell's — just applied to your personal systemd. Two systemd instances, same principle: the environment is *declared and managed*, never assumed.

---

## 13. Putting it all together: the three questions, answered

Let's return to where we started and show how every tool maps to the question it answers. This is the section to reread before you debug something real.

### "What launched this?"
- `systemctl status <svc> --no-pager -l` → the `Main PID`, the `CGroup` tree of every related process, the `Loaded:` path to the defining file.
- `systemctl cat <svc>` → the `ExecStart=` line: the literal command, with full path and arguments. The definitive answer.
- Trace upward: every process descends from PID 1 (systemd), and `status` tells you which *service* (cgroup) it belongs to.

### "Why did it stop?"
- `systemctl --failed` → triage: what's broken right now.
- `systemctl status <svc>` → the `Active:` line shows `failed` plus the **exit code/signal** and **timestamp**; the bottom shows the **last log lines** before death.
- `journalctl -u <svc>` → the full log history when the snippet isn't enough.
- Watch for **"start request repeated too quickly"** → it's not dying, it's *thrashing*; investigate why it can't ever start (Section 7).
- The deep reason this is answerable: systemd **records death as data** — exit codes, signals, timing, final output.

### "What environment did it inherit?"
- Mental model first: **NOT your shell's environment** — a clean, minimal one plus whatever the unit declares (Section 11).
- `systemctl cat <svc>` → the `Environment=` / `EnvironmentFile=` / `User=` lines: what was *declared*.
- `systemctl show <svc>` → the *resolved* environment, the source of truth for "what it actually sees."
- For your personal services: `systemctl --user show-environment`.
- Most "works for me but not as a service" bugs are a `$PATH` or missing-variable issue rooted right here.

### The workflow muscle memory
When something's wrong, the reflexive sequence is almost always:

1. `systemctl status <svc> --no-pager -l` — *look first, always.*
2. If failed and the snippet's not enough → `journalctl -u <svc>`.
3. To understand *what it's supposed to do* → `systemctl cat <svc>`.
4. To verify *resolved* settings/environment → `systemctl show <svc>`.
5. To change something safely → `sudo systemctl edit <svc>` → (`daemon-reload` if hand-edited) → `restart`.
6. To make a change stick across reboots → `enable` (and `--now` to also start it).

---

## 14. The cast of characters: common & critical services on Ubuntu

Up to now we've used `ssh` as a stand-in for "a service." But when you actually run the inspection commands on your own Ubuntu machine, you'll be staring at *dozens* of services with cryptic names, and the natural panic is "which of these matter, and which are weird?" This section is your field guide to the regulars — the tenants you *expect* to find in a healthy Ubuntu building.

The mental shift: a running Linux system is not "your program plus some noise." It's a small society of cooperating background services, each owning one responsibility, most of them started at boot and quietly humming. Knowing the regulars is exactly what lets you later spot the stranger — you can't recognize an intruder if you never learned who's supposed to be there.

I'll group them by *the job they do*, because that's how you'll reason about them.

### The absolute core — if these are unhealthy, the system is in trouble

- **`systemd` itself / `init`** — PID 1, the manager. It doesn't show as a normal service; it *is* the thing managing all the others. Mentioned only so you remember it's always there at the root.
- **`systemd-journald`** — the **journal daemon**: the logging service that captures everything every other service prints. This is the engine behind `journalctl`. If it's down, you go blind — no logs. Critical.
- **`systemd-logind`** — manages **user login sessions** and seats (who's logged in, on which terminal/display, and cleaning up when they leave). It's why your user services start when you log in (Section 12).
- **`dbus` (`dbus.service` / `dbus-broker`)** — the **D-Bus message bus**: a local "switchboard" that lets services talk to each other through structured messages instead of ad-hoc hacks. Tons of desktop and system components depend on it. Think of it as the building's internal phone system.
- **`systemd-udevd`** — the **device manager**: when you plug in a USB stick or the kernel detects hardware, udev is what creates the device entry and fires the right setup. The building's "facilities" crew reacting to physical changes.

### Networking — "is this box on the network?"

- **`systemd-networkd`** and/or **`NetworkManager`** — these *configure your network interfaces* (IP addresses, routes). Servers/cloud images often use `systemd-networkd`; desktops and laptops typically use `NetworkManager` (it handles Wi-Fi roaming, VPNs, the network applet). You'll usually have one or the other as the primary. This is the building's connection to the city grid.
- **`systemd-resolved`** — the **DNS resolver**: turns names like `github.com` into IP addresses for the whole system. When "the internet is down but ping by IP works," this is your first suspect.
- **`ssh` / `sshd`** — the **OpenSSH server**: lets you (and others) log in remotely. On a server this is critical *and* security-sensitive — it's the front door, so it's also the thing attackers probe most. Present on most servers, often *not* installed by default on a fresh desktop.

### Time, scheduling, and housekeeping

- **`systemd-timesyncd`** (or **`chrony`/`ntp`** on servers) — keeps the **clock synchronized** with internet time servers. More important than it sounds: wrong clocks break TLS certificates, logs, and authentication.
- **`cron`** and **`systemd` timers** — **scheduled task** runners. `cron` is the classic; timers are the systemd-native version (a future lesson). Worth knowing both can launch things on a schedule — which, foreshadowing Section 16, is also a favorite hiding spot for malicious persistence.
- **`unattended-upgrades`** — an Ubuntu-specific service that **automatically installs security updates**. Generally a *good* citizen; occasionally the reason a server reboots or a package changes "on its own."
- **`snapd`** — manages **snap packages** (Ubuntu's containerized app format). Many default Ubuntu tools ship as snaps, so this is usually present and busy.

### Logging, auth, and the desktop

- **`rsyslog`** — a traditional logging service that often runs *alongside* journald, writing classic text logs to `/var/log` (e.g. `/var/log/syslog`). Some tooling still expects those files.
- **`polkit` (PolicyKit)** — the **fine-grained authorization** broker: when a desktop app asks "can this user do this privileged action?", polkit decides and (if needed) pops the password dialog. A more nuanced gatekeeper than raw `sudo`.
- **`gdm` / `accounts-daemon` / `udisks2` / `cups`** (desktop & peripherals) — the **graphical login screen** (`gdm`), user-account info (`accounts-daemon`), removable-disk auto-mounting (`udisks2`), and printing (`cups`). You'll see these on desktops/laptops; a headless server typically won't run most of them.

### Often-present application services

Depending on what the machine *does*, you'll also find the "purpose" services — the reason the box exists: a database (`postgresql`, `mysql`/`mariadb`), a web server (`nginx`, `apache2`), a container engine (`docker`, `containerd`), and so on. These aren't "system" services — they're the *tenant you installed on purpose*. The healthy instinct: you should be able to look at every `enabled` service and say "yes, I know why that's here."

> **The point of this catalogue isn't memorization** — it's calibration. After reading your own service list a few times with this guide beside you, the regulars become familiar furniture, and your attention is freed to notice the *one* name that doesn't belong. That's the skill Section 16 builds on.

---

## 15. Inspecting what's actually running — and what to look for

Now the practical heart of your request: *how do I survey my own machine, and what am I actually looking at?* We'll go from the broadest view down to the focused one, and at each step I'll say **what to look for**, not just what to type.

### Step 1 — The health triage (start every inspection here)

```
systemctl --failed
```

You met this in Section 5. Make it your *first* move, always, because it's the fastest "is anything actually broken?" question. **What to look for:** ideally, "0 loaded units listed" — silence is good news. Anything here is a real problem worth opening with `status`.

### Step 2 — The full roster, grouped by state

```
systemctl list-units --type=service --all
```

This lists **every service unit systemd currently knows about**, active or not (Section 6). The additions earn their keep:

- `--type=service` filters out the sockets/targets/mounts so you see *only services* — far less overwhelming than the raw `list-units`.
- `--all` includes the inactive ones, so nothing hides.

**What to look for, column by column:**

- The **`LOAD`** column should say `loaded`. A value like `not-found` means a unit is referenced but its file is missing — often harmless leftovers, occasionally a clue.
- The **`ACTIVE`/`SUB`** columns tell you the run-state: `active`/`running` (alive), `active`/`exited` (ran once and finished on purpose — normal for one-shot setup tasks, *not* a bug), `inactive`/`dead` (parked), `failed` (broken).
- The **`DESCRIPTION`** is your first sniff test: does each name + description make sense together and match something you'd expect on this machine?

### Step 3 — What's set to launch at every boot

```
systemctl list-unit-files --type=service --state=enabled
```

Remember the axis distinction from Section 6: this is about **files on disk and their boot-time enablement**, not what's running *now*. Filtering to `--state=enabled` answers a specifically valuable question: **"what has standing permission to start itself on every boot?"**

**What to look for:** this is the list you should be able to *fully account for*. Every enabled service is something that will resurrect on reboot whether you're watching or not — so it's exactly where unwanted persistence lives. Read it slowly and ask of each entry: *do I know why this is set to auto-start?* An enabled service you can't explain is the single most useful red flag a casual inspection produces.

### Step 4 — Zoom in on anything interesting

For any service that caught your eye:

```
systemctl status <svc> --no-pager -l     # current state, PID, cgroup tree, recent logs
systemctl cat <svc>                       # the unit file: what does it actually run?
systemctl show <svc> --property=ExecStart,User,FragmentPath,Environment
```

That third command is a precision tool worth highlighting: `--property=` asks `show` (Section 8) for *just* the fields you care about instead of the hundreds it normally dumps. The four chosen here are exactly the "who and what" of a service:

- **`ExecStart`** — the literal command (your question #1).
- **`User`** — what identity it runs as.
- **`FragmentPath`** — the exact unit file on disk that defines it (where to go read/audit it).
- **`Environment`** — what it was handed (your question #3).

### Step 5 — Map services to the processes and ports they own

Two cross-checks that connect systemd's view to the rest of the system:

```
systemd-cgls
```
`systemd-cgls` ("cgroup list") draws the **cgroup tree** — every service as a box with its processes nested inside (Section 3, made visible for the *whole machine* at once). This is the truest picture of "what is running and who it belongs to," because it's organized by the boxes systemd actually controls. **What to look for:** processes living under a service whose name doesn't match what they're doing.

```
sudo ss -tulpn
```
`ss` ("socket statistics") lists **listening network ports** and the process behind each. (`-t` TCP, `-u` UDP, `-l` listening, `-p` show the process, `-n` numeric ports.) **What to look for:** every open port should map to a service you recognize from the catalogue in Section 14. A listening port owned by a process you can't explain is a classic "what is *that*?" moment — and a natural bridge into the next section.

> **The inspection mindset:** you're not memorizing output, you're building a *baseline* — a felt sense of what your machine normally looks like. Run these a few times on a healthy system. The investment pays off the day something is wrong, because deviation from a known-normal baseline is the easiest signal in all of security to read.

---

## 16. How would I know a service is *suspicious*?

This is the question that ties everything together, and the honest framing up front: **there is no single command that prints "this service is malware."** Detection is about *contrast with normal* — which is exactly why Section 14 (who belongs) and Section 15 (how to look) came first. A suspicious service is one that violates the patterns you now know to expect. Let's make those patterns concrete.

> **Important context before the checklist:** the goal here is *defensive awareness* on your own machine — learning to read your own system critically. None of this is a substitute for dedicated security tooling on a truly compromised host; if you ever genuinely suspect a breach on an important machine, the right move is to isolate it and bring in proper incident-response tools, because a sophisticated attacker can tamper with the very commands you'd use to look. What follows trains your everyday instinct.

### Signal 1 — The name doesn't add up

Malware loves to **hide in plain sight** by mimicking a real service's name — `systemd-logind` is real, but `systemd-logind` with a subtly different spelling, or `systemd-helperd`, or `sytemd-networkd` (note the missing `s`) is not. Attackers count on your eye sliding past a familiar-looking word.

**What to look for:** names that are *almost* a known service but off by a character; generic-but-official-sounding names (`update-service`, `system-helper`, `kworkerd`) that don't appear in the Section 14 catalogue; or random-looking strings. When a name catches your eye, immediately `systemctl cat` it and read the `ExecStart` — the name is a costume, but the command it runs is the truth.

### Signal 2 — The `ExecStart` is doing something a legitimate service wouldn't

This is the strongest single tell, because the command can't lie about what it does. **What to look for in `ExecStart` (and the unit file generally):**

- Binaries running from **odd locations** — a real service runs from `/usr/bin`, `/usr/sbin`, `/usr/lib`, or a clear app directory. A service launching something out of `/tmp`, `/dev/shm`, `/var/tmp`, a user's home, or a hidden `.dotfile` directory is a glaring anomaly — those are *scratch* spaces, not where installed software lives.
- **Piped-shell one-liners**, e.g. an `ExecStart` that runs `bash -c` to `curl`/`wget` a remote address and pipe it straight into a shell. Legitimate services run an installed program; they don't download-and-execute code on every start. This pattern is a hallmark of malware fetching its real payload.
- **Encoded or obfuscated commands** — base64 blobs, long hex strings, or commands clearly written to be unreadable. Real services have nothing to hide and read plainly.
- **Reverse-shell shapes** — commands connecting *outbound* to an IP and handing over a shell. The combination of "a service whose whole job is to open a connection to an external address and run commands from it" is rarely benign.

### Signal 3 — It's running as the wrong identity

**What to look for:** check the `User=` (via `systemctl show <svc> --property=User`). A small utility service running as **`root`** when it has no business needing root privileges is suspicious — attackers want maximum privilege. Conversely, a service running as an unexpected or unfamiliar user account can indicate something installed outside your knowledge. Match the privilege to the job: a printing helper doesn't need to be root.

### Signal 4 — It lives in the wrong place / was installed strangely

Recall the directory principle (Section 10): vendor units live in `/lib/systemd/system`, *your* local units in `/etc/systemd/system`. **What to look for:** a service whose `FragmentPath` is in `/etc/systemd/system` (admin-installed) that *you* don't remember installing — that's a unit someone with root access put there deliberately. Legitimate packages almost always install to `/lib`. A hand-placed unit in `/etc` is exactly how an attacker establishes persistence, because it survives reboots and package updates.

Also check **timers and the user instance** — persistence hides in less-obvious corners: `systemctl list-timers --all` (scheduled triggers) and `systemctl --user list-units` (services running under your account, which don't need root to install). A thorough look covers all three: system services, timers, *and* user services.

### Signal 5 — The behavior is wrong even if the name looks fine

**What to look for:**

- A service in a **restart loop** (Section 7) that you don't recognize — something repeatedly trying and failing to establish itself.
- A "system" service making **constant outbound network connections** (cross-reference with `ss -tulpn` and the cgroup tree) — especially to raw IP addresses rather than known update servers.
- **Unusual resource use** — the `Memory:`/`CPU:` lines (or `systemd-cgtop`, which shows live resource use *per cgroup*, i.e. per service) revealing a no-name service quietly burning CPU. Cryptominers, the most common Linux malware payload, give themselves away here: they exist to consume CPU.
- A **recently created** unit file (check its timestamp) on a machine you haven't changed lately — "appeared without a corresponding action by me" is intrinsically suspicious.

### The disciplined workflow for "is this thing suspicious?"

When something trips your instinct, walk it down deliberately:

1. **`systemctl cat <svc>`** — read the unit file. Where does `ExecStart` point? What user? Read it like a skeptic.
2. **`systemctl show <svc> --property=FragmentPath,User,ExecStart`** — confirm the resolved truth (in case of drop-in overrides layering on hidden behavior).
3. **`systemctl status <svc> --no-pager -l`** — see its live processes (cgroup tree) and recent log lines.
4. **`sudo ss -tulpn` / `systemd-cgtop`** — does it hold a port or burn resources it shouldn't?
5. **Cross-reference against Section 14** — is this a known regular, or a stranger?
6. **Check the file's location and timestamp** — `/lib` (vendor, expected) vs `/etc` (local, did *I* do this?), and "when did this appear?"

If after that walk you still can't explain *why a service exists, what it runs, and as whom* — that unexplained gap **is** the finding. You don't need certainty of malice; "I cannot account for this" is itself the signal that justifies digging deeper or escalating.

> **The whole lesson in one sentence:** security inspection isn't a magic command, it's *fluency in normal* — and you built that fluency in the preceding fifteen sections. A suspicious service is simply one that breaks the patterns systemd taught you to expect: it runs from the wrong place, as the wrong user, doing something a real service wouldn't, that you can't account for. Learn the regulars, and the stranger announces itself.

---

## 17. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **systemd** | The system & service manager running as PID 1 | The building manager |
| **PID 1** | The first process; ancestor of all others | The manager the kernel installs |
| **unit** | Any single thing systemd manages | A folder in the filing cabinet |
| **service** | A unit that is a program to run/keep alive | A tenant |
| **target** | A named milestone grouping units | "This floor is ready" |
| **cgroup** | Kernel box that contains a service's every process | The apartment's single master lock |
| **active** | Running right now | Tenant is home |
| **enabled** | Set to auto-start at boot (via a symlink) | Lease that auto-renews |
| **failed** | Exited abnormally; death recorded | Apartment flagged with a problem |
| **unit file** | The text config defining a service | The lease document |
| **drop-in / override** | Layered config that modifies a unit without editing it | A clipped-on amendment sheet |
| **daemon-reload** | Re-read unit files from disk into memory | Manager re-reads the filing cabinet |
| **ExecStart** | The command that *is* the service | "What launched this" |
| **ExecStartPre** | A must-succeed command run before ExecStart | Pre-move-in cleaning |
| **`/lib/systemd/system`** | Vendor/package unit files (don't edit) | The master blueprint |
| **`/etc/systemd/system`** | Your local unit files & overrides (do edit) | Your amendment sheets |
| **user instance** | A personal systemd running as you, for your services | Your household assistant |
| **journald** | The logging daemon that captures all service output | The building's CCTV recorder |
| **D-Bus** | Local message bus services use to talk to each other | The internal phone switchboard |
| **`FragmentPath`** | The on-disk file a unit is actually defined by | The exact lease document's location |
| **`systemd-cgls`** | Shows the whole cgroup tree (services + their processes) | A floor plan of every locked apartment |
| **`ss -tulpn`** | Lists listening ports and the process behind each | Which apartments have a phone line to outside |
| **baseline** | Your learned sense of what "normal" looks like | Knowing every regular tenant by sight |

---

## 18. Where to go next

You now have the conceptual scaffolding. The natural next lessons, each of which deepens one pillar here:

- **`journalctl` in depth** — the logging system behind "why did it stop?" (filtering by time, following live, severity levels).
- **Writing your own service from scratch** — take the Section 8 anatomy and build, enable, and debug a real unit.
- **Timers** — the modern, systemd-native replacement for cron, built on the same unit/dependency machinery you now understand.
- **Resource control via cgroups** — `MemoryMax`, `CPUQuota`, and friends: now that you know a service *is* a cgroup, you can cap what the box is allowed to consume.

The throughline for all of them is the same instinct this lesson drilled: **don't guess about a process — ask systemd, and read what it tells you.**
