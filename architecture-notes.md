# Step 2

## Where does isolation happen?

It happens at the pod level: each pod (db included) has an isolated system.

## What restarts automatically?

The pods restart automatically if it crashes but the DB pod keeps the same persistant data

## What does Kubernetes not manage?

- Persistant storage
- Backups

# Step 3

|          | Containers | Virtual machines |
| -------- | ---------- | ---------------- |
| Kernel sharing | Shares the host kernel | Fully independant kernel |
| Startup time | Takes seconds or milliseconds to start | Takes minutes (has to start a full OS) |
| Resource overhead | Lightweight since it shares the host binaries | Heavy since it's a full OS |
| Security Isolation | Medium security | High security |
| Complexity | Easy to integrate in a CI/CD | More complex architecture |

## When would you prefer a VM over a container?

A container is better for light applications without a lot of traffic and without a need for high security since it's lightweight. A VM is better if there's a lot of traffic, or a need for hight security.

## When would you combine both?

Additional security ?

# Step 4

## What changes when you scale?

The LoadBalancer doesn't forward to the same pod each time.
Each pod have a different port.

# What does not change?

The pods should work the same way. The entry point still goes to the service.

# Step 5

## Who recreated the pod?

The deployment controller recreated the pod.

## Why?

Because it should always have 3 replicas.

## What would happen if the node itself failed?

Kubernetes would have restarted the pods on other healthy nodes if there are some.

# Step 6

## What are requests vs limits?

Limits are the maximum allocated resources, and requests are the resources allocations requested by the pod.

## Why are they important in multi-tenant systems?

They're important to make sure the pods don't take too much resources and potentially make the system crash.

# Step 7

## What is the difference between readiness and liveness?

Readiness is the check at the start of the pod to known if it's ready.

Liveness is a check executed every x seconds when the pod is ready to known if it's still working.

## Why does this matter in production?

It's important to known if the app isn't broken even if it has not crashed.

# Connect Kubernetes to virtualization 

## What runs underneath your k3s cluster?

Virtual machines

## Is Kubernetes replacing virtualization?

No, it's a different use

## In a cloud provider, what actually hosts your nodes?

/

# Step 8

## What would run in Kubernetes?

Nodes, Database, Monitoring, Logging

## What would run in VMs?

Nodes

## What would run outside the cluster?

Backups, CI/CD

# Step 9

## Why is this better than plain-text configuration?

Because it's secret : to not expose api keys. Plain-text credentials in a manifest can be read by anyone with access to the repository or the cluster's etcd. Secrets allow credentials to be stored separately from the application code and only mounted into pods that need them.

## Is a Secret encrypted by default? Where?

No, it's only base64-encoded by default (not encrypted). Encryption at rest must be enabled explicitly in the Kubernetes API server configuration (`EncryptionConfiguration`). In managed cloud clusters, providers like GKE/EKS often enable envelope encryption automatically.

# Step 10

## Phase 1 & 2 – Versioned images

Built two versioned images from the same app codebase:
- `quote-app:v1` — original version (tagged from `quote-app:local`)
- `quote-app:v2` — updated version with "QuoteBoard v2" page title change

Commands used:
```bash
docker tag quote-app:local quote-app:v1
# (made code change to views/index.hbs)
docker build -t quote-app:v2 -f docker/Dockerfile .
docker save quote-app:v1 | k3s ctr images import -
docker save quote-app:v2 | k3s ctr images import -
```

## Phase 3 & 4 – Controlled rollout (v1 → v2)

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment quote-app
```

Output:
```
Waiting for deployment "quote-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "quote-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "quote-app" rollout to finish: 1 old replicas are pending termination...
deployment "quote-app" successfully rolled out
```

### What changed during the rollout?

Kubernetes replaced pods one at a time (maxSurge: 1, maxUnavailable: 0). A new pod was started and had to pass the readiness probe before an old pod was terminated. The ReplicaSet hash changed from `6b6584d6f` to `6d5487b76b`.

### What remained constant?

The Service kept routing traffic without interruption. The Deployment name and selector labels did not change. The application remained available throughout (zero downtime).

### How did Kubernetes determine Pod creation/deletion timing?

The readiness probe controls timing: a new pod must report ready before the old one is terminated. With `maxUnavailable: 0`, Kubernetes never decreases below the desired replica count, so it first adds a new pod (surge), waits for readiness, then removes an old one.

## Phase 5 – Controlled failure (broken image)

```bash
kubectl set image deployment/quote-app quote-app=quote-app:broken
kubectl get pods
kubectl get events --sort-by='.lastTimestamp'
```

Output:
```
NAME                         READY   STATUS             RESTARTS   AGE
quote-app-6d5487b76b-4shkh   1/1     Running            0          69s
quote-app-6d5487b76b-bstfm   1/1     Running            0          55s
quote-app-6d5487b76b-t2lbc   1/1     Running            0          62s
quote-app-6f866b9f7-xkqjl    0/1     ErrImagePull       0          26s
```

Key event:
```
Warning   Failed   pod/quote-app-6f866b9f7-xkqjl   Failed to pull image "quote-app:broken":
  failed to resolve reference "docker.io/library/quote-app:broken": pull access denied
Warning   Failed   pod/quote-app-6f866b9f7-xkqjl   Error: ImagePullBackOff
```

### Which component failed first?

The new pod's image pull failed immediately — the container never started.

### Which signal indicated failure fastest?

`ErrImagePull` appeared within seconds (before any probe could even run). The existing v2 pods were unaffected because `maxUnavailable: 0` prevented Kubernetes from terminating them until a new pod was ready — which never happened.

### Production troubleshooting steps

1. `kubectl get pods` — identify the failing pod
2. `kubectl describe pod <pod>` — see the exact pull error and events
3. Verify the image tag exists in the registry
4. `kubectl rollout undo` — rollback immediately while investigating

## Phase 6 – Safe rollback

```bash
kubectl rollout undo deployment quote-app
kubectl rollout status deployment quote-app
```

Output:
```
deployment.apps/quote-app rolled back
deployment "quote-app" successfully rolled out
```

### What did the rollback change?

Kubernetes restored the previous ReplicaSet (`quote-app-6d5487b76b`, running v2 image). The broken ReplicaSet was scaled back to 0.

### What did the rollback NOT change?

The rollout history was preserved (revision 5 was added for the broken attempt, revision 6 for the rollback). The Secret, Service, and resource limits were untouched.

# Step 11 – Rollout Strategy (Option A)

Added explicit rolling update strategy to `deployment.yaml`:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

## What is maxSurge?

`maxSurge: 1` allows Kubernetes to create one extra pod beyond the desired replica count during a rollout. With 3 replicas, there can be up to 4 pods running simultaneously at peak. This enables zero-downtime updates since new pods are started before old ones are removed.

## What is maxUnavailable?

`maxUnavailable: 0` means Kubernetes will never terminate an existing pod unless a replacement is confirmed ready. This guarantees that the full capacity (3 replicas) is always serving traffic during the update.

## Why choose maxUnavailable: 0?

This is the safest production setting. It prioritises availability over speed. The trade-off is that the rollout needs slightly more resources temporarily (the surge pod), but guarantees no capacity loss at any point during the update.
