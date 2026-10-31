### Kubernetes RBAC – ClusterRole and ClusterRoleBinding

1. Introduction
In Kubernetes, RBAC (Role-Based Access Control) manages what users and groups are allowed to do inside a cluster.
So far, we have worked with Role and RoleBinding, which grant permissions within a particular namespace.
But what if we want to grant a user permission to access resources across the entire cluster, such as Nodes, PersistentVolumes, or Namespaces?

This is where `ClusterRole` and `ClusterRoleBinding` come into the picture.

2. Namespace-Scoped vs Cluster-Scoped Resources
Kubernetes resources are divided into two categories:

`Namespace-scoped resources`
These resources belong to a particular namespace.

Examples:
- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- PersistentVolumeClaims
A user can be granted permissions to these resources within a specific namespace using a Role and RoleBinding.

`Cluster-scoped resources`
These resources do not belong to any namespace. They exist at the cluster level.

Examples:
- Nodes
- PersistentVolumes
- Namespaces
- CertificateSigningRequests
- StorageClasses

Step 1: Check namespace-scoped resources

```poweshell
kubectl api-resources --namespaced=true
```
This displays all resources that belong to namespaces.

Observation: These resources can be managed within individual namespaces.

Step 2: Check cluster-scoped resources
```poweshell
kubectl api-resources --namespaced=false
```
This displays resources that are not associated with any namespace.
They are resources at cluster level

Observation: These resources exist at the cluster level and cannot be restricted to a particular namespace through a Role.

3. ClusterRole – Granting Cluster-Level Permissions

A ClusterRole defines permissions for Kubernetes resources at the cluster level.
Unlike a Role, a ClusterRole is not restricted to a particular namespace.

For example, we can create a ClusterRole that allows a user to view Nodes.

`Step 1: Create a ClusterRole`

```poweshell
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
```

Explanation:
- create clusterrole: Creates a ClusterRole.
- node-reader: Name of the ClusterRole.
- --verb=get,list,watch: Specifies allowed operations.
- --resource=nodes: Specifies the resource on which permissions apply.

The ClusterRole grants the following permissions:
Verb	    Purpose
get	        View an individual Node
list	    List all Nodes
watch	    Monitor changes to Nodes

Step 2: Verify the ClusterRole
```powershell
kubectl get clusterroles
```

To inspect its permissions:
```poweshell
kubectl describe clusterrole node-reader
```

Expected permissions:
Resources   Non-Resource URLs   Resource Names   Verbs
nodes       []                  []               [get list watch]


4. ClusterRoleBinding – Assigning Permissions to Users

Creating a ClusterRole only defines permissions. It does not automatically grant those permissions to any user.

To assign these permissions, we create a ClusterRoleBinding.

A ClusterRoleBinding connects a user or group to a ClusterRole and grants its permissions across the cluster.

`Step 1: Create a ClusterRoleBinding`
```poweshell
kubectl create clusterrolebinding node-reader-binding --clusterrole=node-reader --user=learner
```

Explanation:
- create clusterrolebinding: Creates a ClusterRoleBinding.
- node-reader-binding: Name of the binding.
- --clusterrole=node-reader: References the ClusterRole.
- --user=learner: Assigns the permissions to the learner user.

Step 2: Verify the ClusterRoleBinding
```powershell
kubectl get clusterrolebindings
```

Inspect the binding:
```powershell
kubectl describe clusterrolebinding node-reader-binding
```

Expected details:
Name:         node-reader-binding
Role:
  Kind:       ClusterRole
  Name:       node-reader
Subjects:
  Kind       Name
  User       learner


5. Testing Cluster-Level Permissions

`Step 1: Switch to learner context`
```powershell
kubectl config use-context learner
```

Verify:
```powershell
kubectl config current-context
```

Step 2: Check Node permissions
```powershell
kubectl auth can-i get nodes
kubectl auth can-i list nodes
kubectl auth can-i watch nodes
```

Expected: yes
The user should now have permission to get, list, and watch Nodes.

Step 3: List Nodes
```powershell
kubectl get nodes
```

This command should now execute successfully.
Why it works?
Previously, our user received a Forbidden error when accessing Nodes because the pod-reader Role only granted permissions on Pods.

Now we have:
1. Created a ClusterRole with Node permissions.
2. Created a ClusterRoleBinding connecting learner to that ClusterRole.
3. Granted the user cluster-level access to Nodes.

Step 4: Check permissions that are not granted
```powershell
kubectl auth can-i create nodes
kubectl auth can-i delete nodes
```

Expected:
no
no

This is because we only granted get, list, and watch permissions.

6. Understanding the Difference
`Feature`	        `Role`	                `ClusterRole`
Scope	            Namespace	            Cluster-level definition
Used for	        Namespace resources	    Namespace and cluster resources
Can access Nodes?	No	                    Yes, if rules permit
Can access Pods?	Yes	                    Yes, if rules permit

`Feature`           `RoleBinding`                  `ClusterRoleBinding`
Connects	        User to Role or ClusterRole	    User to ClusterRole
Permission scope	Specific namespace	            Entire cluster
Used for	        Namespace-level access	        Cluster-wide access

Important concept: A ClusterRole does not automatically grant cluster-wide access just because it is called a ClusterRole. The binding determines the scope.
- ClusterRole + RoleBinding = Permissions restricted to the RoleBinding's namespace for namespaced resources.
- ClusterRole + ClusterRoleBinding = Cluster-wide permissions.


7. Key Takeaways
1. Kubernetes resources can be namespace-scoped or cluster-scoped.
2. kubectl api-resources --namespaced=true lists namespace-scoped resources.
3. kubectl api-resources --namespaced=false lists cluster-scoped resources.
4. A Role grants permissions within a particular namespace.
5. A ClusterRole defines permissions that can cover cluster-scoped and namespace-scoped resources.
6. A RoleBinding connects users or groups to Roles or ClusterRoles.
7. A ClusterRoleBinding grants the associated ClusterRole permissions across the cluster.
8. Always follow the principle of least privilege when assigning permissions.