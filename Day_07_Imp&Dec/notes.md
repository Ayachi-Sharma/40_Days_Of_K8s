# Day 3 — Task 1: Create a Pod Imperatively Using Nginx

## 1. What is Imperative?

**Imperative approach** means telling Kubernetes *what command to execute* to perform an action.

We directly run commands using `kubectl` to create, update, or delete Kubernetes resources.

Example:
```powershell
kubectl run nginx --image=nginx
```
**Explanation:**
- `kubectl`: Kubernetes command-line tool.
- `run`: Creates a Pod using the specified image.
- `nginx`: Name of the Pod.
- `--image=nginx`: Specifies the container image to use.

This command creates a Pod named `nginx` using the Nginx container image.

Now verify that the Pod was created:
```powershell
kubectl get pods
```
Example output:
```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s
```

Inspect the Pod
```powershell
kubectl describe pod nginx
```
This displays detailed information, including the Pod's image, status, events, and other configuration details.

## 2. What is Declarative?

**Declarative approach** means defining the desired state of a Kubernetes resource in a configuration file, usually YAML/JSON, and asking Kubernetes to make the actual state match it.

Example: Defining a Pod in a YAML file and applying it using:

Write in nginx-pod.yaml:
apiVersion: v1
kind: Pod
metadata:
  name: nginx-new
  labels:
    run: nginx
spec:
  containers:
    - name: nginx
      image: nginx


```powershell
kubectl apply -f nginx-pod.yaml
```
Kubernetes reads the configuration and works toward the desired state described in the file.

Verify both Pods
```powershell
kubectl get pods
```


### Find error in the yaml file 
Given :
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: test
  name: redis
spec:
  containers:
  - image: rediss
    name: redis


At first i created pod using this file, to help troubleshoot it :
```powershell
kubelet apply -f redis-pod.yaml
```

This is where i saw the error
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  104s                default-scheduler  Successfully assigned default/redis to lea-cluster-worker
  Normal   BackOff    28s (x4 over 102s)  kubelet            spec.containers{redis}: Back-off pulling image "rediss"
  Warning  Failed     28s (x4 over 102s)  kubelet            spec.containers{redis}: Error: ImagePullBackOff
  Normal   Pulling    13s (x4 over 103s)  kubelet            spec.containers{redis}: Pulling image "rediss"
  Warning  Failed     11s (x4 over 102s)  kubelet            spec.containers{redis}: Failed to pull image "rediss": failed to pull and unpack image "docker.io/library/rediss:latest": failed to resolve reference "docker.io/library/rediss:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     11s (x4 over 102s)  kubelet            spec.containers{redis}: Error: ErrImagePull


*Understood the image is not present in the docker hub*
Checked dockerhub and in yamk file 
Found that image name has an extra s in redis, fixed it
Removed this pod and recreated it again using same command


Verify the pod:
```powershell
kubectl get pods
```

Check discription of the pod
```powershell
kubectl describe pod
```