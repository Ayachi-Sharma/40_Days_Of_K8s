# Kubernetes Networking, OCI, CNI & Inter-Pod Communication

## 1. What is Kubernetes Networking?

Kubernetes Networking is the system that enables communication between Pods, Services, Nodes, and external networks inside a Kubernetes cluster.

Kubernetes applications usually run inside containers, and these containers need networking to communicate with each other.

Kubernetes networking allows:

- Container-to-container communication
- Pod-to-Pod communication
- Pod-to-Service communication
- External-to-Service communication
- Pod-to-Internet communication

### Kubernetes Networking Model

Kubernetes follows a flat networking model.

The fundamental requirements are:

1. Every Pod gets its own IP address.
2. Pods can communicate with other Pods without Network Address Translation (NAT) in the normal cluster networking model.
3. Nodes can communicate with Pods, and Pods can communicate with Nodes.
4. Containers inside the same Pod share the same network namespace.

Example:

Pod A: 10.244.1.10

Pod B: 10.244.2.20

Pod A can communicate directly with Pod B using its IP address, provided network policies, firewalls, and routing permit it.

Important:

Kubernetes defines the networking requirements, but the actual networking implementation is provided by the container runtime, CNI plugins, and underlying network infrastructure.

---

## 2. What is OCI?

OCI stands for Open Container Initiative.

It is an open standardization project that defines specifications for container technologies.

OCI helps ensure that container images and runtimes follow common standards.

### Main OCI Specifications

1. OCI Image Specification

   Defines how container images are structured, packaged, and distributed.

2. OCI Runtime Specification

   Defines how a container is configured and executed.

3. OCI Distribution Specification

   Defines standard APIs for distributing container images through registries.

### Examples of OCI-Compatible Technologies

- Docker produces and works with OCI-compatible images.
- containerd manages container lifecycle and images.
- runc is a low-level OCI container runtime.
- CRI-O is a Kubernetes-oriented container runtime implementation.

### OCI in Kubernetes

A typical container execution flow is:

Kubernetes API Server
        |
        v
Kubelet
        |
        v
CRI (Container Runtime Interface)
        |
        v
containerd / CRI-O
        |
        v
OCI Runtime (runc)
        |
        v
Container Running

Explanation:

1. Kubernetes schedules a Pod on a Node.
2. Kubelet receives the Pod specification.
3. Kubelet communicates with the container runtime using CRI.
4. The container runtime manages the container.
5. A low-level OCI runtime such as runc creates and starts the container.

### OCI vs CRI vs CNI

| Technology | Full Form | Purpose |
|---|---|---|
| OCI | Open Container Initiative | Defines container standards |
| CRI | Container Runtime Interface | Connects kubelet with container runtimes |
| CNI | Container Network Interface | Configures container networking |

Important:

OCI and CNI are different.

OCI defines standards for container images and runtimes, while CNI provides a framework for configuring container network connectivity.

---

## 3. What is CNI?

CNI stands for Container Network Interface.

CNI is a specification and collection of libraries used to configure network interfaces for containers.

In Kubernetes, CNI plugins are commonly responsible for setting up Pod networking.

### Why Do We Need CNI?

Kubernetes does not implement the complete Pod network itself.

A CNI-based networking solution provides connectivity between Pods and the cluster network.

Typical responsibilities include:

- Configuring Pod network interfaces
- Assigning Pod IP addresses through IPAM
- Setting up routes
- Connecting Pods to the cluster network
- Supporting cross-node communication

Some networking solutions also implement NetworkPolicy enforcement.

### Popular CNI Plugins

1. Calico
   - Supports Pod networking.
   - Supports Kubernetes NetworkPolicy.
   - Can use routing or encapsulation.

2. Cilium
   - Uses eBPF for networking, security, and observability.
   - Supports NetworkPolicy.

3. Flannel
   - Provides relatively simple Pod networking.
   - Does not provide native Kubernetes NetworkPolicy enforcement on its own.

4. Weave Net
   - Provides container networking across nodes.

5. Azure CNI
   - Provides networking for Azure Kubernetes Service (AKS).

Note:

A CNI plugin and a complete Kubernetes networking solution are not always exactly the same thing. A networking solution may combine CNI, IP address management, routing, and policy enforcement.

---

## 4. How Does CNI Work?

When Kubernetes creates a Pod, its network must be configured before the application containers can communicate normally.

### Step-by-Step Process

1. User creates a Pod using kubectl.
2. Kubernetes schedules the Pod on a Node.
3. Kubelet asks the container runtime to create the Pod sandbox.
4. The container runtime invokes the configured CNI networking components.
5. The networking plugin configures the Pod network namespace.
6. A network interface and IP address are assigned to the Pod.
7. Routes and other networking settings are configured.
8. The Pod becomes capable of network communication.

### Networking Flow

kubectl apply -f pod.yaml
          |
          v
Kubernetes API Server
          |
          v
Scheduler selects Node
          |
          v
Kubelet
          |
          v
Container Runtime
          |
          v
CNI Plugin
          |
          v
Pod Network Configured
          |
          v
Pod Gets IP Address
          |
          v
Pod Can Communicate

---

## 5. What is a Network Namespace?

A network namespace is a Linux kernel feature that provides an isolated network environment.

Each network namespace can have its own:

- Network interfaces
- IP addresses
- Routing tables
- Network-related configuration
- Network sockets and ports

### Network Namespace in Kubernetes

Containers inside the same Pod share one network namespace.

Example:

Pod A contains:

- Container 1: nginx
- Container 2: sidecar

Both containers share:

- Same Pod IP address
- Same network interfaces
- Same network namespace
- Same port space

They can communicate using localhost.

Example:

Container 1 runs on port 8080.

Container 2 can access it using:

http://localhost:8080

Important:

Two containers in the same Pod cannot independently bind to the same IP address and port using the same transport protocol.

---

## 6. What is a veth Pair?

A veth pair is a pair of virtual Ethernet interfaces.

It acts like a virtual cable connecting two network namespaces.

In common Linux bridge-based Pod networking implementations:

- One end of the veth pair exists inside the Pod network namespace.
- The other end exists in the host network namespace.
- The host-side interface connects to a bridge or other networking component.

### Example

Pod A
  |
  | eth0
  |
  v
veth pair
  |
  v
Linux Bridge
  |
  v
Node Network

Note:

Not all CNI implementations use exactly the same architecture. Some use eBPF, routing, or other networking mechanisms.

---

## 7. Inter-Pod Communication in Kubernetes

Inter-Pod Communication means communication between two different Pods in a Kubernetes cluster.

There are two main scenarios:

1. Pods running on the same Node.
2. Pods running on different Nodes.

### A. Pod-to-Pod Communication on the Same Node

Suppose:

Node 1 contains:

Pod A: 10.244.1.10

Pod B: 10.244.1.20

Both Pods are running on the same worker Node.

### Communication Flow

Pod A
  |
  v
Pod A Network Interface
  |
  v
Host Networking / Bridge
  |
  v
Pod B Network Interface
  |
  v
Pod B

### Step-by-Step Explanation

1. Pod A wants to communicate with Pod B.
2. Pod A sends a packet to Pod B's IP address.
3. The packet leaves Pod A through its network interface.
4. The Node's networking implementation forwards the packet toward Pod B.
5. Pod B receives the packet.

The traffic can remain within the same Node without traveling to another worker Node.

Important:

The exact forwarding path depends on the CNI networking implementation.

---

### B. Pod-to-Pod Communication on Different Nodes

Suppose:

Node 1:
Pod A: 10.244.1.10

Node 2:
Pod B: 10.244.2.20

Now Pod A needs to communicate with Pod B running on a different Node.

### Communication Flow

Pod A
(Node 1)
   |
   v
Node 1 Networking
   |
   v
Node 1 Network Interface
   |
   v
Physical / Virtual Network
   |
   v
Node 2 Network Interface
   |
   v
Node 2 Networking
   |
   v
Pod B
(Node 2)

### Step-by-Step Explanation

1. Pod A sends a packet to Pod B's IP address.
2. The packet reaches the networking layer on Node 1.
3. The networking solution determines how to reach Pod B's Node.
4. The packet travels through the underlying network.
5. Node 2 receives the packet.
6. Node 2 forwards the packet to Pod B.
7. Pod B receives the packet.

### How Does Cross-Node Networking Work?

Different CNI networking solutions use different techniques.

Common techniques include:

1. Direct Routing

   Nodes use routes to forward traffic toward Pod IP ranges on other Nodes.

2. Overlay Networking

   Pod traffic is encapsulated inside another network packet for transmission across the underlying network.

3. eBPF-Based Networking

   Some solutions use eBPF programs for efficient packet forwarding, load balancing, and policy enforcement.

### What is VXLAN?

VXLAN stands for Virtual Extensible LAN.

It is an encapsulation technology used by some overlay networking solutions.

It allows network traffic to be transported across an underlying IP network.

Example:

Pod A Packet
      |
      v
VXLAN Encapsulation
      |
      v
Underlying Node Network
      |
      v
VXLAN Decapsulation
      |
      v
Pod B

Not every Kubernetes cluster uses VXLAN.

---

## 8. Pod-to-Service Communication

Pod IP addresses are temporary.

When a Pod is deleted and recreated, it may receive a different IP address.

Therefore, Kubernetes provides Services to offer stable access to groups of Pods.

Example:

Backend Pods:

backend-1: 10.244.1.10

backend-2: 10.244.2.20

Backend Service:

backend-service: 10.96.10.20

The frontend Pod communicates with:

http://backend-service

### Communication Flow

Frontend Pod
     |
     v
CoreDNS
     |
     v
Backend Service ClusterIP
     |
     v
Service Traffic Routing
     |
     v
Backend Pod

### Explanation

1. The frontend Pod queries CoreDNS for the backend Service.
2. CoreDNS returns the Service ClusterIP.
3. The frontend Pod sends traffic to the ClusterIP.
4. The cluster's Service networking implementation selects a backend endpoint.
5. The traffic reaches one of the backend Pods.

Note:

kube-proxy commonly implements Service routing using iptables, IPVS, or nftables depending on configuration and version. Some networking solutions replace kube-proxy functionality with eBPF-based implementations.

---

## 9. Kubernetes Service Types

Kubernetes provides different Service types.

### 1. ClusterIP

- Default Service type.
- Exposes the application within the cluster.
- Provides a stable virtual IP address.

Example:

kubectl expose deployment nginx --port=80 --type=ClusterIP

### 2. NodePort

- Exposes a Service through a port on each Node.
- Default NodePort range is 30000–32767.

Example:

kubectl expose deployment nginx --port=80 --type=NodePort

### 3. LoadBalancer

- Exposes a Service using an external load balancer when supported by the infrastructure.
- Commonly used in managed Kubernetes environments such as AKS, EKS, and GKE.

Example:

kubectl expose deployment nginx --port=80 --type=LoadBalancer

### 4. ExternalName

- Maps a Kubernetes Service name to an external DNS name.
- Uses DNS CNAME behavior rather than providing a normal Service ClusterIP.

---

## 10. What is NetworkPolicy?

NetworkPolicy is a Kubernetes resource used to control network traffic between Pods and other network destinations.

It can control:

- Ingress: Incoming traffic to selected Pods.
- Egress: Outgoing traffic from selected Pods.

By default, Pods are non-isolated for ingress and egress unless policies select them for isolation.

### Example NetworkPolicy

This policy allows ingress traffic to Pods labeled app=backend only from Pods labeled app=frontend in the same namespace.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80

Important:

NetworkPolicy enforcement requires a networking solution that supports Kubernetes NetworkPolicy.

Simply creating a NetworkPolicy does not guarantee enforcement if the installed networking plugin does not support it.

---

## 11. Practical: Inter-Pod Communication

We will create two Pods and test communication between them.

### Step 1: Check Kubernetes Nodes

kubectl get nodes -o wide

### Step 2: Create an Nginx Pod

kubectl run nginx-pod --image=nginx:stable

Check the Pod:

kubectl get pods -o wide

Note the Pod IP address.

### Step 3: Create a BusyBox Pod

kubectl run busybox-pod --image=busybox:1.36 --command -- sleep 3600

Check:

kubectl get pods -o wide

### Step 4: Test Pod-to-Pod Communication

Get the Nginx Pod IP:

kubectl get pod nginx-pod -o wide

Suppose the IP is:

10.244.1.10

Enter the BusyBox Pod:

kubectl exec -it busybox-pod -- sh

Inside the BusyBox Pod, run:

wget -qO- http://10.244.1.10

If networking is working, you should see the Nginx welcome HTML.

Note:

The actual Pod IP will be different in your cluster.

### Step 5: Create a Service

Expose the Nginx Pod:

kubectl expose pod nginx-pod --name=nginx-service --port=80 --target-port=80

Check the Service:

kubectl get svc

### Step 6: Test Service Communication

Enter the BusyBox Pod:

kubectl exec -it busybox-pod -- sh

Run:

nslookup nginx-service

Then:

wget -qO- http://nginx-service

This verifies DNS resolution and Pod-to-Service communication.

---

## 12. Practical: Cross-Node Pod Communication

To test communication between Pods on different Nodes, use a multi-node Kubernetes cluster.

### Step 1: Check Nodes

kubectl get nodes

### Step 2: Create Pods on Different Nodes

Create a file named cross-node.yaml.

apiVersion: v1
kind: Pod
metadata:
  name: nginx-node1
  labels:
    app: nginx-node1
spec:
  nodeName: <worker-node-1>
  containers:
  - name: nginx
    image: nginx:stable
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-node2
spec:
  nodeName: <worker-node-2>
  containers:
  - name: busybox
    image: busybox:1.36
    command:
    - sleep
    - "3600"

Replace the placeholders with actual worker Node names.

Apply the configuration:

kubectl apply -f cross-node.yaml

Check Pod placement:

kubectl get pods -o wide

Verify that the Pods are running on different Nodes.

### Step 3: Test Communication

Get the Nginx Pod IP:

kubectl get pod nginx-node1 -o wide

Enter BusyBox:

kubectl exec -it busybox-node2 -- sh

Run:

wget -qO- http://<nginx-pod-ip>

If the Nginx welcome page appears, cross-node Pod communication is working.

---

## 13. Useful Kubernetes Networking Commands

1. Check Nodes:

   kubectl get nodes -o wide

2. Check Pod IP Addresses:

   kubectl get pods -o wide

3. Check Services:

   kubectl get svc

4. Check Service Endpoints:

   kubectl get endpointslice

5. Check NetworkPolicies:

   kubectl get networkpolicy

6. Describe a NetworkPolicy:

   kubectl describe networkpolicy <policy-name>

7. Check CoreDNS Pods:

   kubectl get pods -n kube-system

8. Check Pod DNS Configuration:

   kubectl exec -it <pod-name> -- cat /etc/resolv.conf

9. Test DNS Resolution:

   kubectl exec -it <pod-name> -- nslookup kubernetes.default

10. Test HTTP Connectivity:

    kubectl exec -it <pod-name> -- wget -qO- http://<service-name>

---

## 14. Important Points to Remember

- Kubernetes uses a flat Pod networking model.
- Every Pod receives its own IP address.
- Containers inside the same Pod share a network namespace.
- Containers in the same Pod can communicate through localhost.
- OCI defines standards for container images and runtimes.
- CRI connects kubelet with container runtimes.
- CNI is used to configure container networking.
- CNI networking solutions enable Pod-to-Pod communication.
- veth pairs are commonly used to connect Pod network namespaces to host networking.
- Pods on the same Node communicate through the Node's networking implementation.
- Pods on different Nodes communicate through the underlying network using routing or encapsulation.
- VXLAN is one possible overlay networking technology.
- Kubernetes Services provide stable access to Pods.
- CoreDNS provides DNS-based service discovery.
- NetworkPolicy controls allowed network traffic when supported by the networking solution.

---

## 15. Interview Questions

1. What is Kubernetes Networking?
2. Explain the Kubernetes networking model.
3. What is OCI?
4. What is the difference between OCI, CRI, and CNI?
5. What is a CNI plugin?
6. Name some popular CNI plugins.
7. What is a network namespace?
8. Why do containers inside the same Pod share an IP address?
9. What is a veth pair?
10. How do Pods communicate on the same Node?
11. How do Pods communicate across different Nodes?
12. What is the difference between overlay networking and direct routing?
13. What is VXLAN?
14. How does a Pod communicate with a Kubernetes Service?
15. What is the role of CoreDNS in Kubernetes networking?
16. What is the purpose of kube-proxy?
17. What is NetworkPolicy?
18. What is the difference between ingress and egress traffic?
19. Can two Pods communicate without a Service?
20. What happens to a Pod IP address when the Pod is recreated?

---

## 16. Practical Assignment

Perform the following tasks:

1. Check all Kubernetes Nodes and their IP addresses.
2. Check the IP addresses of existing Pods.
3. Create two Pods on the same Node.
4. Test communication between the Pods using their IP addresses.
5. Create a ClusterIP Service for one Pod.
6. Access the Service from another Pod using its DNS name.
7. Create two Pods on different worker Nodes.
8. Test cross-node Pod communication.
9. Check the CNI networking solution installed in the cluster.
10. Create a NetworkPolicy and test whether it restricts traffic.

## Conclusion

Kubernetes networking enables communication between containers, Pods, Services, and external networks.

OCI defines standards for container technologies, CRI enables communication between kubelet and container runtimes, and CNI provides the framework used to configure Pod networking.

Understanding network namespaces, veth pairs, routing, overlay networks, CoreDNS, Services, and NetworkPolicies is essential for Kubernetes networking and troubleshooting.