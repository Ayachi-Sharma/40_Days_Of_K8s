### Day 22 – Kubernetes Authentication and Authorization

1. Introduction
Kubernetes uses Authentication and Authorization to secure communication with the Kubernetes API Server.
- Authentication: Verifies who you are.
- Authorization: Determines what actions you are allowed to perform.
Whenever a user runs a kubectl command, the request goes to the Kubernetes API Server, where authentication and authorization checks take place.

2. Authentication – Verifying User Identity
Authentication is the process of verifying the identity of a user or client accessing the Kubernetes cluster.

Kubernetes supports different authentication methods, such as:
- Client certificates
- Bearer tokens
- Username and password through configured authentication mechanisms
- External authentication providers

Kubeconfig
Kubeconfig is a configuration file used by kubectl to connect and authenticate with a Kubernetes cluster.

By default, kubectl uses:
$HOME/.kube/config


It contains three important sections:
1. Clusters: Information about the Kubernetes API Server.
2. Users: Authentication credentials, such as certificates or tokens.
3. Contexts: Combines a cluster, user, and namespace.
To view the current configuration:
kubectl config view


To check the current context:
kubectl config current-context


To execute a command using a specific kubeconfig file:
kubectl get pods --kubeconfig config


If the default kubeconfig is configured, we can simply execute:
kubectl get pods


Making API Calls Using Certificates

We can also communicate directly with the Kubernetes API Server by providing authentication and certificate details.

```powershell
kubectl get --raw /api/v1/namespaces/default/pods \
  --server https://localhost:64418 \
  --client-key adam.key \
  --client-certificate adam.crt \
  --certificate-authority ca.crt
```

Explanation:
- --server: Specifies the Kubernetes API Server address.
- --client-key: Specifies the private key of the client.
- --client-certificate: Specifies the client's certificate.
- --certificate-authority: Specifies the CA certificate used to verify the API Server.
This allows kubectl to authenticate directly using client certificates.

3. Authorization – Controlling Access
After authentication, Kubernetes checks whether the authenticated user has permission to perform the requested operation.

Kubernetes supports different authorization mechanisms.
Authorization Modes
1. Node Authorizer
- Controls permissions for kubelets running on Kubernetes nodes.
- Allows nodes to access resources required for their operation.
2. ABAC (Attribute-Based Access Control)
- Grants permissions based on user attributes and policies.
- Can become complex to manage in larger clusters.
3. RBAC (Role-Based Access Control)
- Grants permissions based on roles assigned to users or groups.
- Uses Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings.
- It is the commonly recommended approach for managing Kubernetes access.
4. Webhook Authorization
- Delegates authorization decisions to an external HTTP service.
- Useful when integrating external authorization systems.
4. Authorization Modes Configuration
The Kubernetes API Server supports different authorization modes through the --authorization-mode flag.

Example:
--authorization-mode=Node,RBAC,Webhook


The API Server checks the configured authorization mechanisms in sequence.
- If an authorizer allows the request, access is granted.
- If an authorizer denies the request, access is denied.
- If an authorizer returns no opinion, the next configured authorizer is checked.
- If all authorizers return no opinion, access is denied.
Note: AlwaysAllow and AlwaysDeny are also available, mainly for specific testing or configuration scenarios.

5. Authentication vs Authorization
Authentication	Authorization
Verifies user identity	Checks user permissions
Determines who you are	Determines what you can do
Uses certificates, tokens, etc.	Uses RBAC, Node, Webhook, etc.
Happens first	Happens after authentication

6. Important Commands
kubectl config view	                    Displays kubeconfig configuration
kubectl config current-context	        Displays the current context
kubectl config get-contexts	            Lists available contexts
kubectl auth can-i get pods	            Checks whether the current user can get pods
kubectl auth can-i create deployments	Checks deployment creation permissions
kubectl get pods --kubeconfig config	Uses a specific kubeconfig file

7. Key Takeaways
- Authentication verifies the identity of a user accessing the Kubernetes cluster.
- Authorization determines which resources and operations the user can access.
- Kubeconfig stores cluster, user, and context information.
- The API Server handles authentication and authorization checks.
- RBAC is the commonly recommended mechanism for managing user permissions.
- Authorization mechanisms can be configured using the API Server's --authorization-mode flag.