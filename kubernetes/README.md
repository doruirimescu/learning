# Kubernetes for Engineers — A Lesson Plan (kubectl-first)

> A concept-first roadmap for an engineer who wants to *genuinely understand* Kubernetes through the tool you actually touch it with: **`kubectl`**. This is the *plan*, not the lessons themselves: an ordered arc of modules, each one explaining **what you need to learn, why it matters, and the mental model to hold** — so that when you type a `kubectl` command, you know exactly what you're talking to and what will happen.
>
> The guiding principle, borrowed from the rest of this repo: lead with *why* and with mental models, not with command dumps. In Kubernetes especially, **the commands are trivial once the model is right, and impossible to remember while the model is wrong.** Memorizing `kubectl` flags without the architecture is how people stay scared of Kubernetes for years.

---

## 0. The one idea everything hangs on: Kubernetes is a control loop

Before any module, internalize the single idea that makes Kubernetes *different* from the way you probably think about running software. The instinct from a lifetime of `ssh`, `systemctl start`, and "run this script on that box" is **imperative**: you issue commands, and each command *does a thing right now*. Kubernetes is the opposite. It is **declarative**: you describe the **desired state** of the world ("I want 3 replicas of this container, reachable on this address, using this config"), you hand that description to the cluster, and then a set of background processes works *continuously and forever* to make reality match your description — and to *keep* it matching as things break.

> **The mental model: Kubernetes is a thermostat for your infrastructure.** You don't tell a thermostat "turn on the heater for 12 minutes." You set the *desired temperature* (the **desired state**), and the thermostat continuously compares it to the *actual temperature* (the **current state**) and acts to close the gap — turning the heater on, off, on again, forever, with no further input from you. A Kubernetes object (a Deployment, say) is a desired temperature. A **controller** is the thermostat's logic: a loop that observes current state, compares it to desired, and takes action to reconcile the difference. This **reconciliation loop** is the beating heart of the entire system. Almost everything you'll learn is some specialized controller running this same loop over a different kind of object.

This reframes how you read every later module. When a node dies and your Pods reappear elsewhere, no human and no script did that — a controller noticed "desired: 3 running, actual: 2 running" and acted. When you "deploy a new version," you are not running a deploy script; you are *editing the desired state*, and a controller rolls reality toward it. **Kubernetes never stops reconciling.** This is why it's resilient, and also why it sometimes "fights you" — if you manually kill a Pod that a controller wants alive, the controller wins, instantly and forever, because that's its entire job.

A second idea, equally important: **everything is an object in a single source of truth, and `kubectl` is just a client that reads and writes those objects.** There is one authoritative store of desired + current state (the cluster's database, **etcd**), fronted by one gatekeeper (the **API server**). You never talk to nodes, controllers, or containers directly. You submit object changes to the API server; controllers watch the API server; the API server is the *only* thing that writes to etcd. So `kubectl get`, `kubectl apply`, `kubectl delete` are not "doing things to your apps" — they are **CRUD operations on objects in a database**, and the consequences ripple out from there through controllers. Hold both ideas: **declare desired state as objects; let control loops reconcile reality — and `kubectl` is your window onto that conversation.**

---

## How to read this plan — the learning arc

The modules are ordered as a *dependency chain*: you understand the control loop and the cluster's parts, then the client (`kubectl`) you use to drive it, then the things you schedule (nodes, workloads), then how you organize and connect them (namespaces, config, networking), then how they scale, then how you debug and operate them. You *can* skip around, but each module assumes the ones before it.

| # | Module | One-line purpose |
|---|--------|------------------|
| 1 | **Architecture & the Control Loop** | The reconciliation loop, the cluster's anatomy — control plane vs nodes — and *who actually does what*. |
| 2 | **kubectl & kubeconfig** | The universal client: how it talks to the API server, contexts/clusters/users, the verb-noun grammar, imperative vs declarative. |
| 3 | **Nodes & Scheduling** | What a node is, how the scheduler places Pods, and how you steer or evict that placement (labels, taints, drain). |
| 4 | **Workloads** | The objects you actually run — Pods up through Deployments, StatefulSets, DaemonSets, Jobs — and which to reach for. |
| 5 | **Namespaces** | What they are, **where to use them and where not**, and how they scope quotas, names, and permissions. |
| 6 | **Config, Secrets & Storage** | Separating config and data from code: ConfigMaps, Secrets, volumes, and persistent storage. |
| 7 | **Services & Networking** | How Pods find and reach each other and the outside world: Services, DNS, kube-proxy, Ingress. |
| 8 | **Autoscaling** | The control loop applied to scale: requests/limits, metrics, HPA, VPA, and the Cluster Autoscaler. |
| 9 | **Debugging & Operating with kubectl** | Seeing inside a running cluster and changing it safely: logs, exec, events, rollouts/rollbacks, RBAC. |

A reasonable study order is exactly 1 → 9. If you're impatient to type commands, you can skim Module 2 (`kubectl`) early and keep it open as a reference while you learn — but don't try to *understand* `kubectl` before Module 1, or you'll be memorizing verbs without knowing what they act on. The whole point of this course is that **`kubectl` only makes sense as "a client that does CRUD on objects the control plane reconciles."**

---

## Module 1 — Architecture & the Control Loop: who actually does what

> 📖 **Written:** [`01-architecture-control-loop.md`](01-architecture-control-loop.md) — the full lesson.

**Why this is first.** Every `kubectl` command and every Kubernetes behavior is a consequence of *which component is responsible for what*. If you don't know that the scheduler places Pods but never starts them, that the kubelet starts them but never decides where, that etcd is the only source of truth and the API server is its only gatekeeper — then Kubernetes is a magic box and debugging is guesswork. This module replaces the magic box with a clear org chart.

> **The mental model: a cluster is a company. The control plane is head office (it decides *what* should happen and records it); the nodes are the factory floor (they do the actual work). `kubectl` is you, emailing head office.** You never walk onto the factory floor and start a machine yourself — you send a request to head office, head office records the decision, and the floor managers (one per factory) read those decisions and act. The genius and the frustration of Kubernetes both come from this separation: you are *always* one step removed from the work, talking to a system of records, not to the machines.

Concepts to master:

- **The reconciliation loop, concretely.** Observe current state → compare to desired state → act to close the gap → repeat forever. Every controller is this loop. Internalize it now and the rest of the course is variations on a theme. *Payoff:* you stop being surprised when Kubernetes "undoes" your manual changes.
- **The cluster: control plane vs worker nodes.** The two halves of every cluster and why they're separated. *Payoff:* the foundation for reasoning about failure ("if the control plane is down, do my running apps stop?" — usually no, and you'll understand exactly why).
- **The control-plane components, each as a job title:**
  - **`etcd`** — the company's single filing cabinet. A consistent key-value store holding *all* cluster state (desired and observed). The one source of truth. If it's lost, the cluster's memory is lost.
  - **`kube-apiserver`** — the front desk and the *only* door to etcd. Every read and write — from `kubectl`, from controllers, from kubelets — goes through it. It authenticates, authorizes, validates, and persists. *Everything* talks to the API server and nothing else talks to etcd.
  - **`kube-scheduler`** — the assignment desk. Watches for Pods that have no node yet, and *decides* which node each should run on (based on resources, constraints, affinity). It only writes the decision; it does not start anything.
  - **`kube-controller-manager`** — the building full of thermostats. Runs the built-in controllers (Deployment, ReplicaSet, Node, Job, …), each reconciling its kind of object.
  - **`cloud-controller-manager`** — the liaison to the cloud provider (provisioning load balancers, disks, nodes).
- **The node components, each as a factory-floor role:**
  - **`kubelet`** — the floor manager on each node. Watches the API server for Pods assigned to *its* node and makes them real by talking to the container runtime; reports health back. The bridge between the declarative world and actual running containers.
  - **container runtime** (containerd, CRI-O) — the machine operator that actually pulls images and runs containers.
  - **`kube-proxy`** — the floor's receptionist, wiring up Service networking on the node (more in Module 7).
- **The lifecycle of a single request: `kubectl apply` of a Deployment, end to end.** kubectl → API server (authn/authz/validate) → etcd (persisted) → Deployment controller sees it → creates a ReplicaSet → ReplicaSet controller creates Pods → scheduler assigns each Pod a node → that node's kubelet starts the containers → status flows back up to etcd → `kubectl get` shows it Running. *Payoff:* this one trace is the spine of the whole course; every later module deepens one hop of it.
- **Why this architecture is so resilient (and so eventually-consistent).** Components are decoupled and stateless except etcd; they coordinate only by watching shared state. *Payoff:* explains both the robustness and the "why is it taking a few seconds?" feel.

**What mastery looks like:** given any cluster behavior — a Pod stuck Pending, a node going NotReady, a change you made "not taking" — you can name which component is responsible and where to look, because you have the org chart in your head.

---

## Module 2 — kubectl & kubeconfig: the universal client

> 📖 **To be written:** [`02-kubectl-and-kubeconfig.md`](02-kubectl-and-kubeconfig.md)

**Why now.** With the architecture in hand (Module 1), `kubectl` stops being a grab-bag of commands and becomes one simple thing: **a command-line HTTP client for the API server.** This module makes you fluent and *unafraid* — you'll understand that almost every command is `kubectl <verb> <resource> <name>`, that it's just translating to REST calls against objects, and crucially, **which cluster and identity you're aimed at** (the source of most real-world accidents).

> **The mental model: `kubectl` is a remote control, and `kubeconfig` is which TV it's pointed at.** The same buttons (`get`, `apply`, `delete`) do the same things — but *which cluster they affect* depends entirely on the **context** your remote is currently set to. Pressing "delete" is harmless on the dev TV and a disaster on the prod TV; the buttons look identical. So before you press anything, you check what the remote is pointed at. This is why context-awareness is a survival skill, not a nicety.

Concepts to master:

- **`kubectl` is an API-server client.** Every command becomes an HTTP request to the API server; nothing more magic than that. `kubectl --v=8 get pods` literally shows you the REST calls. *Payoff:* demystifies everything and lets you reason about permissions, latency, and failures.
- **The grammar: `kubectl <verb> <resource> [name] [flags]`.** Verbs (`get`, `describe`, `create`, `apply`, `delete`, `edit`, `logs`, `exec`, `scale`, `rollout`…) × resources (`pods`, `deployments`, `nodes`, `svc`, `ns`…). Once you see the grid, you can compose commands you've never run. *Payoff:* you stop memorizing commands and start *generating* them.
- **`kubeconfig`, the file that defines your access.** Its three lists and how they combine:
  - **clusters** — *which* API server (URL + CA cert) — the TV.
  - **users** — *who you are* (certs, tokens, exec plugins) — your identity.
  - **contexts** — a named *(cluster + user + default namespace)* triple — a saved channel on the remote.
  The **current-context** is the active channel. *Payoff:* this is the mechanism behind "am I about to hit prod?"
- **Living in contexts safely.** `kubectl config get-contexts`, `current-context`, `use-context`; setting a default namespace per context; tools like `kubectx`/`kubens` and shell-prompt indicators. *Payoff:* the habit that prevents the classic "I ran it against the wrong cluster" outage.
- **Imperative vs declarative — the two ways to drive.** Imperative (`kubectl create`, `kubectl run`, `kubectl scale`) tells the cluster to do a thing now — great for learning and one-offs. Declarative (`kubectl apply -f` against YAML in git) submits desired state and lets reconciliation handle it — the production way. *Payoff:* connects directly to Module 0; you'll know *when* each is appropriate and why prod is declarative.
- **The "read" verbs that make you dangerous (in a good way):** `get` (with `-o wide`, `-o yaml`, `-o jsonpath`, `--show-labels`, `-w` to watch), `describe` (human-readable object + recent events), and especially **`kubectl explain`** (the built-in schema documentation for every field — your offline reference). *Payoff:* you can explore any cluster and any object type without a browser.
- **`--dry-run=client/server` and `--dry-run -o yaml` as a YAML generator.** Use imperative commands to *generate* declarative manifests. *Payoff:* fastest way to author correct YAML.
- **Labels and selectors, the cross-cutting query language.** `-l app=web`, `--field-selector`; how labels (not names) are how Kubernetes groups and finds objects. *Payoff:* foundational for Services, Deployments, and bulk operations everywhere later.

**What mastery looks like:** you can sit at an unfamiliar cluster, confirm which context/identity/namespace you're in *before* touching anything, explore its objects with `get`/`describe`/`explain`, and fluently compose verb-resource commands — generating YAML with `--dry-run` rather than memorizing it.

---

## Module 3 — Nodes & Scheduling: where Pods actually land

> 📖 **To be written:** [`03-nodes-and-scheduling.md`](03-nodes-and-scheduling.md)

**Why now.** You know the control plane *decides* and nodes *do the work* (Module 1) and you can drive `kubectl` (Module 2). Now zoom into the node — the unit of compute — and the **scheduler**, the matchmaker that assigns Pods to nodes. This is where "I have capacity but my Pod is Pending" gets explained, and where you learn to *steer* placement.

> **The mental model: the scheduler is a host seating guests in a restaurant.** Each Pod is a party that needs a table (a node) with enough seats (CPU/memory). The host checks which tables have room (filtering), then picks the *best* table among those (scoring) — maybe spreading parties out, maybe honoring "we'd like to sit near the window" (affinity) or "not next to the kitchen" (anti-affinity). Some tables have a "reserved" sign (**taints**) that only parties holding a matching reservation (**tolerations**) may sit at. The host only *seats* the party — the waiter (kubelet) actually serves them.

Concepts to master:

- **What a node is.** A machine (VM or physical) running a kubelet, a runtime, and kube-proxy, registered with the control plane. `kubectl get nodes`, `kubectl describe node`. *Payoff:* the concrete thing behind all that abstraction.
- **Node status and conditions.** `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`; what `NotReady` means and how the node controller reacts (evicting Pods after a grace period). *Payoff:* reading node health is half of cluster debugging.
- **Capacity vs allocatable.** Total node resources minus what's reserved for the kubelet/system = what Pods can actually request. *Payoff:* explains why a node "with free CPU" still won't schedule your Pod.
- **How scheduling actually works: filtering then scoring.** Predicates eliminate impossible nodes (not enough resources, taint not tolerated, selector mismatch); priorities rank the survivors. The Pod's **resource requests** (Module 8) are central here. *Payoff:* you can predict and explain placement, and diagnose `Pending` Pods via `kubectl describe pod` events.
- **Steering placement, from blunt to precise:**
  - **Labels on nodes** + **`nodeSelector`** — the simple "only on nodes labeled `disktype=ssd`."
  - **Node affinity / anti-affinity** — richer required/preferred rules.
  - **Pod affinity / anti-affinity** — "co-locate with" or "keep away from" other Pods (e.g. spread replicas across nodes).
  - **Taints & tolerations** — the *node* repels Pods by default unless they tolerate it (e.g. dedicating GPU nodes, keeping general workloads off control-plane nodes). The inverse mechanism to selectors. *Payoff:* the real tools for "run this *here*, never *there*."
- **Operating on nodes: cordon, drain, uncordon.** `kubectl cordon` (mark unschedulable), `kubectl drain` (evict Pods gracefully for maintenance), `kubectl uncordon`. How PodDisruptionBudgets protect availability during a drain. *Payoff:* the safe node-maintenance workflow every operator needs.

**What mastery looks like:** given a Pending Pod, you can read the scheduler's events and say *exactly* why no node fit (insufficient resources, an untolerated taint, an unsatisfiable affinity rule), and you can confidently take a node out of service for maintenance without dropping traffic.

---

## Module 4 — Workloads: the objects you actually run

> 📖 **To be written:** [`04-workloads.md`](04-workloads.md)

**Why now.** You can place a Pod (Module 3), but you almost never create bare Pods in practice — you create *controllers* that create and manage Pods for you. This module is the menu of workload objects and, crucially, **the controller pattern that connects them**: each higher-level object is a thermostat managing the one below it.

> **The mental model: a chain of managers, each watching the level below.** You don't hire and babysit individual workers (Pods); you tell a *team lead* (Deployment) "keep 3 people doing this job," and the team lead delegates to a *shift roster* (ReplicaSet) that ensures exactly 3 are clocked in, replacing anyone who leaves. You manage the org by editing the top of the chain; the reconciliation cascades downward. This is Module 0 applied to running code.

Concepts to master:

- **The Pod — the atom of scheduling.** One or more tightly-coupled containers sharing a network namespace and storage; why Pods (not containers) are the unit; the **sidecar** pattern; init containers. Pods are *mortal and cattle*, not pets — they get replaced, not repaired. *Payoff:* the object everything else ultimately produces.
- **ReplicaSet — the count-keeper.** Ensures *N* identical Pods exist; you rarely touch it directly. *Payoff:* the simplest concrete controller, and what a Deployment manages under the hood.
- **Deployment — the workhorse for stateless apps.** Manages ReplicaSets to give you **rolling updates** and **rollbacks**: changing the Pod template spins up a new ReplicaSet and scales down the old one gradually. `kubectl rollout status/history/undo`, `kubectl set image`, `kubectl scale`. *Payoff:* this is what you'll run 90% of the time; the safe-deploy mechanics tie into Module 9.
- **StatefulSet — for stateful, identity-needing apps.** Stable network identity and stable per-Pod storage, ordered rollout — for databases, queues, anything where Pods aren't interchangeable. *Payoff:* knowing *when* you need this instead of a Deployment (and the cost of choosing wrong).
- **DaemonSet — one Pod per node.** For node-level agents (log shippers, monitoring, CNI). Scales with the cluster, not with load. *Payoff:* the right tool for "something on every node."
- **Job & CronJob — run-to-completion and scheduled work.** Batch tasks that finish (Job) and time-triggered tasks (CronJob), with retry/parallelism semantics. *Payoff:* the non-server workloads, and a clean contrast to the always-on controllers.
- **The decision framework: which workload for which need?** Stateless service → Deployment. Stateful/ordered → StatefulSet. Per-node agent → DaemonSet. Finite task → Job/CronJob. *Payoff:* the practical chooser you'll use on every new app.
- **Pod health: liveness, readiness, and startup probes.** How Kubernetes knows a Pod is alive (restart if not), ready for traffic (gate it out of Services if not — links to Module 7), and done starting. *Payoff:* the difference between "deployed" and "actually working," and a top source of subtle outages.

**What mastery looks like:** for any application you're handed, you can pick the right workload object and explain why, author its manifest, and trace how a single field change cascades through the controller chain into new Pods on nodes.

---

## Module 5 — Namespaces: where to use them, and where not

> 📖 **To be written:** [`05-namespaces.md`](05-namespaces.md)

**Why now.** You're creating objects (Modules 3–4); namespaces are how you *organize* them within one cluster. This module is deliberately opinionated about the question the user actually has: **where do namespaces help, and where do people misuse them?**

> **The mental model: namespaces are apartments in one building, not separate buildings.** They give each tenant their own rooms, their own labeled mailboxes (names are unique *within* a namespace, so two teams can both have a `web` Deployment), their own utility meter you can cap (**ResourceQuota**), and their own door key (**RBAC** scoped per namespace). But they all share the building's foundation, plumbing, and walls — the **nodes, the network, and the kernel**. So a namespace is an *administrative and naming* boundary, **not a hard security or hardware boundary.** Knowing exactly what a wall *is and isn't* tells you when an apartment is enough and when you need a separate building (a separate cluster).

Concepts to master:

- **What a namespace actually scopes.** A naming and grouping boundary for *namespaced* objects (Pods, Deployments, Services, ConfigMaps…). Names are unique within, not across, a namespace. *Payoff:* the basic "why can two teams both have a `frontend` service?" answer.
- **What it does *not* scope: cluster-scoped objects.** Nodes, PersistentVolumes, Namespaces themselves, ClusterRoles, StorageClasses live *outside* any namespace. `kubectl api-resources --namespaced=true/false`. *Payoff:* explains why `kubectl get nodes -n foo` ignores the namespace, and prevents a common mental-model bug.
- **The default namespaces.** `default` (and why dumping everything there is a smell), `kube-system` (the control plane's own workloads — look but don't touch), `kube-public`, `kube-node-lease`. *Payoff:* orientation in any real cluster.
- **Namespaces in `kubectl`.** `-n <ns>`, `--all-namespaces/-A`, setting a default namespace in your context (Module 2). *Payoff:* the daily ergonomics, and another guardrail against acting in the wrong place.
- **What namespaces *enable* — the three real payoffs:**
  - **RBAC scoping** — grant a team rights only within their namespace (Roles/RoleBindings vs ClusterRoles). *The* main reason namespaces exist for multi-team clusters.
  - **ResourceQuota** — cap total CPU/memory/object counts per namespace, so one team can't starve the cluster.
  - **LimitRange** — default and bound per-Pod requests/limits within a namespace (ties to Module 8).
- **Where to use them (and where not).** **Use** for: per-team or per-app grouping, per-environment-within-a-cluster (with care), applying quotas and RBAC, avoiding name collisions. **Don't rely on** for: strong security isolation between untrusted tenants (they share a kernel and network — use separate clusters or stronger isolation), or as a substitute for separate prod/dev *clusters* when blast-radius matters. *Payoff:* the judgment call this whole module exists to give you.
- **Cross-namespace interaction.** Service DNS names include the namespace (`svc.namespace.svc.cluster.local`) — a preview of Module 7 — so namespaces affect how things *find* each other, not just how they're listed.

**What mastery looks like:** given a "how should we organize this cluster?" question, you can lay out a namespace scheme, attach the right quotas and RBAC, and articulate clearly where namespaces give you real isolation and where you're trusting a soft boundary that demands a separate cluster instead.

---

## Module 6 — Config, Secrets & Storage: separating data from code

> 📖 **To be written:** [`06-config-secrets-storage.md`](06-config-secrets-storage.md)

**Why now.** Your workloads (Module 4) shouldn't bake configuration or carry state inside the container image. This module covers the objects that inject configuration and attach durable storage — and the reconciliation model extends naturally to them.

> **The mental model: the container image is a sealed appliance; config and storage are what you plug into it at the wall.** The same image (the appliance) runs in dev and prod unchanged — what differs is the cartridge you slot in (**ConfigMap**/**Secret**) and whether you attached an external hard drive that survives the appliance being unplugged (**PersistentVolume**). Containers are ephemeral; their on-disk scratch space vanishes when they restart. Anything that must outlive a Pod has to live *outside* the Pod.

Concepts to master:

- **ConfigMaps — non-secret configuration as an object.** Injected as environment variables or mounted as files; decoupling config from image. *Payoff:* the same image across environments.
- **Secrets — for sensitive values.** Same shapes as ConfigMaps but for credentials; the honest truth that base64 ≠ encryption, that etcd-at-rest encryption and RBAC are what actually protect them, and the existence of external secret stores. *Payoff:* handling credentials without lying to yourself about the security model.
- **Env vars vs mounted volumes for config.** Trade-offs: env is simple but static for the Pod's life; mounted ConfigMaps can update. *Payoff:* picking the right injection method.
- **The ephemeral-storage problem.** A container's filesystem resets on restart; `emptyDir` lives only as long as the Pod. *Payoff:* motivates persistent storage.
- **The storage abstraction: PV, PVC, StorageClass.** A **PersistentVolumeClaim** is the Pod's *request* for storage ("I need 10Gi, read-write"); a **PersistentVolume** is the actual disk; a **StorageClass** lets the cluster *dynamically provision* the PV to satisfy the claim. This is the reconciliation loop again — you declare a claim, a controller fulfills it. *Payoff:* the standard, cloud-portable way to give stateful workloads (Module 4's StatefulSets) real disks.
- **Access modes and lifecycle.** ReadWriteOnce/ReadOnlyMany/ReadWriteMany; reclaim policies (what happens to the disk when the claim is deleted). *Payoff:* avoiding both data loss and orphaned-disk surprises.

**What mastery looks like:** you can take a stateful app, externalize all its configuration into ConfigMaps/Secrets, give it durable storage via a PVC backed by a StorageClass, and explain exactly which pieces survive a Pod restart, a node failure, and a claim deletion.

---

## Module 7 — Services & Networking: how things find and reach each other

> 📖 **To be written:** [`07-services-and-networking.md`](07-services-and-networking.md)

**Why now.** Pods are mortal and their IPs change constantly (Module 4) — so you can never hardwire a Pod IP. This module is how Kubernetes solves "how do I reliably reach a moving target?" — a stable front for a shifting set of Pods — and how traffic gets in from outside.

> **The mental model: a Service is a company switchboard number; the Pods are employees who come and go.** You call the department's *published number* (the Service's stable virtual IP and DNS name), and the switchboard (kube-proxy + the Service's endpoint list) routes you to whichever employee is currently working (a healthy, ready Pod). Employees are hired and fired constantly (Pods churn), but the published number never changes. You never need to know any individual's desk extension (Pod IP).

Concepts to master:

- **The flat Pod network and why Pod IPs are useless to depend on.** Every Pod gets an IP, every Pod can reach every other Pod, but those IPs are ephemeral. *Payoff:* the problem Services exist to solve.
- **The Service object and its stable identity.** A stable virtual IP (ClusterIP) + DNS name fronting a dynamic set of Pods selected **by label** (back to Module 2's selectors). *Payoff:* the core abstraction for service-to-service communication.
- **Endpoints / EndpointSlices — the live roster.** How the Service controller keeps the list of *currently ready* backing Pods up to date (readiness probes from Module 4 decide membership). *Payoff:* connects health to routing — an unready Pod silently leaves the rotation.
- **kube-proxy — the switchboard wiring on each node.** How it programs iptables/IPVS rules so the virtual IP load-balances to real Pods. *Payoff:* demystifies "how does a fake IP actually work?"
- **Cluster DNS (CoreDNS) and service discovery.** `service.namespace.svc.cluster.local`; why apps address Services by *name*, and how namespaces (Module 5) show up in the name. *Payoff:* the everyday mechanism of in-cluster discovery.
- **The Service types, as a ladder of exposure:**
  - **ClusterIP** — internal-only (the default); reachable from inside the cluster.
  - **NodePort** — opens a port on every node; the crude way out.
  - **LoadBalancer** — provisions a cloud load balancer (via the cloud-controller-manager, Module 1); the standard external entrypoint for L4.
  - **(headless)** — no virtual IP, returns Pod IPs directly; for StatefulSets and client-side LB.
- **Ingress (and a word on Gateway API) — HTTP routing at the edge.** One entrypoint routing by hostname/path to many Services, with TLS termination — instead of a load balancer per service. *Payoff:* the real-world way HTTP apps are exposed.
- **A note on NetworkPolicies.** Default-open networking and how policies lock down which Pods may talk to which — the network counterpart to RBAC. *Payoff:* the isolation that namespaces alone don't give you (callback to Module 5).

**What mastery looks like:** you can trace a request from an external client through Ingress → Service → a ready Pod, name the component at each hop (cloud LB, kube-proxy, DNS, endpoints), and explain why the path keeps working as Pods are created and destroyed underneath it.

---

## Module 8 — Autoscaling: the control loop applied to scale

> 📖 **To be written:** [`08-autoscaling.md`](08-autoscaling.md)

**Why now.** You have running, reachable workloads (Modules 4, 7). Autoscaling is the payoff of the whole declarative model: instead of *you* deciding the replica count, you declare a *goal* ("keep CPU around 60%") and yet another control loop adjusts the count for you. It builds directly on resource **requests/limits**, which is why it comes after workloads and scheduling.

> **The mental model: autoscaling is cruise control, and there are three different pedals.** You set a target (desired speed / desired utilization); a controller continuously measures the actual value and adjusts to hold the target. But "adjust" can mean three different things: push the existing engine harder by adding *more identical cars* (**Horizontal** Pod Autoscaler — more replicas), put a *bigger engine* in each car (**Vertical** Pod Autoscaler — more CPU/memory per Pod), or *build more road* when the cars won't fit (**Cluster Autoscaler** — more nodes). They operate at different layers and are often combined.

Concepts to master:

- **The foundation: resource requests and limits.** **Requests** are what the scheduler reserves (Module 3) and what utilization is measured *against*; **limits** are the hard ceiling the kubelet enforces (throttling CPU, OOM-killing on memory). Autoscaling is meaningless without sane requests. *Payoff:* the prerequisite that, set wrong, breaks every autoscaler — this is where most autoscaling problems actually live.
- **The metrics pipeline.** `metrics-server` for basic CPU/memory; the custom/external metrics APIs for scaling on queue depth, RPS, etc. `kubectl top nodes/pods`. *Payoff:* you can't scale on a signal you can't measure — and you'll know where the numbers come from.
- **Horizontal Pod Autoscaler (HPA) — more replicas.** Watches a metric against a target and adjusts a Deployment's replica count up/down within min/max bounds; the reconciliation math; stabilization windows to avoid flapping. *Payoff:* the workhorse autoscaler for stateless services.
- **Vertical Pod Autoscaler (VPA) — bigger Pods.** Recommends or sets requests/limits based on observed usage; why it conflicts with HPA on the same metric; why applying it usually means restarting Pods. *Payoff:* right-sizing, especially for workloads that can't scale horizontally.
- **Cluster Autoscaler — more nodes.** When Pods are Pending for lack of capacity (Module 3), it adds nodes; when nodes sit underused, it drains (Module 3) and removes them. The interplay: HPA adds Pods → no room → Cluster Autoscaler adds nodes. *Payoff:* closing the loop so scaling pods doesn't just produce Pending pods. (Plus a nod to newer provisioners like Karpenter.)
- **How they compose, and the failure modes.** HPA + Cluster Autoscaler is the common pair; HPA + VPA needs care; the classic "everything Pending because requests are too high" and "scaling thrash" problems. *Payoff:* designing an autoscaling setup that actually stabilizes instead of oscillating.

**What mastery looks like:** you can set correct requests/limits, attach an HPA targeting a meaningful metric, pair it with a Cluster Autoscaler, and reason about the combined behavior under a load spike — predicting whether you'll get more Pods, more nodes, or a pile of Pending Pods, and why.

---

## Module 9 — Debugging & Operating with kubectl: seeing inside and changing safely

> 📖 **To be written:** [`09-debugging-and-operating.md`](09-debugging-and-operating.md)

**Why last.** This module ties the whole course together: every debugging technique is "use `kubectl` (Module 2) to inspect an object or component you now understand (Modules 1–8)." It's where the architecture pays off — because you know the org chart, you know *where to look* when something breaks, and *how to change things without causing an outage*.

> **The mental model: debugging Kubernetes is following the declare-to-running trace backwards until reality stops matching desire.** You declared a desired state; something is off. So you walk the chain from Module 1 in reverse — is the object as I intended? did the controller create children? did the scheduler place them? did the kubelet start them? are they healthy and in the Service's roster? — and the *first* place desired ≠ actual is your bug. Every `kubectl` debugging command is a probe at one link in that chain.

Concepts to master:

- **`kubectl describe` and events as the first stop.** Events attached to an object are the system telling you, in English, what it tried and what failed (FailedScheduling, ImagePullBackOff, CrashLoopBackOff, FailedMount). `kubectl get events --sort-by=...`. *Payoff:* 80% of problems are diagnosed here before you touch logs.
- **Reading the status of objects.** Pod phases and container states (Pending/Running/Succeeded/Failed; Waiting/Running/Terminated with reasons), restart counts — and mapping each back to which component (Module 1) is stuck. *Payoff:* turning a status string into a precise next step.
- **`kubectl logs`.** Current and previous (`-p`) container logs, `-f` to follow, `-c` for a specific container, `--since`, multi-Pod logs by label. *Payoff:* the application's own account of what went wrong.
- **`kubectl exec` and `kubectl debug`.** Shell into a running container; attach an **ephemeral debug container** to a distroless/minimal Pod that has no shell; `kubectl debug node` for node-level access. *Payoff:* live inspection when logs aren't enough — without rebuilding images.
- **`kubectl port-forward`.** Tunnel a local port straight to a Pod or Service, bypassing Ingress/Service exposure, to test something directly. *Payoff:* isolate "is the app broken or is the networking broken?"
- **`kubectl top` and resource pressure.** Live CPU/memory for nodes and Pods (Module 8's metrics-server); spotting throttling and OOM kills. *Payoff:* connecting performance symptoms to requests/limits.
- **Operating safely: rollouts and rollbacks.** `kubectl rollout status/history/undo`, pause/resume, `kubectl scale`, `kubectl edit` vs `apply` (and why live-editing drifts from your git source of truth). *Payoff:* changing production without the change *being* the outage — and the discipline of declarative ops.
- **RBAC in practice and `kubectl auth can-i`.** Checking what your identity (Module 2) is allowed to do *before* you're surprised by a `Forbidden`; reading Roles/Bindings. *Payoff:* understanding and debugging permission errors instead of fearing them.
- **Context discipline as an operational safeguard.** Re-grounding Module 2: confirm context/namespace before any mutating command; the habits and tooling that prevent wrong-cluster accidents. *Payoff:* the operational maturity that separates "knows kubectl" from "trusted with prod."

**What mastery looks like:** handed a broken app in an unfamiliar cluster, you methodically walk the declare-to-running chain with `kubectl` — events, status, logs, exec, top — pinpoint the first link where desired ≠ actual, fix it via a *declarative* change, and roll it out (or roll it back) without dropping traffic or touching the wrong cluster.

---

## Putting it together: the "declare-to-running" path as the capstone

Everything in this plan converges on one trace you should be able to draw and narrate by the end — the **declare-to-running path**, the Kubernetes analogue of "follow the request through the system":

> You write a Deployment manifest and run `kubectl apply` (Module 2) — pointed at the *intended* context (Modules 2, 9) and namespace (Module 5). The request hits the **API server**, which authenticates and authorizes your identity (RBAC, Modules 2, 9), validates the object, and writes it to **etcd** (Module 1). The **Deployment controller** sees the new object and creates a **ReplicaSet**; the ReplicaSet controller creates **Pods** (Module 4), each carrying its **resource requests** (Module 8) and references to **ConfigMaps/Secrets/PVCs** (Module 6). The **scheduler** filters and scores **nodes** and assigns each Pod one (Module 3); that node's **kubelet** pulls the image, mounts the config and storage, and starts the containers, running **probes** until they're **Ready** (Module 4). A **Service** (Module 7) adds each ready Pod to its endpoint roster; **CoreDNS** gives it a name and **kube-proxy** wires the routing; an **Ingress** exposes it to the world. Under load, the **HPA** adds replicas; if they don't fit, the **Cluster Autoscaler** adds nodes (Module 8). And forever after, every controller in that chain keeps **reconciling** — replacing failed Pods, rerouting around dead nodes — with no further input from you (Module 0). When something breaks, you walk this exact chain backwards with `kubectl` (Module 9) to the first link where desired ≠ actual.

When you can draw that path, name every component, explain *why* each step is built the way it is, and point to which `kubectl` command inspects each hop — you've integrated the whole curriculum. **That trace is the job.**

---

## Suggested study sequence and milestones

1. **Foundations (Modules 1–2):** Build the control-loop mental model and become fluent and *safe* with `kubectl` and contexts. *Milestone:* run `kubectl --v=8 get pods`, identify the REST call, and explain which component answers it; switch contexts and prove which cluster you're aimed at.
2. **What you run and where (Modules 3–4):** Understand the node/scheduler and the workload chain. *Milestone:* deploy an app as a Deployment, diagnose a deliberately-Pending Pod from scheduler events, and watch the controller cascade replace a Pod you delete.
3. **Organizing and connecting (Modules 5–7):** Namespaces, config/storage, and networking. *Milestone:* run the same app in two namespaces with a quota, externalize its config, and reach it by Service DNS from another Pod and from outside via Ingress.
4. **Scaling (Module 8):** Requests/limits and the three autoscalers. *Milestone:* attach an HPA, drive load, and narrate whether you get more Pods, more nodes, or Pending Pods — and why.
5. **Operating (Module 9):** Debug and change safely. *Milestone:* break the app three different ways (bad image, bad config, failing probe) and diagnose each from `describe`/events/logs, then fix it declaratively and roll out.

---

## Glossary (quick reference)

| Term | Meaning |
|------|---------|
| **Declarative / desired state** | You describe *what* you want as objects; you don't issue step-by-step *how* commands. |
| **Reconciliation loop** | A controller continuously observing current state, comparing to desired, and acting to close the gap. The heart of Kubernetes. |
| **Controller** | A control loop that manages one kind of object (Deployment, ReplicaSet, Node, Job…). |
| **Control plane** | The "head office" — API server, etcd, scheduler, controller-manager — that decides and records. |
| **Worker node** | A machine running the kubelet, container runtime, and kube-proxy — where Pods actually run. |
| **etcd** | The cluster's single consistent key-value store; the one source of truth. |
| **kube-apiserver** | The only gateway to etcd; authenticates, authorizes, validates, persists every change. |
| **kube-scheduler** | Assigns Pods to nodes (decides placement; doesn't start anything). |
| **kubelet** | The per-node agent that turns assigned Pods into running containers and reports health. |
| **kube-proxy** | Programs each node's networking so Service virtual IPs load-balance to Pods. |
| **kubectl** | The command-line client for the API server; does CRUD on objects. |
| **kubeconfig** | The file defining clusters, users, and contexts — *which* cluster and *who* you are. |
| **Context** | A named (cluster + user + default namespace) triple; the "channel" your kubectl is set to. |
| **Pod** | The smallest deployable unit: one or more containers sharing network and storage. Mortal. |
| **ReplicaSet / Deployment** | Count-keeper / rolling-update manager for stateless Pods. |
| **StatefulSet / DaemonSet / Job** | Stable-identity workloads / one-per-node agents / run-to-completion tasks. |
| **Namespace** | An administrative + naming boundary within a cluster; scopes RBAC and quotas, *not* hardware/security. |
| **ConfigMap / Secret** | Non-secret / sensitive configuration injected into Pods as env or files. |
| **PV / PVC / StorageClass** | The actual disk / a Pod's claim for storage / the provisioner that fulfills the claim. |
| **Service** | A stable virtual IP + DNS name fronting a dynamic set of label-selected Pods. |
| **Endpoints / EndpointSlice** | The live list of ready Pods currently backing a Service. |
| **Ingress** | Edge HTTP router mapping hostnames/paths to Services, with TLS. |
| **Requests / limits** | Resources the scheduler reserves / the hard ceiling the kubelet enforces. The basis of scheduling and autoscaling. |
| **HPA / VPA / Cluster Autoscaler** | More replicas / bigger Pods / more nodes — the three autoscaling "pedals." |
| **Taint / toleration** | A node repelling Pods by default / a Pod's permission to land on a tainted node. |
| **cordon / drain** | Mark a node unschedulable / gracefully evict its Pods for maintenance. |
| **RBAC** | Role-based access control; what an identity may do, often scoped per namespace. |

---

## Where to go next

Once the plan above is internalized, the natural directions are:
- **Write the full lessons.** Each module here is a candidate for its own deep, concept-first lesson file (`01-architecture-control-loop.md`, etc.) in the style of the [`../linux/`](../linux/) and [`../hft/`](../hft/) courses.
- **Build it with your hands on a throwaway cluster.** Spin up `kind`, `minikube`, or `k3s` locally and run the milestones above — Kubernetes only becomes real when you've watched a controller replace a Pod you killed and diagnosed a Pending Pod yourself.
- **Read the canon.** The official Kubernetes docs (concepts section, not just reference), *Kubernetes Up & Running*, the *Programming Kubernetes* book for the controller/operator pattern, and `kubectl explain` as a daily habit.

> The fastest path to Kubernetes fluency is not memorizing `kubectl` flags — it's holding the control-loop model so firmly that every command, object, and failure is obviously *just another place desired state meets reality*. Once that clicks, `kubectl` is no longer a list to memorize; it's a sentence you compose.
