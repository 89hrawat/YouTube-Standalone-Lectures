# Kubernetes Rolling Updates & Rollbacks (Hands-On)

## Video reference for the lecture is the following:


---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents


- [Introduction](#introduction)  
- [What is a Deployment?](#what-is-a-deployment)  
- [How Deployments build on ReplicaSets](#how-deployments-build-on-replicasets)  
- [How a rolling update works (strategy: RollingUpdate)](#how-a-rolling-update-works-strategy-rollingupdate)  
- [What are annotations (vs. labels)?](#what-are-annotations-vs-labels)  
- [The `kubernetes.io/change-cause` annotation](#the-kubernetesiochange-cause-annotation)  
- [Demo: Zero-Downtime Rolling Updates with Deployments](#demo-zero-downtime-rolling-updates-with-deployments)  
  - [Step 1: Create `nginx-deploy` (v1, nginx 1.25)](#step-1-create-nginx-deploy-v1-nginx-125)  
  - [Step 2: Verify the Annotation and Revision](#step-2-verify-the-annotation-and-revision)  
  - [Step 3: Upgrade to nginx 1.26 (v2)](#step-3-upgrade-to-nginx-126-v2)  
  - [Step 4: Upgrade to nginx 1.27 (v3)](#step-4-upgrade-to-nginx-127-v3)  
  - [Step 5: Perform a Rollback (to v2)](#step-5-perform-a-rollback-to-v2)  
- [Common pitfalls (callouts)](#common-pitfalls-callouts)  
- [Conclusion](#conclusion)  
- [References](#references)  
---

## Introduction

Kubernetes **Deployments** are the standard way to run stateless workloads with safe, declarative updates. A Deployment manages one or more **ReplicaSets** and creates a **new ReplicaSet** whenever the Pod template changes (image, env, labels/annotations). That’s what enables controlled rollouts and easy rollbacks. We’ll focus on the **RollingUpdate** strategy and its two pacing knobs—`maxSurge` (temporary extra Pods) and `maxUnavailable` (allowed dip in availability)—and we’ll use the **`kubernetes.io/change-cause`** annotation so rollout history clearly shows what changed and why. In the demo, we’ll upgrade nginx from 1.25 → 1.26 → 1.27, verify progress with `kubectl rollout`, and practice a rollback to a known-good version.

---

## What is a Deployment?

A **Deployment** is the higher-level controller most teams use to run stateless apps in Kubernetes. It gives you **declarative updates** to Pods and the ReplicaSets that create them: you state the desired spec, and the Deployment controller rolls the cluster toward that state at a controlled pace.

**Why Deployments matter**

* Encapsulate rolling updates & rollbacks (via revision history).
* Handle automated Pod template updates (image, env, labels, etc.).
* Provide simple, safe **scale up/down** and status/health signals.

## How Deployments build on ReplicaSets

A Deployment **manages one active ReplicaSet at a time** and **creates a new ReplicaSet** whenever you change anything in the **Pod template** (`spec.template`). That’s how you get versioned “revisions.” Changes **outside** the Pod template (like just changing `.spec.replicas`) don’t create a new ReplicaSet. This is the mechanism that enables safe, incremental rollouts and clean rollbacks.

> **Expert note:** New ReplicaSets are keyed by a hash of the Pod template. Any template change (image tag, container args, annotations, labels, etc.) generates a new hash → new RS.

## How a rolling update works (strategy: RollingUpdate)

With strategy `RollingUpdate`, the Deployment controller:

1. **Creates** Pods in a brand-new ReplicaSet (the “new” version).
2. **Waits** for those Pods to become **Available** (readiness + optional `minReadySeconds`).
3. **Scales down** the old ReplicaSet proportionally.
4. Repeats until all Pods are on the new version.

Two knobs control the pacing:

* **`maxSurge`**: how many *extra* Pods you may run temporarily (absolute number or %).
* **`maxUnavailable`**: how many existing Pods may be unavailable during the rollout (number or %).

Your chosen values (`maxSurge: 1`, `maxUnavailable: 0`) ensure **no dip** below desired replicas; Kubernetes briefly runs one extra Pod during each step.

## What are annotations (vs. labels)?

**Annotations** are key-value metadata you attach to objects for **non-identifying, informational** purposes—used by tools, pipelines, or humans. Unlike labels (used for selection/grouping), annotations **don’t affect scheduling or selection**; they’re meant for context such as build IDs, Git SHAs, links to runbooks, or change descriptions.

Examples of useful annotation content:

* Build/commit info, image provenance, CI job URLs
* Ticket IDs / Change requests
* “Why” a change happened, by whom, and when

## The `kubernetes.io/change-cause` annotation

Kubernetes recognizes a conventional annotation, **`kubernetes.io/change-cause`**, to record the human-readable reason for a change. This text shows up in **`kubectl rollout history`** so you can see *why* each revision happened. The modern practice is to **set it explicitly** yourself.

**How it ties into history**

* The Deployment’s `kubernetes.io/change-cause` value is **copied to each new ReplicaSet** when that revision is created, and `kubectl rollout history` shows it per revision. (That’s why you set/update the annotation **as part of** the change you’re making.)

**Two easy ways to set it**

1. **Declarative (recommended for GitOps):** include it in your YAML alongside the change that creates a new RS (e.g., image bump).

   ```yaml
   metadata:
     annotations:
       kubernetes.io/change-cause: "Upgrade nginx from 1.25 to 1.26 for security update"
   ```

2. **Imperative (CLI):** annotate just before (or together with) your change:

   ```bash
   kubectl annotate deploy/nginx-rolling \
     kubernetes.io/change-cause="Upgrade nginx 1.25 → 1.26 (rolling)" --overwrite

   kubectl apply -f deployment-v2.yaml
   kubectl rollout history deploy/nginx-rolling
   ```

---

# Demo: Zero-Downtime Rolling Updates with Deployments

In this demo we’ll create a 3-replica Deployment running **nginx:1.25** (v1) with a **RollingUpdate** strategy (`maxSurge: 1`, `maxUnavailable: 0`) and a clear **`kubernetes.io/change-cause`**. We’ll then declaratively upgrade to **1.26** (v2) and **1.27** (v3), using `kubectl diff` + `apply`, and watch how the Deployment creates a new ReplicaSet, surges one Pod, and safely scales the old one down. Along the way we’ll verify progress with `kubectl rollout status`, inspect history with `kubectl rollout history`, and read Pod/RS labels to see versions live. We’ll finish with a **rollback to v2**, confirming the image and showing why the **change-cause** is a better audit breadcrumb than raw revision numbers.

---

## Step 1: Create `nginx-deploy` (v1, nginx 1.25)

**File:** `nginx-v1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx                         # Labels on the Deployment object (organizational)
  annotations:
    kubernetes.io/change-cause: "Initial release (v1) with nginx 1.25"
    # ^ Human-readable "why" — shows in `kubectl rollout history` and is copied to the RS.
spec:
  replicas: 3                          # Desired Pod count (default: 1)
  revisionHistoryLimit: 10             # Old ReplicaSets kept for rollback (default: 10)
  minReadySeconds: 5                   # Seconds Pod must be Ready before counted Available (default: 0)
  progressDeadlineSeconds: 300         # Mark rollout stalled after N seconds (default: 600)
  selector:
    matchLabels:
      app: nginx                       # Must match template.metadata.labels; don't change after create
  strategy:
    type: RollingUpdate                # Deployment strategy (default: RollingUpdate)
    rollingUpdate:
      maxSurge: 1                      # Extra Pods allowed during update (default: 25%)
      maxUnavailable: 0                # Allowed unavailable during update (default: 25%)
  template:
    metadata:
      labels:
        app: nginx                     # <-- Services select Pods by these labels (not Deployment labels)
        version: v1                    # Optional: helps visualize v1→v2→v3
    spec:
      containers:
        - name: nginx
          image: nginx:1.25            # v1 image (imagePullPolicy default for tagged images: IfNotPresent)
          # imagePullPolicy: IfNotPresent
```

Apply + basic verification:

```bash
kubectl apply -f nginx-v1.yaml
kubectl rollout status deploy/nginx-deploy
kubectl get deploy/nginx-deploy
kubectl get rs -l app=nginx
kubectl get pods -l app=nginx -o wide
```

> **Expert note:** `minReadySeconds` is most effective with a `readinessProbe`. We’re skipping probes to keep the demo compact.

---

## Step 2: Verify the Annotation and Revision

Show history:

```bash
kubectl rollout history deployment nginx-deploy
```

Expected:

```
deployment.apps/nginx-deploy
REVISION  CHANGE-CAUSE
1         Initial release (v1) with nginx 1.25
```

* If `kubernetes.io/change-cause` wasn’t set, **CHANGE-CAUSE would be empty**.
* You can also see the revision number on the ReplicaSet:

```bash
kubectl get rs -l app=nginx -o custom-columns=RS:.metadata.name,REV:.metadata.annotations.deployment\.kubernetes\.io/revision,CAUSE:.metadata.annotations.kubernetes\.io/change-cause
```

> **Why this matters:** Revision numbers are **not stable identifiers** (a rollback creates a *new* revision). The **change-cause** tells the human story (“what’s in this version”).

---

## Step 3: Upgrade to nginx 1.26 (v2)

**File:** `nginx-v2.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
  annotations:
    kubernetes.io/change-cause: "v2 with nginx 1.26"   # Update the cause for this change
spec:
  replicas: 3
  revisionHistoryLimit: 10
  minReadySeconds: 5
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
        version: v2                                    # Bump version label (optional but visible)
    spec:
      containers:
        - name: nginx
          image: nginx:1.26                            # v2 image
```

Diff → apply → watch:

```bash
kubectl diff -f nginx-v2.yaml
kubectl apply -f nginx-v2.yaml
kubectl rollout status deploy/nginx-deploy
kubectl get pods -l app=nginx -w        # watch +1 v2, -1 v1 pattern in real time
```

History now:

```
REVISION  CHANGE-CAUSE
1         Initial release (v1) with nginx 1.25
2         v2 with nginx 1.26
```

> **Mechanics recap:** Deployment creates a **new RS** for v2, surges +1 Pod, waits until it’s **Available** (readiness + `minReadySeconds`), then scales the old RS down by 1. Repeats until all are v2.

---

## Step 4: Upgrade to nginx 1.27 (v3)

**File:** `nginx-v3.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
  annotations:
    kubernetes.io/change-cause: "v3 with nginx 1.27"
spec:
  replicas: 3
  revisionHistoryLimit: 10
  minReadySeconds: 5
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
        version: v3
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

Diff → apply → verify:

```bash
kubectl diff -f nginx-v3.yaml
kubectl apply -f nginx-v3.yaml
kubectl rollout status deploy/nginx-deploy
kubectl rollout history deploy/nginx-deploy
```

Expected:

```
REVISION  CHANGE-CAUSE
1         Initial release (v1) with nginx 1.25
2         v2 with nginx 1.26
3         v3 with nginx 1.27
```

---

## Step 5: Perform a Rollback (to v2)

List history, then rollback:

```bash
kubectl rollout history deployment nginx-deploy

kubectl rollout undo deployment nginx-deploy --to-revision=2
# Output: deployment.apps/nginx-deploy rolled back
```

Check history again:

```
REVISION  CHANGE-CAUSE
1         Initial release (v1) with nginx 1.25
3         v3 with nginx 1.27
4         v2 with nginx 1.26
```

**Key learning:** The rollback **creates a new revision (`4`)** with the old template.
Revision **2 no longer exists**; this is why **you should not rely on revision numbers alone**. Use the **CHANGE-CAUSE** to understand what each revision represents.

Verify the actual image:

```bash
kubectl describe deployment nginx-deploy | sed -n '/Image:/p;/OldReplicaSets:/,/NewReplicaSet:/p'
# or, precise:
kubectl get deploy nginx-deploy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

You should see `nginx:1.26`.

---

## Common pitfalls (callouts)

* **Changing `.spec.selector`** on a live Deployment will break ownership and usually fails; **don’t** change it after creation.
* **No readiness probe** → Pods often flip Ready immediately after start; `minReadySeconds` still helps, but probes are **strongly recommended** in prod.
* **Stalled rollout** (bad tag, crash loop) → after `progressDeadlineSeconds`, Deployment is marked **ProgressDeadlineExceeded**. It does **not** auto-cancel/rollback; fix the issue or `rollout undo`.

  ```bash
  kubectl get deploy nginx-deploy -o jsonpath='{.status.conditions}'
  kubectl describe deploy nginx-deploy
  ```
* **History pruning** → older ReplicaSets are removed when they exceed `revisionHistoryLimit`; keep it high enough to support your rollback window.

---

## Conclusion

* A **Deployment** achieves safe updates by **creating a new ReplicaSet** for the changed template and **scaling the new RS up while scaling the old RS down**.
* **`maxSurge`** sets the ceiling for temporary extra Pods; **`maxUnavailable`** sets the availability floor. Together they control rollout **pace and safety**.
* **Revision numbers aren’t stable identifiers**—a rollback creates a fresh revision—so always populate **`kubernetes.io/change-cause`** to keep `kubectl rollout history` meaningful.
* True **zero-downtime** requires both the Kubernetes side (readiness, graceful termination, surge/unavailable) and **app design** (session/state handling, compatible changes, LB draining).
* Practical muscle memory: **diff → apply → watch status → verify RS/Pods → history → rollback**—the same sequence you’ll automate in CI/CD.

---

# References

* Kubernetes Docs — **Deployments (concepts)**: overview, template changes → new ReplicaSet, rolling updates. ([Kubernetes][1])
* Kubernetes Docs — **ReplicaSet (concepts)**: purpose and relationship to Deployments. ([Kubernetes][4])
* Kubernetes Docs — **Rolling update (basics tutorial)**: incremental replacement, zero-downtime goal, readiness gating. ([Kubernetes][3])
* Kubernetes Docs — **kubectl rollout** & **rollout history**: view revisions, inspect details, undo. ([Kubernetes][2])
* Kubernetes Docs — **Annotations (concepts)**: what annotations are vs labels (non-identifying metadata). ([Kubernetes][5])
* Kubernetes Docs — **Well-Known Labels/Annotations**: `kubernetes.io/change-cause` (how it’s populated/used). ([Kubernetes][6])

---
