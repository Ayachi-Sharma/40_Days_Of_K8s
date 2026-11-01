# Day 26 – Kubernetes Networking with Calico and NetworkPolicy

## 1. Create a Kind Cluster with Default CNI Disabled

Kind installs its default CNI plugin (kindnet) when creating a cluster.
Since we want to use Calico as our CNI plugin, we need to disable the default CNI.

Create a file named `kind-calico-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker

networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
```

Create the cluster:

```powershell
kind create cluster --name calico-cluster --config kind-calico-config.yaml
```

Verify the cluster:

```powershell
kubectl get nodes
kubectl cluster-info
```

**Observation:**
Initially, the nodes may show `NotReady` because the default CNI has been disabled and no replacement network plugin has been installed.

---

## 2. Install Calico Network Add-on

Calico is a Container Network Interface (CNI) plugin that provides networking between Kubernetes Pods and supports NetworkPolicy enforcement.

Create cluster's yaml with its reference -
`https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind`

Next go to Install ccalico: manifest

### Step 4: Verify Calico Installation

```powershell
kubectl get pods -n calico-system
kubectl get pods -A
kubectl get nodes
```

Wait until the Calico components are running and all nodes become `Ready`.

**Observation:**

Calico provides the networking functionality required for communication between Pods. It also enforces NetworkPolicies to control traffic between them.

---

## 3. Create Three Deployments

We need to create three Deployments:

| Deployment | Image        | Replicas | Container Port |
|------------|--------------|--------- |--------------- |
| frontend   | nginx        | 1        | 80             |
| backend    | nginx        | 1        | 80             |
| db         | mysql:latest | 1        | 3306           |

Create a file named `deployments.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      role: backend
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mysql
  template:
    metadata:
      labels:
        name: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_USER
              value: mysql
            - name: MYSQL_PASSWORD
              value: mysql
            - name: MYSQL_DATABASE
              value: testdb
            - name: MYSQL_ROOT_PASSWORD
              value: verysecure
          ports:
            - name: mysql
              containerPort: 3306
              protocol: TCP
```

Apply the configuration:

```powershell
kubectl apply -f deployments.yaml
```

Verify the Deployments:

```powershell
kubectl get deployments
```

Verify the Pods and their labels:

```powershell
kubectl get pods --show-labels
```

**Observation:**

- frontend Deployment creates one nginx Pod.
- backend Deployment creates one nginx Pod.
- db Deployment creates one MySQL Pod.
- The labels identify the Pods and are used by Services and NetworkPolicies.

---

## 4. Expose Deployments Using NodePort Services

We can create Services using imperative commands.

### Expose frontend
```powershell
kubectl expose deployment frontend --name=frontend --type=NodePort --port=80 --target-port=80
```

### Expose backend
```powershell
kubectl expose deployment backend --name=backend --type=NodePort --port=80 --target-port=80
```

### Expose database
```powershell
kubectl expose deployment db --name=db --type=NodePort --port=3306 --target-port=3306
```

Verify the Services:

```powershell
kubectl get services
```

Example output:

```text
NAME       TYPE       PORT(S)
frontend   NodePort   80:32xxx/TCP
backend    NodePort   80:31xxx/TCP
db         NodePort   3306:30xxx/TCP
```

**Understanding Service ports:**

- `type: NodePort` exposes the Service through a port on each Kubernetes node.
- `port` is the port exposed by the Service.
- `targetPort` is the destination port on the container.
- `nodePort` is the port allocated on the Kubernetes nodes.

**Observation:**

Kubernetes Services provide stable endpoints for accessing applications. Pods can communicate using Service names instead of depending on changing Pod IP addresses.

For example:

- `http://frontend`
- `http://backend`
- `db:3306`

---

## 5. Test Connectivity Between Pods Before Applying NetworkPolicy

Verify that the frontend, backend, and database applications can communicate with each other.

### Step 1: Get the frontend Pod name
```powershell
kubectl get pods
```

### Step 2: Execute into the frontend container
```powershell
kubectl exec -it <frontend-pod-name> -- bash
```

### Step 3: Test frontend-to-backend connectivity
```bash
curl http://backend
```
Expected: The nginx welcome page from the backend application.

### Step 4: Test frontend-to-database connectivity

Install telnet to verify the connection with database
```bash
app-get install telnet
```

```bash
telnet db 3306
```

Expected: Connection successful.

### Step 5: Exit the container

```bash
exit
```

**Observation:**

Before applying the NetworkPolicy, the frontend Pod can access both the backend Service on port 80 and the database Service on port 3306.
This confirms that inter-Pod communication is working before network restrictions are introduced.

## 6. Create a NetworkPolicy to Restrict Database Access

Now we need to implement the following requirement:

**Only the backend Pod should be able to access the database on TCP port 3306.**

The NetworkPolicy will:

- Select the database Pod.
- Allow incoming traffic only from Pods labelled `role: backend`.
- Allow traffic only on TCP port 3306.
- Block other incoming connections to the database.

Create a file named `db-network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
spec:
  podSelector:
    matchLabels:
      name: mysql

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 3306
```

Apply the NetworkPolicy:

```powershell
kubectl apply -f db-network-policy.yaml
```

Verify:

```powershell
kubectl get networkpolicy
```

Describe the policy:

```powershell
kubectl describe networkpolicy db-network-policy
```

## 7. Verify NetworkPolicy Restrictions

After applying the NetworkPolicy, repeat the connectivity tests.

### Test 1: Backend to Database

```powershell
kubectl exec backend-test -- nc -z -w 3 db 3306
```

Expected: Connection successful.

The backend Pod is allowed to access the database on port 3306.

### Test 2: Frontend to Database

```powershell
kubectl exec frontend-test -- nc -z -w 3 db 3306
```

Expected: Connection timeout or failure.

The frontend Pod is blocked because it does not match the allowed source selector.

### Test 3: Backend to Frontend

```powershell
kubectl exec backend-test -- wget -qO- --timeout=5 http://frontend
```

Expected: nginx welcome page.

Communication remains allowed because the NetworkPolicy only restricts incoming traffic to the database.

### Test 4: Frontend to Backend

```powershell
kubectl exec frontend-test -- wget -qO- --timeout=5 http://backend
```

Expected: nginx welcome page.

Communication remains allowed.

**Note:** The BusyBox `nc` command may not display output on success. Check its exit code immediately after running the command:

```powershell
$LASTEXITCODE
```

- `0` means the connection succeeded.
- A non-zero value means the connection failed.

---

## 8. Final Connectivity Results

| Source     | Destination      | Expected Result |
|------------|----------------- |-----------------|
| Frontend   | Backend          | Allowed         |
| Backend    | Frontend         | Allowed         |
| Backend    | Database (3306)  | Allowed         |
| Frontend   | Database (3306)  | Blocked         |

**Final observation:**

After applying the NetworkPolicy, only the backend Pod can access the database on TCP port 3306.

The frontend and backend can still communicate with each other.

This demonstrates how Kubernetes NetworkPolicies can control communication between application components.

---

## 9. Useful Kubernetes Commands

```powershell
# Check current cluster
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check all Pods
kubectl get pods -A

# Check Deployments
kubectl get deployments

# Check Services
kubectl get services

# Check Pod labels
kubectl get pods --show-labels

# Check NetworkPolicies
kubectl get networkpolicy

# Describe NetworkPolicy
kubectl describe networkpolicy db-network-policy

# Delete temporary test Pods
kubectl delete pod frontend-test backend-test
```

---

## 10. Cleanup

Delete the resources created for this practical:

```powershell
kubectl delete networkpolicy db-network-policy
kubectl delete services frontend backend db
kubectl delete deployments frontend backend db
kubectl delete pod frontend-test backend-test
```

Delete the Kind cluster:

```powershell
kind delete cluster --name calico-cluster
```

---

## 11. Key Takeaways

1. **CNI (Container Network Interface):** Provides networking functionality for Kubernetes Pods.
2. **Calico:** Provides Pod networking and NetworkPolicy enforcement.
3. **Deployment:** Manages application Pods and maintains the desired number of replicas.
4. **NodePort Service:** Exposes an application through a port on Kubernetes nodes.
5. **NetworkPolicy:** Controls incoming and outgoing network traffic for selected Pods.
6. **Pod Selector:** Uses labels to identify the Pods to which a NetworkPolicy applies.
7. **Ingress Policy:** Controls incoming traffic to selected Pods.
8. **Default Network Behavior:** Pods are generally allowed to communicate unless a NetworkPolicy isolates them. Once ingress isolation is applied, only explicitly allowed incoming traffic is permitted.

### Conclusion

In this practical, I created a Kind cluster with the default CNI disabled and installed Calico as the networking add-on.

I deployed frontend, backend, and database applications, exposed them using NodePort Services, and tested connectivity between them.

Finally, I implemented a Kubernetes NetworkPolicy that allows only the backend Pod to access the database on TCP port 3306, restricting access from the frontend Pod.