# K8 essentaial commands code sheet 
To create a deployment 

kubectl create deployment <name> --image=<image_detials> --replicas=<#ofreplicas>
ex: kubectl create deployment web --image=nginx --replicas=3 

# This will create a new deployment name "web" with the image "nginx" with 3 replicas 
kubectl get pods    
NAME                   READY   STATUS              RESTARTS   AGE
web-7887448d46-4grsw   1/1     Running             0          19s
web-7887448d46-bfq2q   1/1     Running             0          19s
web-7887448d46-qrqh4   0/1     ContainerCreating   0          19s

here web is the deployment 7887448d46 is the hash identifying the ReplicaSet
4grsw is random suffix unique to that pod 

# To get the full details about the deployment with IP address 
kubectl get pods -o wide

NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE             NOMINATED NODE   READINESS GATES
web-7887448d46-bfq2q   1/1     Running   0          6m18s   10.1.0.13   docker-desktop   <none>           <none>
web-7887448d46-g4jq6   1/1     Running   0          5m13s   10.1.0.14   docker-desktop   <none>           <none>
web-7887448d46-qrqh4   1/1     Running   0          6m18s   10.1.0.12   docker-desktop   <none>           <none>
sreejithramachandrapanicker@Sreejiths-MacBook-Air ~ % 

# Pods with their Labels 
kubectl get pods --show-lables 

# Filter pods by label 
kubectl get pods -l app=web

# Change replica count 
kubectl scale deployment/web --replicas=N
kubectl scale deployment/web --current-replica=n --target-replica=n+n

# Create a service 
kubectl expose deployment web --type=ClusterIP --port=8080 --target-port=80

--port = The port the service listens on front. Client uset his 
--target-port- The port the container listens on back - the service forwards here 
NodePort=an extra port on every node for access from outside.

# Inspect services 
kubectl get svc 
kubectl describe svc web 

# Which endpoints the services route 
kubectl get endpoints web 

# Checking the logs from matching pods 
kubectl logs -l app=web --prefic 

# Reach cluster IP from you device 
kubectl port-forward svc/web2 9090:8080

# Test a service from inside the cluster 
kubectl run tmp -rm -it --restart=Never --image=curlimages/curl -- curl sS --max-time 5 <srv>:<port>

# lookup Flags
<command> --help

# Filter the pods and show with labels 
kubectl get pods -l app=<labelname>
ex: kubectl get pods -l app=web

# Read the pod status, to describe the pod or svc 
kubectl describe pod <name>

to check the pod status you can check the args section 
exit code 
last status 

exit code status codes 
0	Success
1	General application error
127	Command not found
137	Killed, often out of memory

# Change image of running deployment using set image
kubectl set image deployment/<name> <container-name>=<image>
ex: kubectl set image deployment broken1 nginix=nginx 
kubectl set image deployment web nginx=nginx:1.26
# Change one field of existing resource - patch command in the service 
kubectl patch svc <name> -p '{"spec":{"selector":{"app"="<new name>"}}}'
e: kubectl patch svc shop -p '{"spec":{"selector":{"app":"shopp"}}}'  

# Get pods live status use ctrl + c to terminate 
kubectl get pods -w 

# Cleaning up (you can use multiple names to delete with space)
kubectl delete svc 
kubectl delete deployment 
kubectl delete pod 

# Getting logs 
kubectl logs <name>
# logs from last crashed container 
kubectl logs <name> --previous 

# To know the Replica set details 
```bash
kubectl get rs 
NAME             DESIRED   CURRENT   READY   AGE
web-7b46c4b544   3         3         3       32s
```

# To know the status of deployment rollout - example you have a deployment with nginx 1.25 container and you updated them to nginx:1.26 
# the update done need to know if the pod is stuck or not 
```bash
kubectl rollout status deployment web #<web is the deployment name>
deployment "web" successfully rolled out
```

# To verify the pods status 
```bash 
kubectl get rs
deployment.apps/web 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

# To know the history of rollout in deployment 
```bash 
kubectl rollout history deployment web
deployment.apps/web 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
as of now it shows 000 but it is useful when we do in CICD 

## Rollouts (Day 4)

```bash
kubectl set image deployment <name> <container>=<image>   # update the image
kubectl rollout status deployment <name>                  # wait until the rollout finishes (pipeline gate)
kubectl rollout history deployment <name>                 # list revisions
kubectl rollout history deployment <name> --revision=3    # see what a revision contained
kubectl rollout undo deployment <name>                    # go back one revision
kubectl rollout undo deployment <name> --to-revision=1    # go back to a specific one
kubectl get rs                                            # ReplicaSets: new one up, old one at 0
kubectl annotate deployment <name> kubernetes.io/change-cause="<note>"   # fill CHANGE-CAUSE
```

Notes:
- The container name is on the left of `=` in `set image`.
- The highest revision number is the currently running version.
- `CHANGE-CAUSE <none>` is normal, it only means no note was set.
- A bad rollout stalls and the old Pods keep serving. Roll back first, investigate after.
======================================================================================================

## Day 5: ConfigMaps and Secrets 

# `ConfigMaps` - Holds plain settings

``` bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=dev 

--from-literal=KEY=value - for each key you need to provide seperate --from-literal per key with no brackets and no spaces around =

```
- To check what was stored 
```bash
kubectl get configmap app-config -o yaml 

```

# `Secrets` - Holds passwords 
``` bash
kubectl create secret <command> <secret-name> --from-literal=DB_PASSWORD=Test123

ex: 
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=Test123

```
`<command>`
 docker-registry   Create a secret for use with a Docker registry
 generic           Create a secret from a local file, directory, or literal value
 tls               Create a TLS secret

`When you read the file it will show encoded value`
```bash
kubectl get configmap app-secret -o yaml
apiVersion: v1
data:
  DB_PASSWORD: VGVzdDEyMw==
kind: Secret
metadata:
  creationTimestamp: "2026-10-11T16:03:32Z"
  name: app-secret
  namespace: default
  resourceVersion: "123005"
  uid: 4472652c-2448-4d8d-b18d-03058c8abecc
type: Opaque
```
`VGVzdDEyMw==` - this is encoded value of Test123 - you can check it using the command 

```bash 
echo 'VGVzdDEyMw==' | base64 -d  
Test123%   
```
# to check the values in a pod using exec 
```bash 
kubectl exec <pod-name> -- env | grep -E <>

kubectl exex env-demo -- env | grep -E 'APP_|DB' 
```
# To get the config details 
```bash 
kubectl get configmap          # or the short form: kubectl get cm
kubectl get secret
kubectl get cm -A              # every namespace, not just default
kubectl get cm,secret          # both together
kubectl describe cm app-config # keys and values
kubectl get secret app-secret -o yaml   # full content
```
# To find which ConfigMap a pod uses 
```bash
kubectl describe pod env-demo
kubectl get pod env-demo -o yaml
```
In the output, look under `envFrom`, or `env`/`volumes` for `configMapRef or secretRef` 

Name:             env-demo
Namespace:        default
Priority:         0
Service Account:  default
Node:             docker-desktop/192.168.65.3
Start Time:       Sun, 11 Oct 2026 22:23:31 +0530
Labels:           <none>
Annotations:      <none>
Status:           Running
IP:               10.1.0.50
IPs:
  IP:  10.1.0.50
Containers:
  env-demo:
    Container ID:   docker://481232e1ec97a8f87aef0e6dd009e850fa38cebc5366e166e09728ad93a5bf8d
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:f9ea18bfa4fad859e1ed38259d711da7ccad2c3516e875cec3351a57c859f571
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 11 Oct 2026 22:23:36 +0530
    Ready:          True
    Restart Count:  0
    Environment Variables from:    # Check here 
      app-config  ConfigMap  Optional: false
      app-secret  Secret     Optional: false
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-w44nq (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-w44nq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort

```bash
kubectl api-resources 
```