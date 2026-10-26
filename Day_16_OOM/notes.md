# Day 16: Kubernetes Memory Requests and Limits

## 1. Create a Namespace

```powershell
kubectl create namespace mem-example
```

### Verify the namespace

```powershell
kubectl get namespaces
```

**Explanation:** Namespaces provide logical isolation for Kubernetes resources. All Pods created in this exercise will belong to the `mem-example` namespace.

---

## 2. Install Metrics Server

### Step 1: Apply the Metrics Server manifest

```powershell
kubectl apply -f <metrics-server-yaml-file>
```

Replace `<metrics-server-yaml-file>` with the actual filename or path from your repository.

Alternatively, use the official Metrics Server manifest:
```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Step 2: Verify the deployment

```powershell
kubectl get deployments -n kube-system
kubectl get pods -n kube-system
```

Check whether the Metrics Server Pod is running.

### Step 3: Verify the Metrics API

```powershell
kubectl get apiservices
```

Look for `v1beta1.metrics.k8s.io` and check that its availability status is `True`.

### Step 4: Check node metrics

```powershell
kubectl top nodes
```

**Explanation:**

- Metrics Server collects resource usage metrics from kubelets.
- It makes CPU and memory usage available through the Kubernetes Metrics API.
- Commands such as `kubectl top nodes` and `kubectl top pods` depend on this API.

**Note:** In a local `kind` cluster, Metrics Server may require additional configuration if it cannot verify the kubelet's serving certificate. Only apply the configuration change recommended for your cluster if you encounter that issue.

---

## 3. Specify a Memory Request and Memory Limit

### Objective
Create a Pod with a memory request of 100 MiB and a memory limit of 200 MiB.

### Step 1: Create the Pod

Use the official Kubernetes example:

```powershell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

The manifest defines the following resource configuration:

```yaml
resources:
  requests:
    memory: "100Mi"
  limits:
    memory: "200Mi"
```

The container uses the `polinux/stress` image to attempt to allocate 150 MiB of memory.

### Step 2: Verify the Pod status

```powershell
kubectl get pods -n mem-example
```

Expected status: `Running`, provided the image starts successfully and the cluster has sufficient resources.

### Step 3: Inspect the Pod configuration

```powershell
kubectl get pod memory-demo -n mem-example -o yaml
```

Look for the `resources` section and verify that the request is `100Mi` and the limit is `200Mi`.

### Step 4: Check memory usage

```powershell
kubectl top pod memory-demo -n mem-example
```

The memory usage should be approximately 150 MiB, though the actual value can vary.

### Explanation

- **Memory request (`100Mi`):** Used by the scheduler when determining whether a node has sufficient allocatable memory for the Pod.
- **Memory limit (`200Mi`):** Sets the maximum memory the container is permitted to use.
- The container can use more memory than its request if resources are available, provided it stays within its limit.

---

## 4. Observe What Happens When a Container Exceeds Its Memory Limit

### Step 1: Create the second example

```powershell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example
```

This Pod requests `50Mi`, has a memory limit of `100Mi`, and attempts to allocate `250M` of memory.

### Step 2: Monitor its status

```powershell
kubectl get pod memory-demo-2 -n mem-example -w
```

Press `Ctrl+C` to stop watching.

### Step 3: Inspect the Pod

```powershell
kubectl describe pod memory-demo-2 -n mem-example
kubectl get pod memory-demo-2 -n mem-example -o yaml
```

Look for `OOMKilled` in the container's last termination state. The container may restart repeatedly because its restart policy allows it to restart.

### Explanation

- The container attempts to allocate more memory than its `100Mi` limit.
- The operating system's memory enforcement can terminate the container for exceeding that limit.
- Kubernetes reports the termination reason as `OOMKilled`.
- The kubelet may restart the container, causing the restart count to increase.

---

## 5. Observe What Happens When a Memory Request Is Too Large

### Objective
Understand how a memory request affects Pod scheduling.

### Step 1: Create the third example

```powershell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
```

This Pod requests `1000Gi` of memory, which is far beyond the capacity of a typical local Kubernetes node.

### Step 2: Check its status

```powershell
kubectl get pod memory-demo-3 -n mem-example
```

Expected status: `Pending`.

### Step 3: Inspect the scheduling events

```powershell
kubectl describe pod memory-demo-3 -n mem-example
```

Look for a `FailedScheduling` event reporting insufficient memory.

### Explanation

- The scheduler considers the Pod's memory request, not just its current memory consumption.
- If no node has enough allocatable memory to satisfy the request, the scheduler cannot place the Pod.
- The Pod remains `Pending` until a suitable node becomes available or the request is changed.

**Important:** A large memory limit alone does not necessarily prevent scheduling. The request is the value used for scheduling decisions.

---

## 6. Clean Up the Resources

### Step 1: Delete the example Pods

```powershell
kubectl delete pod memory-demo memory-demo-2 memory-demo-3 -n mem-example
```

### Step 2: Delete the namespace

After completing the exercise and confirming that you no longer need its resources:

```powershell
kubectl delete namespace mem-example
```

This removes the namespace and the resources inside it.

Metrics Server is installed separately in `kube-system`, so deleting `mem-example` does not uninstall it.

---

## Key Learnings

1. Namespaces isolate Kubernetes resources logically.
2. Metrics Server exposes CPU and memory usage metrics through the Metrics API.
3. Memory requests influence scheduling decisions.
4. Memory limits constrain the memory a container can consume.
5. Exceeding a container's memory limit can cause an `OOMKilled` termination.
6. An excessively large memory request can leave a Pod in the `Pending` state.
7. `kubectl top` displays current resource usage, while `kubectl describe` helps diagnose scheduling and container failures.