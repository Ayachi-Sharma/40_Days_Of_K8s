# Kubernetes ETCD Backup and Restore – Complete Documentation

## 1. What is ETCD?

ETCD is a distributed, consistent, highly available key-value store used by Kubernetes to store cluster data. It acts as the database of a Kubernetes cluster.

ETCD stores information such as:
- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Namespaces
- Nodes
- Roles and RoleBindings
- PersistentVolume and PersistentVolumeClaims
- Other Kubernetes resources and cluster configuration

### Why Do We Need ETCD?

Kubernetes uses ETCD to maintain the desired and current state of the cluster. Whenever we create, update, or delete a Kubernetes resource, the API Server stores the corresponding state in ETCD.

For example:
When we create a Deployment:
    kubectl create deployment nginx --image=nginx

The Kubernetes API Server stores the Deployment information in ETCD.
If ETCD data is lost, Kubernetes may lose its stored cluster state.
Therefore, taking regular ETCD backups is important for disaster recovery.

---

## 2. What is ETCD Backup?

ETCD Backup means creating a snapshot of the ETCD database so that the Kubernetes cluster state can be recovered if something goes wrong. An ETCD snapshot contains the Kubernetes data stored in the ETCD keyspace at the time of the backup.

Example: etcd-snapshot.db
This snapshot can later be used to restore the cluster to its previous state.

### Why Do We Need ETCD Backup?

- Protects Kubernetes cluster data.
- Helps recover deleted Kubernetes resources.
- Helps recover from ETCD database corruption.
- Helps recover from control plane failures.
- Provides disaster recovery capability.

---

## 3. What are etcdctl and etcdutl?

Both are command-line tools used for managing and maintaining ETCD.

| etcdctl                           | etcdutl                           |
|-----------------------------------|-----------------------------------|
| Communicates with running ETCD    | Works with ETCD data files        |
| Used for cluster operations       | Used for offline administration   |
| Takes snapshots                   | Restores snapshots                |
| Checks endpoint health            | Checks snapshot status            |

---

## ETCD Backup and Restore Practical

## 4. Step 1: Log in to the Control Plane Node

Connect to your control plane Node.

Check the current user: whoami
Check the hostname: hostname

---

## 6. Step 2: Verify Cluster Health

Check all Kubernetes Nodes:
```powershell
kubectl get nodes -o wide
```
Make sure all Nodes are in Ready status.

Example:

NAME           STATUS   ROLES
master-node    Ready    control-plane
worker-node1   Ready    worker
worker-node2   Ready    worker

Check all Pods:
kubectl get pods -A

Check system Pods:
kubectl get pods -n kube-system

Make sure important system components are healthy before taking the backup.

## 7. Step 3: Find the ETCD Version

ETCD runs as a static Pod in a kubeadm-based Kubernetes cluster.
Check the ETCD Pod:
kubectl get pods -n kube-system | grep etcd

Example:
etcd-master-node   1/1   Running

Find the ETCD version:
kubectl exec -n kube-system etcd-master-node -- etcd --version

Example output:
etcd Version: 3.5.x
Git SHA: ...
Go Version: ...
Go OS/Arch: linux/amd64

Note: Replace etcd-master-node with the actual ETCD Pod name in your cluster.
You can also check the ETCD container image: 
kubectl describe pod etcd-master-node -n kube-system

Look for the Image field.
---

## 8. Step 4: Create an NGINX Deployment with Two Pods

Create a Deployment named nginx with two replicas.
Command:
```powershell
kubectl create deployment nginx --image=nginx --replicas=2
```

Verify the Deployment:
kubectl get deployment

Check the Pods:
kubectl get pods -o wide

Expected output:

NAME                     READY   STATUS
nginx-xxxxxxxxxx-xxxxx   1/1     Running
nginx-xxxxxxxxxx-yyyyy   1/1     Running

Verify that the Deployment has two available replicas:
kubectl get deployment nginx

Expected:
READY: 2/2

---

## 9. Step 5: Identify ETCD Endpoint URL

ETCD uses client URLs to allow clients such as the Kubernetes API Server and etcdctl to communicate with it.
For a typical kubeadm cluster, the local ETCD endpoint is:

https://127.0.0.1:2379

Port 2379 is used for client communication.
Port 2380 is commonly used for peer communication between ETCD members.

### Check ETCD Manifest

Run: sudo cat /etc/kubernetes/manifests/etcd.yaml

Look for the following arguments:

--listen-client-urls=https://127.0.0.1:2379,https://<control-plane-ip>:2379

--advertise-client-urls=https://127.0.0.1:2379

These arguments help identify the client endpoint.

You can also search directly:
sudo grep -E 'listen-client-urls|advertise-client-urls' /etc/kubernetes/manifests/etcd.yaml

For this practical, we will use:
ETCD_ENDPOINT=https://127.0.0.1:2379

Note:
Use an endpoint that is actually configured and reachable in your cluster.

---

## 10. Step 6: Identify Server Certificate and CA Certificate

ETCD uses TLS certificates to secure communication.
For a typical kubeadm cluster, certificates are located inside:
/etc/kubernetes/pki/etcd/

List the certificates:
sudo ls -l /etc/kubernetes/pki/etcd/

Common files:

1. ca.crt
2. server.crt
3. server.key
4. peer.crt
5. peer.key
6. healthcheck-client.crt
7. healthcheck-client.key

### Important Certificates

A. CA Certificate

Path:
/etc/kubernetes/pki/etcd/ca.crt

Purpose:
- Used to verify the identity of the ETCD server.
- Establishes trust for TLS communication.

B. Server Certificate

Path:
/etc/kubernetes/pki/etcd/server.crt

Purpose:
- Used by the ETCD server to identify itself.
- Used for TLS communication.

C. Server Private Key

Path:
/etc/kubernetes/pki/etcd/server.key

Purpose:
- Private key corresponding to the server certificate.
- Must be protected.

You can inspect the ETCD Pod configuration to identify the TLS settings:
kubectl describe pod etcd-master-node -n kube-system

Or:

sudo grep -E 'cert-file|key-file|trusted-ca-file|client-cert-auth' /etc/kubernetes/manifests/etcd.yaml

For this practical, define the certificate paths:
ETCD_ENDPOINT=https://127.0.0.1:2379

ETCD_CACERT=/etc/kubernetes/pki/etcd/ca.crt

ETCD_CERT=/etc/kubernetes/pki/etcd/healthcheck-client.crt

ETCD_KEY=/etc/kubernetes/pki/etcd/healthcheck-client.key

---

## 11. Step 7: Verify ETCD Connectivity

Before taking a backup, verify that etcdctl can communicate with ETCD.

Set the API version:
export ETCDCTL_API=3

Check endpoint health:
sudo ETCDCTL_API=3 etcdctl 
--endpoints=https://127.0.0.1:2379 
--cacert=/etc/kubernetes/pki/etcd/ca.crt 
--cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt 
--key=/etc/kubernetes/pki/etcd/healthcheck-client.key 
endpoint health

Expected output:
https://127.0.0.1:2379 is healthy

If the endpoint is healthy, we can proceed with the backup.

---

## 12. Step 8: Verify etcdctl and etcdutl Installation

Check whether etcdctl is installed:
etcdctl version

Check whether etcdutl is installed:
etcdutl version

If the commands are not found, install the compatible ETCD tools.
In many kubeadm environments, etcdctl is available inside the ETCD container even if it is not installed on the host. For this assignment, make sure both binaries are available on the control plane Node.

---

## 13. Step 9: Explore Snapshot Commands

Before performing the backup, understand the available options.
Check etcdctl snapshot help:
sudo etcdctl snapshot --help

Important commands:

ETCDCTL:
etcdctl snapshot save

Used to create a snapshot of the running ETCD cluster.

ETCDUTL:
etcdutl snapshot restore

Used to restore an ETCD snapshot into a new data directory.

etcdutl snapshot status
Used to inspect the snapshot information.

---

## 14. Step 10: Take ETCD Backup

We will take the ETCD backup at:
/opt/etcd-snapshot.db

Create the backup using etcdctl:
sudo ETCDCTL_API=3 etcdctl 
--endpoints=https://127.0.0.1:2379 
--cacert=/etc/kubernetes/pki/etcd/ca.crt 
--cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt 
--key=/etc/kubernetes/pki/etcd/healthcheck-client.key 
snapshot save /opt/etcd-snapshot.db

### Explanation

- --endpoints: ETCD client endpoint.
- --cacert: CA certificate used to verify the ETCD server.
- --cert: Client certificate used for authentication.
- --key: Private key corresponding to the client certificate.
- snapshot save: Creates an ETCD snapshot.
- /opt/etcd-snapshot.db: Backup file location.

Check the backup file:
sudo ls -lh /opt/etcd-snapshot.db

### Verify Snapshot

Run:
sudo etcdutl snapshot status /opt/etcd-snapshot.db -w table

This displays:

- HASH
- REVISION
- TOTAL KEYS
- TOTAL SIZE

This helps verify that the snapshot is readable.

---

## 15. Step 11: Delete the NGINX Deployment

Now delete the Deployment that we created earlier.
Command:
kubectl delete deployment nginx

Expected output:
deployment.apps "nginx" deleted

Verify:
kubectl get deployment

The nginx Deployment should no longer be present.

Also check:
kubectl get pods

The NGINX Pods should be terminated.
---

# PART 3: ETCD Restore

## 16. Step 12: Restore the ETCD Snapshot

We will restore the backup into a new directory:
/var/lib/etcd-from-backup

Command:
sudo etcdutl snapshot restore /opt/etcd-snapshot.db 
--data-dir=/var/lib/etcd-from-backup

### Explanation

- etcdutl: ETCD administration utility.
- snapshot restore: Restores the snapshot.
- /opt/etcd-snapshot.db: Source backup file.
- --data-dir: Destination directory for the restored ETCD database.

After successful restoration, verify the directory:
sudo ls -l /var/lib/etcd-from-backup
The directory should contain the restored ETCD data.

---

## 17. Step 13: Point ETCD to the Restored Directory

Now we need to configure ETCD to use:
/var/lib/etcd-from-backup

ETCD runs as a static Pod in kubeadm clusters.
Its configuration is stored in: /etc/kubernetes/manifests/etcd.yaml

Open the manifest: sudo vi /etc/kubernetes/manifests/etcd.yaml

Find the volumes section.

You will see something similar to:
volumes:
- hostPath:
    path: /var/lib/etcd
    type: DirectoryOrCreate
  name: etcd-data

Change the hostPath path to:
volumes:
- hostPath:
    path: /var/lib/etcd-from-backup
    type: DirectoryOrCreate
  name: etcd-data

Important:
Change only the hostPath source directory.
The container's mountPath generally remains:

/var/lib/etcd
This is because the ETCD process inside the container continues using the same internal data directory, while the host directory supplying that data has changed.

### Also Check the ETCD Data Directory Argument

Look for:
--data-dir=/var/lib/etcd

For the typical kubeadm static Pod configuration, this container path remains unchanged.
Do not change it to the host path unless your container mount configuration also changes.

### Save the File

In vi:
Press ESC.
Type:
:wq
Press Enter.

---

## 18. Step 14: Understand Static Pod Restart

ETCD is generally deployed as a static Pod in kubeadm clusters. Static Pods are directly managed by kubelet rather than by the Kubernetes Deployment or ReplicaSet controllers.

Kubelet monitors the static Pod manifest directory:
/etc/kubernetes/manifests/

When the ETCD manifest changes, kubelet detects the modification and recreates the static Pod using the updated configuration. The ETCD Pod will restart and use the restored data directory.

---

## 19. Step 15: Restart ETCD Pod

After updating the manifest, kubelet should detect the change and recreate the ETCD Pod.
You can check the Pod:
kubectl get pods -n kube-system

Check ETCD Pod status:
kubectl get pods -n kube-system | grep etcd

If the API Server is temporarily unavailable during the restart, check kubelet directly:
sudo systemctl status kubelet

If necessary, restart kubelet:
sudo systemctl restart kubelet

Kubelet will continue managing the static Pod using the updated manifest.

Check the ETCD Pod again once the API Server becomes available:
kubectl get pods -n kube-system

---

## 20. Step 16: Verify ETCD Is Running from the Restored Directory

Check the ETCD Pod:
kubectl get pods -n kube-system

Check its logs:
kubectl logs -n kube-system etcd-master-node

Check the ETCD endpoint health again:
sudo ETCDCTL_API=3 etcdctl 
--endpoints=https://127.0.0.1:2379 
--cacert=/etc/kubernetes/pki/etcd/ca.crt 
--cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt 
--key=/etc/kubernetes/pki/etcd/healthcheck-client.key 
endpoint health

Expected: https://127.0.0.1:2379 is healthy

---

# PART 4: Verify Restored Kubernetes Resources

## 21. Step 17: Check the NGINX Deployment

Now check whether the deleted Deployment has been restored.
Run:
kubectl get deployments

Expected:

NAME     READY   UP-TO-DATE   AVAILABLE
nginx    2/2     2            2

The NGINX Deployment should appear again because it was present in the ETCD snapshot.

Check the Pods:
kubectl get pods -o wide

Expected:
Two NGINX Pods should be created again.

Check Deployment details:
kubectl describe deployment nginx

Verify the replica count: 
kubectl get deployment nginx

Expected:
READY: 2/2

### Why Did the Deployment Come Back?

Before deleting the Deployment, we created an ETCD snapshot.
That snapshot contained the original Kubernetes state, including:

- Deployment object.
- Replica count.
- Pod template.
- Container image.
- Labels and selectors.

When we restored the snapshot, the API Server could access the previous cluster state again.
The Kubernetes Deployment Controller detected that the Deployment should have two replicas and created the required Pods.

Therefore, the Deployment appeared again.
Important:
The original running Pods are not restored as the same processes. Kubernetes recreates Pods from the restored Deployment specification.

---

# Conclusion

ETCD is the most important data store in Kubernetes because it maintains the cluster's state. Taking regular ETCD snapshots helps protect the cluster against accidental deletion, database corruption, and control plane failures. The etcdctl utility is used to take snapshots from a running ETCD cluster, while etcdutl is used to restore snapshots into a new data directory. After restoring the database, the ETCD static Pod must be configured to use the restored directory. Finally, Kubernetes controllers reconcile the restored state and recreate the required application resources.