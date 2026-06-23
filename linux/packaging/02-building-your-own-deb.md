# Tutorial — Building Your Own `.deb`

> In [Lesson 01](01-package-management.md) you learned the **theory** of a package from the *outside*: a `.deb` is a crate (control archive + data archive), installing it is a **two-phase** journey (unpack → configure), and four **maintainer scripts** (`preinst`/`postinst`/`prerm`/`postrm`) fire at set moments. All of that was something that *happened to you* during a broken upgrade. This tutorial flips it around: **you become the supplier.** You'll hand-assemble a real, working crate — files, control paperwork, and all four scripts — install it, watch the two phases happen in the terminal, then deliberately **break its `postinst` to reproduce the `iF` half-configured state from your incident** and recover it. By the end, none of those words are abstract: you'll have *caused* every one of them on purpose.

---

## 0. The mental model: we are building the crate, not the program

Keep the supply-chain picture from Lesson 01 in mind, but step *behind* the warehouse. Until now you were the customer receiving crates. Now you're the **packer on the supplier's side**, and your job is the inverse of what `dpkg` does:

- **`dpkg` (install)** takes a crate and *explodes* it onto the filesystem: control paperwork to its database, data files to their paths, scripts run at the right moments.
- **You (build)** do the reverse: you *gather* the files into a folder laid out exactly as they should land, you *write* the paperwork (the control file) and the *assembly instructions* (the maintainer scripts), and you *seal it all into one crate.*

And here is the single most clarifying fact about building a `.deb` by hand:

> **A `.deb` is just a directory, compressed.** You build a folder whose internal layout **mirrors the target filesystem** — a file you want at `/usr/bin/foo` goes at `BUILDDIR/usr/bin/foo` — plus one special `DEBIAN/` subfolder holding the control file and scripts. Then a single command zips that folder into a `.deb`. That's the whole trick. Building a package is **laying out a tiny mock-up of how the target machine should look afterward**, then sealing it.

The analogy: you're packing a flat-pack furniture box. The **box's interior is arranged like the room it's destined for** (the data tree mirroring `/`), and you slip in a **manifest + assembly-robot instructions** (the `DEBIAN/` folder). Seal it, ship it, and on the far end the dock worker unpacks the room and runs the robots.

---

## 1. Two ways to build — and why we go the hard way first

There are two worlds for producing a `.deb`, and conflating them is the #1 beginner confusion:

- **The `dpkg-deb` way (binary package, by hand).** You build the directory tree yourself and run `dpkg-deb --build`. It's low-level, exposes *every* part directly, and is **exactly what we want for learning** — because the control file and the four scripts are sitting right there as plain files you wrote. This is the `dpkg` of building: manual, transparent, no magic.
- **The `debhelper` / `dpkg-buildpackage` way (source package, the "real" toolchain).** You write a `debian/` directory (`control`, `rules`, `changelog`, `copyright`...) and a build system (`debhelper`, invoked via `dh`) **generates** the maintainer scripts, compiles the software, and assembles the `.deb` for you. This is how every package in the Ubuntu archive is actually built — but it *hides* the two-phase install and the scripts behind automation, which is the opposite of what we want today.

The relationship mirrors Lesson 01 perfectly: **`dpkg-deb` is to `dpkg` as `debhelper` is to `apt`** — one is the transparent low-level tool, the other is the automated orchestrator built on top. We'll build by hand with `dpkg-deb` so the concepts are naked, then point at the real toolchain in §11.

```
sudo apt install dpkg-dev lintian   # dpkg-deb (build), dpkg-dev (helpers), lintian (the package linter)
```

---

## 2. The package we'll build: `ticktock`

A package that just drops a file in `/usr/bin` would never exercise the maintainer scripts — it'd unpack and be done, no configure work worth the name. To make all four scripts *necessary*, we build a **tiny system service**, because a service forces real configure-time work: create a user to run as, make a directory it can write to, register the unit, start it — and undo all of that cleanly on removal. This is the canonical real-world package shape, and it ties straight back to [systemd Services](../04-systemd-services.md).

**`ticktock`** will:

- install a small daemon script at **`/usr/bin/ticktock`** that appends the time to a log every few seconds;
- install a config file at **`/etc/ticktock/ticktock.conf`** (the tick interval) — declared a **conffile** so user edits survive upgrades;
- install a systemd unit at **`/lib/systemd/system/ticktock.service`**;
- on install (**`postinst`**): create a dedicated system user `ticktock`, create its writable log dir, reload systemd, enable & start the service — *this is the configure phase made concrete*;
- on upgrade (**`preinst`**/**`prerm`**): stop the old version cleanly before files are swapped;
- on removal (**`prerm`**): stop & disable the service before its files vanish;
- on purge (**`postrm`**): delete the log dir and the system user — full cleanup.

Here's the build tree we're going to create. **Stare at this** — the left side is "where it lives in the box," the right side is "where it lands on the machine":

```
ticktock_1.0.0-1_all/                         ← the build directory (name = pkg_version_arch)
├── DEBIAN/                                    ← the CONTROL archive (never installed; metadata + scripts)
│   ├── control                               ← the paperwork (name, version, deps, description)
│   ├── conffiles                             ← list of files dpkg must treat as user-editable config
│   ├── preinst                               ← script: before unpack
│   ├── postinst                              ← script: after unpack  (the configure phase)
│   ├── prerm                                 ← script: before removal
│   └── postrm                                ← script: after removal
├── etc/
│   └── ticktock/
│       └── ticktock.conf            → installs to /etc/ticktock/ticktock.conf
├── usr/
│   └── bin/
│       └── ticktock                 → installs to /usr/bin/ticktock
├── lib/
│   └── systemd/
│       └── system/
│           └── ticktock.service     → installs to /lib/systemd/system/ticktock.service
└── usr/
    └── share/
        └── doc/
            └── ticktock/
                └── copyright        → installs to /usr/share/doc/ticktock/copyright
```

> **The two halves, made literal:** everything under `DEBIAN/` is the **control archive** — it tells `dpkg` *about* the package and *runs at* install/remove, but **none of it is ever copied to the filesystem.** Everything *outside* `DEBIAN/` is the **data archive** — a mirror of the target tree, copied verbatim onto the machine. The capital-letters `DEBIAN/` (build-time metadata folder) vs lowercase `etc`, `usr`, `lib` (the actual installed paths) is the entire mental split, visible right in the folder names.

Let's create the skeleton:

```
mkdir -p ticktock_1.0.0-1_all/DEBIAN
mkdir -p ticktock_1.0.0-1_all/etc/ticktock
mkdir -p ticktock_1.0.0-1_all/usr/bin
mkdir -p ticktock_1.0.0-1_all/lib/systemd/system
mkdir -p ticktock_1.0.0-1_all/usr/share/doc/ticktock
cd ticktock_1.0.0-1_all
```

---

## 3. The data side: lay down the files as they'll land

We fill in the **data archive** first — the actual goods — because they're the easy, concrete part. Remember: wherever you put a file *inside the build tree* is exactly where it appears *on the target*. We're arranging the room.

**The daemon** — `usr/bin/ticktock`. A trivial loop that reads the interval from the config and appends the time to its log:

```sh
#!/bin/sh
# /usr/bin/ticktock — the world's most pointless daemon
. /etc/ticktock/ticktock.conf      # sources INTERVAL=...
while true; do
    echo "tick $(date --iso-8601=seconds)" >> /var/log/ticktock/tick.log
    sleep "${INTERVAL:-5}"
done
```

**The config** — `etc/ticktock/ticktock.conf`. One knob, so we can later prove that edits to it survive an upgrade:

```sh
# /etc/ticktock/ticktock.conf — edit me and I'll survive upgrades
INTERVAL=5
```

**The systemd unit** — `lib/systemd/system/ticktock.service`. Runs the daemon as the `ticktock` user the `postinst` will create:

```ini
[Unit]
Description=Ticktock — appends the time to a log forever
After=network.target

[Service]
ExecStart=/usr/bin/ticktock
User=ticktock
Group=ticktock
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**The copyright file** — `usr/share/doc/ticktock/copyright`. Debian policy expects every package to ship one; `lintian` will nag if it's missing. A minimal stub is fine for learning:

```
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
Upstream-Name: ticktock

Files: *
Copyright: 2026 Your Name <you@example.com>
License: MIT
```

The daemon must be executable; the config and unit must not be:

```
chmod 0755 usr/bin/ticktock
chmod 0644 etc/ticktock/ticktock.conf lib/systemd/system/ticktock.service usr/share/doc/ticktock/copyright
```

> **The mirroring principle, internalized:** there is no "install path" setting anywhere. The path *is* the location in the build tree. This is why building a package feels like set-dressing a stage: you place every prop where the audience (the target system) will find it, then the curtain (`dpkg-deb`) drops.

---

## 4. The `control` file — the paperwork, field by field

Now the **control archive**. The heart of it is `DEBIAN/control` — the metadata `apt` reads to plan, and `dpkg` records in its ledger. Create `DEBIAN/control`:

```
Package: ticktock
Version: 1.0.0-1
Section: utils
Priority: optional
Architecture: all
Depends: adduser, systemd
Maintainer: Your Name <you@example.com>
Description: Appends the time to a log, forever
 A deliberately pointless demonstration daemon used to learn Debian
 packaging. It ticks once per configured interval and writes the
 timestamp to /var/log/ticktock/tick.log.
```

Every field earns its place:

- **`Package`** — the name `apt`/`dpkg` know it by. Lowercase, no spaces.
- **`Version`** — `upstream-debianrevision`. `1.0.0` is *your software's* version; `-1` is the *packaging* revision (bump it when you change only the packaging, not the software). This is the exact string `apt-cache policy` compares to decide upgrade vs downgrade (Lesson 01 §8).
- **`Section`** / **`Priority`** — catalog aisle (`utils`, `net`, `admin`...) and how essential it is (`optional` is the normal choice). Cosmetic-ish, but policy wants them.
- **`Architecture`** — `all` means **architecture-independent** (true here — it's a shell script that runs anywhere). Compiled C would be `amd64`, `arm64`, etc. This is the `_all` (or `_amd64`) you see in `.deb` filenames.
- **`Depends`** — **the dependency chain you now create yourself.** We need `adduser` (our `postinst` calls it) and `systemd` (we ship a unit). Listing them here is the promise `apt` relies on to fetch prerequisites *before* handing our crate to `dpkg`. Get this wrong and your `postinst` will explode on a machine lacking the tool — a real, common packaging bug.
- **`Maintainer`** — who to blame. Format matters; `lintian` checks it.
- **`Description`** — first line is the short summary; subsequent lines, **each indented by one space**, are the extended description. That leading space is mandatory syntax, not decoration.

> **This file is the bridge between the two layers of Lesson 01.** `apt` reads `Depends` to resolve the graph; `dpkg` stores the whole thing in `/var/lib/dpkg/status`. When you later run `dpkg -l ticktock` and see `Version 1.0.0-1`, or `apt-cache policy ticktock`, you're reading *this file*, post-install.

---

## 5. `conffiles` — teaching dpkg which files are "yours to edit"

Create `DEBIAN/conffiles` with one line:

```
/etc/ticktock/ticktock.conf
```

This tiny file resolves a question Lesson 01 raised but didn't fully explain: why does `apt remove` *keep* `/etc` config, why does `purge` delete it, and what happens to a config file you've **edited** when the package upgrades and ships a *new* default?

By listing a path in `conffiles`, you promise dpkg "this is **user-owned configuration**, not a plain data file," and dpkg then applies its **conffile logic**:

- On **upgrade**, if you *haven't* touched the file, dpkg silently installs the new default. If you *have* edited it **and** the package ships a different new default, dpkg **stops and asks you** ("keep your version / take the maintainer's / show a diff") — the famous `Configuration file '...' ==> ...?` prompt. Your edits are never silently clobbered.
- On **`remove`**, conffiles are **kept** (that `rc` state from Lesson 01 §2 — "removed, config remains").
- On **`purge`**, conffiles are **deleted**.

> **This is the mechanism behind `remove` vs `purge`.** It isn't magic in `apt` — it's the `conffiles` list you, the packager, declared. We'll *watch* this exact behavior in §10–11: edit the config, upgrade, and see it preserved; then purge, and see it finally swept away.

---

## 6. The four maintainer scripts — the abstract made executable

This is the centerpiece. These four scripts are the "assembly-robot instructions," and writing them yourself is what makes the two-phase install stop being abstract. Three things you must know first, because they're where hand-written scripts go wrong:

1. **They run as root**, sourced into the install at precise moments.
2. **They must be executable** (`chmod 0755`) or `dpkg-deb` will refuse to build.
3. **dpkg calls each one with arguments** telling it *which* situation it's in — `configure`, `remove`, `purge`, `upgrade`, etc. A correct script **branches on `$1`**, because "install" and "upgrade" and "purge" need different behavior. Ignoring `$1` is the single most common maintainer-script bug. Each starts with `set -e` so any failed command aborts the script (and thus the phase) — which is *exactly* how a bad `postinst` produces the broken state we'll reproduce in §9.

Here's where each fires in the two-phase lifecycle:

```
INSTALL / UPGRADE                          REMOVE / PURGE
─────────────────                          ──────────────
preinst   ($1 = install | upgrade)         prerm   ($1 = remove | upgrade)
   │                                          │
   ▼  ── dpkg unpacks the data files ──       ▼  ── dpkg deletes the data files ──
   │      (state: UNPACKED, "iU")             │
   ▼                                          ▼
postinst  ($1 = configure)                 postrm  ($1 = remove | purge | upgrade)
   │                                          │
   ▼  package is now CONFIGURED ("ii")        ▼  package gone (purge also drops conffiles)
```

### `DEBIAN/preinst` — *before* the files land

```sh
#!/bin/sh
set -e
# Fires BEFORE unpacking. On upgrade, the OLD service is still running and its
# files are about to be overwritten — stop it first so we don't swap binaries
# under a live process.
case "$1" in
    upgrade)
        if systemctl is-active --quiet ticktock.service; then
            systemctl stop ticktock.service
        fi
        ;;
    install)
        : # fresh install: nothing running yet, nothing to do
        ;;
esac
exit 0
```

Note the branch: on a **fresh `install`** there's nothing to stop; only on **`upgrade`** is there a running old version to quiesce. This is why `$1` matters.

### `DEBIAN/postinst` — *after* the files land (THE configure phase)

This is the big one — the script that *is* the "configure" step, the one whose failure left your real system at `iF`. It does all the setup the raw files can't:

```sh
#!/bin/sh
set -e
case "$1" in
    configure)
        # 1. Create the dedicated system user/group the service runs as.
        #    Guard with getent so re-running (idempotency!) is safe.
        if ! getent passwd ticktock >/dev/null; then
            adduser --system --group --no-create-home \
                    --home /nonexistent ticktock
        fi

        # 2. Create the writable log directory, owned by that user.
        mkdir -p /var/log/ticktock
        chown ticktock:ticktock /var/log/ticktock

        # 3. Register the new unit with systemd, then enable + start it.
        systemctl daemon-reload
        systemctl enable --now ticktock.service
        ;;
esac
exit 0
```

Three ideas worth pausing on:

- **`configure` is the only case we handle**, because that's the argument dpkg passes after a successful unpack. (On a *re*-configure, dpkg passes `configure <old-version>`; our code is written to not care, which is good.)
- **Idempotency** — the `getent` guard and `mkdir -p` mean running this script *twice* causes no harm. This matters enormously: when you run `dpkg --configure -a` to recover a broken install (§9), `postinst` runs *again*, and a non-idempotent script would fail the second time. **Good maintainer scripts are safe to re-run** — that's *why* `dpkg --configure -a` is a safe recovery move.
- This is where the **`TMPDIR` failure from your incident would bite**: if any command here needed a writable temp dir and couldn't get one, `set -e` would abort, dpkg would leave `ticktock` at `iF`, and the install would be "broken." We'll trigger that deliberately in §9.

### `DEBIAN/prerm` — *before* the files are removed

```sh
#!/bin/sh
set -e
# Fires BEFORE files are deleted. Stop the service while its binary still exists.
case "$1" in
    remove|deconfigure)
        if systemctl is-active --quiet ticktock.service; then
            systemctl stop ticktock.service
        fi
        systemctl disable ticktock.service >/dev/null 2>&1 || true
        ;;
    upgrade)
        : # the new package's scripts will handle restart; nothing to do here
        ;;
esac
exit 0
```

### `DEBIAN/postrm` — *after* the files are gone (and cleanup on purge)

```sh
#!/bin/sh
set -e
case "$1" in
    purge)
        # Full cleanup: only on PURGE do we erase state the user might have wanted.
        rm -rf /var/log/ticktock
        if getent passwd ticktock >/dev/null; then
            deluser --system ticktock >/dev/null 2>&1 || true
        fi
        systemctl daemon-reload >/dev/null 2>&1 || true
        ;;
    remove)
        # Plain remove: the unit file is gone, so just refresh systemd's view.
        systemctl daemon-reload >/dev/null 2>&1 || true
        ;;
esac
exit 0
```

The **`remove` vs `purge` split here is the whole point**: a plain `remove` leaves the log dir and the system user in place (in case you reinstall); only `purge` — the deliberate "wipe everything" — tears them down. This is the same distinction you saw from the *outside* in Lesson 01, now written by your own hand.

Make all four executable (non-negotiable):

```
chmod 0755 DEBIAN/preinst DEBIAN/postinst DEBIAN/prerm DEBIAN/postrm
```

> **The payoff:** you have now *authored* the exact four scripts that, until now, were abstract names in a table. `postinst` isn't "the configure script" anymore — it's *those eleven lines that make a user and start a service*. And you can already see how it breaks: any line failing under `set -e` strands the package half-configured.

---

## 7. Seal the crate: `dpkg-deb --build`

Everything's in place. From *inside* the build dir, step up one level and build:

```
cd ..
dpkg-deb --build --root-owner-group ticktock_1.0.0-1_all
```

- **`--build <dir>`** zips the directory tree into `ticktock_1.0.0-1_all.deb` (it names the output after the folder).
- **`--root-owner-group`** forces every file in the package to be owned by `root:root`. Without it, the files would be owned by *you* (uid 1000), which is wrong for a system package — historically you needed `fakeroot` to fake root ownership during build; this flag is the modern shortcut.

Now **inspect what you built**, using the very same `dpkg-deb` query commands from Lesson 01 — but this time pointed at your own crate:

```
dpkg-deb --info     ticktock_1.0.0-1_all.deb   # the control paperwork + which scripts it carries
dpkg-deb --contents ticktock_1.0.0-1_all.deb   # every file in the data archive, with paths/perms
```

`--info` will show your `control` fields *and* list the maintainer scripts it found. `--contents` will show your data tree — confirm `/usr/bin/ticktock` is `rwxr-xr-x` and the others aren't executable.

Finally, **lint it** — `lintian` is the packaging equivalent of a compiler's `-Wall`, checking your crate against Debian policy:

```
lintian ticktock_1.0.0-1_all.deb
```

Expect a few minor warnings (we cut corners — no `changelog.Debian.gz`, a stub copyright). That's fine for learning; in a real package you'd resolve them. Reading lintian's tags is itself a fast way to *learn* policy.

> **Filename anatomy, now obvious:** `ticktock_1.0.0-1_all.deb` = `Package_Version_Architecture.deb`. Every `.deb` you've ever downloaded follows this, and now you know it's literally `control`'s three fields, joined by underscores.

---

## 8. Install it — and watch the two phases happen in real time

The moment of truth. Install your crate. Prefer `apt` so dependency resolution runs (note the **`./`** — it tells `apt` "a local file," not a repo package):

```
sudo apt install ./ticktock_1.0.0-1_all.deb
```

**Read the output carefully** — you will literally see the two phases Lesson 01 described, narrated:

```
Unpacking ticktock (1.0.0-1) ...      ← PHASE 1: data files laid down  (briefly the "iU" state)
Setting up ticktock (1.0.0-1) ...     ← PHASE 2: your postinst runs    (the "configure" step → "ii")
```

"**Setting up**" *is* your `postinst` executing. Now verify every layer of what just happened:

```
systemctl status ticktock.service     # active (running) — your postinst started it
cat /var/log/ticktock/tick.log        # ticks accumulating — the daemon is alive
getent passwd ticktock                 # the system user your postinst created exists
dpkg -l ticktock                       # state code "ii" — healthy, fully configured
dpkg -L ticktock                       # every file YOUR package installed (package → files)
dpkg -S /usr/bin/ticktock              # "ticktock: /usr/bin/ticktock" (file → package)
```

That `ii` from `dpkg -l` is the healthy end-state from Lesson 01 §2 — you can now point at exactly what produced it: a clean unpack followed by a `postinst` that exited 0.

> **You've closed the loop.** Every query command you ran *against the system* in Lesson 01 now returns data *about a package you authored*. The ledger entry, the file list, the `ii` state — all of it traces back to the folder you built in §2–6.

---

## 9. Break it on purpose: reproducing your `iF` incident in miniature

This is the most valuable five minutes in the tutorial. We'll make `postinst` fail and watch dpkg leave the package **half-configured (`iF`)** — the exact state your broken upgrade was stuck in — then recover it the exact way you recovered the real thing.

First, **purge the good version** so we start clean:

```
sudo apt purge -y ticktock
```

Now sabotage the `postinst`. Edit `ticktock_1.0.0-1_all/DEBIAN/postinst` and insert a guaranteed failure right in the middle of the configure work — simulating, say, a script that needs a temp dir that isn't writable (your `TMPDIR` incident):

```sh
    configure)
        if ! getent passwd ticktock >/dev/null; then
            adduser --system --group --no-create-home --home /nonexistent ticktock
        fi

        echo "writing temp state..." > "$TMPDIR/ticktock.state"   # ← FAILS if TMPDIR is unset/unwritable
        false                                                     # ← or just force a non-zero exit

        mkdir -p /var/log/ticktock
        ...
```

(The bare `false` is the simplest guaranteed failure; the `$TMPDIR` line is the realistic one. Either works.) Rebuild and install:

```
dpkg-deb --build --root-owner-group ticktock_1.0.0-1_all
sudo apt install ./ticktock_1.0.0-1_all.deb
```

Watch it **unpack fine, then fail at "Setting up"**:

```
Unpacking ticktock (1.0.0-1) ...
Setting up ticktock (1.0.0-1) ...
dpkg: error processing package ticktock (--configure):
 installed ticktock package post-installation script subprocess returned error exit status 1
```

There it is — your incident, reproduced. The files are on disk (unpack succeeded) but configure died. Confirm the broken state with the diagnostics from Lesson 01 §2:

```
dpkg -l ticktock          #  "iF" — desired-install, half-configured.  NOT healthy.
dpkg --audit              #  reports ticktock as broken
```

Now **recover it exactly as you recovered the real machine.** First fix the cause (un-sabotage the script — remove the failing lines, or in the real incident, fix `TMPDIR`), rebuild... but here's the deeper lesson: you don't even need to reinstall. Because the files are already unpacked, you just need to **finish the interrupted configure phase** — and that's precisely what one command does:

```
# (after fixing the postinst and rebuilding so the on-disk script is correct)
sudo dpkg-deb --build --root-owner-group ticktock_1.0.0-1_all && sudo dpkg -i ticktock_1.0.0-1_all.deb
sudo dpkg --configure -a      # ← re-runs the (now-fixed, idempotent) postinst → drags iF up to ii
```

`dpkg --configure -a` re-runs `postinst` for every package stuck unconfigured — and because you wrote `postinst` to be **idempotent** (§6), re-running it is safe. The package climbs from `iF` to `ii`. If the failure had also left dependencies tangled, `sudo apt-get -f install` would finish the job — the layer-2 half of the recovery.

> **The whole of Lesson 01 §9, now felt in your hands:** a broken upgrade is a *configure-phase failure leaving packages unconfigured*, and the fix is *"complete the phase that got interrupted"* via `dpkg --configure -a`. You just caused it and cured it with a package you wrote. The recovery is no longer a memorized incantation — you understand *mechanically* why it works (the files are already down; only `postinst` needs to succeed) and why it's safe (you made the script idempotent).

---

## 10. Upgrade it — watch `preinst`/`prerm` fire, and conffiles survive

Let's exercise the *other* lifecycle path and prove the conffile promise from §5. With the good `ticktock` installed, **edit its config to a value you'll recognize**:

```
sudoedit /etc/ticktock/ticktock.conf      # change INTERVAL=5  →  INTERVAL=2
```

Now ship a **version `1.0.0-2`**. Bump the `Version` in `DEBIAN/control`, and — to prove preservation honestly — *also* change the default config the package ships, so dpkg has a genuine conflict to handle:

```
# in DEBIAN/control:  Version: 1.0.0-2
# in etc/ticktock/ticktock.conf:  change the default to INTERVAL=10
```

Rebuild into a correctly-named new folder/file and install over the top:

```
# (rename the build dir to ticktock_1.0.0-2_all, or just rebuild and name the .deb accordingly)
dpkg-deb --build --root-owner-group ticktock_1.0.0-2_all
sudo apt install ./ticktock_1.0.0-2_all.deb
```

Watch the **upgrade-path scripts** fire (different from the install path): the *old* package's `prerm upgrade` and your `preinst upgrade` stop the running service, dpkg swaps the files, then `postinst configure` restarts it. And critically — because `ticktock.conf` is a **conffile you edited** and the new package ships a *different* default — dpkg **stops and asks**:

```
Configuration file '/etc/ticktock/ticktock.conf'
 ==> Modified (by you or by a script) since installation.
 ==> Package distributor has shipped an updated version.
   What would you like to do about it? ...
```

Your `INTERVAL=2` edit is safe unless *you* choose to overwrite it. **That prompt is the conffiles mechanism from §5, live** — and it only appeared because you both declared the file in `conffiles` *and* edited it. Pick "keep your version" (`N`) and confirm with `grep INTERVAL /etc/ticktock/ticktock.conf` that your `2` survived.

---

## 11. Remove vs purge — watch your own cleanup logic run

Close the lifecycle. First a plain **remove**, then a **purge**, and watch your `prerm`/`postrm` branches do exactly what you wrote:

```
sudo apt remove ticktock
systemctl status ticktock.service     # gone — your prerm stopped & disabled it
ls /var/log/ticktock                  # STILL THERE — remove keeps it (your postrm only cleans on purge)
getent passwd ticktock                # user STILL EXISTS — likewise
dpkg -l ticktock                      # state "rc" — removed, config-files remain (Lesson 01 §2)
```

The `rc` state and the surviving log dir/user are *your design*: `postrm remove` deliberately left them. Now **purge**:

```
sudo apt purge ticktock
ls /var/log/ticktock                  # GONE — your postrm 'purge' branch rm -rf'd it
getent passwd ticktock                # GONE — your postrm deleted the user
dpkg -l ticktock                      # no longer listed at all
```

Every difference between `remove` and `purge` that you observed *from the outside* in Lesson 01 is now traceable to the `case "$1"` branches *you wrote* in `postrm`. The abstraction is fully dissolved.

---

## 12. How real packages are built (the toolchain we skipped)

You built a **binary package by hand** to see every part naked. Real-world packaging — everything in the Ubuntu archive — uses the **source-package** toolchain, which automates exactly what you did manually. A quick map so you recognize it:

- **A `debian/` directory** (note: lowercase, lives in the *source* tree — different from the build-time `DEBIAN/` you made) holds the recipe: **`control`** (same fields, plus source-level ones), **`changelog`** (every version + what changed — `dpkg-buildpackage` reads the version from *here*, not from a folder name), **`rules`** (a `Makefile` that drives the build), **`copyright`**, and optional **`ticktock.install`**, **`postinst`**, etc.
- **`debhelper`**, invoked through the tiny **`dh`** command in `rules`, runs dozens of `dh_*` helpers that do the tedious correct things automatically: **`dh_installsystemd`** *generates* the service-handling `postinst`/`prerm` code you hand-wrote (using `deb-systemd-helper` for policy-correct enable/disable), **`dh_systemd_start`**, **`dh_installdocs`**, **`dh_strip`**, and so on. The `#DEBHELPER#` token in a maintainer script is where that generated code gets spliced in.
- **`dpkg-buildpackage -b`** (often via **`debuild`**, which also runs `lintian`) orchestrates the whole thing into a `.deb` — the `apt`-level orchestrator to `dpkg-deb`'s `dpkg`-level manual build.
- **`dh_make`** scaffolds a starter `debian/` directory from an existing source tarball, so you rarely start from a blank page.

> **Why we did it the hard way:** every one of those tools exists to *generate the files you just wrote by hand.* `dh_installsystemd` writes a `postinst` that creates users and starts services — but now you know *what it's writing and why*, so when it misbehaves (or you read a real package's generated scripts) it's legible instead of magic. Learn the manual crate first; reach for `debhelper` once the concepts are bone-deep.

---

## 13. The command cheatsheet

Grouped by phase of the build-and-test loop.

### Build & inspect

| Command | What it does |
|---------|-------------|
| `dpkg-deb --build --root-owner-group DIR` | Seal a build tree into `DIR.deb`, files owned by root |
| `dpkg-deb --info pkg.deb` | Show the control paperwork + which maintainer scripts it carries |
| `dpkg-deb --contents pkg.deb` | List every file in the data archive (paths + perms) |
| `dpkg-deb --raw-extract pkg.deb out/` | Unpack a `.deb` back into a build tree (reverse-engineer one) |
| `lintian pkg.deb` | Lint the package against Debian policy (the packaging `-Wall`) |

### Install, verify, recover

| Command | What it does |
|---------|-------------|
| `sudo apt install ./pkg.deb` | Install a local `.deb` *with* dependency resolution (note the `./`) |
| `sudo dpkg -i pkg.deb` | Install it raw (no dep resolution — for testing layer 1) |
| `dpkg -l NAME` | State code: `ii` healthy, `iU`/`iF` broken, `rc` removed-config-kept |
| `dpkg -L NAME` / `dpkg -S /path` | Files this package installed / which package owns a file |
| `dpkg --audit` | List packages stuck in a broken/half-configured state |
| `sudo dpkg --configure -a` | Re-run pending `postinst`s — finish interrupted configures (`iF`→`ii`) |
| `sudo apt-get -f install` | Fix broken dependencies after a failed install |

### Remove

| Command | What it does |
|---------|-------------|
| `sudo apt remove NAME` | Uninstall, **keep** conffiles (→ `rc` state) |
| `sudo apt purge NAME` | Uninstall **and** delete conffiles + run `postrm purge` |

---

## 14. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **build tree** | A directory whose layout mirrors the target filesystem, plus `DEBIAN/` | A flat-pack box arranged like the destination room |
| **`DEBIAN/` (capitalized)** | Build-time control folder: metadata + scripts; never installed | The manifest + robot instructions slipped into the box |
| **data archive** | Everything outside `DEBIAN/` — the files that land on disk | The furniture parts themselves |
| **control archive** | `control` + scripts + `conffiles` — the paperwork dpkg reads/runs | The shipping manifest and assembly steps |
| **`control` file** | Name, version, architecture, **Depends**, description | The packing manifest |
| **`Architecture: all`** | Arch-independent package (scripts, data) vs `amd64`/`arm64` | A part that fits any model |
| **`Depends`** | Packages that must be present first — the chain you declare | "Requires these other parts to assemble" |
| **maintainer scripts** | `preinst`/`postinst`/`prerm`/`postrm` — run at lifecycle moments | The assembly/teardown robots |
| **`postinst configure`** | The script that *is* the configure phase (create users, start service) | The robot that finishes assembly and powers on |
| **`$1` argument** | Tells a script its situation: `configure`/`remove`/`purge`/`upgrade` | The work-order telling the robot which job this is |
| **idempotent** | Safe to run twice — why `dpkg --configure -a` can re-run `postinst` | A robot that checks before acting, so re-running is harmless |
| **conffile** | A path in `DEBIAN/conffiles` dpkg treats as user-editable config | A part the customer is allowed to customize and keep |
| **two-phase install** | Unpack (`iU`) then configure (`ii`); they fail independently | Empty the box, *then* assemble & switch on |
| **`iF` / `iU`** | Half-configured / unpacked — the broken states between phases | A half-assembled kit on the floor |
| **`dpkg-deb`** | The low-level "build/inspect one crate" tool | The hand-packing station |
| **`debhelper` / `dh`** | The toolchain that *generates* scripts & builds the real `.deb` | The automated packing line |
| **`lintian`** | Policy linter for packages | QA inspector at the end of the line |

---

## 15. Where to go next

You've built, installed, broken, fixed, upgraded, and purged a real package. Natural next steps:

- **Convert `ticktock` to a proper source package** — scaffold a `debian/` dir with `dh_make`, replace your hand-written service scripts with `dh_installsystemd`, and build with `debuild`. Diff the *generated* `postinst` against the one you wrote in §6 — the best possible way to learn what the automation is doing.
- **Ship a compiled program** — repeat with a tiny C program so `Architecture: amd64`, `Depends` on real shared libraries (`dpkg-shlibdeps` computes them), and `dh_strip`/`dh_shlibdeps` earn their keep.
- **Host it in a repository** — sign it, build an index with `reprepro` or `aptly`, add it to `sources.list` with a `signed-by` key (Lesson 01 §5, §7), and `apt install` your own package *over the network* — completing the supply chain from the warehouse end.
- **[01 — Package Management](01-package-management.md)** — re-read it now. Every command (`dpkg -L`, `dpkg --configure -a`, the `ii`/`iF` codes, `remove` vs `purge`) will read completely differently, because you've now stood on *both* sides of the crate.
- **[systemd Services](../04-systemd-services.md)** — your `postinst` enabled a unit and your `prerm` stopped it; go deeper on the unit-file machinery your package leaned on.

The instinct to carry forward: **a `.deb` is not a mysterious binary blob — it's a folder you could have arranged by hand**, with paperwork and four scripts. Once you've built one, every package on your system becomes legible: you can `dpkg-deb --raw-extract` *any* `.deb`, read its `control` and its `postinst`, and know exactly what it will do to your machine before you install it.
