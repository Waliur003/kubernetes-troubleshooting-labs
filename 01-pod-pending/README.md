# Lab 01 — Pod Stuck in Pending

## Overview

This lab demonstrates one of the most common Kubernetes scheduling problems: a Pod remains in the `Pending` state because the cluster does not have enough resources to satisfy the Pod's resource request.

In this lab, the Pod intentionally requests much more memory than the Kubernetes node can provide.

The goal is to learn how to:

- Identify a Pod stuck in `Pending`
- Use `kubectl describe` to investigate the problem
- Read Kubernetes scheduling events
- Understand resource requests and limits
- Compare Pod requirements with node capacity
- Identify the root cause
- Apply a safe remediation
- Verify that the Pod successfully reaches the `Running` state

---

# Architecture

The lab uses a simple Kubernetes cluster with one node.

```text
                Kubernetes Cluster
                        │
                        ▼
                    Scheduler
                        │
                        ▼
                  Minikube Node
                        │
                 Limited Memory
                        │
                        ▼
                 memory-hungry-pod
```

The broken Pod requests:

```text
100 GiB Memory
```

The node has far less memory available.

The result is:

```text
Pod
 ↓
Scheduler
 ↓
Check Node Resources
 ↓
Not Enough Memory
 ↓
FailedScheduling
 ↓
Pending
```

---

# Problem Scenario

A developer deploys a new application into Kubernetes.

The Pod is created successfully, but it never starts.

Running:

```bash
kubectl get pods
```

shows:

```text
NAME                READY   STATUS    RESTARTS   AGE
memory-hungry-pod   0/1     Pending   0          20s
```

The Pod remains in the `Pending` state.

The Cloud Engineer must determine why the Kubernetes scheduler cannot assign the Pod to a node.

---

# Expected Behavior

The expected behavior is:

```text
Pod Created
 ↓
Scheduler Finds Suitable Node
 ↓
Pod Assigned to Node
 ↓
Container Starts
 ↓
Pod Becomes Running
```

Expected status:

```text
READY   STATUS
1/1     Running
```

However, in the broken configuration, the scheduling process fails.

---

# Broken Configuration

File:

```text
broken.yaml
```

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: memory-hungry-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.27

      resources:
        requests:
          memory: "100Gi"
          cpu: "500m"

        limits:
          memory: "100Gi"
          cpu: "1"
```

The important configuration is:

```yaml
requests:
  memory: "100Gi"
```

The Pod tells Kubernetes:

```text
I require 100 GiB of memory before I can run.
```

The local Minikube node does not have anywhere close to 100 GiB available.

---

# Reproduce the Problem

## Step 1 — Start Kubernetes

If using Minikube:

```bash
minikube start
```

Check cluster status:

```bash
minikube status
```

Verify the node:

```bash
kubectl get nodes
```

Example:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   5m    v1.x.x
```

The node must be:

```text
Ready
```

before starting the lab.

---

# Step 2 — Check Node Capacity

Run:

```bash
kubectl describe node minikube
```

Look for:

```text
Capacity:
```

and:

```text
Allocatable:
```

Example:

```text
Capacity:
  cpu:                2
  memory:             3900000Ki

Allocatable:
  cpu:                2
  memory:             3700000Ki
```

The exact values depend on the local environment.

The important difference is:

```text
Capacity
=
Total resources on the node
```

and:

```text
Allocatable
=
Resources Kubernetes can provide to workloads
```

---

# Step 3 — Deploy the Broken Pod

Run:

```bash
kubectl apply -f broken.yaml
```

Expected output:

```text
pod/memory-hungry-pod created
```

This only confirms that Kubernetes accepted the Pod definition.

It does not mean the Pod successfully started.

---

# Step 4 — Observe the Problem

Run:

```bash
kubectl get pods
```

Expected result:

```text
NAME                READY   STATUS    RESTARTS   AGE
memory-hungry-pod   0/1     Pending   0          10s
```

The important value is:

```text
STATUS = Pending
```

---

# What Does Pending Mean?

A Pod in `Pending` usually means Kubernetes knows that the Pod exists, but the Pod has not successfully started.

One common reason is that the scheduler cannot find a suitable node.

Possible causes include:

```text
Insufficient CPU

Insufficient memory

NodeSelector mismatch

Node affinity mismatch

Untolerated taints

PersistentVolumeClaim problems

Storage topology problems

Node availability problems
```

In this lab, the problem is insufficient memory.

---

# Initial Investigation

When a Pod is stuck in `Pending`, the first useful command is:

```bash
kubectl describe pod memory-hungry-pod
```

The `describe` command provides detailed information about:

- Pod configuration
- Container configuration
- Resource requests
- Node assignment
- Pod conditions
- Kubernetes events

The most important section for this problem is:

```text
Events:
```

---

# Evidence

The Events section should contain something similar to:

```text
Warning  FailedScheduling

0/1 nodes are available:
1 Insufficient memory.
```

This is the strongest evidence for the root cause.

Kubernetes is explicitly reporting:

```text
FailedScheduling
```

and:

```text
Insufficient memory
```

---

# Troubleshooting Workflow

The troubleshooting process is:

```text
Pod Pending
     ↓
kubectl describe pod
     ↓
Check Events
     ↓
FailedScheduling
     ↓
Insufficient memory
     ↓
Check Pod resource requests
     ↓
Check node allocatable resources
     ↓
Compare requirements
     ↓
Identify root cause
```

---

# Resource Requests vs Limits

Kubernetes resource configuration commonly contains two important sections:

```yaml
requests:
```

and:

```yaml
limits:
```

## Requests

Example:

```yaml
requests:
  memory: "128Mi"
  cpu: "100m"
```

Requests represent the amount of resources Kubernetes should consider when scheduling the Pod.

The scheduler uses requests to determine whether a node has enough capacity.

Conceptually:

```text
Pod Request
      ↓
Scheduler
      ↓
Compare With Node Capacity
      ↓
Choose Node
```

If no node can satisfy the request, the Pod remains `Pending`.

---

# Limits

Example:

```yaml
limits:
  memory: "256Mi"
  cpu: "250m"
```

Limits define how much resource the container is allowed to consume.

Conceptually:

```text
Request
=
Scheduling requirement
```

```text
Limit
=
Maximum allowed resource usage
```

The two values serve different purposes.

---

# Root Cause

The root cause is:

```text
The Pod requests more memory than the Kubernetes node can provide.
```

Broken request:

```yaml
memory: "100Gi"
```

Node memory:

```text
Only a few GiB
```

Therefore:

```text
Pod Memory Request
>
Available Node Memory
```

The scheduler cannot place the Pod.

The result is:

```text
FailedScheduling
```

and:

```text
Pending
```

---

# Root Cause Diagram

```text
memory-hungry-pod
        │
        │ requests 100Gi
        ▼
   Kubernetes Scheduler
        │
        ▼
     Minikube Node
        │
        │ only a few GiB available
        ▼
Resource Requirement Cannot Be Satisfied
        │
        ▼
   FailedScheduling
        │
        ▼
      Pending
```

---

# Remediation

The remediation in this lab is to replace the unrealistic memory requirement with values that the node can actually provide.

Broken:

```yaml
resources:
  requests:
    memory: "100Gi"
    cpu: "500m"

  limits:
    memory: "100Gi"
    cpu: "1"
```

Fixed:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"

  limits:
    memory: "256Mi"
    cpu: "250m"
```

---

# Fixed Configuration

File:

```text
fixed.yaml
```

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: memory-hungry-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.27

      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"

        limits:
          memory: "256Mi"
          cpu: "250m"
```

---

# Apply the Remediation

Delete the broken Pod:

```bash
kubectl delete pod memory-hungry-pod
```

Expected:

```text
pod "memory-hungry-pod" deleted
```

Apply the corrected configuration:

```bash
kubectl apply -f fixed.yaml
```

Expected:

```text
pod/memory-hungry-pod created
```

---

# Verification

## Verify Pod Status

Run:

```bash
kubectl get pods
```

Expected:

```text
NAME                READY   STATUS    RESTARTS   AGE
memory-hungry-pod   1/1     Running   0          15s
```

The important result is:

```text
STATUS = Running
```

---

# Verify Node Assignment

Run:

```bash
kubectl get pod memory-hungry-pod -o wide
```

Example:

```text
NAME                READY   STATUS    NODE
memory-hungry-pod   1/1     Running   minikube
```

The presence of a node confirms that the scheduler successfully assigned the Pod.

---

# Verify With Describe

Run:

```bash
kubectl describe pod memory-hungry-pod
```

Check:

```text
Node:
```

and:

```text
Status:
```

Expected:

```text
Node: minikube/...
Status: Running
```

The previous `FailedScheduling` problem should no longer prevent the Pod from running.

---

# Verify Application Logs

Run:

```bash
kubectl logs memory-hungry-pod
```

Nginx should produce normal startup logs.

This confirms:

```text
Pod scheduled
 ↓
Container started
 ↓
Application started
```

---

# Verify Application Connectivity

Forward local port `8080` to container port `80`:

```bash
kubectl port-forward pod/memory-hungry-pod 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

The nginx welcome page HTML should be returned.

The full verification chain is:

```text
Pod Scheduled
 ↓
Pod Running
 ↓
Container Running
 ↓
Nginx Running
 ↓
Application Reachable
```

---

# Production Remediation Considerations

In this lab, reducing the memory request is the correct solution because the `100Gi` request was intentionally unrealistic.

In production, automatically reducing resource requests is not always the correct remediation.

Suppose an application genuinely requires:

```text
16 GiB RAM
```

Reducing the request to:

```text
128 MiB
```

might allow Kubernetes to schedule the Pod, but the application could later fail because it does not have enough resources.

A production Cloud Engineer should determine:

```text
What resources does the application actually require?
```

Then compare:

```text
Application Requirement
        vs
Cluster Capacity
```

The correct remediation may be different depending on the result.

---

# Production Remediation Options

## Option 1 — Reduce an Incorrect Resource Request

Use this when the application request is unnecessarily high.

Example:

```yaml
requests:
  memory: "512Mi"
```

instead of:

```yaml
requests:
  memory: "32Gi"
```

This should only be done when workload measurements support the change.

---

# Option 2 — Increase Node Capacity

If the application legitimately requires more memory, use larger nodes.

Example architecture:

```text
Current Worker Node
4 GiB RAM

        ↓

Larger Worker Node
32 GiB RAM
```

In AWS EKS, this may involve using a larger EC2 instance type in the node group.

---

# Option 3 — Add More Cluster Capacity

Additional worker nodes may help when existing workloads have consumed available capacity.

```text
Before

Worker Node 1
Nearly Full

Worker Node 2
Nearly Full
```

Add capacity:

```text
Worker Node 1

Worker Node 2

Worker Node 3
```

However, adding more nodes will not solve a Pod request that is larger than every individual node.

For example:

```text
Pod requests 100Gi
```

and every node has:

```text
8Gi
```

Adding ten more `8Gi` nodes still does not provide a single suitable node for that Pod.

This is an important Kubernetes scheduling concept.

---

# Option 4 — Use a Dedicated Node Group

Large workloads may require a separate node group.

Example:

```text
EKS Cluster
   │
   ├── General Node Group
   │      ├── 4 GiB
   │      ├── 4 GiB
   │      └── 4 GiB
   │
   └── Memory Optimized Node Group
          ├── 32 GiB
          └── 32 GiB
```

Large memory workloads can then be scheduled onto appropriate nodes using labels, affinity rules, and possibly taints/tolerations.

---

# Option 5 — Investigate Existing Resource Consumption

The node may technically be large enough, but existing workloads may already reserve most of its capacity.

Check:

```bash
kubectl describe node minikube
```

Look for:

```text
Allocated resources:
```

Also check usage:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -A
```

This helps determine whether another workload is consuming excessive resources.

---

# Important Scheduling Concept

Kubernetes does not simply ask:

```text
Does the server physically have enough RAM?
```

The scheduler also considers Kubernetes resource reservations and scheduling rules.

Conceptually:

```text
Node Allocatable Resources
       ↓
Subtract Existing Pod Requests
       ↓
Remaining Schedulable Capacity
       ↓
Can New Pod Request Fit?
```

If not:

```text
FailedScheduling
```

---

# Common Causes of Pending Pods

A `Pending` Pod is not always caused by memory.

Other common causes include:

## Insufficient CPU

Example event:

```text
0/1 nodes are available:
1 Insufficient cpu.
```

---

## NodeSelector Mismatch

The Pod requires:

```yaml
nodeSelector:
  workload: backend
```

but no node has:

```text
workload=backend
```

Result:

```text
Pending
```

---

## Untolerated Taint

A node has:

```text
dedicated=database:NoSchedule
```

but the Pod has no matching toleration.

Result:

```text
Pending
```

---

## Node Affinity Failure

A Pod requires a particular node label that no node satisfies.

Result:

```text
FailedScheduling
```

---

## PersistentVolumeClaim Problem

A Pod requires storage through a PVC.

If the PVC cannot bind:

```text
PVC Pending
```

the Pod may also remain unable to start.

---

# Troubleshooting Commands Learned

## Check Pod State

```bash
kubectl get pods
```

---

## Show Additional Pod Information

```bash
kubectl get pods -o wide
```

---

## Investigate Pod

```bash
kubectl describe pod memory-hungry-pod
```

---

## Check Kubernetes Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Inspect Nodes

```bash
kubectl get nodes
```

---

## Inspect Node Resources

```bash
kubectl describe node minikube
```

---

## Check Resource Usage

If Metrics Server is installed:

```bash
kubectl top nodes
```

```bash
kubectl top pods
```

---

# Troubleshooting Cheat Sheet

When seeing:

```text
Pending
```

follow:

```text
1. kubectl get pods

2. kubectl describe pod <pod>

3. Read Events

4. Look for FailedScheduling

5. Identify reason

6. Check node resources

7. Check Pod requests

8. Check scheduling constraints

9. Apply remediation

10. Verify Pod becomes Running
```

---

# Cloud Engineering Relevance

This type of issue is common in Cloud Engineering because engineers manage both applications and the infrastructure on which Kubernetes workloads run.

A Cloud Engineer may need to decide whether the correct solution is:

```text
Change Kubernetes resource configuration
```

or:

```text
Scale cloud infrastructure
```

In AWS EKS, this may involve:

- EC2 worker node instance types
- Managed Node Groups
- Cluster autoscaling
- Node labels
- Node affinity
- Taints and tolerations
- Workload resource requests
- Capacity planning

The Kubernetes error may therefore require a cloud infrastructure change rather than only a Kubernetes configuration change.

---

# Cloud Security Relevance

Resource configuration also has a security and resilience impact.

A workload with unrealistic resource requests can consume scheduling capacity or prevent important workloads from running.

Poor resource configuration may contribute to:

```text
Resource exhaustion
Denial of service
Cluster instability
Unexpected cloud costs
Reduced availability
```

Cloud Security Engineers should therefore understand resource isolation and resource governance.

Production environments may use controls such as:

```text
ResourceQuota
LimitRange
Namespaces
Admission policies
Monitoring
Autoscaling
```

to prevent poorly configured workloads from negatively affecting the cluster.

---

# Prevention

Several practices can help prevent this issue.

## Define Realistic Resource Requests

Do not guess resource requirements.

Use metrics and historical workload behavior.

---

## Monitor Cluster Capacity

Track:

```text
CPU usage
Memory usage
Pod requests
Pod limits
Node utilization
Pending Pods
```

---

## Use ResourceQuota

Namespaces can be given resource quotas to prevent uncontrolled resource usage.

---

## Use LimitRange

LimitRange can define defaults and constraints for Pod resources.

---

## Use Autoscaling

Production clusters may use node autoscaling to add capacity when Pods cannot schedule because available nodes are full.

However, autoscaling cannot solve every scheduling problem.

The requested Pod must still fit onto at least one available node type.

---

# Cleanup

Delete the lab Pod:

```bash
kubectl delete pod memory-hungry-pod
```

Verify:

```bash
kubectl get pods
```

The Pod should no longer exist.

---

# What I Learned

In this lab, I learned that a Kubernetes Pod may remain in the `Pending` state when the scheduler cannot find a node that satisfies its requirements.

I also learned that `kubectl describe pod` is one of the most important commands for troubleshooting scheduling problems because the Events section can show the exact reason for the failure.

The main troubleshooting pattern is:

```text
Pending Pod
 ↓
Describe Pod
 ↓
Check Events
 ↓
FailedScheduling
 ↓
Identify Scheduling Constraint
 ↓
Compare Pod Requirements With Node Capacity
 ↓
Apply Correct Remediation
 ↓
Verify Running State
```

I also learned that resource requests and limits have different purposes.

Requests influence Kubernetes scheduling, while limits restrict how much resource a running container may use.

The most important lesson is that production troubleshooting should not focus only on making the error disappear.

The correct process is:

```text
Identify Root Cause
 ↓
Understand Application Requirements
 ↓
Understand Cluster Capacity
 ↓
Choose Safe Remediation
 ↓
Verify Recovery
 ↓
Prevent Recurrence
```

---

# Key Takeaways

- `Pending` commonly indicates a scheduling problem.
- `kubectl describe pod` is the first major troubleshooting command.
- Always inspect the `Events` section.
- `FailedScheduling` tells you the scheduler could not find a suitable node.
- Resource requests influence scheduling.
- Resource limits control maximum resource consumption.
- Compare Pod requests with node allocatable capacity.
- Do not blindly reduce resource requests in production.
- The correct remediation may require changing the workload or scaling infrastructure.
- Always verify the Pod after remediation.
- Troubleshooting should identify the root cause, not only remove the symptom.

---

# Final Troubleshooting Flow

```text
              Pod Pending
                   │
                   ▼
          kubectl describe pod
                   │
                   ▼
              Check Events
                   │
                   ▼
          FailedScheduling
                   │
                   ▼
       "Insufficient memory"
                   │
                   ▼
        Inspect Pod Requests
                   │
                   ▼
        Inspect Node Capacity
                   │
                   ▼
         Compare Requirements
                   │
                   ▼
          Identify Root Cause
                   │
                   ▼
        Apply Safe Remediation
                   │
                   ▼
          Recreate / Redeploy
                   │
                   ▼
           Pod = Running
                   │
                   ▼
        Verify Application Works
```