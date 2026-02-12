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

Because it's secret : to not expose api keys

## Is a Secret encrypted by default? Where?

No it's encoded in base64
