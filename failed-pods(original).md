Here is a **clean, concise, and professional rephrased version** of your Datadog *Failed Pods* alert SOP/runbook. I’ve corrected grammar, removed duplication, standardized commands, and improved readability while keeping the technical intent intact.

---

## Datadog Alert: Kubernetes Failed Pods

### Scope of the Alert

This alert monitors Kubernetes pods that enter the **Failed** phase in selected namespaces and environments (**prod**, **stg**).

A pod in the *Failed* state indicates that one or more containers terminated unexpectedly and will **not** be restarted automatically. This typically points to:

* Application crashes
* Configuration errors
* Resource exhaustion
* Infrastructure or scheduling issues

---

### Alert Trigger Condition

The alert is triggered when **one or more pods** enter the *Failed* state within the **last 15 minutes**.

**Datadog Monitor ID:** `537592`

**Query Logic (simplified):**

* Metric: `kubernetes_state.pod.status_phase`
* Filter:

  * `pod_phase: failed`
  * `kube_cluster_name: caas`
  * Selected system and platform namespaces
  * `env IN (prod, stg)`
* Condition: `>= 1`

---

### Prerequisites

* Access to the affected Kubernetes cluster
* Permission to run `kubectl` commands
* Familiarity with Kubernetes pod lifecycle and termination reasons
* Access to Datadog dashboards and alerts
* Proper cluster context configured (kubeconfig / Rancher)

---

### Tools Required

* `kubectl`
* Datadog UI (dashboards & alerts)

---

## Investigation Steps

### 1. Identify Failed Pods

```bash
kubectl get pods -n <NAMESPACE> \
  --field-selector=status.phase=Failed
```

---

### 2. Inspect Pod Details and Termination Reason

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

Retrieve failure reason and message:

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.reason}{"\n"}{.status.message}{"\n"}'
```

---

### 3. Check Container Exit Codes

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.containerStatuses[*].state.terminated.exitCode}{"\n"}'
```

---

### 4. Review Recent Events

```bash
kubectl get events -n <NAMESPACE> \
  --field-selector involvedObject.name=<POD_NAME> \
  --sort-by=.lastTimestamp
```

Look specifically for:

* `OOMKilled`
* `Error`
* `FailedScheduling`
* `NodeLost`

---

### 5. Node & Scheduling Diagnostics

```bash
kubectl get nodes
kubectl describe node <NODE_NAME>
```

Check pod-to-node mapping:

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -i node
```

Look for node conditions such as:

* `MemoryPressure`
* `DiskPressure`
* `Insufficient CPU`
* `Insufficient Memory`

---

## Remediation Actions

### Quick Fixes

#### Restart the Workload

```bash
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

#### Restart the Pod

```bash
kubectl delete pod <POD_NAME> -n <NAMESPACE>
```

#### Drain a Problematic Node (if transient issue)

```bash
kubectl cordon <NODE_NAME>
kubectl drain <NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

---

### Resource Adjustment (for OOMKilled Issues)

Update resource requests and limits:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

### Fix Configuration Issues

Edit configuration objects if misconfigured:

```bash
kubectl edit configmap <CONFIGMAP_NAME> -n <NAMESPACE>
kubectl edit secret <SECRET_NAME> -n <NAMESPACE>
```

---

## Common Root Causes

### 1. Application-Level Issues

* Application crash on startup
* Missing or invalid environment variables
* Incorrect ConfigMap or Secret values
* Dependency failures (e.g., database unreachable)

### 2. Resource Exhaustion

* Memory limits causing `OOMKilled`
* CPU starvation or throttling
* Node-level memory or disk pressure

### 3. Infrastructure & Scheduling Issues

* Taints, affinities, or resource constraints
* Node unreachable or lost
* Image pull failures

---

## Permanent Fixes & Prevention

* Improve application resilience

  * Add retry logic
  * Implement graceful shutdowns
  * Validate configuration at startup

* Tune resource requests and limits appropriately

* Review and simplify scheduling constraints (node affinity, taints)

---

## Post-Remediation Verification

* Confirm the pod is no longer in *Failed* state
* Ensure the workload is running and healthy
* Verify the Datadog alert has cleared
* Document findings and resolution steps in the incident tracker or SOP

---

### Important Note

Pods entering a *Failed* state often indicate **deeper application or infrastructure problems**. Always prioritize **root-cause analysis** and ensure workloads are properly configured, resilient, and adequately resourced.

---

If you want, I can also:

* Convert this into a **runbook template**
* Optimize it for **on-call response**
* Map causes → fixes in a **decision table**
* Make it **Datadog alert–friendly (short version)**
