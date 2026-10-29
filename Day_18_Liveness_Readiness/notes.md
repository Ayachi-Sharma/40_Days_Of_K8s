# Day 18: Kubernetes Liveness and Readiness Probes

## 1. Introduction to Kubernetes Probes

- **Liveness Probe:** Checks whether a container is still healthy. If it fails repeatedly, Kubernetes restarts the container.
- **Readiness Probe:** Checks whether a container is ready to receive traffic. If it fails, the Pod is removed from the ready endpoints of its Service.
- **Startup Probe:** Gives a container additional time to start before liveness and readiness checks begin.


---

## 2. Create a Pod With an Exec Liveness Probe

### Step 1: Create the manifest

Create a file named `liveness-exec.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  labels:
    test: liveness
spec:
  containers:
    - name: liveness
      image: busybox:1.28
      args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
```

### Step 2: Create the Pod

```bash
kubectl apply -f liveness-exec.yaml
```

### Step 3: Verify the Pod

```bash
kubectl get pods
kubectl describe pod liveness-exec
```

### Step 4: Observe the liveness probe

```bash
kubectl get pod liveness-exec --watch
```

After the container starts:

1. The command creates `/tmp/healthy`.
2. The liveness probe begins after five seconds and checks the file every five seconds.
3. After 30 seconds, the command removes `/tmp/healthy`.
4. The next probe attempts to run `cat /tmp/healthy` and fails because the file no longer exists.
5. After repeated probe failures, kubelet restarts the container.

Check the restart count:

```bash
kubectl get pod liveness-exec
kubectl describe pod liveness-exec
```

Look for the restart count and probe-related events.

### Explanation

- `exec` runs a command inside the container.
- `cat /tmp/healthy` succeeds when the file exists and fails when it does not.
- `initialDelaySeconds: 5` delays the first probe until five seconds after the container starts.
- `periodSeconds: 5` sets the interval between probes.
- Kubernetes restarts the container after the configured failure threshold is reached. The default `failureThreshold` is three consecutive failures.

---

## 3. Create a Pod With HTTP Liveness and Readiness Probes

### Step 1: Create the manifest

Create a file named `liveness-readiness-http.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-readiness
  labels:
    app: liveness-readiness
spec:
  containers:
    - name: agnhost
      image: registry.k8s.io/e2e-test-images/agnhost:2.40
      args:
        - liveness
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
```

### Step 2: Create the Pod

```bash
kubectl apply -f liveness-readiness-http.yaml
```

### Step 3: Verify the Pod

```bash
kubectl get pods
kubectl describe pod liveness-readiness
```

Check whether the Pod becomes `Running` and `Ready`.

### Step 4: Verify the probes

```bash
kubectl describe pod liveness-readiness
```

Check the container's Liveness and Readiness probe details and the Events section.

### Explanation

- `httpGet` performs an HTTP request against the specified path and port.
- The liveness probe checks whether the application is healthy.
- The readiness probe checks whether the application is ready to receive traffic.
- Both probes start after five seconds and run every ten seconds.
- A successful HTTP response with a status code from 200 through 399 is considered successful.

**Important:** The `args: ["liveness"]` setting starts the `agnhost` liveness test server, which serves the health endpoint on port 8080. The container port declaration alone does not start an HTTP server.

---

## 4. Understand TCP Liveness Probes

Example manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcp-pod
  labels:
    app: tcp-pod
spec:
  containers:
    - name: goproxy
      image: registry.k8s.io/goproxy:0.1
      ports:
        - containerPort: 8080
      livenessProbe:
        tcpSocket:
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 5
```

### Explanation

- `tcpSocket` attempts to establish a TCP connection to the specified port.
- `initialDelaySeconds: 10` delays the first check until ten seconds after the container starts.
- `periodSeconds: 5` runs the check every five seconds.
- If the TCP connection cannot be established repeatedly, kubelet restarts the container after the configured failure threshold is reached.

**Note:** The example probes port `3000`, even though the manifest declares container port `8080`. The probe checks the actual port specified under `tcpSocket`; `containerPort` does not configure the application to listen on a port. Ensure the application really listens on port `3000` if you want this probe to succeed.

---

## 5. Key Learnings

1. Liveness probes determine whether a container should be restarted.
2. Readiness probes determine whether a Pod should receive traffic through Services.
3. Exec probes run commands inside the container to verify its health.
4. HTTP probes check an application's health endpoint using HTTP requests.
5. TCP probes verify whether a TCP connection can be established on a port.
6. `initialDelaySeconds` controls when the first probe begins.
7. `periodSeconds` controls the interval between probe attempts.
8. Kubernetes waits for the configured failure threshold before taking the relevant action.

## 6. Clean Up

Delete the Pods created during this exercise:

```bash
kubectl delete pod liveness-exec liveness-readiness tcp-pod
```

If a Pod was not created, Kubernetes will report that it was not found; this does not affect the other deletions.