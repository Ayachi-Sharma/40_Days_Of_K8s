### Day 23 – Kubernetes RBAC (Role-Based Access Control)

1. Role-Based Access Control

Step 1: Switch to the krishna context

First, check the available contexts:
```powershell
kubectl config get-contexts
```

Switch to the krishna context:
```powershell
kubectl config use-context krishna
```

Verify the current context:
```powershell
kubectl config current-context
```

Step 2: Try Creating a Pod
Create a Pod using the nginx image:
```powershell
kubectl run nginx --image=nginx
```

Expected output:
The request should be denied with a Forbidden error because the user krishna does not have permission to create Pods by default.

Example:
Error from server (Forbidden):
pods is forbidden: User "krishna" cannot create resource "pods"
in API group "" in the namespace "default"


Observation: Authentication is successful because Kubernetes recognizes the user krishna, but authorization fails because the user does not have permission to create Pods.

3. Granting Access to the User

Now we will grant limited permissions to krishna using RBAC.

Step 1: Switch Back to Admin Context
Switch back to your original admin context:

```powershell
kubectl config get-contexts
```
```
kubectl config use-context <kind-svc-cluster>
```powershell

*Replace kind-svc-cluster with your actual admin context name if different.*

Verify:
```powershell
kubectl config current-context
```

Important: All Role and RoleBinding creation steps below must be performed using an account with sufficient administrative permissions.

Step 2: Create a Role Named pod-reader

Create a file named pod-reader-role.yaml.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Explanation:
- kind: Role: Creates a namespace-scoped Role.
- apiGroups: [""]: Refers to the core API group where Pods belong.
- resources: ["pods"]: Specifies that permissions apply to Pods.
- verbs: Defines the allowed operations:
  - get: View an individual Pod.
  - watch: Watch changes to Pods.
  - list: List Pods in the namespace.

Create the Role:
```powershell
kubectl apply -f pod-reader-role.yaml
```

Verify:
```powershell
kubectl get role
```

Expected output:
NAME         CREATED AT
pod-reader   ...


`Step 3: Create a RoleBinding`
Now we need to connect the user krishna with the Role pod-reader.

Create a file named pod-reader-binding.yaml.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: krishna
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Explanation:
- kind: RoleBinding: Connects a user with a Role.
- name: read-pods: Name of the RoleBinding.
- subjects: Specifies the user receiving permissions.
- name: krishna: The user to whom permissions are granted.
- roleRef: References the pod-reader Role.

Create the RoleBinding:
```powershell
kubectl apply -f pod-reader-binding.yaml
```

Verify both objects:
```powershell
kubectl get role
kubectl get rolebinding
```

To inspect their configurations:
```powershell
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

4. Verify User Permissions
Step 1: Switch to the krishna Context
```powershell
kubectl config use-context krishna
```

Verify:
```powershell
kubectl config current-context
```

Step 2: Try Creating a Pod

```powershell
kubectl run nginx --image=nginx
```

Expected: Forbidden
The operation should fail because the Role only grants get, watch, and list permissions.

The user does not have permission to create Pods.

Example:
Error from server (Forbidden):
pods is forbidden: User "krishna" cannot create resource "pods"


Step 3: List Pods

```powershell
kubectl get pods
```

Expected: Allowed
The command should execute successfully because the Role grants the list permission on Pods.

It may display:
No resources found in default namespace.


This is not an error. It simply means there are no Pods in the namespace.
Step 4: Try Creating a Deployment
Now try creating a Deployment:
kubectl create deployment nginx-deploy --image=nginx


Expected: Forbidden
The operation should fail because:
- A Deployment is an apps API group resource.
- The Role only grants permissions on Pods.
- The user does not have permission to create Deployments.
Example:
Error from server (Forbidden):
deployments.apps is forbidden:
User "krishna" cannot create resource "deployments"


5. Verify Permissions Using kubectl auth can-i
```poweshell
kubectl auth can-i get pods
kubectl auth can-i list pods
kubectl auth can-i watch pods
kubectl auth can-i create pods
kubectl auth can-i create deployments
```

Expected results:
Operation	          Expected
Get Pods	          yes
List Pods	          yes
Watch Pods	        yes
Create Pods	        no
Create Deployments	no

6. Understanding the RBAC Flow
User: krishna
Authenticated user

RoleBinding: read-pods
Connects krishna to pod-reader

Role: pod-reader
Permissions: get, list, watch Pods

Allowed
Get, List, Watch Pods

Denied
Create Pods and Deployments


7. Key Takeaways
1. Authentication verifies the identity of a user.
2. Authorization determines what actions a user can perform.
3. A Role defines permissions within a particular namespace.
4. A RoleBinding connects a user or group to a Role.
5. RBAC follows the principle of least privilege: users should receive only the permissions they need.
6. Having permission to list Pods does not mean having permission to create Pods.
7. A RoleBinding does not automatically grant permissions to resources outside its referenced Role.