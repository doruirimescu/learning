# Kubernetes Sub-Course — Local Guide

This folder is a concept-first course on **Kubernetes for engineers who want to truly understand it through `kubectl`**. It follows the same style as the `../linux/` and `../hft/` courses: lead with mental models and analogies, explain the *why* behind everything, end each lesson with a glossary and "where to go next" — no dry command/flag dumps. Every `kubectl` command is wrapped in the *why* and tied to the running mental model.

## Conventions

- **Lesson files:** markdown, named `NN-topic.md` (two-digit ordering + kebab-case subject), e.g. `01-architecture-control-loop.md`. They live directly in this `kubernetes/` folder.
- **`README.md`** is the master lesson plan / curriculum: the ordered module arc, the *why* of each, and the "declare-to-running" capstone. Keep it in sync as lessons get written.
- **Style:** comprehensive and user-friendly; each big concept gets its own section; every term/command is tied to a running mental model. The anchoring frame for the whole course is **Module 0: Kubernetes is a control loop — you declare desired state, and the control plane continuously reconciles reality toward it.** `kubectl` is your conversation with that loop.

## Module → lesson map (planned numbering)

1. `01-architecture-control-loop.md` — the reconciliation loop, cluster, control plane (API server, etcd, scheduler, controller-manager), nodes (kubelet, kube-proxy, runtime).
2. `02-kubectl-and-kubeconfig.md` — kubectl as the API-server client, kubeconfig, contexts/clusters/users, the verb-noun grammar, imperative vs declarative, get/describe/explain/apply.
3. `03-nodes-and-scheduling.md` — node anatomy, conditions, capacity vs allocatable, labels, taints/tolerations, affinity, cordon/drain.
4. `04-workloads.md` — Pods → ReplicaSets → Deployments, plus StatefulSets, DaemonSets, Jobs/CronJobs; the controller pattern; where each fits.
5. `05-namespaces.md` — what namespaces are, **where to use them and where not**, scoping, ResourceQuota/LimitRange, RBAC boundaries, multi-tenancy.
6. `06-config-secrets-storage.md` — ConfigMaps, Secrets, env vs volumes, PersistentVolumes/PVCs, StorageClasses.
7. `07-services-and-networking.md` — Services (ClusterIP/NodePort/LoadBalancer), Endpoints, kube-proxy, cluster DNS, Ingress.
8. `08-autoscaling.md` — requests/limits as the foundation, metrics-server, HPA, VPA, Cluster Autoscaler — the control loop applied to scale.
9. `09-debugging-and-operating.md` — logs/exec/port-forward/describe/events/top/debug, rollouts & rollbacks, RBAC in practice, contexts in real workflows.

Lessons are intended to be read 1 → 9 (dependency-ordered): you can't reason about `kubectl` (2) without the control loop (1), can't reason about autoscaling (8) without workloads (4) and requests/limits, and so on. See `README.md` for the full rationale and the declare-to-running capstone that integrates all modules.
