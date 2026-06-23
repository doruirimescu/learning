# Lesson — Snap & Flatpak: the Other Packaging Worlds

> In [Lesson 01](01-package-management.md) you learned the `apt`/`dpkg` model: a package is a crate whose files get **shelved into the one shared building** — the single filesystem tree — where they share libraries with everything else. In [Lesson 02](02-building-your-own-deb.md) you built such a crate yourself. That model has quietly run Debian and Ubuntu for thirty years. But open Ubuntu's "Software" app today, install Firefox, and you are *not* getting a `.deb` — you're getting a **snap**. Install Spotify and you might reach for a **flatpak**. Run `df -h` and you'll find dozens of mysterious `/snap/...` loop mounts that `apt` never created. There are now **three packaging worlds coexisting on one Ubuntu machine**, and they work on fundamentally *different* principles. This lesson explains why the new worlds exist, how Snap and Flatpak each work, and — the practical payoff — *why the same program can be installed two or three different ways at once, and how to tell which one you're actually running.*

---

## 0. The mental model: from shared shelves to sealed containers

Hold the supply-chain picture from Lesson 01 in mind, because the new worlds are best understood as a **reaction against** it.

In the `apt`/`dpkg` model, installing software means **unpacking its parts onto communal shelves.** Your whole system is *one shared building*: there's exactly one copy of `libssl3`, one copy of `libc6`, and *every* program reaches for those same shared shelves. This is brilliantly space-efficient — a security fix to `libssl3` instantly protects every program that uses it — but it imposes a hard social contract: **everyone must agree on one version of every shared part.** A program that needs a *newer* `libssl` than the system ships is simply out of luck.

Snap and Flatpak make the opposite bet. Instead of shelving parts into the shared building, each application arrives as a **sealed shipping container**: a self-contained mini-warehouse that **brings its own copies of every library it needs**, plugs into your system as a single unit, and **never touches the communal shelves at all.** Two apps can ship two *different* versions of `libssl` inside their own containers and neither knows or cares.

```
   apt / dpkg  ── parts unpacked onto ──▶  ┌─────────── ONE SHARED BUILDING ───────────┐
                                           │  libssl3 (one copy)  libc6 (one copy)     │
                                           │  app A ──┐  app B ──┘   (all share shelves)│
                                           └────────────────────────────────────────────┘

   snap/flatpak ── each app is a ──▶   ┌─ container A ─┐  ┌─ container B ─┐
                                       │ app A         │  │ app B         │
                                       │ +its libssl   │  │ +its libssl   │   ← own copies,
                                       │ (sealed)      │  │ (sealed)      │     behind glass
                                       └──────┬────────┘  └──────┬────────┘
                                              └──── portals ─────┘  ← controlled hatches to the
                                                                       real system (files, camera…)
```

Three consequences follow from "sealed container," and they *are* the three defining traits of both Snap and Flatpak:

1. **Self-contained** — it carries its own dependencies, so it runs the same on Ubuntu, Fedora, or Arch, regardless of what's on the communal shelves.
2. **Sandboxed** — because it's sealed, it can be *confined*: it sees the system only through controlled **portals**, so a compromised app can't freely roam your files.
3. **Auto-updating** — the container "phones home" to its store and restocks itself, rather than waiting for you to run an upgrade.

> **Why this framing first:** every detail below — squashfs images, runtimes, channels, portals, interfaces — is a *mechanism serving one of those three traits.* Snap and Flatpak disagree on the *implementation* (who runs the store, how libraries are shared, how the sandbox works), but they share the same core bet: **trade the space efficiency and shared-fate of communal shelves for the independence and isolation of sealed containers.** Keep the container image in your head and the rest is detail.

---

## 1. Why a new world was needed: the three pains of shared shelves

The `apt` model didn't fail — it succeeded so completely that its *constraints* became painful as software sped up. Three specific pains drove the new formats:

**Pain 1 — Dependency hell and the "one version for everyone" rule.** Because the whole system shares one copy of each library, two programs that need *incompatible* versions of the same library cannot both be installed cleanly. App developers must target whatever ancient library versions the *oldest* supported distro ships. The shared shelf is a lowest-common-denominator.

**Pain 2 — Stable distros ship old software (on purpose).** Ubuntu's stability comes from *freezing* library versions for a release's lifetime — `noble`'s `libfoo` is the same for years (Lesson 01's codenames). Great for servers; frustrating for a desktop user who wants the *latest* Blender or VS Code, which may need libraries newer than the frozen set. Backporting every app is enormous, never-ending work for distro maintainers.

**Pain 3 — Distro fragmentation for developers.** A developer who writes one app must otherwise produce a `.deb` for Debian/Ubuntu, an `.rpm` for Fedora, a `PKGBUILD` for Arch, each with that distro's library names and versions — and test against all of them. It's a tax that scales with the number of distros. A **single self-contained bundle that runs everywhere** erases that tax.

> **The unifying complaint:** all three pains are the *same* problem wearing different hats — *the shared library shelf forces global agreement on versions.* Snap and Flatpak dissolve the problem by **abolishing the shared shelf for desktop apps**: if every app brings its own libraries, there's nothing to agree on. The cost (you'll feel it in §5) is duplication and disk — but for big GUI apps that update weekly, that trade is often worth it.

---

## 2. The three pillars (and what each costs)

Before splitting into Snap-specifics and Flatpak-specifics, nail the three shared pillars — and, crucially, the *price* of each, because the prices are exactly what people complain about.

### Pillar 1 — Self-contained (bundle your world)

Each package ships the app **plus its dependencies** — often plus a whole **base runtime** (a minimal userland: the common libraries an app expects). The app links against *its own* copies, not the system's.

- **Benefit:** runs on any distro, any age; no dependency conflicts; developer builds once.
- **Cost:** **duplication.** Ten apps may each carry their own copy of similar libraries → more disk, more RAM, more bandwidth. (Flatpak softens this with *shared runtimes* and deduplication; Snap softens it with *base snaps* — §3, §4.) And a security fix to a bundled library must be re-shipped by *each app* rather than patched once system-wide — the dark side of losing shared-fate.

### Pillar 2 — Sandboxed (operate behind glass)

Because a sealed container has a clear boundary, the system can **confine** it: restrict what files it sees, what devices it can touch, what the network looks like. The app reaches the outside world only through controlled **portals** — "may I open a file?" pops a system dialog; the app never gets blanket access to your home directory.

- **Benefit:** a malicious or compromised app is *boxed in* — it can't silently read your SSH keys or webcam.
- **Cost:** **friction and leakiness.** Sandboxes break things that "just worked" with a `.deb` — themes don't apply, an app can't see a file outside its allowed paths, two sandboxed apps can't easily talk. And confinement is only as strong as its configuration: a `classic` snap or an over-permissioned flatpak is barely sandboxed at all. The sandbox is a *spectrum*, not a binary.

### Pillar 3 — Auto-updating (the container restocks itself)

The package manager runs as a background service that periodically checks the store and **updates apps automatically**, so you're always on the latest version without `apt upgrade`.

- **Benefit:** security fixes and features arrive without you lifting a finger; great for non-technical users.
- **Cost:** **loss of control.** An app can update (and change, or break) underneath you; an update can land mid-use; pinning or deferring is possible but historically awkward (a major Snap criticism — §3). For servers and reproducible systems, "it changed itself" is a bug, not a feature — which is part of why these formats target *desktops*, not servers.

> **The recurring theme:** every pillar is a **benefit and its mirror-image cost.** Self-contained buys portability with disk; sandboxed buys safety with friction; auto-updating buys freshness with control. Whether the trade is worth it depends entirely on *what* you're installing — which is exactly why these formats *coexist* with `apt` rather than replacing it (§8).

---

## 3. Snap — Canonical's vertically-integrated world

**Snap** is Canonical's (the makers of Ubuntu) take on the sealed container. Its defining characteristic is that it's **vertically integrated**: Canonical controls the format, the tooling, *and* the one and only store. Think of it as a **single franchise chain** — one company owns the warehouses, the trucks, and the restocking schedule.

### The shape of a snap: a squashfs image, mounted

A `.snap` file is a **compressed, read-only filesystem image** (a `squashfs`). When you install one, `snapd` (the background daemon) doesn't unpack files onto your shelves — it **mounts the image** as a loopback device under `/snap/<name>/<revision>/`. The app runs *directly out of its mounted image*, which stays read-only and pristine.

This is the answer to a mystery from [Lesson 02 — Filesystems](../02-filesystem-and-mounts.md) and [Lesson 03 — Disk Usage](../03-disk-usage-and-resource-monitoring.md): **those dozens of `loop` / `squashfs` mounts cluttering your `df -h` and `lsblk` output are snaps.** Each installed snap (and often each *revision* of it) shows up as its own mounted loop device. Now you know what they are and why they're read-only.

```
df -h
Filesystem      Size  Used  ...  Mounted on
/dev/loop3       128K  128K  ...  /snap/bare/5
/dev/loop8        77M   77M  ...  /snap/core22/1033
/dev/loop12      165M  165M  ...  /snap/firefox/4173      ← Firefox, mounted from its squashfs image
```

The **base snap** (`core22`, `core20`, ...) is the minimal shared runtime several snaps build on — Snap's nod to *not* duplicating the absolute basics in every package.

### The single store and the auto-refresh

Snaps come from **one place: the Snap Store**, run by Canonical. There is no `sources.list` equivalent you point at other warehouses — the store backend is centralized and (controversially) **proprietary**: you can't self-host an alternative the way you can with apt repos or, as we'll see, Flatpak remotes.

`snapd` checks for updates several times a day and **refreshes snaps automatically** by default. You can defer or hold refreshes, but historically there was *no supported way to disable auto-update entirely* — a frequent complaint. Snap keeps the **previous revision** around so you can `snap revert` if an update breaks something (at the cost of more disk).

### Channels: choosing your risk level

A snap isn't just "a version" — you subscribe to a **channel**, which has two parts: a **track** (e.g. `latest`, or a versioned track like `18`) and a **risk level**. The four risk levels form a stability ladder:

| Risk | Meaning |
|------|---------|
| `stable` | Production-ready; what you get by default. |
| `candidate` | Almost-stable, final testing. |
| `beta` | Tested but expect rough edges. |
| `edge` | Bleeding-edge nightly builds; may be broken. |

```
snap install code --channel=latest/stable     # explicit channel
snap refresh code --channel=latest/edge        # switch an installed snap to a riskier channel
```

This is genuinely nicer than apt for "I want the bleeding-edge of *this one app* but stable everything else" — the channel is per-snap.

### Confinement: how sandboxed is it, really?

Snaps run in one of three **confinement** modes, and knowing which matters for security:

- **`strict`** — fully sandboxed: the app is boxed by AppArmor, seccomp, cgroups, and namespaces, and can only reach resources you grant. The intended default.
- **`classic`** — **no confinement.** The snap has the run of your system like a normal `.deb`. Used for tools that genuinely need it (IDEs, compilers). Installing one requires an explicit `--classic` flag and a "you sure?" — because you're opting *out* of the sandbox. (`sudo snap install code --classic`.)
- **`devmode`** — for developers: the sandbox *logs* violations instead of *enforcing* them, so you can see what permissions a snap-in-progress needs.

### Interfaces: plugs and slots

A strict snap's access to the outside is mediated by **interfaces**. A snap declares **plugs** ("I want to use the network / your camera / your home folder"); the system (or another snap) provides **slots**. A **connection** joins a plug to a slot and grants the access. Many are auto-connected for trusted snaps; sensitive ones you connect manually:

```
snap connections firefox          # what this snap is allowed to access, and what's connected
snap connect foo:camera           # manually grant the camera interface
```

This is Snap's portal/permission system — the Pillar-2 sandbox made operable.

### The Snap controversy (worth knowing, because it's why people seek Flatpak)

Snap is excellent technology wrapped in decisions that frustrate many users, and understanding the friction explains the whole ecosystem's politics:

- **The proprietary, single store** — you *must* use Canonical's backend; there's no federation. Many object on principle to a centralized, closed gatekeeper for open-source software.
- **Forced auto-updates** — no clean off switch for a long time.
- **`df`/`mount` clutter** and slower first-launch times (mounting + sandbox setup).
- **Ubuntu pushing snaps into `apt`** — on modern Ubuntu, `apt install firefox` or `chromium` actually installs the *snap* via a transitional shim (§6). Critics call this a bait-and-switch; Canonical calls it convergence. Either way, it's *the* reason you'll find a program installed "two ways."

---

## 4. Flatpak — the freedesktop, decentralized world

**Flatpak** comes from the freedesktop/GNOME community, not a single company, and its design is almost a **point-by-point answer to Snap's centralization.** If Snap is a single franchise chain, **Flatpak is an open marketplace**: many independent suppliers (remotes) agreeing on shared standards, with no one company owning the pipes.

### Remotes, not a store: decentralized by design

Flatpak has no built-in store. Instead you add **remotes** — repositories you choose to trust, exactly analogous to apt's `sources.list` (Lesson 01 §5). The dominant remote is **Flathub**, a community-run hub that hosts most apps — but it's *just a remote*, not a privileged built-in. Anyone (a company, a project, you) can host their own. This is the core philosophical difference from Snap.

```
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak remotes                 # list the warehouses you trust
```

Because Ubuntu ships Snap, **Flatpak is not installed by default** — you add it yourself (`sudo apt install flatpak`, then add Flathub). The irony is delicious: you use `apt` to install the *competing* packaging system.

### Runtimes: the shared-base middle ground

Here's Flatpak's cleverest idea, and its biggest divergence from "every container ships everything." Rather than each app bundling the *entire* world, apps declare a dependency on a shared **runtime** — a standardized base platform like `org.freedesktop.Platform`, `org.gnome.Platform`, or `org.kde.Platform`. **Many apps share one installed runtime.** An app ships only *its own* code and the few libraries beyond the runtime.

This is a deliberate compromise on Pillar 1: not the pure "bundle absolutely everything" of a naive container, nor the "share one global copy" of apt, but **"share a big standardized base, bundle the rest."** Combined with Flatpak's storage layer — **OSTree**, a content-addressed, git-like store that **deduplicates identical files** across apps and runtimes via hardlinks — the disk cost is markedly *less* brutal than the naive bundling model. (When `flatpak install` says it's also installing `org.gnome.Platform//46`, that's a runtime being fetched once for many future apps to share.)

```
flatpak list --runtime          # the shared base platforms installed
flatpak list --app              # the actual applications
```

### Portals and bubblewrap: the sandbox

Flatpak confines apps with **bubblewrap** (`bwrap`, an unprivileged sandbox) plus seccomp, and grants controlled access through **XDG Desktop Portals** (`xdg-desktop-portal`) — the same portal standard other systems are converging on. When a flatpak app opens a file, a *portal* shows the file picker and hands back only the chosen file; the app never sees your whole disk.

Permissions are **per-app and inspectable**, and you can tighten or loosen them without reinstalling:

```
flatpak info --show-permissions org.mozilla.firefox
flatpak override --user --nofilesystem=home org.mozilla.firefox   # revoke broad home access
```

There's also a popular GUI, **Flatseal**, for editing these permissions visually — Flatpak's answer to "the sandbox should be the user's to control."

### Per-user vs system, and where data lives

Flatpak can install **`--system`** (all users, needs root) or **`--user`** (just you, no root) — a flexibility apt and snap don't really offer. App *data* lives under **`~/.var/app/<app-id>/`**, neatly namespaced per app (so uninstalling and wiping is clean). Apps are named by **reverse-DNS App IDs** — `org.mozilla.firefox`, `com.spotify.Client` — which double as a globally-unique namespace (no two `firefox` collisions across vendors).

```
flatpak install flathub org.mozilla.firefox
flatpak run org.mozilla.firefox
flatpak update                  # YOU run this — Flatpak doesn't force background auto-updates like snap
flatpak uninstall --delete-data org.mozilla.firefox
```

Note the Pillar-3 difference: Flatpak leans toward **you (or your desktop's Software app) triggering updates**, rather than a daemon forcing them — more control, less hand-holding.

---

## 5. Snap vs Flatpak, head to head

Both are sealed-container systems; they differ in *philosophy and plumbing*. The contrasts that actually matter:

| Dimension | **Snap** | **Flatpak** |
|-----------|----------|-------------|
| Origin / governance | Canonical (one company) | freedesktop/community (no single owner) |
| Store model | **One** central, proprietary store | **Many** remotes; **Flathub** is the popular one |
| On Ubuntu | **Preinstalled**, default | Not installed; you add it |
| How libraries are shared | `base` snaps | **Runtimes** (`org.gnome.Platform`, …) + OSTree dedup |
| On-disk form | squashfs image **mounted** at `/snap/...` (loop devices) | OSTree repo under `/var/lib/flatpak`, `~/.local/share/flatpak` |
| Sandbox | AppArmor + seccomp + namespaces; `strict`/`classic`/`devmode` | bubblewrap + seccomp + **XDG portals** |
| Permissions | **interfaces** (plugs/slots), `snap connect` | per-app perms, `flatpak override`, Flatseal GUI |
| Updates | Daemon **auto-refreshes** (hard to disable) | Usually **user/Software-triggered** (`flatpak update`) |
| Server/CLI use | Yes — also targets servers, IoT, CLI tools | Almost exclusively **desktop GUI** apps |
| Versioning | **channels** (track + risk: stable/candidate/beta/edge) | branches per remote; less formalized risk ladder |

Rules of thumb that fall out of this:
- **Snap** reaches further down the stack (servers, CLI tools, even the boot base on Ubuntu Core) and is the path of least resistance *on Ubuntu specifically*.
- **Flatpak** is the cross-distro desktop favorite, preferred by people who want decentralization, finer permission control, and no forced auto-updates — and it's the de-facto standard on Fedora, Steam Deck, and most non-Ubuntu desktops.

---

## 6. Why a program is installed two (or three) ways — and how to tell which you're running

This is the practical heart of the lesson, and the source of real, daily confusion: **the same application can be present as a `.deb`, a snap, and a flatpak simultaneously**, each a *different version*, each updating on its own schedule, sometimes each with its **own duplicate icon** in your app menu. Here's *why* it happens and *how to untangle it.*

### Why it happens

- **Ubuntu actively routes you to snaps.** The clearest case: **Firefox.** On Ubuntu 22.04+, the `apt` package `firefox` is a **transitional shim** — a tiny `.deb` whose whole job is to install the Firefox *snap*. So `sudo apt install firefox` gives you a snap. **Chromium** on Ubuntu is snap-only (no real `.deb` at all). Ubuntu's "Software" GUI defaults to showing snaps.
- **You added Flatpak for an app Ubuntu doesn't package well**, and installed, say, Spotify or an IDE from Flathub — now it coexists with whatever apt/snap version exists.
- **Vendors offer their own `.deb`** (Google Chrome, VS Code, Docker) via their *own apt repo* (Lesson 01 §7) *while also* publishing a snap and/or flatpak. You install whichever you found first.
- **Result:** you can genuinely end up with three Firefoxes — an apt shim→snap, a real flatpak, and maybe a vendor build — each with separate profiles, extensions, and update cadence. Two icons in the launcher that look identical but are different installs is a classic "wait, why are my bookmarks gone?" moment.

### How to tell which one a command actually runs

Start with *what the shell resolves*, then follow it home:

```
which firefox                      # e.g. /usr/bin/firefox  ...but that's often a wrapper/symlink
readlink -f $(which firefox)       # follow it: /snap/bin/firefox → reveals it's the SNAP
```

`/snap/bin/...` ⇒ it's a snap. A real binary under `/usr/...` from a `.deb` ⇒ apt. Flatpak apps usually aren't on `$PATH` as a bare name at all — you launch them via `flatpak run <app-id>` or the desktop entry. Then **ask each manager directly** — the definitive cross-check:

```
dpkg -l firefox                    # is there a .deb (even a transitional shim)?     [apt world]
apt-cache policy firefox           # ...and which version/repo apt sees
snap list firefox                  # is there a snap, and which revision/channel?    [snap world]
flatpak list | grep -i firefox     # is there a flatpak, and which remote?           [flatpak world]
```

Running all three is the reliable way to enumerate *every* copy. To see versions side by side, `snap info firefox` (channels + installed revision), `flatpak info org.mozilla.firefox`, and `apt-cache policy firefox` together tell the full story.

### Disk and duplication, made visible

Tie it back to [Lesson 03 — Disk Usage](../03-disk-usage-and-resource-monitoring.md): all this coexistence has a footprint.

```
df -h | grep -E 'loop|snap'        # every snap (and revision) as a mounted loop device
du -sh /var/lib/snapd /var/lib/flatpak ~/snap ~/.var/app   # space each world consumes
snap list --all                    # includes OLD revisions snap keeps for rollback (disk hogs)
sudo snap set system refresh.retain=2    # keep fewer old revisions to reclaim space
flatpak uninstall --unused         # drop runtimes no installed app needs anymore
```

> **The untangling instinct:** when a GUI app behaves oddly — wrong version, missing extensions, duplicate icon, "my settings vanished" — your first move is **"which packaging world is this actually coming from?"** Run `readlink -f $(which app)` and then the three `dpkg -l` / `snap list` / `flatpak list` checks. Nine times out of ten the mystery is *"you launched the snap but configured the flatpak"* (or vice-versa). Naming the world is the whole diagnosis.

---

## 7. A quick word on AppImage (the fourth way)

For completeness: **AppImage** is a *third* self-contained format, even simpler. It's a **single executable file** that bundles the app and its dependencies — you download it, `chmod +x`, and run. **No installer, no store, no daemon, no system integration, and (by default) no sandbox.** It's the "portable .exe" of Linux: maximally simple and self-contained, but you manage updates yourself (some apps self-update), it doesn't auto-integrate into your menus, and it offers none of Snap/Flatpak's confinement out of the box. Good for "just let me run this one tool"; not a managed ecosystem.

So the full landscape is **four** coexisting worlds: **apt/dpkg** (shared, system-integrated), **Snap** and **Flatpak** (managed sealed containers), and **AppImage** (unmanaged single-file bundle).

---

## 8. When to reach for which

These formats *coexist* precisely because each is best at something different. A practical decision guide:

- **Use `apt`/`dpkg` for:** the operating system itself, system services, CLI tools, libraries, anything on a **server**, and anything you want **version-frozen, shared, and under your update control**. The base system should always be apt.
- **Use Snap for:** desktop apps on Ubuntu where it's the path of least resistance, things that want **channels** (track bleeding-edge of one app), and Canonical-ecosystem tooling. It's also the only first-class option for some apps *on Ubuntu* (Chromium).
- **Use Flatpak for:** **desktop GUI apps** you want the latest of, especially cross-distro, with **fine-grained sandbox control** and **no forced auto-updates** — the community favorite for proprietary apps (Spotify, Discord) and fast-moving creative tools (Blender, GIMP, OBS).
- **Use AppImage for:** one-off "run this tool once" situations where you don't want to install anything.

> **The big picture:** there's no winner — there's a **layering.** `apt` owns the system and the server; the container formats own the *desktop application* layer where freshness and isolation matter more than shared-fate efficiency. Ubuntu's choice to default GUI apps to Snap, and the community's pull toward Flatpak, is why a single machine routinely runs all of them at once. Knowing *which world a given program lives in* — and how to check — is the durable skill.

---

## 9. The command cheatsheet

### Snap

| Command | What it does |
|---------|-------------|
| `snap find <term>` | Search the Snap Store |
| `sudo snap install <name>` | Install (add `--classic` for unconfined, `--channel=` to pick risk) |
| `snap list` / `snap list --all` | Installed snaps / including old revisions kept for rollback |
| `snap info <name>` | Channels, installed revision, confinement, description |
| `sudo snap refresh [name]` | Update (all, or one); `--channel=` to switch channel |
| `sudo snap revert <name>` | Roll back to the previous revision |
| `snap connections <name>` / `snap connect <plug>` | Inspect / grant interface permissions |
| `snap changes` / `snap logs <name>` | Recent snapd actions / a snap service's logs |
| `sudo snap remove <name>` | Uninstall (add `--purge` to drop its data snapshot) |

### Flatpak

| Command | What it does |
|---------|-------------|
| `flatpak remote-add --if-not-exists flathub <url>` | Add the Flathub remote (one-time setup) |
| `flatpak remotes` | List trusted remotes (warehouses) |
| `flatpak search <term>` | Search across remotes |
| `flatpak install flathub <app-id>` | Install an app (`--user` for per-user, `--system` for all) |
| `flatpak list --app` / `--runtime` | Installed apps / shared runtimes |
| `flatpak run <app-id>` | Launch an app |
| `flatpak update` | Update everything (you trigger it) |
| `flatpak info --show-permissions <app-id>` | What the app is allowed to access |
| `flatpak override --user --nofilesystem=home <app-id>` | Tighten/loosen permissions without reinstall |
| `flatpak uninstall --unused` / `--delete-data <app-id>` | Drop orphaned runtimes / remove app + its data |

### "Which world is this app in?" (the untangling kit)

| Command | Answers |
|---------|---------|
| `readlink -f $(which app)` | `/snap/bin/...` ⇒ snap; `/usr/...` ⇒ deb |
| `dpkg -l app` / `apt-cache policy app` | Is there a `.deb` (or transitional shim)? Which version? |
| `snap list app` / `snap info app` | Is there a snap? Which channel/revision? |
| `flatpak list \| grep app` / `flatpak info <id>` | Is there a flatpak? From which remote? |
| `df -h \| grep loop` | Every installed snap, as a mounted loop device |

---

## 10. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **shared-library model** | apt/dpkg: one copy of each library, shared system-wide | One communal building with shared shelves |
| **self-contained bundle** | A package shipping its own dependencies | A sealed shipping container you plug in |
| **sandbox / confinement** | Restricting what an app can see/touch | The container operates behind glass |
| **portal** | Controlled gateway for an app to reach files/devices | A serving hatch in the glass |
| **auto-update** | A daemon refreshes apps without you asking | The landlord restocks your container |
| **Snap** | Canonical's vertically-integrated container format | A single franchise chain (one owner) |
| **`snapd`** | The daemon that mounts and manages snaps | The chain's central operations |
| **squashfs / loop mount** | A snap is a read-only image mounted at `/snap/...` | A pre-packed warehouse plugged in, not unpacked |
| **base snap** | Shared minimal runtime (`core22`) several snaps use | The chain's standard kit shared across stores |
| **channel (track/risk)** | stable/candidate/beta/edge — your stability ladder | Choosing how fresh (and risky) your stock is |
| **confinement: strict/classic/devmode** | Fully boxed / unconfined / dev-logging | Behind glass / open floor / glass-with-alarms-off |
| **interface (plug/slot)** | The permission mechanism connecting an app to a resource | Plugging a hose into a labeled tap |
| **Flatpak** | Community, decentralized container format | An open marketplace of suppliers |
| **remote** | A repository you add (like an apt source) | A warehouse you choose to trust |
| **Flathub** | The dominant (but not privileged) Flatpak remote | The biggest stall in the marketplace |
| **runtime** | Shared base platform many apps depend on | A standard chassis many products share |
| **OSTree** | Content-addressed store that deduplicates files | A warehouse that keeps one copy of identical crates |
| **bubblewrap** | The unprivileged sandbox Flatpak uses | The glass enclosure itself |
| **App ID** | Reverse-DNS app name (`org.mozilla.firefox`) | A globally-unique product barcode |
| **AppImage** | A single self-contained executable; no install/store/sandbox | A portable all-in-one appliance you just plug in |
| **transitional shim** | A tiny `.deb` whose job is to install a snap | A storefront sign that just redirects you to the chain |

---

## 11. Where to go next

You now have the full packaging landscape — the shared-shelf world of `apt` and the sealed-container worlds beside it. Natural follow-ons:

- **[01 — Package Management](01-package-management.md)** — re-read §6 (third-party repos) and §5 (sources) with fresh eyes: Flatpak *remotes* are the same idea as apt sources, and Snap's *lack* of them is the whole controversy.
- **[02 — Building Your Own `.deb`](02-building-your-own-deb.md)** — the natural sequel is **building a snap (`snapcraft`) or a flatpak (`flatpak-builder` + a manifest)**. Having hand-built a `.deb`, you'll recognize the manifest as "the control file and scripts, but declaring a *runtime* and *permissions* instead of system dependencies."
- **[Disk Usage & Resource Monitoring](../03-disk-usage-and-resource-monitoring.md)** — go reclaim space: prune old snap revisions (`snap set system refresh.retain`), drop unused flatpak runtimes (`flatpak uninstall --unused`), and finally understand every `loop` mount in your `df -h`.
- **[systemd Services](../04-systemd-services.md)** — `snapd` is a systemd service, snap apps' background daemons run as systemd units, and Flatpak leans on the user session bus and portals (themselves services). The container worlds sit *on top of* the systemd machinery you already know.

The instinct to carry forward: **"how is this software packaged?" is now a real question with four possible answers** — apt, snap, flatpak, AppImage — each with different update, sandbox, and disk behavior. When a desktop app surprises you, identify its world *first* (`readlink -f $(which app)`, then the three managers), because the world determines every rule the program plays by.
