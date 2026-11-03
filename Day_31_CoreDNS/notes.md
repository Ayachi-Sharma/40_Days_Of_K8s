# CoreDNS in Kubernetes – Complete Documentation

## 1. What is CoreDNS?

CoreDNS is a DNS server used in Kubernetes to provide DNS-based service discovery within the cluster.

It allows Kubernetes Pods to communicate with Services using their names instead of remembering their IP addresses.

For example:

Instead of accessing a Service using its ClusterIP:

http://10.96.0.10

We can access it using its DNS name:

http://nginx.default.svc.cluster.local

CoreDNS automatically resolves the Service name into its corresponding IP address.

### Why Do We Need CoreDNS?

- Provides DNS-based service discovery.
- Allows Pods to communicate using Service names.
- Resolves internal Kubernetes DNS names.
- Forwards external DNS queries to upstream DNS servers.
- Caches DNS responses to improve performance.

---

## 2. How CoreDNS Works in Kubernetes

CoreDNS runs inside the Kubernetes cluster as Pods, usually in the `kube-system` namespace.

It is generally deployed as a Deployment and exposed through a Kubernetes Service named `kube-dns`.

Note: Even though the Service is named `kube-dns`, CoreDNS is the actual DNS implementation running behind it.

### DNS Resolution Flow

Suppose we have two Pods:

- Frontend Pod
- Backend Pod

The frontend application needs to communicate with the backend Service.

Instead of using the backend Service IP address, it uses:

backend.default.svc.cluster.local

The following process takes place:

1. The frontend Pod sends a DNS query for the backend Service.
2. The query is sent to the DNS server IP configured inside the Pod.
3. The DNS server IP belongs to the Kubernetes DNS Service (kube-dns).
4. The kube-dns Service forwards the request to one of the CoreDNS Pods.
5. CoreDNS checks its Kubernetes DNS records.
6. It finds the backend Service and its ClusterIP.
7. CoreDNS returns the ClusterIP to the frontend Pod.
8. The frontend Pod uses that IP address to communicate with the backend Service.

### DNS Resolution Flow Diagram

Frontend Pod
      |
      | DNS Query:
      | backend.default.svc.cluster.local
      |
      v
Pod DNS Configuration
(/etc/resolv.conf)
      |
      v
kube-dns Service
(ClusterIP: 10.96.0.10)
      |
      v
CoreDNS Pod
(kube-system namespace)
      |
      v
Kubernetes API
(Service and Endpoint Information)
      |
      v
CoreDNS resolves the Service name
      |
      v
Returns ClusterIP
      |
      v
Frontend Pod communicates with Backend Service

Note:
The CoreDNS Kubernetes plugin watches Kubernetes resources through the Kubernetes API and maintains DNS information for Services and other supported resources.

---

## 3. Where Does CoreDNS Run?

CoreDNS normally runs inside the `kube-system` namespace.

Check CoreDNS Pods:

kubectl get pods -n kube-system

Example output:

NAME                       READY   STATUS    RESTARTS
coredns-xxxxxxxxxx-xxxxx   1/1     Running   0
coredns-xxxxxxxxxx-yyyyy   1/1     Running   0

Check CoreDNS Deployment:

kubectl get deployment -n kube-system

Get detailed information:

kubectl describe deployment coredns -n kube-system

Check the DNS Service:

kubectl get svc -n kube-system

You will find a Service named:

kube-dns

Get its details:

kubectl describe svc kube-dns -n kube-system

Important:

- CoreDNS Pods run in the kube-system namespace.
- CoreDNS is generally managed through a Deployment.
- The kube-dns Service provides a stable ClusterIP for DNS queries.
- The Service forwards DNS traffic to the CoreDNS Pods.

---

## 4. CoreDNS ConfigMap

CoreDNS uses a ConfigMap to store its configuration.

The ConfigMap is generally named `coredns` and is located in the `kube-system` namespace.

Check the ConfigMap:

kubectl get configmap -n kube-system

View CoreDNS configuration:

kubectl get configmap coredns -n kube-system -o yaml

Edit the ConfigMap:

kubectl edit configmap coredns -n kube-system

The ConfigMap contains a configuration file called `Corefile`.

---

## 5. What is Corefile?

Corefile is the main configuration file used by CoreDNS.

It defines:
- Which DNS zones CoreDNS should handle.
- Which plugins should be enabled.
- How DNS queries should be processed.
- Where external DNS queries should be forwarded.

### Default CoreDNS Corefile

A typical Kubernetes CoreDNS configuration looks like this:

.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

### Explanation of CoreDNS Plugins

1. errors

   Logs DNS errors to standard output.

2. health

   Provides a health-check endpoint for CoreDNS.

   Default endpoint:

   http://localhost:8080/health

3. ready

   Provides a readiness endpoint to check whether CoreDNS plugins are ready.

   Default endpoint:

   http://localhost:8181/ready

4. kubernetes

   Enables Kubernetes DNS-based service discovery.

   It allows CoreDNS to resolve Kubernetes Services and supported Pod DNS records.

   Example:

   backend.default.svc.cluster.local

5. prometheus

   Exposes CoreDNS metrics for Prometheus monitoring.

   Default metrics endpoint:

   :9153

6. forward

   Forwards DNS queries to an upstream DNS server.

   Example:

   forward . /etc/resolv.conf

   This forwards queries that are not handled by the Kubernetes plugin to the DNS servers configured in /etc/resolv.conf.

7. cache

   Enables DNS response caching.

   Example:

   cache 30

   It configures the cache plugin with a 30-second default cache duration for its entries, subject to the plugin's cache behavior and record TTLs.

8. loop

   Detects simple forwarding loops in DNS configuration.

9. reload

   Automatically reloads the Corefile when its configuration changes.

10. loadbalance

    Randomizes the order of records in DNS responses to distribute traffic among multiple available addresses.

---

## 6. Kubernetes DNS Naming Convention

Kubernetes creates DNS records for Services.

The standard DNS format for a Service is:

<service-name>.<namespace>.svc.cluster.local

Explanation:

- service-name: Name of the Kubernetes Service.
- namespace: Namespace where the Service exists.
- svc: Indicates that it is a Service.
- cluster.local: Default Kubernetes cluster domain.

Example:

nginx.default.svc.cluster.local

Here:

- nginx = Service name
- default = Namespace
- svc = Service identifier
- cluster.local = Cluster domain

### Different Ways to Access a Service

Suppose we have:

Service Name: backend
Namespace: default

We can access it using:

1. backend
2. backend.default
3. backend.default.svc
4. backend.default.svc.cluster.local

Short names work because Kubernetes configures DNS search domains inside Pods.

For Services in another namespace, specify the namespace:

backend.prod.svc.cluster.local

---

## 7. DNS Resolution Between Namespaces

Kubernetes DNS allows Services to be discovered across different namespaces.

Example:

We have two namespaces:

- frontend
- backend

A Service named `api` is running in the backend namespace.

If a Pod in the frontend namespace tries:

nslookup api

It may not find the Service because the short name is searched in the frontend namespace.

Instead, use:

nslookup api.backend

Or use the complete DNS name:

api.backend.svc.cluster.local

This allows applications to communicate with Services across namespaces.

---

## 8. CoreDNS Caching

CoreDNS supports DNS caching through its cache plugin.

Example:

cache 30

This enables caching of DNS responses and configures a 30-second default TTL for cache entries.

### How Does CoreDNS Caching Work?

1. A Pod sends a DNS query for a Service.
2. CoreDNS receives the query.
3. If the answer is available in its cache, CoreDNS returns the cached response.
4. Otherwise, CoreDNS resolves the query using its Kubernetes plugin or forwards it to an upstream DNS server.
5. The response is cached according to the cache configuration and applicable TTL.
6. Subsequent queries can use the cached response.

### Benefits

- Reduces DNS lookup latency.
- Reduces repeated DNS queries.
- Improves application performance.
- Reduces unnecessary DNS traffic.

---

## 9. CoreDNS and External DNS Resolution

CoreDNS also helps Kubernetes Pods resolve external domain names.

For example, a Pod wants to access:

www.google.com

The process is:

1. The Pod sends a DNS query to the kube-dns Service.
2. CoreDNS receives the query.
3. CoreDNS checks whether it can resolve the name using its Kubernetes DNS plugin.
4. Since google.com is an external domain, the query is forwarded using the forward plugin.
5. The upstream DNS server resolves the domain.
6. CoreDNS returns the IP address to the Pod.

The default Corefile configuration uses:

forward . /etc/resolv.conf

This means CoreDNS forwards applicable external queries to the DNS servers configured in its own /etc/resolv.conf.

---

## 10. Pod DNS Configuration

Every Kubernetes Pod receives DNS configuration when it is created.

Check DNS configuration inside a Pod:

kubectl exec -it <pod-name> -- cat /etc/resolv.conf

Example:

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

Explanation:

1. nameserver

   Contains the IP address of the Kubernetes DNS Service.

2. search

   Contains DNS search domains that help resolve short Service names.

3. options

   Contains DNS resolver options.

### DNS Policy in Kubernetes

Kubernetes supports four DNS policies.

1. ClusterFirst

   This is the default DNS policy.

   Pod queries for cluster DNS names are sent to the cluster DNS server.

   External queries that are not resolved by cluster DNS are forwarded according to the CoreDNS configuration.

2. Default

   The Pod inherits DNS configuration from the node where it is running.

3. ClusterFirstWithHostNet

   Used for Pods running with hostNetwork: true.

   It allows these Pods to use cluster DNS.

4. None

   The Pod ignores the DNS settings normally provided by Kubernetes.

   DNS configuration must be supplied using dnsConfig.

---

## 11. Practical: Testing CoreDNS in Kubernetes

Let's create a test Pod and verify DNS resolution.

### Step 1: Check CoreDNS Pods

kubectl get pods -n kube-system

Make sure CoreDNS Pods are running.

### Step 2: Check DNS Service

kubectl get svc kube-dns -n kube-system

Note the ClusterIP.

### Step 3: Create a BusyBox Pod

Create a YAML file named dns-test.yaml.

apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  containers:
  - name: dns-test
    image: busybox:1.36
    command:
    - sleep
    - "3600"
  restartPolicy: Never

Create the Pod:

kubectl apply -f dns-test.yaml

Check its status:

kubectl get pods

### Step 4: Check DNS Configuration

kubectl exec -it dns-test -- cat /etc/resolv.conf

Observe the nameserver and search domains.

### Step 5: Test Kubernetes DNS

Run:

kubectl exec -it dns-test -- nslookup kubernetes.default

This should resolve the Kubernetes Service.

### Step 6: Test External DNS

Run:

kubectl exec -it dns-test -- nslookup google.com

This tests external DNS resolution through CoreDNS and its upstream resolver.

### Step 7: Check CoreDNS Logs

kubectl logs -n kube-system deployment/coredns

This helps identify DNS-related errors.

Note:
If you want to see individual DNS queries in CoreDNS logs, you can temporarily enable the `log` plugin in the Corefile. It is not enabled by default in the typical configuration.

---

## 12. Troubleshooting CoreDNS

If Pods cannot resolve Service names or external domains, check the following.

### 1. Check CoreDNS Pods

kubectl get pods -n kube-system

If Pods are not Running, inspect their events and logs.

### 2. Check CoreDNS Service

kubectl get svc kube-dns -n kube-system

Verify that the Service exists and has a ClusterIP.

### 3. Check CoreDNS Endpoints

kubectl get endpointslice -n kube-system -l kubernetes.io/service-name=kube-dns

Verify that DNS endpoints are available.

### 4. Check CoreDNS Logs

kubectl logs -n kube-system deployment/coredns

Look for errors related to DNS queries, plugins, or upstream resolvers.

### 5. Check Pod DNS Configuration

kubectl exec -it <pod-name> -- cat /etc/resolv.conf

Verify that the nameserver points to the expected cluster DNS IP.

### 6. Check CoreDNS ConfigMap

kubectl get configmap coredns -n kube-system -o yaml

Verify that the Corefile configuration is correct.

### 7. Test DNS Resolution

kubectl exec -it <pod-name> -- nslookup kubernetes.default

If this fails, investigate CoreDNS availability, Service endpoints, network connectivity, and DNS configuration.

---

## 13. Important Commands – Quick Revision

1. Get CoreDNS Pods:

   kubectl get pods -n kube-system

2. Get CoreDNS Deployment:

   kubectl get deployment -n kube-system

3. Get DNS Service:

   kubectl get svc kube-dns -n kube-system

4. Describe DNS Service:

   kubectl describe svc kube-dns -n kube-system

5. Get CoreDNS ConfigMap:

   kubectl get configmap coredns -n kube-system -o yaml

6. Edit CoreDNS ConfigMap:

   kubectl edit configmap coredns -n kube-system

7. Check CoreDNS Logs:

   kubectl logs -n kube-system deployment/coredns

8. Check DNS Endpoints:

   kubectl get endpointslice -n kube-system -l kubernetes.io/service-name=kube-dns

9. Check Pod DNS Configuration:

   kubectl exec -it <pod-name> -- cat /etc/resolv.conf

10. Test Kubernetes DNS:

    kubectl exec -it <pod-name> -- nslookup kubernetes.default

11. Test External DNS:

    kubectl exec -it <pod-name> -- nslookup google.com

---

## 14. Important Points to Remember

- CoreDNS is the DNS solution commonly used in Kubernetes.
- CoreDNS runs inside the kube-system namespace.
- CoreDNS is generally deployed using a Deployment.
- The Kubernetes DNS Service is named kube-dns.
- CoreDNS uses a ConfigMap named coredns.
- Corefile is the main configuration file of CoreDNS.
- The Kubernetes plugin enables Kubernetes DNS-based service discovery.
- The forward plugin forwards external DNS queries to upstream DNS servers.
- The cache plugin improves DNS resolution performance.
- The default Kubernetes cluster domain is cluster.local.
- Service DNS format is service-name.namespace.svc.cluster.local.
- Every Pod receives DNS configuration through /etc/resolv.conf.
- ClusterFirst is the default Pod DNS policy.

---

## 15. Interview Questions

1. What is CoreDNS in Kubernetes?
2. Why do we need CoreDNS?
3. In which namespace does CoreDNS run?
4. What is the purpose of the kube-dns Service?
5. What is a CoreDNS ConfigMap?
6. What is a Corefile?
7. Explain the working of CoreDNS in Kubernetes.
8. How does a Pod communicate with another Service using DNS?
9. What is the difference between CoreDNS and kube-dns?
10. What is the purpose of the Kubernetes plugin?
11. What is the purpose of the forward plugin?
12. How does CoreDNS caching work?
13. What is the default Kubernetes cluster domain?
14. What is the DNS naming convention for Kubernetes Services?
15. How can you troubleshoot DNS resolution issues in Kubernetes?
16. What is the purpose of /etc/resolv.conf inside a Pod?
17. What are the different DNS policies available in Kubernetes?

---

## 16. Practical Assignment

Perform the following tasks:

1. Check CoreDNS Pods in the kube-system namespace.
2. Check the kube-dns Service and its ClusterIP.
3. Inspect the CoreDNS ConfigMap.
4. Understand the different plugins configured in the Corefile.
5. Create a BusyBox Pod.
6. Check the Pod's /etc/resolv.conf file.
7. Resolve kubernetes.default using nslookup.
8. Resolve google.com using nslookup.
9. Check CoreDNS logs.
10. Investigate what happens when CoreDNS Pods are unavailable.

**Conclusion:**

CoreDNS is an important component of Kubernetes networking. It provides service discovery and DNS resolution, allowing applications running inside Pods to communicate with Services using DNS names instead of hardcoded IP addresses.