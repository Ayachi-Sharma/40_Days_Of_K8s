# Kubernetes – Cluster Troubleshooting and crictl

## 1. Introduction

In this task, we learn how to troubleshoot Kubernetes clusters and inspect containers directly through the container runtime.

Official documentation:
- https://kubernetes.io/docs/tasks/debug/debug-cluster/
- https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/

### Objectives

1. Understand the Kubernetes cluster troubleshooting process.
2. Check the health of Nodes and Pods.
3. Troubleshoot control-plane and worker-node components.
4. Understand `crictl` and its role in Kubernetes.
5. Inspect Pods, containers, and images using `crictl`.
6. Read container logs without using `kubectl`.

---

## 2. Basic Cluster Troubleshooting

First, verify that the Kubernetes cluster is accessible.

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

Check Node status:

```bash
kubectl get nodes -o wide
```

A healthy Node should normally show:

```text
STATUS: Ready
```

### Check Node Details

```bash
kubectl describe node <node-name>
```

Important sections:
- Conditions: Shows whether the Node is Ready and under resource pressure.
- Capacity: Total CPU and memory resources.
- Allocatable: Resources available for Kubernetes workloads.
- Events: Recent warnings or problems affecting the Node.

Common Node conditions:

| Condition | Meaning |
|---|---|
| Ready | Node is healthy and can accept workloads |
| MemoryPressure | Node has insufficient memory |
| DiskPressure | Node has insufficient disk space |
| PIDPressure | Node has insufficient available process IDs |
| NetworkUnavailable | Node networking is not correctly configured |

### Check Cluster Events

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Events help identify scheduling failures, image pull errors, volume issues, and other Kubernetes problems.

---

## 3. Troubleshooting Pods

Check Pods across all namespaces:

```bash
kubectl get pods -A
```

Describe a problematic Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Check application logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

For a Pod with multiple containers:

```bash
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

For a previously terminated container:

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

### Common Pod Problems

| Status | Possible reason |
|---|---|
| Pending | Scheduling or resource problem |
| ImagePullBackOff | Image cannot be pulled |
| CrashLoopBackOff | Container repeatedly crashes |
| CreateContainerConfigError | Invalid container configuration |
| ContainerCreating | Container setup is still in progress |

Always inspect the Pod's Events and logs before changing its configuration.

---

## 4. Troubleshooting Control-Plane Components

In a kubeadm-based cluster, control-plane components commonly run as static Pods in the `kube-system` namespace.

Components include:

- kube-apiserver
- kube-scheduler
- kube-controller-manager
- etcd

Check their status:

```bash
kubectl get pods -n kube-system -o wide
```

Inspect a component:

```bash
kubectl describe pod <pod-name> -n kube-system
```

Check its logs:

```bash
kubectl logs <pod-name> -n kube-system
```

If the API server is unavailable, `kubectl` may not work. In that situation, connect directly to the control-plane node and inspect the container runtime and system logs.

---

## 5. Troubleshooting Worker Nodes

Worker nodes run the kubelet, container runtime, and networking components required to run Pods.

Connect to a worker node:

```bash
ssh username@worker-node-ip
```

Check kubelet status:

```bash
sudo systemctl status kubelet
```

Read kubelet logs:

```bash
sudo journalctl -u kubelet -n 50
```

Follow logs continuously:

```bash
sudo journalctl -u kubelet -f
```

Check container runtime status (containerd example):

```bash
sudo systemctl status containerd
```

Read containerd logs:

```bash
sudo journalctl -u containerd -n 50
```

---

## 6. What is crictl?

`crictl` is a command-line tool used to interact with container runtimes that implement the Kubernetes Container Runtime Interface (CRI).

It is useful for troubleshooting containers directly on Kubernetes nodes.

With `crictl`, we can:

- List Pods managed by the runtime.
- List running and stopped containers.
- Inspect containers.
- View container logs.
- Execute commands inside running containers.
- List container images.

Unlike `kubectl`, `crictl` communicates directly with the container runtime.

**Important:** Run `crictl` on the node where the relevant container is running.

---

## 7. Configure crictl

Check whether crictl is installed:

```bash
crictl --version
```

If it is installed but cannot connect to the runtime, configure its endpoint.

For containerd, create or edit:

```bash
sudo vi /etc/crictl.yaml
```

Add:

```yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
```

Save the file and test:

```bash
sudo crictl info
```

The endpoint depends on the container runtime. Do not assume every Kubernetes cluster uses containerd.

---

## 8. Useful crictl Commands

### List Pods

```bash
sudo crictl pods
```

List all Pods managed by the container runtime, including their state and namespace.

### List Running Containers

```bash
sudo crictl ps
```

### List All Containers

```bash
sudo crictl ps -a
```

This includes stopped containers.

### Inspect a Container

```bash
sudo crictl inspect <container-id>
```

Displays detailed container configuration and runtime information.

### View Container Logs

```bash
sudo crictl logs <container-id>
```

Display only the latest 50 lines:

```bash
sudo crictl logs --tail=50 <container-id>
```

### Execute a Command Inside a Container

```bash
sudo crictl exec -it <container-id> sh
```

### List Images

```bash
sudo crictl images
```

List containers and filter by name:

```bash
sudo crictl ps -a | grep kube-apiserver
```

---

## 9. Practical: Inspect kube-apiserver Without kubectl

On a kubeadm control-plane node, execute:

### Step 1: List running containers

```bash
sudo crictl ps
```

### Step 2: Find kube-apiserver

```bash
sudo crictl ps | grep kube-apiserver
```

Copy its container ID.

### Step 3: Inspect the container

```bash
sudo crictl inspect <container-id>
```

### Step 4: Read its logs

```bash
sudo crictl logs <container-id>
```

This allows us to inspect kube-apiserver logs directly through the container runtime, even when the Kubernetes API is unavailable.

---

## 10. Difference Between kubectl and crictl

| kubectl | crictl |
|---|---|
| Communicates with Kubernetes API Server | Communicates with container runtime |
| Manages Kubernetes resources | Inspects runtime-level resources |
| Works with Pods, Deployments, Services, etc. | Works with runtime Pods, containers, and images |
| Requires API access | Does not require Kubernetes API access |

Example:

```bash
kubectl get pods -A
```

Lists Pods through the Kubernetes API.

```bash
sudo crictl ps -a
```

Lists containers directly through the container runtime.

---

## 11. Quick Troubleshooting Reference

| Problem | Command |
|---|---|
| Check Nodes | `kubectl get nodes` |
| Inspect Node | `kubectl describe node <node>` |
| Check all Pods | `kubectl get pods -A` |
| Inspect Pod | `kubectl describe pod <pod> -n <namespace>` |
| Read Pod logs | `kubectl logs <pod> -n <namespace>` |
| Check events | `kubectl get events -A` |
| Check kubelet | `sudo systemctl status kubelet` |
| Read kubelet logs | `sudo journalctl -u kubelet` |
| Check containerd | `sudo systemctl status containerd` |
| List runtime Pods | `sudo crictl pods` |
| List containers | `sudo crictl ps -a` |
| Inspect container | `sudo crictl inspect <id>` |
| Read container logs | `sudo crictl logs <id>` |
| List images | `sudo crictl images` |

---

## 12. Conclusion

In this task, we learned how to troubleshoot Kubernetes at both the cluster level and container-runtime level.

Key learnings:

1. Use `kubectl` to inspect Nodes, Pods, and Kubernetes events.
2. Use `journalctl` to investigate systemd service logs.
3. Use `crictl` to communicate directly with the container runtime.
4. Inspect containers and read logs without depending on the Kubernetes API Server.
5. Use direct node-level troubleshooting when normal Kubernetes API access is unavailable.