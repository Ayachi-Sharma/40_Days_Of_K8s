# Day 29 – Kubernetes Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## 1. Objective

The objective of this task is to understand how Kubernetes manages persistent storage using PersistentVolumes (PV) and PersistentVolumeClaims (PVC).

We will:
1. Create a PersistentVolume named `pv-demo`.
2. Create a PersistentVolumeClaim named `pvc-demo`.
3. Bind the PVC to the available PersistentVolume.
4. Mount the PVC inside an Nginx Pod.
5. Create a file inside the mounted directory to verify storage.

## 2. Understanding PersistentVolume (PV)

A PersistentVolume (PV) is a storage resource in a Kubernetes cluster that is provisioned by an administrator or dynamically through a StorageClass.

**Key points:**
- PV represents storage available in the cluster.
- It has an independent lifecycle from individual Pods.
- It defines storage capacity, access modes, and the underlying storage type.
- It can be claimed by a PersistentVolumeClaim.

### Create PersistentVolume

Create a file named `pv.yaml`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 512Mi
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  hostPath:
    path: /data/config
    type: DirectoryOrCreate
```

Apply the configuration:

```bash
kubectl apply -f pv.yaml
```

Verify the PersistentVolume:

```bash
kubectl get pv
```

```bash
kubectl describe pv pv-demo
```

**Explanation:**
- `capacity`: Defines the total storage capacity as 512Mi.
- `accessModes`: ReadWriteMany allows the volume to be mounted as read-write by multiple nodes, subject to the capabilities of the storage backend.
- `storageClassName: ""`: Specifies that the PV does not belong to a StorageClass.
- `hostPath`: Uses a directory from the node's local filesystem.
- `DirectoryOrCreate`: Creates the directory if it does not already exist.

## 3. Understanding PersistentVolumeClaim (PVC)

A PersistentVolumeClaim (PVC) is a request for storage made by a user or application.

**Key points:**
- PVC requests storage from an available PV.
- It specifies the required storage capacity and access mode.
- Kubernetes matches a PVC with a suitable PV.
- Once successfully matched, the PVC gets the `Bound` status.

### Create PersistentVolumeClaim

Create a file named `pvc.yaml`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 256Mi
```

Apply the configuration:

```bash
kubectl apply -f pvc.yaml
```

Verify the PVC:

```bash
kubectl get pvc
```

Check the PV:

```bash
kubectl get pv
```

**Expected result:**

- PV `pv-demo` should show `Bound`.
- PVC `pvc-demo` should show `Bound`.

### Why does the PVC get Bound?

The PV provides 512Mi of storage, while the PVC requests only 256Mi.

Since the PV satisfies the PVC's storage requirement, access mode, and storage class, Kubernetes can bind the PVC to the PV.

**Important:** Setting `storageClassName: ""` prevents Kubernetes from selecting a dynamically provisioned StorageClass for this claim.

## 4. Create a Pod and Mount the PVC

Now we will create a Pod named `app` using the `nginx:latest` image and mount the PVC inside it.

Create a file named `pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: config-volume
          mountPath: /var/app/config
  volumes:
    - name: config-volume
      persistentVolumeClaim:
        claimName: pvc-demo
```

Apply the configuration:

```bash
kubectl apply -f pod.yaml
```

Check the Pod status:

```bash
kubectl get pods
```

Verify the mounted volume:

```bash
kubectl describe pod app
```

The PVC `pvc-demo` is mounted inside the container at:

```text
/var/app/config
```

## 5. Access the Pod and Create a File

Open an interactive shell inside the Pod:

```bash
kubectl exec -it app -- /bin/bash
```

Navigate to the mounted directory:

```bash
cd /var/app/config
```

Create a file:

```bash
touch demo.txt
```

Add some content:

```bash
echo "Kubernetes Persistent Volume Test" > demo.txt
```

Verify the file:

```bash
ls -l
cat demo.txt
```

Expected output:

```text
Kubernetes Persistent Volume Test
```

Exit the container:

```bash
exit
```

## 6. Verify Persistent Storage

Check whether the file exists inside the Pod:

```bash
kubectl exec app -- ls -l /var/app/config
```

The file `demo.txt` should be present in the mounted directory.

Since the directory is backed by a PersistentVolume, the data is stored outside the container's writable layer.

## 7. Important Commands

```bash
# Create PV
kubectl apply -f pv.yaml

# Create PVC
kubectl apply -f pvc.yaml

# Check PersistentVolumes
kubectl get pv

# Check PersistentVolumeClaims
kubectl get pvc

# Describe PV
kubectl describe pv pv-demo

# Describe PVC
kubectl describe pvc pvc-demo

# Create Pod
kubectl apply -f pod.yaml

# Check Pods
kubectl get pods

# Access Pod shell
kubectl exec -it app -- /bin/bash

# Verify mounted directory
kubectl exec app -- ls -l /var/app/config
```

## 8. Difference Between PV and PVC

| PersistentVolume (PV)                       | PersistentVolumeClaim (PVC)           |
|---------------------------------------------|---------------------------------------|
| Represents actual storage resources.        | Represents a request for storage.     |
| Created by an administrator or provisioner. | Created by a user or application.     |
| Defines capacity and access modes.          | Requests capacity and access modes.   |
| Exists independently of a Pod.              | Gets bound to a suitable PV.          |  

## 9. Conclusion

In this task, we successfully learned how to configure persistent storage in Kubernetes using PersistentVolumes and PersistentVolumeClaims.

We created a 512Mi PersistentVolume, requested 256Mi through a PVC, mounted the claim inside an Nginx Pod, and created a file in the mounted directory.

**Key takeaways:**
- PV represents the actual storage resource.
- PVC requests storage from the cluster.
- Kubernetes binds a PVC to a suitable PV.
- Pods consume persistent storage through PVCs.
- Data stored in a PV can survive Pod deletion.