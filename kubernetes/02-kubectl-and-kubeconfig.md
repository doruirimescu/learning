# Lesson 2 — kubectl & kubeconfig: The Universal Client

> [Lesson 1](01-architecture-control-loop.md) ended with a promise: now that you know `kubectl` is "just a client at the API-server front desk," the tool stops being a grab-bag of commands to memorize and becomes one simple, knowable thing. This lesson makes good on that. We'll demystify `kubectl` completely — it is a command-line program that turns your typing into HTTP requests to the API server, and almost nothing more. We'll learn the *grammar* that lets you compose commands you've never seen, and — most importantly for staying employed — we'll learn `kubeconfig`, the file that decides **which cluster you're aimed at and who you are**, because pointing the same command at the wrong cluster is the single most common self-inflicted disaster in this whole field.
>
> This lesson assumes you've internalized Lesson 1's model: state lives in etcd, the API server is the only door, and everything is CRUD on objects. If that sentence feels solid, this lesson will feel easy.

---

## 0. First, the mental model: a remote control and which TV it's pointed at

Everything in this lesson hangs on one picture.

> **`kubectl` is a remote control, and `kubeconfig` is which TV it's pointed at.** The buttons on the remote — `get`, `apply`, `delete`, `scale` — do the *same thing* no matter what. But what they *affect* depends entirely on which TV the remote is currently aimed at. Press "delete" while pointed at the playroom TV (your dev cluster) and you've turned off a cartoon. Press the *identical* button while pointed at the home-theater TV (production) and you've killed the movie everyone was watching. The buttons look the same. The remote gives you no warning. **So before you press anything that changes state, you check which TV you're pointed at.** That habit — boring, repetitive, unglamorous — is what separates engineers trusted with production from those who aren't.

Hold this image through the whole lesson. The first half ("the remote") is about the buttons: what `kubectl` is and how its grammar works. The second half ("which TV") is about `kubeconfig` and **contexts**: how you control, and verify, what you're aimed at. The second half is the one that saves your career.

---

## 1. What kubectl actually is: a command-line client for the API server

Let's kill the magic immediately. In Lesson 1 we established that the **API server** is the single door to the cluster, that it speaks HTTP, and that *everything* — controllers, kubelets, you — coordinates by reading and writing objects through it. `kubectl` is simply **a convenient command-line client for that HTTP API.** That's the entire truth of it.

When you type a command, `kubectl` does four unglamorous things:

1. Reads your **kubeconfig** to learn *which* API server to talk to and *how to identify you* (§4).
2. Translates your command into one or more **HTTP requests** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) against the API server's REST endpoints.
3. Sends them, attaching your credentials.
4. Receives the JSON responses and **formats them** for your terminal.

That's it. `kubectl get pods` becomes an HTTP `GET` for the list of Pod objects. `kubectl delete pod web` becomes an HTTP `DELETE` on one Pod object. `kubectl apply -f web.yaml` becomes a request to create-or-update the objects in that file. The cluster does the real work (controllers reconciling, as in Lesson 1); `kubectl` just *asks the front desk*, politely, over HTTP.

### Proving it to yourself: `--v=8`

You don't have to take this on faith. `kubectl` has a verbosity flag, `--v`, and at level 8 it prints the actual HTTP requests and responses it's making. Run:

```
kubectl --v=8 get pods
```

and you'll see lines like `GET https://<api-server>:6443/api/v1/namespaces/default/pods` followed by the raw JSON that comes back. **That URL *is* the truth of what `kubectl get pods` does.** Seeing this once, early, rewires how you think: every command becomes "what REST call does this make, against which object, as whom?" — and from there, permission errors, latency, and odd behavior all become reasoning problems instead of mysteries. Whenever a command surprises you later in your career, `--v=8` is how you find out what it *actually* sent.

This also explains a fact that trips people up: **`kubectl` can do nothing the API server won't let it do.** If you get `Forbidden`, that's not `kubectl` being difficult — it's the API server's authorization (RBAC, Lesson 9) rejecting *your identity*. `kubectl` is just the messenger relaying the door's answer.

---

## 2. The grammar: `kubectl <verb> <resource> [name] [flags]`

Here is the realization that turns `kubectl` from "a hundred commands to memorize" into "a small grid you can compose": **almost every `kubectl` command follows one pattern.**

```
kubectl  <verb>  <resource type>  [specific name]  [flags]
         ─────    ────────────     ───────────       ─────
         what to  what kind of     which one         options
         do        object          (optional)
```

Read it as a sentence. `kubectl get pods` = "get [me the] pods." `kubectl delete deployment web` = "delete [the] deployment [named] web." `kubectl describe node node-7` = "describe [the] node [named] node-7." Once you see the structure, you can build commands you've never run by **mixing a verb you know with a resource you know** — the grid composes.

### The verbs (what to do)

These are the actions — and notice how cleanly they map onto Lesson 1's "CRUD on objects" and the HTTP methods from §1:

| Verb | What it does | Underlying operation |
|------|--------------|---------------------|
| **get** | List objects (brief). | HTTP GET |
| **describe** | Show one object in rich, human-readable detail + recent events. | GET (several, assembled) |
| **create** | Make a new object (errors if it exists). | POST |
| **apply** | Create-or-update to match a file (the declarative workhorse, §5). | PATCH/POST |
| **edit** | Open an object in your editor and save changes back. | GET then PATCH |
| **delete** | Remove an object. | DELETE |
| **scale** | Change a replica count. | PATCH |
| **logs** | Read a container's logs (Lesson 9). | GET (a special subresource) |
| **exec** | Run a command inside a container (Lesson 9). | a streaming connection |
| **rollout** | Manage/inspect a Deployment's rollout (Lesson 4). | several |
| **explain** | Show the *schema docs* for a resource type (§3). | GET (from the API's own docs) |

### The resources (what kind of object)

These are the *kinds* of objects from Lesson 1 and the lessons ahead: `pods`, `deployments`, `replicasets`, `services` (short: `svc`), `nodes`, `namespaces` (`ns`), `configmaps` (`cm`), `secrets`, `persistentvolumeclaims` (`pvc`), and dozens more. Most have **short aliases** (`po`, `deploy`, `svc`, `ns`) to save typing.

You never have to memorize the full list, because `kubectl` will tell you it:

```
kubectl api-resources
```

prints *every* resource type the cluster knows about, its short name, whether it's namespaced (Lesson 5), and its API group. This command is your live, cluster-accurate menu — including custom resource types someone added. **When you don't know what you can act on, ask the cluster, don't guess.**

### Why the grammar is the whole point

Memorizing commands is fragile; understanding the grammar is durable. If you know the verb `describe` and you learn there's a resource called `pvc`, you *already* know `kubectl describe pvc <name>` works — you didn't have to be taught it. The grid (≈15 common verbs × dozens of resources) gives you hundreds of valid commands from a handful of memorized pieces. **Learn the axes, not the cells.**

---

## 3. The three "read" verbs that make you dangerous (in a good way)

Before you ever *change* a cluster, you should be able to *explore* any cluster fluently and safely. Three read-only verbs do almost all of this, and mastering them early pays off in every later lesson.

### `get` — the list view, with dials

`kubectl get <resource>` lists objects. Plain, it's a terse table. But its output flags turn it into a precision instrument:

- **`-o wide`** — more columns (e.g. which node a Pod is on, its IP). Your everyday "tell me more."
- **`-o yaml`** — the *complete* object as stored in etcd, every field, including the `status` the system filled in. This is how you see *exactly* what the API server holds. (`-o json` for the same in JSON.)
- **`-o jsonpath=...`** — extract one specific field, for scripting (e.g. pull out every Pod's image). The surgical option.
- **`--show-labels`** — show each object's labels (which matter enormously — §6).
- **`-w`** (watch) — *don't return; stream changes live.* Run `kubectl get pods -w`, then in another terminal delete a Pod, and you'll *see* the reconciliation loop from Lesson 1 replace it in real time. This is the single best way to feel the control loop working.
- **`-A` / `--all-namespaces`** — across every namespace, not just your current one (Lesson 5).

### `describe` — the human-readable dossier (and events)

`kubectl describe <resource> <name>` assembles a rich, readable report on one object: its spec, its current status, and — critically — the **recent events** attached to it. Events are the system narrating, in plain English, what it tried and what happened: `Scheduled`, `Pulling image`, `Started`, or failures like `FailedScheduling`, `ImagePullBackOff`, `CrashLoopBackOff`. **For a huge fraction of "why isn't this working?" problems, `kubectl describe` is the answer before you ever look at logs** — which is exactly why Lesson 9 (debugging) leans on it so heavily. Get in the habit now: when something's off, `describe` it and read the events at the bottom.

### `explain` — the built-in, offline schema manual

This one is underused and quietly transformative. `kubectl explain <resource>` prints the *documentation for every field of that object type*, straight from the API server's own schema. You can drill down with dots:

```
kubectl explain pod.spec.containers.resources
```

tells you precisely what fields exist under a container's `resources` (the requests/limits you'll meet in Lesson 8) and what each means. **This is your reference manual, built into the tool, always matching the exact cluster version you're talking to** — no browser, no guessing whether a docs page matches your version. When you're authoring YAML and unsure what a field is called or whether it exists, `explain` is the authoritative answer.

Between these three — `get` to list, `describe` to dig into one object and read its events, `explain` to learn the schema — you can walk into *any* cluster and understand it without changing a thing. That's the safe foundation to build the dangerous (state-changing) skills on.

---

## 4. kubeconfig: the file that defines what you're pointed at and who you are

Now the second half of the lesson — "which TV." When you run a `kubectl` command, how does it know *which* API server to call and *what credentials* to send? The answer is a single configuration file, the **kubeconfig**, by default at `~/.kube/config`. (You can point at a different one with the `KUBECONFIG` environment variable or `--kubeconfig`.) Understanding its structure is non-negotiable, because this file is the thing standing between "I ran a harmless command" and "I ran a command against production."

A kubeconfig contains **three lists plus one pointer.** Keep the remote-control analogy alive as we go:

### clusters — *which TV* (the API server)

A **cluster** entry describes an API server you might talk to: its URL (e.g. `https://prod.example.com:6443`) and the certificate authority used to trust it. This is the *TV itself* — a specific cluster's front door. A kubeconfig can list many: `dev`, `staging`, `prod`.

### users — *who you are* (your identity)

A **user** entry is a set of *credentials* — a client certificate, a bearer token, or (very common in the cloud) an **exec plugin** that fetches a short-lived token from AWS/GCP/Azure on demand. This is *your identity badge*. The API server uses it to authenticate you and then to authorize you via RBAC (Lesson 9). The same human might have a `dev-admin` identity and a `prod-readonly` identity. **Your identity decides what you're *allowed* to do once you reach a cluster.**

Note the clean split: a **cluster** is *where*, a **user** is *who*. They're independent lists precisely so you can combine them flexibly.

### contexts — a saved channel: (cluster + user + namespace)

A **context** ties it together. It's a named bundle of **three things**:

- a **cluster** (which TV),
- a **user** (who you are there),
- a default **namespace** (which "room" of that cluster — Lesson 5).

A context is a *saved channel on your remote*: "channel `prod`" means "the prod cluster, as my prod-readonly identity, defaulting to the `payments` namespace." Switching context is how you re-aim the whole remote in one move.

### current-context — *the channel you're on right now*

Finally, one pointer: **current-context** names which context is *active*. **This is the most important value in the file**, because it's the answer to "which TV am I pointed at *right now*?" Every command you run without an explicit override goes to whatever current-context says. The disasters all start with a wrong current-context and a confident keystroke.

Here's the shape of it, trimmed for clarity:

```yaml
clusters:                      # the TVs
- name: prod
  cluster: { server: https://prod.example.com:6443, certificate-authority-data: ... }
- name: dev
  cluster: { server: https://dev.example.com:6443,  certificate-authority-data: ... }

users:                         # the identities
- name: prod-readonly
  user: { exec: { ... fetch a short-lived cloud token ... } }
- name: dev-admin
  user: { client-certificate-data: ..., client-key-data: ... }

contexts:                      # the saved channels = cluster + user + namespace
- name: prod
  context: { cluster: prod, user: prod-readonly, namespace: payments }
- name: dev
  context: { cluster: dev,  user: dev-admin,     namespace: default }

current-context: dev           # ← the channel you're on RIGHT NOW
```

Read that and you can answer the only question that matters before a risky command: *am I about to act on prod, and as whom?*

---

## 5. Living in contexts safely: the habit that saves your career

You now know what a context *is*. Here's how you *live* in them without becoming a cautionary tale. The commands are few; the discipline is everything.

The relevant verbs all live under `kubectl config`:

- **`kubectl config get-contexts`** — list all contexts; the active one is marked with a `*`. *Your "what channels exist, and which am I on?"*
- **`kubectl config current-context`** — print just the active context's name. The fastest "where am I?" check. Make this reflex.
- **`kubectl config use-context <name>`** — switch the active context (change the channel). This is how you move from dev to prod.
- **`kubectl config set-context --current --namespace=<ns>`** — change the default namespace for the current context, so you stop typing `-n` everywhere (Lesson 5).

### The discipline

> **Before any command that *changes* state — `apply`, `delete`, `scale`, `edit`, `rollout` — confirm your context and namespace first.** Read-only commands (`get`, `describe`) are forgiving; mutating commands against the wrong cluster are not. The two seconds it takes to run `kubectl config current-context` is the cheapest insurance in the entire field.

Because humans are bad at remembering to check, the ecosystem builds *ambient awareness* so you can't help but see where you are:

- **`kubectx` and `kubens`** — tiny, beloved helper tools that make switching context and namespace fast and visible (`kubectx prod`, `kubens payments`), often with a picker. They turn "change the channel" into a one-word command.
- **Shell-prompt integrations** (`kube-ps1`, `starship`, and similar) — these *put your current context and namespace right in your terminal prompt*, so the wrong-cluster answer is *always on screen*. This is the highest-leverage safety habit there is: when "prod" is glowing in your prompt, you simply don't fat-finger a `delete` thinking you're in dev. Many teams also color the prompt **red for production**.
- **Explicit overrides for one-offs.** Any command takes `--context=<name>` and `--namespace=<ns>` to override the defaults for *that single command*, without changing your active context. When in doubt about a sensitive command, name the target explicitly rather than trusting the ambient state.

The throughline of Lesson 1 was "you're always one step removed, acting through records." The throughline here is its safety corollary: **because the same harmless-looking command can hit any cluster, knowing *which* cluster is a primary skill, not an afterthought.** We'll return to this in Lesson 9 as a core operational safeguard.

---

## 6. Imperative vs declarative: the two ways to drive the remote

Lesson 1 drew the big philosophical line between **imperative** ("do this now") and **declarative** ("here's what I want; reconcile to it"). `kubectl` lets you work in *both* modes, and knowing *when* to use each — and why production is almost always declarative — is a mark of fluency.

### Imperative: tell the cluster to do a thing now

Imperative commands act immediately and you don't keep a file around:

- `kubectl create deployment web --image=nginx` — make a Deployment right now.
- `kubectl run tmp --image=busybox -it -- sh` — start a throwaway Pod and drop into a shell.
- `kubectl scale deployment web --replicas=5` — change the count, now.
- `kubectl delete pod web-xyz` — remove this object, now.

**Strengths:** fast, immediate, great for *learning*, *experimenting*, and *one-off* operational fixes. This is how you'll poke at things while studying.

**Weakness:** there's no lasting record of *what you intended*. The cluster knows the current state, but six months later nobody can answer "why is it configured this way?" or review a change before it happens. Imperative changes are invisible to git, to code review, and to your teammates.

### Declarative: write down what you want, apply it, let reconciliation work

Declarative work means you keep the desired state in **YAML files**, usually in **git**, and push them with one verb:

```
kubectl apply -f web.yaml
kubectl apply -f ./manifests/        # a whole directory
```

`apply` is the create-or-update verb: it compares what's in the file to what's in the cluster and makes the cluster match — creating what's missing, updating what changed, leaving the rest. Run it twice with no file change and the second run does nothing (it's **idempotent**). This is Lesson 1's whole philosophy expressed as a command: **you edit the desired state; controllers reconcile reality toward it.**

**Strengths:** the YAML is a reviewable, version-controlled, auditable *source of truth*. You can diff it, code-review it, roll it back via git, and recreate the entire setup on a fresh cluster by re-applying. This is the foundation of **GitOps** (Lesson 9), where a tool watches git and applies changes automatically.

**Weakness:** slightly more ceremony for a quick experiment — which is exactly why you use imperative for *experiments* and declarative for *anything that lives.*

### The rule of thumb

> **Imperative to learn and to firefight; declarative to live.** Anything that should persist, be reviewed, or be reproducible belongs in a YAML file under version control, applied with `kubectl apply`. The throwaway and the exploratory can be imperative. A production change made imperatively (`kubectl edit` in anger) that *isn't* reflected back into git is **configuration drift** — the cluster no longer matches its source of truth, and the next `apply` will silently undo your fix. Lesson 9 returns to this; for now, just internalize the divide.

---

## 7. Generating YAML the easy way: `--dry-run` + imperative commands

There's a delightful bridge between the two modes that solves the most annoying part of declarative work — *writing the YAML by hand.* You don't have to. Let an imperative command write it for you:

```
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
```

`--dry-run=client` means "**don't actually do it** — just show me what you *would* send," and `-o yaml` means "as YAML." So this *generates* a correct, complete Deployment manifest without touching the cluster, which you then edit to taste and check into git. It's the fastest, least error-prone way to author manifests: start from machine-generated correctness, then tweak. This is the practical glue between §6's two modes — **use imperative ergonomics to produce declarative artifacts.**

Two flavors of dry-run, worth distinguishing:

- **`--dry-run=client`** — `kubectl` builds the object locally and shows it; the API server is never asked. Pure generation.
- **`--dry-run=server`** — the request *is* sent to the API server, which runs full validation and admission checks **but does not persist it.** This answers "*would* this actually be accepted by this cluster?" — catching schema errors and policy rejections before you commit. Use server dry-run to *validate* a real manifest; use client dry-run to *generate* a starting one.

---

## 8. Labels and selectors: how Kubernetes groups and finds things

We close with a cross-cutting idea that you'll meet in nearly every later lesson, and that the README flagged as foundational: **Kubernetes groups and finds objects by *labels*, not by names.**

A **label** is a key-value tag you attach to an object: `app=web`, `env=prod`, `tier=frontend`. Labels are arbitrary, you can attach many, and they carry no built-in meaning — *you* define the scheme. A **selector** is a query over labels: "all objects where `app=web`."

Why this matters so much: **names identify *one* object; labels identify *sets* of objects** — and Kubernetes is fundamentally about acting on sets that change over time. This is the mechanism behind huge chunks of the system you'll meet later:

- A **Deployment** knows which Pods are "its" Pods by a **label selector**, not by names — which is exactly how it can manage a constantly-churning set of Pods (Lesson 4). It doesn't track Pod names; it asks "which Pods match `app=web`?"
- A **Service** routes traffic to whichever Pods match its selector (Lesson 7) — so as Pods come and go, the set updates automatically.
- You operate in bulk with `-l`: `kubectl get pods -l app=web,env=prod` (Pods matching *both*), `kubectl delete pods -l app=web` (delete the whole set at once).
- **`--show-labels`** on a `get` reveals what's tagged on each object, and `kubectl label` adds or changes labels.

There's also a related query, **`--field-selector`**, for filtering on built-in fields rather than your labels (e.g. `--field-selector status.phase=Running` to list only running Pods). Labels are *your* tags; field selectors query *intrinsic* fields.

You don't need the full depth now — Lessons 4 and 7 will show labels doing real work. But plant the idea firmly: **in Kubernetes, the way you address "all the things like this" is a label selector, and that indirection is what lets controllers and Services track moving sets of objects.** It's the same theme as Lesson 1 — you describe *intent* ("everything tagged `app=web`"), and the system resolves it against current reality.

---

## 9. Recap — the mental model to carry forward

- **`kubectl` is just an API-server client.** Every command becomes an HTTP request against objects; `--v=8` shows you the literal calls. It can do nothing the API server's authorization won't allow. (§1)
- **The grammar is `kubectl <verb> <resource> [name] [flags]`.** Learn the verbs and resources as two axes and you can compose hundreds of commands; `kubectl api-resources` is the live menu. (§2)
- **Three read verbs explore any cluster safely:** `get` (with `-o wide/yaml/jsonpath`, `--show-labels`, `-w`), `describe` (rich detail + events — your first debugging stop), and `explain` (the built-in, version-accurate schema manual). (§3)
- **`kubeconfig` = clusters (which TV) + users (who you are) + contexts (saved channels) + current-context (the channel you're on now).** This file decides what you're aimed at and as whom. (§4)
- **Context discipline is a survival skill.** Confirm `current-context` (and namespace) before any mutating command; use prompt integrations and `kubectx`/`kubens` so the answer is always on screen. (§5)
- **Imperative to learn and firefight; declarative (`apply` YAML in git) to live.** Drift is when reality diverges from the source of truth. (§6)
- **Generate manifests with `--dry-run=client -o yaml`; validate them with `--dry-run=server`.** (§7)
- **Labels + selectors are how Kubernetes addresses *sets* of objects** — the mechanism under Deployments, Services, and bulk operations. (§8)

The remote control is now in your hand, and you know how to check which TV it's pointed at. Everything from here on is *pressing real buttons* — the objects those buttons act on.

---

## Glossary (quick reference)

| Term | Meaning |
|------|---------|
| **kubectl** | Command-line client for the API server; turns commands into HTTP requests on objects. |
| **`--v=8`** | Verbosity flag that prints the literal HTTP requests/responses kubectl makes. |
| **Verb / resource grammar** | `kubectl <verb> <resource> [name] [flags]` — compose commands from a small grid. |
| **api-resources** | `kubectl api-resources` — the cluster's live list of every resource type and its short name. |
| **get / describe / explain** | List objects / show one in rich detail + events / show a resource type's schema docs. |
| **`-o wide/yaml/json/jsonpath`** | Output formats for `get` — more columns / full object / machine formats / extract one field. |
| **`-w` (watch)** | Stream changes live instead of returning — see the control loop work in real time. |
| **Events** | The system's plain-English narration of what it tried on an object; shown by `describe`. |
| **kubeconfig** | The file (default `~/.kube/config`) defining clusters, users, contexts, and the current context. |
| **Cluster (kubeconfig entry)** | An API server you can target: its URL + trust cert. "Which TV." |
| **User (kubeconfig entry)** | A set of credentials = your identity to a cluster. "Who you are." |
| **Context** | A named (cluster + user + namespace) bundle. "A saved channel." |
| **current-context** | The active context — what unqualified commands target. "The channel you're on now." |
| **use-context / current-context** | Switch the active context / print it. |
| **kubectx / kubens** | Helper tools to switch context / namespace quickly and visibly. |
| **Imperative** | Commands that act immediately (`create`, `run`, `scale`, `delete`). For learning and one-offs. |
| **Declarative** | Describe desired state in YAML and `apply` it; the version-controlled, production way. |
| **apply** | Create-or-update objects to match a file; idempotent. The declarative workhorse. |
| **Drift** | When live cluster state diverges from the YAML source of truth (e.g. an imperative hotfix). |
| **`--dry-run=client` / `=server`** | Show what *would* be sent (generate YAML) / validate against the API server without persisting. |
| **Label** | An arbitrary key-value tag on an object (`app=web`). |
| **Selector** | A query over labels (`-l app=web`) — how Kubernetes addresses *sets* of objects. |
| **`--field-selector`** | Filter on intrinsic object fields (e.g. `status.phase=Running`) rather than labels. |

---

## Where to go next

You can now drive the remote and verify which TV it's pointed at. Next, start pressing buttons on real objects:

- **Lesson 3 — [Nodes & Scheduling](03-nodes-and-scheduling.md):** Use your new `kubectl` fluency to inspect the *machines* (`kubectl get nodes`, `kubectl describe node`) and understand how the scheduler decides where Pods land — and how to read a stuck `Pending` Pod from its events.
- **Build the muscle now, on a throwaway cluster.** On `kind`/`minikube`: run `kubectl --v=8 get pods` and find the URL; run `kubectl config get-contexts` and `current-context`; generate a manifest with `kubectl create deployment web --image=nginx --dry-run=client -o yaml`; then `kubectl get pods -w` in one terminal while you `kubectl delete` a Pod in another, and watch the loop heal it.
- **Set up your safety rails before you ever touch a shared cluster.** Install a prompt integration that shows your context and namespace (and colors prod red), and `kubectx`/`kubens`. Do this *now*, while the stakes are zero, so the habit is automatic when they aren't.

> `kubectl` was never a list to memorize — it's a sentence you compose (verb + resource), aimed at a target you always verify (context), in a mode you choose deliberately (imperative to explore, declarative to live). Hold that, and the rest of the course is just learning the nouns.
