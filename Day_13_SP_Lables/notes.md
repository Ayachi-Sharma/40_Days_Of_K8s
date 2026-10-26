# Day 12: Kubernetes Scheduling, Static Pods, and Labels

## 1. Create a Pod and Schedule It Manually Without the Scheduler

### Step 1: Create a Pod manifest
Create a file named `manual-pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: svc-cluster-worker
  containers:
    - name: nginx
      image: nginx
```
**Note:** Replace `svc-cluster-worker` with an actual node name obtained using `kubectl get nodes`.

### Step 2: Create the Pod
```bash
kubectl apply -f manual-pod.yaml
```

### Step 3: Verify the Pod
```bash
kubectl get pods -o wide
kubectl describe pod manual-pod
```

### Explanation
- Normally, the scheduler selects a suitable node for a newly created Pod.
- The `spec.nodeName` field directly specifies the node where the Pod should run.
- When `nodeName` is set, the scheduler does not select a node for that Pod.
- The kubelet on the specified node attempts to start the Pod's containers.

**Important:** The specified node must exist and be able to run the Pod. Setting `nodeName` bypasses normal scheduler placement, not the kubelet's responsibilities.

---

## 2. Restart Control Plane Components Using Static Pod Manifests

### Step 1: Access the control plane node

For a `kind` cluster, identify the control plane container:
```bash
docker ps
```

Enter the control plane container:
```bash
docker exec -it svc-cluster-control-plane bash
```

### Step 2: Navigate to the static Pod manifest directory

```bash
cd /etc/kubernetes/manifests
ls
```

Typical manifest files include:
- `etcd.yaml`
- `kube-apiserver.yaml`
- `kube-controller-manager.yaml`
- `kube-scheduler.yaml`

These files define the static Pods used to run the control plane components.

### Step 3: Understand how restarting works

Kubelet monitors the static Pod manifests and ensures that the corresponding containers are running.

To trigger a restart, modify a harmless manifest property, such as adding a temporary annotation under the Pod's metadata, then save the file. Kubelet detects the manifest change and reconciles the static Pod.

Alternatively, if the environment and runtime permit it, stop the relevant container through the container runtime and let kubelet recreate it.

Verify the control plane Pods from your terminal:

```bash
kubectl get pods -n kube-system -o wide
```

### Explanation

- Static Pods are managed directly by kubelet rather than scheduled by the Kubernetes scheduler.
- Kubelet reads the static Pod manifest files from the configured directory.
- When a static Pod's configuration changes or its container stops, kubelet attempts to reconcile the desired state.
- In a kubeadm cluster, the default static Pod manifest directory is `/etc/kubernetes/manifests`.

**Important safety note:** Do not delete, rename, or modify control plane manifests casually. Restarting `kube-apiserver`, `etcd`, or other control plane components can disrupt the cluster. Back up any file before experimenting, and avoid restarting `etcd` unless you understand the recovery implications.

For `kind`, control plane components and kubelet run inside the node container, so this exercise differs from working on a dedicated Linux control plane VM.

---

## 3. Create Three Nginx Pods With Different Labels

### Objective
Create three Pods named `pod1`, `pod2`, and `pod3`, with different environment labels.

### Step 1: Create the Pod manifest

Create a file named `pods-labels.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    env: test
spec:
  containers:
    - name: nginx
      image: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    env: dev
spec:
  containers:
    - name: nginx
      image: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: pod3
  labels:
    env: prod
spec:
  containers:
    - name: nginx
      image: nginx
```

### Step 2: Apply the manifest

```bash
kubectl apply -f pods-labels.yaml
```

### Step 3: Verify the Pods and labels

```bash
kubectl get pods
kubectl get pods --show-labels
```

Expected labels:

| Pod | Label |
|---|---|
| `pod1` | `env=test` |
| `pod2` | `env=dev` |
| `pod3` | `env=prod` |

### Explanation

- Labels are key-value pairs attached to Kubernetes objects.
- The `env` key identifies the environment associated with each Pod.
- `---` separates multiple Kubernetes resource definitions in one YAML file.
- Labels help organize and select resources without relying on Pod names.

---

## 4. Filter Pods Using Labels

### Objective
Use `kubectl` label selectors to retrieve only Pods belonging to the development and production environments.

### Step 1: Filter development Pods

```bash
kubectl get pods -l env=dev
```

Expected result: `pod2`.

### Step 2: Filter production Pods

```bash
kubectl get pods -l env=prod
```

Expected result: `pod3`.

### Step 3: Filter both development and production Pods

```bash
kubectl get pods -l 'env in (dev,prod)'
```
Expected results: `pod2` and `pod3`.

### Step 4: Display additional information

```bash
kubectl get pods -l 'env in (dev,prod)' --show-labels -o wide
```

### Explanation

- `-l` or `--selector` specifies a label selector.
- `env=dev` selects Pods whose `env` label equals `dev`.
- `env=prod` selects Pods whose `env` label equals `prod`.
- `env in (dev,prod)` selects Pods whose `env` value is either `dev` or `prod`.

## Key Learnings

1. The scheduler assigns nodes to Pods unless a node is specified explicitly through `spec.nodeName`.
2. Kubelet manages static Pods and their containers independently of scheduler placement.
3. Static Pod manifests are commonly stored in `/etc/kubernetes/manifests` on kubeadm-based clusters.
4. Labels organize Kubernetes resources using key-value pairs.
5. Label selectors filter resources by their labels using `kubectl get -l`.
