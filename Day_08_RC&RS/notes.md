# ReplicaSet — Create 3 Nginx Replicas

## Objective
Create a Kubernetes ReplicaSet that maintains three Pods using the Nginx container image.

## What is a ReplicaSet?
A ReplicaSet is a Kubernetes workload resource that maintains a specified number of identical Pod replicas. If a managed Pod is deleted or fails, the ReplicaSet attempts to create another Pod to maintain the desired count.

## YAML Configuration

File: `nginx-pod.yaml`

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

## Commands Used

Create or update the ReplicaSet:

```powershell
kubectl apply -f replicaset.yaml
```

List ReplicaSets:

```powershell
kubectl get replicasets
```

List Pods:

```powershell
kubectl get pods
```

## Expected Result
The ReplicaSet should show three desired replicas, three current replicas, and eventually three ready replicas. The Pods should reach the `Running` status.


### Now change the replica number from yaml file
replace it with 4 and verfy again

Apply the updated configuration
```powershell
kubectl apply -f replicaset.yaml
```

```powershell
```
Expected output:
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicaset   4         4         4       2m5
kubectl get replicasets5s

### Now change the replica number from Command line
```powershell
kubectl scale replicaset nginx-replicaset --replicas=6
```

Verify:
```poweshell
kubectl get replicasets
```

# Deployment

## Create an Nginx Deployment

Create a Deployment named `nginx` with three replicas using image `nginx:1.23.0`.

### Command

```powershell
kubectl create deployment nginx --image=nginx:1.23.0 --replicas=3
```

### Explanation
- `kubectl create deployment`: Creates a Kubernetes Deployment.
- `nginx`: Name of the Deployment.
- `--image=nginx:1.23.0`: Specifies the container image and version.
- `--replicas=3`: Requests three Pod replicas.

### Note
The task also requires the Deployment label `tier=backend` and Pod template label `app=v1`. These will be configured in the next step using the Deployment YAML.

### Verification

```powershell
kubectl get deployments
kubectl get pods
```
### Update the Nginx Image

**Command:**

```powershell
kubectl set image deployment/nginx nginx=nginx:1.23.4
```

**Explanation:**
- `deployment/nginx` identifies the Deployment.
- The first `nginx` after the space is the container name.
- `nginx:1.23.4` is the new container image and version.

This updates the Deployment's Pod template. Kubernetes creates a new ReplicaSet and performs a rolling update to replace the old Pods with Pods using the new image.

**Verification:** We'll check the rollout status and confirm the image in the next step.

### Update different things in a Deployment:

`Reason`              `Commands`
Container image	      kubectl set image deployment/nginx nginx=nginx:1.23.4
Number of replicas	  kubectl scale deployment/nginx --replicas=5
Environment variable	kubectl set env deployment/nginx KEY=value
Container resources	  kubectl edit deployment/nginx
Labels	              kubectl label deployment nginx tier=backend

### Check the change has been applied to all replicas 

```powershell
kubectl get deployment nginx -o wide
kubectl describe deploy/nginx
```

## Assign a Change Cause to a Deployment Revision

### Step 1: Add the Change Cause

Run the following command:
```powershell
kubectl annotate deployment nginx kubernetes.io/change-cause="Pick up patch version"
```

**Explanation:**
- `kubectl annotate`: Adds or updates an annotation on a Kubernetes resource.
- `deployment nginx`: Specifies the Deployment to annotate.
- `kubernetes.io/change-cause`: The annotation key used to record the reason for a Deployment change.
- `Pick up patch version`: Describes why the image was updated from `nginx:1.23.0` to `nginx:1.23.4`.

### Step 2: Verify the Rollout History

```powershell
kubectl rollout history deployment/nginx
```

Expected output:
2         Pick up patch version

## Scale the Deployment to 5 Replicas

```powershell
kubectl scale deploy/nginx --replicas=5
```

### Explanation
- `kubectl scale`: Changes the desired replica count.
- `deploy/nginx`: Specifies the Deployment to scale.
- `--replicas=5`: Sets the desired number of Pods to five.

### Verification

```powershell
kubectl get deployment nginx
kubectl get pods
```

### Expected Result
The Deployment should show five desired and ready replicas once scaling completes.

### Key Learning
Scaling a Deployment adjusts the number of Pods managed by its ReplicaSet. Kubernetes creates or removes Pods to match the desired replica count.


## Roll Back the Deployment to Revision 1

### Step 1: Check the Rollout History

```powershell
kubectl rollout history deployment/nginx
```

Output:

```text
REVISION  CHANGE-CAUSE
1         <none>
2         Pick up patch version
```

**Explanation:**
- Revision 1 represents the original Deployment configuration.
- Revision 2 represents the updated configuration using `nginx:1.23.4`.
- The change cause helps identify why the new revision was created.

### Step 2: Roll Back to Revision 1

```powershell
kubectl rollout undo deployment/nginx --to-revision=1
```

**Explanation:**
- `kubectl rollout undo` rolls back a Deployment to a previous revision.
- `deploy/nginx` specifies the Deployment.
- `--to-revision=1` selects Revision 1 as the rollback target.

Kubernetes restores the Pod template from Revision 1 and creates a rollout to replace the Pods with the previous configuration.

### Step 3: Verify the Rollout

```powershell
kubectl rollout status deployment/nginx
```
Expected output: 
REVISION  CHANGE-CAUSE
2         Pick up patch version
3         <none>

### Step 4: Verify the Image Version

```powershell
kubectl get deployment nginx -o wide
```

### Step 5: Verify the Replica Count

```powershell
kubectl get deployment nginx
```

The Deployment should still have *five replicas* because the replica count was changed independently of the Pod template revision.

### Key Learning
A Deployment rollback restores a previous Pod template configuration. Scaling is independent of the Pod template, so rolling back the image does not automatically revert the replica count.

Also, a rollback creates a new revision rather than deleting the revision history. The rollout history records the sequence of Deployment configurations.


# Kubernetes Troubleshooting: Deployment API Version Error
## 1. Original YAML
The manifest contained the following configuration:
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    env: demo
```

The full manifest also defined three replicas, a Pod template, an Nginx container, and a matching selector.

## 2. Error Encountered
```powershell
kubectl apply -f .\deployment.yaml
```
Error:
```text
error: resource mapping not found for name: "nginx-deploy" namespace: "" from ".\\deployment.yaml": no matches for kind "Deployment" in version "v1"
ensure CRDs are installed first
```

## 3. Root Cause

The manifest used the incorrect API version for a Deployment:
```yaml
apiVersion: v1
```
The `v1` API group is used for core resources such as Pods and Services. A Kubernetes Deployment belongs to the `apps` API group and must use:
```yaml
apiVersion: apps/v1
```

## 4. Corrected YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    env: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      env: demo
  template:
    metadata:
      labels:
        env: demo
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

### Corrections made

1. Changed `apiVersion: v1` to `apiVersion: apps/v1`.
2. Removed `name: nginx` out of `spec.template.metadata` because the Pod template's `metadata` describes the Pod's labels and annotations not name. 
3. Ensured `spec.selector.matchLabels.env` matches `spec.template.metadata.labels.env`.

## 5. Apply the Corrected Manifest

```powershell
kubectl apply -f .\deployment.yaml
```

Verify the Deployment:
```powershell
kubectl get deployments
kubectl get pods
```

Expected result: the Deployment should have three desired replicas, and the Pods should eventually reach `Running` and `Ready` status.

## 6. Troubleshooting Commands

| Apply the YAML | `kubectl apply -f .\deployment.yaml` |
| Validate on the server without saving changes | `kubectl apply --dry-run=server -f .\deployment.yaml` |
| List Deployments | `kubectl get deployments` |
| Inspect a Deployment | `kubectl describe deployment nginx-deploy` |
| List Pods | `kubectl get pods` |
| Inspect a Pod | `kubectl describe pod <pod-name>` |
| View recent namespace events | `kubectl get events --sort-by=.metadata.creationTimestamp` |

## Key Learnings

- Always use the API version supported by the resource kind.
- A Deployment uses `apps/v1`; a Pod uses `v1`.
- YAML indentation determines the structure of Kubernetes resources.
- A Deployment's selector must match the labels in its Pod template.
- `kubectl describe` inspects existing resources; it does not diagnose a manifest that failed before resource creation.
- Read the exact error message before deciding whether the problem is syntax, schema, API mapping, image pulling, scheduling, or container startup.
