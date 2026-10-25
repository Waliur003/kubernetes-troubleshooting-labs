# Lab 04 — ErrImagePull

## Overview

This lab demonstrates the Kubernetes `ErrImagePull` error.

`ErrImagePull` occurs when Kubernetes attempts to download a container image and the image pull fails.

This is usually the first image-related error you see before Kubernetes transitions into `ImagePullBackOff`.

The purpose of this lab is to learn how to:

- Recognize `ErrImagePull`
- Understand the relationship between `ErrImagePull` and `ImagePullBackOff`
- Use `kubectl describe pod`
- Read Kubernetes Events
- Identify exact image pull failure messages
- Distinguish image-not-found errors from authentication, IAM, DNS, and network problems
- Apply remediation
- Verify that the container starts successfully after the fix

---

# Architecture

The normal container image flow is:

```text
Kubernetes Pod
     │
     ▼
Container Runtime
     │
     ▼
Container Registry
     │
     ▼
Container Image
     │
     ▼
Container Starts
```

In this lab, the process fails during the image retrieval stage:

```text
Pod Created
    ↓
Container Runtime Requests Image
    ↓
Registry Cannot Provide Image
    ↓
Image Pull Fails
    ↓
ErrImagePull
```

If Kubernetes keeps retrying:

```text
ErrImagePull
     ↓
Retry
     ↓
Failure
     ↓
Backoff
     ↓
ImagePullBackOff
```

---

# Problem Scenario

A Kubernetes Pod is deployed with an invalid container image tag.

The Kubernetes API accepts the Pod definition, but the container cannot start because Kubernetes cannot retrieve the requested image.

Running:

```bash
kubectl get pods
```

may initially show:

```text
NAME               READY   STATUS         RESTARTS   AGE
errimagepull-pod   0/1     ErrImagePull   0          5s
```

Later, Kubernetes may change the status to:

```text
ImagePullBackOff
```

The Cloud Engineer must inspect the exact image pull error and identify the root cause.

---

# Expected Behavior

A healthy image deployment should follow:

```text
Pod Created
    ↓
Image Requested
    ↓
Image Found
    ↓
Image Downloaded
    ↓
Container Created
    ↓
Container Started
    ↓
Pod Running
```

Expected status:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

In the broken configuration, the image cannot be downloaded.

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
  name: errimagepull-pod

spec:
  containers:
    - name: nginx
      image: nginx:not-a-real-tag-12345
```

The problematic line is:

```yaml
image: nginx:not-a-real-tag-12345
```

The image tag does not exist.

---

# Reproduce the Problem

## Step 1 — Start Kubernetes

If using Minikube:

```bash
minikube start
```

Check status:

```bash
minikube status
```

Verify the node:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

The node should be:

```text
Ready
```

before starting the lab.

---

# Step 2 — Apply the Broken Configuration

Run:

```bash
kubectl apply -f broken.yaml
```

Expected:

```text
pod/errimagepull-pod created
```

Kubernetes has accepted the Pod definition.

This does not mean the image was successfully downloaded.

---

# Step 3 — Observe ErrImagePull

Immediately run:

```bash
kubectl get pods
```

You may see:

```text
NAME               READY   STATUS         RESTARTS   AGE
errimagepull-pod   0/1     ErrImagePull   0          5s
```

The `ErrImagePull` state may only appear briefly.

To observe state changes live:

```bash
kubectl get pods -w
```

You may see something similar to:

```text
ContainerCreating
ErrImagePull
ImagePullBackOff
```

Stop watching with:

```text
Ctrl + C
```

---

# What ErrImagePull Means

`ErrImagePull` means:

```text
Kubernetes attempted to retrieve the image
and the image pull operation failed.
```

The important point is that `ErrImagePull` is not the real root cause.

It is the high-level Kubernetes symptom.

The real cause may be:

```text
Wrong image name
Wrong tag
Repository does not exist
Authentication failure
IAM permission failure
Registry DNS failure
Network timeout
Registry unavailable
```

The exact reason must be discovered from Kubernetes Events.

---

# ErrImagePull vs ImagePullBackOff

These states are related but represent different stages.

## ErrImagePull

```text
Image pull attempt
      ↓
Immediate failure
      ↓
ErrImagePull
```

## ImagePullBackOff

```text
Repeated image pull failures
      ↓
Kubernetes delays retries
      ↓
ImagePullBackOff
```

A common progression is:

```text
ContainerCreating
      ↓
ErrImagePull
      ↓
ImagePullBackOff
```

---

# Initial Investigation

Run:

```bash
kubectl describe pod errimagepull-pod
```

Look at the container state.

You may see:

```text
State:
  Waiting

Reason:
  ErrImagePull
```

If Kubernetes has already retried several times, you may instead see:

```text
State:
  Waiting

Reason:
  ImagePullBackOff
```

Now scroll to:

```text
Events:
```

The Events section is the most important part of this lab.

---

# Read the Exact Image Pull Error

You may see something similar to:

```text
Failed to pull image "nginx:not-a-real-tag-12345"
```

You may also see:

```text
manifest unknown
```

or:

```text
not found
```

The exact wording matters.

For example:

```text
manifest unknown
```

usually points toward:

```text
Incorrect or missing image tag
```

while:

```text
unauthorized
```

points toward:

```text
Registry authentication or authorization
```

---

# Check Kubernetes Events Directly

Run:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for a progression such as:

```text
Pulling image "nginx:not-a-real-tag-12345"

Failed to pull image

Error: ErrImagePull

Back-off pulling image
```

This provides a timeline of what Kubernetes attempted.

---

# Why Logs Usually Do Not Help

Try:

```bash
kubectl logs errimagepull-pod
```

You will likely receive an error.

Why?

Because:

```text
Image never downloaded
      ↓
Container never successfully created
      ↓
Application never started
      ↓
Application logs do not exist
```

For `ErrImagePull`, the most useful troubleshooting tools are:

```text
kubectl describe pod
+
Kubernetes Events
```

rather than application logs.

---

# Root Cause

The root cause in this lab is:

```text
The Pod references a container image tag that does not exist.
```

Broken image:

```text
nginx:not-a-real-tag-12345
```

The registry cannot provide that image.

The result is:

```text
Image Pull Attempt
      ↓
Registry Error
      ↓
ErrImagePull
```

---

# Root Cause Diagram

```text
errimagepull-pod
      │
      ▼
Container Runtime
      │
      ▼
Container Registry
      │
      ▼
nginx:not-a-real-tag-12345
      │
      ▼
Tag Does Not Exist
      │
      ▼
Pull Failure
      │
      ▼
ErrImagePull
```

---

# Remediation

The remediation is to replace the invalid image tag with a valid image.

Broken:

```yaml
image: nginx:not-a-real-tag-12345
```

Fixed:

```yaml
image: nginx:1.27
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
  name: errimagepull-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

---

# Apply the Remediation

Delete the broken Pod:

```bash
kubectl delete pod errimagepull-pod
```

Expected:

```text
pod "errimagepull-pod" deleted
```

Apply the fixed configuration:

```bash
kubectl apply -f fixed.yaml
```

Expected:

```text
pod/errimagepull-pod created
```

---

# Verification

Run:

```bash
kubectl get pods
```

Expected:

```text
NAME               READY   STATUS    RESTARTS   AGE
errimagepull-pod   1/1     Running   0          10s
```

The important result is:

```text
STATUS = Running
```

---

# Verify With kubectl describe

Run:

```bash
kubectl describe pod errimagepull-pod
```

Look for:

```text
Image:
  nginx:1.27
```

and:

```text
State:
  Running
```

The Events section may show:

```text
Pulling image
Pulled image
Created container
Started container
```

This proves the image pull problem is resolved.

---

# Verify Logs

Now run:

```bash
kubectl logs errimagepull-pod
```

The command should work because the nginx container successfully started.

This confirms:

```text
Image Downloaded
      ↓
Container Created
      ↓
Container Started
      ↓
Application Running
```

---

# Verify Application Connectivity

Forward local port `8080` to nginx port `80`:

```bash
kubectl port-forward pod/errimagepull-pod 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

You should receive the nginx HTML response.

Stop port forwarding with:

```text
Ctrl + C
```

---

# Error Classification

When troubleshooting `ErrImagePull`, classify the exact event message.

| Error Message | Likely Cause |
|---|---|
| `manifest unknown` | Invalid or missing tag |
| `not found` | Wrong image or repository |
| `repository does not exist` | Incorrect repository |
| `pull access denied` | Registry authentication/authorization |
| `unauthorized` | Invalid credentials or permissions |
| `no such host` | DNS resolution failure |
| `i/o timeout` | Network connectivity problem |
| `connection refused` | Registry or network service issue |

The Event text is more useful than the high-level `ErrImagePull` status alone.

---

# Scenario 1 — Wrong Image Name

Broken:

```yaml
image: ngnix:1.27
```

Correct:

```yaml
image: nginx:1.27
```

A typo in the repository name can cause:

```text
Image Not Found
      ↓
ErrImagePull
```

---

# Scenario 2 — Wrong Image Tag

Suppose the registry contains:

```text
myapp:v1
myapp:v2
myapp:v3
```

but Kubernetes requests:

```yaml
image: myapp:v99
```

Result:

```text
ErrImagePull
```

The remediation is either:

```text
Use an existing tag
```

or:

```text
Build and push the intended image
```

---

# Scenario 3 — Private Registry Authentication Failure

A private registry may require credentials.

Architecture:

```text
Kubernetes
    ↓
Private Registry
    ↓
Authentication Required
```

If credentials are missing:

```text
Image Pull
   ↓
Authentication Failure
   ↓
ErrImagePull
```

A Kubernetes image pull Secret may be required.

Example:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.example.com \
  --docker-username=<username> \
  --docker-password=<password>
```

Reference it:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

Never commit real registry credentials to GitHub.

---

# Scenario 4 — DNS Failure

An Event may show:

```text
no such host
```

This indicates Kubernetes or the underlying node cannot resolve the registry hostname.

Possible causes include:

```text
DNS configuration failure
Incorrect registry hostname
Node DNS failure
Network misconfiguration
```

This is not an image-tag problem.

---

# Scenario 5 — Network Timeout

An Event may show:

```text
i/o timeout
```

or:

```text
connection timed out
```

This suggests:

```text
Network connectivity problem
```

Potential areas to check:

```text
Routes
NAT
Firewall
Proxy
VPC endpoints
Registry availability
```

---

# AWS EKS Scenario

This error is especially important in AWS EKS.

A typical image-pull architecture is:

```text
Amazon EKS
     ↓
Worker Node
     ↓
Container Runtime
     ↓
Amazon ECR
     ↓
Private Container Image
```

Possible causes include:

```text
Wrong ECR repository
Wrong image tag
Wrong AWS region
IAM permissions missing
Repository deleted
Image not pushed
Network access failure
```

---

# Verify ECR Repository

Example:

```bash
aws ecr describe-repositories \
  --repository-names my-app \
  --region us-east-1
```

If the repository does not exist, Kubernetes cannot retrieve the image.

---

# Verify Image Tags in ECR

Run:

```bash
aws ecr list-images \
  --repository-name my-app \
  --region us-east-1
```

Or:

```bash
aws ecr describe-images \
  --repository-name my-app \
  --region us-east-1
```

Verify that the tag used by Kubernetes actually exists.

---

# AWS Region Problems

An ECR image URL contains a region.

Example:

```text
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1
```

If your deployment points to the wrong region or wrong registry:

```text
Image Retrieval Fails
      ↓
ErrImagePull
```

Always verify:

```text
AWS Account
Region
Repository
Image Tag
```

---

# ECR IAM Problems

The identity responsible for retrieving an image must have appropriate ECR permissions.

Conceptually:

```text
Worker Node
    ↓
IAM Role
    ↓
ECR Permissions
    ↓
Retrieve Image
```

If permissions are missing:

```text
Authorization Fails
      ↓
Image Pull Fails
      ↓
ErrImagePull
```

The correct remediation is to grant the required least-privilege permissions.

Avoid solving the problem by attaching broad administrator permissions.

---

# ECR Networking Problems

Worker nodes in private subnets still require access to services needed for image retrieval.

Depending on the architecture, access may involve:

```text
Private Subnet
     ↓
NAT Gateway
     ↓
Required AWS Services
```

or appropriate VPC endpoints.

If the image exists and IAM is correct but the node cannot reach the registry:

```text
Image Exists
IAM Correct
      ↓
Network Path Fails
      ↓
ErrImagePull
```

This is why Cloud Engineers must troubleshoot beyond Kubernetes itself.

---

# CI/CD Scenario

A common CI/CD failure can produce `ErrImagePull`.

Example:

```text
GitHub Actions
     ↓
Build Image
     ↓
Tag Image
     ↓
Deployment Updated
     ↓
Image Push Fails
```

Kubernetes then requests an image that does not exist in the registry.

Result:

```text
ErrImagePull
```

The safer pipeline order is:

```text
Build
 ↓
Test
 ↓
Scan
 ↓
Push
 ↓
Verify Push
 ↓
Deploy
```

---

# Git SHA Image Tag Scenario

Many production pipelines use a Git commit SHA.

Example:

```text
myapp:abc1234
```

CI pushes:

```text
myapp:abc1234
```

but the Deployment accidentally references:

```text
myapp:def9876
```

Result:

```text
ErrImagePull
```

The CI/CD pipeline and Kubernetes deployment must use the same image identifier.

---

# ErrImagePull vs CrashLoopBackOff

Do not confuse these errors.

## ErrImagePull

```text
Image Pull Fails
      ↓
Container Never Starts
```

Application logs usually do not exist.

---

## CrashLoopBackOff

```text
Image Successfully Pulled
      ↓
Container Starts
      ↓
Application Crashes
      ↓
Container Restarts
```

Application logs usually exist.

---

# ErrImagePull vs ImagePullBackOff

Also remember:

```text
ErrImagePull
=
The current image pull attempt failed.
```

```text
ImagePullBackOff
=
Kubernetes is delaying another image pull attempt after repeated failures.
```

They often belong to the same incident.

---

# Important Commands

## Check Pod Status

```bash
kubectl get pods
```

---

## Watch Pod State Changes

```bash
kubectl get pods -w
```

This is useful for observing:

```text
ContainerCreating
      ↓
ErrImagePull
      ↓
ImagePullBackOff
```

---

## Describe Pod

```bash
kubectl describe pod errimagepull-pod
```

This is the most important troubleshooting command for this lab.

---

## Check Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Inspect Pod Configuration

```bash
kubectl get pod errimagepull-pod -o yaml
```

---

## Extract Image Name

```bash
kubectl get pod errimagepull-pod \
  -o jsonpath='{.spec.containers[*].image}'
```

---

# Troubleshooting Cheat Sheet

When you see:

```text
ErrImagePull
```

follow:

```text
1. kubectl get pods

2. kubectl describe pod <pod>

3. Read Events

4. Read the exact registry error

5. Check image repository

6. Check image name

7. Check tag

8. Confirm image exists

9. Check authentication

10. Check imagePullSecrets

11. Check IAM if using cloud registry

12. Check DNS

13. Check network connectivity

14. Fix root cause

15. Redeploy

16. Verify Running
```

---

# Troubleshooting Decision Tree

```text
                 ErrImagePull
                       │
                       ▼
             kubectl describe pod
                       │
                       ▼
              Read Exact Event
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
    Not Found        Unauthorized      Timeout
       │               │               │
       ▼               ▼               ▼
 Check Image       Check Credentials  Check Network
 Repo / Tag        Secret / IAM       DNS / Route
       │               │               │
       └───────────────┼───────────────┘
                       ▼
                   Remediate
                       │
                       ▼
                    Redeploy
                       │
                       ▼
                    Running
```

---

# Production Remediation Strategy

Do not repeatedly delete the Pod without fixing the cause.

If the Deployment still contains an invalid image:

```text
Delete Pod
   ↓
New Pod Created
   ↓
Same Invalid Image
   ↓
ErrImagePull Again
```

The correct approach is:

```text
Observe
   ↓
Read Events
   ↓
Classify Error
   ↓
Identify Root Cause
   ↓
Fix Image / Auth / IAM / DNS / Network
   ↓
Redeploy
   ↓
Verify
```

---

# Cloud Engineering Relevance

`ErrImagePull` is a good example of a Kubernetes error that may actually originate from a different system.

Possible layers include:

```text
Kubernetes
   │
   ├── Pod Configuration
   └── Deployment Configuration

Container Registry
   │
   ├── Repository
   └── Image Tag

CI/CD
   │
   ├── Build
   ├── Tag
   └── Push

Cloud
   │
   ├── IAM
   ├── Networking
   ├── DNS
   └── Registry Service
```

A Cloud Engineer must identify which layer failed.

---

# Cloud Security Relevance

Image pull failures may also be security related.

Examples:

```text
Registry access denied
IAM policy changed
Credential expired
Secret removed
Repository policy changed
Private registry restrictions
```

Security remediation should preserve:

```text
Least Privilege
```

Do not solve registry access failures by making the repository public unless public access is intentionally required.

---

# Secure Container Image Practices

Production environments should consider:

- Private trusted registries
- Vulnerability scanning
- Immutable image tags
- Image digest pinning where appropriate
- Least-privilege registry access
- CI/CD security scanning
- Image provenance
- Signed images where appropriate
- Removal of vulnerable images
- Registry logging and monitoring

A secure delivery flow can look like:

```text
Source Code
    ↓
Build Container
    ↓
Security Scan
    ↓
Push Trusted Image
    ↓
Deploy Exact Image
    ↓
Kubernetes
```

---

# Prevention

## Validate Images Before Deployment

Confirm:

```text
Repository exists
Image exists
Tag exists
Image is accessible
```

before updating the Kubernetes workload.

---

## Verify CI/CD Push Success

Do not deploy until image push succeeds.

Correct:

```text
Build
 ↓
Push
 ↓
Verify
 ↓
Deploy
```

---

## Use Consistent Tags

The same tag should flow through:

```text
Docker Build
   ↓
Registry Push
   ↓
Kubernetes Deployment
```

---

## Use Traceable Image Versions

Prefer:

```text
myapp:v1.4.3
```

or:

```text
myapp:abc1234
```

instead of relying only on:

```text
myapp:latest
```

Traceable versions make troubleshooting and rollback easier.

---

## Monitor Kubernetes Events

Monitoring should detect repeated:

```text
ErrImagePull
```

and:

```text
ImagePullBackOff
```

events.

---

# Cleanup

Delete the Pod:

```bash
kubectl delete pod errimagepull-pod
```

Verify:

```bash
kubectl get pods
```

The lab environment should now be clean.

---

# What I Learned

In this lab, I learned that `ErrImagePull` represents the immediate failure of a container image pull attempt.

I learned that the Pod status alone is not enough to determine the real problem.

The most important troubleshooting evidence comes from:

```text
kubectl describe pod
+
Kubernetes Events
```

I also learned that the same high-level `ErrImagePull` error can represent very different underlying causes, including:

```text
Wrong tag
Wrong repository
Authentication failure
IAM failure
DNS failure
Network failure
```

The exact Event message determines the next troubleshooting step.

---

# Key Takeaways

- `ErrImagePull` means the current image pull attempt failed.
- It often appears before `ImagePullBackOff`.
- The container usually never started.
- Application logs may not exist.
- Use `kubectl describe pod`.
- Read the Events section carefully.
- Do not assume every pull failure means the tag is wrong.
- `manifest unknown` usually suggests an image/tag problem.
- `unauthorized` suggests authentication or authorization.
- `no such host` suggests DNS.
- `i/o timeout` suggests networking.
- On EKS, check ECR, IAM, AWS region, and network access.
- Fix the root cause instead of repeatedly deleting Pods.

---

# Final Troubleshooting Flow

```text
                    ErrImagePull
                         │
                         ▼
                  kubectl get pods
                         │
                         ▼
                kubectl describe pod
                         │
                         ▼
                  Read Exact Event
                         │
           ┌─────────────┼─────────────┐
           │             │             │
           ▼             ▼             ▼
      Image Problem    Auth Problem   Network Problem
           │             │             │
           ▼             ▼             ▼
      Repo / Tag      Secret / IAM   DNS / Route
           │             │             │
           └─────────────┼─────────────┘
                         ▼
                  Fix Root Cause
                         │
                         ▼
                     Redeploy
                         │
                         ▼
                  Image Pulled
                         │
                         ▼
                 Container Starts
                         │
                         ▼
                      Running
```