# Lesson — Package Management: `dpkg`, `apt`, `apt-get`, `aptitude`

> This is the lesson behind **your broken upgrade.** When an `apt full-upgrade` half-finishes and leaves the system in a state where *every* subsequent `apt` command fails with "you have held broken packages" or "dpkg was interrupted, you must manually run...", it feels like a single catastrophic event. It isn't. It's a **two-layer machine** that got stuck *between* its two layers, and once you can see those two layers clearly, the recovery stops being voodoo and becomes a checklist. So this lesson is built bottom-up: first the crate, then the worker who installs one crate, then the manager who orchestrates thousands of crates — and only *then* the daily commands, because the commands are obvious once the machine underneath them is.

---

## 0. The mental model: a two-layer supply chain

Here is the single idea that makes all of `dpkg`/`apt`/`apt-get`/`aptitude` fall into place. Debian/Ubuntu package management is **two completely separate tools stacked on top of each other**, and almost every confusion (and every recovery command) comes from not knowing which layer you're talking to.

Picture a factory's supply chain:

1. **`dpkg` — the dock worker.** He takes *one* shipping crate (a `.deb` file) that is already sitting on the dock, pries it open, and shelves its contents in the right places. He is meticulous and low-level. But he is also profoundly *narrow*: he handles exactly the one crate you hand him. He does **not** know what a "dependency" is, he will **not** drive to a warehouse to fetch a missing part, and he has **no idea** the internet exists. Hand him a crate that needs another crate first, and he'll just refuse and walk away, leaving a mess on the dock.

2. **`apt` — the logistics manager.** He never touches a crate himself. Instead, he reads your *order* ("install `nginx`"), consults the **catalogs** of every approved supplier, works out the *entire* list of crates that order implies (nginx needs this library, which needs that one...), figures out the **correct order** to open them in, drives to the **warehouses** (repositories) over the network, downloads every crate to the dock, and then **hands them to the dock worker one at a time, in sequence.** He is the brain; `dpkg` is the hands.

```
   YOU ──order──▶  apt  ──fetches .deb crates from──▶  repositories (warehouses)
                    │
                    └──hands crates one-by-one to──▶  dpkg ──installs onto──▶ your filesystem
                                                       │
                                                       └── records everything in /var/lib/dpkg (the ledger)
```

> **Why this framing first:** every command in this lesson is "talk to layer 1" or "talk to layer 2." `dpkg -l`, `dpkg --configure -a`, `dpkg -L` → you're talking to the **dock worker** about crates already on site. `apt update`, `apt install`, `apt full-upgrade` → you're talking to the **manager** about orders and warehouses. And the dreaded broken-upgrade state is simply: *the manager handed the worker a stack of crates, the worker got interrupted halfway through one of them, and now the dock is in a half-shelved mess that the manager refuses to build on top of until it's cleaned up.* Hold this two-layer picture and the rest of the lesson is just detail.

---

## 1. What a package actually *is* (open the crate)

Before any command makes sense, you need to know what's inside a `.deb` file, because the recovery commands operate on its internal parts.

A `.deb` is just an **archive** — a crate — and inside it are **two bundles plus a label**:

- **The data archive** — the actual *goods*: the program's binaries, libraries, config files, man pages. This is the stuff that ends up scattered across `/usr/bin`, `/etc`, `/usr/share`, etc. It's a little snapshot of "these files, at these paths, with these permissions."
- **The control archive** — the *paperwork*. This holds the **metadata** (package name, version, architecture, and crucially its **dependencies**: "I need `libc6 >= 2.34` and `libssl3`") and a set of **maintainer scripts** — small shell scripts the package carries that run at install/remove time to do setup work the raw files can't. (Section 3 is entirely about these scripts, because one of them is where your upgrade broke.)

The analogy: the crate contains both **the parts** (data) and a **packing manifest + assembly instructions** (control). The dock worker reads the manifest to know what this crate needs and where things go, follows the assembly instructions, then shelves the parts.

You can literally look inside one:

```
dpkg-deb --info  package.deb     # show the control paperwork (metadata + which scripts it carries)
dpkg-deb --contents package.deb  # list every file the data archive will lay down
```

> **The takeaway:** a package is not "a program." It's *files + metadata + scripts*, bundled. The metadata (dependencies) is what `apt` reads to plan; the scripts are what runs during install and where things can go wrong; the files are what `dpkg` shelves and later remembers so it can cleanly remove them.

---

## 2. `dpkg` — the dock worker and his ledger

`dpkg` ("Debian package") is layer 1: it installs, removes, and **remembers** individual packages. Its defining trait, worth repeating until it's instinct: **it operates on `.deb` files already present on your machine, one at a time, and does zero dependency resolution and zero downloading.**

```
sudo dpkg -i package.deb     # install (or upgrade) this one already-downloaded crate
sudo dpkg -r package         # remove the package, but KEEP its config files
sudo dpkg -P package         # purge: remove the package AND its config files
```

If you `dpkg -i` something whose dependencies aren't installed, `dpkg` does **not** error out gracefully and stop — it *unpacks the files anyway* and then leaves the package in a broken "unpacked but not configured" state, printing "dependency problems." That half-state is exactly the mess `apt` later has to clean up. This is *why* you almost never run `dpkg -i` directly for normal installs — you let `apt` do it, because `apt` won't hand `dpkg` a crate until all its prerequisites are on the dock.

### The ledger: `/var/lib/dpkg`

The thing that makes `dpkg` more than a fancy `tar` extractor is its **database** at `/var/lib/dpkg/` — the worker's **ledger** of everything ever installed: every package, its version, its exact list of files, and its current *state*. This ledger is sacred. It's how the system knows what's installed, what files belong to what, and what to delete on removal. (Corrupt or lose it and you have a genuinely sick system — it's worth knowing it exists and is backed up under `/var/backups/`.)

Querying the ledger is how you answer "what's on this machine?":

```
dpkg -l                  # list every known package + its state + version (the full ledger)
dpkg -l | grep nginx     # ...filtered
dpkg -L nginx            # list every FILE that nginx installed (reverse: package → its files)
dpkg -S /usr/sbin/nginx  # which package OWNS this file? (reverse: file → its package)
dpkg --audit             # report packages left in a broken/half-installed state
```

Notice the beautiful symmetry of `-L` and `-S`: the ledger lets you go **package → files** (`-L`, "what did this install?") *and* **file → package** (`-S`, "what put this file here?"). That second one is invaluable: "where did `/usr/bin/foo` come from?" → `dpkg -S /usr/bin/foo`.

### Reading the state codes in `dpkg -l` (this is how you spot trouble)

Every line of `dpkg -l` starts with a **two-letter status code** (there's a rarely-seen third column for errors). It looks cryptic but it's just **"what you want" + "what actually is":**

- **First letter — the *desired* state** (what should happen to this package): `i` = install, `h` = **hold** (freeze it, see §6), `r` = remove, `p` = purge, `u` = unknown.
- **Second letter — the *current* state** (what's actually true right now): `i` = installed and **configured** (the healthy state), `u` = **unpacked** (files laid down but setup not finished), `c` = only config files remain, `f` = **half-configured**, `h` = **half-installed**, `n` = not installed.

So you read the pair as a sentence:

| Code | Reads as | Meaning |
|------|----------|---------|
| `ii` | want-install, is-installed | **Healthy.** The normal state of everything. |
| `iU` | want-install, is-unpacked | **Broken!** Files unpacked but never configured — interrupted mid-install. |
| `iF` | want-install, is-half-configured | **Broken!** Its `postinst` script failed partway. |
| `iH` | want-install, is-half-installed | **Broken!** Unpacking itself was interrupted. |
| `rc` | want-remove, only-config-left | Package removed, but its `/etc` config files were kept. |
| `hi` | **held**, is-installed | Installed and deliberately **frozen** (won't upgrade — see §6). |

> **The diagnostic instinct:** when an upgrade breaks, your *first* move is `dpkg -l | grep -v '^ii'` (or `dpkg --audit`) to find every package whose code *isn't* the healthy `ii`. Those `iU`/`iF`/`iH` lines are the half-shelved crates. The recovery in §9 is literally "get every one of those back to `ii`." The state codes turn a vague "something's broken" into a precise list of what.

---

## 3. The two-phase install, and where your upgrade actually broke

This is the conceptual heart of the lesson, because the broken-upgrade incident lives *exactly* here. Installing a package is **not one step, it's two**, and they can fail independently:

1. **Unpack** — `dpkg` lays the data-archive files down onto the filesystem at their proper paths. After this phase the *files exist* but the package isn't "live" yet. State: **unpacked**.
2. **Configure** — `dpkg` runs the package's **`postinst`** (post-installation) maintainer script to do the finishing setup: generate config, create a system user, register and start a service, compile caches, run database migrations, etc. Only after this succeeds is the package **configured** = the healthy `ii`.

So a package's life is **unpacked → configured**, and "installed" really means "made it all the way to configured."

### Maintainer scripts: the four assembly robots

Remember the "assembly instructions" in the control archive (§1)? Those are the **maintainer scripts** — up to four little programs the package carries, each firing at a specific moment so the package can do work that plain files can't:

| Script | Fires | Job (analogy: a robot on the assembly line) |
|--------|-------|---------------------------------|
| **`preinst`** | *before* unpacking | Prep the site — stop a running old service, back something up, before files land. |
| **`postinst`** | *after* unpacking | **The big one.** Finish assembly: create users, write config, register & start the service. This is the **configure** step. |
| **`prerm`** | *before* removal | Wind down gracefully — stop the service before its files are yanked. |
| **`postrm`** | *after* removal | Sweep up — delete generated files, remove users, clean caches. |

These scripts run **as root**, and — critically — **they can fail.** When a `postinst` exits non-zero, `dpkg` leaves that package stuck in **half-configured** (`iF`): unpacked, but configure never completed. The package is in limbo.

### How `TMPDIR` broke your upgrade

Now the incident makes sense. Many maintainer scripts (and the package tools themselves) **write temporary files during configure** — into whatever `$TMPDIR` points at (or `/tmp` by default). If `$TMPDIR` is pointed at a directory that is **full, read-only, `noexec`, or simply gone**, those scripts can't write (or can't *execute*) their temp files, so they **fail**, so `postinst` exits non-zero, so `dpkg` leaves a pile of packages stuck at `iU`/`iF`, and since `apt` was midway through handing over a whole *stack* of crates, the *entire* upgrade aborts with the dock in disarray. Every later `apt`/`dpkg` command then refuses, saying "dpkg was interrupted, run `dpkg --configure -a`."

That is the *whole* story of the broken upgrade: **a bad `$TMPDIR` made the configure phase fail across many packages at once.** Which is exactly why the safe-upgrade pattern in §10 forces `TMPDIR=/tmp` (a known-good, writable, executable, on-its-own-space location) for the duration.

> **The mental model to keep:** "installed" is a *journey* — unpack, then configure — and the configure leg runs arbitrary scripts that need a sane environment. A broken upgrade is almost always a **configure-phase failure**, and the fix is almost always **"finish configuring everything that's stuck"** (`dpkg --configure -a`). Once you see installs as two-phase, the recovery commands stop being incantations and become obvious: they're just *"complete the phase that got interrupted."*

---

## 4. `apt` — the logistics manager (and `apt` vs `apt-get` vs `aptitude`)

Layer 2 is the **dependency-resolving, repository-fetching brain**. You hand it an intention; it does all the planning, downloading, ordering, and then drives `dpkg` to do the actual installs. It's what you use 99% of the time.

But there are *three* front-ends to this layer, and you should know which is which:

- **`apt-get`** (and its sibling `apt-cache`) — the **original, stable, scriptable** interface, around since the late 90s. Rock-solid, every flag guaranteed not to change, *no* fancy progress bar. **Use it in scripts and in recovery**, precisely because its behavior is frozen and predictable. The `-f install` recovery (§9) lives here.
- **`apt`** — a **newer (2014+), friendlier** front-end over the *same* engine, meant for **humans at a terminal**: a progress bar, colored output, sensible defaults (e.g. `apt search`, `apt list`), and it merges the most-used `apt-get` and `apt-cache` commands into one tool. Its output format is explicitly *not* guaranteed stable, which is why you shouldn't script against it. **Use it interactively** — it's what you'll type daily.
- **`aptitude`** — an **older, heavier** alternative front-end with a full-screen **interactive text UI** (a TUI you navigate with a menu) *and* a command line. Its claim to fame was smarter, more interactive **dependency conflict resolution**: when an upgrade hits a conflict, `aptitude` proposes a series of solutions and lets you accept/reject them interactively. It's not installed by default on modern Ubuntu and is somewhat legacy now, but it's still beloved for untangling gnarly dependency knots by hand. Think of it as "`apt` with a cockpit for manual dependency surgery."

**The practical rule:** type **`apt`** by hand; reach for **`apt-get`** when scripting or recovering; pull out **`aptitude`** only when you're in dependency hell and want an interactive co-pilot to negotiate a way out.

### Dependency chains: what the manager is actually doing

When you ask for `nginx`, `apt` reads its control metadata and sees it *Depends* on, say, `libssl3`, which *Depends* on `libc6`, and so on — a **dependency chain** (really a graph). `apt`'s core job is to **walk that graph**, collect the full closure of everything required, check what's already installed, download only what's missing, and compute a **safe order** to hand crates to `dpkg` (a library must be configured before the thing that links against it). It also respects reverse constraints — *Conflicts* ("these two can't coexist"), *Breaks*, *Replaces* — and the softer *Recommends* (installed by default) and *Suggests* (not). This graph-walking is the entire value `apt` adds over `dpkg`, and it's why you let it drive.

> **The division of labor, cemented:** `apt` *thinks* (resolve the graph, fetch, order); `dpkg` *acts* (unpack, run scripts, record in the ledger). Every successful install is the manager handing an ordered stack of crates to the worker. Every *failed* one is that handoff getting interrupted — and you fix it by talking to the worker directly (`dpkg --configure -a`) to finish the stack, *then* returning control to the manager.

---

## 5. The repositories: catalogs, codenames, and how to view/update/upgrade the sources list

`apt` is only as good as its **list of suppliers**. A **repository** is a warehouse on the internet that publishes a **catalog** (an index of available packages and versions, cryptographically signed) plus the `.deb` files themselves. Your list of approved warehouses is the **sources list**, and managing it is one of your explicit questions — so let's go deep.

### Anatomy of a repository line

The classic format lives in **`/etc/apt/sources.list`** and **`/etc/apt/sources.list.d/*.list`** (the `.d` directory is where *additional* / third-party sources go, one file each — keeping them separate from the main file so they're easy to add and remove). A line looks like:

```
deb  http://archive.ubuntu.com/ubuntu  noble  main restricted universe multiverse
│    │                                  │      └─────────── components ───────────┘
│    │                                  └── suite / CODENAME (which release)
│    └── the warehouse URL
└── "deb" = binary packages  ("deb-src" = source code packages)
```

Four parts, each answering a question:

- **`deb` / `deb-src`** — do you want compiled **binary** packages (`deb`, almost always) or **source** code (`deb-src`)?
- **The URL** — *which warehouse* (the Ubuntu archive, a PPA, a vendor like Docker or Google).
- **The suite / codename** — *which edition of the catalog.* Ubuntu releases have **codenames** (`focal` = 20.04, `jammy` = 22.04, `noble` = 24.04); Debian uses `bullseye`, `bookworm`, etc. You can also use *rolling* names like `stable`. There are also **pockets**: `noble` (the release as shipped), `noble-updates` (post-release updates), `noble-security` (security fixes), `noble-backports` (newer optional stuff). **The codename is why an upgrade can quietly become a release upgrade** — change `jammy` to `noble` and `apt full-upgrade` will haul your *entire OS* up a version.
- **The components** — *which sections of the catalog* you trust: on Ubuntu, `main` (canonical-supported, free), `restricted` (proprietary drivers), `universe` (community), `multiverse` (legally-restricted). Removing `universe` hides thousands of packages.

### The newer `deb822` format

Modern Ubuntu is moving to a more readable multi-line format in **`/etc/apt/sources.list.d/*.sources`**:

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Same four ideas, one field per concept — and note `Signed-By`, which names the **GPG key** that this repo's catalog must be signed with (§7).

### Viewing the sources list

To *see* every supplier you currently trust, you don't want to `cat` a dozen files by hand — there's a purpose-built command:

```
apt-cache policy            # ★ the master view: every repo, its pocket, and its PRIORITY
```

`apt-cache policy` (no arguments) prints your **entire effective source configuration** — every repository line `apt` is actually using, with its **pin-priority** (which source wins when a package exists in several). It's the authoritative "what are my suppliers, really?" view, accounting for all the files at once. To read the raw files instead:

```
grep -rv '^#\|^$' /etc/apt/sources.list /etc/apt/sources.list.d/   # all active source lines, comments stripped
```

### Updating vs upgrading — two words people fatally confuse

This distinction is the most important vocabulary in the whole lesson:

- **`apt update`** = **refresh the catalogs.** It contacts every repository and downloads the latest *index* of what's available — it does **not** install or change a single program. It's "go collect fresh catalogs from all the warehouses so I know what's currently on offer." After it runs, `apt` *knows* about new versions but hasn't touched them.
- **`apt upgrade`** = **actually install** the newer versions of packages you already have, based on the catalogs `update` last fetched.

So the order is always **`update` (learn what's new) → `upgrade` (apply it).** Running `upgrade` without a fresh `update` upgrades against a stale catalog. After `update`, you can preview what *would* change:

```
sudo apt update            # refresh catalogs (do this first, always)
apt list --upgradable      # which installed packages now have a newer version available
sudo apt upgrade           # install those newer versions
```

### Editing / adding / removing repositories

- **Add a PPA** (Ubuntu Personal Package Archive): `sudo add-apt-repository ppa:user/project` — this writes a file into `sources.list.d/` *and* installs its signing key for you, then you `apt update`.
- **Add a vendor repo** manually: drop a `.list`/`.sources` file in `/etc/apt/sources.list.d/` and its key in `/etc/apt/keyrings/` (§7).
- **Remove one**: delete its file from `sources.list.d/` (or `sudo add-apt-repository --remove ppa:...`), then `apt update`. Because third-party sources are *separate files*, removing a supplier is just deleting its file — clean and reversible.

> **The supplier-list mindset:** `update` deals with the **catalogs** (refresh the indexes), `upgrade` deals with the **goods** (install newer versions). The sources files are your roster of trusted warehouses; `apt-cache policy` is how you read that roster as `apt` truly sees it; and the **codename** field is the single most consequential thing in those files, because it silently decides whether "upgrade" means "patch what I have" or "move to a whole new OS release."

---

## 6. The upgrade verbs, held packages, metapackages, and autoremove

Now the daily verbs, each with its precise meaning.

### `upgrade` vs `full-upgrade` (a.k.a. `dist-upgrade`)

Both install newer versions of installed packages. The difference is **what they're allowed to do when an upgrade requires structural change**:

- **`apt upgrade`** — the **cautious** one. It will upgrade packages, but it will **never remove an installed package** to make an upgrade happen. If upgrading package A would require *removing* package B (because of a new conflict or a changed dependency), `apt upgrade` **holds A back** rather than touch B. You'll see "the following packages have been kept back."
- **`apt full-upgrade`** (old name: `apt-get dist-upgrade`) — the **thorough** one. It's allowed to **remove** packages if that's what's needed to complete the upgrade set. This is what resolves the "kept back" packages, and it's what you need for changes that restructure dependencies (and for moving across release pockets). It's more powerful and therefore needs more attention to *what it proposes to remove*.

The rule: routine patching → `apt upgrade`. Anything where packages get "kept back," or a real release/structural upgrade → `apt full-upgrade` — **after reading the removal list.**

### Held packages (the deliberate freeze)

A **hold** is you telling `apt`: *"never automatically upgrade or remove this package, even if a newer version exists."* It's a deliberate freeze — used to pin a kernel, a driver, or a database at a known-good version. Held packages show as `hi` in `dpkg -l` and are why an upgrade might *intentionally* skip something.

```
sudo apt-mark hold nginx     # freeze nginx at its current version
sudo apt-mark unhold nginx   # release the freeze
apt-mark showhold            # list everything currently held  ← "which packages are frozen?"
```

> Note the subtle overlap with "kept back": **kept back** is `apt` *temporarily* declining to upgrade something because `upgrade` (not `full-upgrade`) won't remove what's in the way — it's situational. **Held** is *you* permanently freezing it on purpose. Different causes, similar-looking result ("this didn't upgrade"). `apt-mark showhold` tells you which is which.

### Metapackages

A **metapackage** is a package that contains **no files of its own** — it's an *empty crate whose only content is dependencies.* Its entire purpose is to **pull in a curated bundle** of other packages. `ubuntu-desktop`, `build-essential`, `gnome` — installing one is shorthand for "install this whole coordinated set and keep it coherent." The analogy: ordering a "starter kit" SKU that itself ships nothing but is a single line item that causes a dozen real crates to be ordered. Removing a metapackage doesn't remove the bundle (the members stay, now just no longer "wanted"), which feeds directly into autoremove.

### `remove` vs `purge` vs `autoremove`

- **`apt remove pkg`** — uninstall the package's files, **keep its `/etc` config** (so reinstalling restores your settings). Leaves the `rc` state.
- **`apt purge pkg`** — uninstall **and** delete its config files too. A clean wipe.
- **`apt autoremove`** — remove packages that were installed *only* as **automatic dependencies** of something else and are now **orphaned** (nothing still needs them). When you remove an app (or a metapackage), the libraries it dragged in become orphans; `autoremove` is the **garbage collector** that sweeps them up. Read its list before confirming — occasionally it wants to remove something you actually care about.

> **The "wanted vs needed" model:** `apt` tracks every package as either **manually installed** ("you asked for this") or **automatically installed** ("I pulled this in as a dependency"). `autoremove` deletes automatically-installed packages that no longer have any manual package depending on them. This is why removing a metapackage then running `autoremove` cleans up the whole bundle — the members lose their last "wanted" anchor and become collectable.

---

## 7. Third-party repositories and the trust problem

Adding a repo means telling your machine to **download and run software, as root, from a stranger's warehouse.** So repositories are **cryptographically signed**, and `apt` refuses a catalog whose signature it can't verify against a key you've explicitly trusted. This is the security backbone of the whole system.

The modern, correct way to add a third-party repo has **two** parts:

1. **Install the vendor's GPG key** into a dedicated keyring file under `/etc/apt/keyrings/` (e.g. `docker.gpg`). This is the *only* key that repo is allowed to be signed with.
2. **Add the source line** with **`signed-by=`** pointing at exactly that keyring (or `Signed-By:` in deb822). This scopes the key to *only* that repo.

```
deb [signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable
```

The old way — dumping every vendor's key into one global `apt-key` trusted set — is **deprecated and dangerous**, because it let *any* trusted vendor sign *any* package (a compromised PPA could impersonate Ubuntu's own archive). The `signed-by` approach **scopes each key to its own repo**: the Docker key can only vouch for the Docker repo. Always use it.

> **The trust mindset:** every repo you add expands your "who can run code as root on this box" trust boundary. Prefer official Ubuntu sources; add third parties deliberately; always pin the key to the repo with `signed-by`; and remember that a flaky or abandoned PPA is a common cause of `apt update` errors *and* of upgrade breakage (its catalog goes stale or its packages conflict with the main archive).

---

## 8. How to know what's installed, upgradable, held — and downgraded

This pulls together your second explicit question — *how to tell the state of things*, including downgrades. The tools split by *which question* you're asking:

### "What's available / installed / upgradable?" — `apt list`

```
apt list --installed         # everything currently installed (+ versions)
apt list --upgradable        # installed packages with a newer version available  ← run after `apt update`
apt list --all-versions pkg  # every version of pkg across all your repos
```

### "Why this version? Which repo? Is it pinned/downgraded?" — `apt-cache policy pkg`

This is the **single most informative per-package command**, and it's how you detect a downgrade:

```
apt-cache policy nginx
```

It prints three things that together tell the whole story:

- **`Installed:`** — the version *currently on the machine.*
- **`Candidate:`** — the version `apt` *would install* (the best available across your repos given priorities).
- **A priority-ranked table** of every version on offer and **which repository** each comes from.

Now the **downgrade test**: compare `Installed` to `Candidate` and to the version table.

- `Installed` **==** `Candidate` → you're on the latest available. Normal.
- `Installed` **<** `Candidate` → an upgrade is available (this package would show in `apt list --upgradable`).
- `Installed` **>** `Candidate` → **you're running a version *newer* than any repo currently offers** — i.e. effectively **downgraded relative to your repos** (often the aftermath of a removed PPA, a manual `dpkg -i` of a newer build, or a forced pin). `apt` will even *want to downgrade it* to the candidate on the next `full-upgrade` if pinning allows. This mismatch is the classic fingerprint of "this package is in a weird state."

### "Who actually did what, and when?" — the audit logs

There's no single `apt list --downgraded` command, because "downgraded" is a *historical event*, not a current attribute. The real source of truth for downgrades, and for reconstructing exactly what your broken upgrade did, is the **logs**:

```
less /var/log/apt/history.log    # human-readable: every apt action, with Install/Upgrade/Downgrade/Remove lines + timestamps
less /var/log/apt/term.log       # the full terminal output of those runs (the actual dpkg chatter, errors and all)
zcat /var/log/dpkg.log*          # the low-level ledger log: every unpack/configure/remove dpkg performed
```

`history.log` is gold: it explicitly labels actions as **`Upgrade:`**, **`Downgrade:`**, `Install:`, `Remove:`, `Purge:`, each with the from→to versions and a timestamp. To find every downgrade that ever happened:

```
grep -i '^Downgrade:' /var/log/apt/history.log
```

And `grep ' installed ' /var/log/dpkg.log` (or filter by date) reconstructs the exact sequence `dpkg` executed — invaluable for pinpointing *which* package's configure step died first during the broken upgrade.

> **The "what state am I in?" toolkit:** `apt list --upgradable` (what's behind), `apt-mark showhold` (what's frozen), `dpkg --audit` / `dpkg -l | grep -v '^ii'` (what's broken), `apt-cache policy pkg` (why this version, and is it effectively downgraded?), and `/var/log/apt/history.log` (what actually happened, including every downgrade, with timestamps). Between them you can fully characterize the health of the package system before *and* after an incident.

---

## 9. The broken-upgrade recovery playbook

Now everything assembles into the recovery — and because you understand the two layers and the two install phases, every command below is obvious rather than magic. The situation: `apt full-upgrade` was interrupted (bad `$TMPDIR`, full disk, power loss, a failed `postinst`), the dock is full of half-shelved crates (`iU`/`iF`/`iH`), and every `apt`/`dpkg` command now refuses, demanding you finish the job.

**Step 1 — Diagnose: find the half-shelved crates (talk to the worker).**

```
dpkg --audit                 # report packages in a broken/half-configured state
dpkg -l | grep -v '^ii'      # everything NOT in the healthy 'ii' state
```

This gives you the exact list of what's stuck and in which sub-state. You're now looking at the precise damage, not guessing.

**Step 2 — Finish the interrupted configure phase (the single most important command).**

```
sudo dpkg --configure -a     # "configure ALL pending packages" — finish every interrupted install
```

This is the direct fix for §3's failure: it walks every unpacked-but-not-configured package and **runs its `postinst` again**, dragging each from `iU`/`iF` up to the healthy `ii`. *This is why fixing `$TMPDIR` first matters* — if `$TMPDIR` is still broken, these `postinst` scripts fail *again*. So in practice: fix the environment (point `TMPDIR` at `/tmp`), **then** `dpkg --configure -a`.

**Step 3 — Let the manager repair the dependency graph (talk to the brain).**

```
sudo apt-get -f install      # -f = "--fix-broken": resolve broken dependencies, finish the half-done set
sudo apt-get check           # verify there are no broken dependencies left (a dry diagnostic)
```

`apt-get -f install` ("fix broken") tells the *manager* to look at the now-configured-but-possibly-incomplete set, work out what's still missing or inconsistent, and **download/install/remove whatever it takes** to make the dependency graph whole again. (Note we use `apt-get`, not `apt`, for recovery — its behavior is stable and unsurprising, which is what you want when the system is already fragile.)

**Step 4 — Re-verify, then resume normally.**

```
dpkg --audit                 # should now report nothing
dpkg -l | grep -v '^ii'      # should be clean
sudo apt update && sudo apt full-upgrade   # now safe to finish the upgrade you started
```

> **The recovery, in one sentence:** *fix the environment, tell the **worker** to finish configuring everything (`dpkg --configure -a`), tell the **manager** to fix any broken dependencies (`apt-get -f install`), then verify and resume.* It maps one-to-one onto the two-layer model: a layer-1 cleanup (finish the crates on the dock) followed by a layer-2 reconciliation (make the order whole again).

---

## 10. The safe-upgrade pattern (and why each piece exists)

Given all the above, here's the disciplined upgrade ritual — the one that would have *prevented* your broken upgrade — with every part justified:

```
# 1. Refresh catalogs in a known-good environment
sudo env TMPDIR=/tmp TMP=/tmp TEMP=/tmp TEMPDIR=/tmp apt update

# 2. SIMULATE the upgrade — change nothing, just print the full plan
sudo env TMPDIR=/tmp TMP=/tmp TEMP=/tmp TEMPDIR=/tmp apt -s full-upgrade

# 3. If the plan looks sane, do it for real
sudo env TMPDIR=/tmp TMP=/tmp TEMP=/tmp TEMPDIR=/tmp apt full-upgrade
```

Why each piece:

- **`env TMPDIR=/tmp TMP=/tmp TEMP=/tmp TEMPDIR=/tmp`** — force *all four* temp-dir variables to a directory that is known to be **writable, executable (`exec`), and on real disk with free space.** This is the direct antidote to the incident: it guarantees the configure-phase scripts (§3) have a sane place to write their temp files, so `postinst` can't fail for that reason. (We set all four spellings because different scripts read different ones — belt and suspenders.) `sudo env VAR=val cmd` is the idiom for "run this command as root *with* these environment variables set," because a plain `sudo` would otherwise sanitize them away.
- **`apt -s full-upgrade`** — `-s` = **simulate** (a.k.a. `--dry-run`). It computes and prints the *entire* plan — every package to be upgraded, installed, **and especially removed** — and touches *nothing*. **This is the safety check:** you read the removal list *before* committing. If `full-upgrade` proposes to remove something alarming (a kernel, your desktop, a database), you find out here, with zero damage done, instead of mid-install.
- **`full-upgrade`** (not plain `upgrade`) — because you *want* it to resolve "kept back" packages and structural changes — but you only run it for real *after* the simulation in step 2 has shown you it's safe.

> **The habit to internalize:** **refresh in a sane environment → simulate and read the plan → apply.** The simulation step is the cheap insurance that turns "I hope this upgrade is fine" into "I've seen exactly what it will do." Combined with forcing a good `TMPDIR`, it closes both failure modes that produced your broken upgrade: a hostile temp environment, and committing to a removal set you never inspected.

---

## 11. The command cheatsheet

Grouped by **which layer you're talking to** and **what you're trying to do.**

### Layer 2 — `apt` (the manager): daily driving

| Command | What it does |
|---------|-------------|
| `sudo apt update` | Refresh repository catalogs (changes nothing installed) |
| `apt list --upgradable` | Which installed packages now have newer versions |
| `sudo apt upgrade` | Install newer versions — but never removes anything |
| `sudo apt full-upgrade` | Install newer versions, **allowed to remove** to resolve conflicts |
| `sudo apt install pkg` | Resolve deps, fetch, and install a package |
| `sudo apt remove pkg` | Uninstall, **keep** config files |
| `sudo apt purge pkg` | Uninstall **and delete** config files |
| `sudo apt autoremove` | Garbage-collect orphaned auto-installed dependencies |
| `apt -s full-upgrade` | **Simulate** — print the full plan, change nothing |
| `apt search term` / `apt show pkg` | Find / inspect packages |

### Layer 1 — `dpkg` (the worker): inspection & low-level

| Command | What it does |
|---------|-------------|
| `dpkg -l [pattern]` | List packages + state codes (`ii` healthy; `iU`/`iF` broken; `hi` held) |
| `dpkg -L pkg` | List every file a package installed (package → files) |
| `dpkg -S /path/to/file` | Which package owns this file (file → package) |
| `dpkg --audit` | Report packages in a broken/half-configured state |
| `sudo dpkg -i file.deb` | Install one already-downloaded `.deb` (no deps, no download) |
| `dpkg-deb --contents file.deb` | List files inside a `.deb` without installing |

### State, history & holds (knowing your situation)

| Command | What it tells you |
|---------|-------------|
| `apt-cache policy` | Every repo `apt` uses + priorities (your effective sources) |
| `apt-cache policy pkg` | Installed vs Candidate version + source repos (**spot downgrades** here) |
| `apt-mark showhold` | Which packages are **frozen** (held) |
| `apt list --installed` | Everything installed, with versions |
| `grep -i '^Downgrade:' /var/log/apt/history.log` | Every downgrade that has ever happened, with timestamps |
| `less /var/log/apt/history.log` | Full audited history of installs/upgrades/removes/downgrades |

### Recovery (a broken upgrade)

| Command | What it does |
|---------|-------------|
| `sudo dpkg --configure -a` | **Finish configuring** every interrupted package (the key fix) |
| `sudo apt-get -f install` | Fix broken dependencies / complete a half-done set |
| `sudo apt-get check` | Verify no broken dependencies remain |
| `dpkg -l \| grep -v '^ii'` | List everything not in the healthy state |

### Repositories & marks

| Command | What it does |
|---------|-------------|
| `sudo add-apt-repository ppa:user/project` | Add a PPA (writes a source file + installs its key) |
| `sudo add-apt-repository --remove ppa:user/project` | Remove that PPA |
| `sudo apt-mark hold pkg` / `unhold pkg` | Freeze / unfreeze a package's version |
| `sudo apt clean` / `autoclean` | Clear the downloaded `.deb` cache in `/var/cache/apt` |

---

## 12. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **`dpkg`** | Low-level installer of single `.deb` files; no deps, no network | The dock worker who shelves one crate |
| **`apt` / `apt-get`** | High-level resolver that plans, fetches, and drives `dpkg` | The logistics manager who fills the order |
| **`aptitude`** | Alternative front-end with an interactive dependency-resolver TUI | The manager's manual-override cockpit |
| **`.deb` package** | Archive of data files + control metadata + maintainer scripts | A shipping crate: parts + manifest + assembly steps |
| **dependency** | Another package this one needs to function | A part the crate can't be assembled without |
| **`/var/lib/dpkg`** | `dpkg`'s database of what's installed and which files belong to it | The worker's permanent ledger |
| **unpacked vs configured** | Files laid down (unpacked) vs setup-script finished (configured = healthy) | Crate emptied onto the floor vs fully assembled & switched on |
| **maintainer scripts** | `preinst`/`postinst`/`prerm`/`postrm` — run setup/teardown at install/remove | Assembly-line robots firing at set moments |
| **`postinst`** | The configure-phase script; its failure = a broken install | The robot that finishes assembly and powers it on |
| **state code (`ii`, `iU`, `hi`...)** | desired-state + current-state per package in `dpkg -l` | "What we want" + "what's actually true" on the crate's tag |
| **repository** | A signed online warehouse publishing a package catalog + `.deb`s | A supplier's warehouse and catalog |
| **sources list** | `/etc/apt/sources.list(.d)` — your roster of trusted repos | Your list of approved suppliers |
| **codename / suite** | Which release's catalog (`jammy`, `noble`, `bookworm`) | Which edition of the catalog |
| **component** | Section of a repo (`main`, `universe`...) | Which aisles of the warehouse you shop |
| **`update` vs `upgrade`** | Refresh catalogs vs install newer versions | Collect fresh catalogs vs actually buy the goods |
| **`upgrade` vs `full-upgrade`** | Won't remove anything vs may remove to resolve conflicts | Cautious restock vs restock allowed to clear shelf space |
| **held package** | A version you've deliberately frozen (`apt-mark hold`) | A SKU you've marked "do not reorder" |
| **kept back** | `upgrade` temporarily skipping a pkg it would have to remove something for | A reorder skipped because it'd require clearing other stock |
| **metapackage** | A package with no files, only dependencies — a curated bundle | A "starter kit" SKU that ships nothing but orders a dozen crates |
| **`autoremove`** | Removes orphaned auto-installed dependencies | Sweeping up parts nothing needs anymore |
| **`signed-by` key** | GPG key scoped to vouch for exactly one repo | A supplier's seal that's only valid on *their* crates |
| **`TMPDIR`** | Where scripts write temp files; a bad one breaks the configure phase | The workbench the assembly robots need clear and usable |

---

## 13. Where to go next

With the two-layer model and the upgrade/recovery flow in hand, the natural follow-ons:

- **APT pinning & preferences** (`/etc/apt/preferences.d/`, the priority numbers you saw in `apt-cache policy`) — how to deliberately prefer or hold back versions across repos, the controlled version of the "downgrade" situations in §8.
- **Snap and Flatpak** — the *other* packaging worlds now coexisting with `apt` on Ubuntu (self-contained, sandboxed, auto-updating), and why a program might be installed two different ways.
- **[Building your own `.deb`](02-building-your-own-deb.md)** — the very next lesson in this folder. Write the control file and the four maintainer scripts yourself, and the two-phase install stops being abstract — you'll *cause* (and fix) the `iU`/`iF` states from §2 on purpose.
- **[Disk Usage & Resource Monitoring](../03-disk-usage-and-resource-monitoring.md)** — directly relevant: a **full disk** (especially on `/` or `/var` or wherever `TMPDIR` points) is a leading cause of failed `postinst` scripts and broken upgrades. The `apt clean` / package-cache note in that lesson's reclaim table connects straight back here.
- **[systemd Services](../04-systemd-services.md)** — what a `postinst` so often does last is *register and start a service*; understanding units explains the final step of "configure."

The instinct to carry forward: **whenever package management misbehaves, ask "which layer?"** — is a *crate* half-shelved on the dock (`dpkg`, fix with `dpkg --configure -a`), or is the *order/graph* inconsistent (`apt`, fix with `apt-get -f install`)? Name the layer, and the right command names itself.
