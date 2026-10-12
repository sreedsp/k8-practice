# Kubernetes Troubleshooting Runbook

Local cluster: Docker Desktop. Each scenario: **Setup → Symptom → Checked → Cause → Fix → Verified**.

---

## Quick Reference

### Pod statuses

| Status | Meaning | First check |
|---|---|---|
| `ErrImagePull` | Could not pull the image | `kubectl describe pod <pod>` (Events) |
| `ImagePullBackOff` | Retrying the pull with back-off | Image name / tag / registry access |
| `CrashLoopBackOff` | Container starts, exits, restarts repeatedly | `kubectl logs <pod> --previous` |
| `Pending` | Not scheduled | `kubectl describe pod <pod>` (resources, nodes) |
ErrImagePull = pull failed. 
ImagePullBackOff = waiting to retry. The Events message tells you which part was wrong, the name or the tag."

### curl error codes

| Code | Meaning | Usually |
|---|---|---|
| 6 | Could not resolve host | DNS / wrong Service name |
| 7 | Failed to connect | Nothing listening / no endpoints / wrong port |
| 28 | Timed out | Network policy / firewall / wrong IP |

### Container exit codes

| Code | Meaning |
|---|---|
| 0 | Success (normal exit) |
| 1 | Application error |
| 126 | Command found but not executable |
| 127 | Command not found |
| 137 | Killed (SIGKILL) – often OOM or `kubectl delete` |

---

## Scenario 1 – Wrong image name (ErrImagePull)

### Setup

```bash
kubectl create deployment broken1 --image=nginix
```

### Symptom

```bash
kubectl get pods
```

```text
NAME                       READY   STATUS             RESTARTS   AGE
broken1-66d5c478c6-xxxxx   0/1     ErrImagePull       0          10s
broken1-66d5c478c6-xxxxx   0/1     ImagePullBackOff   0          25s
```

### Checked

```bash
kubectl describe pod <pod-name>
```

> **TODO:** paste the `Events:` section here (Failed to pull image "nginix" ... repository does not exist).

### Cause

Typo in the image name: `nginix` instead of `nginx`.

### Fix

```bash
kubectl set image deployment broken1 nginix=nginx
```

(`nginix` on the left is the **container name**, which was also created from the typo'd image name.)

### Verified

```bash
kubectl get pods
```

A new Pod appears and reaches `Running`. The Pod name hash changed (`66d5c478c6` → `f9d49868f`) because the Deployment created a **new ReplicaSet** (rolling update).

---

## Scenario 2 – Service selector does not match Pod labels

### Setup

Deployment `shop` with Pods labelled `app=shop`; Service `shop` created with selector `shopp` (typo).

### Symptom

Service exists, but traffic never reaches Pods.

```bash
kubectl get endpoints shop
```

```text
NAME   ENDPOINTS   AGE
shop   <none>      1m
```

### Checked

```bash
kubectl get pods --show-labels
kubectl describe svc shop
kubectl get endpoints shop
```

### Cause

Service selector (`shopp`) did not match the Pod label (`app=shop`), so the Service had **no endpoints**.

### Fix

```bash
kubectl patch svc shop -p '{"spec":{"selector":{"app":"shop"}}}'
```

### Verified

```bash
kubectl describe svc shop
kubectl get endpoints shop
```

Endpoints now list Pod IPs. Test from inside the cluster:

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl http://shop
```

> **TODO:** paste the curl output here (expect the nginx welcome page).

Test from inside the cluster. 

```bash 
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl -v http://<svc>:<port>
```
The curl error code tells you where it breaks: `6 means DNS or the wrong Service name`, `7 means connection refused`, `28 means timeout`.

Check the `front door`. Run `kubectl describe svc <name>` and confirm Port is the one you're calling.

Check the `back door`. In the same output, confirm TargetPort equals the port the container really listens on. Then `kubectl logs <pod>` confirms the app is up on that port.
---

## Scenario 3 – CrashLoopBackOff

### Setup

```bash
kubectl run crashy --image=busybox -- sh -c "echo starting; sleep 5; exit 1"
```

### Symptom

```bash
kubectl get pods
```

Status cycles `Running` → `Error` → `CrashLoopBackOff`; `RESTARTS` keeps increasing.

### Checked

```bash
kubectl describe pod crashy          # Last State, Exit Code, Events
kubectl logs crashy --previous       # logs of the crashed container
```

### Cause

The container's command exits with code **1** after 5 seconds. Kubernetes restarts it each time, with growing back-off delays.

### Fix

Correct the command/application so it does not exit (or exits 0 for a one-off job).

> **TODO:** paste `kubectl get pods` after the fix (expect `Running`, `RESTARTS 0`).

### Real-life playbook

1. **What changed?** Recent deploy, config, image tag, secret.
2. `kubectl describe pod` – Last State, exit code, Events.
3. `kubectl logs <pod> --previous` – the crashed run, not the fresh one.
4. **Triage the exit code** (see table above): 1 = app error, 127 = bad command, 137 = killed/OOM.
5. **Roll back first in production**, investigate after:
   ```bash
   kubectl rollout undo deployment <name>
   ```
6. **Collect evidence for the developer:** logs, describe output, exit code, timeline, what changed.

---

### Symptom 

## Concept Notes

### Label vs Selector

- **Label**: key=value tag on an object (Pod has `app=shop`).
- **Selector**: a query that finds objects by label (Service selects `app=shop`).
- A Service only routes to Pods whose labels match its selector.

### Ports

| Field | Meaning |
|---|---|
| `port` | Front door – the port the Service listens on |
| `targetPort` | Back door – the port the container listens on |
| `nodePort` | Port opened on every node (NodePort / LoadBalancer) |

### Three addresses

1. Pod IP – changes when the Pod is replaced.
2. Service ClusterIP / DNS name – stable inside the cluster.
3. NodePort / LoadBalancer IP – reachable from outside.

### Controllers

Deployment → ReplicaSet → Pods. The controller keeps the actual state equal to the desired state (reconciliation loop).

### Check order when an app is unreachable

1. `kubectl get pods` – Running? Ready?
2. `kubectl describe pod` / `kubectl logs` – errors?
3. `kubectl get svc` and `kubectl get endpoints <svc>` – endpoints present?
4. Selector vs labels – match?
5. Ports – `port` / `targetPort` correct?
6. Test from a temporary Pod with curl.

### Linux

| Keys | Effect |
|---|---|
| `Ctrl+C` | Terminates the foreground process |
| `Ctrl+Z` | Suspends it (still alive in the background) |

```bash
jobs          # list suspended/background jobs
kill %1       # kill job number 1
```

# maxSurge rounds up to 1, so Kubernetes may run 1 extra new Pod (4 in total).
# maxUnavailable rounds down to 0, so all 3 old Pods must stay available.