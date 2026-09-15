# Kubernetes Real-World Project Lab
### From Hands-On Basics to Production-Style Incidents — Topics 1–18

Your companion workbook drills each topic in isolation. This one does the opposite: it fuses everything from Topics 1–18 into escalating projects that behave like real infrastructure — nothing works perfectly the first time, something always breaks, and you have to work out why with nothing but `kubectl`, logs, and events.

Build everything on your own kubeadm cluster, not a throwaway kind/minikube box. Real Calico networking, a real MetalLB IP pool, real ingress-nginx + cert-manager TLS, and real Prometheus scraping teach instincts a toy cluster can't — and they're the exact instincts interviewers probe for.

---

## How This Lab Works

Each project has the same shape:

- **Scenario** — the business framing, so you practice translating "the CEO says the site is slow" into the right `kubectl` command.
- **Build it** — a requirements list, not a YAML file. Writing the manifest yourself is the point.
- **Break it** — a deliberate fault you inject yourself, with a diagnostic goal but no fix given. Reproducing a failure on purpose and then hunting it down is the single highest-leverage practice for both real incidents and interviews.
- **Security checklist / Observability checklist** — small, non-negotiable items so security and monitoring stop being "extra topics" and become habits baked into every deployment.
- **Success criteria** — how you know you're actually done, not just that you stopped getting errors.
- **Interview angle** — the questions this project should let you answer fluently, out loud, without notes.

No solutions or hints are given anywhere in this document — same philosophy as your existing workbook. If you get stuck, that's the exercise working correctly.

**Optional stretch:** your cluster already runs ArgoCD and Jenkins. Once a project works via `kubectl apply`, try shipping it again through an ArgoCD Application or a Jenkins pipeline instead. That's free revision of tools you already operate — GitOps/CI/CD itself is Topic 35, further down the road, so treat this as a bonus rep, not a requirement.

---

## Coverage Map — Nothing From Topics 1–18 Is Skipped

| # | Topic | Primary Project(s) |
|---|-------|---------------------|
| 1 | What is Kubernetes & why | Intro discussion, Project 0 |
| 2 | Containers/Docker & the problem K8s solves | Project 0 |
| 3 | Kubernetes Architecture | Project 0 |
| 4 | Setting up your practice environment | Project 0 |
| 5 | kubectl basics | Project 1.5, and every project after |
| 6 | Pods | Project 1, Project 1.5 |
| 7 | ReplicaSets & Deployments | Project 1 |
| 8 | Services | Project 1, Project 2 |
| 9 | Ingress | Project 1, Project 7 |
| 10 | ConfigMaps & Secrets | Project 2, Project 7 |
| 11 | Storage: Volumes, PV, PVC, StorageClass | Project 3 |
| 12 | Namespaces, Labels, Selectors & Annotations | Project 1.5, Project 2 |
| 13 | Autoscaling (HPA/VPA/Cluster Autoscaler) | Project 4 |
| 14 | Health Checks: Probes | Project 2, Project 4 |
| 15 | Resource Requests, Limits & QoS | Project 2, Project 4 |
| 16 | StatefulSets, DaemonSets, Jobs & CronJobs | Project 3, Project 6 |
| 17 | RBAC & Security (RBAC + SecurityContext + Pod Security Admission) | Project 5, Project 7 |
| 18 | Monitoring & Logging | Project 4, Project 6, Project 7 |

---

## Project 0 — Cluster Reality Check 🟢 (Topics 1–4)

**Scenario:** Before building anything, prove your control plane and its bolted-on components are actually healthy — not just "pods are Running."

**Build it:**
- Confirm kube-apiserver, scheduler, controller-manager, and etcd health directly from their static pod manifests and logs on the control-plane node, not just `kubectl get nodes`.
- Confirm Calico node status is healthy on every node.
- Confirm MetalLB's speaker pods are up and its IP address pool actually has spare addresses.
- Confirm the ingress-nginx controller pod is Running and its admission webhook is reachable.
- Confirm cert-manager is running with at least one working ClusterIssuer.
- Check real headroom with `kubectl top nodes` before piling workloads on.

**Break it:**
- Stop the kubelet on one worker node and time exactly how long it takes for the node to flip to `NotReady` and for its pods to be marked for eviction. Explain the node heartbeat/lease mechanism in your own words afterward.
- Kill the Calico pod on one node and compare what happens to *existing* pod-to-pod traffic on that node versus *new* pod scheduling there.
- Apply a Deployment, then change a running pod's **label** (not its image) so it no longer matches its ReplicaSet's selector — not editing the spec, just the label. Watch a brand-new replacement pod appear while your edited pod keeps running, now orphaned. Explain precisely what the reconciliation loop noticed and what it deliberately ignored.
- Trace one `kubectl apply` end-to-end using only `kubectl get events` and control-plane component logs, and sketch your own sequence diagram of api-server → etcd → scheduler → kubelet.

**Interview angle:** static pods vs. DaemonSets for control-plane components; what a kubelet heartbeat timeout actually triggers; why a CNI failure often looks like "everything is slow," not "everything is down"; what "reconciliation" really watches (object existence and count via label selectors) versus what it does *not* enforce (content drift on an already-running, still-matching pod).

---

## Project 1 — Deploy the Front Door 🟢/🟡 (Topics 5–9)

**Scenario:** You're the new engineer at a small startup. Ship a public-facing "coming soon" site.

**Build it:**
- A `project1` namespace, a Deployment (3 replicas) with sane resource requests/limits.
- A ClusterIP Service and an Ingress through your real ingress-nginx controller with a real host header.
- TLS via a cert-manager ClusterIssuer.
- A second path (e.g. `/api`) on the same Ingress routing to a second small backend Service. Test `pathType: Prefix` versus `pathType: Exact` explicitly and confirm they match different sets of URLs.
- A rolling update to a new image tag, then a rollback — both done with `kubectl` commands only, no re-applying an old YAML file.

**Break it:**
- Typo the Service's selector so it silently stops matching the Pod labels. Diagnose it using only `kubectl describe` and `kubectl get endpoints` — don't diff the YAML to find it.
- Scale to 0 replicas mid-`curl`-loop and note the *exact* client-side error and why it looks that way.
- Point the Ingress at a Service name that doesn't exist, then find the cause from the ingress-nginx controller's own logs.

**Security checklist:** containers aren't running as root (or you can justify why one must); resource limits are set so one misbehaving pod can't starve the node.

**Observability checklist:** `kubectl top pod` numbers roughly match what you see (or don't see) as a scrape target in Prometheus — and you know why.

**Interview angle:** ClusterIP vs. NodePort vs. LoadBalancer and how kube-proxy actually routes a Service; a 502 vs. a 503 from ingress-nginx; Deployment rollout mechanics (`maxSurge`/`maxUnavailable`); the exact rollback command and how revision history works; when `pathType: Prefix` and `Exact` disagree on the same URL.

---

## Project 1.5 — kubectl Muscle Memory & Pod Internals 🟢/🟡 (Topics 5 & 6, deeper)

**Scenario:** kubectl and Pods feel like "day one" material, but both hide some of the most commonly asked interview questions. This project deliberately slows down on them before you move on to anything more advanced.

**Build it:**
- Generate a Deployment manifest with `--dry-run=client -o yaml` without applying it, hand-edit the replica count, then apply it for real.
- Use `kubectl explain` on three fields you've never touched before and summarize each in your own words.
- Filter pods across a namespace with mixed workloads using a multi-condition label selector — e.g. an `env in (dev,staging)` condition combined with a separate tier label.
- Build a 3-container Pod — init container, main app, sidecar — where the init container must succeed before the other two start. Prove the ordering with timestamps in the logs, not by reading the manifest and assuming.

**Break it:**
- Given only a Pod name (no YAML files on disk), use `kubectl` commands alone to determine its image, node, labels, restart count, and its last 20 log lines.
- Run `kubectl diff` before applying a change to a live resource, and deliberately construct a case where it catches something `apply` alone would have silently changed.
- Use an ephemeral debug container (`kubectl debug`) to troubleshoot a running pod whose image has no shell and no debugging tools at all.
- Compare `restartPolicy: Always`, `OnFailure`, and `Never` by crashing a container under each and recording exactly what happens under each setting.
- Delete a running pod and, using only `kubectl describe` and events, narrate the exact sequence between the delete call and the container process stopping (SIGTERM, the grace period, then SIGKILL if it hasn't exited). Then add a `preStop` hook and a longer `terminationGracePeriodSeconds` to a deliberately slow-shutting-down app, and show how that changes the sequence during a rolling update.

**Interview angle:** why `kubectl create` and `kubectl apply` behave differently on a second run against the same object; what `kubectl explain` is actually reading from; why a Pod's IP changing on every restart makes hardcoding it a production bug; the exact SIGTERM → grace period → SIGKILL sequence, and precisely where a slow-shutdown app causes user-facing errors during a rollout.

---

## Project 2 — Config Without Chaos 🟡 (Topics 8, 10, 12, 14, 15)

**Scenario:** The same app now needs per-environment config, a database password, and to survive being memory- and CPU-hungry without taking the node down with it.

**Build it:**
- `dev` and `staging` namespaces running the same Deployment with different ConfigMap values.
- A Secret for a DB password — mount it as an env var in one namespace and as a volume file in the other, and note the practical difference in update behavior.
- Liveness, readiness, and startup probes tuned to the app's real boot time.
- Resource requests/limits that produce a **Guaranteed** QoS pod in one namespace and a **Burstable** one in the other.
- A namespace-level `LimitRange` applied to one namespace, then a pod created there with no resources specified — confirm it inherits the defaults instead of landing in `BestEffort`.
- A call from a pod in the `dev` namespace to the `staging` namespace's Service using its full `<svc>.<namespace>.svc.cluster.local` FQDN, and confirm the short name alone does *not* resolve across namespaces.

**Break it:**
- Set a memory limit below what the app actually needs and let it get OOMKilled. Explain exit code 137 as if writing the incident's root-cause line.
- Set an aggressive liveness probe timeout so healthy pods get killed mid-startup — reproduce a crash loop that's *config-caused*, not app-caused, and prove that distinction from events and logs alone.
- Update the ConfigMap and check whether already-running pods pick up the change without a restart. Explain why or why not. Then add a checksum annotation of the ConfigMap's content to the pod template, and confirm a ConfigMap change now correctly triggers a real rolling restart, closing the gap you just found.

**Security checklist:** confirm the Secret's value doesn't show up in `kubectl describe pod`; know exactly what "base64, not encrypted" means for anyone with `get secret` access (this sets up Project 5).

**Observability checklist:** find pod restart count as a real metric in Prometheus. Write an alerting rule that would have fired automatically during your liveness-probe crash loop above (e.g., restart rate > 0 over 5 minutes), and confirm it actually fires when you re-run that fault.

**Interview angle:** QoS classes and node-pressure eviction order; a failing probe vs. a failing app; live ConfigMap updates vs. Secret updates and why the checksum-annotation trick exists; env var vs. volume-mounted secrets; short Service names vs. fully-qualified DNS names across namespaces.

---

## Project 3 — The Stateful Tier 🟡/🟠 (Topics 11, 16 — StatefulSets)

**Scenario:** Add a real database tier behind the app.

**Build it:**
- Before wiring in persistent storage: run the same database briefly on an `emptyDir` volume, write data to it, delete the pod, and confirm the data is gone. This is your baseline for why the rest of this project matters.
- A single-replica StatefulSet (e.g., a small Postgres/MySQL image) with a PVC per pod via a StorageClass, and a headless Service for stable network identity.
- Confirm the pod's hostname is stable and its PVC survives a pod restart.
- Take a manual backup that proves the data exists independently of the pod.
- Set the reclaim policy to `Retain`, delete a PVC, and confirm the underlying storage still exists and needs manual cleanup. Repeat with `Delete` and compare exactly what's left behind.

**Break it:**
- Delete the StatefulSet's pod (not the PVC) and confirm data survives. Then deliberately delete the PVC and watch data loss happen. Write down, precisely, which command is reversible and which isn't.
- Fill the PVC to 100% and diagnose the failure from the *application's* perspective, not just `df -h`.
- Stop your storage provisioner and watch a brand-new PVC get stuck `Pending` — explain exactly what's blocking it.

**Security checklist:** the DB pod isn't exposed beyond ClusterIP; its volume isn't world-writable by default.

**Observability checklist:** define the PVC usage percentage as something you'd alert on *before* 100%, not after.

**Interview angle:** PV/PVC lifecycle and reclaim policies (Retain vs. Delete); what a headless Service does differently from a normal one; why StatefulSet ordinal identity matters specifically for databases; what happens to a locally-provisioned volume when its node dies, vs. a network-attached one; when `emptyDir` is actually the right choice versus a real anti-pattern.

---

## Project 4 — Scale Under Load 🟠 (Topics 13, 14, 15, 18)

**Scenario:** The app goes semi-viral. Prove it scales gracefully — and that you'd actually know if it didn't.

**Build it:**
- An HPA on the Project 1/2 Deployment, driven by CPU.
- Real load (e.g., `hey`, `k6`, or `ab`) against the live Ingress endpoint, watching replicas climb in real time.
- One Grafana panel per golden signal — latency, traffic, errors, saturation — for this service, each backed by a real Prometheus metric you configured, not a placeholder.

**Break it:**
- Unset the Deployment's resource requests and watch the HPA refuse to work at all. Explain exactly why it can't function without them.
- Push load hard enough to hit a node's genuine CPU ceiling and watch the scheduler leave a replica `Pending` with nowhere to go — read the Events and explain the difference between this and a scheduling-constraint problem.
- Set HPA `minReplicas` unrealistically low and force a scale-down mid-load, then measure the dropped-request impact.
- Set a CPU *limit* noticeably below the app's real usage and generate load again. Using only Grafana panels and `kubectl top` — no application logs — diagnose why users report slowness while every pod still shows `Running`/`Ready`. Confirm afterward that what you found was throttling, not a genuine app slowdown.

**Observability checklist:** deliberately remove or misconfigure one pod's Prometheus scrape annotation, and confirm from Prometheus's own target list — not guesswork — that a missing/misconfigured scrape config, not a NetworkPolicy (you haven't covered those yet), is why it's absent.

**Interview angle:** the actual HPA formula (desired replicas from the current/target metric ratio); why HPA needs requests set; VPA vs. HPA vs. Cluster Autoscaler and when each applies; a `Pending` pod from scheduling constraints vs. one from raw capacity exhaustion; CPU throttling vs. an OOMKill as two entirely different resource-limit failure modes.

---

## Project 5 — Lock It Down: RBAC, SecurityContext & Pod Security Admission 🟠 (Topic 17)

**Scenario:** A second team joins and should only touch its own namespace. A CI robot account should be able to deploy — and nothing else. And every pod in the cluster should be running with the least privilege the Linux kernel allows, not just the least privilege the Kubernetes API allows.

**Build it:**
- Two Roles/RoleBindings scoping two ServiceAccounts to their own namespace only (pods + deployments: get/list/watch — no Secrets access).
- A dedicated CI ServiceAccount with exactly enough verbs to roll out a Deployment update, and nothing more — no `exec`, no Secret reads, no cluster-wide anything.
- Prove both restrictions with `kubectl auth can-i --as=<sa>` *before* attempting the real action.
- A SecurityContext (`runAsNonRoot`, dropped Linux capabilities, read-only root filesystem) applied to one of your app's pods. Confirm it still starts, then remove one setting at a time to find out which one actually breaks the app, and why.
- One namespace labeled with the `restricted` Pod Security Admission profile — first in `warn`/`audit` mode, so you can find every violation it flags and fix them, and only then flipped to `enforce`, with zero pods rejected on the flip.

**Break it:**
- Grant the CI account `cluster-admin`, then audit exactly what that exposes with `kubectl auth can-i --list --as=<sa>`. Write the one-line justification you'd need to defend that grant in a real access review — and notice how hard that is to write honestly.
- Exec into a pod that auto-mounted its default ServiceAccount token and see what that identity alone could reach if it had been over-permissioned. Then fix it by disabling `automountServiceAccountToken` wherever it isn't needed.
- On a throwaway namespace, skip the `warn`/`audit` step and flip straight to `restricted` `enforce` while it still has a root-requiring pod running. Watch the rejection at creation time, and explain exactly what a real rollout plan would have done differently.

**Security checklist:** this whole project *is* the checklist — least privilege proven with `auth can-i`, not assumed from memory; pod-level and container-level security settings proven by breaking each one, not copy-pasted from a blog post.

**Observability checklist:** find out whether your cluster currently has API audit logging enabled at all. If it doesn't, write that down as a real, standing finding.

**Interview angle:** Role vs. ClusterRole vs. RoleBinding vs. ClusterRoleBinding; how `kubectl auth can-i` actually resolves; why a compromised pod's default SA token is a genuine attack path; least privilege applied specifically to CI/CD identities; the practical difference between a pod-level and container-level SecurityContext; what the `restricted` Pod Security Admission profile blocks that `baseline` doesn't.

---

## Project 6 — The Background Work 🟠 (Topic 16 — DaemonSets, Jobs, CronJobs; Topic 18)

**Scenario:** Add the unglamorous but essential background machinery: nightly backups, a per-node log shipper, a one-off migration.

**Build it:**
- A CronJob that runs the Project 3 DB backup on a schedule, writing to a PVC.
- A Job that performs a one-time schema migration with a real `backoffLimit` and retry behavior.
- A DaemonSet running a lightweight log-shipping agent (Fluent Bit or similar) on every node — including tainted ones, via a correct toleration — forwarding container logs to one central destination you can actually query afterward.

**Break it:**
- Set the CronJob's `concurrencyPolicy` incorrectly and let two backup runs overlap. Diagnose the conflict from Job history, not guesswork.
- Force the migration Job to fail every run and watch it hit `backoffLimit` and stop. Explain precisely what "stop" means for a Job, and how that differs from what would happen to a Deployment in the same situation.
- Taint a node *after* the DaemonSet is already running there and confirm the existing DaemonSet pod is untouched. Explain why — it's a scheduling-time decision, not a live enforcement.
- Kill the log-shipper pod on one node for a few minutes and figure out, after restoring it, whether the logs generated during the gap were lost or just delayed — and how you'd actually prove which one happened.

**Interview angle:** Job vs. CronJob vs. Deployment restart semantics; `backoffLimit` and `activeDeadlineSeconds`; DaemonSet scheduling and tolerations; why DaemonSets bypass normal scheduling pressure for taints already tolerated at creation time; why a DaemonSet-per-node model for log shipping scales differently than a sidecar-per-pod model.

---

## Project 7 — Capstone: Everything, At Once, Breaking 🔴 (All of Topics 1–18)

**Scenario:** Merge Projects 1–6 into one 3-tier production-style app — frontend Deployment behind Ingress+TLS, backend API with an HPA, a StatefulSet database with scheduled backups, RBAC- and SecurityContext-hardened human and CI access, a DaemonSet log shipper, and real Prometheus/Grafana dashboards. Keep this running afterward as your permanent lab — your existing workbook's Section 21 will ask you to build something like this again, at even greater depth.

**Multi-fault game day** — each fault is independent; inject and diagnose them one at a time:

1. A ConfigMap edit silently breaks the backend's DB connection string — only the application logs reveal it, `kubectl get` shows nothing wrong.
2. Force the cert-manager certificate to expire. Ingress TLS starts failing while plain HTTP still works — recognize "half the stack is fine" as the actual symptom pattern.
3. Drain one node mid-load-test. Confirm the app survives with zero dropped requests, or find exactly why it doesn't — note explicitly that you don't yet have a PodDisruptionBudget (that's Topic 30, still ahead of you) and write that down as a real gap, not a failure of this exercise.
4. Delete the CI ServiceAccount's RoleBinding and simulate a pipeline deploy using that SA's own credentials. Diagnose it from the SA's point of view, not the cluster's.
5. Pre-seed seven failed CronJob runs so the backup has silently been failing for a week. Find it via Job history and Prometheus/kube-state-metrics — not by being told.

**Write-up requirement:** for at least two of the five faults, write a one-page incident summary — timeline, root cause, fix, one concrete action item — as if reporting to a lead.

**Interview angle (say this out loud, no notes):** narrate your entire stack's request path — client → Ingress → Service → Pod → database — then, for each hop, say exactly what would break there and how you'd detect it. This "trace the request, then trace the failure" format is the single most common systems-design question at this level.

---

## Final Interview Simulation — Rapid Mixed Round

Answer each in under 90 seconds, out loud, without opening a terminal:

1. A pod shows `Running` but `0/1 Ready`, and the Service isn't sending it traffic. Name your diagnostic commands, in the exact order you'd run them.
2. Walk through everything that happens between running `kubectl apply -f deployment.yaml` and a new pod serving traffic.
3. Why might doubling replica count *not* increase throughput at all?
4. What's the practical blast-radius difference between deleting a Namespace and deleting everything inside it one resource at a time?
5. A Deployment rollout is stuck at `2/3` updated replicas. What are the three most likely causes, and how do you tell them apart?
6. Explain the difference between a container restart and a pod eviction — and which one a Deployment's replica count "sees."
7. Your app works via `kubectl port-forward` but not through the Ingress. Narrow down where in the chain it's actually broken.
8. What does it mean for a Secret to be "not really encrypted," and what would you actually need to enable to change that?
9. Why does an HPA sit at its current replica count doing nothing, even though CPU usage is clearly high on `kubectl top pod`?
10. A StatefulSet pod is stuck `Pending` after a node failure. What's different about this compared to a Deployment pod in the same situation?
11. You're asked to justify why a workload needs `cluster-admin`. What questions do you ask before agreeing to it?
12. Give one concrete example each of a liveness probe misconfiguration and a readiness probe misconfiguration, and how their failure symptoms differ.
13. You edit a running pod's label so it no longer matches its ReplicaSet's selector. What happens to that pod, and what happens next in the cluster?
14. Narrate the exact sequence between `kubectl delete pod` and the container process actually stopping, and name the one setting most likely to make that sequence too slow for a rolling update to stay zero-downtime.

---

## What's Next

This lab deliberately stops at Topic 18 — no Helm, no NetworkPolicies, no PodDisruptionBudgets, because you haven't covered them yet, and a couple of the capstone faults above call that out explicitly as real gaps rather than papering over them. When you continue into Topics 19–23 of your existing workbook (Troubleshooting Playbook, Helm, the full Real-Time Production Project, the Best Practices Checklist, and the consolidated interview questions), come back and re-run Project 7 through Helm and then through GitOps — the underlying app won't change, but the operational surface area will, and that contrast is worth as much as the first build.

Keep the capstone cluster alive and keep adding faults to it over time — a lab that's been broken and fixed fifteen different ways is worth far more, for both CKA practice and interview readiness, than one built once and left alone.
