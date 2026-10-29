# Day 19: Kubernetes Secrets – Inject Data into Container Environment Variables

## 1. Introduction to Kubernetes Secrets

A **Secret** is a Kubernetes object used to store sensitive data such as usernames, passwords, tokens, and credentials.
Instead of hardcoding sensitive values in a Pod manifest, we can store them in a Secret and reference them from the container configuration.

## 2. Create a Secret With a Single Key-Value Pair

### Step 1: Create the Secret

Create a Secret named `backend-user` containing a username:

```powershell
kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
```

### Step 2: Verify the Secret

```powershell
kubectl get secrets
kubectl describe secret backend-user
```

### Explanation

- `kubectl create secret generic` creates a generic Kubernetes Secret.
- `backend-user` is the Secret's name.
- `--from-literal` defines a key-value pair.
- `backend-username` is the key, and `backend-admin` is its value.

## 3. Inject Secret Data Into a Container Environment Variable

### Step 1: Create the Pod manifest

Create a file named `pod-single-secret-env-variable.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
```

### Step 2: Create the Pod

```powershell
kubectl apply -f pod-single-secret-env-variable.yaml
```

### Step 3: Verify the Pod

```powershell
kubectl get pods
```

Wait until the Pod is running.

### Step 4: Check the environment variable

```powershell
kubectl exec env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
```

Expected output:

```text
backend-admin
```

### Explanation

- `env` defines environment variables for the container.
- `valueFrom` specifies that the value comes from another Kubernetes resource.
- `secretKeyRef` references a specific key inside a Secret.
- `name: backend-user` identifies the Secret.
- `key: backend-username` identifies the key whose value should be injected.
- `SECRET_USERNAME` is the environment variable available inside the container.

## 4. Inject Environment Variables From Multiple Secrets

### Step 1: Create the Secrets

```powershell
kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
```

```powershell
kubectl create secret generic db-user --from-literal=db-username='db-admin'
```

If `backend-user` already exists from the previous step, do not create it again.

### Step 2: Create the Pod manifest

Create `pod-multiple-secret-env-variable.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envvars-multiple-secrets
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: BACKEND_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-user
              key: db-username
```

### Step 3: Create the Pod

```powershell
kubectl apply -f pod-multiple-secret-env-variable.yaml
```

### Step 4: Verify the environment variables

```powershell
kubectl exec envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'
```

Expected output (order may vary):

```text
BACKEND_USERNAME=backend-admin
DB_USERNAME=db-admin
```

### Explanation

A container can reference multiple Secrets by defining multiple entries under `env`. Each entry maps a Secret key to a container environment variable.

## 5. Inject All Secret Keys Using `envFrom`

### Step 1: Create a Secret

```powershell
kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
```

### Step 2: Create the Pod manifest

Create `pod-secret-envFrom.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      envFrom:
        - secretRef:
            name: test-secret
```

### Step 3: Create the Pod

```powershell
kubectl apply -f pod-secret-envFrom.yaml
```

### Step 4: Verify the environment variables

```powershell
kubectl exec envfrom-secret -- /bin/sh -c 'echo "username: $username"; echo "password: $password"'
```

Expected output:

```text
username: my-app
password: 39528$vdg7Jb
```

### Explanation

- `envFrom` imports all eligible key-value pairs from the referenced Secret.
- Each Secret key becomes an environment variable with the same name.
- This approach is useful when a container needs several values from the same Secret.

**Security note:** The example password is only for practice. Avoid putting real credentials directly in shell history or source-controlled files.

## 6. Important Observations

1. Secrets can be consumed by containers as environment variables.
2. `secretKeyRef` injects a specific key from a Secret.
3. `envFrom` imports all eligible keys from a Secret.
4. The Secret must exist in the same namespace as the Pod.
5. If a Secret is updated, a container's existing environment variables do not automatically change; the container needs to be restarted to receive the updated values.
6. Kubernetes Secret values are base64-encoded in the `data` field by default, which is encoding, not encryption. Proper access controls and encryption at rest are important for production environments.

## 7. Clean Up

Delete the Pods:

```powershell
kubectl delete pod env-single-secret envvars-multiple-secrets envfrom-secret
```

Delete the Secrets:

```powershell
kubectl delete secret backend-user db-user test-secret
```

## Key Learnings

- A Kubernetes Secret stores sensitive key-value data.
- `kubectl create secret generic` creates a Secret from literal values or files.
- `secretKeyRef` maps one Secret key to one environment variable.
- `envFrom` exposes multiple Secret keys as environment variables.
- Secrets help separate sensitive configuration from application manifests, but they still require appropriate security controls.