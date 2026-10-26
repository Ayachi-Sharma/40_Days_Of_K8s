# Day 17: Kubernetes Horizontal Pod Autoscaling (HPA)

## 1. Introduction to Horizontal Pod Autoscaling

**Horizontal Pod Autoscaler (HPA)** is a Kubernetes resource that automatically scales the number of replicas in a Deployment, ReplicaSet, or other supported scalable workload based on observed metrics such as CPU or memory utilization.

- When resource utilization increases, HPA can increase the number of replicas.
- When resource utilization decreases, HPA can reduce the number of replicas.
- HPA respects the configured minimum and maximum replica counts.

In this exercise, the HPA scales the `php-apache` Deployment based on CPU utilization.

---

## 2. Create the Deployment and Service

### Step 1: Create the manifest

Create a file named `deploy.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
    - port: 80
  selector:
    run: php-apache
```

### Step 2: Apply the manifest

```bash
kubectl apply -f deploy.yaml
```

### Step 3: Verify the resources

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

### Explanation

- The Deployment manages the `php-apache` Pods.
- The Service exposes the Pods internally so other Pods can access them using the service name `php-apache`.
- `requests.cpu: 200m` defines the CPU amount used as the baseline for calculating CPU utilization.
- `limits.cpu: 500m` sets the maximum CPU allocation for each container.

**Important:** HPA CPU utilization is calculated relative to the CPU request, not the CPU limit. The CPU request must be configured for utilization-based scaling.

---

## 3. Configure Horizontal Pod Autoscaling

### Step 1: Create the HPA

You used the following command:
```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

This creates an HPA for the `php-apache` Deployment.

### Explanation

- `--cpu-percent=50`: Target average CPU utilization is 50% of the requested CPU.
- `--min=1`: The minimum number of replicas is one.
- `--max=10`: The maximum number of replicas is ten.

### Step 2: Verify the HPA

```bash
kubectl get hpa
```

For more details:
```bash
kubectl describe hpa php-apache
```

The output displays the target CPU utilization, current utilization, and desired replica count.

**Note:** If the current CPU metric appears as `<unknown>`,wait for some time and recheck or check whether Metrics Server is installed and functioning correctly.

---

## 4. Generate Load on the Application

### Step 1: Run the load generator

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

### Explanation

- `kubectl run` creates a temporary Pod named `load-generator`.
- `busybox:1.28` provides the `wget` command used to send HTTP requests.
- `http://php-apache` accesses the Service, which routes requests to the application Pods.
- The loop repeatedly sends requests to generate load.
- `--rm` removes the load-generator Pod when the interactive command terminates and the Pod exits.

### Step 2: Watch the HPA

Open another terminal and run:

```bash
kubectl get hpa php-apache --watch
```

You can also watch the Deployment:

```bash
kubectl get deployment php-apache --watch
```

And inspect the Pods:

```bash
kubectl get pods -o wide --watch
```

### Observation

As the load generator sends continuous requests, CPU utilization increases. When the observed utilization exceeds the target, the HPA can increase the desired replica count.

The Deployment then creates additional Pods to handle the load, up to the configured maximum of ten replicas.

---

## 5. Observe Scale-Down After Removing the Load

### Step 1: Stop the load generator

Return to the terminal running the load generator and press `Ctrl+C`.

Because the command uses `--rm`, the temporary Pod is removed when the command terminates successfully.

Verify:

```bash
kubectl get pods
```

### Step 2: Watch the HPA again

```bash
kubectl get hpa php-apache --watch
```

### Step 3: Observe the replica count

```bash
kubectl get deployment php-apache --watch
```

### Observation

After the load generator stops, CPU utilization decreases. Once the HPA observes the lower utilization and its scale-down behavior permits it, the Deployment gradually reduces the number of replicas.
In your experiment, the replica count returned to the minimum of one.

### Explanation

- HPA periodically evaluates the current metrics against the target.
- When utilization falls, HPA can reduce the desired replica count.
- Scale-down is not necessarily immediate; Kubernetes uses stabilization behavior to avoid frequent fluctuations.
- The replica count cannot fall below the configured minimum of one.

---

## 6. Verify the HPA Configuration

Run:

```bash
kubectl get hpa php-apache
kubectl describe hpa php-apache
kubectl get deployment php-apache
```

Check the following:

- Minimum replicas: `1`
- Maximum replicas: `10`
- CPU utilization target: `50%`
- Desired and current replica counts
- Events showing scaling activity

---

## 7. Key Learnings

1. HPA automatically adjusts the number of replicas based on observed metrics.
2. CPU utilization targets are calculated relative to container CPU requests.
3. Metrics Server provides resource usage metrics required for CPU-based HPA scaling.
4. The Service provides a stable endpoint for sending requests to the application Pods.
5. A load generator can simulate traffic and trigger scale-up.
6. Removing the load allows HPA to scale down after its evaluation and stabilization periods.
7. `kubectl get hpa --watch` helps monitor scaling behavior in real time.
8. The minimum and maximum replica settings control the permitted scaling range.