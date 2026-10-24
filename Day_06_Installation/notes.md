# Day 3: Creating Kubernetes Clusters Using KIND

## Objective

In this task, I created local Kubernetes clusters using KIND (Kubernetes IN Docker). I first created a single-node cluster using Kubernetes v1.29, deleted it, and then created a multi-node cluster named `cka-cluster2` using Kubernetes v1.30.

The final cluster consists of one control-plane node and three worker nodes.

## 1. Prerequisites

- Docker Desktop installed and running.
- KIND installed.
- kubectl installed and accessible from the terminal.

Verify the installations:

```powershell
docker --version
kind version
kubectl version --client
```

## 2. Create a Single-Node Cluster Using Kubernetes v1.29

Create the cluster:

```powershell
kind create cluster --name cka-cluster1 --image kindest/node:v1.29.0
```

**Explanation:**
- `kind create cluster` creates a local Kubernetes cluster.
- `--name cka-cluster1` specifies the cluster name.
- `--image kindest/node:v1.29.0` selects a KIND node image containing Kubernetes v1.29.0.

Verify the cluster:

```powershell
kubectl cluster-info --context kind-cka-cluster1
kubectl get nodes
```

The default configuration creates a single control-plane node.

## 3. Delete the Single-Node Cluster

Delete the cluster after verifying it:

```powershell
kind delete cluster --name cka-cluster1
```
This deletes the specified KIND cluster and its node containers.

Verify the remaining KIND clusters:
```powershell
kind get clusters
```

## 4. Create a Multi-Node Cluster Configuration

Create a file named `kind-config.yaml` in your working directory with the following contents:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cka-cluster2
nodes:
  - role: control-plane
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
```

**Explanation:**
- `kind: Cluster` specifies a KIND cluster configuration.
- `apiVersion` specifies the configuration schema.
- `name: cka-cluster2` sets the cluster name.
- `role: control-plane` defines the control-plane node.
- `role: worker` defines each worker node.
- `image: kindest/node:v1.30.0` selects Kubernetes v1.30.0 for every node.

## 5. Create the Multi-Node Cluster

Run:

```powershell
kind create cluster --config kind-config.yaml
```

## 6. Set the Current Context

Set the newly created cluster as the current kubectl context:

```powershell
kubectl config use-context kind-cka-cluster2
```

Verify the current context:

```powershell
kubectl config current-context
```

Expected context:
```text
kind-cka-cluster2
```

## 7. Verify the Nodes

Run:
```powershell
kubectl get nodes
```

Check the node status. All four nodes should eventually show `Ready`.

For more details:

```powershell
kubectl get nodes -o wide               //Extended details of the node
kubectl cluster-info
```

## 8. Verify the Docker Containers

KIND runs Kubernetes nodes inside Docker containers.

Run:
```powershell
docker ps
```

You should see four KIND node containers corresponding to the control-plane node and the three worker nodes.

To display only the KIND containers:

```powershell
docker ps --filter "name=cka-cluster2"
```

## Key Learnings

- KIND allows Kubernetes clusters to run locally using Docker containers as nodes.
- A single-node cluster is useful for basic learning and testing.
- A multi-node cluster helps practise scheduling and workloads across worker nodes.
- `kubectl` communicates with the cluster through its configured context.
- The `Ready` status indicates that Kubernetes considers a node healthy and ready to accept workloads, subject to its scheduling conditions.