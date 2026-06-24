# Lesson 10 — D-Bus: the System's Switchboard

> [Lesson 9](09-ipc-pipes-and-sockets.md) ended on the Unix domain socket: a two-way telephone line between two local processes. That's enough for *one* pair to talk. But a real Linux desktop or server has **dozens** of programs that all need to reach each other — `systemctl` must call PID 1, your file manager must ask to mount a drive, a power daemon must tell the screen to dim, an app must pop a notification, the network applet must query NetworkManager. If every pair invented its own socket and its own private language, you'd have an unmaintainable snarl of point-to-point wires with no way to *discover* who's out there or *broadcast* an event to many listeners. **D-Bus is the answer to "this doesn't scale."** It's a structured **message bus** — a single switchboard everyone plugs into — built directly on the socket you just learned. [Lesson 4](04-systemd-services.md) already called it *"the building's internal phone system";* this lesson makes that real. And it pays off a hook from the packaging course too: the **portals** that sandbox Flatpak apps ([packaging/03](packaging/03-snap-and-flatpak.md)) are D-Bus.

---

## 0. The mental model: from a tangle of wires to one switchboard

Start from where Lesson 9 left off. Two processes talk over a Unix socket — fine. Now add a *third* service, and a fourth, all needing to talk to each other. With raw sockets you face three problems that get worse with every new program:

1. **No directory.** To call a service, you must already know the exact socket path it listens on. There's no "phone book" to ask *"is NetworkManager running, and where?"*
2. **No common language.** Each socket carries raw bytes; every pair must invent and agree on its own private wire format. Ten services talking to each other is potentially *dozens* of bespoke mini-protocols.
3. **No broadcast.** A socket connects *two* endpoints. If "the battery is low" needs to reach five different programs, the sender must open five connections and know all five in advance.

The classic fix for "everything-talks-to-everything" is to stop wiring peers directly and instead route everyone through **one central exchange.** That exchange is the **D-Bus daemon**: every program opens *one* socket — to the bus — and from then on talks to *everyone else through it.* The bus is a **telephone switchboard**:

```
   Without a bus (chaos):              With D-Bus (one switchboard):

   A───B   A───C                          A   B   C   D
   │ ╳ │   │ ╳ │                           \  |   |  /
   B───D   C───D       ───────▶             \ |   | /
   (N² ad-hoc sockets,                     ┌──────────┐
    each its own language)                 │  D-Bus   │  ← everyone connects once,
                                           │  daemon  │     speaks ONE language,
                                           └──────────┘     can call OR broadcast
```

Plugging into the switchboard buys you exactly the three missing things:

1. **A directory** — the bus knows every connected program by a **name**, and you can ask it "who's here?" and even "what can you do?" (*introspection*).
2. **A common language** — every message uses **one standard format** (typed arguments, standard headers), so any program can call any other without a bespoke protocol.
3. **Broadcast** — besides directed calls, the bus relays **signals**: one program emits "battery low" and the bus delivers it to *everyone who subscribed*, with the sender knowing nothing about the receivers.

> **Why this framing first:** D-Bus has a reputation for being baroque — bus names, object paths, interfaces, methods, signals, properties. It isn't baroque; it's a **switchboard plus an addressing scheme**, and every piece of jargon is one part of "how do I phone exactly the right service, department, and action through the exchange?" Hold the switchboard image and the whole vocabulary in §3 falls out of it naturally. And remember the foundation from Lesson 9: **the bus daemon is "just" a well-known server on a Unix socket** — everything below is *structure layered on that one socket.*

---

## 1. It's a socket underneath (the Lesson 9 anchor)

Before the abstractions, ground D-Bus in what you already know. The D-Bus daemon is a server process (`dbus-daemon`, or its faster modern replacement `dbus-broker`) doing exactly the Lesson 9 client–server dance: it `bind`s a Unix socket at a **well-known path** and `listen`s. Every other program is a *client* that `connect`s to that path. You can see the socket with the very tools from last lesson:

```
ls -l /run/dbus/system_bus_socket      # there it is — type 's', the system bus's phone line
sudo lsof /run/dbus/system_bus_socket  # every process currently plugged into the switchboard
```

That single socket is the switchboard's front desk. Once connected, a client doesn't open more sockets to reach other services — it sends all its messages *through this one connection*, and the daemon **routes** each message to the right recipient (for a call) or to all subscribers (for a signal). **D-Bus = a routing daemon on a Unix socket + a standard message format + an addressing scheme.** That's the whole thing; the rest is detail.

> **The anchor:** whenever D-Bus feels abstract, come back here. There is a real socket at `/run/dbus/system_bus_socket`, a real daemon `accept()`ing connections, and real bytes flowing — exactly the mechanism from Lesson 9. The "bus" is what that daemon *does* with the messages: read the address, route to the destination.

---

## 2. Two buses: system vs session (and why there are two)

The first thing to internalize is that a typical machine runs **two kinds of bus**, for the same reason a building separates its *facilities phone line* from *each tenant's personal phone*: **scope and privilege.**

- **The system bus** — **one per machine**, started at boot, used for **system-wide, privileged** services that concern the whole computer: `systemd` (PID 1 itself), `logind` (logins, power, lid-close, lock), NetworkManager, the Bluetooth daemon, `polkit`, UDisks (mounting drives). Its socket is the well-known `/run/dbus/system_bus_socket`. Access is **tightly controlled** — not just anyone may ask to reconfigure the network.

- **The session bus** — **one per logged-in user session**, used for **desktop / user-level** chatter: notifications, the screensaver, media-player controls (MPRIS), search providers, clipboard managers, and the **Flatpak/portal** machinery from the packaging course. Its address lives in the environment variable `$DBUS_SESSION_BUS_ADDRESS` (typically a socket at `/run/user/<uid>/bus` — another callback to Lesson 9's `/run`, here the per-user runtime dir).

```
echo $DBUS_SESSION_BUS_ADDRESS         # your session bus (e.g. unix:path=/run/user/1000/bus)
busctl --system  list                  # services on the SYSTEM bus (machine-wide)
busctl --user    list                  # services on YOUR SESSION bus (desktop apps)
```

The split mirrors the privilege boundary you already know: the **system bus is "root's switchboard"** (machine-wide, guarded), the **session bus is "your switchboard"** (your apps, your session, torn down at logout). A desktop app uses the session bus for everyday things and reaches *across* to the system bus (through guards like polkit, §7) only when it needs something privileged — like mounting a USB stick.

> **The placement:** two buses isn't redundancy, it's **separation of concerns by privilege and lifetime.** System bus = the machine's shared, long-lived, guarded exchange. Session bus = your session's private, ephemeral exchange. When you debug D-Bus, your *first* question is always "which bus?" — `--system` or `--user` — because the same tools point at either.

---

## 3. The addressing scheme: how you phone exactly the right thing

This is the conceptual core, and it's the part that looks intimidating until you see it's just a **four-level postal address** for reaching one specific action inside one specific program. To "call" something on D-Bus you name four things, narrowing from "which program" down to "which action":

```
   busctl call \
     org.freedesktop.login1 \                         ← 1. BUS NAME      (which program)
     /org/freedesktop/login1 \                        ← 2. OBJECT PATH   (which thing inside it)
     org.freedesktop.login1.Manager \                 ← 3. INTERFACE     (which set of capabilities)
     ListSessions                                     ← 4. MEMBER        (which method/action)
```

Walk the four levels with the switchboard analogy:

1. **Bus name — *which program*.** The connection's identity on the bus, like a company's phone number. There are two flavors: a **well-known name** in reverse-DNS form that services claim so others can find them — `org.freedesktop.NetworkManager`, `org.freedesktop.login1` — and a **unique name** the bus auto-assigns every connection, like `:1.42` (think of it as the raw line number behind the friendly name). You dial the well-known name; the bus translates it to whoever currently owns it.

2. **Object path — *which thing inside that program*.** One program can expose many **objects**, arranged in a hierarchy that *looks* like a filesystem path: `/org/freedesktop/NetworkManager/Devices/0` might be your Ethernet card, `/Devices/1` your Wi-Fi. It's the **department or extension** within the company — "connect me to the second-floor Wi-Fi desk." (The slashes are pure namespacing; these aren't real files.)

3. **Interface — *which set of capabilities*.** An **interface** is a named contract: a group of related methods, signals, and properties, in reverse-DNS form like `org.freedesktop.login1.Manager`. One object can implement several interfaces (its own, plus standard ones like `org.freedesktop.DBus.Properties`). It's the **service catalog** you're referencing — "from the *Manager* services, please."

4. **Member — *which action*.** Finally, the specific thing to do: a **method** to call (`ListSessions`), a **property** to read/write (`Version`), or a **signal** to listen for (`SessionNew`). The actual verb.

So a D-Bus call reads as one sentence: **"Through the bus, phone `login1` (name), connect me to the manager object (path), from its Manager interface (catalog), and invoke ListSessions (action)."** Intimidating syntax, mundane idea.

> **The reframe:** bus name → object path → interface → member is just **company → department → service-catalog → specific request.** Every D-Bus tool, library, and error message is organized around these four coordinates. Once you can read an address as those four answers, you can read *any* D-Bus interaction.

---

## 4. The four message types — calls vs broadcasts

Everything crossing the bus is one of just **four message types**, and they split cleanly into "directed conversation" and "broadcast":

**Directed (a phone call between two parties):**
- **Method call** — "please do X / give me Y," sent to a specific destination. Like dialing and asking a question.
- **Method return** — the reply carrying the result. The answer to your question.
- **Error** — a reply that says "that failed, here's why." A specific kind of return.

**Broadcast (a public-address announcement):**
- **Signal** — "this just happened!" emitted by a service to *no one in particular*. The bus delivers it to **every client that subscribed** to that signal. The sender neither knows nor cares who's listening.

The method-call trio is **request/response**, like the two-way socket conversation from Lesson 9 but with structure and routing added. Signals are the thing raw sockets *couldn't* do: **one-to-many broadcast with decoupled sender and receivers.** "Battery low," "a new login session started," "the screen locked," "a USB drive appeared" — these are signals, and any number of programs can react without the emitter knowing they exist.

### ⚠️ A crucial naming collision

A **D-Bus signal is NOT a Unix signal.** [Lesson 7](07-processes-and-monitoring.md)'s signals (`SIGTERM`, `SIGKILL`) are the kernel's payload-free process pokes ([Lesson 9 §2](09-ipc-pipes-and-sockets.md)). A **D-Bus signal** is a *rich, structured broadcast message* carrying typed data, relayed by the bus daemon to subscribers. Same word, completely different mechanism and layer. Keep them apart: *Unix signal = kernel → process, no data; D-Bus signal = service → bus → many apps, with data.*

> **The placement:** D-Bus gives you both halves of communication that a bare socket makes awkward — **point-to-point Q&A** (the method trio) *and* **publish/subscribe broadcast** (signals) — over one connection, with the daemon doing the routing. That combination is exactly why the whole desktop and half of systemd are built on it.

---

## 5. Properties and introspection — the self-describing bus

Two more standard features turn the bus from "a way to send messages" into "a discoverable, self-documenting system."

**Properties** — besides methods (actions) and signals (events), an object exposes **properties**: named, typed values you can *Get*, *Set*, and watch for changes. NetworkManager's connectivity state, logind's idle status, a media player's current track are all properties. They ride on a standard interface (`org.freedesktop.DBus.Properties`) with `Get`/`Set`/`GetAll` methods and a `PropertiesChanged` signal — so you can both *read* a value and *subscribe* to be told when it changes. Properties are the "status fields" of a service; methods are its "buttons."

**Introspection** — the feature that makes D-Bus *explorable*. Every object implements `org.freedesktop.DBus.Introspectable`, whose `Introspect` method returns a machine-readable description of **everything that object offers**: its interfaces, methods (with argument types), signals, and properties. This is why you can sit down at an unfamiliar machine and *discover* its entire D-Bus API with no documentation — the bus describes itself. Tools like `busctl introspect` and the `d-feet`/`Bustle` GUIs are just pretty front-ends over `Introspect`.

```
busctl --system introspect org.freedesktop.login1 /org/freedesktop/login1
# → lists logind's Manager interface: methods (Suspend, ListSessions...), signals, properties
```

> **The placement:** properties + introspection are why D-Bus is a *platform*, not just a pipe. A service publishes a typed, self-describing API; any client can discover it at runtime, call its methods, read its properties, and subscribe to its signals — all in one standard language. This is the payoff of routing everything through one structured switchboard.

---

## 6. Service activation — the bus starts things on demand

Here's an elegant tie-back to [Lesson 4](04-systemd-services.md). A service needn't already be running for you to address it. With **D-Bus activation**, the bus knows (from small `.service` files registered with it) *which program owns which well-known name*. When a message arrives for a name that **nobody currently owns**, the bus **starts that program on demand**, waits for it to claim the name, then delivers the message. The caller just sends a message; the service springs to life to receive it.

This is the *exact same idea* as systemd's **socket activation** from Lesson 4 (a service started lazily on first contact) — and indeed, on modern systems the two are unified: D-Bus activation is typically handled *by systemd*, so a `dbus-daemon` `.service` activation becomes a systemd unit start, with all of systemd's dependency, logging (Lesson 5), and resource-control machinery behind it. The switchboard and the service manager are wired together: addressing a name can *boot the tenant.*

> **The placement:** activation means the bus is **lazy and self-assembling** — services don't all have to be pre-started; they materialize when first addressed. It's the same lazy-start pattern as socket units, now keyed on a *bus name* instead of a port. One more reason the system bus is woven straight into systemd.

---

## 7. Security: who may call what (and polkit)

Routing every privileged request through one switchboard would be reckless without guards — so D-Bus has two layers of access control, and the second one (polkit) is something you've *seen* without naming.

**Bus policy** — the daemon enforces rules about **who may own which name and who may send which messages**. On the system bus these live in policy files (`/usr/share/dbus-1/system.d/`, `/etc/dbus-1/system.d/`): e.g. "only root may own `org.freedesktop.NetworkManager`," "only members of group `X` may call method `Y`." The bus checks the **verified peer credentials** it gets from the Unix socket (Lesson 9 §5 — the socket can hand the kernel-attested uid/pid of the caller), so a service can *trust* who's really calling. This is the security payoff of that credential-passing feature.

**Polkit (PolicyKit)** — for finer, *interactive* authorization, the system bus pairs with **polkit**, itself a system-bus service. The pattern you know from daily desktop use: you click "Install updates" or "Mount this drive" and a dialog asks for your password. Under the hood, the unprivileged app sends a D-Bus method call to a privileged system service (e.g. packagekit, UDisks); that service asks **polkit** "is this user *authorized* to do this action right now?"; polkit consults its rules and, if needed, **triggers the password prompt**; only on success does the privileged action proceed.

```
busctl --system introspect org.freedesktop.PolicyKit1.Authority \
       /org/freedesktop/PolicyKit1/Authority    # polkit's own D-Bus API
```

> **The placement:** the switchboard is also a **checkpoint.** Because every privileged request crosses the system bus, the bus (via policy + verified socket credentials) and polkit (via authorization rules + the password prompt) are the natural place to enforce "are you allowed to do this?" That authentication dialog you see a dozen times a week *is* a D-Bus + polkit conversation. It's how a sandboxed, unprivileged GUI app safely performs root-level actions without itself being root.

---

## 8. Doing it: `busctl` and friends

Enough theory — here's how to actually use the bus. The premier tool is **`busctl`** (part of systemd), because it speaks both buses and leans on introspection. The four verbs map onto the four addressing coordinates and message types from §3–4:

```
# WHO is on the bus? (the directory)
busctl --system list                 # every connection: well-known names + unique :1.x names

# WHAT can a service do? (introspection — §5)
busctl --system introspect org.freedesktop.login1 /org/freedesktop/login1

# CALL a method (§3, §4) — ask logind to list active sessions
busctl --system call org.freedesktop.login1 /org/freedesktop/login1 \
       org.freedesktop.login1.Manager ListSessions

# READ a property (§5) — what version of logind is running?
busctl --system get-property org.freedesktop.login1 /org/freedesktop/login1 \
       org.freedesktop.login1.Manager Version

# WATCH the bus live — see calls and signals stream past (the best way to learn)
busctl --system monitor                          # everything
busctl --user   monitor org.freedesktop.Notifications   # just one service
```

A satisfying session-bus example — fire a desktop notification *by hand*, proving the popup you see daily is just a method call:

```
busctl --user call org.freedesktop.Notifications /org/freedesktop/Notifications \
       org.freedesktop.Notifications Notify \
       susssasa\{sv\}i "hello" 0 "" "From D-Bus" "I am a raw method call" 0 0 5000
# → a notification pops up. That gibberish after Notify is the typed argument SIGNATURE.
```

Other front-ends you'll encounter: **`gdbus`** (GLib's CLI, common in GNOME scripting), **`dbus-send`**/**`dbus-monitor`** (the older reference tools), and GUIs **`d-feet`** and **`Bustle`** (visual introspection and message tracing). They all do the same four things — list, introspect, call, monitor — over the same buses.

> **The habit:** to understand *any* D-Bus-driven behavior on your system, **`busctl monitor` while you trigger it.** Lock your screen, plug in a USB drive, toggle Wi-Fi — and watch the calls and signals fly. It's the `strace` of the bus (and a direct sibling of Lesson 9's `strace` habit): the fastest way to turn "magic desktop behavior" into "oh, it called *that* method."

---

## 9. Where you already rely on D-Bus

To cement how central this is, here's the bus hiding behind things you use constantly — most of this lesson's concepts in the wild:

- **`systemctl` and `loginctl`** — when you run `systemctl restart nginx`, you're not poking files; the command sends a **D-Bus method call to PID 1** over the system bus. systemd's *entire* control surface is a D-Bus API. (Lesson 4, now demystified: that's *how* `systemctl` talks to the manager.)
- **Power & session** — suspend, lock, "what happens when I close the lid," idle detection: **logind** signals and methods on the system bus.
- **NetworkManager / Bluetooth** — `nmcli`, the Wi-Fi applet, pairing a headset: all D-Bus. Devices are *objects*, connection state is *properties*, "carrier changed" is a *signal*.
- **Desktop notifications** — every popup is a `Notify` method call on the session bus (you just sent one in §8).
- **Mounting drives** — your file manager mounting a USB stick goes app → UDisks → **polkit** authorization → mount. The password prompt is §7 in action.
- **Flatpak portals** ([packaging/03](packaging/03-snap-and-flatpak.md)) — a sandboxed app can't touch your files directly; it calls a **portal** *over the session bus*, which shows the file picker and hands back only the chosen file. The portal/permission system that lesson described **is** D-Bus. (And there's the promised payoff of that hook.)
- **Media keys (MPRIS)** — play/pause from your keyboard controlling whatever player is active: a standard D-Bus interface every compliant player implements.

> **The realization:** D-Bus isn't an obscure corner — it's the **nervous system of a modern Linux machine.** The desktop "just working" (notifications, power, network, mounting, sandboxing) and systemd "just working" (`systemctl`, logind) are the same switchboard, carrying method calls and signals between the dozens of cooperating processes that a bare socket could never have coordinated.

---

## 10. The command cheatsheet

Pick your bus with `--system` (machine-wide) or `--user` (your session), then:

| Command | What it does |
|---------|-------------|
| `busctl --system list` | Directory: every connection's well-known + unique names |
| `busctl --user list` | Same, for your session bus (desktop apps) |
| `busctl introspect NAME PATH` | Discover an object's interfaces, methods, signals, properties |
| `busctl call NAME PATH IFACE METHOD [SIG ARGS]` | Invoke a method (request/response) |
| `busctl get-property NAME PATH IFACE PROP` | Read a property's current value |
| `busctl set-property NAME PATH IFACE PROP SIG VAL` | Change a writable property |
| `busctl monitor [NAME]` | **Live-watch** all (or one service's) calls & signals — the bus's `strace` |
| `busctl tree NAME` | Show a service's object-path hierarchy |
| `gdbus introspect --session --dest NAME --object-path PATH` | GLib equivalent (common in GNOME) |
| `dbus-monitor --system` | Older reference monitor |
| `echo $DBUS_SESSION_BUS_ADDRESS` | Find your session bus socket |
| `lsof /run/dbus/system_bus_socket` | Who's plugged into the system bus (Lesson 9 tools) |

---

## 11. A glossary to cement the vocabulary

| Term | One-line meaning | The analogy |
|------|-----------------|-------------|
| **D-Bus** | A structured message bus routing IPC through one daemon | A telephone switchboard everyone plugs into |
| **bus daemon** | `dbus-daemon`/`dbus-broker` — the server on a Unix socket that routes messages | The switchboard operator |
| **system bus** | One per machine; privileged, system-wide services | The building's guarded facilities line |
| **session bus** | One per user session; desktop/app chatter | Your personal phone, gone at logout |
| **bus name** | A connection's identity: well-known (`org.x.Y`) or unique (`:1.42`) | A company's phone number |
| **object path** | `/like/a/path` — which object inside a service | The department/extension |
| **interface** | Named group of methods/signals/properties (`org.x.Y.Z`) | A service catalog |
| **member** | The specific method, signal, or property | The specific request/action |
| **method call / return / error** | Directed request → response (or failure) | A phone call and its answer |
| **signal (D-Bus)** | Broadcast "this happened" to all subscribers — *not* a Unix signal | A public-address announcement |
| **property** | A named, typed, gettable/settable value with a change signal | A status field on the service |
| **introspection** | An object describing its own API on request | The company handing you its full directory |
| **service activation** | The bus starts a service on first message to its name | Dialing an extension that summons the tenant |
| **polkit** | Authorization service + password prompt for privileged actions | The checkpoint guard asking for ID |
| **peer credentials** | Kernel-verified uid/pid of the socket's other end | Caller ID the operator can trust |
| **`busctl`** | systemd's tool to list/introspect/call/monitor the bus | The handset for working the switchboard |
| **portal** | A session-bus service giving sandboxed apps mediated access | A clerk who fetches one file for a quarantined guest |

---

## 12. Where to go next

You've now climbed from "two processes share a socket" (Lesson 9) to "the whole machine coordinates over one structured bus" (this lesson). Natural continuations:

- **[Lesson 9 — IPC, Pipes & Sockets](09-ipc-pipes-and-sockets.md)** — re-read §5 (Unix sockets, peer credentials): you'll now see *why* those features exist — they're exactly what the bus daemon and polkit lean on.
- **[Lesson 4 — systemd Services](04-systemd-services.md)** — revisit it knowing `systemctl` is a D-Bus client of PID 1, and that D-Bus activation and socket activation are the same lazy-start idea. The systemd↔D-Bus pairing is the backbone of the running system.
- **Write a tiny D-Bus service** — expose your own object, interface, and a method/signal (a few lines with `python-dbus`/`pydbus` or GLib), then call it with `busctl`. Nothing makes the four-level address concrete like *owning a name* and watching your method appear in `introspect`.
- **[packaging/03 — Snap & Flatpak](packaging/03-snap-and-flatpak.md)** — re-read the portals section; "sandboxed app asks a portal for a file" is now legible as a session-bus method call to a privileged helper.
- **Networking (future)** — the network-socket half of Lesson 9, expanded. D-Bus is deliberately *local* (one machine's switchboard); cross-machine messaging is a different world with its own trade-offs.

The instinct to carry forward: **when something on a Linux desktop or systemd box "just happens" — a notification, a screen lock, a service restart, a mount prompt — a D-Bus message moved between processes.** Reach for `busctl monitor`, watch the call or signal, read its four-part address, and the magic resolves into "program A invoked method M on program B through the switchboard." The nervous system becomes visible.
