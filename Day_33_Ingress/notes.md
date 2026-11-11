# Kubernetes Ingress, Ingress Controller, LoadBalancer & Egress

# 1. What is Ingress in Kubernetes?

Ingress is a Kubernetes API resource that manages external HTTP and HTTPS access to Services running inside a Kubernetes cluster.

It provides rules to route incoming traffic from outside the cluster to internal Kubernetes Services.

Ingress supports:
- Host-based routing.
- Path-based routing.
- HTTP and HTTPS routing.
- TLS/SSL termination.
- Routing traffic to different Services based on URL paths.

### Example

Suppose we have two applications running inside Kubernetes:

1. Frontend Service
2. Backend Service

We want users to access both applications using a single external IP address.

We can configure Ingress rules:

www.example.com/       → Frontend Service

www.example.com/api    → Backend Service

Ingress routes incoming HTTP requests to the appropriate Service based on the configured rules.

### Why Do We Need Ingress?

Without Ingress, we might expose multiple applications using separate NodePort or LoadBalancer Services.

For example:

Frontend → LoadBalancer IP 1

Backend → LoadBalancer IP 2

With Ingress, we can route multiple applications through a common entry point.

Benefits:
- Reduces the need for multiple external load balancers.
- Supports host-based routing.
- Supports path-based routing.
- Provides centralized HTTP/HTTPS routing.
- Supports TLS termination.

Important:

Ingress itself does not handle traffic. It only defines routing rules. An Ingress Controller is required to implement those rules.

---

# 2. What is an Ingress Controller?

An Ingress Controller is a component that watches Kubernetes Ingress resources and configures an HTTP/HTTPS proxy or load balancer to route traffic according to the defined rules.

In simple words:

Ingress = Routing Rules

Ingress Controller = Executes Routing Rules

### How Does an Ingress Controller Work?

1. A user creates an Ingress resource.
2. The Ingress resource is stored in the Kubernetes API Server.
3. The Ingress Controller watches the Kubernetes API.
4. It detects the newly created Ingress resource.
5. It reads the routing rules.
6. It configures its proxy to handle incoming traffic.
7. When a request arrives, it routes traffic to the appropriate Kubernetes Service.
8. The Service forwards traffic to one of its backend Pods.

### Popular Ingress Controllers

1. NGINX Ingress Controller
   - Uses NGINX as the reverse proxy.
   - Supports host-based and path-based routing.
   - Supports TLS termination.

2. Traefik
   - Provides dynamic routing.
   - Supports Kubernetes Ingress resources.

3. HAProxy Ingress
   - Uses HAProxy for traffic routing and load balancing.

4. Contour
   - Uses Envoy as its proxy.

5. Kong Ingress Controller
   - Provides API gateway and ingress capabilities.

Note:

Ingress controllers are separate software components that must be installed and configured in the cluster. Creating an Ingress resource alone does not make an application accessible.

---

# 3. How Ingress Works in Kubernetes

Let's understand the complete traffic flow.

Suppose we have:

Application: Web

Deployment: web

Service: web

Ingress Host: hello-world.exposed

### Architecture

User Browser
     |
     | HTTP Request
     | http://hello-world.exposed
     |
     v
DNS Resolution
     |
     v
Ingress Controller
     |
     | Checks Ingress Rules
     |
     v
Kubernetes Service
     |
     v
Backend Pod
     |
     v
Application Response
     |
     v
Hello World

### Step-by-Step Explanation

1. The user enters http://hello-world.exposed in the browser.
2. DNS resolves hello-world.exposed to the IP address configured for the Ingress entry point.
3. The request reaches the Ingress Controller.
4. The Ingress Controller checks the HTTP Host header and URL path.
5. It identifies the matching Ingress rule.
6. It forwards the request to the corresponding Kubernetes Service.
7. The Service forwards the request to one of its backend Pods.
8. The application processes the request and returns the response.

---

# 4. Types of Ingress Routing

Ingress mainly supports two common routing approaches.

## A. Host-Based Routing

Host-based routing means routing traffic to different Services based on the domain name.

Example:

app1.example.com → frontend-service

app2.example.com → backend-service

Both applications can be accessed through the same Ingress Controller entry point.

## B. Path-Based Routing

Path-based routing means routing traffic based on the URL path.

Example:

example.com/       → frontend-service

example.com/api    → backend-service

example.com/admin  → admin-service

The Ingress Controller checks the requested URL path and forwards the request to the appropriate Service.

---

# 5. What is a LoadBalancer in Kubernetes?

A LoadBalancer is a Kubernetes Service type that exposes an application externally using a load balancer provided by the underlying infrastructure.

It distributes incoming traffic across the backend Pods selected by the Service.

Example:

Frontend Deployment
       |
       v
Frontend Service
(Type: LoadBalancer)
       |
       v
External Load Balancer
       |
       v
Users

### How Does LoadBalancer Work?

1. Create a Service with type LoadBalancer.
2. Kubernetes requests an external load balancer from the infrastructure integration.
3. The infrastructure provisions or configures a load balancer if supported.
4. The load balancer receives external traffic.
5. Traffic is forwarded to the Kubernetes Service.
6. The Service routes traffic to backend Pods.

### Example YAML

apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP

### Create the Service

kubectl apply -f loadbalancer.yaml

Check the Service:

kubectl get svc

Example output:

NAME          TYPE           CLUSTER-IP     EXTERNAL-IP
web-service   LoadBalancer   10.96.0.20    20.x.x.x

Note:

The external IP is assigned only when a supported load balancer implementation is available.

In local clusters such as kind, a LoadBalancer Service may remain in Pending state unless an additional solution such as MetalLB or another load balancer integration is configured.

---

# 6. Difference Between ClusterIP, NodePort, LoadBalancer and Ingress

| Feature | ClusterIP | NodePort | LoadBalancer | Ingress |
|---------|-----------|----------|-------------|---------|
| Access | Internal | External through Node IP | External | HTTP/HTTPS external routing |
| IP | Cluster IP | Node IP | External LB IP | Controller entry point |
| Port | Service port | Node port | Service port | Usually 80/443 |
| Use case | Internal communication | Simple external access | External exposure | Host/path routing |
| Requires controller | No | No | Infrastructure integration | Yes |

Important:

Ingress is not a Service type. It is a separate Kubernetes API resource.

Ingress Controller traffic is commonly exposed through a NodePort or LoadBalancer Service.

---

# 7. What is Egress in Kubernetes?

Egress refers to outgoing network traffic from a Pod or application toward another destination.

Examples:

- A Pod accessing Google.
- A Pod connecting to an external database.
- A Pod downloading packages from the Internet.
- A Pod communicating with an external API.

### Ingress vs Egress

Ingress = Incoming traffic

Egress = Outgoing traffic

Example:

User → Kubernetes Application = Ingress

Kubernetes Application → External API = Egress

### Egress Traffic Flow

Kubernetes Pod
      |
      | Outgoing Request
      |
      v
Node Networking
      |
      v
Cluster Network / NAT (if required)
      |
      v
Internet
      |
      v
External Server

### How Does Egress Work?

1. An application inside a Pod sends an outgoing request.
2. The packet leaves the Pod through its network interface.
3. The CNI networking implementation handles the packet.
4. The packet is routed through the Node or cluster network.
5. Source NAT may be applied depending on the networking configuration.
6. The request reaches the external destination.
7. The response returns through the network to the Pod.

Note:

The exact egress path depends on the CNI plugin, routing configuration, cloud infrastructure, and whether NAT is required.

---

# 8. Egress NetworkPolicy

Kubernetes NetworkPolicy can control outgoing traffic from selected Pods.

For example, we may want to allow a Pod to communicate only with a specific destination.

### Example: Allow Egress Traffic on Port 443

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-https
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: TCP
      port: 443

Explanation:

1. podSelector selects Pods with the label app=frontend.
2. policyTypes specifies Egress.
3. egress defines the allowed outgoing traffic.
4. Only TCP traffic on port 443 is allowed by this policy.

Important:

Once a Pod is selected by an Egress policy, its outgoing traffic is restricted to what the applicable policies allow.

Also, DNS traffic may need to be explicitly allowed when egress policies restrict all outgoing traffic. Otherwise, the Pod may be unable to resolve domain names.

NetworkPolicy enforcement depends on the installed networking plugin supporting it.

---

# 9. Ingress vs Egress

| Ingress | Egress |
|---------|--------|
| Incoming network traffic | Outgoing network traffic |
| Traffic enters a Pod or application | Traffic leaves a Pod or application |
| Example: User accessing a website | Example: Pod calling an external API |
| Controlled through ingress rules in NetworkPolicy | Controlled through egress rules in NetworkPolicy |

---

# 10. Practical Assignment – Create an Ingress in Kubernetes

## Task Details

Namespace: default

Application Image: username/nodejs-hello-world:1.0.0

Deployment Name: web

Replicas: 1

Container Port: 3000

Service Name: web

Service Type: ClusterIP

Ingress Host: hello-world.exposed

Ingress Path: /

Expected Response:

Hello World

### Step 1: Verify Ingress Controller

Before creating an Ingress, make sure an Ingress Controller is running.

Check Pods:

kubectl get pods -A

Check Services:

kubectl get svc -A

If using NGINX Ingress Controller, you can check:

kubectl get pods -n ingress-nginx

Check its Services:

kubectl get svc -n ingress-nginx

Make sure the controller is running and its entry point is accessible.

Note:

For kind, the Ingress Controller must also be configured so that traffic from your host machine can reach it. This commonly involves host port mappings and suitable NodePort or hostPort configuration.

### Step 2: Create a Deployment

Create a file named web-deployment.yaml.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: username/nodejs-hello-world:1.0.0
        ports:
        - containerPort: 3000
          protocol: TCP

Apply the configuration:

kubectl apply -f web-deployment.yaml

Check Deployment:

kubectl get deployment web

Check Pods:

kubectl get pods -o wide

### Step 3: Expose Deployment Using ClusterIP Service

Create a file named web-service.yaml.

apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP

Explanation:

- type: ClusterIP means the Service is accessible internally within the cluster.
- selector identifies Pods having the label app=web.
- port: 80 is the Service port.
- targetPort: 3000 is the application container port.

Apply:

kubectl apply -f web-service.yaml

Check Service:

kubectl get svc web

Check Endpoints:

kubectl get endpoints web

### Step 4: Test the Application

First, verify that the application responds on port 3000 inside the Pod.

Get Pod name:

kubectl get pods

Test using port forwarding:

kubectl port-forward deployment/web 3000:3000

Open another terminal and execute:

curl http://localhost:3000/

Expected Output:

Hello World

You can also test the Service internally using a temporary BusyBox Pod:

kubectl run test-pod --image=busybox:1.36 --restart=Never -it --rm -- wget -qO- http://web/

Expected Output:

Hello World

### Step 5: Create an Ingress Resource

Create a file named web-ingress.yaml.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
spec:
  rules:
  - host: hello-world.exposed
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80

Explanation:

- apiVersion: networking.k8s.io/v1 defines the Ingress API version.
- kind: Ingress specifies the resource type.
- name: web-ingress is the Ingress resource name.
- host: hello-world.exposed defines the hostname.
- path: / matches requests starting with /.
- pathType: Prefix specifies prefix-based path matching.
- service.name: web identifies the backend Service.
- service.port.number: 80 identifies the Service port.

Apply:

kubectl apply -f web-ingress.yaml

### Step 6: List Ingress Resources

Run:

kubectl get ingress

Or:

kubectl get ingress -o wide

Describe the Ingress:

kubectl describe ingress web-ingress

Check its ADDRESS field and events.

### Step 7: Add Entry in Hosts File

We need to map hello-world.exposed to the IP address through which the Ingress Controller can be accessed.

First, identify the correct virtual node or Ingress Controller entry-point IP.

For example:

192.168.1.100 hello-world.exposed

On Linux, edit:

sudo nano /etc/hosts

Add:

<VIRTUAL_NODE_IP> hello-world.exposed

On Windows, since we are using PowerShell, edit the hosts file at:

C:\Windows\System32\drivers\etc\hosts

Open Notepad as Administrator and open the hosts file.

Add:

<VIRTUAL_NODE_IP> hello-world.exposed

Replace <VIRTUAL_NODE_IP> with the actual IP address used to access the Ingress Controller.

Important:

Do not blindly use the Pod IP or Service ClusterIP. The hosts entry must point to an address that your machine can reach for the Ingress Controller.

### Step 8: Test the Ingress

Run:

curl http://hello-world.exposed/

Expected Output:

Hello World

You can also open the following URL in your browser:

http://hello-world.exposed

Expected Result:

Hello World

---

# 11. Complete Architecture of the Ingress Assignment

User Browser
     |
     | http://hello-world.exposed/
     |
     v
Hosts File
     |
     | Resolves hostname to Ingress entry-point IP
     |
     v
Ingress Controller
     |
     | Matches Host and Path Rules
     |
     v
Ingress Resource
     |
     v
ClusterIP Service (web)
     |
     | Port 80 → Target Port 3000
     |
     v
Web Pod
     |
     v
Node.js Application
     |
     v
Response: Hello World

---

# 12. Common Ingress Troubleshooting

### Problem 1: Ingress ADDRESS is Empty

Check whether the Ingress Controller is running:

kubectl get pods -A

Check Ingress:

kubectl describe ingress web-ingress

An empty ADDRESS does not always mean the Ingress is broken. Some local controllers and kind configurations do not automatically populate the ADDRESS field.

### Problem 2: 404 Not Found

Possible reasons:

- Hostname does not match.
- Ingress path is incorrect.
- Ingress Controller is not processing the Ingress.
- Ingress class configuration is incorrect.

Check:

kubectl describe ingress web-ingress

### Problem 3: 502 Bad Gateway or 503 Service Unavailable

Possible reasons:

- Backend Pod is not running.
- Service selector does not match Pod labels.
- Service targetPort is incorrect.
- Application is not listening on the expected port.

Check:

kubectl get pods -o wide

kubectl get svc web

kubectl get endpoints web

kubectl describe svc web

### Problem 4: Hostname Not Resolving

Check:

nslookup hello-world.exposed

Check the hosts file entry.

Make sure the hostname maps to the correct Ingress entry-point IP.

### Problem 5: Ingress Controller Is Not Running

Check:

kubectl get pods -n ingress-nginx

Check logs:

kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

The exact Deployment name and namespace may differ depending on the installation.

---

# 13. Important Commands – Quick Revision

1. Check all Pods:

   kubectl get pods -A

2. Check Ingress Controllers:

   kubectl get pods -n ingress-nginx

3. Check Ingress Resources:

   kubectl get ingress

4. Describe Ingress:

   kubectl describe ingress web-ingress

5. Check Ingress Services:

   kubectl get svc -n ingress-nginx

6. Check Deployment:

   kubectl get deployment

7. Check Services:

   kubectl get svc

8. Check Endpoints:

   kubectl get endpoints web

9. Check NetworkPolicies:

   kubectl get networkpolicy

10. Test Application:

    curl http://hello-world.exposed/

11. Port Forwarding:

    kubectl port-forward deployment/web 3000:3000

---

# 14. Important Points to Remember

- Ingress manages external HTTP and HTTPS access to Kubernetes Services.
- Ingress is a Kubernetes API resource.
- An Ingress Controller is required to implement Ingress rules.
- Ingress supports host-based and path-based routing.
- NGINX is a popular Ingress Controller.
- Ingress can route multiple applications through one entry point.
- ClusterIP exposes a Service internally.
- NodePort exposes a Service through a port on each Node.
- LoadBalancer exposes a Service through an external load balancer when supported.
- Egress means outgoing network traffic.
- Ingress means incoming network traffic.
- NetworkPolicy can control ingress and egress traffic.
- NetworkPolicy enforcement requires a compatible CNI implementation.
- Ingress is not a replacement for a Kubernetes Service.
- An Ingress Controller commonly routes traffic to Services, not directly to Deployments.

---

# 15. Interview Questions

1. What is Ingress in Kubernetes?
2. Why do we need an Ingress Controller?
3. What is the difference between Ingress and Ingress Controller?
4. Name some popular Ingress Controllers.
5. Explain how Ingress works in Kubernetes.
6. What is host-based routing?
7. What is path-based routing?
8. What is the difference between Ingress and NodePort?
9. What is the difference between Ingress and LoadBalancer?
10. Can multiple applications use the same Ingress Controller?
11. What is TLS termination?
12. What is LoadBalancer in Kubernetes?
13. What happens if an Ingress Controller is not installed?
14. What is Egress in Kubernetes?
15. What is the difference between Ingress and Egress?
16. How does a NetworkPolicy control Egress traffic?
17. Can Ingress route traffic directly to a Pod?
18. What is the purpose of an IngressClass?
19. Why do we need a hosts file entry in this practical assignment?
20. How would you troubleshoot a 502 Bad Gateway error?

---

# 16. Practical Assignment Checklist

1. Verify that an Ingress Controller is running.
2. Create a Deployment named web.
3. Use the image username/nodejs-hello-world:1.0.0.
4. Configure one replica and container port 3000.
5. Create a ClusterIP Service named web.
6. Verify the application returns Hello World.
7. Create an Ingress named web-ingress.
8. Configure hostname hello-world.exposed.
9. Route path / to the web Service on port 80.
10. Verify the Ingress resource.
11. Add the correct Ingress entry-point IP to the hosts file.
12. Access http://hello-world.exposed.
13. Verify that the response is Hello World.

# Conclusion

Ingress provides a way to manage external HTTP and HTTPS access to applications running inside Kubernetes.

The Ingress Controller implements the routing rules, while Kubernetes Services provide stable access to backend Pods.

LoadBalancer exposes applications through external load-balancing infrastructure, and Egress controls or describes outgoing network communication.

Understanding these concepts is important for managing application traffic, exposing applications, and troubleshooting Kubernetes networking.