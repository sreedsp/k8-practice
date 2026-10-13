# Kubernetes Essential Commands Sheet

Pattern for any object: `kubectl get <type>` to list, `kubectl describe <type> <name>` to inspect, `-A` for all namespaces.
Look up flags with `kubectl <command> --help`, and YAML fields with `kubectl explain <path>`.

---

## Habits (before every change)

```bash
kubectl config current-context        # which cluster am I on?
kubectl get pods -A                   # don't assume the namespace
kubectl api-resources                 # all object types and short names
kubectl explain pod.spec.containers   # YAML field help
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml   # generate YAML to edit

kubectl get ns                                   # list namespaces
kubectl get pods -n kube-system                  # one namespace
kubectl get pods -A                              # all namespaces
kubectl create namespace dev
kubectl create deployment web --image=nginx -n dev
kubectl config set-context --current --namespace=dev   # make dev your default
kubectl config view --minify | grep namespace    # which namespace am I in?
```

---
## Namespaces

```bash
kubectl get ns
kubectl create namespace dev
kubectl get pods -n dev
kubectl get pods -A
kubectl create deployment web --image=nginx -n dev
kubectl config set-context --current --namespace=dev     # change the default namespace
kubectl delete namespace dev                             # removes everything inside it
```

- A command without `-n` acts on the current namespace (`default` unless changed). Check before deleting.
- The same name can exist in different namespaces.
- Namespaces separate names, RBAC and quotas, but not network traffic by default.
- Cross-namespace DNS: `<service>.<namespace>.svc.cluster.local`.
- `kube-system` holds the cluster's own components (API server, etcd, scheduler, controller manager, CoreDNS).

## Deployments and Pods

```bash
kubectl create deployment <name> --image=<image> --replicas=<n>
# example
kubectl create deployment web --image=nginx --replicas=3
```

This creates a Deployment named `web` with the `nginx` image and 3 replicas.

```bash
kubectl get pods
```

```text
NAME                   READY   STATUS              RESTARTS   AGE
web-7887448d46-4grsw   1/1     Running             0          19s
web-7887448d46-bfq2q   1/1     Running             0          19s
web-7887448d46-qrqh4   0/1     ContainerCreating   0          19s
```

Pod name = `web` (the Deployment) + `7887448d46` (hash identifying the ReplicaSet) + `4grsw` (random suffix, unique to the Pod).

```bash
kubectl get pods -o wide              # adds Pod IP and node
kubectl get pods --show-labels        # Pods with their labels
kubectl get pods -l app=web           # filter by label
kubectl get pods -w                   # live status, Ctrl+C to stop
kubectl get rs                        # ReplicaSets (new one up, old one at 0)
```

```text
NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE
web-7887448d46-bfq2q   1/1     Running   0          6m18s   10.1.0.13   docker-desktop
```

### Scaling

```bash
kubectl scale deployment/web --replicas=5
kubectl scale deployment/web --current-replicas=3 --replicas=5   # only if it is currently 3
```

### Cleaning up

```bash
kubectl delete svc <name>
kubectl delete deployment <name>
kubectl delete pod <name>
kubectl delete deployment web shop    # several names separated by spaces
```

---

## Services

```bash
kubectl expose deployment web --type=ClusterIP --port=8080 --target-port=80
kubectl get svc
kubectl describe svc web
kubectl get endpoints web             # which Pod IPs the Service routes to
```

| Port | Meaning |
|---|---|
| `--port` | Front door: the port the Service listens on (clients use this) |
| `--target-port` | Back door: the port the container listens on (Service forwards here) |
| NodePort | Side door: an extra port on every node for access from outside |

### Test and reach a Service

```bash
# reach a ClusterIP from your Mac (local port : service port), Ctrl+C to stop
kubectl port-forward svc/web2 9090:8080

# test from inside the cluster
kubectl run tmp --rm -it --restart=Never --image=curlimages/curl -- curl -sS --max-time 5 http://<svc>:<port>
```

| curl error | Usually means |
|---|---|
| 6 | Could not resolve host: DNS or wrong Service name |
| 7 | Connection refused: nothing listening, no endpoints, or wrong port |
| 28 | Timed out: network policy, firewall or wrong IP |

### Change one field of an existing resource

```bash
kubectl patch svc <name> -p '{"spec":{"selector":{"app":"<label value>"}}}'
# example: fix a selector typo so it matches the Pods' label app=shop
kubectl patch svc shop -p '{"spec":{"selector":{"app":"shop"}}}'
```

---

## Logs and diagnosing Pods

```bash
kubectl logs <pod>
kubectl logs <pod> --previous         # logs from the last crashed container
kubectl logs -l app=web --prefix      # logs from all matching Pods, prefixed with the Pod name
kubectl describe pod <pod>            # State, Last State, Exit Code, Events
```

In `describe pod`, check: **State / Last State**, **Exit Code**, and the **Events** section at the bottom.

| Pod status | Meaning | First check |
|---|---|---|
| `ErrImagePull` | The image pull failed just now | Events message: name or tag wrong? |
| `ImagePullBackOff` | Waiting before retrying the pull | Same as above |
| `CrashLoopBackOff` | Container starts, exits, restarts repeatedly | `logs --previous`, exit code |
| `Pending` | Not scheduled | `describe pod` (resources, missing PVC) |

| Exit code | Meaning |
|---|---|
| 0 | Success |
| 1 | General application error |
| 126 | Command found but not executable |
| 127 | Command not found |
| 137 | Killed, often out of memory |

Gotchas:
- A `command` list in Pod YAML has **no shell**. For `&&` use `["sh", "-c", "echo hi && sleep 3600"]` and keep `sleep` last.
- `connection refused` to `127.0.0.1:6443` means the local cluster isn't running. Start Docker Desktop, then `kubectl get nodes`.

---

## Rollouts (Day 4)

```bash
kubectl set image deployment <name> <container>=<image>   # container name is LEFT of =
# examples
kubectl set image deployment broken1 nginix=nginx
kubectl set image deployment web nginx=nginx:1.26

kubectl rollout status deployment <name>                  # waits until the rollout finishes (pipeline gate)
kubectl rollout history deployment <name>                 # list revisions
kubectl rollout history deployment <name> --revision=3    # see what a revision contained
kubectl rollout undo deployment <name>                    # back one revision
kubectl rollout undo deployment <name> --to-revision=1    # back to a specific revision
kubectl rollout restart deployment <name>                 # replace Pods (picks up new env vars)
kubectl annotate deployment <name> kubernetes.io/change-cause="<note>"   # fill CHANGE-CAUSE
```

```bash
kubectl rollout status deployment web
```

```text
deployment "web" successfully rolled out
```

```bash
kubectl rollout history deployment web
```

```text
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```

Notes:
- `rollout status` is how you tell whether a rollout is finished or stuck. It reports the rollout, not why it failed.
- Defaults: `maxSurge` 25% (rounds up) and `maxUnavailable` 25% (rounds down). With 3 replicas: 1 extra Pod, 0 unavailable.
- Bad image: the new Pod never becomes Ready, the rollout stalls, and the old Pods keep serving. Roll back first, investigate after.
- The highest revision number is the version running now. After an `undo`, the old template reappears under a new number (history `1, 2, 3` becomes `1, 3, 4`).
- `CHANGE-CAUSE <none>` is normal. It only means no note was set. In CI/CD it is usually filled in.

---

## ConfigMaps and Secrets (Day 5)

**ConfigMap** holds plain settings. **Secret** holds passwords (base64-encoded, not encrypted).

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=dev
kubectl get configmap app-config -o yaml
```

`--from-literal=KEY=value`: repeat the flag once per key, with no brackets and no spaces around `=`.

```bash
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=Test123
kubectl get secret app-secret -o yaml
```

Secret types for `kubectl create secret <type>`:

| Type | Use |
|---|---|
| `generic` | From a literal value, a file, or a directory |
| `docker-registry` | Credentials for a container registry |
| `tls` | A TLS certificate and key |

The stored value is encoded:

```text
data:
  DB_PASSWORD: VGVzdDEyMw==
```

```bash
echo 'VGVzdDEyMw==' | base64 -d
```

```text
Test123
```

(A `%` after the output is just zsh marking a missing newline.)

### Load every key into a Pod

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

### Check values inside a Pod

```bash
kubectl exec env-demo -- env | grep -E 'APP_|DB_'
```

### Find ConfigMaps and Secrets

```bash
kubectl get cm,secret                 # cm = configmap
kubectl get cm -A                     # every namespace
kubectl describe cm app-config
kubectl get secret app-secret -o yaml
kubectl edit configmap app-config
```

### Find which ConfigMap a Pod uses

```bash
kubectl describe pod env-demo
kubectl get pod env-demo -o yaml
```

Look under `envFrom`, or `env` / `volumes`, for `configMapRef` or `secretRef`:

```text
Environment Variables from:
  app-config  ConfigMap  Optional: false
  app-secret  Secret     Optional: false
```

Notes:
- Env vars are read once when the container starts. After changing a ConfigMap, restart the Pod or run `rollout restart`.
- A ConfigMap mounted as a file refreshes by itself after a short delay.
- Secrets are protected by RBAC, encryption at rest, and (on Azure) Key Vault. Never commit secret values to Git.

---

## Volumes and Persistent Storage (Day 6)

A container's filesystem is temporary and is lost with the Pod.

| Object | What it is |
|---|---|
| Volume | A directory mounted into the container. `emptyDir` survives container restarts but not Pod deletion. |
| PersistentVolumeClaim (PVC) | A request for storage |
| PersistentVolume (PV) | The actual storage that satisfies the claim |
| StorageClass | The recipe that creates a PV automatically when a PVC asks for one (Azure Disk on AKS) |

```bash
kubectl get storageclass
kubectl get pvc,pv                    # comma, no space
kubectl exec my-pod -- sh -c 'echo hello > /data/note.txt'
kubectl exec my-pod -- cat /data/note.txt
```

PVC template:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Pod that mounts it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: busybox
      command: ["sh", "-c", "echo Hello && sleep 3600"]
      volumeMounts:
        - mountPath: /data
          name: my-volume
  volumes:
    - name: my-volume
      persistentVolumeClaim:
        claimName: my-pvc
```

Notes:
- `volumes` goes at Pod `spec` level. `volumeMounts` goes inside the container. The two `name` values must match.
- Create the PVC before the Pod, or the Pod stays `Pending`.
- Reclaim policy decides what happens to the data when the PVC is deleted: `Delete` removes the PV and the data, `Retain` keeps them. Check with `kubectl get pv` (RECLAIM POLICY column).
- Deleting a Pod does not lose PVC data. Deleting the PVC can.

---

## Git

```bash
git remote -v
git remote set-url origin git@github.com:sreedsp/<repo>.git   # SSH, no token needed
ssh -T git@github.com
git add .
git commit -m "message"
git push -u origin main
```

- `https://` URLs need a token. `git@github.com:` uses your SSH key.
- Check `git status` before `git add .` so no secrets or large files go in.
- Typing a backtick on this Mac: `Option` + the key left of `1`.