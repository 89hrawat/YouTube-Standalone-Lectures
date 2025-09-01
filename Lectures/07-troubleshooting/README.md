# Kubernetes Troubleshooting MASTERCLASS | Control Plane, Data Plane & Pod Failures

## Video reference for the lecture is the following:

[![Watch the video](https://img.youtube.com/vi/81nnmV4iXw4/maxresdefault.jpg)](https://www.youtube.com/watch?v=81nnmV4iXw4&ab_channel=CloudWithVarJosh)

---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

* [Introduction](#introduction)
* [Control-Plane Basics (kubeadm clusters)](#control-plane-basics-kubeadm-clusters)
* [Pre-Requisites for Control Plane](#prerequisites-for-control-plane-lecture)
* [kubeconfig & contexts (quick refresher)](#kubeconfig--contexts-quick-refresher)
* [Control-Plane Troubleshooting Scenarios (exam-style)](#control-plane-troubleshooting-scenarios-exam-style)
* [1) API server unreachable / `kubectl` times out](#1-api-server-unreachable--kubectl-times-out)
  * [Tooling primer: `ss` and the flags you’re using](#tooling-primer-ss-and-the-flags-youre-using)
  * [Established connections (remove `-l`)](#established-connections-remove--l)
* [2) etcd unhealthy → API server crashloop](#2-etcd-unhealthy--api-server-crashloop)
* [3) Pods stay **Pending** — scheduler unavailable *or* no feasible node](#3-pods-stay-pending--scheduler-unavailable-or-no-feasible-node)
* [4) Controller-Manager down → controllers inactive (no replicas, no GC)](#4-controller-manager-down--controllers-inactive-no-replicas-no-gc)
* [5) Certificates expired (very common exam task)](#5-certificates-expired-very-common-exam-task)
  * [After cert renewal: controller-manager & scheduler NotReady](#after-cert-renewal-controller-manager--scheduler-notready) 
* [Prerequisites for Data-Plane](#prerequisites-for-data-plane-lecture)  
  * [Note for viewers: Data-plane & prerequisites](#note-for-viewers-data-plane--prerequisites)  
* [Quick refresher: the data-plane pieces](#quick-refresher-the-data-plane-pieces)  
* [Read This First: How to Use the Scenarios](#read-this-first-how-to-use-the-scenarios)  
  * [1) Node NotReady / kubelet not registered](#1-node-notready--kubelet-not-registered)  
  * [2) Pods stuck ContainerCreating / CNI not initialized](#2-pods-stuck-containercreating--cni-not-initialized)  
  * [3) Pod-to-Pod across nodes / Service ClusterIP not reachable](#3-pod-to-pod-across-nodes--service-clusterip-not-reachable)  
  * [4) DNS failures (CoreDNS)](#4-dns-failures-coredns)  
  * [5) ImagePullBackOff / ErrImagePull](#5-imagepullbackoff--errimagepull)  
  * [6) CrashLoopBackOff / OOMKilled / probe failures](#6-crashloopbackoff--oomkilled--probe-failures)  
  * [7) PVC Pending / volume mount errors (CSI)](#7-pvc-pending--volume-mount-errors-csi)  
  * [8) NodePort / LoadBalancer not reachable](#8-nodeport--loadbalancer-not-reachable)    
* [Pod Deletion and Termination Signals](#pod-deletion-and-termination-signals)
    - [What Happens When You Run `kubectl delete pod`](#what-happens-when-you-run-kubectl-delete-pod)
    - [Force Delete a Pod](#force-delete-a-pod)
* [Restart Policies](#restart-policies)
* [Image Pull Policies](#image-pull-policies)
* [Pod Lifecycle Phases](#pod-lifecycle-phases)
    - [Pending](#1-pending)
    - [Running](#2-running)
    - [Succeeded](#3-succeeded)
    - [Failed](#4-failed)
    - [Unknown](#5-unknown)
    - [Pod Status in Multi-Container Pods](#pod-status-in-multi-container-pods)
    - [Differentiation Between Succeeded and Completed](#differentiation-between-succeeded-and-completed)
    - [Summary Table](#summary-table)
    - [Demo 1: Observing Short-Lived Pod Lifecycle](#demo-1-observing-short-lived-pod-lifecycle)
    - [Demo 2: Observing Longer Pod Lifecycle](#demo-2-observing-longer-pod-lifecycle)
    - [Quick Debugging Workflow](#quick-debugging-workflow)
* [Pod Status Deep Dive](#pod-status-deep-dive)
* [Debugging Common Pod Errors](#debugging-common-pod-errors)
* [Conclusion](#conclusion)

---

## Introduction

Welcome. This guide is a **hands on troubleshooting masterclass** for Kubernetes across the **control plane**, the **data plane**, and **pods**. We **break clusters on purpose** and then **fix them step by step**, so you see the exact checks and commands that matter in real incidents.

You will begin with **control plane basics on kubeadm clusters** plus a quick refresher on **kubeconfig and contexts**, then work through **exam style scenarios** for the **API server**, **etcd**, the **scheduler**, the **controller manager**, and **certificates**. After that we shift to the **data plane**, where most on call issues appear, and finish with **pod lifecycle**, **restart and image pull policies**, and **common error patterns** you will meet in practice.

This masterclass is a **single cut that stitches Day 22, Day 57, and Day 58** from my CKA series into one flow. It also stands on its own. You **do not need to jump across individual days** because the **combined GitHub notes** are linked here. If you hear a day number, it simply points to the CKA playlist sequence for learners who prefer to follow the full series.

---

### Prerequisites for Control Plane lecture

* **Day 7 — Kubernetes Architecture**

  * [Watch on YouTube](https://www.youtube.com/watch?v=-9Cslu8PTjU&ab_channel=CloudWithVarJosh)
  * [View GitHub repo](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2007)

* **Day 15 — Manual Scheduling & Static Pods**

  * [Watch on YouTube](https://www.youtube.com/watch?v=moZNbHD5Lxg&ab_channel=CloudWithVarJosh)
  * [View GitHub repo](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2015)

* **TLS in Kubernetes — Masterclass**

  * [Watch on YouTube](https://www.youtube.com/watch?v=J2Rx2fEJftQ&list=PLmPit9IIdzwQzlveiZU1bhR6PaDGKwshI&index=1&t=3371s&pp=gAQBiAQB)
  * [View GitHub repo](https://github.com/CloudWithVarJosh/TLS-In-Kubernetes-Masterclass)

---

## Control-Plane Basics (kubeadm clusters)

* **Who “schedules” control-plane pods?**
  They’re **static pods** started directly by the **kubelet**, not the scheduler. Manifests live in **`/etc/kubernetes/manifests/`** (`kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`, `etcd.yaml`). The kubelet **watches this folder** (`staticPodPath`) and (re)creates the pods from those files. If you delete a static pod from the API, it **comes back**—the file is the source of truth.

* **Why do static pods show up in `kubectl get pods`? → Mirror pods.**
  For every static pod it runs, the kubelet **publishes a read-only “mirror pod”** object to the API server so you can see status in `kubectl`.

  * **Name pattern:** `<podName>-<nodeName>` (e.g., `kube-apiserver-control-plane`).
  * **Behavior:** You can **see** status/logs via the mirror pod, but edits/deletes don’t change the real static pod. To modify/remove it, **edit/delete the file** in `/etc/kubernetes/manifests/`.
  * **Tip:** Mirror pods typically carry a mirror annotation and always show which **node** hosts the underlying static pod, making node mapping obvious.


---

## kubeconfig & contexts (quick refresher)

* Your `kubectl` uses a **kubeconfig** (default `~/.kube/config`) that defines **clusters**, **users** (credentials), and **contexts** (cluster+user+namespace).
* Common files on a kubeadm control plane:

  * `/etc/kubernetes/admin.conf` (admin kubeconfig)
  * Copy it to your workstation as `~/.kube/config` to manage the cluster.
* Handy commands:

  ```bash
  kubectl config get-contexts
  kubectl config use-context <name>
  kubectl config view --minify
  # set a default namespace in the current context
  kubectl config set-context --current --namespace dev
  ```

---

# Control-Plane Troubleshooting Scenarios (exam-style)

---

## 1) API server unreachable / `kubectl` times out

**Problem**
`kubectl get nodes` hangs or shows `connection refused` / `i/o timeout` to port **6443**.

**Likely causes**

* Bad flags / paths in `/etc/kubernetes/manifests/kube-apiserver.yaml` (certs, keys, `--advertise-address`, `--client-ca-file`, admission plugins).
* Port **6443** not listening or occupied by another process.
* Host firewall / cloud security group blocking **6443**.
* etcd unreachable (see your etcd scenario).

**Checks**

```bash
# Is the API server actually listening on 6443? (socket statistics)
sudo ss -lntp | grep 6443

# Is the apiserver container running or crash-looping? (CRI runtime view)
sudo crictl ps -a | grep kube-apiserver

# What is the apiserver complaining about? (log tail from CRI files)
sudo tail -n 50 /var/log/containers/kube-apiserver-*.log

# Any typos/wrong paths/addresses in the static pod manifest?
sudo sed -n '1,120p' /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Fix**

* Correct wrong cert/key paths (under `/etc/kubernetes/pki/`), fix `--advertise-address` to the node’s reachable IP, clean up `--enable-admission-plugins`.
* Save the manifest; **kubelet auto-restarts** the static pod.
* Open/unblock **6443** in host firewall / security groups.

**Example**
`--advertise-address=127.0.0.1` in a multi-node cluster → remote `kubectl` can’t reach. Change to the control-plane node IP (e.g., `172.31.x.x`) and the API becomes reachable.

---

## Tooling primer: `ss` and the flags you’re using

**`ss` = socket statistics** (shows listening sockets and live connections).

Common flags you’ll use:

* `-t` → TCP only
* `-l` → **listening** sockets (servers waiting for connections)
* `-p` → show owning **process**
* `-n` → numeric output (skip DNS lookups)
* `-s` → summary totals

### Reading `ss -tlpns` (listening TCP sockets)

```
State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port  Process
LISTEN  0       4096         127.0.0.1:9099         0.0.0.0:*      users:(("calico-node",pid=2752,fd=6))
LISTEN  0       4096         127.0.0.1:9098         0.0.0.0:*      users:(("calico-typha",pid=1578,fd=7))
LISTEN  0       4096         127.0.0.1:2381         0.0.0.0:*      users:(("etcd",pid=1183,fd=14))
LISTEN  0       4096         127.0.0.1:2379         0.0.0.0:*      users:(("etcd",pid=1183,fd=7))
LISTEN  0       4096         127.0.0.1:10249        0.0.0.0:*      users:(("kube-proxy",pid=1538,fd=18))
LISTEN  0       4096         127.0.0.1:10248        0.0.0.0:*      users:(("kubelet",pid=927,fd=14))
LISTEN  0       4096         127.0.0.1:10257        0.0.0.0:*      users:(("kube-controller",pid=4600,fd=3))
LISTEN  0       4096         127.0.0.1:10259        0.0.0.0:*      users:(("kube-scheduler",pid=4561,fd=3))
LISTEN  0       4096         127.0.0.1:34961        0.0.0.0:*      users:(("containerd",pid=576,fd=10))
LISTEN  0       4096        127.0.0.54:53           0.0.0.0:*      users:(("systemd-resolve",pid=350,fd=17))
LISTEN  0       4096           0.0.0.0:22           0.0.0.0:*      users:(("sshd",pid=3924,fd=3),("systemd",pid=1,fd=87))
LISTEN  0       4096     172.31.35.156:2380         0.0.0.0:*      users:(("etcd",pid=1183,fd=6))
LISTEN  0       4096     172.31.35.156:2379         0.0.0.0:*      users:(("etcd",pid=1183,fd=8))
LISTEN  0       4096     127.0.0.53%lo:53           0.0.0.0:*      users:(("systemd-resolve",pid=350,fd=15))
LISTEN  0       4096                 *:5473               *:*      users:(("calico-typha",pid=1578,fd=6))
LISTEN  0       4096              [::]:22              [::]:*      users:(("sshd",pid=3924,fd=4),("systemd",pid=1,fd=88))
LISTEN  0       4096                 *:10256              *:*      users:(("kube-proxy",pid=1538,fd=17))
LISTEN  0       4096                 *:10250              *:*      users:(("kubelet",pid=927,fd=16))
LISTEN  0       4096                 *:6443               *:*      users:(("kube-apiserver",pid=7103,fd=3))
```

**Column meanings**

* **Local Address\:Port** → where **this VM** is listening (server side).
* **Peer Address\:Port** on a LISTEN row is **always** `0.0.0.0:*` (no specific peer yet).
* **Recv-Q / Send-Q** → current queued vs backlog capacity.

### Local vs Peer (quick mental model)

* **Local** = “where *I* am listening/connected.”
* **Peer** on a LISTEN row is wildcard (`0.0.0.0:*`) because there’s **no remote yet**. Firewalls/SGs still enforce who can actually connect.

---

## Established connections (remove `-l`)

To see who’s **actually talking** to whom, drop `-l`:

```bash
# Established TCP sessions only
ss -tnp state established | grep -i 172.31.44.224
# Local is your VM’s endpoint; Peer is the remote node/port.
```

**Example (control-plane view)**

```
[::ffff:172.31.35.156]:6443  [::ffff:172.31.44.224]:32192  # worker → apiserver
[::ffff:172.31.35.156]:10250 [::ffff:172.31.44.224]:40342  # worker/pod → control-plane kubelet
[::ffff:172.31.35.156]:5473  [::ffff:172.31.44.224]:47678  # calico felix ↔ typha
```

**Example (worker-1 view)**

```
172.31.44.224:34778   172.31.35.156:6443   # worker client → apiserver
172.31.44.224:10250   192.168.226.86:45078 # kubelet server → scraper/client
172.31.44.224:47662   172.31.35.156:5473   # calico felix → typha
172.31.44.224:22      223.228.207.145:35358# your SSH session
```

**Reading these lines**

* If the **Local** port is a well-known server port (e.g., **6443**, **10250**), that row is the **server side** on your VM.
* If the **Peer** port is the well-known one, your VM is the **client** to a remote server.

---

## 2) etcd unhealthy → API server crashloop

**Problem**
`kube-apiserver` logs show timeouts to **127.0.0.1:2379**, or the **etcd** pod is `CrashLoopBackOff`.

### Likely causes → How to fix

* **Wrong `--data-dir` / missing hostPath mount**
  *Fix:* Point `--data-dir` back to the host-mounted path (kubeadm default **/var/lib/etcd**) and ensure the volumeMount covers it; perms **root\:root 0700**.

* **Removed localhost from client URLs** (`--listen-client-urls` / `--advertise-client-urls`)
  *Fix:* For kubeadm “stacked” etcd, keep **[https://127.0.0.1:2379](https://127.0.0.1:2379)** or update `kube-apiserver` `--etcd-servers` to the node IP and ensure cert SANs match.

* **TLS cert/key path or SAN mismatch**
  *Fix:* Correct paths in `etcd.yaml`; if SANs are wrong, **renew etcd certs** with kubeadm (or reissue with proper SANs).

* **Disk full / slow I/O on etcd data dir**
  *Fix:* Free space on the filesystem hosting `/var/lib/etcd`; investigate I/O stalls; once healthy, consider `etcdctl defrag` if DB is large/fragmented.

* **Firewall/Security group blocks** (multi-node/HA etcd)
  *Fix:* Allow **2379/tcp** (client) and **2380/tcp** (peer) between members/control plane.

* **Clock skew** (TLS and raft can misbehave)
  *Fix:* Sync time (chrony/systemd-timesyncd) so nodes are within a few seconds.

### Quick checks (with what they tell you)

```bash
sudo crictl ps -a | grep etcd                    # etcd container state (running/crashing)
sudo tail -n 100 /var/log/containers/etcd-*.log  # recent etcd logs for path/TLS/disk errors
sudo ss -lntp | grep -E ':2379|:2380'            # who’s listening on etcd ports (server sockets)
df -h                                            # disk full?
sudo sed -n '1,160p' /etc/kubernetes/manifests/etcd.yaml | \
  grep -E '--data-dir|listen-client-urls|advertise-client-urls'
# ^ effective flags (data dir & URLs)

# etcd health (kubeadm defaults)
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
etcdctl endpoint health                          # basic liveness
etcdctl endpoint status --write-out=table        # DB size, leader, alarms
```

### Symptom clues in logs (to pinpoint root cause)

* **Bad data dir / not mounted:** lines showing `"data-dir":"…"` and `"opened backend db","path":"…"` under an unexpected path; tiny DB + messages like “started as single-node” imply a **fresh/empty store**.
* **TLS issues:** `x509` errors, “no such file or directory” for cert/key paths.
* **Connectivity mismatch:** apiserver logs show timeouts to `127.0.0.1:2379` while etcd listens only on node IP (or vice versa).
* **Disk/I/O:** “no space left on device”, long WAL sync, fs errors.

That’s enough to **identify** the likely issue from the logs and **apply the minimal fix** quickly during the recording.

---

## 3) Pods stay **Pending** — scheduler unavailable *or* no feasible node

**Problem**
New Pods remain **Pending** and you don’t see a `Scheduled` event.

### Fast triage (20 seconds)

```bash
kubectl get pod <p> -o wide                     # NODE empty => unscheduled
kubectl describe pod <p> | sed -n '/Events:/,$p' # shows "0/N nodes are available: …" reasons
kubectl -n kube-system get pods | grep scheduler # scheduler pod health (static pod on kubeadm)
```

---

### A) Scheduler **unavailable**

**Likely causes**

* Bad `--kubeconfig` in `/etc/kubernetes/manifests/kube-scheduler.yaml`.
* Health/secure port mis-set (e.g., `--secure-port=0`); liveness probe fails.
* Clock skew breaks leader election.

**Checks**

```bash
# Show scheduler logs (works even if pod name changes)
kubectl -n kube-system logs -l component=kube-scheduler --tail=80

# Inspect effective flags
sudo sed -n '1,160p' /etc/kubernetes/manifests/kube-scheduler.yaml | \
  grep -E 'kubeconfig|secure-port|port'

timedatectl status   # NTP/time sync
```

**Fix**

* Point `--kubeconfig` to `/etc/kubernetes/scheduler.conf` and ensure certs exist.
* Restore `--secure-port=10259` (so liveness/healthz works).
* Sync time (chrony/systemd-timesyncd); wait for leader election to settle.

*Example:* `--kubeconfig` pointed to a wrong path → scheduler can’t auth to API → pods never schedule. Revert the path.

---

### B) Scheduler **running** but can’t place the Pod

**Common blockers**

* **Insufficient resources** (CPU/mem/ephemeral-storage vs *requests*).
* **Taints without tolerations** (e.g., control-plane taint).
* **Label/affinity/topology** mismatch (nodeSelector/affinity/anti-affinity/spread).
* **PVC Pending** / storage topology (SC, access mode, zone).
* **Extended resources** requested (GPU, hugepages) but not present.

**Checks**

```bash
kubectl describe pod <p> | sed -n '/Events:/,$p'  # exact text like:
# "0/N nodes are available: Insufficient cpu; node(s) had taint {...}; node affinity/selector mismatch …"

kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
kubectl get pvc                                   # any volumes Pending?
```

**Fix**

* Lower requests or free capacity / add nodes.
* Add tolerations or remove/adjust taints.
* Correct labels/affinity/spread constraints.
* Fix StorageClass/zone/access mode so PVC binds.

---

## 4) Controller-Manager down → controllers inactive (no replicas, no GC)

**Problem**
You scale or create a Deployment, but **no new Pods appear**. Other controller actions stall: **Endpoints/EndpointSlices empty**, **CSR approvals** hang, **node lifecycle taints** don’t update, **garbage collection** (ownerReferences) doesn’t run.

### Likely causes

* `kube-controller-manager` **not running** / stuck **CrashLoopBackOff** / **not leader**.
* Bad **`--kubeconfig`** in `/etc/kubernetes/manifests/kube-controller-manager.yaml`.
* Health port broken (e.g., `--secure-port=0`) so **liveness probe fails**.
* Missing/mispointed **cluster-signing cert/key** (`--cluster-signing-cert-file`, `--cluster-signing-key-file`).
* Significant **clock skew** breaking leader election.

### Checks (what each tells you)

```bash
kubectl -n kube-system get pods -l component=kube-controller-manager
# Is the controller-manager pod healthy or crashlooping?

kubectl -n kube-system logs -l component=kube-controller-manager --tail=80
# Auth/flag errors (bad kubeconfig), probe failures, TLS/path issues.

sudo sed -n '1,160p' /etc/kubernetes/manifests/kube-controller-manager.yaml | \
  grep -E 'kubeconfig|secure-port|cluster-signing|feature-gates'
# Verify critical flags/paths and secure port.

kubectl get leases -n kube-system | grep controller-manager
# Is there an active leader election lease?

sudo ss -lntp | grep 10257
# Controller-manager health/secure port listening?
```

**Prove reconciliation is stalled**

```bash
kubectl create deploy cm-test --image=nginx --replicas=1
kubectl scale deploy cm-test --replicas=3
kubectl get deploy cm-test ; kubectl get rs,pods -l app=cm-test
# Desired=3 but ReplicaSet/Pods don’t increase while CM is down.

kubectl describe deploy cm-test | sed -n '/Events:/,$p'
# No "Scaled up replica set ..." events.
```

### Fix

* Restore **`--kubeconfig=/etc/kubernetes/controller-manager.conf`** and ensure the referenced certs exist.
* Restore **`--secure-port=10257`** so the liveness probe succeeds.
* Point **cluster-signing** flags to valid files under `/etc/kubernetes/pki/`.
* **Sync time** (chrony/systemd-timesyncd) if leader election is flapping.
* Save manifest—**kubelet auto-restarts** the static pod; watch leadership resume.

**Example**
`--cluster-signing-key-file` missing → CSRs remain **Pending**, new nodes can’t join. **Fix:** restore the key or correct the path; controller-manager restarts, CSR approvals proceed, and Deployments start scaling again.


Here’s a **single, stitched, exam-style** section you can drop into your repo. I’ve added **inline comments** to every command so viewers can follow along.

---

## 5) Certificates expired (very common exam task)

**Problem**
`x509: certificate has expired` in API server / kubelet logs; nodes flip **NotReady** and `kubectl` fails TLS.

**Likely causes**

* kubeadm-managed certs passed their **NotAfter** date.
* **Clock skew** makes otherwise-valid certs appear expired or not-yet-valid.

---

### Quick Checks (what’s wrong?)

```bash
sudo kubeadm certs check-expiration                 # kubeadm summary of all control-plane cert expiries
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -enddate                                   # print API server cert NotAfter (expiry)
journalctl -u kubelet -p err --since "2h" | grep -i x509   # search recent kubelet errors for x509 messages
```

---
### Break (demo) — simulate expiry by jumping time

```bash
# Stop time sync so the clock won’t snap back automatically (pick whatever exists)
sudo systemctl stop systemd-timesyncd 2>/dev/null || true   # stop systemd-timesyncd if present
sudo systemctl stop chronyd 2>/dev/null || true             # stop chrony if present

# Jump node clock past current cert validity to force x509 failures
sudo date -s '2027-12-01 12:00:00'                          # set future date/time for the demo

# (Optional) Re-check to see the effect in tooling/logs
sudo kubeadm certs check-expiration                         # shows “expired” due to future clock
journalctl -u kubelet -p err --since "15m" | grep -i x509   # kubelet now logs cert errors
```

**Symptom you’ll show:** `kubectl` and kubelet report **expired / not yet valid** certs; nodes turn **NotReady**.

---

### Fix (exam way) — rotate with kubeadm, refresh kubeconfigs

```bash
# Renew all control-plane certs (server + client certs used by components)
sudo kubeadm certs renew all

# Re-generate kubeconfigs that embed client certs (admin, scheduler, controller-manager)
sudo kubeadm init phase kubeconfig all

# Restart kubelet so static pods (apiserver, etcd, CM, scheduler) reload new certs
sudo systemctl restart kubelet

# Use the fresh admin.conf for your kubectl (helpful on the control-plane node)
mkdir -p ~/.kube                                          # ensure kube config dir
sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config      # copy new admin kubeconfig
sudo chown $(id -u):$(id -g) ~/.kube/config               # fix ownership for current user

# Verify cluster health after rotation
kubectl get nodes                                         # should return Ready once components settle
sudo kubeadm certs check-expiration                       # confirm fresh NotAfter dates
```

---

### Optional cleanup — restore real time **then re-renew** (important!)

> If you renewed while the clock was in the future, the new certs have **NotBefore in 2027**.
> After resetting time, **renew again** so certs are valid *now*.

```bash
# Restore correct time (enable exactly one time service)
sudo date -s 'NOW'                                        # snap back to current time
sudo systemctl start systemd-timesyncd 2>/dev/null || true # start systemd-timesyncd if used
sudo systemctl start chronyd 2>/dev/null || true          # or start chrony if that’s your NTP agent

# Re-issue certs with correct time (only needed if you renewed while clock was future)
sudo kubeadm certs renew all                              # mint certs with proper NotBefore/NotAfter
sudo kubeadm init phase kubeconfig all                    # refresh kubeconfigs again
sudo systemctl restart kubelet                            # reload components

# Use the fresh admin.conf for your kubectl (helpful on the control-plane node)
mkdir -p ~/.kube                                          # ensure kube config dir
sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config      # copy new admin kubeconfig
sudo chown $(id -u):$(id -g) ~/.kube/config               # fix ownership for current user

# Final verification
kubectl get nodes                                         # Ready again
sudo kubeadm certs check-expiration                       # dates look sane (no future NotBefore)
```
---

## After cert renewal: controller-manager & scheduler NotReady

### Why this happens

1. **Time jump + reissued certs**
   When you renew certs while the clock is wrong (or flip time back), the controllers may fail to authenticate (NotBefore/NotAfter mismatch) until you **re-renew** with correct time and regenerate kubeconfigs. That part you already fixed.

2. **Stale containers in containerd after rapid restarts**
   The controller static pods get restarted many times during cert rotation. Occasionally **containerd leaves a dead container record** that still “owns” the name. Kubelet then can’t create the new container and you see:

   ```
   Error: failed to reserve container name "...": name "..._kube-system_..." is reserved for "<old-container-id>"
   ```

   This is a runtime bookkeeping issue, not an image or manifest bug.

---

### Fixing controller-manager

```bash
# List all controller-manager containers (running/exited)
sudo crictl ps -a | grep -i kube-controller-manager

# Force-remove any stuck/old container IDs you see
sudo crictl rm -f <CONTAINER_ID_1> <CONTAINER_ID_2>

# (Optional) if a pod sandbox is also stuck, remove it too:
sudo crictl pods | grep -i kube-controller-manager
sudo crictl stopp <POD_SANDBOX_ID>; sudo crictl rmp <POD_SANDBOX_ID>

# Nudge kubelet to resync static pods
sudo systemctl restart kubelet
```

**Verify:**

```bash
kubectl -n kube-system get pods -l component=kube-controller-manager
kubectl -n kube-system logs pod/<controller-manager-pod> --tail=50
kubectl -n kube-system get lease | grep controller-manager   # leader election healthy
```

---

### Fixing scheduler

Same pattern:

```bash
sudo crictl ps -a | grep -i kube-scheduler
sudo crictl rm -f <CONTAINER_IDs>
sudo crictl pods | grep -i kube-scheduler && \
  sudo crictl stopp <POD_SANDBOX_ID>; sudo crictl rmp <POD_SANDBOX_ID>
sudo systemctl restart kubelet
```

**Verify:**

```bash
kubectl -n kube-system get pods -l component=kube-scheduler
kubectl -n kube-system logs pod/<kube-scheduler-pod> --tail=50
kubectl -n kube-system get lease | grep scheduler
```

---

### Also double-check (common after cert work)

```bash
# Ensure kubeconfigs point to fresh certs
sudo kubeadm init phase kubeconfig all

# Confirm cert dates & that API auth works now
sudo kubeadm certs check-expiration
kubectl get nodes
```

---

### TL;DR

* The “**name is reserved for <ID>**” error = **stale containerd state** after many rapid restarts.
* **crictl rm -f** the old containers (and sandbox if needed), then **restart kubelet**.
* Ensure you **re-renewed certs with the correct system time** and regenerated the controller/scheduler kubeconfigs.

---

### Prerequisites for Data Plane lecture

* **Day 7 — Kubernetes Architecture**  
  [Watch on YouTube](https://www.youtube.com/watch?v=-9Cslu8PTjU&ab_channel=CloudWithVarJosh)  
  [View GitHub repo](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2007)  

* **Day 22 — Pod Termination, Restart Policies, Image Pull Policy, Lifecycle & Common Errors**  
  [Watch on YouTube](https://www.youtube.com/watch?v=miZl-7QI3Pw&ab_channel=CloudWithVarJosh)  
  [View GitHub repo](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2022)  

* **Day 48 — Kubernetes DNS Explained**  
  [Watch on YouTube](https://www.youtube.com/watch?v=9TosJ-z9x6Y&ab_channel=CloudWithVarJosh)  
  [View GitHub repo](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2048)  

* **TLS in Kubernetes — Masterclass**  
  [Watch on YouTube](https://www.youtube.com/watch?v=J2Rx2fEJftQ&list=PLmPit9IIdzwQzlveiZU1bhR6PaDGKwshI&index=1&t=3371s&pp=gAQBiAQB)  
  [View GitHub repo](https://github.com/CloudWithVarJosh/TLS-In-Kubernetes-Masterclass)  

#### Note for viewers: Data-plane & prerequisites

The **data plane** is where your workloads actually run (kubelet, runtime, CNI, kube-proxy, CoreDNS, CSI). Most of what you’ve studied so far **directly shapes its behavior**—**Pod Priority & Preemption**, **Admission controls**, **Pod Security (SecurityContext & Linux Capabilities)**, **Requests/Limits & Probes**, and **Storage (SC/PVC/PV)**.
**Advice:** focus on troubleshooting **only after** these basics are clear. When stuck, revisit the linked lectures, then debug in this order: **control plane OK → node/kubelet → CNI/IP → Services/endpoints (kube-proxy) → DNS → Storage → app config**.

---

### Quick refresher: the data-plane pieces

**kubelet (node agent)**

* Registers the node, starts/stops Pods via the CRI, runs probes, reports status.
* Reads `kubelet.conf`; talks to the API server and container runtime.
* If unhealthy → node **NotReady**, probes/exec/logs may stall.
* Think: “Is the node alive and manageable?”

**Container runtime (containerd/CRI-O)**

* Pulls images, creates containers/sandboxes, exposes the **CRI socket** to kubelet.
* If down/mispointed → kubelet can’t create/attach pods; pulls fail.
* Existing containers may keep running, but are unmanaged.
* Think: “CRI up and kubelet pointing to the right socket?”

**CNI (pod networking)**

* Assigns Pod IPs, wires veth/route rules, enforces network policy (via plugin).
* Config lives in **`/etc/cni/net.d`**; plugin runs as a DaemonSet.
* If missing/unhealthy → Pods stuck **ContainerCreating** (“CNI plugin not initialized”).
* Think: “Does a new Pod get an IP immediately?”

**kube-proxy (traffic director)**

* Programs iptables/IPVS to map **Service** VIPs/NodePorts → healthy Pod endpoints.
* Watches Endpoints/EndpointSlices; honors sessionAffinity, externalTrafficPolicy.
* If unhealthy → ClusterIP/NodePort traffic fails or is flaky.
* Think: “Do Services have endpoints, and is kube-proxy applying rules?”

**CoreDNS (cluster DNS)**

* Resolves service names for Pods via the `kube-dns` Service (ClusterIP).
* Forwards externals upstream; depends on kube-proxy and endpoints.
* If down/scaled to 0 → `nslookup kubernetes.default` fails in Pods.
* Think: “Can a Pod resolve service names right now?”

**CSI (storage interface)**

* Provisions and mounts volumes via controller/node plugins; uses **StorageClass/PVC/PV**.
* If controller/node driver is unhealthy → **PVC Pending** or mount errors.
* Defaults matter (a default SC avoids PVCs hanging).
* Think: “Do PVCs bind, and do Pods mount successfully?”

---

## Read This First: How to Use the Scenarios

These are **exam-style, high-probability** data-plane failures. Each scenario is written as **Problem → Likely causes → Checks → Fix**, with **minimal changes** and quick verification. Assume the lab stack: **kubeadm + containerd + Calico**, and stay inside the cluster (no cloud/firewall specifics).

**Approach (fast path)**

* Start at **node health**: kubelet/CRI, conditions, pressure.
* Then **CNI & Pod IPs**: `/etc/cni/net.d/`, CNI DaemonSet, pod gets an IP.
* Next **Services**: selectors → **Endpoints**; then **kube-proxy** health.
* Then **DNS**: CoreDNS pods + in-pod lookups.
* Storage next (**PVC/CSI**), then **app-level** issues (probes/limits).
* **Read Events first**; `describe` usually spells out the failure.
* Make the **smallest fix** that turns signals green; verify immediately.
* Use a throwaway **busybox** pod for `nslookup`/`wget` checks; clean up.

With that flow in mind, jump into the scenarios.

---

## 1) Node **NotReady** / kubelet not registered

**Problem**
`kubectl get nodes` shows **NotReady** (or the node never registers); new Pods won’t schedule there.

### Likely causes → How to fix

* **kubelet service stopped / stuck**
  *Fix:* `sudo systemctl start kubelet` → recheck `kubectl get nodes`.

* **Swap ON**
  *Fix:* `sudo swapoff -a` (and remove swap from `/etc/fstab`), then `sudo systemctl start kubelet`.

* **kubelet kubeconfig/certs bad** (wrong `server:` port, bad CA path, expired cert)
  *Fix:* Restore **/etc/kubernetes/kubelet.conf** (correct `server: https://<apiserver>:6443`, valid CA). `sudo systemctl restart kubelet`.

* **Container runtime down (containerd/cri-o)**
  *Fix:* `sudo systemctl start containerd` (or cri-o), then restart kubelet.

* **Clock skew / DNS to API**
  *Fix:* `sudo systemctl restart systemd-timesyncd` (or chrony); fix `/etc/hosts`/DNS; ensure `:6443` reachable.

* **Resource pressure (Disk/Memory/PIDs)**
  *Fix:* Free disk/inodes, lower load; kubelet comes back once thresholds clear.

* *(Optional)* **cgroup driver mismatch**
  *Fix:* Align kubelet & runtime drivers; restart both.

### Quick checks (with what they tell you)

```bash
# Cluster view
kubectl get nodes -o wide
kubectl describe node <node>         # Conditions: Ready, Memory/Disk/PID pressure

# On the node
sudo systemctl status kubelet        # service up?
sudo journalctl -u kubelet -b -o cat | tail -n 80
sudo systemctl status containerd     # runtime up?
crictl info | head -20               # CRI & cgroup driver

# Swap / resources
swapon --show
free -h
df -h

# API reachability & time
getent hosts <api-server-hostname>
nc -vz <api-server-hostname> 6443    # or: curl -vk https://<api>:6443
timedatectl
```

### Symptom clues in logs (what to look for)

* **Swap:** `running with swap on is not supported`
* **Certs/CA:** `x509: certificate signed by unknown authority`, `no such file or directory` (CA/key), `certificate has expired or is not yet valid`
* **API connect:** `failed to run Kubelet: cannot connect ... :6443`, `connection refused`, `i/o timeout`
* **Runtime/CRI:** `container runtime network not ready`, `failed to connect to CRI`
* **Pressure:** `eviction manager: ...` / `imagefs has insufficient space`

---


## 2) Pods stuck **ContainerCreating** / **CNI not initialized**

**Problem**
Pods never get IPs; Events show *CNI plugin not initialized* or *failed to set up sandbox*.

### Likely causes → How to fix

* **CNI DaemonSet not running / scaled to 0**
  *Fix:* `kubectl -n kube-system get ds` → scale DS back or re-apply CNI manifest.
* **`/etc/cni/net.d` missing/empty or wrong plugin conf**
  *Fix:* Restore valid CNI config (or re-install add-on), then `sudo systemctl restart kubelet`.
* **Wrong Pod CIDR / mismatched cluster networking config**
  *Fix:* Align CNI config with cluster Pod CIDR; re-apply and restart kubelet.

### Checks (quick and surgical)

```bash
kubectl -n kube-system get pods -o wide | egrep -i 'calico|cilium|flannel|weave'     # CNI pods healthy?
kubectl -n kube-system get ds -o wide | egrep -i 'calico|cilium|flannel|weave'       # DS desired/ready match?
kubectl describe pod <p> | sed -n '/Events:/,$p'                                      # exact CNI errors
ls -l /etc/cni/net.d/                                                                 # config present?
sudo journalctl -u kubelet -b -o cat | egrep -i 'cni|sandbox|plugin|network' | tail  # kubelet hints
```

### Fix (minimal moves, then verify)

```bash
# If DS was scaled down or crashed:
kubectl -n kube-system rollout restart ds/<cni-daemonset>   # or scale back to desired

# If config missing:
# (restore your backup or re-apply the CNI add-on manifest)
sudo systemctl restart kubelet

# Verify:
kubectl delete pod <p>; kubectl get pod <p> -o wide         # pod gets IP, goes Running
```

**Example**
`Events: CNI plugin not initialized` + empty `/etc/cni/net.d` → re-apply CNI add-on → restart kubelet → new Pod gets IP and **Running**.


---

## 3) Pod-to-Pod across nodes / **Service ClusterIP** not reachable

**Problem**
Pods on different nodes can’t talk; `curl http://<svc-name>:<port>` (or `curl <ClusterIP>:<port>`) fails.

### Likely causes → How to fix

* **Service has no endpoints** (selector mismatch / pods not Ready)
  *Fix:* Correct the Service **selector** to match pod labels; ensure backends are **Running & Ready**.
* **`targetPort` wrong / app not listening**
  *Fix:* Set Service **`targetPort`** to the container’s actual listening port (or update the container to listen on the declared port).
* **kube-proxy not programming rules** (DS unhealthy)
  *Fix:* Check kube-proxy DaemonSet; restart/rollout if needed.
* **NetworkPolicy blocking traffic**
  *Fix:* Inspect policies in the namespace; add an allow rule or remove the blocking policy.

### Checks (quick)

```bash
# Service & backends
kubectl get svc <svc> -o wide
kubectl describe svc <svc>                  # selector, port/targetPort
kubectl get endpoints,endpointslices <svc> -o wide
kubectl get pods -l <svc-selector> -o wide  # Ready=TRUE? correct labels?

# kube-proxy health
kubectl -n kube-system get ds kube-proxy -o wide
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

# NetworkPolicy (namespace-wide)
kubectl get netpol
```

### Fix (minimal moves, then verify)

```bash
# If endpoints are empty: fix selector or pod labels
kubectl patch svc <svc> -p '{"spec":{"selector":{"app":"<correct-label>"}}}'

# If targetPort is wrong: set it to the container’s port
kubectl patch svc <svc> -p '{"spec":{"ports":[{"port":80,"targetPort":8080}]}}'  # example

# If kube-proxy unhealthy:
kubectl -n kube-system rollout restart ds/kube-proxy

# If NetworkPolicy blocks traffic: adjust or remove the policy
# (e.g., temporarily allow all within the namespace for verification)

# Verify:
kubectl get endpoints <svc> -o wide
kubectl run curl --image=busybox:1.36 --restart=Never -- sh -c 'wget -qO- http://<svc>:<port> || echo FAIL'
```

**Example**
Endpoints show `<none>` → fix Service selector to match pod label (e.g., `app=web`) → endpoints populate → `wget http://<svc>:<port>` succeeds.

---

## 4) **DNS** failures (CoreDNS)

**Problem**
Pods say **no such host**; Service names don’t resolve.

**Likely causes**

* CoreDNS pods down/crashing
* Kubelet `--cluster-dns` wrong; pod `/etc/resolv.conf` points to bad IP
* NetworkPolicy blocks DNS; CoreDNS can’t reach upstream

**Checks**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide    # CoreDNS healthy?
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=80      # loop/forward errors?
# From a test pod:
nslookup kubernetes.default.svc.cluster.local 10.96.0.10       # replace with your cluster DNS IP
cat /etc/resolv.conf                                           # search/domain/options inside the pod
kubectl get svc -n kube-system kube-dns -o wide                # ClusterIP of DNS
```

**Fix**

* Restart/fix CoreDNS; correct the cluster DNS IP used by kubelet.
* Remove blocking NetworkPolicies; ensure upstream resolvers are reachable.

**Example**
Pod `resolv.conf` shows a wrong nameserver → set kubelet `--cluster-dns=10.96.0.10`, restart kubelet; name lookups work.

---

## 5) **ImagePullBackOff / ErrImagePull**

**Problem**
Pod won’t start because the image can’t be pulled.

### Likely causes → How to fix

* **Wrong image name/tag**
  *Fix:* Point `image:` to a valid repo\:tag; redeploy.
* **Private registry requires auth**
  *Fix:* Create an `imagePullSecret` and reference it in the Pod/ServiceAccount.
* **Stale local image / policy**
  *Fix:* Ensure correct tag; delete the Pod (or `rollout restart`) to force a fresh pull.

### Checks (quick)

```bash
kubectl describe pod <p> | sed -n '/Events:/,$p'   # shows the exact pull error
crictl pull <image:tag>                            # try pulling from the node
kubectl get sa <sa> -o yaml | grep -A2 imagePullSecrets  # is the secret attached?
```

### Fix (minimal, then verify)

```bash
# Example for private registry:
kubectl create secret docker-registry regcred \
  --docker-server=<REG> --docker-username=<U> --docker-password=<P> --docker-email=<E>
# Reference in Pod spec or attach to the ServiceAccount
kubectl delete pod <p>   # force re-pull
kubectl get pods -w
```

**Example**
Events show `manifest unknown` → fix **tag** → Pod pulls and becomes **Running**.

---

## 6) **CrashLoopBackOff / OOMKilled / probe failures**

**Problem**
Container starts then repeatedly exits; readiness never becomes **Ready**.

### Likely causes → How to fix

* **Bad command/args or missing config/volume**
  *Fix:* Correct `command/args`, env, mounts; reapply.
* **Probes too aggressive** (app needs warm-up)
  *Fix:* Relax `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`.
* **OOMKilled** (exit 137) / limits too low
  *Fix:* Raise memory **requests/limits** (or reduce app usage).

### Checks (quick)

```bash
kubectl logs <p> --previous
kubectl describe pod <p> | sed -n '/State:/,/Events:/p'   # OOMKilled? ExitCode?
kubectl get deploy <d> -o yaml | sed -n '/livenessProbe:/,/readinessProbe:/p'
```

### Fix (minimal, then verify)

```bash
# Example: bump memory & restart
kubectl set resources deploy/<d> --requests=memory=256Mi --limits=memory=512Mi
kubectl rollout restart deploy/<d>
kubectl get pods -w
```

**Example**
Describe shows **OOMKilled** → increase memory limits → Pod stabilizes.

---

## 7) **PVC Pending / volume mount errors (CSI)**

**Problem**
Pod stuck **Pending** (PVC not Bound) or **CreateContainerError** (mount/permission).

### Likely causes → How to fix

* **No default StorageClass / wrong accessModes/size**
  *Fix:* Mark an SC as **default**; align PVC size & access modes.
* **CSI driver unhealthy**
  *Fix:* Ensure CSI controller/node Pods are **Running**; then retry.
* **Bad mount spec (name/path/fsType/permissions)**
  *Fix:* Correct `volumeMounts`/`volumes` names & paths; set proper `fsType` or `fsGroup` if needed.

### Checks (quick)

```bash
kubectl get pvc,pv,sc
kubectl describe pvc <pvc>                    # why not Bound?
kubectl -n kube-system get pods | grep -i csi # driver health
kubectl describe pod <p> | sed -n '/Mounts:/,/Events:/p'
```

### Fix (minimal, then verify)

```bash
# Make an SC default (example)
kubectl annotate sc <sc-name> storageclass.kubernetes.io/is-default-class=true --overwrite
# Retry the workload
kubectl delete pod <p> && kubectl apply -f <pod>.yaml
kubectl get pvc && kubectl get pod <p> -o wide
```

**Example**
No default SC → PVC **Pending** → mark SC default → PVC **Bound** → Pod **Running**.

---

## 8) **NodePort / LoadBalancer** not reachable

**Problem**
`curl http://<nodeIP>:<nodePort>` (or via LB) fails.

### Likely causes → How to fix

* **Service selector/targetPort mismatch**
  *Fix:* Match Service **selector** to pod labels; set correct **targetPort**.
* **`externalTrafficPolicy: Local` with no local endpoints**
  *Fix:* Schedule a backend on that node or set policy to **Cluster**.
* **kube-proxy rules not present** (DS unhealthy)
  *Fix:* Check kube-proxy DaemonSet; restart/rollout.

### Checks (quick)

```bash
kubectl describe svc <svc>                   # type, nodePort, targetPort, policy
kubectl get endpoints <svc> -o wide          # endpoints exist? on that node for Local?
kubectl -n kube-system get ds kube-proxy -o wide
# On the node, is kube-proxy listening on the nodePort?
sudo ss -lnt 'sport = :<nodePort>'
```

### Fix (minimal, then verify)

```bash
# Fix selector or targetPort as needed (patch or reapply manifest)
# If policy=Local, ensure a backend Pod on that node
kubectl -n kube-system rollout restart ds/kube-proxy   # if kube-proxy unhealthy

kubectl get endpoints <svc> -o wide
```

**Example**
Service set to **Local** but node has **no endpoint** → place a Pod on that node (or switch to **Cluster**) → NodePort works.

---

## Pod Deletion and Termination Signals

![Alt text](/images/22a.png)

### **What Happens When You Run `kubectl delete pod`?**

- **Command Execution:**  
  When you run:
  ```bash
  kubectl delete pod <pod-name>
  ```
  Kubernetes initiates a graceful termination sequence rather than an abrupt shutdown.

- **Grace Period & Signals:**  
  - **SIGTERM:** As soon as the delete command is issued, Kubernetes sends the SIGTERM signal to the main process of each container inside the pod. This signal tells the application: “It’s time to shut down gracefully. Please wrap up any ongoing tasks.”
  - **Termination Grace Period:** By default, Kubernetes waits for 30 seconds (configurable via `terminationGracePeriodSeconds` in the pod spec) for the container to exit gracefully.
  - **SIGKILL:** If the container does not exit within this grace period, Kubernetes sends a SIGKILL signal to force an immediate shutdown.

Containers are typically provided with a graceful shutdown period during termination to allow the application to handle the termination signal **(e.g., SIGTERM)** appropriately. This mechanism ensures that important operations, such as closing active connections, flushing cached data, or completing ongoing tasks, are carried out before the container stops.

### **Example: Graceful Shutdown of a Web Server**

Imagine you have a Pod running an NGINX server. With a graceful shutdown enabled in your container code, it might finish serving current requests and close connections cleanly when receiving a SIGTERM. If it takes too long—over the grace period—SIGKILL will terminate it immediately.

**YAML Snippet:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx- graceful
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: nginx
    image: nginx:latest
```
---

### Force Delete a Pod

To force delete a pod:
```
kubectl delete pod mypod --force=true --grace-period=0
```

This **forces the pod to terminate immediately**, and under the hood, it behaves similarly to sending a `SIGKILL` signal to the container's process.

**Explanation:**

- `--force=true` bypasses the normal graceful deletion process.
- `--grace-period=0` tells Kubernetes **not to wait** and to immediately kill the pod.
- Kubernetes doesn't directly send UNIX signals itself but **delegates** this to the container runtime (like containerd or Docker).
  
When Kubernetes receives this forceful deletion request:
1. It immediately removes the pod from the API server (even if it's still running on the node).
2. It then asks the container runtime to kill the container **without giving the app time to shut down gracefully**, similar to a `SIGKILL`.

---

### **Important:**
If you run:

```
kubectl delete pod mypod --force=true
```
**without specifying `--grace-period=0`**, it still tries to respect the pod's default termination grace period (which is usually 30 seconds) unless overridden by the pod's spec.

To **force immediate termination** (equivalent to `SIGKILL`):
```
kubectl delete pod mypod --force=true --grace-period=0
```
| **Command** | **API Server (Pod object)** | **Kubelet (Container termination)** | **Grace Period Behavior** | **Explanation** |
|------------|----------------------------|-------------------------------------|---------------------------|-----------------|
| `kubectl delete pod mypod` | Deletes Pod gracefully | Kubelet follows normal process | Graceful shutdown | Pod object removed, Kubelet gracefully stops container (SIGTERM, waits for grace period, then SIGKILL). |
| `kubectl delete pod mypod --force=true` | Deletes Pod immediately | Kubelet still respects terminationGracePeriodSeconds | Graceful unless explicitly overridden | **Pod object deleted immediately, but Kubelet still monitors the running container and honors the grace period.** |
| `kubectl delete pod mypod --force=true --grace-period=0` | Deletes Pod immediately | Kubelet kills container immediately | No grace period, immediate SIGKILL | **Pod deleted from API server, Kubelet stops monitoring and forcefully kills the container instantly.** |

---

## Restart Policies

![Alt text](/images/22b.png)

Restart policies define how Kubernetes responds when containers within a pod terminate. These policies are defined at the pod level and apply to all containers within the pod. There are three types of restart policies:


### **1. Always**
- **Use Case:**  
  Commonly used in Deployments, ReplicaSets, and StatefulSets, where continuous availability and scalability are key requirements.

- **Behavior:**  
  The container is always restarted regardless of its exit code (whether it succeeds with exit code `0` or fails with a non-zero exit code).

- **Example:**  
  In a typical web application or API server where uptime is critical, using `Always` ensures the pod remains perpetually running. Kubernetes automatically replaces failed containers to minimize downtime.

- **Kubernetes Objects That Use `Always`:**  
  - **Deployments**: Ensures pods are always running as per the desired replicas.  
  - **ReplicaSets**: Provides fault tolerance and ensures the specified number of replicas.  
  - **StatefulSets**: Manages stateful applications with persistence and ordered updates.  
  - **DaemonSets**: Ensures one pod per node is always running for background tasks.

**Example**
For objects like **Deployments**, which rely on the `Always` restart policy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      restartPolicy: Always
      containers:
      - name: nginx
        image: nginx:latest
```

In this example:
- The **kind** is `Deployment`.
- The `restartPolicy` is `Always` (the default for `Deployments`).

---

### **2. OnFailure**
- **Use Case:**  
  Often used in **Jobs** or **CronJobs**, particularly for batch tasks or one-time processes that may fail and require retries to complete successfully.

- **Behavior:**  
  The container is restarted **only** if it exits with a failure (non-zero exit code). If the container exits successfully (exit code `0`), no restart occurs.

- **Example:**  
  A data processing job that processes input files might fail if a file is missing or corrupted. With `OnFailure`, the job will be retried to handle transient errors.

- **Kubernetes Objects That Use `OnFailure`:**  
  - **Jobs**: Ensures task completion by retrying failed containers.  
  - **CronJobs**: Executes periodic tasks and retries in case of failures.  
  - **Standalone Pods**: Sometimes used for debugging or simple one-off tasks.

**Example**
For objects like **Jobs**, which commonly use the `OnFailure` restart policy:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-example
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: my-batch-job:latest
```

In this example:
- The **kind** is `Job`.
- The `restartPolicy` is `OnFailure` (appropriate for retrying failed batch tasks).

---

### **3. Never**
- **Use Case:**  
  Used when restarting is undesirable, such as for one-time diagnostics, debugging, or exploratory pods where the goal is to observe the pod's behavior or logs post-exit.

- **Behavior:**  
  The container is **not restarted**, regardless of the exit code (whether it succeeds or fails).

- **Example:**  
  A diagnostic pod that runs a specific command to analyze the environment (e.g., checking cluster DNS resolution) and then exits with results.

- **Kubernetes Objects That Use `Never`:**  
  - **Standalone Pods**: Diagnostic or ephemeral workloads that don't require restarts.  
  - **Jobs**: Rarely, but sometimes a one-time Job may use it for immediate analysis without retries.

**Example**
For standalone diagnostic pods or Ephemeral Jobs where restarting is not desirable:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: diagnostic-pod
spec:
  restartPolicy: Never
  containers:
  - name: debug-container
    image: busybox
    command: ["nslookup", "kubernetes.default.svc.cluster.local"]
```

In this example:
- The **kind** remains `Pod`, as standalone pods typically use the `Never` restart policy for one-off tasks.

---

### **Summary Table:**

| **Restart Policy** | **Use Case**                                     | **Default Kubernetes Objects**                        | **Details**                                                                                  |
|--------------------|-------------------------------------------------|-----------------------------------------------------|----------------------------------------------------------------------------------------------|
| **Always**         | **Continuous availability and scalability**          | **Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Standalone Pods (Manual)** | - Ensures Pods are **restarted indefinitely**, regardless of exit codes.<br>- Suitable for **long-running workloads** such as web servers or background tasks.<br>- **Default** for objects designed to maintain **high availability and fault tolerance**.<br>- **Example**: Deployments use ReplicaSets, which always manage Pods with `restartPolicy: Always`. |
| **OnFailure**      | **Batch jobs or retrying failed tasks**              | **Jobs, CronJobs, Standalone Pods**                     | - Restarts containers **only if they fail** (non-zero exit code).<br>- Stops if the container exits successfully.<br>- **Default** for finite workloads such as batch processing and scheduled jobs.<br>- **Example**: Jobs are designed to retry failed tasks but exit cleanly on success. |
| **Never**          | **One-off diagnostics or exploratory tasks**         | **Not default for any object**                         | - Containers are **not restarted**, regardless of the exit code.<br>- Must be **explicitly set**.<br>- Suitable for **one-off tasks** or debugging scenarios where restarts are undesirable.<br>- **Example**: Running diagnostic commands in standalone Pods without restarts. |

---

## Image Pull Policies

![Alt text](/images/22c.png)

The `imagePullPolicy` in Kubernetes specifies how the container runtime pulls container images for pods. It governs whether Kubernetes should pull the image from a container registry or use a cached version already present on the node. This setting is vital for controlling how your deployments behave in development, testing, and production environments. Kubernetes supports three types of `imagePullPolicy`:

---

### **1. Always**
- **Description:**  
  The `Always` policy instructs Kubernetes to always pull the container image from the registry, even if a matching image is already present on the node.

- **Behavior:**  
  - Kubernetes pulls the specified image every time the pod starts.
  - This ensures that the most recent version of the image is used.

- **When to Use:**  
  - Use `Always` in **development environments** to ensure you're always working with the latest image changes.
  - Recommended when you rely on the `latest` image tag (not ideal in production, though).

- **Example:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: always-example
  spec:
    containers:
    - name: my-app
      image: myrepo/my-app:latest
      imagePullPolicy: Always
  ```
  In this example, Kubernetes will pull the `my-app:latest` image from the registry every time the pod is created or restarted, ensuring that updates to the `latest` tag are reflected.

---

### **2. IfNotPresent**
- **Description:**  
  The `IfNotPresent` policy directs Kubernetes to pull the container image from the registry **only if the image is not already available on the node**.

- **Behavior:**  
  - If the image is present on the node's local cache, Kubernetes uses it without pulling a new version.
  - If the image is missing, Kubernetes fetches it from the container registry.

- **When to Use:**  
  - Use `IfNotPresent` in **production environments** to avoid unnecessary image pulls, which saves bandwidth and improves pod startup times.
  - Appropriate for images with specific tags, such as `my-app:v1.2.3`, because tagged versions rarely change and you want predictable behavior.

- **Example:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: ifnotpresent-example
  spec:
    containers:
    - name: my-app
      image: myrepo/my-app:v1.2.3
      imagePullPolicy: IfNotPresent
  ```
  Here, Kubernetes will pull the `my-app:v1.2.3` image only if it’s not already on the node. If the image is present locally, it will use the cached version.

---

### **3. Never**
- **Description:**  
  The `Never` policy tells Kubernetes to **never pull the image** from the registry, assuming the image is already present on the node.

- **Behavior:**  
  - If the image is not present locally on the node, the pod will fail to start.
  - Kubernetes does not attempt to contact the registry at all.

- **When to Use:**  
  - Use `Never` in **air-gapped environments** or scenarios where nodes are preloaded with container images and external image pulls are not allowed.
  - Suitable for testing environments where you explicitly preload images manually.

- **Example:**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: never-example
  spec:
    containers:
    - name: my-app
      image: myrepo/my-app:v1.2.3
      imagePullPolicy: Never
  ```
  In this case, Kubernetes will attempt to use the `my-app:v1.2.3` image from the node’s local cache. If the image is unavailable, the pod fails with an error.

---

### **Default Behavior**


The default value of `imagePullPolicy` depends on how you define the image in your pod specification:

1. **If the image tag is `latest`** (e.g., `myrepo/my-app:latest`) **or no tag is specified at all** (e.g., `myrepo/my-app`, which defaults to `:latest`):
   - The default `imagePullPolicy` is **`Always`**.
  
2. **If the image has a specific, non-latest tag** (e.g., `myrepo/my-app:v1.2.3`):
   - The default `imagePullPolicy` is **`IfNotPresent`**.

**Best Practice:**  
To avoid unexpected behavior (especially when tags like `latest` are reused or image changes aren't reflected), it is recommended to **explicitly set the `imagePullPolicy`** in your pod specification.

---

### **Which `imagePullPolicy` Should You Use?**
| **ImagePullPolicy** | **When to Use**                                                                                                                                                         |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Always**         | - Useful in **development environments** to ensure you always pull the latest image version.<br>- Recommended when using the `latest` tag or frequently updated images. |
| **IfNotPresent**   | - Ideal for **production environments** to improve startup time and avoid unnecessary pulls.<br>- Best used with specific, immutable image tags like `v1.2.3`.           |
| **Never**          | - Suitable for **air-gapped environments** or systems where images are **preloaded manually**.<br>- Helpful in controlled testing scenarios to avoid any registry pulls. |

---

### **Key Considerations**
1. **Avoid Using `latest` Tag in Production:**  
   Using `Always` with `latest` in production can lead to unpredictable behavior if the image changes without notice. Instead, use specific version tags (`v1.2.3`) for stability.

2. **Preload Images for Faster Startup:**  
   In air-gapped environments or production, preload images on nodes to reduce reliance on the registry. Combine `imagePullPolicy: Never` with image preloading for tight control.

3. **Monitor Bandwidth Usage:**  
   Frequent image pulls with `Always` can increase bandwidth consumption and impact cluster performance. Use `IfNotPresent` or `Never` where appropriate to optimize.

---

## Pod Lifecycle Phases

![Alt text](/images/22d.png)

In Kubernetes, the lifecycle of a Pod is divided into distinct **phases** that represent its current state at a given point in time. These phases help administrators understand what the Pod is doing and whether it is functioning as expected.
Here are the phases in detail:

#### **1. Pending**
- **What it means:**  
The Pod has been accepted by the Kubernetes system, but one or more of its containers have not yet been created or started running. This phase encompasses the initial preparation of the Pod and its containers.

**This phase includes activities like:**
- **Scheduling:** Assigning the Pod to a node based on available resources, constraints, and scheduling policies.
- **Image Pulling:** Fetching container images from the registry if they are not already present on the node.
- **Container Creation (Subphase `ContainerCreating`):**  
  Once the Pod is scheduled, Kubernetes enters the `ContainerCreating` subphase where:
  - The container runtime (e.g., Docker, containerd) sets up the container environment.
  - Storage volumes, networking, and environment variables are initialized for the container.

- **Common Causes:**  
  - Insufficient cluster resources (e.g., CPU, memory, storage).
  - Unsatisfied scheduling constraints (like node affinity, taints, etc.).
  - Image download delays or errors, such as `ErrImagePull` or `ImagePullBackOff`.

- **Key Debugging Tips:**  
  Use `kubectl describe pod <pod-name>` to check for errors or delays in the `Events` section, such as:
  - "FailedScheduling" due to resource constraints.
  - "Failed to pull image" due to incorrect image names or registry authentication issues.

### **How `ContainerCreating` Fits In:**
The `ContainerCreating` subphase of `Pending` represents the final preparatory step before containers move to the `Running` phase. During this time, Kubernetes ensures that the container environment is set up properly, including:
- Pulling the container image (if not cached).
- Setting up storage volumes and network configurations.
- Initializing any environment variables or secrets.

---

#### **2. Running**
- **What it means:**  
  The Pod has been successfully scheduled to a node, and at least one container is in the `Running` state.
  - However, the overall Pod status in `kubectl get pods` reflects the **worst container state**. For example:
    - If one container is `Running` but another is in `ErrImagePull`, the Pod status will show `ErrImagePull`.
  


---

#### **3. Succeeded**
- **What it means:**  
  The Pod has completed its lifecycle, and all containers within the Pod have terminated successfully (exit code `0`).
  - No restarts will occur, as this state is terminal.
- **Typical Use Case**
  Batch jobs, short-lived Pods, cronjobs where containers run once and exit cleanly.

---

#### **4. Failed**
- **What it means:**  
  The Pod lifecycle has ended, but one or more containers have failed (exited with **non-zero** exit codes).
  - This phase is also terminal, meaning the Pod won't restart (restartPolicy=Never) unless recreated by a controller.

---

#### **5. Unknown**
- **What it means:**  
  The Pod's state cannot be determined, often because the node hosting the Pod is unreachable or down.
  - This phase is rare and typically indicates infrastructure issues.

---

### **Pod Status in Multi-Container Pods**
When you run `kubectl get pods`, the Pod status shown represents the **worst container's state**. Kubernetes aggregates container statuses within the Pod and prioritizes showing errors over successes. 

- **Example:**  
  If a Pod has 4 containers and:
  - 3 containers are `Running`
  - 1 container is in `ErrImagePull`
  - The overall Pod status will be `ErrImagePull`.

This behavior ensures that issues within a Pod are highlighted immediately.

---

### **Differentiation Between `Succeeded` and `Completed`**

- **`Succeeded`:**  
  - This is the **official lifecycle phase** of the Pod in Kubernetes, as described internally in the Pod's status object.
  - You’ll see `Succeeded` when running `kubectl describe pod <pod-name>`. Example output:
    ```
    Status: Succeeded
    ```

- **`Completed`:**  
  - This is the **human-readable status** displayed in the `STATUS` column when you run `kubectl get pods`.
  - Kubernetes uses "Completed" to summarize the `Succeeded` lifecycle phase for easier interpretation.

**Key Difference:**  
- `"Succeeded"` is the lifecycle phase tracked internally by Kubernetes.
- `"Completed"` is the user-friendly representation shown in the `kubectl get pods` output.

---


### **Summary Table**

| **Phase**    | **Description**                                                                                                         |
|-------------|-------------------------------------------------------------------------------------------------------------------------|
| **Pending**  | Pod has been accepted by the Kubernetes cluster, but containers are not yet running. Reasons include image pull delays, unscheduled pods, etc. |
| **Running**  | Pod has been scheduled to a node, and all containers are either running or starting.                                    |
| **Succeeded**| All containers in the Pod have terminated **successfully** (`exit code 0`), and the Pod won’t restart.                  |
| **Failed**   | All containers have terminated, **but at least one exited with failure** (`non-zero exit code`), and is not set for automatic restart.      |
| **Unknown**  | Kubernetes cannot determine the Pod state, often due to communication issues with the node where the Pod is running. Often because of Infrastructure failures.  |

---

## **Demo 1: Observing Short-Lived Pod Lifecycle**
In this example, you'll create a Pod that performs a quick task (`ls` command) and exits. You’ll observe the lifecycle phases using real-time monitoring.

### **Steps**
1. Run the following command to create the Pod:
   ```bash
   kubectl run ubuntu-2 --image=ubuntu --restart=Never -- ls
   ```
   This will create a Pod to execute the `ls` command and terminate immediately.

2. In another terminal, monitor the Pod's status transitions in real time:
   ```bash
   kubectl get pods -w
   ```
   You will observe the following lifecycle phases:
   - **`Pending`**: The Pod is being scheduled and the container image is being pulled.
   - **`ContainerCreating`**: Kubernetes sets up the container runtime environment.
   - **`Completed`**: The task finishes successfully and the Pod exits.

### **Key Takeaway**
The Pod did not enter the visible `Running` phase because the task (`ls` command) completed so quickly that Kubernetes transitioned the Pod directly to `Completed`.

---

## **Demo 2: Observing Longer Pod Lifecycle**
In this example, you’ll create a Pod that runs a longer task (`sleep 5`) to observe all lifecycle phases, including `Running`.

### **Steps**
1. Run the following command to create the Pod:
   ```bash
   kubectl run ubuntu-2 --image=ubuntu --restart=Never -- sleep 5
   ```
   This will create a Pod to execute the `sleep` command, pausing for 5 seconds before terminating.

2. In another terminal, monitor the Pod's status transitions in real time:
   ```bash
   kubectl get pods -w
   ```
   You will observe the following lifecycle phases:
   - **`Pending`**: The Pod is being scheduled and the container image is being pulled.
   - **`ContainerCreating`**: Kubernetes sets up the container runtime environment.
   - **`Running`**: The container actively executes the `sleep 5` command.
   - **`Completed`**: The task finishes successfully and the Pod exits.

### **Key Takeaway**
Here, the container task (`sleep 5`) lasts long enough for the Pod to enter the `Running` phase before transitioning to `Completed`.

---

### **Quick Debugging Workflow**
- **Check Pod Status:**
  ```bash
  kubectl get pods
  ```
  Look for statuses like `Pending`, `Running`, `Completed`, or error states like `ErrImagePull` or `CrashLoopBackOff`.

- **Describe the Pod:**
  ```bash
  kubectl describe pod <pod-name>
  ```
  Inspect `Events`, `Conditions`, and individual container statuses to diagnose issues.

- **Check Container Logs:**
  ```bash
  kubectl logs <pod-name> -c <container-name>
  ```

---

## Pod Status Deep Dive
Below is a table summarizing the common Pod statuses, their descriptions, causes, and debugging steps:

| **Pod Status/Error**       | **Description**                                                                                          | **Common Causes**                                                                                                   | **How to Debug**                                                                                                                             |
|----------------------------|----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| **CrashLoopBackOff**        | Container repeatedly crashes and is restarted with exponential backoff.                                  | - Application errors or bugs<br>- Missing dependencies or misconfigurations<br>- Resource exhaustion (e.g., OOMKilled)<br>- Liveness probe failures | - Inspect logs: `kubectl logs <pod-name>` or `kubectl logs --previous`<br>- Check environment variables, ConfigMaps, and Secrets.<br>- Verify liveness probe setup. |
| **ImagePullBackOff**        | Kubernetes is retrying to pull the container image, but previous attempts failed.                        | - Invalid image name or tag<br>- Authentication issues with private registries<br>- Network connectivity problems | - Describe Pod: `kubectl describe pod <pod-name>`<br>- Verify image name and tag.<br>- Test pulling the image manually (`docker pull`).       |
| **ErrImagePull**            | Immediate failure to pull the container image.                                                          | - Invalid image or tag<br>- Registry authentication issues<br>- Registry network connectivity issues              | - Same as `ImagePullBackOff`.<br>- Check the `Events` section with `kubectl describe pod`.                                                   |
| **OOMKilled**               | Container terminated because it exceeded its memory limit.                                              | - Application memory leaks or high memory usage<br>- Insufficient memory limits in the Pod spec                   | - Check events: `kubectl describe pod <pod-name>`<br>- Monitor memory usage with `kubectl top pod`.<br>- Adjust memory requests and limits.   |
| **Unknown**                 | Pod state cannot be determined due to node communication failure.                                       | - Node failure or network issues                                                                                   | - Check node status: `kubectl get nodes`.<br>- Investigate node logs for issues.                                                             |
                               
---

## Debugging Common Pod Errors

### **Step-by-Step Debugging Workflow:**

1. **Inspect the Pod Description:**
   - Run:
     ```bash
     kubectl describe pod <pod-name>
     ```
   - **What to Look For:**  
     - Check the events section at the bottom.
     - Look for clues like “Failed to pull image” or “Back-off restarting failed container.”

2. **Check Container Logs:**
   - Run:
     ```bash
     kubectl logs <pod-name>
     ```
   - If the container is crashing repeatedly (CrashLoopBackOff), get logs from the previous instance:
     ```bash
     kubectl logs <pod-name> --previous
     ```
   - **What to Look For:**  
     - Application error messages
     - Exception stack traces or resource limitations (OOM errors)

3. **Validate Configuration:**
   - Inspect your pod YAML for correct image names, resource limits, and environment variables.
   - Confirm the container’s command/entrypoint settings (CMD vs. ENTRYPOINT in Docker).

4. **Recreate Scenarios in a Test Pod:**
   - Sometimes isolating a problematic configuration in a minimal pod helps diagnose the issue. Use a simple test container to simulate environment variables or image pull policies.

---

## Conclusion

You have now walked the **control plane**, the **data plane**, and **pod behavior** with a **repeatable approach**. You diagnosed **API server reachability**, **etcd health**, **scheduler and controller manager availability**, verified **node and kubelet health**, **CNI and DNS**, and practiced reading **pod states**, **restart policies**, **image pull policies**, and the **errors** that appear most often.

To make this stick, **practice each scenario in your own lab**, then repeat on a cloud cluster such as **EKS**, **GKE**, or **AKS**. Keep a **personal checklist** of the commands you used most, note the fixes that worked, and maintain a **known good baseline** for quick comparison. If you have suggestions or notice gaps, I would be grateful to hear them in the comments.

If you would like to keep learning with me, you are welcome to **subscribe**. I will be covering **Jenkins**, **Argo CD**, **Observability**, and more practical sessions next.

---

## References

**Control-Plane**
* [Create static Pods (mirror pod behavior)](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
* [Certificate management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
* [kubeadm certs (renew subcommands)](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
* [kubeadm certs renew all](https://kubernetes.io/docs/reference/setup-tools/kubeadm/generated/kubeadm_certs/kubeadm_certs_renew_all/)
* [kubeadm certs renew apiserver](https://kubernetes.io/docs/reference/setup-tools/kubeadm/generated/kubeadm_certs/kubeadm_certs_renew_apiserver/)
* [kube-scheduler reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
* [kube-controller-manager reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
* [kube-apiserver reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
* [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
* [etcdctl: check cluster status & health](https://etcd.io/docs/v3.5/tutorials/how-to-check-cluster-status/)

**Data-Plane**
* Kubernetes Docs — **Troubleshooting Clusters & Nodes**
  [https://kubernetes.io/docs/tasks/debug/](https://kubernetes.io/docs/tasks/debug/)
* Kubernetes Docs — **Nodes & Kubelet** (config, certificates, logs)
  [https://kubernetes.io/docs/concepts/architecture/nodes/](https://kubernetes.io/docs/concepts/architecture/nodes/)
  [https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
* Kubernetes Docs — **Container Runtime Interface (CRI)**
  [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
* Kubernetes Docs — **Cluster Networking & Services**
  [https://kubernetes.io/docs/concepts/cluster-administration/networking/](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
  [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
* Kubernetes Docs — **kube-proxy**
  [https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
* Kubernetes Docs — **DNS for Services and Pods** (CoreDNS)
  [https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
* CoreDNS Docs — **Kubernetes plugin** & ops notes
  [https://coredns.io/plugins/kubernetes/](https://coredns.io/plugins/kubernetes/)
* Kubernetes Docs — **Storage: PV/PVC/StorageClass** & **CSI**
  [https://kubernetes.io/docs/concepts/storage/](https://kubernetes.io/docs/concepts/storage/)
  [https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/)
  [https://kubernetes.io/docs/concepts/storage/storage-classes/](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* containerd — **Getting Started & Troubleshooting**
  [https://github.com/containerd/containerd/blob/main/docs/](https://github.com/containerd/containerd/blob/main/docs/)
* Calico (if you’re using it) — **Install & Troubleshoot**
  [https://docs.tigera.io/calico/latest/](https://docs.tigera.io/calico/latest/)

**Pods**
- [Kubernetes Official Documentation: Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Official Documentation: Container Images](https://kubernetes.io/docs/concepts/containers/images/)

---

