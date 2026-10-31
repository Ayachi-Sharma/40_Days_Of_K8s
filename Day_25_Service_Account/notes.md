### Day 25 – Kubernetes Service Accounts

1. Introduction
A ServiceAccount in Kubernetes provides an identity to applications, processes, and workloads running inside Pods.

In Kubernetes, there are two main types of accounts:

1. User Accounts
- Used by humans such as Kubernetes administrators, developers, and operators.
- Used when accessing the Kubernetes cluster through tools such as kubectl.
- Example: learner, the user we created during Day 21.

2. Service Accounts
- Used by applications, automated processes, and workloads running inside Pods.
- Allow applications to authenticate with the Kubernetes API Server.
- Can be assigned specific permissions using RBAC.

For example, imagine an application running inside a Pod needs to retrieve information about other Pods from the Kubernetes API Server. We can assign a ServiceAccount to that Pod and grant it the required permissions using RBAC.
Important: A ServiceAccount provides an identity, but it does not automatically grant permissions to access Kubernetes resources. Permissions must be assigned separately.

2. Default ServiceAccount
Kubernetes automatically creates a ServiceAccount named default in every namespace.

For example:
- default namespace has its own default ServiceAccount.
- kube-system namespace has its own default ServiceAccount.
- kube-node-lease namespace has its own default ServiceAccount.

When a Pod is created without specifying a ServiceAccount, Kubernetes automatically assigns the default ServiceAccount of that namespace.

To check ServiceAccounts:
```powershell
kubectl get serviceaccounts
```

To check ServiceAccounts across all namespaces:
```powershell
kubectl get sa -A
```


3. Creating a ServiceAccount

`Step 1: Create a ServiceAccount`
```powershell
kubectl create sa app-sa
```

Step 2: Verify the ServiceAccount
```powershell
kubectl get sa
```

To see its details:
```powershell
kubectl describe sa app-sa
```

To view its YAML configuration:
```powershell
kubectl get sa app-sa -o yaml
```

Remember, creating a ServiceAccount does not automatically grant it access to Pods, Deployments, or other Kubernetes resources.


4. Manually Creating a Long-Lived API Token for a ServiceAccount

By default, modern Kubernetes versions (v1.24+) do not automatically create long-lived Secret-based API tokens for ServiceAccounts.
Instead, Kubernetes uses short-lived, automatically rotated tokens through the TokenRequest API.
However, sometimes we may need a manually created, long-lived API token for a ServiceAccount.

Kubernetes allows us to create a Secret of type: kubernetes.io/service-account-token
This Secret is associated with a ServiceAccount and Kubernetes populates it with an authentication token.

`Creating a Long-Lived API Token`

Step 1: Create a Secret YAML file

Create a file named sa-token.yaml.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-sa-token
  annotations:
    kubernetes.io/service-account.name: app-sa
type: kubernetes.io/service-account-token
```

Explanation:
- apiVersion: v1: Specifies the core Kubernetes API.
- kind: Secret: Creates a Kubernetes Secret.
- metadata.name: Name of the Secret.
- annotations: Associates the Secret with a ServiceAccount.
- kubernetes.io/service-account.name: Specifies the ServiceAccount name.
- type: Specifies that this Secret stores a ServiceAccount token.

Step 2: Create the Secret
```powershell
kubectl apply -f sa-token.yaml
```

Expected:
secret/app-sa-token created


Step 3: Verify the Secret
```powershell
kubectl get secrets
```

The Secret initially may not contain the token immediately because Kubernetes needs time to populate it.

Check its details:
```powershell
kubectl describe secret app-sa-token
```

You should eventually see:
Type:  kubernetes.io/service-account-token

Data
====
ca.crt:      ...
namespace:   ...
token:       ...


5. Adding an ImagePullSecret to a ServiceAccount

An ImagePullSecret is a Kubernetes Secret that stores authentication credentials required to pull container images from a private container registry.

For example, if an application image is stored in a private Docker Hub repository, Kubernetes needs registry credentials to download that image.

Instead of specifying the image pull credentials separately in every Pod, we can attach an ImagePullSecret to a ServiceAccount.

Whenever a new Pod uses that ServiceAccount, Kubernetes can automatically populate the Pod's imagePullSecrets field.

Step 1: Create an ImagePullSecret

kubectl create secret docker-registry myregistrykey `
  --docker-server=https://index.docker.io/v1/ `
  --docker-username=<YOUR_DOCKER_USERNAME> `
  --docker-password=<YOUR_DOCKER_ACCESS_TOKEN>

Replace the placeholders with your actual private registry credentials. Do not use angle-bracket placeholders literally.

Explanation:
- create secret docker-registry: Creates a registry authentication Secret.
- myregistrykey: Name of the Secret.
- --docker-server: Registry server address.
- --docker-username: Registry username.
- --docker-password: Registry password or access token.

Security note: Prefer a Docker access token over your account password. Avoid committing registry credentials to GitHub or sharing them in screenshots.

Step 2: Verify the Secret
```powershell
kubectl get secrets
```

Check its details:
```powershell
kubectl describe secret myregistrykey
```
Expected type:
TYPE: kubernetes.io/dockerconfigjson


Step 3: Attach ImagePullSecret to the ServiceAccount

kubectl patch serviceaccount app-sa `
  -p '{"imagePullSecrets":[{"name":"myregistrykey"}]}'


Explanation:
- patch serviceaccount: Updates an existing ServiceAccount.
- app-sa: Name of our ServiceAccount.
- imagePullSecrets: Specifies the registry authentication Secret.
- myregistrykey: Name of the Secret to be attached.

*If the above command does not work in powershell*
Option 2:
`Variable created`
```powershell
$patch = @'
imagePullSecrets:
- name: myregistrykey
'@
```

`Using patch variable at attach`
```powershell
kubectl patch serviceaccount app-sa --type=merge --patch $patch
```
Step 4: Verify ImagePullSecret
```powershell
kubectl get sa app-sa -o yaml
```

Look for:
imagePullSecrets:
- name: myregistrykey


*Observation*: The app-sa ServiceAccount now references the myregistrykey Secret. New Pods using this ServiceAccount can automatically receive this image pull configuration.

*Important*: The Secret and ServiceAccount must be in the same namespace as the Pod. Also, attaching an ImagePullSecret does not grant API permissions; it only provides registry authentication configuration.


6. Granting RBAC Permissions to a ServiceAccount

Our objective is to allow app-sa to:
- Get Pods.
- List Pods.
- Watch Pods.
We will not grant permission to create or delete Pods.

Step 1: Create a Role
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
  verbs: ["get", "list", "watch"]
```

Explanation:
- kind: Role: Defines namespace-level permissions.
- metadata.name: Name of the Role.
- namespace: Namespace where the Role applies.
- apiGroups: [""]: Specifies the core API group.
- resources: ["pods"]: Specifies Pods as the target resource.
- verbs: Defines the allowed operations.

Apply the YAML file:
```powershell
kubectl apply -f pod-reader-role.yaml
```

Verify:
```powershell
kubectl get role
```

Step 2: Create a RoleBinding

Create a file named pod-reader-binding.yaml.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Explanation:
- kind: RoleBinding: Connects a subject with a Role.
- subjects: Specifies who receives the permissions.
- kind: ServiceAccount: Indicates that the subject is a ServiceAccount.
- name: app-sa: Name of our ServiceAccount.
- namespace: default: Namespace where the ServiceAccount exists.
- roleRef: References the pod-reader Role.

Apply the RoleBinding:
```powershell
kubectl apply -f pod-reader-binding.yaml
```

Step 3: Verify Role and RoleBinding
```powershell
kubectl get role pod-reader
```

Observation: The app-sa ServiceAccount is now associated with the pod-reader Role through the RoleBinding.

7. Verifying ServiceAccount Permissions

For this verification, switch to your admin context:
kubectl config use-context kind-lea-cluster

Step 1: Check permissions using kubectl auth can-i

The ServiceAccount identity follows this format:
system:serviceaccount:<namespace>:<serviceaccount-name>
system:serviceaccount:default:app-sa


Check whether it can list Pods:
kubectl auth can-i list pods --as=system:serviceaccount:default:app-sa

Expected:
yes


Check other permissions:
kubectl auth can-i get pods --as=system:serviceaccount:default:app-sa

kubectl auth can-i watch pods --as=system:serviceaccount:default:app-sa

kubectl auth can-i create pods --as=system:serviceaccount:default:app-sa

kubectl auth can-i delete pods --as=system:serviceaccount:default:app-sa


Expected results:
Permission  	Result
Get Pods	    yes
List Pods	    yes
Watch Pods	    yes
Create Pods	    no
Delete Pods	    no

*Observation*: The ServiceAccount has only the permissions defined in the Role.

Note: The account running these checks must have permission to impersonate the ServiceAccount. Your kind admin context should normally have this permission.

Step 2: Assign the ServiceAccount to a Pod
A ServiceAccount becomes the identity of an application when we assign it to a Pod.

Create a file named sa-pod.yaml.
apiVersion: v1
kind: Pod
metadata:
  name: sa-test-pod
  namespace: default
spec:
  serviceAccountName: app-sa
  containers:
  - name: nginx
    image: nginx


Create the Pod:
kubectl apply -f sa-pod.yaml


Verify:
kubectl get pod sa-test-pod


Check which ServiceAccount is assigned:
kubectl get pod sa-test-pod -o jsonpath="{.spec.serviceAccountName}"


Expected:
app-sa


Check whether the ImagePullSecret was automatically added:
kubectl get pod sa-test-pod -o jsonpath="{.spec.imagePullSecrets}"


Expected:
[{"name":"myregistrykey"}]


This confirms that the Pod is using app-sa and has inherited its configured ImagePullSecret.

The example uses the public nginx image for simplicity. To test actual private registry authentication, replace it with an image from your private repository and use valid registry credentials.

Important: A Pod using app-sa receives the ServiceAccount's identity and associated RBAC permissions when communicating with the Kubernetes API Server. The permissions are not automatically given to every Pod in the namespace.


8. Important Commands Reference
`Command`                                 	`Purpose`
kubectl create sa app-sa	                Creates ServiceAccount
kubectl get sa	                            Lists ServiceAccounts
kubectl get sa -A                       	Lists ServiceAccounts across namespaces
kubectl create secret docker-registry	    Creates registry Secret
kubectl patch sa app-sa	Attaches            ImagePullSecret
kubectl get sa app-sa -o yaml	            Checks ServiceAccount configuration
kubectl get role	                        Lists Roles
kubectl get rolebinding	                    Lists RoleBindings
kubectl auth can-i list pods --as=...	    Checks ServiceAccount permissions

9. Key Takeaways

1. ServiceAccounts provide identities to applications and workloads running inside Pods.
2. User Accounts are primarily for humans, whereas ServiceAccounts are primarily for applications.
3. Kubernetes automatically creates a default ServiceAccount in every namespace.
4. A ServiceAccount does not automatically have permissions to access Kubernetes resources.
5. RBAC Roles and RoleBindings can grant specific permissions to ServiceAccounts.
6. ImagePullSecrets store private registry credentials.
7. An ImagePullSecret can be attached to a ServiceAccount so that new Pods using it inherit the image pull configuration.
8. The serviceAccountName field assigns a ServiceAccount to a Pod.
9. Always follow the principle of least privilege when granting ServiceAccount permissions.