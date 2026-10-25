# Lab 03 — ImagePullBackOff

## Overview

This lab demonstrates the Kubernetes `ImagePullBackOff` error.

`ImagePullBackOff` happens when Kubernetes cannot successfully download the container image required by a Pod.

In this lab, the Pod intentionally references an image tag that does not exist.

The purpose of this lab is to learn how to:

- Recognize `ErrImagePull`
- Recognize `ImagePullBackOff`
- Understand the difference between image pull failures and application crashes
- Use `kubectl describe pod`
- Inspect Kubernetes Events
- Identify invalid image names or tags
- Understand private registry and Amazon ECR problems
- Apply remediation
- Verify that the corrected container image starts successfully

---

# Architecture

The basic flow is:

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
Requested Image
```

In this lab:

```text
Pod
 ↓
Request:
nginx:this-tag-does-not-exist
 ↓
Container Registry
 ↓
Image Tag Not Found
 ↓
Pull Failure
 ↓
ErrImagePull
 ↓
ImagePullBackOff
```

---

# Problem Scenario

A Kubernetes Pod is deployed using an incorrect container image tag.

The Kubernetes API accepts the Pod configuration, but the container cannot start because Kubernetes cannot retrieve the requested image.

Running:

```bash
kubectl get pods
```

shows:

```text
NAME                   READY   STATUS             RESTARTS   AGE
imagepullbackoff-pod   0/1     ImagePullBackOff   0          30s
```

The Cloud Engineer must determine why the image cannot be pulled and apply the correct remediation.

---

# Expected Behavior

Normally:

```text
Pod Created
    ↓
Container Runtime Requests Image
    ↓
Registry Returns Image
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

In the broken configuration, the image retrieval process fails before the container can start.

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
  name: imagepullbackoff-pod

spec:
  containers:
    - name: nginx
      image: nginx:this-tag-does-not-exist
```

The problem is:

```yaml
image: nginx:this-tag-does-not-exist
```

The requested image tag does not exist in the registry.

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

The important value is:

```text
STATUS = Ready
```

---

# Step 2 — Apply the Broken Configuration

Run:

```bash
kubectl apply -f broken.yaml
```

Expected:

```text
pod/imagepullbackoff-pod created
```

This means Kubernetes accepted the manifest.

It does not mean the container successfully started.

---

# Step 3 — Observe the Failure

Run:

```bash
kubectl get pods
```

You may initially see:

```text
ContainerCreating
```

followed by:

```text
ErrImagePull
```

After Kubernetes retries, you should see:

```text
ImagePullBackOff
```

Example:

```text
NAME                   READY   STATUS             RESTARTS   AGE
imagepullbackoff-pod   0/1     ImagePullBackOff   0          40s
```

---

# Understanding ErrImagePull

`ErrImagePull` means Kubernetes attempted to retrieve the image and the pull operation failed.

The flow is:

```text
Pod Created
    ↓
Container Runtime Requests Image
    ↓
Image Pull Fails
    ↓
ErrImagePull
```

This is normally the first visible error.

---

# Understanding ImagePullBackOff

After repeated image pull failures, Kubernetes starts delaying additional attempts.

The flow becomes:

```text
Image Pull Attempt
      ↓
Failure
      ↓
Retry
      ↓
Failure
      ↓
Backoff Delay
      ↓
ImagePullBackOff
```

The `BackOff` part means Kubernetes is waiting before attempting another image pull.

---

# Initial Investigation

Run:

```bash
kubectl describe pod imagepullbackoff-pod
```

Look at the container State.

You may see:

```text
State:
  Waiting

Reason:
  ImagePullBackOff
```

Then scroll to:

```text
Events:
```

This section normally contains the most useful information.

---

# Evidence

The Events section may contain messages similar to:

```text
Failed to pull image "nginx:this-tag-does-not-exist"
```

You may also see messages containing:

```text
manifest unknown
```

or:

```text
not found
```

and later:

```text
Back-off pulling image
```

This provides strong evidence that the failure happened during image retrieval.

---

# Check Kubernetes Events Directly

Run:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for events such as:

```text
Pulling
Failed
BackOff
```

These Events help identify the exact stage where the Pod failed.

---

# Check the Configured Image

Run:

```bash
kubectl get pod imagepullbackoff-pod -o yaml
```

Look for:

```yaml
image:
```

The configured image should show:

```text
nginx:this-tag-does-not-exist
```

Now compare that value with the image that should actually exist.

---

# Why kubectl logs Is Less Useful Here

Try:

```bash
kubectl logs imagepullbackoff-pod
```

The command may return an error because the container never started.

This is an important troubleshooting distinction.

```text
Image could not be downloaded
        ↓
Container was never created successfully
        ↓
Application never started
        ↓
Application logs do not exist
```

For `ImagePullBackOff`, the most useful tools are usually:

```text
kubectl describe pod
+
Kubernetes Events
```

rather than application logs.

---

# Root Cause

The root cause is:

```text
The Kubernetes Pod references a container image tag that does not exist.
```

Broken image:

```text
nginx:this-tag-does-not-exist
```

Because the registry cannot provide the requested image:

```text
Image Pull Fails
      ↓
ErrImagePull
      ↓
Repeated Failure
      ↓
ImagePullBackOff
```

---

# Root Cause Diagram

```text
imagepullbackoff-pod
        │
        ▼
Container Runtime
        │
        ▼
Container Registry
        │
        ▼
nginx:this-tag-does-not-exist
        │
        ▼
Tag Not Found
        │
        ▼
Image Pull Failure
        │
        ▼
ErrImagePull
        │
        ▼
ImagePullBackOff
```

---

# Remediation

The remediation in this lab is to replace the invalid image tag with a valid image.

Broken:

```yaml
image: nginx:this-tag-does-not-exist
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
  name: imagepullbackoff-pod

spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

---

# Apply the Remediation

Delete the broken Pod:

```bash
kubectl delete pod imagepullbackoff-pod
```

Expected:

```text
pod "imagepullbackoff-pod" deleted
```

Apply the corrected configuration:

```bash
kubectl apply -f fixed.yaml
```

Expected:

```text
pod/imagepullbackoff-pod created
```

---

# Verification

Run:

```bash
kubectl get pods
```

Expected:

```text
NAME                   READY   STATUS    RESTARTS   AGE
imagepullbackoff-pod   1/1     Running   0          10s
```

The important values are:

```text
READY = 1/1
STATUS = Running
RESTARTS = 0
```

---

# Verify With kubectl describe

Run:

```bash
kubectl describe pod imagepullbackoff-pod
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

This proves the image retrieval problem has been resolved.

---

# Verify With Logs

Now:

```bash
kubectl logs imagepullbackoff-pod
```

should work because the container actually started.

This confirms:

```text
Image Pulled
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
kubectl port-forward pod/imagepullbackoff-pod 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

The nginx HTML response should be returned.

This verifies:

```text
Correct Image
     ↓
Successful Pull
     ↓
Container Running
     ↓
Nginx Running
     ↓
Application Reachable
```

Stop port forwarding with:

```text
Ctrl + C
```

---

# ImagePullBackOff vs CrashLoopBackOff

This distinction is extremely important.

## ImagePullBackOff

```text
Image Cannot Be Downloaded
       ↓
Container Never Starts
       ↓
Application Never Runs
```

Useful troubleshooting:

```text
kubectl describe pod
Events
Registry checks
Authentication checks
IAM checks
Network checks
```

---

## CrashLoopBackOff

```text
Image Successfully Downloaded
       ↓
Container Starts
       ↓
Application Fails
       ↓
Container Restarts
```

Useful troubleshooting:

```text
kubectl logs
kubectl logs --previous
kubectl describe pod
Exit codes
Application configuration
```

---

# Common Causes of ImagePullBackOff

`ImagePullBackOff` can have several root causes.

Common examples include:

- Incorrect image name
- Incorrect image tag
- Image does not exist
- Image was deleted
- Private registry authentication failure
- Missing `imagePullSecret`
- Invalid registry hostname
- Registry unavailable
- Registry network failure
- AWS ECR permissions missing
- Wrong AWS region
- ECR repository does not exist
- Node cannot reach ECR
- CI/CD pipeline referenced an image before it was pushed

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

The typo causes Kubernetes to request the wrong repository.

Result:

```text
ErrImagePull
 ↓
ImagePullBackOff
```

---

# Scenario 2 — Wrong Image Tag

Suppose a registry contains:

```text
myapp:v1
myapp:v2
myapp:v3
```

But Kubernetes requests:

```yaml
image: myapp:v5
```

The tag does not exist.

Result:

```text
ImagePullBackOff
```

Remediation:

```yaml
image: myapp:v3
```

or build and push the required `v5` image if that is the intended release.

---

# Scenario 3 — Private Registry Authentication Failure

Architecture:

```text
Kubernetes
    ↓
Private Registry
    ↓
Authentication Required
```

Without valid credentials:

```text
Image Pull
   ↓
Authentication Failure
   ↓
ImagePullBackOff
```

A Kubernetes Docker registry Secret may be required.

Example:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.example.com \
  --docker-username=<username> \
  --docker-password=<password>
```

Then reference it:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

Real credentials should never be committed to GitHub.

---

# Scenario 4 — Amazon ECR ImagePullBackOff

This is especially important in AWS EKS.

Architecture:

```text
Amazon EKS
    ↓
Worker Node
    ↓
Container Runtime
    ↓
Amazon ECR
    ↓
Container Image
```

Possible failures include:

```text
Wrong ECR Repository
Wrong Tag
Wrong AWS Region
Image Not Pushed
Repository Deleted
IAM Permission Failure
Network Connectivity Failure
```

---

# Verify an ECR Repository

Example:

```bash
aws ecr describe-repositories \
  --repository-names my-app \
  --region us-east-1
```

---

# Verify Images in ECR

Run:

```bash
aws ecr describe-images \
  --repository-name my-app \
  --region us-east-1
```

or:

```bash
aws ecr list-images \
  --repository-name my-app \
  --region us-east-1
```

Verify that the tag referenced by Kubernetes actually exists.

---

# ECR Troubleshooting Flow

```text
ImagePullBackOff
       ↓
kubectl describe pod
       ↓
Inspect ECR URL
       ↓
Check Repository Exists
       ↓
Check Tag Exists
       ↓
Check AWS Region
       ↓
Check IAM
       ↓
Check Network
       ↓
Remediate
```

---

# ECR IAM Problems

Private ECR image retrieval requires the correct AWS permissions on the identity responsible for pulling images.

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

If required permissions are unavailable:

```text
ECR Authorization Fails
       ↓
Image Cannot Be Downloaded
       ↓
ImagePullBackOff
```

The correct remediation is to fix the relevant ECR permissions with least privilege.

Do not use broad administrator permissions as a shortcut.

---

# ECR Network Problems

Nodes in private subnets must have a valid network path to the services required for pulling images.

Depending on the architecture, access may involve:

```text
Private Subnet
      ↓
NAT Gateway
      ↓
AWS Services
```

or appropriate VPC endpoints.

If network access is broken:

```text
Image Exists
      ↓
IAM Is Correct
      ↓
Node Cannot Reach Registry
      ↓
Pull Times Out
      ↓
ImagePullBackOff
```

This demonstrates an important lesson:

```text
ImagePullBackOff does not always mean the image is wrong.
```

It can also mean the registry cannot be reached.

---

# Scenario 5 — CI/CD Image Does Not Exist Yet

Example workflow:

```text
GitHub Actions
     ↓
Build Docker Image
     ↓
Tag Image
     ↓
Push to ECR
     ↓
Deploy to EKS
```

Correct order matters.

Bad pipeline:

```text
Deploy myapp:abc123
      ↓
Kubernetes requests image
      ↓
Image has not been pushed
      ↓
ImagePullBackOff
```

Correct:

```text
Build
 ↓
Tag
 ↓
Push
 ↓
Confirm Push
 ↓
Deploy
```

---

# Scenario 6 — Wrong Git Commit Tag

Many CI/CD pipelines use Git commit SHA as the image tag.

Example:

```text
myapp:a12bc34
```

Suppose CI pushes:

```text
myapp:a12bc34
```

but Deployment references:

```text
myapp:d98ef76
```

The Deployment requests an image that does not exist.

Result:

```text
ImagePullBackOff
```

The correct fix is to synchronize the image tag used by CI/CD and the Kubernetes manifest.

---

# Why Immutable Tags Are Helpful

Avoid relying only on:

```yaml
image: myapp:latest
```

because `latest` does not clearly identify which application version is deployed.

Prefer:

```yaml
image: myapp:v1.4.3
```

or:

```yaml
image: myapp:a12bc34
```

This improves:

- Traceability
- Rollbacks
- Incident investigation
- Release management
- Deployment consistency

---

# Important Commands

## Check Pod Status

```bash
kubectl get pods
```

---

## Inspect Pod

```bash
kubectl describe pod imagepullbackoff-pod
```

This is the most important command for this lab.

---

## Inspect Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Inspect Pod YAML

```bash
kubectl get pod imagepullbackoff-pod -o yaml
```

---

## Check Container Image

You can also retrieve the configured image with:

```bash
kubectl get pod imagepullbackoff-pod \
  -o jsonpath='{.spec.containers[*].image}'
```

---

# Troubleshooting Cheat Sheet

When you see:

```text
ImagePullBackOff
```

follow:

```text
1. kubectl get pods

2. kubectl describe pod <pod>

3. Read Events

4. Check image repository

5. Check image name

6. Check image tag

7. Confirm image exists

8. Check private registry authentication

9. Check imagePullSecrets if used

10. Check cloud IAM

11. Check network connectivity

12. Correct the root cause

13. Redeploy

14. Verify Running
```

---

# Troubleshooting Decision Tree

```text
             ImagePullBackOff
                    │
                    ▼
           kubectl describe pod
                    │
                    ▼
             Read Error Message
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
    Not Found     Denied      Timeout
        │           │           │
        ▼           ▼           ▼
 Check Image    Check Auth    Check Network
 Name / Tag     IAM / Secret  DNS / Routes
        │           │           │
        └───────────┼───────────┘
                    ▼
               Remediation
                    │
                    ▼
                 Redeploy
                    │
                    ▼
                  Running
```

---

# Production Remediation Strategy

Do not repeatedly delete the Pod.

If the controlling workload still references the wrong image:

```text
Delete Pod
   ↓
Controller Creates New Pod
   ↓
Same Wrong Image
   ↓
ImagePullBackOff Again
```

Instead:

```text
Observe Error
     ↓
Inspect Events
     ↓
Determine Registry Failure
     ↓
Fix Image / Authentication / IAM / Network
     ↓
Update Workload
     ↓
Redeploy
     ↓
Verify
```

---

# Cloud Engineering Relevance

Container image delivery is a major part of Kubernetes operations.

A Cloud Engineer may need to troubleshoot several different infrastructure layers:

```text
Kubernetes
   │
   ├── Pod
   ├── Deployment
   └── Image Configuration

Container Registry
   │
   ├── Repository
   ├── Image
   └── Tag

Cloud Platform
   │
   ├── IAM
   ├── Networking
   ├── ECR
   └── VPC

CI/CD
   │
   ├── Build
   ├── Tag
   ├── Push
   └── Deploy
```

A failure at any of these layers may appear as `ImagePullBackOff`.

---

# Cloud Security Relevance

Image pulling is also a security-sensitive operation.

Security problems may include:

- Unauthorized registry access
- Expired credentials
- Incorrect IAM policy
- Deleted registry Secret
- Compromised or untrusted images
- Public registry exposure
- Overly broad ECR permissions
- Supply-chain security failures

The remediation should preserve least privilege.

Avoid fixing a private-registry problem by making the repository public unless public access is deliberately required.

---

# Secure Image Practices

Production environments should consider:

- Private trusted registries
- Vulnerability scanning
- Immutable image tags
- Image digests
- Least-privilege registry access
- CI/CD image scanning
- Restricted deployment permissions
- Signed images where appropriate
- Removal of vulnerable images
- Registry auditing

A common secure pipeline is:

```text
Source Code
    ↓
Build Image
    ↓
Security Scan
    ↓
Push Trusted Image
    ↓
Deploy Exact Version
    ↓
Kubernetes
```

---

# Prevention

## Validate Image References Before Deployment

Confirm that:

```text
Repository Exists
Tag Exists
Image Is Accessible
```

before updating Kubernetes.

---

## Use CI/CD Validation

The pipeline should verify a successful image push before deployment.

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

## Use Consistent Image Tags

Use the same tag between:

```text
Docker Build
ECR Push
Kubernetes Deployment
```

---

## Monitor Kubernetes Events

Alert on repeated:

```text
Failed
```

and:

```text
BackOff
```

image events.

---

## Protect Registry Credentials

Do not store registry passwords directly inside:

```text
GitHub repository
Dockerfile
Plain YAML
Application code
```

Use secure secret-management mechanisms.

---

# Cleanup

Delete the lab Pod:

```bash
kubectl delete pod imagepullbackoff-pod
```

Verify:

```bash
kubectl get pods
```

The lab environment should now be clean.

---

# What I Learned

In this lab, I learned that `ImagePullBackOff` happens before the container application starts.

I learned that Kubernetes first attempts to retrieve a container image and may initially report:

```text
ErrImagePull
```

After repeated failures, the state can become:

```text
ImagePullBackOff
```

I also learned that the most useful troubleshooting information normally comes from:

```bash
kubectl describe pod <pod>
```

especially the Events section.

Unlike `CrashLoopBackOff`, application logs may not exist because the container never successfully started.

---

# Key Takeaways

- `ErrImagePull` means an image pull attempt failed.
- `ImagePullBackOff` means Kubernetes is backing off before retrying image retrieval.
- The container usually never started.
- Application logs may therefore be unavailable.
- Use `kubectl describe pod`.
- Always inspect Events.
- Verify image name and tag.
- Verify the image exists.
- Check private registry authentication.
- Check `imagePullSecrets` where applicable.
- On EKS, check ECR, IAM, AWS region, and networking.
- Do not repeatedly delete the Pod without fixing the underlying cause.
- Use immutable and traceable image versions in production.

---

# Final Troubleshooting Flow

```text
                  ImagePullBackOff
                         │
                         ▼
                  kubectl get pods
                         │
                         ▼
                kubectl describe pod
                         │
                         ▼
                     Events
                         │
                         ▼
                Image Pull Failure
                         │
            ┌────────────┼────────────┐
            │            │            │
            ▼            ▼            ▼
       Image/Tag      Authentication  Network
         Wrong          Failure       Failure
            │            │            │
            ▼            ▼            ▼
       Fix Image      Fix Secret     Fix Route /
       Reference      IAM Access     Connectivity
            │            │            │
            └────────────┼────────────┘
                         ▼
                     Redeploy
                         │
                         ▼
                  Image Pulled
                         │
                         ▼
                Container Started
                         │
                         ▼
                     Running
```