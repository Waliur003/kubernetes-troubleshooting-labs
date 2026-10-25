# Lab 02 — CrashLoopBackOff

## Overview

This lab demonstrates one of the most common Kubernetes container failure states: `CrashLoopBackOff`.

A Pod enters `CrashLoopBackOff` when its container starts, crashes, Kubernetes restarts it, and the same failure happens repeatedly.

The purpose of this lab is to learn how to:

- Recognize `CrashLoopBackOff`
- Understand why Kubernetes repeatedly restarts a failed container
- Use `kubectl describe`
- Inspect container exit codes
- Read current logs
- Read previous container logs
- Identify the real root cause
- Apply remediation
- Verify that the container remains healthy after the fix

---

# Architecture

This lab uses a single Kubernetes Pod running a BusyBox container.

```text
Kubernetes Cluster
      │
      ▼
     Pod
      │
      ▼
BusyBox Container
      │
      ▼
Broken Command
      │
      ▼
    exit 1
      │
      ▼
Container Terminates
      │
      ▼
Kubernetes Restarts Container
      │
      ▼
CrashLoopBackOff
```

---

# Problem Scenario

A new container is deployed into Kubernetes.

The container starts successfully, prints a few messages, and then exits with an error code.

Kubernetes attempts to restart the container.

The container fails again.

After repeated failures, Kubernetes begins delaying restart attempts.

Running:

```bash
kubectl get pods
```

eventually shows:

```text
NAME            READY   STATUS             RESTARTS   AGE
crashloop-pod   0/1     CrashLoopBackOff   4          1m
```

The Cloud Engineer must identify why the container continuously crashes.

---

# Expected Behavior

The expected behavior is:

```text
Pod Created
   ↓
Container Starts
   ↓
Application Process Runs
   ↓
Container Remains Running
   ↓
Pod Becomes Ready
```

Expected Pod status:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

In the broken configuration, the application process exits immediately.

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
  name: crashloop-pod

spec:
  containers:
    - name: crashloop-container
      image: busybox:1.36

      command:
        - /bin/sh
        - -c
        - |
          echo "Starting application..."
          echo "Something went wrong."
          exit 1
```

The important line is:

```bash
exit 1
```

---

# Understanding Exit Codes

Linux processes use exit codes to indicate whether a command completed successfully.

```text
exit 0
=
Success
```

```text
exit 1
=
Failure
```

The broken container executes:

```text
Start shell
   ↓
Print messages
   ↓
exit 1
   ↓
Process terminates
   ↓
Container terminates
```

Because the container terminates with an error, Kubernetes attempts to restart it.

---

# Reproduce the Problem

## Step 1 — Start the Kubernetes Cluster

If using Minikube:

```bash
minikube start
```

Check status:

```bash
minikube status
```

Verify nodes:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

The node should show:

```text
Ready
```

---

# Step 2 — Deploy the Broken Pod

Run:

```bash
kubectl apply -f broken.yaml
```

Expected:

```text
pod/crashloop-pod created
```

Immediately inspect the Pod:

```bash
kubectl get pods
```

At first, the Pod may briefly show:

```text
Error
```

or:

```text
Running
```

because the container is cycling quickly.

Run the command again:

```bash
kubectl get pods
```

After repeated failures, the state should become:

```text
CrashLoopBackOff
```

---

# What CrashLoopBackOff Means

`CrashLoopBackOff` does not mean Kubernetes itself has crashed.

It means:

```text
Container Starts
      ↓
Application Fails
      ↓
Container Terminates
      ↓
Kubernetes Restarts It
      ↓
Container Fails Again
      ↓
Kubernetes Waits
      ↓
Restart Attempt
```

The repeated delay is the `BackOff` behavior.

Kubernetes does not restart the container continuously at full speed.

Instead, it gradually increases the delay between restart attempts.

---

# Initial Investigation

Start with:

```bash
kubectl get pod crashloop-pod
```

Then:

```bash
kubectl get pod crashloop-pod -o wide
```

The important fields are:

```text
STATUS
RESTARTS
```

Example:

```text
NAME            READY   STATUS             RESTARTS
crashloop-pod   0/1     CrashLoopBackOff   5
```

An increasing restart count is an important sign that the container repeatedly fails.

---

# Inspect the Pod

Run:

```bash
kubectl describe pod crashloop-pod
```

Look for:

```text
Containers:
```

Then inspect:

```text
State:
```

and:

```text
Last State:
```

You may see:

```text
State:
  Waiting
  Reason: CrashLoopBackOff
```

and:

```text
Last State:
  Terminated
  Reason: Error
  Exit Code: 1
```

This gives strong evidence that the application process exited with an error.

---

# Important Evidence

The most useful evidence is:

```text
Last State = Terminated
```

```text
Reason = Error
```

```text
Exit Code = 1
```

This tells us that:

```text
The container did successfully start.
```

but:

```text
The process inside the container terminated with an error.
```

This distinction is important.

---

# Check Kubernetes Events

At the bottom of:

```bash
kubectl describe pod crashloop-pod
```

look for the `Events` section.

You may see something similar to:

```text
Back-off restarting failed container
```

You can also run:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Look for events related to:

```text
crashloop-pod
```

---

# Check Current Logs

Run:

```bash
kubectl logs crashloop-pod
```

Expected output:

```text
Starting application...
Something went wrong.
```

This tells us what the process did immediately before terminating.

---

# Check Previous Container Logs

One of the most important commands for troubleshooting `CrashLoopBackOff` is:

```bash
kubectl logs crashloop-pod --previous
```

Why is this useful?

Because Kubernetes may already have restarted the container.

The current container instance may not contain the logs from the previous failed instance.

Conceptually:

```text
Container Instance #1
      ↓
     Crash
      ↓
Container Instance #2
```

Using:

```bash
kubectl logs --previous
```

allows us to inspect logs from:

```text
Container Instance #1
```

This is extremely valuable when a container crashes very quickly.

---

# Troubleshooting Workflow

```text
CrashLoopBackOff
       ↓
kubectl describe pod
       ↓
Inspect State
       ↓
Inspect Last State
       ↓
Check Exit Code
       ↓
kubectl logs
       ↓
kubectl logs --previous
       ↓
Check Events
       ↓
Identify Root Cause
```

---

# Root Cause

The root cause is:

```text
The container command intentionally exits with exit code 1.
```

The broken command is:

```bash
exit 1
```

This tells the operating system that the process failed.

Once the main container process exits, the container stops.

Kubernetes then attempts to restart it.

The same failure occurs repeatedly.

Result:

```text
CrashLoopBackOff
```

---

# Root Cause Diagram

```text
crashloop-pod
      │
      ▼
BusyBox Container
      │
      ▼
Shell Starts
      │
      ▼
Commands Execute
      │
      ▼
exit 1
      │
      ▼
Container Terminates
      │
      ▼
Kubernetes Restarts Container
      │
      ▼
Same Error Happens Again
      │
      ▼
CrashLoopBackOff
```

---

# Restart Policy

Regular Kubernetes Pods normally use:

```yaml
restartPolicy: Always
```

for application workloads unless otherwise configured.

You can inspect the full Pod configuration:

```bash
kubectl get pod crashloop-pod -o yaml
```

The combination is:

```text
Container Terminates
+
restartPolicy Always
=
Container Restart
```

If the same error occurs repeatedly:

```text
CrashLoopBackOff
```

---

# Remediation

The remediation is to remove the failing process behavior and keep the application process running.

Broken behavior:

```text
Start
 ↓
exit 1
 ↓
Crash
```

Fixed behavior:

```text
Start
 ↓
Application remains active
 ↓
Container remains running
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
  name: crashloop-pod

spec:
  containers:
    - name: crashloop-container
      image: busybox:1.36

      command:
        - /bin/sh
        - -c
        - |
          echo "Application started successfully."
          while true
          do
            echo "Application is running."
            sleep 30
          done
```

---

# Understanding the Fixed Configuration

The fixed container executes:

```text
Start shell
     ↓
Print startup message
     ↓
Enter loop
     ↓
Print application status
     ↓
Sleep 30 seconds
     ↓
Repeat
```

Because the main process remains alive:

```text
Container stays running
```

Kubernetes no longer needs to restart it.

---

# Apply the Remediation

Delete the broken Pod:

```bash
kubectl delete pod crashloop-pod
```

Expected:

```text
pod "crashloop-pod" deleted
```

Apply the fixed version:

```bash
kubectl apply -f fixed.yaml
```

Expected:

```text
pod/crashloop-pod created
```

---

# Verification

Check the Pod:

```bash
kubectl get pods
```

Expected:

```text
NAME            READY   STATUS    RESTARTS   AGE
crashloop-pod   1/1     Running   0          10s
```

The important values are:

```text
READY = 1/1
```

```text
STATUS = Running
```

```text
RESTARTS = 0
```

---

# Verify Restart Count

Run:

```bash
kubectl get pod crashloop-pod
```

Wait 30–60 seconds.

Run again:

```bash
kubectl get pod crashloop-pod
```

The restart count should remain:

```text
0
```

This proves that the application process is no longer crashing.

---

# Verify Container Logs

Run:

```bash
kubectl logs crashloop-pod
```

Expected:

```text
Application started successfully.
Application is running.
Application is running.
```

This confirms the process remains alive.

---

# Follow Logs in Real Time

Run:

```bash
kubectl logs -f crashloop-pod
```

You should see:

```text
Application is running.
```

approximately every 30 seconds.

Stop log following with:

```text
Ctrl + C
```

---

# Verify With Describe

Run:

```bash
kubectl describe pod crashloop-pod
```

The container should now show:

```text
State:
  Running
```

The Pod should show:

```text
Ready:
  True
```

and the restart count should remain low or zero for the newly created Pod.

---

# Broken vs Fixed

## Broken

```yaml
command:
  - /bin/sh
  - -c
  - |
    echo "Starting application..."
    echo "Something went wrong."
    exit 1
```

Behavior:

```text
Application Starts
      ↓
Application Exits
      ↓
Container Stops
      ↓
Kubernetes Restarts
      ↓
Repeated Failure
      ↓
CrashLoopBackOff
```

---

## Fixed

```yaml
command:
  - /bin/sh
  - -c
  - |
    echo "Application started successfully."
    while true
    do
      echo "Application is running."
      sleep 30
    done
```

Behavior:

```text
Application Starts
      ↓
Process Remains Running
      ↓
Container Remains Running
      ↓
Pod Ready
```

---

# Real Production Causes of CrashLoopBackOff

In production, `CrashLoopBackOff` can be caused by many different problems.

Common causes include:

- Application bugs
- Incorrect startup commands
- Missing environment variables
- Missing ConfigMaps
- Missing Secrets
- Incorrect credentials
- Database connectivity failures
- Dependency failures
- Permission problems
- Incorrect file paths
- Port binding failures
- Application configuration errors
- Memory failures
- Incorrect health probes
- Startup timing problems

`CrashLoopBackOff` is a symptom.

The actual root cause must be discovered from logs, events, exit codes, and application configuration.

---

# Example — Missing Environment Variable

Imagine an application requires:

```text
DATABASE_URL
```

During startup:

```text
Application Starts
      ↓
Checks DATABASE_URL
      ↓
Variable Missing
      ↓
Application Exits
      ↓
Container Stops
      ↓
Restart
      ↓
CrashLoopBackOff
```

Application logs may show:

```text
DATABASE_URL environment variable is required
```

A possible remediation might be:

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: database-secret
        key: url
```

---

# Example — Database Connectivity Failure

Architecture:

```text
Backend Pod
    │
    ▼
Database Service
    │
    ▼
PostgreSQL
```

Suppose the backend requires database connectivity before startup can complete.

If the database is unavailable:

```text
Backend Starts
      ↓
Connect to Database
      ↓
Connection Refused
      ↓
Application Exits
      ↓
Kubernetes Restarts Container
      ↓
CrashLoopBackOff
```

Logs may show:

```text
connection refused
```

or:

```text
connection timeout
```

The Kubernetes Pod is not necessarily the root problem.

The database or network path may be the actual root cause.

---

# Example — Incorrect Container Command

Suppose the image contains:

```text
/app/main.py
```

but Kubernetes starts:

```text
python /app/missing.py
```

The application may return:

```text
python: can't open file '/app/missing.py'
```

The process exits.

Result:

```text
CrashLoopBackOff
```

The remediation is to correct the container command.

---

# Example — ConfigMap Failure

An application may require configuration such as:

```text
APP_ENV
API_HOST
LOG_LEVEL
```

If the configuration is missing or invalid, the application may fail immediately.

Useful commands:

```bash
kubectl get configmap
```

```bash
kubectl describe configmap <configmap-name>
```

Then inspect whether the Pod correctly references the ConfigMap.

---

# Example — Secret Failure

Applications frequently require:

```text
Database password
API token
Cloud credentials
TLS information
```

Check Secret resources:

```bash
kubectl get secret
```

Inspect Pod references:

```bash
kubectl describe pod <pod-name>
```

Avoid unnecessarily printing secret values during troubleshooting.

---

# CrashLoopBackOff vs OOMKilled

These are related but different concepts.

`CrashLoopBackOff` describes:

```text
Repeated container failure and restart behavior
```

`OOMKilled` describes:

```text
A container terminated because it exceeded available or allowed memory
```

Example:

```text
Application Consumes Too Much Memory
        ↓
Container OOMKilled
        ↓
Kubernetes Restarts Container
        ↓
Memory Problem Happens Again
        ↓
CrashLoopBackOff
```

You may therefore see:

```text
Current State:
Waiting
Reason:
CrashLoopBackOff
```

while:

```text
Last State:
Terminated
Reason:
OOMKilled
```

This distinction is important because the true root cause is memory exhaustion.

OOMKilled is covered in a separate lab.

---

# CrashLoopBackOff vs ImagePullBackOff

Do not confuse these two errors.

## CrashLoopBackOff

```text
Image successfully downloaded
      ↓
Container successfully started
      ↓
Process failed
      ↓
Container crashed
```

## ImagePullBackOff

```text
Kubernetes attempts to download image
      ↓
Download fails
      ↓
Container never starts
```

The troubleshooting approach is therefore different.

---

# Important Commands

## Check Pods

```bash
kubectl get pods
```

---

## Get Additional Information

```bash
kubectl get pods -o wide
```

---

## Describe the Pod

```bash
kubectl describe pod crashloop-pod
```

---

## Check Current Logs

```bash
kubectl logs crashloop-pod
```

---

## Check Previous Logs

```bash
kubectl logs crashloop-pod --previous
```

This is one of the most important commands in this lab.

---

## Follow Logs

```bash
kubectl logs -f crashloop-pod
```

---

## Check Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Inspect Full Pod YAML

```bash
kubectl get pod crashloop-pod -o yaml
```

---

# Troubleshooting Cheat Sheet

When you see:

```text
CrashLoopBackOff
```

use this process:

```text
1. kubectl get pods

2. kubectl describe pod <pod>

3. Check Current State

4. Check Last State

5. Check Exit Code

6. kubectl logs <pod>

7. kubectl logs <pod> --previous

8. Check Events

9. Identify application or configuration failure

10. Apply remediation

11. Redeploy

12. Verify Running status

13. Verify restart count stops increasing
```

---

# Production Remediation Strategy

Deleting a Pod alone is normally not a real fix.

For example:

```bash
kubectl delete pod crashloop-pod
```

may temporarily remove the failed Pod.

However:

```text
Broken Configuration Still Exists
       ↓
New Pod Starts
       ↓
Same Failure Happens
       ↓
CrashLoopBackOff Returns
```

The correct process is:

```text
Observe Failure
      ↓
Collect Logs
      ↓
Inspect Previous Logs
      ↓
Read Events
      ↓
Inspect Exit Code
      ↓
Identify Root Cause
      ↓
Fix Code / Config / Dependency
      ↓
Redeploy
      ↓
Verify
```

---

# Production Deployment Scenario

In production, workloads are normally managed by a Deployment.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
```

Example:

```bash
kubectl get deployments
```

Then:

```bash
kubectl get pods
```

Inspect the failing Pod:

```bash
kubectl describe pod <pod-name>
```

Read previous logs:

```bash
kubectl logs <pod-name> --previous
```

If multiple containers exist:

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

---

# AWS EKS Scenario

A realistic AWS environment could look like:

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Build Docker Image
   ↓
Amazon ECR
   ↓
Amazon EKS
   ↓
Deployment
   ↓
Backend Pod
```

Suppose a new image is deployed.

The application requires:

```text
DATABASE_HOST
```

but the environment variable was not configured.

Result:

```text
Container Starts
      ↓
Application Reads Environment
      ↓
DATABASE_HOST Missing
      ↓
Application Exits
      ↓
Kubernetes Restarts Container
      ↓
CrashLoopBackOff
```

Troubleshooting:

```bash
kubectl get pods
```

```bash
kubectl describe pod <backend-pod>
```

```bash
kubectl logs <backend-pod> --previous
```

Logs may reveal:

```text
DATABASE_HOST is required
```

Then inspect the Deployment:

```bash
kubectl describe deployment backend
```

or:

```bash
kubectl get deployment backend -o yaml
```

Fix the missing environment variable and redeploy.

---

# Cloud Engineering Relevance

Cloud Engineers frequently troubleshoot failed application containers.

`CrashLoopBackOff` requires understanding both Kubernetes and the application running inside Kubernetes.

Possible problem layers include:

```text
Kubernetes
    │
    ├── Pod Configuration
    ├── ConfigMap
    ├── Secret
    ├── Volumes
    └── Health Probes

Application
    │
    ├── Startup Command
    ├── Application Code
    ├── Environment Variables
    └── Dependencies

Cloud
    │
    ├── Database
    ├── IAM
    ├── Storage
    ├── Networking
    └── External Services
```

A good Cloud Engineer does not assume that `CrashLoopBackOff` means Kubernetes itself is broken.

---

# Cloud Security Relevance

`CrashLoopBackOff` may also reveal security-related problems.

Examples include:

- Application cannot access a required Secret
- IAM permissions are missing
- A security policy blocks a dependency
- Filesystem permissions prevent startup
- A hardened container prevents an unsafe operation
- A workload expects root privileges but runs as non-root
- NetworkPolicy prevents access to a required backend
- Secret rotation causes invalid application credentials

The correct security remediation should maintain least privilege.

Do not solve security-related startup failures by unnecessarily granting:

```text
cluster-admin
```

or:

```text
AdministratorAccess
```

Instead, identify the specific permission required.

---

# Prevention

## Validate Application Startup Before Deployment

Test container images locally before deployment.

Example:

```bash
docker run <image>
```

Verify that the main process remains running.

---

## Use Clear Application Logs

Applications should log useful startup failures.

Good:

```text
DATABASE_URL missing
```

Better than:

```text
Application failed
```

Clear logs reduce troubleshooting time.

---

## Validate Required Environment Variables

Applications should clearly validate required configuration during startup.

---

## Use Health Probes Carefully

Incorrect liveness probes can create restart loops.

Health checks must match actual application behavior.

---

## Monitor Restart Counts

A Pod may currently show:

```text
Running
```

while still continuously restarting.

Always check:

```text
RESTARTS
```

A high or increasing number requires investigation.

---

## Monitor Kubernetes Events

Alerting systems should detect repeated:

```text
BackOff
```

and:

```text
Unhealthy
```

events.

---

## Use Deployment Rollbacks

If a new release causes many Pods to enter `CrashLoopBackOff`, rolling back may restore service quickly.

Example:

```bash
kubectl rollout undo deployment/backend
```

Then investigate the failed version separately.

---

# Cleanup

Delete the fixed Pod:

```bash
kubectl delete pod crashloop-pod
```

Verify:

```bash
kubectl get pods
```

The lab environment should now be clean.

---

# What I Learned

In this lab, I learned that `CrashLoopBackOff` occurs when a container repeatedly starts, fails, and is restarted by Kubernetes.

I learned that the most useful troubleshooting information comes from:

```text
Pod Status
Events
Current Logs
Previous Logs
Exit Code
Last Container State
```

I also learned that:

```bash
kubectl logs <pod> --previous
```

is especially important because the current container instance may already be different from the one that originally failed.

The troubleshooting process should focus on identifying why the application process exited.

---

# Key Takeaways

- `CrashLoopBackOff` is a symptom, not the root cause.
- The image normally downloaded successfully.
- The container normally started successfully.
- The process inside the container then failed.
- Check the Pod restart count.
- Use `kubectl describe pod`.
- Inspect `Last State`.
- Inspect the exit code.
- Read current logs.
- Read previous logs.
- Do not fix the problem by repeatedly deleting the Pod.
- Fix the underlying application, configuration, dependency, or resource problem.
- Verify that the restart count stops increasing after remediation.

---

# Final Troubleshooting Flow

```text
                 CrashLoopBackOff
                         │
                         ▼
                 kubectl get pods
                         │
                         ▼
               kubectl describe pod
                         │
                         ▼
                  Check Last State
                         │
                         ▼
                   Check Exit Code
                         │
                         ▼
                   Current Logs
                         │
                         ▼
                  Previous Logs
                         │
                         ▼
                    Check Events
                         │
                         ▼
               Identify Root Cause
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Code        Config      Dependency
          Error        Error        Failure
             │           │           │
             └───────────┼───────────┘
                         ▼
                  Apply Remediation
                         │
                         ▼
                      Redeploy
                         │
                         ▼
                  Pod = Running
                         │
                         ▼
                 RESTARTS Stable
```