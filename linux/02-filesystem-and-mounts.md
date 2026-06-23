# Lesson 2 — The Linux Filesystem: One Tree, Many Disks

> This is the most *foundational* lesson in the series — everything else stands on it. [Lesson 3](03-disk-usage-and-resource-monitoring.md) talked about "the single tree and mount points" and promised this deeper explanation; here it is. By the end you'll understand not just *what* the Linux filesystem is, but you'll be able to read a real `df -h` output — including the genuinely confusing one at the end of this lesson — and explain **where every line comes from**: which are real disks, which are partitions of one disk, which live in RAM, which are firmware, and which are halfway across the internet. If mounts have never quite clicked, this is the lesson where they do.

---

## 0. The mental model: one tree, not many drives

Start with the idea that everything else hangs from, because it's the single biggest difference between Linux and the Windows world many people come from.

**On Windows**, storage is a set of *separate, parallel worlds*, each with a letter: `C:`, `D:`, `E:`. Your second hard drive is a different universe (`D:\`) from your first (`C:\`). To move between them you change drives.

**On Linux, there is exactly one filesystem tree**, and it grows from a single root directory written as `/` (just a forward slash). *Everything* — every file, every directory, and the contents of *every* disk — hangs somewhere off that one root. There are no drive letters. Your operating system, your home folder, your USB stick, a network share in Azure: all of them appear as directories within the *same* unified tree.

The analogy I'll use throughout — and it's worth holding onto, because the whole lesson is built on it: **the filesystem is one enormous building with a single front door (`/`).** Inside are rooms (directories) and rooms-within-rooms (subdirectories). You walk the hallways seamlessly. But here's the twist that makes Linux storage make sense: **some of the doors in this building don't lead to rooms of the original building at all — they open into entirely separate buildings** (other disks), grafted on so smoothly that, walking the halls, you'd never know you'd crossed into a different physical structure. The act of connecting one of those separate buildings to a door is called **mounting**, and it is the central concept of this lesson.

So there are really two big questions this lesson answers, and they're worth stating up front:

1. **What's the layout of the building?** — i.e. what are all those top-level directories (`/etc`, `/var`, `/home`, `/dev`...) *for*? (Sections 1–2)
2. **How do separate disks get grafted into it?** — i.e. partitions, filesystems, and *mounting* (Sections 3–8)

> **Why "one tree" is a brilliant design, not a quirk:** because there's a single namespace, a program never needs to care *which physical disk* a file lives on — it just uses a path like `/home/doru/notes.txt`. You can move `/home` to a bigger disk tomorrow and **not a single program needs to change**, because the *path* stays the same even though the *hardware behind it* changed. The unified tree is what lets the physical and logical layers move independently. Hold that thought — it's the entire point of mounting.

---

## 1. The map of the building: the Filesystem Hierarchy Standard

If everything lives in one tree, you need to know the *layout* — which is not random. Linux follows a convention called the **Filesystem Hierarchy Standard (FHS)**: a shared agreement about what each top-level directory is *for*, so that every Linux system is navigable the same way. Think of it as the building's standardized floor plan — every branch of this franchise puts the kitchen, the offices, and the storage in the same places.

Here are the rooms that matter, each with its *purpose* (the "why it exists" is what makes them stick):

- **`/`** — the **root** of everything. The front door. Every path ultimately starts here. (Don't confuse it with `/root`, below — different thing.)
- **`/etc`** — **system configuration.** Plain-text config files for the whole machine live here (`/etc/fstab`, `/etc/ssh/`, the systemd unit overrides from Lesson 4). The building's *rulebooks and settings binders*. Mnemonic: "Editable Text Configuration."
- **`/home`** — **users' personal directories.** `/home/doru` is your stuff: documents, downloads, your dotfiles. One subdirectory per human. The *private apartments.*
- **`/root`** — the **home directory of the `root` (administrator) user** specifically. Kept separate from `/home` so the admin's files are available even if `/home` (often a separate disk!) isn't mounted. The *building superintendent's apartment*, deliberately near the entrance.
- **`/usr`** — the bulk of the **installed software**: programs (`/usr/bin`), libraries (`/usr/lib`), shared data. Counterintuitively *not* "user files" — historically "Unix System Resources." The *main supply and equipment wing.*
- **`/var`** — **variable data** that grows and changes as the system runs: logs (`/var/log`), caches, spools, databases, container storage. The room that *fills up over time* — which is exactly why Lesson 3's "disk is full" hunts so often end here.
- **`/tmp`** — **temporary scratch space**, often wiped on reboot, that any program can scribble in. The *whiteboard nobody promises to preserve.*
- **`/boot`** — the files needed to **start the machine**: the Linux kernel itself and the boot loader. The *ignition system.* (`/boot/efi`, which you'll see in the payoff, is a special sub-case we'll explain.)
- **`/dev`** — **device files**: the doorways to hardware, represented as files (Section 2). The *utility access panels.*
- **`/proc`** and **`/sys`** — **virtual windows into the running kernel** — not real files on disk, but live kernel data presented *as if* it were files (you met `/proc/<pid>/` in Lesson 7). The building's *live status dashboards.*
- **`/run`** — **runtime state** for currently-running programs (PIDs, sockets, locks), recreated fresh on every boot. Scratch space for "what's happening right now."
- **`/mnt`** and **`/media`** — conventional places to **attach extra/removable filesystems**: `/mnt` for manual/temporary mounts (the Azure share in the payoff lands here), `/media` for auto-mounted removable media (USB sticks, DVDs). The *loading docks* where you connect outside storage.
- **`/opt`** — **optional, self-contained third-party software** that doesn't follow the usual layout. The *guest wing.*

> **You don't memorize this — you internalize the *logic*.** Config in `/etc`, your files in `/home`, software in `/usr`, growing data in `/var`, hardware in `/dev`, kernel windows in `/proc` and `/sys`, runtime scratch in `/run`. Once the *purposes* click, you can predict where things live and read any system fluently. The FHS is the reason a Linux machine you've never seen before is still navigable on sight.

---

## 2. "Everything is a file" and the `/dev` directory

Before we can talk about disks and partitions, we need one of Linux's most famous design philosophies, because it explains what those `/dev/nvme0n1p5`-looking things in your `df` output actually *are*.

**In Linux, almost everything is represented as a file** — including hardware. Your disk, your terminal, even sources of random numbers — the kernel presents them as file-like objects living under **`/dev`** ("devices"). This is profound: it means the same simple operations (open, read, write) work on a text file, a disk, and a network of plumbing underneath. The hardware's complexity is hidden behind the familiar idea of "a thing you can read from and write to."

The devices we care about here are **block devices** — storage hardware that the kernel reads and writes in fixed-size *blocks*. They show up in `/dev` with names that *encode what they are*, and decoding those names removes most of the mystery from your `df` output:

- **`/dev/sda`, `/dev/sdb`** — SATA/SCSI/USB disks. `sd` = "SCSI disk"; the trailing letter counts them: `sda` is the first such disk, `sdb` the second.
- **`/dev/nvme0n1`** — an **NVMe SSD** (the fast modern kind that plugs directly into the motherboard). Decode it piece by piece: `nvme` = the protocol, `0` = controller number 0 (the first NVMe device), `n1` = namespace 1 (NVMe's term for a storage unit on that controller). So `nvme0n1` is "the first storage unit on the first NVMe controller" — *one physical SSD.*

That last one is the key to your `df` output, because you'll see `/dev/nvme0n1p1`, `...p5`, `...p6` — and now the suffix is decodable too. The **`p1`, `p5`, `p6` are partitions** of that *single* NVMe disk. Which brings us to the next concept.

> **The takeaway:** a name like `/dev/nvme0n1p5` isn't gibberish — it's a precise address: *partition 5, on namespace 1, of NVMe controller 0.* Once you can read the name, "where is this coming from?" is half-answered: it's a slice of your one internal SSD.

---

## 3. Partitions: where they come from, and why one disk becomes many

Here's a question that confuses everyone looking at output like yours: *"I have one SSD — so why do I see `nvme0n1p1`, `p5`, and `p6`? Where did those come from?"*

The answer is **partitioning**: the practice of dividing a single physical disk into multiple independent regions, each of which *behaves like a separate disk.* A partition is a contiguous slice of the drive, walled off from the others.

The analogy: your physical SSD is one big **plot of land**. Partitioning is drawing **fence lines** to divide it into separate lots, each usable for a different purpose — one lot for the house, one for a garden, one for a workshop. The land is one piece of hardware; the lots are logical divisions you imposed on it.

How does the disk *know* where the fences are? Through a small **partition table** written at the very start of the disk — a table of contents listing each partition's start, size, and type. Modern systems use a scheme called **GPT** (GUID Partition Table); older ones used **MBR**. When you look at the disk, the kernel reads this table and presents each partition as its own block device: `nvme0n1p1`, `nvme0n1p2`, and so on. The `pN` suffix literally means "partition N as recorded in this disk's partition table."

**Where did *yours* come from?** Almost certainly **the Ubuntu installer created them when you installed the OS.** A typical install carves the disk into purpose-specific partitions: a small one for booting (the EFI partition), a big one for the operating system (root, `/`), and often a separate one for your personal files (`/home`). Your numbering (`p1`, then `p5`, `p6`) reflects the installer's particular layout — gaps in the numbers are completely normal and usually mean other partitions exist that simply aren't shown in `df` (read on for *why* some partitions are invisible to `df`).

> **A crucial subtlety that explains "missing" partitions:** `df` lists **mounted filesystems**, *not* all partitions. A partition that isn't mounted — most commonly a **swap partition** (disk space used as overflow for RAM, which holds no filesystem and is never "mounted" into the tree) — simply *won't appear in `df` at all.* So if you see `p1`, `p5`, `p6` in `df` but suspect there's a `p2`/`p3`/`p4`, they're likely swap, recovery, or unmounted partitions. **The tool that shows you *every* partition regardless of mounting is `lsblk`** (from Lesson 3) — always cross-check there when `df`'s picture seems incomplete.

---

## 4. Filesystems: turning a raw partition into something usable

A partition, freshly created, is just a walled-off expanse of empty blocks — like a plot of land that's been fenced but has no building, no rooms, no addressing system. You can't just drop files into raw blocks and expect to find them again. You need an *organizing scheme*: a way to track which blocks belong to which file, where directories are, what each file is named, who owns it. That scheme is a **filesystem**, and writing a fresh one onto a partition is called **formatting**.

So the layering, bottom to top, is:

1. **Physical disk** (`/dev/nvme0n1`) — the hardware.
2. **Partition** (`/dev/nvme0n1p5`) — a fenced-off slice of it.
3. **Filesystem** — the organizing structure *written into* that partition (the shelving, labeling, and catalog system).
4. **Mount point** — *where in the tree* that filesystem is attached so you can actually use it (Section 5).

Different filesystems organize storage in different ways, optimized for different goals. The ones you'll meet:

- **`ext4`** — the long-standing default on Ubuntu for regular disks. Reliable, general-purpose. Your `/` and `/home` are almost certainly ext4.
- **`vfat` / FAT32** — an ancient, simple, *universally understood* format. Important for one specific job: the **EFI System Partition** (`/boot/efi`) *must* be FAT, because the computer's firmware knows how to read FAT and uses it before Linux is even running (Section 10 explains why).
- **`xfs`, `btrfs`, `zfs`** — other filesystems with various advanced features (snapshots, huge scale). Good to recognize the names.
- **`tmpfs`** — a filesystem that lives in **RAM**, not on disk. This one is so important to your `df` output that it gets its own section (Section 7).

> **Device vs. filesystem — the distinction that prevents confusion:** a *partition* is the container (the fenced lot); the *filesystem* is the organization *inside* it (the building and its addressing). The same partition could be reformatted with a different filesystem, erasing everything — because the filesystem *is* the structure that makes the bytes meaningful. When Lesson 3 said `df` reports "per filesystem," this is what it meant: each row is one formatted, mounted filesystem.

---

## 5. Mounting: the keystone concept (this is the one that finally clicks)

Everything so far has been setup for *this*. You have a formatted filesystem sitting in a partition. It's organized and ready. But it is **not yet part of the tree** — you can't reach it, because nothing connects it to a path you can navigate to. **Mounting is the act of attaching a filesystem to a specific directory in the unified tree**, and that directory is called its **mount point.** From the instant you mount it, walking into that directory means walking into that filesystem.

Here is the analogy that makes it permanent in your mind. Recall the building with doors. **A mount point is an ordinary empty room with a door.** Before you mount anything, the room is just... an empty room in the main building. Now you **mount a separate building (a disk/filesystem) onto that door**: from then on, opening that door no longer shows the empty room — it leads *into the other building entirely.* Everything inside the other building now appears as if it were rooms beyond that door. **Unmounting** reverses it: you disconnect the other building, and the door once again opens onto the original empty room.

Two consequences fall out of this that explain real behavior you'll encounter:

- **A mount point is just a directory.** `/home`, `/boot/efi`, `/mnt/doru-fileshare` — these are ordinary directories that *happen to have a filesystem mounted on them.* Nothing about the directory itself is special; it's the *act of mounting* that makes it a portal.
- **Whatever was in the directory *before* mounting becomes hidden** while something is mounted there (it's "behind" the mounted building's door — still there, just shadowed until you unmount). This is why mount points are usually created empty.

The "**Mounted on**" column in `df` — the rightmost one in your output — is *literally telling you which door each filesystem is attached to.* That's the whole meaning of that column: "this filesystem (`/dev/nvme0n1p6`) is reachable by walking into *this* directory (`/home`)."

The commands:

- **`mount`** (run with no arguments) — lists everything currently mounted and where. The friendlier, tree-shaped versions are **`findmnt`** (a readable tree of all mounts) and **`lsblk`** (devices with their mount points).
- **`sudo mount /dev/sdb1 /mnt/usb`** — manually attach a filesystem to a directory. ("Connect this building to this door.")
- **`sudo umount /mnt/usb`** — detach it. (Note the missing 'n' — it's `umount`, a classic Unix spelling quirk.) This is also why you "safely eject" a USB stick: unmounting flushes any not-yet-written data and cleanly disconnects, so you don't pull the building away mid-delivery.

> **The click moment:** "mounting" sounds technical but it's just *grafting a separate storage device onto a directory so it becomes part of the one big tree.* The path you navigate (`/home/...`) is the *logical* address; the device behind the mount (`/dev/nvme0n1p6`) is the *physical* reality; the **mount** is the bridge between them. Every confusing thing about Linux storage dissolves once you see that the tree is *assembled* from many separate filesystems, each grafted at a mount point.

---

## 6. Making mounts stick: `/etc/fstab` and UUIDs

Mounting with the `mount` command is *temporary* — it lasts until reboot. So how does your machine know, every single time it boots, to mount `/dev/nvme0n1p6` at `/home` and the EFI partition at `/boot/efi`, automatically? Through a configuration file: **`/etc/fstab`** ("filesystem table"), the persistent list of *what to mount where, and how*, read at every boot.

Each line in `/etc/fstab` is one mount instruction with several fields: *what* device, *where* to mount it (the mount point), *what* filesystem type, *which* options, and a couple of fields for checking/dumping. It's the building's **standing work order**: "every morning when we open, connect building X to door Y, building Z to door W."

One detail in `fstab` is worth understanding because it explains a name you'll see: devices are usually identified there not by `/dev/nvme0n1p6` but by a **UUID** — a long unique identifier like `UUID=a1b2c3d4-...`. Why? Because device *names* can change: plug in another disk, or change a cable, and what was `nvme0n1` might enumerate differently, and suddenly `/home` would mount the wrong thing. The **UUID is baked into the filesystem itself when it's formatted**, so it identifies the *correct* filesystem no matter what the kernel happens to name the device this boot. It's the difference between "the building at 3rd and Main" (an address that could be reassigned) and "the building with *this* unique deed number" (unambiguous forever). Robust mounting uses the deed number.

> **Connecting forward to Lesson 4:** `/etc/fstab` is the *classic* mechanism, but systemd (Lesson 4) actually translates these entries into **mount units** and performs the mounting as part of boot — which is why you saw mount as one of systemd's unit *types*. So "what mounts your disks at boot" is: firmware → kernel → systemd reading `fstab`. We'll trace that whole chain in Section 10.

---

## 7. The other kind of filesystem: virtual / RAM-backed (and why most of your `df` rows are these)

Here's the fact that makes your specific `df` output so confusing at first glance — and so illuminating once understood: **most of those lines are not disks at all.** They're **virtual filesystems** — filesystems the kernel conjures up that have *no physical storage behind them.* The most important kind for reading `df` is **`tmpfs`**.

**`tmpfs` is a filesystem that lives entirely in RAM** (spilling into swap if needed). It's real, you can read and write files in it, it shows up in `df` — but it's *memory pretending to be a disk*, not a partition on your SSD. The moment you understand `tmpfs`, two-thirds of your `df` output stops being mysterious. Key properties:

- **It's fast** (it's RAM) and **ephemeral** (gone on reboot — perfect for runtime scratch data that shouldn't persist).
- **Its "Size" is a configured *maximum*, not real consumption.** When `df` shows a tmpfs as `16G`, that does *not* mean it's using 16 GB of RAM — it means it's *allowed* to grow up to ~16 GB, and it only consumes actual memory for what's *currently written* (the "Used" column). A `tmpfs` with `322M` used is using 322 MB of RAM right now, no matter how big its ceiling looks.
- This is also why a tmpfs ties **disk** to **memory** (Lesson 8): filling a tmpfs consumes RAM, not disk space.

Linux uses tmpfs for all the places that need fast, temporary, reboot-fresh scratch space: `/run` (runtime state), `/dev/shm` (shared memory for inter-process communication), `/run/lock` (lock files), `/run/user/<id>` (per-user runtime), and on modern Ubuntu, often `/tmp` itself.

The other virtual filesystems you'll see (each presenting *kernel or firmware data* as files, not storing anything on disk):

- **`proc`** (at `/proc`) and **`sysfs`** (at `/sys`) — live windows into the kernel and hardware (Lesson 7's `/proc/<pid>/`).
- **`devtmpfs`** (at `/dev`) — the device files from Section 2.
- **`efivarfs`** (at `/sys/firmware/efi/efivars`) — exposes the **UEFI firmware's stored variables** (boot settings kept in a tiny chip on the motherboard) as files. This one has a famous gotcha we'll hit in the payoff: it's *minuscule* and can show a scary-looking high "Use%" that means absolutely nothing about your disk.

> **The reframe that decodes `df`:** a `df` listing is *not* "a list of my disks." It's "a list of everything currently mounted into the tree" — and that's a *mix* of real disk partitions, RAM-backed `tmpfs` scratch areas, kernel/firmware virtual filesystems, and possibly network shares. The skill is *sorting each row into its category*, which is exactly what we'll now do with yours.

---

## 8. Filesystems over the network

One more category, because your output contains a striking example. A mounted filesystem doesn't have to be local hardware *at all* — it can live on **another machine across the network**, mounted into your tree so that reading a "local" path actually sends bytes over the internet. The common protocols:

- **NFS** — the traditional Unix network filesystem.
- **CIFS/SMB** — the Windows-world file-sharing protocol, also used by cloud services like **Azure Files**. A path like `//server/share` mounted at a local directory is an SMB mount.

The magic of the unified tree (Section 0) shines here: once mounted, a network share at `/mnt/doru-fileshare` looks and acts like any other directory. Your programs neither know nor care that every read is a network round-trip to a datacenter. That's the payoff of "one tree, paths hide the hardware" taken to its logical extreme — the "hardware" is in the cloud.

---

## 9. The payoff: decoding your actual `df -h`, line by line

Now we read *your* output with everything we've built. Here it is again:

```
Filesystem                                        Size  Used Avail Use% Mounted on
tmpfs                                             6.2G  3.3M  6.2G   1% /run
/dev/nvme0n1p5                                    196G  130G   57G  70% /
tmpfs                                              16G  322M   15G   3% /dev/shm
efivarfs                                          184K  169K   11K  95% /sys/firmware/efi/efivars
tmpfs                                             5.0M   12K  5.0M   1% /run/lock
tmpfs                                             1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs                                             1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/nvme0n1p1                                    256M   53M  204M  21% /boot/efi
/dev/nvme0n1p6                                    344G  216G  112G  66% /home
tmpfs                                              16G  105M   16G   1% /tmp
//dorucloud.file.core.windows.net/doru-fileshare  5.0G  5.0G     0 100% /mnt/doru-fileshare
tmpfs                                             3.1G  1.8M  3.1G   1% /run/user/1000
```

The single most clarifying move is to **sort the twelve rows into their categories** from the sections above. Do that and the wall of text becomes three short, sensible stories.

### Category A — Your actual disk: the three real partitions of `/dev/nvme0n1`

These are the only rows that occupy space on your physical SSD. All three are partitions of your *one* NVMe drive (Section 3):

- **`/dev/nvme0n1p5` → `/` (196G, 70% used)** — your **root filesystem**: the operating system, `/usr`, `/etc`, `/var`, and everything not on another mount. Partition 5 of the SSD. At 70% it's healthy but worth keeping an eye on.
- **`/dev/nvme0n1p6` → `/home` (344G, 66% used)** — your **personal files**, on a *separate, larger* partition. This is the classic "keep `/home` separate" layout (Section 3): your data lives on its own partition so it's insulated from the OS. Partition 6.
- **`/dev/nvme0n1p1` → `/boot/efi` (256M, 21% used)** — the tiny **EFI System Partition**, formatted FAT (Section 4), holding the boot loader the firmware needs (Section 10 explains why it's first and small). Partition 1.

Notice the gap: you have `p1`, `p5`, `p6` but no `p2`/`p3`/`p4` here. Per Section 3, those almost certainly exist (likely a **swap** partition and/or recovery) but aren't *mounted filesystems*, so `df` doesn't show them. **Run `lsblk` to see the complete partition picture** — that's the tool that reveals what `df` omits.

### Category B — RAM, not disk: the `tmpfs` family (seven rows!)

Over half your listing is `tmpfs` — *memory pretending to be disk* (Section 7), consuming **zero** SSD space:

- **`tmpfs → /run` (6.2G)** and **`tmpfs → /run/lock` (5.0M)** and **`tmpfs → /run/user/1000` (3.1G)** — runtime scratch state for the system and for your user session (user ID `1000` is you, the first human account). Recreated fresh every boot.
- **`tmpfs → /dev/shm` (16G)** — **shared memory** for inter-process communication; programs use it to pass data fast.
- **`tmpfs → /tmp` (16G)** — your **temporary directory**, RAM-backed on this system (so big temp files here eat *memory*, not disk).
- **`tmpfs → /run/credentials/systemd-journald.service`** and **`.../systemd-resolved.service` (1.0M each, 0 used)** — tiny, locked-down RAM areas where **systemd hands secrets/credentials to specific services** (you'll recognize `journald` from Lesson 5 and `resolved` from Lesson 4's service catalogue). They're empty here; they exist as secure scratch space.

Remember: those big sizes (`16G`) are *ceilings*, not usage. The `Used` column is the truth — e.g. `/dev/shm` is using 322 MB of RAM, not 16 GB.

### Category C — Firmware: the `efivarfs` curiosity (and its harmless "95%")

- **`efivarfs → /sys/firmware/efi/efivars` (184K, 95% used)** — this is **not disk** and **not a problem.** It's a virtual filesystem exposing your motherboard's **UEFI firmware variables** (boot settings stored in a tiny NVRAM chip) as files (Section 7). The whole thing is 184 *kilobytes* — and `95%` of a space that small is meaningless for your storage. **This row alarms people and should be ignored**; a high percentage on a firmware area is normal and says nothing about your SSD's health. A perfect lesson in *reading `df` by category*: a number that would be terrifying on a disk partition is irrelevant on firmware NVRAM.

### Category D — The network: your Azure file share (and a *real* 100%)

- **`//dorucloud.file.core.windows.net/doru-fileshare → /mnt/doru-fileshare` (5.0G, 100% used)** — a **CIFS/SMB network mount** of an **Azure Files** share (Section 8), grafted into your tree at `/mnt` (the conventional spot for extra mounts, Section 1). The `//host/share` form is the dead giveaway of an SMB mount. Every read/write here crosses the internet to Microsoft's datacenter.
  - **This `100%` is the one you should actually care about** — unlike the firmware row, this is a real, full filesystem: your 5 GB Azure share has **0 bytes available.** Writes to it will fail until you free space or raise the share's quota. *This* is the row a trained eye stops on.

> **The whole output, now legible:** three rows are your real SSD (root, home, boot), seven are RAM-backed scratch (`tmpfs`), one is harmless firmware (`efivarfs`), and one is a full network share to worry about (Azure). You went from "where is all this coming from?" to a precise account of every line — and you can now do this for *any* machine's `df`. **That sorting instinct — disk vs. RAM vs. virtual vs. network — is the real skill this lesson teaches.**

---

## 10. The boot story: where the partitions come from and how the tree assembles

To truly close the loop on "where do these partitions come from," it helps to see the *moment they're used* — the boot sequence — because it explains *why* the partitions are shaped the way they are (especially that small FAT `/boot/efi`). This also stitches together Lessons 2, 4, and 6 into one picture.

When you press the power button, the tree you've been exploring doesn't exist yet — it has to be *assembled*, in order:

1. **Firmware (UEFI) runs first.** Built into the motherboard, it knows nothing about Linux or ext4. But it *can* read **FAT**, and it knows to look for a special partition — the **EFI System Partition** — to find a boot loader. *This is why `/boot/efi` exists, is FAT-formatted, and is partition 1:* it's the one partition the firmware itself can read, the bridge from "dumb firmware" to "your operating system." (Section 4's "vfat must be FAT" claim, now justified.)
2. **The firmware loads the boot loader** (GRUB on Ubuntu) from that EFI partition.
3. **The boot loader loads the Linux kernel** (from `/boot`) into memory and hands control to it.
4. **The kernel initializes hardware** (narrating it in `dmesg`, Lesson 6), then **mounts the root filesystem** (`/dev/nvme0n1p5` at `/`) — the first graft onto the tree. Now `/` exists.
5. **The kernel starts `systemd` (PID 1)** — Lesson 4's building manager — as the first process.
6. **systemd mounts everything else**, reading `/etc/fstab` (Section 6): it grafts `/home` (`p6`), `/boot/efi` (`p1`), and the Azure share onto their mount points, and sets up all the `tmpfs` scratch areas. The tree you saw in `df` is now fully assembled.

> **The grand connection:** the partitions "come from" the installer (Section 3), but they *become a navigable tree* through this boot-time assembly — firmware to kernel to systemd, each grafting more filesystems on. Your `df` output is a **snapshot of the finished tree** that this sequence built. Lessons 2 (this tree), 4 (systemd doing the mounting), and 6 (the kernel booting) are three views of the same machine coming to life.

---

## 11. The command cheatsheet

Grouped by intent.

### See the storage layout (start here)

| Command | What it shows |
|---------|--------------|
| `lsblk` | Tree of **all** disks → partitions → mount points (**shows what `df` omits**, e.g. swap) |
| `lsblk -f` | Same, plus filesystem type, label, and UUID |
| `df -h` | Capacity of every **mounted filesystem**, human-readable |
| `df -hT` | Same, with a **Type** column (instantly reveals tmpfs vs ext4 vs vfat vs cifs) |
| `findmnt` | All mounts as a readable **tree** |
| `mount` | Raw list of everything mounted and its options |

### Understand a specific filesystem or device

| Command | What it shows |
|---------|--------------|
| `findmnt /home` | What's mounted at `/home` and how |
| `df -h /home` | Capacity of the filesystem backing a given path |
| `lsblk -f /dev/nvme0n1` | The partitions and filesystems on one disk |
| `blkid` | Each partition's UUID and filesystem type |
| `cat /etc/fstab` | The persistent "what mounts where at boot" table |

### Mount and unmount

| Command | What it does |
|---------|-------------|
| `sudo mount /dev/sdb1 /mnt/usb` | Attach a filesystem to a directory (temporary, until reboot) |
| `sudo umount /mnt/usb` | Detach it (note: no 'n' — `umount`) |
| `mount \| grep nvme` | Check what's mounted from a given device |

### Identify what kind each filesystem is

| Look for | It means |
|----------|----------|
| `/dev/sd*`, `/dev/nvme*` | A real disk partition |
| `tmpfs` | RAM-backed scratch (no disk used; size is a ceiling) |
| `proc`, `sysfs`, `devtmpfs`, `efivarfs` | Virtual kernel/firmware filesystem (not storage) |
| `//host/share` (cifs) or `host:/path` (nfs) | A network share |

---

## 12. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **`/` (root)** | The single origin of the entire filesystem tree | The building's only front door |
| **the unified tree** | All files & disks hang off one root (no drive letters) | One building, many connected wings |
| **FHS** | The standard layout of top-level directories | The franchise floor plan |
| **block device** | Storage hardware presented as a file under `/dev` | A utility access panel |
| **`/dev/nvme0n1`** | One physical NVMe SSD (controller 0, namespace 1) | One plot of land |
| **partition (`p5`)** | A walled-off slice of a disk, acting as its own disk | A fenced lot on the plot |
| **partition table (GPT)** | The disk's table-of-contents listing its partitions | The deed map of the lots |
| **filesystem (ext4, vfat)** | The organizing structure written into a partition | The building & addressing inside a lot |
| **formatting** | Writing a fresh filesystem onto a partition | Constructing the building |
| **mounting** | Attaching a filesystem to a directory in the tree | Connecting a separate building to a door |
| **mount point** | The directory a filesystem is attached at | The door it's connected to |
| **`/etc/fstab`** | Persistent list of what to mount where at boot | The standing daily work order |
| **UUID** | A filesystem's permanent unique ID (vs. changeable device name) | The unique deed number, not the street address |
| **`tmpfs`** | A filesystem living in RAM; ephemeral; size is a ceiling | A whiteboard, not a warehouse |
| **`/dev/shm`** | tmpfs for shared memory between processes | A shared scratch pad |
| **`efivarfs`** | Virtual view of UEFI firmware variables (tiny; ignore its %) | The motherboard's settings binder |
| **EFI System Partition** | Small FAT partition the firmware reads to boot | The ignition the firmware understands |
| **CIFS/SMB, NFS** | Filesystems mounted over the network | A wing that's actually in another city |
| **`lsblk`** | Lists all devices/partitions and mount points | The full map, including unmounted lots |
| **`df` vs `lsblk`** | `df` = mounted filesystems; `lsblk` = all partitions | Occupied rooms vs. every lot on the deed |

---

## 13. Where to go next

With the tree, partitions, and mounting understood, the rest of the series has solid ground:

- **[Lesson 3 — Disk Usage & Resource Monitoring](03-disk-usage-and-resource-monitoring.md)** — now that you know *what* a filesystem and mount point are, go measure how full each is and hunt what fills them. This lesson is the *why*; Lesson 3 is the *watch*.
- **[Lesson 4 — systemd Services](04-systemd-services.md)** — the manager that actually performs the boot-time mounting (Section 10), via mount units and `fstab`.
- **[Lesson 8 — Memory Deep-Dive](08-memory-deep-dive.md)** — pairs directly with `tmpfs` (Section 7): since tmpfs *is* memory, understanding RAM, the page cache, and swap completes the picture of where your "disk" space in `/tmp` and `/dev/shm` really goes.
- **Partitioning & LVM hands-on** — creating and resizing partitions, `mkfs` (formatting), and **LVM** (logical volumes that let a single mount point span *multiple* physical disks — mounting taken to its flexible extreme).
- **Permissions & ownership** — now that you can find any file in the tree, the natural next question is *who's allowed to read and write it* (`chmod`, `chown`, the `rwx` model).

The instinct to carry forward: **a Linux system is one tree assembled from many separate filesystems — so when you see a path, ask "which mounted filesystem does this actually live on?", and when you see a `df` row, ask "is this real disk, RAM, virtual, or network?"**
