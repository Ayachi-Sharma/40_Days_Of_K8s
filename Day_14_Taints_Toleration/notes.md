# Day 14: Kubernetes Taints and Tolerations

## 1. Apply Taints to Both Worker Nodes

### Step 1: Verify the nodes
```bash
kubectl get nodes
```
Identify the names of both worker nodes and the control plane node.

### Step 2: Taint worker01
```bash
kubectl taint nodes worker01 gpu=true:NoSchedule
```

### Step 3: Taint worker02
```bash
kubectl taint nodes worker02 gpu=false:NoSchedule
```

**Explanation:**
- `gpu` is the taint key.
- `true` and `false` are the taint values.
- `NoSchedule` prevents new Pods without matching tolerations from being scheduled on these nodes.

Verify the taints:
```bash
kubectl describe node worker01
kubectl describe node worker02
```
Look for the `Taints` field in the output.

---

## 2. Create an Nginx Pod Without a Toleration

### Step 1: Create the Pod
```bash
kubectl run nginx --image=nginx
```

### Step 2: Check the Pod status
```bash
kubectl get pods -o wide
kubectl describe pod nginx
```

### Observation
The Pod remains in the `Pending` state if all available nodes reject it.
Check the Events section in `kubectl describe pod nginx` to see the scheduling failure reason.

---

## 3. Add a Toleration for Worker01

### Step 1: Delete the original Pod
```bash
kubectl delete pod nginx
```

### Step 2: Create a Pod manifest
Create a file named `nginx-toleration.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

### Step 3: Create the Pod
```bash
kubectl apply -f nginx-toleration.yaml
```

### Step 4: Verify the Pod
```bash
kubectl get pod nginx -o wide
kubectl describe pod nginx
```

Nginx pod must be running on the node which has taint gpu=true.
---

## 4. Remove the Taint From the Control Plane Node

### Step 1: Identify the control plane node
```bash
kubectl get nodes
```

Use the actual control plane node name in the following command.

### Step 2: Remove its default taint

```bash
kubectl taint nodes <control-plane-node> node-role.kubernetes.io/control-plane:NoSchedule-
```

Replace `<control-plane-node>` with the real node name.
Taints are taken fom the describe command

### Step 3: Verify the change

```bash
kubectl describe node <control-plane-node>
```
Check the `Taints` field to confirm the control plane taint is gone.

**Note:** This removes the default control plane taint only. Any other taints on the node remain.

---

## 5. Create a Redis Pod

### Step 1: Create the Redis Pod
```bash
kubectl run redis --image=redis
```

### Step 2: Check its status and node
```bash
kubectl get pods -o wide
kubectl describe pod redis
```

### Explanation
After removing the control plane taint, the control plane node becomes eligible to accept ordinary Pods.

However, Kubernetes does not guarantee that Redis will be placed there. Since worker01 and worker02 also tolerate no matching Redis tolerations, the scheduler can place Redis on the control plane only if the remaining scheduling constraints and node conditions make it the suitable choice.

---

## 6. Add the Control Plane Taint Back

```bash
kubectl taint nodes <control-plane-node> node-role.kubernetes.io/control-plane:NoSchedule
```

Verify:
```bash
kubectl describe node <control-plane-node>
kubectl get nodes
```

The default control plane taint should appear again.

**Important:** The `NoSchedule` effect prevents new Pods from being scheduled on that node if they do not tolerate the taint. It does not automatically evict Pods already running there.

---

## Key Learnings

1. Taints repel Pods that do not have matching tolerations.
2. Tolerations allow Pods to use tainted nodes but do not guarantee that those nodes will be selected.
3. `NoSchedule` affects new scheduling decisions without evicting existing Pods.
4. `kubectl describe pod` helps diagnose scheduling failures through the Events section.
5. `nodeSelector` can be combined with tolerations to target a specific node.
6. The control plane taint can be removed to allow normal workloads and restored afterward.