# Lab 05 — Pod Running but Application Failing

## Overview

This lab demonstrates an important Kubernetes troubleshooting scenario:

> A Pod can show `Running` while the application inside the container is still unavailable or misconfigured.

In this lab, the container process stays alive and Kubernetes reports the Pod as healthy-looking, but the application listens on a different port than expected.

The purpose of this lab is to learn how to:

- Understand the difference between Pod status and application health
- Use `kubectl logs`
- Use `kubectl describe pod`
- Test an application inside a container
- Identify port mismatches
- Understand what `containerPort` does and does not do
- Verify the actual listening port
- Apply remediation
- Verify application connectivity after the fix
- Understand why readiness probes matter in production

---

# Architecture

The broken architecture is:

```text
Kubernetes Pod
     │
     ▼
Container
     │
     ▼
Python HTTP Server
     │
     ▼
Listening on 8081
```

However, the Pod configuration describes:

```text
containerPort: 8080
```

So the mismatch is:

```text
Expected Port
8080

Actual Application Port
8081
```

The Pod remains:

```text
Running
```

because the process itself is still alive.

---

# Problem Scenario

A new application is deployed into Kubernetes.

Running:

```bash
kubectl get pods
```

shows:

```text
NAME                     READY   STATUS    RESTARTS   AGE
running-but-broken-pod   1/1     Running   0          20s
```

At first glance, everything looks healthy.

However, requests sent to the expected application port fail.

This creates an important troubleshooting question:

```text
If the Pod is Running,
why is the application unavailable?
```

The Cloud Engineer must investigate the application inside the Pod instead of assuming Kubernetes status tells the full story.

---

# Expected Behavior

The expected traffic flow is:

```text
Client
  ↓
Port 8080
  ↓
Container
  ↓
Application
  ↓
Successful Response
```

Expected result:

```text
Hello from Kubernetes
```

However, the broken container starts the application on port `8081`.

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
  name: running-but-broken-pod

spec:
  containers:
    - name: web
      image: python:3.12-slim

      command:
        - /bin/sh
        - -c
        - |
          mkdir -p /app
          echo "Hello from Kubernetes" > /app/index.html
          cd /app
          python -m http.server 8081

      ports:
        - containerPort: 8080
```

The important mismatch is:

```text
Application:
8081
```

while:

```text
containerPort:
8080
```

---

# Important Concept — containerPort Does Not Reconfigure the Application

This configuration:

```yaml
ports:
  - containerPort: 8080
```

does not make the application automatically listen on port `8080`.

It mainly documents the port the container is expected to expose and can be used by other Kubernetes tooling or manifests.

The application process still controls its own listening port.

In this lab:

```bash
python -m http.server 8081
```

means the actual application listens on:

```text
8081
```

regardless of:

```yaml
containerPort: 8080
```

---

# Reproduce the Problem

## Step 1 — Start Kubernetes

If using Minikube:

```bash
minikube start
```

Check:

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

---

# Step 2 — Apply the Broken Pod

Run:

```bash
kubectl apply -f broken.yaml
```

Expected:

```text
pod/running-but-broken-pod created
```

Now:

```bash
kubectl get pods
```

Expected:

```text
NAME                     READY   STATUS    RESTARTS   AGE
running-but-broken-pod   1/1     Running   0          10s
```

This is the important part.

There is no obvious Kubernetes failure state.

---

# Why the Pod Shows Running

Kubernetes sees that the main container process is alive.

The Python HTTP server is still running.

So from a container lifecycle perspective:

```text
Container Process Alive
       ↓
Container Running
       ↓
Pod Running
```

However:

```text
Running
≠
Application configured correctly
```

---

# Initial Investigation

Start with:

```bash
kubectl get pods
```

Then inspect the Pod:

```bash
kubectl describe pod running-but-broken-pod
```

You should see:

```text
State:
  Running
```

and likely:

```text
Ready:
  True
```

This tells us Kubernetes does not see a container lifecycle failure.

The investigation now needs to move inside the application.

---

# Check Application Logs

Run:

```bash
kubectl logs running-but-broken-pod
```

You should see something similar to:

```text
Serving HTTP on 0.0.0.0 port 8081
```

This is the first major clue.

The application is actually listening on:

```text
8081
```

not:

```text
8080
```

---

# Test the Expected Port

Run:

```bash
kubectl port-forward pod/running-but-broken-pod 8080:8080
```

In another terminal:

```bash
curl http://localhost:8080
```

The request should fail.

Possible result:

```text
connection refused
```

The reason is simple:

```text
Nothing inside the container is listening on port 8080.
```

Stop port forwarding with:

```text
Ctrl + C
```

---

# Test the Actual Application Port

Now forward:

```bash
kubectl port-forward pod/running-but-broken-pod 8081:8081
```

In another terminal:

```bash
curl http://localhost:8081
```

Expected:

```text
Hello from Kubernetes
```

This proves:

```text
Application Works
```

but:

```text
Application Configuration Does Not Match Expected Port
```

Stop port forwarding:

```text
Ctrl + C
```

---

# Enter the Container

Run:

```bash
kubectl exec -it running-but-broken-pod -- /bin/sh
```

Inside the container:

```bash
python --version
```

Then test the working application port:

```bash
python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8081').read().decode())"
```

Expected:

```text
Hello from Kubernetes
```

Now try the wrong port:

```bash
python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8080').read().decode())"
```

This should fail because nothing is listening on port `8080`.

Exit:

```bash
exit
```

---

# Root Cause

The root cause is:

```text
The application listens on port 8081,
but the expected Kubernetes application port is 8080.
```

Evidence:

```text
Pod Status:
Running
```

```text
Application Logs:
Serving HTTP on port 8081
```

```text
Port Test:
8080 fails
8081 succeeds
```

This confirms a port mismatch.

---

# Root Cause Diagram

```text
running-but-broken-pod
        │
        ▼
Python Container
        │
        ▼
HTTP Server
        │
        ├── Expected: 8080
        │
        └── Actual:   8081
                         │
                         ▼
                 Request to 8080 fails
```

---

# Remediation

The application should be configured to listen on the intended port.

Broken:

```bash
python -m http.server 8081
```

Fixed:

```bash
python -m http.server 8080
```

The application and Kubernetes configuration now agree.

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
  name: running-but-broken-pod

spec:
  containers:
    - name: web
      image: python:3.12-slim

      command:
        - /bin/sh
        - -c
        - |
          mkdir -p /app
          echo "Hello from Kubernetes" > /app/index.html
          cd /app
          python -m http.server 8080

      ports:
        - containerPort: 8080
```

Now:

```text
Application Port = 8080
```

and:

```text
containerPort = 8080
```

---

# Apply the Remediation

Delete the broken Pod:

```bash
kubectl delete pod running-but-broken-pod
```

Expected:

```text
pod "running-but-broken-pod" deleted
```

Apply the fixed configuration:

```bash
kubectl apply -f fixed.yaml
```

Expected:

```text
pod/running-but-broken-pod created
```

---

# Verification

Check the Pod:

```bash
kubectl get pods
```

Expected:

```text
NAME                     READY   STATUS    RESTARTS   AGE
running-but-broken-pod   1/1     Running   0          10s
```

The Pod status may look the same as before.

That is why application-level verification is necessary.

---

# Verify Logs

Run:

```bash
kubectl logs running-but-broken-pod
```

Expected:

```text
Serving HTTP on 0.0.0.0 port 8080
```

This confirms the application now listens on the expected port.

---

# Verify Application Connectivity

Run:

```bash
kubectl port-forward pod/running-but-broken-pod 8080:8080
```

In another terminal:

```bash
curl http://localhost:8080
```

Expected:

```text
Hello from Kubernetes
```

The complete verification flow is now:

```text
Pod Running
    ↓
Container Running
    ↓
Application Listening on 8080
    ↓
HTTP Request Reaches Application
    ↓
Successful Response
```

---

# Why Pod Status Alone Is Not Enough

Kubernetes can tell us a lot about infrastructure and container lifecycle.

For example:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
Running
```

But application health is another layer.

A Pod can be:

```text
Running
```

while the application is:

```text
Returning HTTP 500
```

or:

```text
Listening on Wrong Port
```

or:

```text
Unable to Reach Database
```

or:

```text
Unable to Access S3
```

or:

```text
Returning Invalid Responses
```

Therefore:

```text
Pod Health
≠
Application Health
```

---

# Real Production Example — Service Port Mismatch

Suppose the application listens on:

```text
8081
```

but the Service contains:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Traffic becomes:

```text
Client
   ↓
Service Port 80
   ↓
targetPort 8080
   ↓
Pod
   ↓
Nothing Listening
   ↓
Connection Failure
```

The Pod can still show:

```text
Running
```

This type of issue is covered more deeply in the Service troubleshooting labs.

---

# Real Production Example — HTTP 500

Suppose a backend process remains alive.

Kubernetes shows:

```text
Running
```

but every request returns:

```text
HTTP 500 Internal Server Error
```

Possible causes include:

```text
Application bug
Database failure
Missing configuration
External API failure
Invalid credentials
```

The container itself may remain healthy from Kubernetes' basic lifecycle perspective.

---

# Real Production Example — Database Failure

Architecture:

```text
User
 ↓
Backend Pod
 ↓
Database
```

Suppose the backend starts successfully.

The Pod becomes:

```text
Running
```

But the database is unavailable.

The application may return errors to users while the process remains alive.

Troubleshooting would involve:

```text
Application Logs
DNS
Network Connectivity
Database Port
Credentials
Database Health
```

---

# Real Production Example — Application Listening Only on localhost

Suppose an application binds to:

```text
127.0.0.1:8080
```

It may work when tested inside the container:

```bash
curl localhost:8080
```

but fail when traffic comes from another Pod or Service.

Applications that must accept external container traffic commonly need to listen on:

```text
0.0.0.0
```

instead.

This is a common container networking problem.

---

# Check Environment Variables

Application configuration is frequently controlled through environment variables.

Run:

```bash
kubectl exec running-but-broken-pod -- env
```

Possible variables include:

```text
PORT
APP_ENV
DATABASE_HOST
API_URL
LOG_LEVEL
```

A Pod may remain Running even if one of these values is incorrect.

Be careful not to expose sensitive environment variables in screenshots or logs.

---

# Check the Running Process

Depending on the container image, commands such as:

```bash
ps
```

or:

```bash
ps aux
```

may show the application process.

For this lab, the expected process is similar to:

```text
python -m http.server 8080
```

Inspecting the process helps confirm what application actually started.

---

# Check Pod IP

Run:

```bash
kubectl get pod running-but-broken-pod -o wide
```

This shows information such as:

```text
NODE
IP
STATUS
```

The Pod IP can be useful for debugging.

However, applications should normally not depend on hardcoded Pod IPs because Pod IPs can change.

Services provide stable discovery.

---

# Why Readiness Probes Matter

This lab does not configure a readiness probe.

Without one, Kubernetes does not actively test whether the application responds correctly on the expected endpoint.

A readiness probe can check something like:

```text
HTTP GET /health
Port 8080
```

Conceptually:

```text
Container Process Running
        ↓
Readiness Check
        ↓
Application Responding?
     ┌──┴──┐
    Yes    No
     │      │
     ▼      ▼
 Ready   Not Ready
```

This helps prevent traffic from being sent to an unhealthy application.

Readiness probe failures are covered in a later lab.

---

# Troubleshooting Workflow

When:

```text
Pod = Running
```

but:

```text
Application = Not Working
```

use:

```text
kubectl get pods
        ↓
kubectl describe pod
        ↓
kubectl logs
        ↓
Check Application Port
        ↓
kubectl exec
        ↓
Test Application Internally
        ↓
Check Service
        ↓
Check targetPort
        ↓
Check Configuration
        ↓
Check Dependencies
```

---

# Troubleshooting Checklist

## 1. Check Pod State

```bash
kubectl get pods
```

---

## 2. Describe Pod

```bash
kubectl describe pod <pod-name>
```

---

## 3. Read Logs

```bash
kubectl logs <pod-name>
```

---

## 4. Enter Container

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

---

## 5. Test Application Locally

Example:

```text
localhost:<application-port>
```

---

## 6. Check Listening Port

Confirm that the application's real port matches the intended Kubernetes configuration.

---

## 7. Check Environment Variables

```bash
kubectl exec <pod-name> -- env
```

---

## 8. Check Service Configuration

```bash
kubectl get svc
```

```bash
kubectl describe svc <service-name>
```

---

## 9. Check EndpointSlices

```bash
kubectl get endpointslices
```

---

## 10. Check Dependencies

Investigate:

```text
Database
External API
DNS
Cloud services
Authentication
NetworkPolicy
```

---

# AWS EKS Scenario

A production architecture may look like:

```text
Internet
   ↓
Route 53
   ↓
ALB
   ↓
Ingress
   ↓
Service
   ↓
Pod
   ↓
Application
```

Suppose:

```text
Pod = Running
```

but users receive:

```text
502 Bad Gateway
```

Possible causes include:

```text
Application listening on wrong port
Wrong Service targetPort
Incorrect ALB health check
Application bound to localhost
Service selector mismatch
Database unavailable
NetworkPolicy blocking traffic
```

The engineer must troubleshoot each layer rather than focusing only on the Pod state.

---

# End-to-End Traffic Model

A useful model to memorize is:

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
Pod IP
 ↓
Container Port
 ↓
Application Process
 ↓
Application Dependency
```

A failure at any layer can make the application unavailable.

---

# Cloud Engineering Relevance

This scenario is very common in Cloud Engineering because infrastructure can appear healthy while the application itself remains broken.

Cloud Engineers must understand:

- Pod lifecycle
- Container processes
- Application ports
- Kubernetes Services
- targetPort
- Application logs
- Environment variables
- Dependency connectivity
- Health probes
- Ingress
- Load Balancers

The key skill is learning to troubleshoot beyond the Kubernetes status column.

---

# Cloud Security Relevance

Security controls can also cause a Running application to fail.

Examples include:

```text
NetworkPolicy blocks database traffic
IAM denies S3 access
Secret contains invalid credentials
Filesystem policy blocks required access
SecurityContext prevents an operation
External API authorization fails
```

The container process may remain alive while required operations fail.

Security troubleshooting should identify the exact blocked dependency instead of disabling security controls broadly.

---

# Production Remediation Strategy

Do not assume a restart will solve the problem.

For example:

```text
Pod Running
   ↓
Wrong Application Port
```

Deleting the Pod creates:

```text
New Pod
   ↓
Same Configuration
   ↓
Same Wrong Port
   ↓
Same Failure
```

The real remediation is:

```text
Observe Symptoms
      ↓
Read Logs
      ↓
Test Application
      ↓
Compare Actual vs Expected Configuration
      ↓
Identify Mismatch
      ↓
Correct Configuration
      ↓
Redeploy
      ↓
Verify Application Response
```

---

# Prevention

## Use Consistent Port Configuration

Keep application port settings aligned across:

```text
Application
Container
Deployment
Service
Ingress
Load Balancer
```

---

## Use Readiness Probes

Configure application-level health checks so Kubernetes knows whether the application can receive traffic.

---

## Log Startup Configuration

Applications should log useful startup information such as:

```text
Listening on port 8080
Environment: production
```

without exposing secrets.

---

## Test Applications Before Deployment

Validate the container locally before deploying:

```bash
docker run ...
```

Then test the expected application port.

---

## Monitor Application Metrics

Pod status alone is not enough.

Monitor:

```text
HTTP error rate
Latency
Health endpoints
Dependency health
Request success rate
```

---

# Cleanup

Delete the lab Pod:

```bash
kubectl delete pod running-but-broken-pod
```

Verify:

```bash
kubectl get pods
```

The lab environment should now be clean.

---

# What I Learned

In this lab, I learned that a Kubernetes Pod showing `Running` does not guarantee that the application inside it is healthy or reachable.

I learned that the correct troubleshooting process must move beyond Kubernetes Pod status and inspect:

```text
Application logs
Application port
Application process
Configuration
Network path
Dependencies
```

I also learned that:

```yaml
containerPort: 8080
```

does not force an application to listen on port `8080`.

The application itself controls its listening port.

---

# Key Takeaways

- `Running` means the container process is running.
- `Running` does not guarantee application health.
- Always inspect application logs.
- Test the application directly.
- Verify the actual listening port.
- `containerPort` does not reconfigure application behavior.
- Compare actual configuration with intended configuration.
- Check environment variables and dependencies.
- Use readiness probes in production.
- Verify the complete traffic path.
- A restart alone will not fix persistent misconfiguration.

---

# Final Troubleshooting Flow

```text
              Pod = Running
                    │
                    ▼
          Application Still Fails
                    │
                    ▼
                Check Logs
                    │
                    ▼
          Check Application Port
                    │
                    ▼
          Test Inside Container
                    │
                    ▼
         Compare Actual vs Expected
                    │
                    ▼
             Identify Mismatch
                    │
                    ▼
              Fix Configuration
                    │
                    ▼
                 Redeploy
                    │
                    ▼
            Test Application
                    │
                    ▼
             Successful Response
```