# Kubernetes: Multi-Container Pod Using an Init Container


## 1. What Is an Init Container?

An **Init Container** is a special container that runs before the main application container starts.

It is commonly used to perform initialization tasks such as:
- Checking whether a dependency is available.
- Preparing configuration files.
- Downloading required files.
- Setting up data required by the application.

### Key Learning
Init Containers must complete successfully before the main application containers start.
A Pod can have multiple Init Containers. They execute sequentially, in the order defined in the YAML file.

---

## 2. Create a Pod YAML File

Create a file named `init-container.yaml`.

Example configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-cont
  labels:
    name: myapp
spec:
  containers:
    - name: nginx-container
      image: nginx
      command: ['sh', '-c', 'echo the app is running && sleep 3600']
      ports:
        - containerPort: 80
      env:
      - name: FIRSTNAME
        value: "piyush"
  initContainers:
    - name: init-container
      image: busybox:1.36
      command: ['sh', -c]
        args: ['echo Waiting service to be up; sleep 2; done']
```
---

## 3. Create the Pod

Apply the YAML file:
```powershell
kubectl apply -f init-container.yaml
```

Expected output:
```text
pod/init-cont created
```

---

## 4. Check the Pod Status

```powershell
kubectl get pods
```

Inspect the Pod in detail:
```powershell
kubectl describe pod init-cont
```

Look for the following sections:

- **Init Containers** — shows the initialization container and its state.
- **Containers** — shows the main Nginx container and its state.
- **Events** — shows relevant scheduling, image-pulling, and container-starting events.

Depending on when you inspect the Pod, its status may briefly show `Init:0/1` while initialization is in progress.

---

## 5. Check the Init Container Logs

Run:

```powershell
kubectl logs init-cont -c init-container
```

Expected output:
```text
Initialization completed
```

### Explanation

- `kubectl logs` retrieves container logs.
- `init-cont` is the Pod name.
- `-c init-container` specifies which container's logs to display.

The `-c` option is important because a Pod can contain multiple containers.

---

## 6. Verify the Main Application Container

Check the Pod again:

```powershell
kubectl get pod init-cont
```

Verify that Nginx is running:
```powershell
kubectl exec -it init-cont -c nginx-container -- sh
```

Inside the container, run:
```sh
nginx -v
```

Exit the container:
```sh
exit
```

You can also inspect the Nginx container logs:
```powershell
kubectl logs init-cont -c nginx-container
```


## 7. Understand the Execution Flow

The execution sequence is:

1. Kubernetes schedules the Pod onto a node.
2. Kubernetes starts the Init Container.
3. The initialization command executes.
4. The Init Container exits successfully.
5. Kubernetes starts the main Nginx container.
6. Nginx continues running.

If the Init Container fails, Kubernetes retries it according to the Pod's restart policy. The main application container will not start until initialization succeeds.

---

## 8. Clean Up

Delete the Pod when the exercise is complete:

```powershell
kubectl delete pod init-cont
```

Verify deletion:

```powershell
kubectl get pods
```

---

## Final Summary

1. An Init Container runs before the main application containers.
2. Init Containers must complete successfully before application startup.
3. Multiple Init Containers run sequentially.
4. Use `kubectl logs POD_NAME -c CONTAINER_NAME` to inspect a specific container's logs.
5. Use `kubectl describe pod POD_NAME` to inspect container states and troubleshooting events.
6. Init Containers are useful for preparing an application before it starts.

**Remember:** An Init Container is not a continuously running sidecar. It performs its initialization task, completes, and then allows the main application container to start.