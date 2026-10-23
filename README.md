# Kubernetes Troubleshooting Labs

A hands-on Kubernetes troubleshooting repository focused on diagnosing and remediating common problems faced by Cloud Engineers, DevOps Engineers, Platform Engineers, SREs, and Cloud Security Engineers.

This repository contains **40 practical Kubernetes troubleshooting scenarios** covering Pods, Deployments, Services, networking, DNS, storage, autoscaling, RBAC, security policies, AWS EKS, IAM, Ingress, Load Balancers, node failures, and end-to-end application troubleshooting.

The main goal is not only to fix Kubernetes problems, but to understand:

- What failed
- Why it failed
- How to investigate it
- How to identify the root cause
- How to remediate the issue
- How to verify the fix
- How to prevent the same issue in production

---

# Project Goal

Kubernetes problems are rarely solved by running one command.

A production problem may involve multiple layers:

```text
User
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
EndpointSlice
 ↓
Pod
 ↓
Container
 ↓
Application
 ↓
Database / External Service
```

In cloud environments such as AWS EKS, the troubleshooting path becomes even larger:

```text
Application
   │
   ├── Kubernetes Layer
   │    ├── Pod
   │    ├── Deployment
   │    ├── Service
   │    ├── DNS
   │    ├── Ingress
   │    ├── RBAC
   │    ├── NetworkPolicy
   │    ├── Storage
   │    └── Autoscaling
   │
   └── AWS Layer
        ├── IAM
        ├── EKS Pod Identity
        ├── Security Groups
        ├── VPC
        ├── Subnets
        ├── Route Tables
        ├── ALB / NLB
        ├── ECR
        ├── EBS
        └── EC2 Worker Nodes
```

This repository is designed to build a structured troubleshooting mindset instead of relying on random commands.

---

# Troubleshooting Methodology

Every lab follows the same troubleshooting lifecycle.

```text
Observe
   ↓
Identify Symptoms
   ↓
Inspect Kubernetes Resources
   ↓
Check Events
   ↓
Check Logs
   ↓
Check Configuration
   ↓
Test Networking
   ↓
Check Permissions
   ↓
Check Resource Usage
   ↓
Check Cloud Infrastructure
   ↓
Identify Root Cause
   ↓
Apply Remediation
   ↓
Verify Recovery
   ↓
Document Prevention Measures
```

The objective is always:

```text
Symptom
   ↓
Evidence
   ↓
Root Cause
   ↓
Remediation
   ↓
Verification
```

---

# Repository Structure

```text
kubernetes-troubleshooting-labs/
│
├── README.md
│
├── 01-pod-pending/
├── 02-crashloopbackoff/
├── 03-imagepullbackoff/
├── 04-errimagepull/
├── 05-pod-running-app-failing/
├── 06-service-cannot-reach-pods/
├── 07-wrong-target-port/
├── 08-dns-resolution-failure/
├── 09-wrong-namespace/
├── 10-pods-not-visible/
├── 11-wrong-kubernetes-context/
├── 12-kubectl-unauthorized/
├── 13-rbac-forbidden/
├── 14-eks-aws-access-denied/
├── 15-networkpolicy-blocking-traffic/
├── 16-oomkilled/
├── 17-cpu-throttling/
├── 18-node-notready/
├── 19-disk-pressure/
├── 20-memory-pressure/
├── 21-pod-eviction/
├── 22-pvc-pending/
├── 23-volume-mount-error/
├── 24-readiness-probe-failure/
├── 25-liveness-probe-restart-loop/
├── 26-deployment-rollout-failure/
├── 27-unavailable-deployment-replicas/
├── 28-ingress-404/
├── 29-loadbalancer-pending/
├── 30-configmap-not-updated/
├── 31-secret-not-updated/
├── 32-kubectl-exec-failure/
├── 33-database-connectivity-failure/
├── 34-internal-app-working-external-failing/
├── 35-invalid-kubernetes-yaml/
├── 36-nodeselector-scheduling-failure/
├── 37-taint-toleration-failure/
├── 38-hpa-not-scaling/
├── 39-pod-restarting/
└── 40-end-to-end-troubleshooting/
```

---

# Lab Structure

Each troubleshooting folder is designed to contain both the broken configuration and the remediation.

Example:

```text
02-crashloopbackoff/
│
├── README.md
├── broken.yaml
└── fixed.yaml
```

Some labs may also include additional files:

```text
lab-folder/
│
├── README.md
├── broken.yaml
├── fixed.yaml
├── service.yaml
├── configmap.yaml
├── secret-example.yaml
├── networkpolicy.yaml
└── architecture.md
```

Each lab README follows this structure:

```text
# Problem

# Scenario

# Architecture

# Symptoms

# Expected Behavior

# Broken Configuration

# How to Reproduce

# Initial Investigation

# Troubleshooting Commands

# Evidence

# Root Cause

# Remediation

# Fixed Configuration

# Verification

# Production Prevention

# Cloud Engineering Relevance

# Cloud Security Relevance

# Key Takeaways
```

---

# Core Troubleshooting Commands

These commands are used throughout the repository.

## Inspect Pods

```bash
kubectl get pods
```

Show Pods across all namespaces:

```bash
kubectl get pods -A
```

Show additional networking and node information:

```bash
kubectl get pods -o wide
```

---

## Inspect a Pod

```bash
kubectl describe pod <pod-name>
```

This is one of the most important Kubernetes troubleshooting commands.

It can reveal:

- Scheduling failures
- Probe failures
- Image pull problems
- Volume problems
- Container restart information
- Node assignment
- Kubernetes Events

---

# Check Logs

```bash
kubectl logs <pod-name>
```

Follow logs:

```bash
kubectl logs -f <pod-name>
```

If the container restarted:

```bash
kubectl logs <pod-name> --previous
```

For multi-container Pods:

```bash
kubectl logs <pod-name> -c <container-name>
```

---

# Check Events

```bash
kubectl get events
```

Sort events:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Events are extremely useful for identifying:

```text
FailedScheduling
FailedMount
FailedAttachVolume
BackOff
Unhealthy
FailedPull
Evicted
```

---

# Enter a Running Container

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Some images may contain Bash:

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

Minimal or distroless containers may contain neither shell.

---

# Check Deployments

```bash
kubectl get deployments
```

Detailed information:

```bash
kubectl describe deployment <deployment-name>
```

---

# Check ReplicaSets

```bash
kubectl get replicasets
```

---

# Check Services

```bash
kubectl get services
```

Short form:

```bash
kubectl get svc
```

Describe:

```bash
kubectl describe svc <service-name>
```

---

# Check Service Endpoints

```bash
kubectl get endpoints
```

Modern clusters commonly use EndpointSlices:

```bash
kubectl get endpointslices
```

This is especially useful when a Service exists but does not reach any Pods.

---

# Check Ingress

```bash
kubectl get ingress
```

Describe:

```bash
kubectl describe ingress <ingress-name>
```

---

# Check Nodes

```bash
kubectl get nodes
```

Detailed information:

```bash
kubectl describe node <node-name>
```

---

# Check Resource Usage

```bash
kubectl top pods
```

```bash
kubectl top nodes
```

These commands are useful for:

```text
OOMKilled
MemoryPressure
CPU throttling
Capacity planning
HPA troubleshooting
```

---

# Check Storage

```bash
kubectl get pvc
```

```bash
kubectl get pv
```

```bash
kubectl get storageclass
```

Describe a claim:

```bash
kubectl describe pvc <pvc-name>
```

---

# Check RBAC

Test whether the current identity has permission:

```bash
kubectl auth can-i get pods
```

Example:

```bash
kubectl auth can-i delete pods -n production
```

List Roles:

```bash
kubectl get role -A
```

List RoleBindings:

```bash
kubectl get rolebinding -A
```

List ClusterRoles:

```bash
kubectl get clusterrole
```

List ClusterRoleBindings:

```bash
kubectl get clusterrolebinding
```

---

# Check Kubernetes Context

```bash
kubectl config current-context
```

List configured contexts:

```bash
kubectl config get-contexts
```

Switch context:

```bash
kubectl config use-context <context-name>
```

This is especially important when working with multiple clusters such as:

```text
development
staging
production
```

---

# Validate Kubernetes YAML

Client-side dry run:

```bash
kubectl apply --dry-run=client -f deployment.yaml
```

Inspect available fields:

```bash
kubectl explain deployment
```

Example:

```bash
kubectl explain deployment.spec
```

---

# Lab Categories

The repository is divided into several major troubleshooting categories.

---

# 1. Pod and Container Troubleshooting

Labs:

```text
01-pod-pending
02-crashloopbackoff
03-imagepullbackoff
04-errimagepull
05-pod-running-app-failing
16-oomkilled
21-pod-eviction
24-readiness-probe-failure
25-liveness-probe-restart-loop
32-kubectl-exec-failure
39-pod-restarting
```

These labs focus on problems occurring inside or directly around Pods and containers.

Common symptoms include:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
OOMKilled
Evicted
0/1 Ready
High Restart Count
```

Typical root causes include:

```text
Application crashes
Missing configuration
Wrong environment variables
Bad container command
Insufficient CPU
Insufficient memory
Health probe errors
Image registry failures
```

---

# 2. Service and Networking Troubleshooting

Labs:

```text
06-service-cannot-reach-pods
07-wrong-target-port
08-dns-resolution-failure
15-networkpolicy-blocking-traffic
28-ingress-404
29-loadbalancer-pending
33-database-connectivity-failure
34-internal-app-working-external-failing
```

These labs focus on Kubernetes traffic flow.

Typical architecture:

```text
Client
 ↓
Ingress
 ↓
Service
 ↓
EndpointSlice
 ↓
Pod
 ↓
Application
```

Common causes include:

```text
Wrong Service selector
Incorrect targetPort
DNS failure
NetworkPolicy denial
Ingress routing errors
Load Balancer provisioning failure
Firewall or Security Group rules
```

---

# 3. Kubernetes Identity and Access Troubleshooting

Labs:

```text
11-wrong-kubernetes-context
12-kubectl-unauthorized
13-rbac-forbidden
14-eks-aws-access-denied
```

These labs distinguish between several important concepts.

```text
Authentication
=
Who are you?

Authorization
=
What are you allowed to do?
```

Cloud environments add another identity layer:

```text
Kubernetes Identity
+
Cloud IAM Identity
```

For AWS EKS:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity / IAM Integration
 ↓
IAM Role
 ↓
AWS Service
```

---

# 4. Node Troubleshooting

Labs:

```text
18-node-notready
19-disk-pressure
20-memory-pressure
21-pod-eviction
36-nodeselector-scheduling-failure
37-taint-toleration-failure
```

These scenarios focus on Kubernetes worker nodes.

Important conditions include:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Node problems can affect many Pods at the same time.

---

# 5. Storage Troubleshooting

Labs:

```text
22-pvc-pending
23-volume-mount-error
```

Kubernetes storage generally follows:

```text
Pod
 ↓
PersistentVolumeClaim
 ↓
PersistentVolume
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Cloud Storage
```

For AWS:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
EBS CSI Driver
 ↓
Amazon EBS
```

Storage troubleshooting requires checking both Kubernetes and cloud infrastructure.

---

# 6. Deployment and Release Troubleshooting

Labs:

```text
26-deployment-rollout-failure
27-unavailable-deployment-replicas
30-configmap-not-updated
31-secret-not-updated
```

These labs cover application deployment problems such as:

```text
Failed rollout
Bad application version
Unavailable replicas
Stale configuration
Stale secrets
```

Useful commands include:

```bash
kubectl rollout status deployment/<deployment-name>
```

```bash
kubectl rollout history deployment/<deployment-name>
```

Rollback:

```bash
kubectl rollout undo deployment/<deployment-name>
```

Restart:

```bash
kubectl rollout restart deployment/<deployment-name>
```

---

# 7. Scheduling Troubleshooting

Labs:

```text
01-pod-pending
36-nodeselector-scheduling-failure
37-taint-toleration-failure
```

Kubernetes scheduling can fail because of:

```text
Insufficient CPU
Insufficient Memory
NodeSelector mismatch
Node affinity mismatch
Taints
Tolerations
Storage topology
PVC requirements
```

The first command to use is usually:

```bash
kubectl describe pod <pod-name>
```

Then check the Events section.

---

# 8. Autoscaling Troubleshooting

Lab:

```text
38-hpa-not-scaling
```

Horizontal Pod Autoscaler depends on metrics and workload configuration.

Typical flow:

```text
Application Load
 ↓
CPU / Memory Metric
 ↓
Metrics System
 ↓
HPA
 ↓
Deployment
 ↓
More or Fewer Pods
```

Common problems include:

```text
Metrics unavailable
Metrics Server unavailable
Missing CPU requests
Incorrect target
Incorrect min/max replicas
Application load not reaching threshold
```

---

# 9. Kubernetes Configuration Troubleshooting

Labs:

```text
09-wrong-namespace
10-pods-not-visible
30-configmap-not-updated
31-secret-not-updated
35-invalid-kubernetes-yaml
```

These problems commonly result from:

```text
Wrong namespace
Wrong kubectl context
Invalid YAML
Wrong API field
Stale configuration
Missing configuration
```

---

# 10. AWS EKS Troubleshooting

Several labs include EKS-specific scenarios:

```text
03-imagepullbackoff
11-wrong-kubernetes-context
12-kubectl-unauthorized
14-eks-aws-access-denied
18-node-notready
22-pvc-pending
23-volume-mount-error
29-loadbalancer-pending
34-internal-app-working-external-failing
40-end-to-end-troubleshooting
```

The objective is to understand that an EKS issue may exist in Kubernetes or AWS.

```text
Problem
 │
 ├── Kubernetes?
 │    ├── Pod
 │    ├── Service
 │    ├── Ingress
 │    ├── RBAC
 │    ├── DNS
 │    ├── Storage
 │    └── NetworkPolicy
 │
 └── AWS?
      ├── IAM
      ├── EKS
      ├── EC2
      ├── ECR
      ├── EBS
      ├── VPC
      ├── Subnets
      ├── Security Groups
      └── ALB / NLB
```

---

# Troubleshooting Remediation Strategy

A major focus of this repository is remediation.

The objective is not:

```text
Make error disappear
```

The objective is:

```text
Find root cause
      ↓
Apply minimum necessary change
      ↓
Restore service
      ↓
Validate security
      ↓
Prevent recurrence
```

---

# Remediation Principle 1 — Fix the Root Cause

Example:

```text
Pod Pending
```

Bad remediation:

```text
Keep deleting the Pod
```

Correct approach:

```text
Inspect Pod
 ↓
Find FailedScheduling event
 ↓
Identify insufficient memory
 ↓
Adjust workload or node capacity
 ↓
Verify scheduling
```

---

# Remediation Principle 2 — Avoid Over-Permissioning

Example:

```text
RBAC Forbidden
```

Bad fix:

```text
Give cluster-admin
```

Better:

```text
Determine required operation
 ↓
Create least-privilege Role
 ↓
Bind correct user or ServiceAccount
 ↓
Verify with kubectl auth can-i
```

Cloud Security principle:

```text
Minimum required privilege
```

---

# Remediation Principle 3 — Do Not Disable Security Controls Just to Restore Connectivity

Example:

```text
NetworkPolicy blocks application
```

Bad remediation:

```text
Delete all NetworkPolicies
```

Better:

```text
Identify required traffic path
 ↓
Identify source workload
 ↓
Identify destination workload
 ↓
Identify required port
 ↓
Create precise allow rule
 ↓
Keep default-deny posture
```

---

# Remediation Principle 4 — Validate After Every Fix

After remediation:

```text
Check Pod
 ↓
Check Deployment
 ↓
Check Service
 ↓
Test connectivity
 ↓
Check logs
 ↓
Check events
```

Example:

```bash
kubectl get pods
```

```bash
kubectl rollout status deployment/backend
```

```bash
kubectl get svc
```

```bash
kubectl get endpointslices
```

---

# Remediation Principle 5 — Roll Back Bad Releases

If a new version breaks production:

```text
v1 working
 ↓
v2 deployment
 ↓
errors
 ↓
rollback
```

Command:

```bash
kubectl rollout undo deployment/<deployment-name>
```

Then investigate the new version separately.

Availability should be restored before continuing risky debugging where possible.

---

# Remediation Principle 6 — Protect Credentials

Never solve credential problems by placing permanent credentials directly inside:

```text
Dockerfile
Git repository
Deployment YAML
Container image
Application source code
```

Prefer workload identity mechanisms.

For EKS:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity / appropriate IAM integration
 ↓
IAM Role
 ↓
AWS Service
```

---

# Remediation Principle 7 — Use Resource Requests and Limits Carefully

Requests help scheduling:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

Limits control maximum resources:

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Incorrect values can cause:

```text
Pending Pods
OOMKilled
CPU throttling
Poor cluster utilization
```

Resource configuration should be based on observed workload behavior.

---

# Remediation Principle 8 — Use Health Probes Correctly

Kubernetes health checks have different purposes.

```text
Readiness Probe
=
Can the Pod receive traffic?
```

```text
Liveness Probe
=
Should Kubernetes restart the container?
```

```text
Startup Probe
=
Has the application finished starting?
```

Poor probe configuration can create outages even when the application itself is healthy.

---

# Remediation Principle 9 — Troubleshoot Layer by Layer

Do not jump directly to the application.

Example external request:

```text
User
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress
 ↓
Service
 ↓
EndpointSlice
 ↓
Pod
 ↓
Application
```

Start at one end and verify each layer.

Example:

```text
DNS works?
 ↓
Load Balancer healthy?
 ↓
Ingress routing correct?
 ↓
Service selector correct?
 ↓
Endpoint exists?
 ↓
Pod ready?
 ↓
Application listening?
```

This makes troubleshooting much faster.

---

# Remediation Principle 10 — Use Evidence Before Changing Production

Before changing a resource, collect:

```text
Current state
Events
Logs
Metrics
Configuration
Recent deployment changes
```

Useful commands:

```bash
kubectl describe pod <pod>
```

```bash
kubectl logs <pod> --previous
```

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

```bash
kubectl get deployment <deployment> -o yaml
```

This helps avoid destroying useful evidence.

---

# Common Kubernetes Errors Covered

This repository covers errors and symptoms such as:

```text
Pending

CrashLoopBackOff

ImagePullBackOff

ErrImagePull

OOMKilled

Evicted

NotReady

DiskPressure

MemoryPressure

Unauthorized

Forbidden

FailedScheduling

FailedMount

FailedAttachVolume

Readiness probe failed

Liveness probe failed

Connection refused

DNS resolution failure

404 from Ingress

LoadBalancer Pending

PVC Pending

HPA metrics unknown

Deployment unavailable
```

---

# Kubernetes Troubleshooting Decision Tree

A simplified troubleshooting decision process:

```text
Application not working
        │
        ▼
Is Pod running?
        │
    ┌───┴────┐
    │        │
   No       Yes
    │        │
    ▼        ▼
Describe   Check readiness
Pod        and application logs
    │        │
    ▼        ▼
Events    Is Service working?
             │
         ┌───┴────┐
         │        │
        No       Yes
         │        │
         ▼        ▼
 Check selector  Check Ingress
 and endpoints       │
                     ▼
                Check Load Balancer
                     │
                     ▼
                 Check DNS
```

---

# Cloud Engineering Skills Demonstrated

This repository demonstrates practical experience with:

- Kubernetes architecture
- Pod troubleshooting
- Deployment troubleshooting
- Service networking
- Kubernetes DNS
- Ingress
- Load Balancers
- Resource management
- Autoscaling
- Persistent storage
- Scheduling
- Kubernetes RBAC
- Kubernetes networking
- Health probes
- Configuration management
- Kubernetes troubleshooting methodology
- AWS EKS
- IAM integration
- Amazon ECR
- Amazon EBS
- AWS Load Balancing
- Kubernetes observability

---

# Cloud Security Skills Demonstrated

Security-focused labs demonstrate:

- Least-privilege RBAC
- Kubernetes authentication
- Kubernetes authorization
- ServiceAccounts
- EKS workload identity
- IAM troubleshooting
- NetworkPolicy
- Network segmentation
- Secret management
- Avoiding static AWS credentials
- Security-aware remediation
- Production access control
- Troubleshooting without disabling security controls
- Kubernetes workload isolation

---

# Production Troubleshooting Mindset

A Cloud Engineer should avoid thinking:

```text
Something failed.
Restart everything.
```

Instead:

```text
Something failed.
      ↓
What changed?
      ↓
What layer failed?
      ↓
What evidence exists?
      ↓
What is the root cause?
      ↓
What is the safest remediation?
      ↓
How do I verify recovery?
      ↓
How do I stop this happening again?
```

That is the troubleshooting mindset used throughout this repository.

---

# Recommended Lab Order

For beginners:

```text
01 → 05
```

Learn Pods and containers first.

Then:

```text
06 → 10
```

Learn Services, ports, DNS, and namespaces.

Then:

```text
11 → 15
```

Learn authentication, RBAC, AWS IAM, and NetworkPolicy.

Then:

```text
16 → 23
```

Learn resources, nodes, eviction, and storage.

Then:

```text
24 → 31
```

Learn health checks, Deployments, Ingress, Load Balancers, and configuration.

Then:

```text
32 → 39
```

Learn advanced application and scheduling troubleshooting.

Finally:

```text
40-end-to-end-troubleshooting
```

This final lab combines multiple Kubernetes and AWS layers into one production-style incident.

---

# Final End-to-End Troubleshooting Model

The final goal of this repository is to become comfortable troubleshooting this entire path:

```text
                            USER
                              │
                              ▼
                         DNS / Route 53
                              │
                              ▼
                         ALB / NLB
                              │
                              ▼
                            Ingress
                              │
                              ▼
                            Service
                              │
                              ▼
                        EndpointSlice
                              │
                              ▼
                              Pod
                              │
                              ▼
                           Container
                              │
                              ▼
                         Application
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              Database       S3       External API
```

And simultaneously understand supporting infrastructure:

```text
                        Kubernetes Cluster
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
       Nodes                 Storage               Identity
         │                     │                     │
      kubelet                 PVC                ServiceAccount
      runtime                  │                     │
      network                  PV                RBAC / IAM
         │                     │                     │
         ▼                     ▼                     ▼
      EC2/EKS                EBS                  AWS IAM
```

---

# Key Troubleshooting Formula

The core formula used throughout every lab is:

```text
OBSERVE
   ↓
DESCRIBE
   ↓
LOGS
   ↓
EVENTS
   ↓
NETWORK
   ↓
PERMISSIONS
   ↓
RESOURCES
   ↓
ROOT CAUSE
   ↓
REMEDIATE
   ↓
VERIFY
```

Or simply:

```text
Get
Describe
Logs
Events
Test
Fix
Verify
```

---

# Project Purpose

This repository is built as a practical learning and portfolio project for Kubernetes, Cloud Engineering, AWS EKS, DevOps, Platform Engineering, and Cloud Security.

Instead of only showing how to deploy Kubernetes applications, the project focuses on the more important operational question:

> **What do I do when Kubernetes stops working?**

Each lab intentionally introduces a failure, investigates the failure using Kubernetes and cloud troubleshooting tools, identifies the root cause, applies a safe remediation, and verifies that the system has recovered.

The final result is a collection of realistic Kubernetes troubleshooting scenarios that demonstrate both infrastructure knowledge and operational problem-solving skills.
