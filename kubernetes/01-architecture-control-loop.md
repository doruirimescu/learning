# Lesson 1 — Architecture & the Control Loop: Who Actually Does What

> [The README](README.md) made one promise above all others: **Kubernetes is a control loop — you declare desired state, and the control plane continuously reconciles reality toward it.** Everything else in this course is a variation on that one idea. This lesson is where we earn it. Before a single `kubectl` command, before Pods or Services or autoscalers, we build an honest mental model of *what a cluster is made of* and *who does what inside it*. By the end you'll understand why Kubernetes feels less like "running a program" and more like "filing a request with a bureaucracy that never sleeps" — and why that design, strange as it seems at first, is exactly what makes it so resilient.
>
> This lesson assumes **no prior Kubernetes knowledge**. We build every concept from the ground up. It's long because it's foundational: every later lesson leans on the org chart and the loop you'll learn here.

---

## 0. First, the vocabulary: cluster, object, state

We're going to throw around words like "cluster," "control plane," "object," and "desired state." Let's pin them down so they mean something concrete.

A **container** is a packaged, isolated process: your application plus everything it needs to run (libraries, files), sealed into an image that runs the same anywhere. If you've used Docker, this is that. Kubernetes' job is to run lots of containers, across lots of machines, reliably. It is, at its simplest, **a system for running containers on a fleet of machines so you don't have to babysit them.**

A **cluster** is that fleet of machines plus the brain that coordinates them, all acting as one system. When you "use Kubernetes," you use a cluster. A cluster has two kinds of machines, and the entire lesson hinges on the difference:

- The **control plane** — the brain. It makes decisions and remembers everything. It does *not* run your applications (usually).
- The **worker nodes** (just "nodes") — the muscle. This is where your containers actually run.

A **node** is a single machine — a virtual machine in the cloud, or a physical server — that has joined the cluster and is willing to run containers.

An **object** (sometimes "resource") is a record in Kubernetes describing something you want or something that exists: a Deployment, a Pod, a Service, a Node. Crucially, **an object is just data** — a structured document, usually written as YAML, stored in the cluster's database. When you "create a Deployment," you are not starting a program; you are *writing down a record* that says "I would like this to exist." Hold that thought — it's the whole game.

**State** comes in two flavors that you must never confuse:

- **Desired state** — what you *want* the world to look like. "I want 3 copies of my web app running." You write this down.
- **Current state** (or *observed* / *actual* state) — what the world *actually* looks like right now. "There are 2 copies running; one crashed." Kubernetes observes this and writes it down too.

That's the whole starting vocabulary. Now the single most important idea in the lesson — the one that, once it clicks, makes everything else obvious.

---

## 1. The central surprise: you describe the destination, not the route

Most engineers arrive at Kubernetes with a lifetime of **imperative** habits. You `ssh` into a server. You type `systemctl start nginx`. You run a deploy script that copies files, restarts a process, checks it came up. Every command *does a thing, right now*, and *you* are responsible for the sequence: do this, then that, then check, then fix if it failed. You are the conductor, and the machine does exactly what you say, when you say it, and nothing more.

Kubernetes throws this model out. It is **declarative**, and the difference is not a detail — it's the entire personality of the system:

> **In Kubernetes, you do not tell the system *what to do*. You tell it *what you want to be true*, and the system figures out how to make it true — and how to keep it true, forever, without you.**

This is the same shift as the difference between giving someone turn-by-turn driving directions ("turn left, go 200 meters, turn right…") versus handing them a destination address and letting them navigate, re-route around traffic, and recover if they take a wrong turn. The imperative way, *you* own every step and every failure. The declarative way, you own the *goal*, and the system owns the steps and the recovery.

Concretely: you never tell Kubernetes "start a container on machine 7." You write down "I want 3 copies of this app to exist," hand that to the cluster, and walk away. Something inside the cluster reads your wish, notices reality doesn't match, and acts — starting containers, choosing machines, replacing them when they die — *continuously*, with no further input from you.

This inverts where your effort goes. In the imperative world, the hard part is *executing and babysitting* the steps. In the declarative world, the hard part is *describing what you want accurately* — and once you've done that, the babysitting is the system's problem. This is why people say Kubernetes is "self-healing." It's not magic. It's just that the system's only job is to keep closing the gap between what-you-want and what-is, over and over, forever.

If you remember only one thing from this lesson, remember this: **you declare desired state; the system reconciles reality toward it.** Everything below is the machinery that makes that sentence true.

---

## 2. The reconciliation loop: the thermostat at the heart of everything

So *how* does the system keep reality matching your wishes? Through a pattern so central it deserves its own name and its own picture: the **reconciliation loop** (also called the **control loop**, which is why this whole course is "control-loop-first").

> **The mental model: Kubernetes is a thermostat for your infrastructure.** You do not tell a thermostat "turn on the heater for 12 minutes." You set the *desired temperature* — say, 21°C. The thermostat then does one thing, forever: it **observes** the current temperature, **compares** it to the desired temperature, and **acts** to close the gap (heater on if too cold, off if too warm). Then it does it again. And again. Forever. You set the goal once; the loop holds it against a world that keeps drifting (someone opens a window, the sun sets, the room cools).

A reconciliation loop is exactly this, three steps repeating endlessly:

1. **Observe** the current state of the world.
2. **Compare** it to the desired state.
3. **Act** to reduce the difference — then go back to step 1.

The piece of software that runs this loop for a particular kind of object is called a **controller**. And here is the load-bearing insight for the entire course:

> **Almost everything in Kubernetes is a controller running this same loop over a different kind of object.** A Deployment controller loops to keep the right number of app copies running. A Node controller loops to notice dead machines. A Job controller loops to run a task to completion. An autoscaler (Lesson 8) loops to keep CPU usage near a target. *They are all thermostats.* Learn the loop once, and you understand the behavior of the entire system.

This single idea explains the most surprising behaviors you'll meet:

- **Self-healing.** A container crashes (current state: 2 running, desired: 3). The controller observes the gap on its next loop and starts a replacement. No alert paged a human; no script ran. The loop simply did its job.
- **"Kubernetes fights me."** Suppose you manually delete a Pod that a Deployment wants alive. The controller observes "desired: 3, actual: 2" and *immediately* creates another. To you it looks like the Pod "won't die." It's not stubbornness — you edited *reality*, but you didn't edit the *desired state*, so the loop dutifully restored the gap. **To actually remove it, you must change what you *want* (edit the Deployment), not what *is* (kill the Pod).** This one realization saves new users weeks of confusion.
- **Eventual consistency.** Change takes a moment because nothing happens "instantly on command" — it happens on the *next turn of the loop*. Usually that's sub-second, but the model is "converge toward the goal," not "execute now."

Keep the thermostat in your head. We'll now meet the actual machines and processes that run these loops — but every one of them is, underneath, this same observe-compare-act cycle.

---

## 3. One source of truth, one gatekeeper, and kubectl is just a visitor

Before the org chart, one more structural idea that makes everything click — *where the state lives* and *how anything touches it*.

> **There is exactly one authoritative record of the cluster's state, and exactly one door to reach it.** The record is a database called **etcd**. The door is a process called the **API server**. *Nothing* — not you, not a controller, not a worker machine — ever touches etcd directly. Everything goes through the API server. Always.

Picture a government records office. There is one master filing cabinet (etcd) holding every official document — every desired state you've declared, every current state the system has observed. There is one clerk at the front desk (the API server). Citizens (you, via `kubectl`), inspectors (controllers), and field offices (the worker nodes) all line up at the *same desk*. To read a record, you ask the clerk. To change one, you hand the clerk a revised form; the clerk checks your ID (authentication), checks you're allowed (authorization), checks the form is filled out correctly (validation), and *then* files it in the cabinet. Only the clerk writes to the cabinet.

This has profound consequences, and they're worth stating plainly because they reshape how you think about *every* command you'll ever run:

- **`kubectl` is not special.** It is just one visitor at the desk — a command-line program that turns your commands into requests to the API server. When you run `kubectl get pods`, it sends an HTTP GET to the API server, which reads etcd and hands back the records. When you run `kubectl apply`, it sends the new desired-state document for the clerk to file. That's *all* `kubectl` is: a convenient client for the API server. (Lesson 2 is entirely about this client.)
- **You never talk to your containers or your machines directly.** You only ever read and write *records* at the front desk. The effects — containers starting, machines being chosen — ripple out *afterward*, as controllers and nodes read those records and react. You are always one step removed from the actual work. This is the single biggest mental adjustment from the `ssh`-and-do-it world, and it's why debugging in Lesson 9 is "inspect the records," not "log into the box."
- **Every change is mediated, audited, and validated.** Because everything funnels through one gatekeeper, Kubernetes can authenticate, authorize (RBAC, Lesson 9), and validate *every* change in one place. The single door is also the single checkpoint.

So the shape of the whole system is: **state lives in etcd; the API server is its only gateway; and everyone — humans, controllers, machines — coordinates by reading and writing through that gateway.** Now we can meet "everyone."

---

## 4. The org chart: control plane (head office) vs nodes (the factory floor)

Here is the mental model for the cluster as a whole:

> **A cluster is a company. The control plane is *head office* — it decides what should happen and keeps all the records. The worker nodes are the *factory floor* — they do the actual physical work of running your containers. You, with `kubectl`, are an external client emailing head office.** You never walk onto the factory floor and start a machine with your own hands. You send a request to head office; head office records the decision; and the floor managers read those decisions and act. You are *always* one step removed from the work — talking to a system of records, not to the machines doing the labor.

Two halves, with sharply different jobs:

| | **Control plane** ("head office") | **Worker nodes** ("factory floor") |
|---|---|---|
| **Job** | Decide and remember. | Execute. |
| **Runs** | The cluster's brain components (below). | Your application containers. |
| **Knows** | The entire cluster's desired + current state. | Only about the Pods assigned to *itself*. |
| **If it's down** | You can't *make changes*; existing apps often keep running. | The apps on *that* node stop; the control plane reschedules them elsewhere. |

That last row is worth pausing on, because it reveals the design's resilience. The control plane is the *coordinator*, not the *doer*. If head office goes dark for a minute, the factory floor keeps running the orders it already received — your apps stay up; you just can't issue new instructions. And if one factory burns down (a node dies), head office notices the gap and re-assigns that work to other factories. The separation of "deciding" from "doing" is what lets each survive the other's failure.

In production, the control plane usually runs on its own dedicated machines (often replicated 3 or 5 times for high availability — if the brain matters, you don't run just one). On a tiny local cluster (`minikube`, `kind`), the control plane and a worker might share one machine, but the *roles* are still distinct. Always think in terms of the roles, not the machine count.

Now let's open up each half and meet the individual workers.

---

## 5. Inside head office: the control-plane components

The control plane is not one program — it's a handful of specialized components, each with one clear job. The cleanest way to learn them is as **job titles in the company.** Here they are, each as a person at head office.

### etcd — the company's single filing cabinet

**etcd** is a database — specifically a *consistent, distributed key-value store*. It is the **single source of truth** for the entire cluster. Every object — every Deployment you've declared, every Pod's current status, every Node's health — lives here, and *only* here.

- **Why "consistent" matters:** when the cabinet is replicated across several machines for safety, etcd guarantees they agree before confirming a write (using a consensus algorithm called Raft — you don't need the details, just the property). There is never a moment where two readers see two different "truths." This is the bedrock the whole system's correctness rests on.
- **The stakes:** etcd *is* the cluster's memory. Lose etcd with no backup, and you've lost the cluster's entire knowledge of what should exist — even if the apps are still running, the brain has amnesia. This is why "back up etcd" is the first commandment of cluster operations.
- **You rarely touch it directly.** You interact with its contents *through the API server*, never by querying etcd yourself. It sits behind the clerk, as established in §3.

### kube-apiserver — the front desk and the only door

The **API server** (`kube-apiserver`) is the gatekeeper from §3 made concrete. It is the front-end to the entire control plane and the *only* component that reads from and writes to etcd.

Its job, on every single request:

1. **Authenticate** — who are you? (a certificate, a token, a cloud identity — Lesson 2's `kubeconfig` provides this)
2. **Authorize** — are you allowed to do this? (RBAC — Lesson 9)
3. **Validate / admit** — is this a well-formed, acceptable object? (schema checks, and pluggable "admission" policies)
4. **Persist** — write the result to etcd.

Everything talks to the API server and *only* the API server: `kubectl`, every controller, every node's kubelet. It is the hub of a wheel; all spokes connect through it, and no two spokes connect to each other. This hub-and-spoke design is why the system is so extensible — add a new kind of controller and it just talks to the same front desk as everyone else.

### kube-scheduler — the assignment desk

When you ask for a Pod, *something* has to decide *which node* it should run on. That's the **scheduler** (`kube-scheduler`), and it has exactly one job:

> The scheduler watches for Pods that have been created but **not yet assigned a node** ("unscheduled" Pods). For each one, it picks the best node and writes that choice down. **That's it.** It decides *where*; it does not start anything.

This is a crucial and commonly-misunderstood boundary. The scheduler is a *planner*, like a restaurant host deciding which table a party should sit at. It picks the table and notes it on the reservation — but it doesn't cook the food or seat the guests. It chooses by filtering nodes that *can't* fit the Pod (not enough CPU/memory, a constraint isn't met) and then scoring the ones that can, to pick the best. (Lesson 3 is devoted to how this works and how you steer it.) Once it decides, it does one tiny thing: it updates the Pod's record to say "you belong on node-7." Then it moves on. *Something else* will notice that assignment and actually start the containers — read on.

### kube-controller-manager — the building full of thermostats

Remember §2: almost everything is a controller running the observe-compare-act loop. The **controller manager** (`kube-controller-manager`) is a single process that *hosts* most of the built-in controllers — bundled together for efficiency, but conceptually a whole department of independent thermostats, each watching its own kind of object. Among them:

- The **Deployment controller** — keeps the desired app rollout reconciled (Lesson 4).
- The **ReplicaSet controller** — keeps the desired *number* of identical Pods existing.
- The **Node controller** — watches node health and reacts when one stops reporting in.
- The **Job controller** — drives run-to-completion tasks.
- …and many more.

Each one is a loop: watch the API server for "my kind of object," compare desired to actual, and make API calls to close the gap. When you internalize that this is *one building full of identical-shaped loops*, the behavior of the whole system becomes predictable rather than mysterious.

### cloud-controller-manager — the liaison to the cloud

When your cluster runs in a cloud (AWS, GCP, Azure), some reconciliation requires talking to the *cloud provider's* API — provisioning a load balancer (Lesson 7), attaching a disk (Lesson 6), or registering new machines (Lesson 8). The **cloud-controller-manager** is the department that speaks the cloud's language, kept separate so the core of Kubernetes stays cloud-agnostic. On a local or bare-metal cluster, it's simply absent. Mention it now so the load balancers and disks in later lessons have a name attached to "who actually calls the cloud."

---

## 6. On the factory floor: the node components

Head office decides and records. But records don't run containers. On *every* worker node sit a few components whose job is to turn those records into running reality. Three roles:

### kubelet — the floor manager on each node

The **kubelet** is the single most important node component and the bridge between the declarative world and actual running containers. Picture it as the **manager of one factory**, and its loop (yes, another loop) is:

> Watch the API server for "which Pods are assigned to *my* node?" For each one, make it real — tell the container runtime to pull the images and start the containers — and then continuously report each Pod's actual status back up to the API server.

Notice how this dovetails with the scheduler from §5. The scheduler *wrote down* "this Pod belongs on node-7." The kubelet on node-7 is watching for exactly that, sees the assignment, and *acts* on it — pulling images, starting containers, wiring up storage and config. **The scheduler decides; the kubelet does.** Neither does the other's job, and that clean split is why each can be simple and reliable.

The kubelet also runs the **health probes** (Lesson 4) — periodically checking "is this container alive? is it ready for traffic?" — and reports the answers upward. It is the eyes and hands of head office on each individual machine.

### The container runtime — the machine operator

The kubelet doesn't actually know how to pull an image or start a container itself; it delegates that to a **container runtime** — software like **containerd** or **CRI-O**. The runtime is the operator that physically works the machines: it pulls the image from a registry, sets up the container's isolation, and runs the process. The kubelet talks to it through a standard interface (the **CRI**, Container Runtime Interface), which is why Kubernetes can run on different runtimes interchangeably. You'll rarely interact with the runtime directly; just know it sits one layer below the kubelet, doing the literal work of running containers.

### kube-proxy — the floor's network receptionist

Your containers need to talk to each other over the network, and they need to be reachable by a *stable* address even as individual Pods come and go. **kube-proxy** runs on each node and programs the node's networking rules so that traffic aimed at a **Service** (a stable virtual address — Lesson 7) gets routed to one of the live Pods behind it. Think of it as the receptionist who knows which employees are currently in and forwards each call to a working one. We'll give Services and kube-proxy a full lesson later; for now, just place it on the org chart as "the per-node component that makes cluster networking work."

---

## 7. The whole machine in motion: tracing `kubectl apply` end to end

You now have every character. Let's watch them work together on the single most important story in this course — what *actually* happens when you deploy an app. This trace is the spine of the entire curriculum; every later lesson zooms into one of its hops. Read it slowly.

You write a small YAML file describing a **Deployment** — "I want 3 replicas of my web container" — and run `kubectl apply -f web.yaml`. Here is the full journey, narrated by component:

1. **`kubectl` → API server.** `kubectl` (Lesson 2) reads your YAML, figures out which cluster and identity you're using (from your `kubeconfig`), and sends the Deployment object to the API server as an HTTP request. *You are now done.* You walked away from the front desk; everything below happens without you.

2. **API server → etcd.** The API server **authenticates** you, **authorizes** the action (are you allowed to create Deployments here?), **validates** the object, and **writes** it to etcd. The desired state — "3 replicas wanted" — is now officially recorded. At this instant, *zero containers are running.* All that exists is a wish in the filing cabinet.

3. **Deployment controller wakes up.** The Deployment controller (in the controller-manager) is *watching* the API server for Deployment objects. It sees yours, runs its loop — "desired: a rollout of 3 replicas; actual: nothing exists yet" — and acts by creating a **ReplicaSet** object (the count-keeper, Lesson 4). It doesn't make Pods itself; it delegates, manager-style, to a more specialized controller.

4. **ReplicaSet controller wakes up.** The ReplicaSet controller is watching for ReplicaSets. It sees the new one, runs *its* loop — "desired: 3 Pods; actual: 0 Pods" — and acts by creating **3 Pod objects** in the API server. Note: these Pods are records only. They have no node yet, and nothing is running. They sit in etcd in a `Pending` state, homeless.

5. **Scheduler wakes up.** The scheduler is watching for Pods with no node assigned. It sees the 3 homeless Pods. For each, it **filters** nodes that can't fit it and **scores** the rest (Lesson 3), picks the best node, and writes the assignment into the Pod's record: "you go on node-A." Still nothing is *running* — the scheduler only chose addresses.

6. **kubelet wakes up.** On each chosen node, the **kubelet** is watching for Pods assigned to it. node-A's kubelet sees "a Pod is now assigned to me," and finally — *finally* — real work happens: it tells the **container runtime** to pull the image and start the container(s). It mounts any config or storage the Pod needs (Lesson 6). The container starts running.

7. **Status flows back up.** The kubelet reports the Pod's actual status — "Running, healthy" — back to the API server, which records it in etcd. As each Pod becomes **Ready** (its readiness probe passes, Lesson 4), **kube-proxy** and the Service machinery (Lesson 7) start routing traffic to it.

8. **`kubectl get` shows the result.** You run `kubectl get pods` and see 3 Pods `Running`. What you're actually doing is asking the front desk to read the *current state* records that the kubelets reported. You're reading the cabinet, not the machines.

9. **And then it never stops.** Every controller in that chain keeps looping, forever. Kill a Pod and the ReplicaSet controller notices the gap and makes a new one (which the scheduler places and a kubelet starts — the chain re-runs). A whole node dies, and the Node controller notices it stopped reporting, marks its Pods as gone, and the ReplicaSet controller replaces them elsewhere. **You declared "3 replicas" once; the system holds that truth against a hostile, failing world indefinitely.**

Look back at this trace and notice the recurring shape: at *every* hop, a component **watches** the API server for its kind of object, **compares** desired to actual, and **acts** by writing a new object — which becomes the trigger for the *next* component down the line. The whole system is a **chain of reconciliation loops, coordinating only through the shared records at the front desk.** No component calls another directly. No central coordinator orchestrates the sequence. Each just watches, reacts, and writes — and out of that emerges a self-healing deployment. *That* is the architecture of Kubernetes.

---

## 8. Why this design is so resilient (and why it feels "eventually consistent")

The strangeness of Kubernetes — the indirection, the never talking to machines directly, the slight delays — is not accidental. It's the price of two properties that matter enormously at scale.

**It's resilient because the components are decoupled.** No component depends on any other being alive *at that instant*; they coordinate only through state in etcd. The scheduler can restart and pick up where it left off by re-reading the records. A controller can crash and, on restart, simply observe current-vs-desired again and continue — because its job is stateless: "look at the world, fix the gap." There is no fragile chain of direct calls where one failure cascades. Each thermostat just keeps trying to close its gap whenever it's running. This is called a **level-triggered** design (act on the *current* gap) rather than **edge-triggered** (act on a one-time event you might miss) — and it's why Kubernetes recovers so gracefully from its own components failing.

**It's eventually consistent because nothing happens "on command."** Everything happens on the *next turn of some loop*. Usually that's milliseconds to seconds, fast enough to feel instant. But the *model* is convergence, not immediate execution. This is why a freshly-applied change might take a moment to fully appear, and why you think in terms of "the cluster is converging toward what I asked for" rather than "my command ran." Once you expect this, the small delays stop being surprising and become a sign the system is working exactly as designed.

The trade-off, stated honestly: you give up the *directness and immediacy* of imperative control, and in return you get *self-healing, scalability, and resilience* you could never hand-script. For a single server, Kubernetes is absurd overhead. For a fleet that must stay up while machines fail underneath it, this design is why the whole thing exists.

---

## 9. Three things people get wrong at this stage

A few misconceptions are worth killing now, while the model is fresh — each is a direct consequence of *not* fully trusting the control-loop picture.

- **"I'll just SSH in and fix the container."** You can, but the loop will often undo you, because you changed *reality* without changing *desired state* (§2). The Kubernetes-native fix is always to change the *record* (the desired state) and let reconciliation carry it out. Reaching for SSH is a sign you've slipped back into imperative thinking.
- **"The control plane runs my apps."** No — the *worker nodes* run your apps; the control plane only decides and records (§4). (You *can* run workloads on control-plane nodes in tiny clusters, but the roles are still distinct, and in production they're kept apart on purpose.)
- **"`kubectl` does things to my containers."** `kubectl` only does CRUD on *objects* at the API server (§3). Every visible effect on your containers is a *downstream consequence* of a controller or kubelet reacting to those object changes. Internalizing this is exactly what makes Lesson 2 (`kubectl`) and Lesson 9 (debugging) feel easy instead of arbitrary.

---

## 10. Recap — the mental model to carry forward

You've built the foundation the rest of the course stands on. The load-bearing ideas, compressed:

- **Declarative, not imperative.** You describe *what you want to be true* (desired state) as *objects*; you don't issue step-by-step commands. (§1)
- **The reconciliation loop.** A **controller** endlessly **observes → compares → acts** to close the gap between desired and current state. Almost everything in Kubernetes is a controller running this loop — the thermostat. (§2)
- **One source of truth, one door.** All state lives in **etcd**; the **API server** is the only thing that reads/writes it; everyone else — including `kubectl` — is just a client at that one door. (§3)
- **Head office vs the factory floor.** The **control plane** decides and records (etcd, API server, scheduler, controller-manager); the **worker nodes** execute (kubelet, container runtime, kube-proxy). You're an external client emailing head office; you never touch the floor directly. (§4–6)
- **The declare-to-running chain.** `kubectl apply` → API server → etcd → Deployment controller → ReplicaSet controller → scheduler → kubelet → running container → status back up — a chain of loops, each watching the shared records and triggering the next. (§7)
- **Resilient and eventually consistent** because components are decoupled and level-triggered, coordinating only through state. (§8)

Hold the thermostat and the org chart in your head together, and you can predict the behavior of nearly any Kubernetes situation you'll meet — which is exactly what makes the next lessons feel like *details of a system you already understand*, rather than a pile of commands to memorize.

---

## Glossary (quick reference)

| Term | Meaning |
|------|---------|
| **Cluster** | A fleet of machines plus the brain coordinating them, acting as one system to run containers. |
| **Control plane** | The "head office" — decides and records. Components: etcd, API server, scheduler, controller-manager, cloud-controller-manager. |
| **Worker node (node)** | A "factory floor" machine that actually runs your containers. Components: kubelet, runtime, kube-proxy. |
| **Object / resource** | A structured record (usually YAML) describing something desired or observed; stored in etcd. Just data. |
| **Desired state** | What you want to be true ("3 replicas"). You declare it. |
| **Current / observed state** | What is actually true right now. The system observes and records it. |
| **Declarative** | You describe the goal, not the steps. The system figures out and maintains the *how*. |
| **Reconciliation / control loop** | Observe → compare → act, repeating forever, to close the gap between desired and current state. |
| **Controller** | Software running a reconciliation loop over one kind of object. The "thermostat." |
| **Level-triggered** | Acting on the *current gap* (robust to missed events), as opposed to edge-triggered (acting on one-time events). |
| **etcd** | The single consistent key-value store holding all cluster state; the one source of truth. |
| **kube-apiserver** | The only gateway to etcd; authenticates, authorizes, validates, and persists every change. |
| **kube-scheduler** | Decides *which node* an unscheduled Pod runs on. Plans placement; starts nothing. |
| **kube-controller-manager** | Hosts the built-in controllers (Deployment, ReplicaSet, Node, Job, …). |
| **cloud-controller-manager** | Talks to the cloud provider's API (load balancers, disks, nodes). |
| **kubelet** | Per-node agent that turns assigned Pods into running containers and reports their status. |
| **Container runtime** | containerd/CRI-O — pulls images and runs containers, on the kubelet's behalf. |
| **kube-proxy** | Per-node component that programs networking so Service addresses route to live Pods. |
| **kubectl** | The command-line client for the API server; does CRUD on objects. (Lesson 2.) |
| **Pod** | The smallest deployable unit — one or more containers run together. (Lesson 4.) |
| **Deployment / ReplicaSet** | Controllers managing rolling updates / a fixed count of identical Pods. (Lesson 4.) |
| **Eventually consistent** | The cluster *converges* toward desired state over loop iterations, rather than executing instantly on command. |

---

## Where to go next

You now have the org chart and the loop. The natural next steps:

- **Lesson 2 — [kubectl & kubeconfig](02-kubectl-and-kubeconfig.md):** Now that you know `kubectl` is "just a client at the API-server front desk," learn to wield it fluently and *safely* — the verb-noun grammar, and the all-important **contexts** that decide *which cluster* you're aimed at.
- **See the chain with your own eyes.** On any cluster, run `kubectl get pods -A` to see the control-plane components themselves running as Pods in the `kube-system` namespace — etcd, the API server, the scheduler, the controller-manager are all *right there*. Then run `kubectl --v=8 get pods` and watch `kubectl` make the literal HTTP calls to the API server from §3.
- **Watch a loop heal in real time.** Once you've done Lesson 4, create a Deployment, run `kubectl get pods -w` to watch, then delete one of its Pods — and *see* the ReplicaSet controller create a replacement within a second. That single experiment makes the thermostat real in a way no diagram can.

> The whole rest of this course is "details of a system you already understand." Every object is something you declare; every behavior is a controller closing a gap; every command is a visit to the front desk. Hold the thermostat and the org chart, and Kubernetes stops being a magic box.
