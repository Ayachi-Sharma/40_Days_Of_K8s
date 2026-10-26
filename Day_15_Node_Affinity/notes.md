# Day 15: Kubernetes Node Affinity

## 1. Create an Nginx Pod With Required Node Affinity

### Objective
Create a Pod that can be scheduled only on a node with the label `disktype=ssd`, using `requiredDuringSchedulingIgnoredDuringExecution`.

### Step 1: Create the Pod manifest

Create a file named `pod1.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: nginx
      image: nginx
```

### Step 2: Create the Pod

```bash
kubectl apply -f pod1.yaml
```

### Step 3: Check the Pod status

```bash
kubectl get pods -o wide
kubectl describe pod pod1
```

### Observation

The Pod remains in the `Pending` state if no eligible node has the label `disktype=ssd`.

Check the Events section of `kubectl describe pod pod1` for a scheduling message indicating that the node affinity requirements are not satisfied.

---

## 2. Add the Required Label to Worker01

### Step 1: Label worker01

```bash
kubectl label nodes worker01 disktype=ssd
```

### Step 2: Verify the label
```bash
kubectl get nodes --show-labels
```

Alternatively:

```bash
kubectl describe node worker01
```

### Step 3: Check the Pod again

```bash
kubectl get pods -o wide
```

### Expected Result
The `pod1` Pod should be scheduled on `worker01`, provided it is healthy and has sufficient resources.

---

## 3. Create a Redis Pod Using Preferred Node Affinity

### Step 1: Create the Pod manifest

Create a file named `pod2.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - hdd
  containers:
    - name: redis
      image: redis
```

### Step 2: Add the `disktype` label to worker02

```powershell
kubectl label nodes lea-cluster-worker2 disktype=""
```

This adds the `disktype` label with an empty value.

Verify the label:
```powershell
kubectl get nodes --show-labels
```

### Step 4: Create the Pod and verify placement

```powershell
kubectl apply -f pod2.yaml
kubectl get pod pod2 -o wide
```

If `pod2` already exists, apply the updated manifest after creating the file. If Kubernetes rejects the update because the Pod's scheduling fields are immutable, delete and recreate it:

```powershell
kubectl delete pod pod2
kubectl apply -f pod2.yaml
```

### Explanation

- `preferredDuringSchedulingIgnoredDuringExecution` expresses a scheduling preference rather than a mandatory condition.
- `weight` determines the relative importance of the preference, from 1 to 100.
- `operator: In` requires the label value to match `hdd` for the preference to match.
- `disktype=""` creates a label with an empty value, which does not match `disktype=hdd`.
- `nodeSelector` ensures that the Pod is scheduled on `lea-cluster-worker2`, provided the node is healthy and can accommodate the Pod.

**Key learning:** Preferred node affinity influences node selection but does not guarantee placement. Use a `nodeSelector` alongside preferred affinity when you must target a specific node.

1. Node affinity controls Pod placement using node labels.
2. `requiredDuringSchedulingIgnoredDuringExecution` defines a mandatory rule for scheduling.
3. `operator: In` matches a label key against one or more specified values.
4. `operator: Exists` checks whether a label key exists, regardless of its value.
5. `kubectl label nodes` adds labels to Kubernetes nodes.
6. Node affinity makes nodes eligible or ineligible; it does not necessarily guarantee placement on one specific node.
7. Use a `nodeSelector` or a more specific affinity rule when a Pod must run on a particular node.