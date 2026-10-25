### Kubernetes Services & Deployments — Lab Documentation

`Create a Service named myapp of type ClusterIP`

```powershell
kubectl create service clusterip myapp --tcp=80:80
```

Alternatively, you can create the Service using a YAML file:
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP


Verify the Service:
kubectl get svc myapp
kubectl describe svc myapp


`Create a Deployment named myapp`

Create a Deployment with one replica using the image nginx:1.23.4-alpine and expose container port 80.

# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.23.4-alpine
          ports:
            - containerPort: 80

Apply the configuration:
kubectl apply -f deployment.yaml

```powershell
kubectl create deployment myapp --image=nginx:1.23.4-alpine --replicas=1
```

Verify:
kubectl get deployments
kubectl get pods
kubectl get svc myapp


Explanation:
- A Deployment manages the desired number of Pod replicas.
- The container runs NGINX using the specified image.
- containerPort: 80 documents the port on which the container is expected to receive traffic.
- The Pod label app: myapp matches the Service selector, allowing the Service to route traffic to the Pod.


`Scale the Deployment to 2 replicas`

Command:
kubectl scale deployment myapp --replicas=2


Verify:
kubectl get deployment myapp
kubectl get pods -l app=myapp


Expected result:
The Deployment should show two desired and ready replicas once both Pods are running and ready.

Explanation:
Scaling increases the number of Pods managed by the Deployment. The Service automatically routes requests to the matching, ready Pods.

`Create a temporary BusyBox Pod and run wget against the Service IP`
First, get the Service IP:
```powershell
kubectl get svc myapp
```

Example output:
NAME    TYPE        CLUSTER-IP      PORT(S)
myapp   ClusterIP   10.96.73.112    80/TCP

*Your actual ClusterIP may be different.*

Create a temporary BusyBox Pod:
kubectl run busybox-test --image=busybox --restart=Never -- sleep 3600


Run wget from inside the Pod using the Service IP:
kubectl exec busybox-test -- wget -qO- http://10.96.73.112


Replace 10.96.73.112 with your actual Service IP.

Alternatively, use the Service DNS name:
kubectl exec busybox-test -- wget -qO- http://myapp


Expected result:
The command should return the default NGINX welcome page HTML.

Delete the temporary Pod after testing:
kubectl delete pod busybox-test


Explanation:

A ClusterIP Service is accessible from within the cluster. Kubernetes DNS allows Pods in the same namespace to access the Service using its name, myapp, instead of its IP address.

`Run wget against the Service from outside the cluster`

From your local machine or PowerShell, try:
wget http://10.96.73.112


Replace the example IP with the actual ClusterIP.

In PowerShell, you can also use:
curl.exe http://10.96.73.112


Expected result:
The request will normally fail because a ClusterIP is intended for internal cluster communication and is not directly exposed outside the cluster.

Explanation:
The Service's internal virtual IP is reachable through the cluster networking system. Your local machine generally cannot access that IP directly.

`Change the Service type so Pods can be reached outside the cluster`
Change the Service type from ClusterIP to NodePort.
```powershell
kubectl edit svc myapp 
```
*File must have opened in notepad edit the type: NodePort*

Verify the updated Service:
kubectl get svc myapp

Example output:
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
myapp   NodePort   10.96.73.112    <none>        80:30001/TCP


In this example:
- 80 is the Service port.
- 30001 is the assigned NodePort.
- Requests sent to a reachable node IP on port 30001 can be forwarded to the Service's Pods.
The actual NodePort may differ. Kubernetes normally assigns a port from the range 30000–32767.

Explanation:
NodePort exposes a Service on a port of each cluster node. It allows access from outside the cluster when the node address is reachable and firewall rules permit the connection.

`Run wget against the Service from outside the cluster`
Get the node's IP address and the assigned NodePort.
kubectl get nodes -o wide
kubectl get svc myapp

Run the request using the node IP and NodePort:
wget -qO- http://<NODE-IP>:<NODE-PORT>

For example:
wget -qO- http://192.0.2.10:30001

The IP above is an example only. Replace it with your actual reachable node IP and the NodePort assigned to your Service.

In PowerShell, you can use:
curl.exe http://<NODE-IP>:<NODE-PORT>

Expected result:
If your cluster networking supports direct NodePort access and the required ports are allowed, you should receive the NGINX welcome page.

*Investigated why NodePort access failed*
We tried accessing the Service through the node IP and port 30393, but the connection failed.

We discovered that:
- We were using a kind cluster.
- kind runs Kubernetes nodes inside Docker containers.
- Docker had a host port mapping for 30001, but not for 30393.
- Therefore, the NodePort wasn't directly accessible from Windows through that port mapping.

Used port forwarding to access NGINX
```powershell
kubectl port-forward service/myapp 8080:80
```

On another terminal again verified:
```powershell
curl.exe http://localhost:8080
```
The NGINX page loaded successfully!

Important note for kind or Minikube:
- With kind, external access to NodePort may require port mappings configured when creating the cluster.
- With Minikube, you can try:
minikube service myapp --url
Use the URL returned by the command to test connectivity.

`Can you expose Pods as a Service without a Deployment?`
Yes. A Deployment is not required to create a Service.

A Service routes traffic to Pods based on labels and selectors. Those Pods can be created by:
- A Deployment
- A StatefulSet
- A DaemonSet
- A Job
- Manually created Pod resources
For example, a standalone Pod could use this label:
metadata:
  labels:
    app: myapp


A Service with the selector below can route traffic to that Pod:
selector:
  app: myapp


As long as the Pod is running, ready, and matches the selector, it can receive traffic through the Service.

Why use a Deployment, then?
A Deployment provides automatic Pod replacement, replica management, and rolling updates. It is generally preferred for stateless applications such as NGINX.

A Service can also work without a selector when you manually manage the corresponding EndpointSlices, such as when directing traffic to an external backend.

`Under what conditions would you use ClusterIP, NodePort, LoadBalancer, and ExternalName`

Kubernetes defines four standard Service types: ClusterIP, NodePort, LoadBalancer, and ExternalName.

Service type	Purpose	When to use it
ClusterIP	    Exposes the Service through an internal cluster IP.	For communication 
                between applications inside the cluster, such as a frontend accessing a backend API.

NodePort	    Exposes the Service on a port on each node.	For development, testing, 
                learning labs, or environments where direct node access is suitable.

LoadBalancer	Requests an external load balancer from a supported infrastructure provider.	
                For exposing an application externally, particularly in cloud environments such as AWS or Azure.

ExternalName	Maps a Kubernetes Service name to an external DNS name using a CNAME record.
                When an application inside Kubernetes needs to refer to an external service through a Kubernetes DNS name.

ClusterIP
- This is the default Service type.
- It is intended for internal communication.
- It does not directly expose the application to external clients.
Example: A backend API communicating with a database inside the cluster.
NodePort
- Opens a port on each node.
- External clients can connect through a reachable node IP and NodePort.
- Firewall rules and cluster networking must permit the traffic.
Example: Testing an application from a local machine in a learning environment.
LoadBalancer
- Requests an external load balancer when supported by the cluster's infrastructure.
- Commonly used for applications that must be accessible from outside the cluster.
- The exact implementation depends on the cloud provider or installed load-balancer solution.
Example: Exposing a web application running on an AKS or EKS cluster.
ExternalName
- Creates a DNS alias for an external hostname.
- Does not create a proxy or expose Pods.
- Does not allocate a ClusterIP in the usual ExternalName configuration.
Example: Allowing an application to access an external database using a Kubernetes Service DNS name that resolves to the database's external hostname.

Terminology note: External is not a standard Kubernetes Service type. The likely intended type is ExternalName.

Conclusion
In this lab, we created a ClusterIP Service and an NGINX Deployment, scaled the Deployment to two replicas, tested internal connectivity using BusyBox and wget, and changed the Service type to NodePort to explore external access.

The key takeaway is that a Service provides a stable way to access matching Pods, while a Deployment manages the lifecycle and replica count of Pods. ClusterIP is intended for internal access, NodePort exposes a port on each node, LoadBalancer provides an external load-balancing entry point, and ExternalName provides a DNS alias to an external hostname.