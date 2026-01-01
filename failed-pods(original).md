# Kubernetes Failed Pods - Standard Operating Procedure

## Overview

This procedure addresses Kubernetes pods that have entered a Failed phase, indicating that containers terminated unexpectedly without automatic restart capability. Failed pods typically signal application crashes, configuration errors, resource limits exceeded, or infrastructure problems.

**For New Team Members:** This SOP serves as both a troubleshooting guide and a learning resource. Follow each step carefully, and don't hesitate to ask for help from senior team members when needed.

## Alert Configuration

**Monitor ID:** Datadog Monitor - 537592

**Trigger Condition:** One or more pods in Failed status detected within the last 15 minutes

**Monitored Scope:**
- Cluster: caas
- Environments: prod, stg
- Namespaces: aks-command, argocd, caas-logging, cattle-fleet-system, cattle-impersonation, cattle-logging, cattle-system, calico-system, cloudability, datadog, fleet-system, flagger-system, flux-system, kube-node-lease, kube-public, kube-system, kured, mop-rancher, secrets-system, temporal, temporal-system, tigera-operator, ngc-tooling-system, ngc-automation, keda, kyverno, victoria-metrics, wiz

## Requirements

**Access & Permissions:**
- Kubernetes cluster access
- kubectl command execution rights
- Datadog dashboard access
- Configured cluster context (kubeconfig or Rancher)

**Knowledge:**
- Pod lifecycle understanding
- Container termination causes
- Kubernetes troubleshooting fundamentals

**New to Kubernetes?** Review the "Understanding Pod Failures" section below before proceeding.

---

## Understanding Pod Failures (Learning Section)

### What is a Pod?
A pod is the smallest deployable unit in Kubernetes, containing one or more containers that run your application.

### Pod Lifecycle Phases
- **Pending:** Pod is accepted but not yet running
- **Running:** Pod is bound to a node and containers are running
- **Succeeded:** All containers terminated successfully
- **Failed:** At least one container terminated with an error (this is what we're troubleshooting)
- **Unknown:** Pod status cannot be determined

### Why Do Pods Fail?
Pods fail when containers inside them exit with a non-zero exit code or are terminated by the system. Common reasons include:
- Application crashes or bugs
- Out of memory errors (OOMKilled)
- Missing configuration files
- Failed health checks
- Node failures

### Key Concepts to Understand

**Exit Codes:**
- `0`: Success
- `1`: General application error
- `137`: OOMKilled (Out of Memory - killed by system)
- `143`: SIGTERM (graceful termination requested)
- `255`: Exit status out of range

**Resource Requests vs Limits:**
- **Requests:** Minimum resources guaranteed to the pod
- **Limits:** Maximum resources the pod can use before being throttled or killed

**Events:** Kubernetes generates events for significant pod lifecycle changes. These are your first clue when troubleshooting.

---

## Investigation Steps

Follow these steps in order. Each step builds on the previous one to help you understand the problem.

### Step 1: Identify Failed Pods

**What you're doing:** Finding which pods are in Failed state

```bash
kubectl get pods -n <NAMESPACE> --field-selector=status.phase=Failed
```

**What to look for:**
- Pod name
- Age (how long ago it failed)
- Restart count (indicates if this is recurring)

**Example Output:**
```
NAME                    READY   STATUS   RESTARTS   AGE
my-app-pod-abc123       0/1     Failed   0          5m
```

**Learning Tip:** The `--field-selector` flag filters pods by their status. You can also use `kubectl get pods -A` to see pods across all namespaces.

---

### Step 2: Examine Pod Details

**What you're doing:** Getting detailed information about why the pod failed

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

**What to look for:**
- **Last State:** Shows the termination reason
- **Exit Code:** Tells you how the container exited
- **Reason:** May show "Error", "OOMKilled", "Completed", etc.
- **Message:** Human-readable explanation
- **Events section:** Chronological list of what happened

**Alternative command for quick status:**
```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.reason}{"\n"}{.status.message}{"\n"}'
```

**Learning Tip:** The `describe` command is your best friend for troubleshooting. It shows comprehensive information about the resource.

---

### Step 3: Check Container Exit Codes

**What you're doing:** Finding the specific exit code to understand how the container stopped

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.containerStatuses[*].state.terminated.exitCode}{"\n"}'
```

**Exit Code Reference:**
- **0:** Clean exit (check why it exited if it shouldn't have)
- **1:** General error (check application logs)
- **137:** OOMKilled - pod used too much memory
- **143:** Received SIGTERM signal (usually during shutdown)
- **126:** Command cannot execute
- **127:** Command not found

**Learning Tip:** Exit codes above 128 typically indicate the process was terminated by a signal. Subtract 128 from the exit code to get the signal number (e.g., 137 - 128 = 9 = SIGKILL).

---

### Step 4: Review Recent Events

**What you're doing:** Looking at the timeline of events that led to the failure

```bash
kubectl get events -n <NAMESPACE> \
  --field-selector involvedObject.name=<POD_NAME> \
  --sort-by='.lastTimestamp'
```

**Key Event Messages to Look For:**
- **"OOMKilled":** Container exceeded memory limit
- **"Error":** Generic container error
- **"FailedScheduling":** Pod couldn't be placed on any node
- **"NodeLost":** Node became unavailable
- **"BackOff":** Container is in crash loop
- **"ImagePullBackOff":** Cannot pull container image

**Learning Tip:** Events are temporary and are cleaned up after a few hours. Capture them early when investigating issues.

---

### Step 5: Inspect Node Health

**What you're doing:** Checking if the node where the pod ran has any issues

```bash
# List all nodes and their status
kubectl get nodes

# Get detailed information about a specific node
kubectl describe node <NODE_NAME>

# Find which node the pod was on
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -i "node"
```

**What to look for in node description:**
- **Conditions:** Ready, DiskPressure, MemoryPressure, PIDPressure
- **Allocatable resources:** How much CPU/memory is available
- **Allocated resources:** How much is being used

**Learning Tip:** If a node shows MemoryPressure or DiskPressure, it may start evicting pods to free up resources.

---

### Step 6: Check Resource Constraints

**What you're doing:** Determining if the node is running out of resources

```bash
kubectl describe node <NODE_NAME>
```

**Resource Issues to Look For:**
- **Insufficient memory:** Node doesn't have enough free memory
- **Insufficient cpu:** Node doesn't have enough CPU
- **DiskPressure:** Node is running out of disk space
- **PIDPressure:** Too many processes running

**How to Read Resource Information:**
```
Allocated resources:
  CPU Requests: 1500m (75% of 2 cores)
  Memory Requests: 3Gi (75% of 4Gi)
```

**Learning Tip:** "m" stands for millicores (1000m = 1 CPU core). "Gi" is Gibibytes (binary gigabytes).

---

## Common Root Causes & How to Diagnose

### 1. Application Issues

**Symptoms:**
- Exit code 1 or other non-zero codes (except 137)
- Error messages in pod description
- Events showing "Error" or "CrashLoopBackOff"

**How to Investigate:**
```bash
# Check application logs (even for failed pods)
kubectl logs <POD_NAME> -n <NAMESPACE>

# Check previous container logs if pod restarted
kubectl logs <POD_NAME> -n <NAMESPACE> --previous

# Check environment variables
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].env}'
```

**Common Application Problems:**
- Unhandled exceptions during startup or runtime
- Missing or incorrectly formatted environment variables
- Invalid configmaps or secrets
- Unreachable dependencies (databases, APIs, services)
- Configuration errors in application code

**Learning Tip:** Always check logs first. They usually contain stack traces or error messages that point to the exact problem.

---

### 2. Resource Problems (OOMKilled)

**Symptoms:**
- Exit code 137
- Reason shows "OOMKilled" in pod description
- Events show "OOMKilling" message

**How to Investigate:**
```bash
# Check pod resource limits
kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].resources}'

# Check actual memory usage (if pod is still running)
kubectl top pod <POD_NAME> -n <NAMESPACE>
```

**Why This Happens:**
- Pod's memory limit is set too low for the workload
- Memory leak in the application
- Unexpected spike in traffic causing high memory usage
- Node running out of memory affecting all pods

**Learning Tip:** OOMKilled is one of the most common pod failure reasons. It means the kernel killed the process because it exceeded its memory limit.

---

### 3. Infrastructure & Scheduling Issues

**Symptoms:**
- Pod stuck in Pending before failing
- Events show "FailedScheduling"
- Node-related errors in events

**How to Investigate:**
```bash
# Check pod scheduling requirements
kubectl get pod <POD_NAME> -n <NAMESPACE> -o yaml | grep -A 20 "affinity\|nodeSelector\|tolerations"

# Check available nodes
kubectl get nodes -o wide

# Check node taints
kubectl describe node <NODE_NAME> | grep -i taint
```

**Common Infrastructure Problems:**
- Scheduling failures due to taints, affinity rules, or resource unavailability
- Node unreachable or removed from cluster
- Image pull failures leading to pod termination
- Network issues preventing pod startup

**Learning Tip:** Taints and tolerations control which pods can run on which nodes. If a pod can't tolerate a node's taints, it won't be scheduled there.

---

## Resolution Actions

### For New Team Members: When to Escalate

**Handle yourself:**
- Restarting a deployment or deleting a failed pod in non-prod
- Investigating and documenting the issue
- Applying resource limit increases after approval

**Escalate immediately:**
- Production outages affecting users
- Node failures or cluster-wide issues
- Security-related failures
- Uncertainty about the impact of a fix

---

### Immediate Fixes

#### Fix 1: Restart Deployment (Safe for most cases)

**When to use:** Application errors, transient issues, configuration changes

```bash
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

**What this does:** Creates new pods with the same configuration, gracefully replacing old ones

**Learning Tip:** This is the safest fix because it doesn't delete anything immediately. Old pods stay until new ones are healthy.

---

#### Fix 2: Delete Failed Pod (Use with caution)

**When to use:** Pod is stuck in Failed state and won't restart automatically

```bash
kubectl delete pod <POD_NAME> -n <NAMESPACE>
```

**What this does:** Forcefully removes the pod. If it's managed by a deployment/statefulset, a new pod will be created automatically.

**⚠️ Warning:** Only do this if you're sure the pod is managed by a controller (deployment, statefulset, daemonset). Standalone pods won't be recreated!

---

#### Fix 3: Drain Problematic Node

**When to use:** Node showing issues, multiple pods failing on the same node

```bash
# Prevent new pods from being scheduled on the node
kubectl cordon <NODE_NAME>

# Move all pods off the node safely
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

**What this does:** Safely evicts all pods from the node and reschedules them elsewhere

**⚠️ Escalate First:** This impacts multiple workloads. Get approval before draining production nodes.

**Learning Tip:** `cordon` prevents new pods from scheduling. `drain` goes further and moves existing pods. Always `cordon` before `drain`.

---

#### Fix 4: Adjust Memory Limits (OOMKilled cases)

**When to use:** Exit code 137 (OOMKilled), after confirming the pod legitimately needs more memory

```bash
# Quick patch (temporary fix)
kubectl patch deploy <DEPLOYMENT_NAME> -n <NAMESPACE> \
  -p='{"spec":{"template":{"spec":{"containers":[{"name":"<CONTAINER_NAME>","resources":{"limits":{"memory":"1Gi"}}}]}}}}'
```

**Better approach:** Update the deployment YAML file permanently

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

**How to decide on memory values:**
1. Check historical usage: `kubectl top pod <POD_NAME> -n <NAMESPACE>`
2. Set requests to average usage
3. Set limits to 1.5-2x the average (provides buffer for spikes)

**Learning Tip:** Always set both requests and limits. Requests ensure the pod gets scheduled, limits prevent it from consuming too much.

---

#### Fix 5: Update Configuration

**When to use:** Configuration errors in ConfigMaps or Secrets

```bash
# Edit ConfigMap
kubectl edit configmap <CONFIGMAP_NAME> -n <NAMESPACE>

# Edit Secret
kubectl edit secret <SECRET_NAME> -n <NAMESPACE>
```

**After editing:** Restart the deployment to pick up changes

```bash
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

**Learning Tip:** ConfigMaps and Secrets aren't automatically reloaded by pods. You need to restart the pods or configure your application to watch for changes.

---

## Long-Term Solutions (Preventing Recurrence)

### 1. Enhance Application Resilience

**What to do:**
- Implement retry logic for external dependencies
- Add graceful shutdown handlers
- Add startup configuration validation
- Improve error handling and logging
- Implement health checks (readiness and liveness probes)

**Example: Adding Health Checks**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Learning Tip:** Liveness probes restart unhealthy pods. Readiness probes remove unhealthy pods from service traffic. Both help maintain application availability.

---

### 2. Optimize Resource Allocation

**Proper Resource Configuration:**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**Best Practices:**
- Monitor actual usage over time
- Set requests based on average usage
- Set limits with 50-100% buffer for spikes
- Use Vertical Pod Autoscaler (VPA) for automatic tuning
- Consider Horizontal Pod Autoscaler (HPA) for scaling

**How to Monitor Resource Usage:**
```bash
# Current usage
kubectl top pod -n <NAMESPACE>

# Check metrics in Datadog
# Navigate to Infrastructure -> Kubernetes -> Pods
```

---

### 3. Refine Scheduling Configuration

**When to use affinity rules:**
- Keep related pods together (affinity)
- Spread pods across nodes for high availability (anti-affinity)
- Restrict pods to specific node types (node affinity)

**Example: Node Affinity (for SSD nodes)**
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**Example: Pod Anti-Affinity (spread across nodes)**
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - my-app
        topologyKey: kubernetes.io/hostname
```

**Learning Tip:** "Required" rules are hard requirements. "Preferred" rules are soft preferences. Use "preferred" when possible for more flexible scheduling.

---

## Verification Checklist

After implementing fixes, verify the following:

- [ ] Pod status is no longer Failed (should be Running or Completed)
- [ ] Check pod is actually running: `kubectl get pod <POD_NAME> -n <NAMESPACE>`
- [ ] Workload is passing health checks: `kubectl describe pod <POD_NAME> -n <NAMESPACE>` (check Conditions)
- [ ] Application logs show no errors: `kubectl logs <POD_NAME> -n <NAMESPACE>`
- [ ] Datadog monitor shows the alert has cleared
- [ ] No new Failed pods appear in the affected namespace (monitor for 15-30 minutes)
- [ ] Service is responding correctly (test endpoints if applicable)

**For Learning:** Run through this checklist every time. It builds good troubleshooting habits.

---

## Documentation & Knowledge Sharing

### What to Document

Record the following in your incident tracking system or team wiki:

**Issue Details:**
- Date and time of incident
- Failed pod details (name, namespace, cluster)
- Alert ID and monitor information
- Affected services or users

**Investigation Summary:**
- Exit code and failure reason
- Relevant events and logs
- Node and resource status
- Any patterns observed

**Resolution:**
- Root cause identified
- Resolution steps taken (with commands used)
- Who performed the fix
- Time to resolution

**Prevention:**
- Permanent fixes implemented
- Configuration changes made
- Recommendations for preventing recurrence
- Follow-up tasks created

### Knowledge Sharing

**For New Team Members:**
- Share your findings in team chat or standup
- Add common failure patterns to team documentation
- Create runbook entries for recurring issues
- Ask questions about anything unclear

**Building Expertise:**
- Review similar past incidents to learn patterns
- Pair with senior engineers during investigations
- Document your learning in personal notes
- Contribute to improving this SOP

---

## Quick Reference Card

### Most Common Scenarios

**Scenario 1: OOMKilled (Exit Code 137)**
```bash
# Confirm OOMKill
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -i oom

# Increase memory limit
kubectl patch deploy <DEPLOYMENT_NAME> -n <NAMESPACE> \
  -p='{"spec":{"template":{"spec":{"containers":[{"name":"<CONTAINER_NAME>","resources":{"limits":{"memory":"1Gi"}}}]}}}}'
```

**Scenario 2: Application Error (Exit Code 1)**
```bash
# Check logs
kubectl logs <POD_NAME> -n <NAMESPACE> --previous

# Restart deployment
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

**Scenario 3: Configuration Issue**
```bash
# Check config
kubectl get configmap <CONFIGMAP_NAME> -n <NAMESPACE> -o yaml

# Edit and restart
kubectl edit configmap <CONFIGMAP_NAME> -n <NAMESPACE>
kubectl rollout restart deployment <DEPLOYMENT_NAME> -n <NAMESPACE>
```

**Scenario 4: Node Problem**
```bash
# Check node status
kubectl get nodes
kubectl describe node <NODE_NAME>

# Drain if needed (escalate first in prod)
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

---

## Additional Learning Resources

### Internal Resources
- Team wiki: [Link to your team's Kubernetes documentation]
- Slack channels: #kubernetes-help, #incident-response
- Datadog dashboards: [Links to relevant dashboards]
- Previous incidents: [Link to incident tracker]

### External Resources
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Exit Code Reference](https://tldp.org/LDP/abs/html/exitcodes.html)

### Practice Safely
- Use development or staging environments for learning
- Run commands with `--dry-run=client` to see what would happen
- Always have a senior engineer review changes in production

---

## Getting Help

**When You're Stuck:**
1. Document what you've tried so far
2. Note any error messages or unexpected behavior
3. Reach out in #kubernetes-help with your findings
4. Tag on-call engineer for production issues

**Escalation Path:**
- Level 1: Team lead or senior engineer
- Level 2: Platform team / SRE team
- Level 3: On-call manager

**Remember:** It's always better to ask for help than to make changes you're uncertain about, especially in production!

---

## Important Reminders

⚠️ **Before Making Changes:**
- Understand the impact of your fix
- Check if it's production or non-production
- Have a rollback plan
- Get approval for high-impact changes

✅ **After Fixing Issues:**
- Verify the fix worked
- Document what you did
- Share learnings with the team
- Create tickets for permanent fixes

🎯 **Building Expertise:**
- Each incident is a learning opportunity
- Don't just fix it—understand why it happened
- Ask "why" multiple times to find root causes
- Contribute to team knowledge base

---

**Note:** Failed pods often indicate systemic issues requiring deeper investigation. Always perform root cause analysis and implement permanent fixes to prevent recurrence. As you gain experience, you'll develop intuition for diagnosing issues quickly, but always follow the investigative steps to ensure you haven't missed anything.