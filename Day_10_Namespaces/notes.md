# Kubernetes Networking: Namespaces, Deployments, Services, and Cross-Namespace Communication

## 1. Create Two Namespaces

A **Namespace** provides logical separation for Kubernetes resources within a cluster. Resources in different namespaces can have the same name.

Create the namespaces:
```powershell
kubectl create namespace ns1
kubectl create namespace ns2
```

Verify:
```powershell
kubectl get namespaces
```

The output should include `ns1` and `ns2`.


## 2. Create Deployments in Both Namespaces

Create a Deployment named `deploy-ns1` in `ns1`:

```powershell
kubectl create deployment deploy-ns1 --image=nginx --replicas=1 -n ns1
```

Create a Deployment named `deploy-ns2` in `ns2`:

```powershell
kubectl create deployment deploy-ns2 --image=nginx --replicas=1 -n ns2
```

Verify the Deployments:

```powershell
kubectl get deployments -n ns1
kubectl get deployments -n ns2
```

Check their Pods:

```powershell
kubectl get pods -n ns1 -o wide
kubectl get pods -n ns2 -o wide
```

The `-o wide` option displays additional details, including each Pod's IP address and the node where it is running.


## 3. Get the Pod IP Addresses

Run:

```powershell
kubectl get pods -n ns1 -o wide
kubectl get pods -n ns2 -o wide
```
Record the IP address of the Pod belonging to each Deployment.

Example only:
```text
NAME                         READY   STATUS    IP
deploy-ns1-xxxxxxxxxx-xxxxx  1/1     Running   10.244.1.10
```

Your actual Pod names and IP addresses will differ. Use the real IPs returned by your cluster in the commands below.


## 4. Test Pod-to-Pod Communication Across Namespaces

First, obtain the Pod name in `ns1`:

```powershell
kubectl get pods -n ns1
```

Exec into that Pod:
```powershell
kubectl exec -it <pod-name-in-ns1> -n ns1 -- sh
```

Inside the container, run:
```sh
curl http://<pod-ip-in-ns2>
```
Replace `<pod-ip-in-ns2>` with the actual IP address of the Nginx Pod in `ns2`.
A successful response should contain Nginx's HTML welcome page.

Exit the container:
```sh
exit
```
*Do the same in ns2 shell using ns1's IP*

### Why Does This Work?
Pods can communicate across namespaces using Pod IP addresses when the cluster network allows it and no NetworkPolicy blocks the connection.
The namespace affects resource names and scope, but it does not inherently block Pod-to-Pod networking.


## 5. Scale Both Deployments to Three Replicas

Scale `deploy-ns1`:
```powershell
kubectl scale deployment deploy-ns1 --replicas=3 -n ns1
```

Scale `deploy-ns2`:
```powershell
kubectl scale deployment deploy-ns2 --replicas=3 -n ns2
```

Verify:
```powershell
kubectl get deployments -n ns1
kubectl get deployments -n ns2
```

Check all Pods:
```powershell
kubectl get pods -n ns1 -o wide
kubectl get pods -n ns2 -o wide
```

*Wait until all three Pods in each namespace are ready.*


## 6. Create a Service for Each Deployment

A **Service** provides a stable IP address and DNS name for accessing a group of Pods. This is useful because individual Pod IP addresses can change when Pods are recreated.

Create a ClusterIP Service for `deploy-ns1`:
```powershell
kubectl expose deployment deploy-ns1 --name=svc-ns1 --port=80 -n ns1
```

Create a ClusterIP Service for `deploy-ns2`:
```powershell
kubectl expose deployment deploy-ns2 --name=svc-ns2 --port=80 -n ns2
```

Verify the Services:
```powershell
kubectl get services -n ns1
kubectl get services -n ns2
```

Check that the Services have ClusterIP addresses and that their endpoints include the corresponding Pods:
```powershell
kubectl get endpoints svc-ns1 -n ns1
kubectl get endpoints svc-ns2 -n ns2
```


## 7. Curl the Service IP Across Namespaces

Retrieve the Service IP addresses:
```powershell
kubectl get svc -n ns1
kubectl get svc -n ns2
```

Exec into a Pod in `ns1`:
```powershell
kubectl get pods -n ns1
kubectl exec -it <pod-name-in-ns1> -n ns1 -- sh
```

Inside the container, curl the ClusterIP of `svc-ns2`:
```sh
curl http://<svc-ns2-cluster-ip>
```
The request should return the Nginx welcome page.

*Exit and test the reverse direction by exec-ing into a Pod in `ns2`:*

```powershell
kubectl exec -it <pod-name-in-ns2> -n ns2 -- sh
```

Inside the container:
```sh
curl http://<svc-ns1-cluster-ip>
```
This should also return the Nginx welcome page.


## 8. Try Curling the Short Service Name

While inside a Pod in `ns1`, run:
```sh
curl http://svc-ns2
```

The short name `svc-ns2` normally resolves to a Service in the **same namespace as the calling Pod**. Since the Pod is in `ns1` and the Service is in `ns2`, this name does not identify `ns2`'s Service.

You may see a DNS resolution error, such as:
```text
curl: (6) Could not resolve host: svc-ns2
```
Exact error wording can vary.

*Do the same in ns2 shell*

**Important:** A short Service name can resolve if a Service with that name exists in the caller's own namespace. The failure here occurs because `svc-ns2` is in a different namespace.



## 9. Use the Fully Qualified Domain Name (FQDN)

Kubernetes Service DNS names follow this general format:

```text
<service-name>.<namespace>.svc.cluster.local
```

From a Pod in `ns1`, access the Service in `ns2` using:
```sh
curl http://svc-ns2.ns2.svc.cluster.local
```

From a Pod in `ns2`, access the Service in `ns1` using:
```sh
curl http://svc-ns1.ns1.svc.cluster.local
```

Both requests should return the Nginx welcome page.

### Why Does the FQDN Work?
The FQDN identifies the Service and its namespace explicitly.

For example:
- `svc-ns2` — short name, looked up in the caller's namespace.
- `svc-ns2.ns2.svc.cluster.local` — identifies `svc-ns2` in namespace `ns2`.

`cluster.local` is the default cluster DNS domain in many Kubernetes clusters. If your cluster uses a different DNS domain, the FQDN will differ.


## 10. Delete Both Namespaces

Delete `ns1`:
```powershell
kubectl delete namespace ns1
```

Delete `ns2`:
```powershell
kubectl delete namespace ns2
```

Verify:
```powershell
kubectl get namespaces
```
The namespaces should no longer appear once deletion completes.

### Resources
Deleting a namespace also deletes the namespaced resources inside it, including:
- Deployments
- ReplicaSets
- Pods
- Services
You do not need to delete each of these resources individually.


## Troubleshooting Commands
| List namespaces           | `kubectl get namespaces`                                          |
| List Pods with IPs        | `kubectl get pods -n ns1 -o wide`                                 |
| Inspect a Deployment      | `kubectl describe deployment deploy-ns1 -n ns1`                   |
| Inspect a Service         | `kubectl describe service svc-ns2 -n ns2`                         |
| Check Service endpoints   | `kubectl get endpoints svc-ns2 -n ns2`                            |
| Check DNS configuration   | `kubectl exec -it <pod-name> -n ns1 -- cat /etc/resolv.conf`      |
| Inspect recent events     | `kubectl get events -n ns1 --sort-by=.metadata.creationTimestamp` |


## Final Summary :
1. Namespaces logically separate Kubernetes resources.
2. Pods can communicate across namespaces by Pod IP if networking policies permit it.
3. Deployments and ReplicaSets maintain the desired number of Pod replicas.
4. Services provide stable ClusterIP addresses and DNS names for Pods.
5. Short Service names resolve within the caller's namespace by default.
6. Namespace-qualified names and FQDNs allow reliable cross-namespace Service discovery.
7. Deleting a namespace removes the namespaced resources inside it.