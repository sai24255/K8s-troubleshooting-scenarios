Understood.
Below is the **FINAL, COMPLETE, SINGLE SOP** that **does NOT skip anything** from our earlier discussion.

This SOP:

* Works for **everyone** (Junior → SRE → On-call)
* Starts from **basic cluster connection**
* Includes **real pod-level checks** (like `grep -i node`)
* Explains **what to do, why to do it, and what you learn**
* Is **not textbook** — it is **how production is actually handled**
* Can be used as a **learning document + operational runbook**

This is the **one document** you keep.
No back-chat dependency.

---

# 📘 FINAL SOP – SINGLE SOURCE OF TRUTH

## Datadog Alert: Kubernetes **Failed Pods**

**(Juniors + SREs + On-Call | Operational + Learning SOP)**

---

## 1. PURPOSE

This SOP defines **how to connect to the cluster, validate access, investigate Failed Pods, perform safe remediation, escalate correctly, and learn from the incident**.

It is written so that:

* A **junior** can safely handle alerts
* An **SRE** can reason deeply and fix permanently
* Everyone follows the **same thinking model**

---

## 2. ALERT CONTEXT (WHY THIS ALERT EXISTS)

**Alert:** Datadog – Failed Pods
**Monitor ID:** 537592
**Cluster:** caas
**Environments:** prod, stg
**Trigger:** ≥ 1 pod in `Failed` phase in last 15 minutes

### What “Failed” Means

* A container exited
* Kubernetes **will NOT restart** the pod
* Human decision is required

> Failed ≠ CrashLoopBackOff
> Failed means **Kubernetes gave up**

---

## 3. CORE MENTAL MODEL (FOR EVERYONE)

Every Failed Pod investigation answers **3 questions**:

1. **Why did the container exit?**
2. **Is it safe to restart once?**
3. **Who owns the permanent fix?**

All commands below exist only to answer these.

---

# 🔌 PART 0 – BASIC CLUSTER CONNECTION & SANITY CHECKS

> If these fail, **STOP**. Do not continue.

---

### 0.1 Check kubectl

```bash
kubectl version --client
```

---

### 0.2 Check kubeconfig & context

```bash
kubectl config current-context
```

Confirm you are on the **correct cluster**.

---

### 0.3 Verify cluster access

```bash
kubectl get nodes
```

| Result                   | Meaning           |
| ------------------------ | ----------------- |
| Nodes listed             | Access OK         |
| Forbidden / Unauthorized | Permission issue  |
| Timeout                  | Network/VPN issue |

---

### 0.4 Verify namespace exists

```bash
kubectl get ns
```

---

# 🔍 PART A – FAILED POD BASIC CHECKS (REAL OPERATIONS)

This is the **minimum required investigation**.

---

## Step 1: Confirm Failed Pod Exists NOW

```bash
kubectl get pods -n <NAMESPACE> --field-selector=status.phase=Failed
```

* If **no pod found** → alert is historical → document & close
* If found → note pod name & namespace

---

## Step 2: Describe the Pod (PRIMARY TRUTH)

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

From this output, extract:

* Exit code
* Reason
* Events (bottom)
* Node name

> **Events are Kubernetes explaining its decision**

---

## Step 3: Identify Node Involved (CRITICAL)

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -i node
```

Why this matters:

* Pods fail **on nodes**
* Node correlation reveals infra issues early

---

## Step 4: Pod Wide View

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o wide
```

Check:

* Node column
* Restart count
* Status

---

## Step 5: Get Exit Code (FACT)

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
-o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'
```

| Exit Code | Meaning      | Failure Layer    |
| --------- | ------------ | ---------------- |
| 1         | App crash    | Application      |
| 137       | OOMKilled    | Node / Resources |
| 0         | Job finished | Expected         |

---

## Step 6: Get Termination Reason & Message

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
-o jsonpath='{.status.containerStatuses[0].state.terminated.reason}{"\n"}{.status.containerStatuses[0].state.terminated.message}'
```

---

## Step 7: Review Pod Events (WHY)

```bash
kubectl get events -n <NAMESPACE> \
--field-selector involvedObject.name=<POD_NAME> \
--sort-by=.lastTimestamp
```

Look for:

* OOMKilled
* FailedScheduling
* NodeLost
* ImagePullBackOff
* Error

---

# 🧑‍💻 PART B – JUNIOR ENGINEER HANDLING (SAFE ZONE)

> Juniors **only operate here**.

---

## Scenario 1: Application Crash

**How you know**

* Exit code = 1
* Error in events

**Action (ONCE ONLY)**

```bash
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

**Why**

* Tests if failure is transient

**Tell On-Call**

> Application crash (exit 1). Restarted once. Monitoring.

---

## Scenario 2: OOMKilled (Memory)

**How you know**

* Exit code = 137
* Event = OOMKilled

**Temporary recovery**

```bash
kubectl delete pod <POD_NAME> -n <NAMESPACE>
```

**Why**

* Pod is already dead
* Delete only triggers recreation

**Tell On-Call**

> Pod OOMKilled. Restarted. Resource tuning required.

❌ Do NOT change resources

---

## Scenario 3: FailedScheduling

**How you know**

* Event = FailedScheduling

**Action**

* ❌ No changes

**Tell On-Call**

> Scheduler rejected pod (resource / affinity issue).

---

## Scenario 4: Node-Related Failure

**How you know**

* NodeLost
* Node NotReady

```bash
kubectl describe node <NODE_NAME>
```

**Tell On-Call immediately**

> Pod failed on node `<node>`. Node unhealthy. No disruptive action taken.

❌ Junior must NEVER cordon/drain

---

## Scenario 5: One-Time Transient Failure

**How you know**

* Restart successful
* No repeat failures

**Action**

* Monitor 15 minutes
* Close alert

---

## 🚫 Junior Hard Stops

* No YAML edits
* No resource changes
* No node drain/cordon
* No repeated restarts

---

# 🛠️ PART C – SRE / ON-CALL REMEDIATION (PERMANENT FIXES)

---

## Resource Tuning (Design Fix)

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Why:

* Requests protect scheduling
* Limits protect nodes

Apply via GitOps / CI-CD only.

---

## Node Remediation (Infra Surgery)

```bash
kubectl cordon <NODE_NAME>

kubectl drain <NODE_NAME> \
--ignore-daemonsets \
--delete-emptydir-data
```

Only after:

* Approval
* Replica validation
* Stakeholder communication

---

## Config Fixes

```bash
kubectl edit configmap <CONFIGMAP_NAME> -n <NAMESPACE>
kubectl edit secret <SECRET_NAME> -n <NAMESPACE>
```

---

# ✅ PART D – VERIFICATION (MANDATORY)

```bash
kubectl get pods -n <NAMESPACE>
```

Confirm:

* Pod Running
* No Failed pods
* Datadog alert cleared
* Stable for 15 minutes

---

# 📣 PART E – ESCALATION FORMAT (USE EXACTLY)

> Pod: `<name>`
> Namespace: `<ns>`
> Node: `<node>`
> Exit Code: `<code>`
> Reason/Event: `<message>`
> Action Taken: `<what you did>`
> Current Status: `<running/failing>`

---

# 🧠 PART F – LEARNING (HOW SREs GROW)

After closing the alert, answer:

1. Which layer failed? (App / Scheduler / Node)
2. What signal detected it?
3. What change prevents repeat?

If you cannot answer these → the incident is **not fully resolved**.

---

## ⭐ FINAL GOLDEN RULE

> **Connect → Observe → Classify → Act once → Escalate → Learn**

---

This SOP now includes:

* All basics
* All pod-level checks
* Junior + SRE handling
* Learning path
* Real production thinking

This is **complete**.
