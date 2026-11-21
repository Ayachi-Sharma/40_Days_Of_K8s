# Day 32 – Kubernetes Metrics Server and Cluster Troubleshooting

## 1. Introduction

In this task, we work with the Kubernetes Metrics Server and learn how to monitor resource usage and troubleshoot Kubernetes components.

The main objectives are:

1. Install Metrics Server using the manifest provided in the Day16 folder.
2. Understand how the Kubernetes resource metrics pipeline works.
3. Identify the Pod consuming the highest CPU in the `kube-system` namespace and save its name to a file.
4. Read kubelet logs on a worker node using `journalctl`.
5. Read kube-apiserver container logs using `crictl`, without using `kubectl`.

---

## 2. Metrics Server

### What is Metrics Server?

Metrics Server is a cluster-level component that collects CPU and memory usage metrics from Kubernetes nodes and Pods.

It collects metrics from the kubelet running on each node and makes them available through the Kubernetes Metrics API.

These metrics are used by:

- `kubectl top` to display CPU and memory usage.
- Horizontal Pod Autoscaler (HPA) to scale workloads based on resource usage.
- Other autoscaling components that consume the Metrics API.

**Important:** Metrics Server provides basic CPU and memory metrics. It is not a complete monitoring solution like Prometheus and Grafana.

### Install Metrics Server

Navigate to the Day16 folder containing the Metrics Server manifest.

```bash
cd Day16
```

Apply the manifest:

```bash
kubectl apply -f <metrics-server-manifest.yaml>
```

Replace the filename with the actual manifest name present in the folder.

If the folder contains only the required Metrics Server manifest, you can also apply it using:

```bash
kubectl apply -f .
```

Check whether the Metrics Server Pod is running:

```bash
kubectl get pods -n kube-system
```

Check the Metrics Server Deployment:

```bash
kubectl get deployment metrics-server -n kube-system
```

Check the Metrics API:

```bash
kubectl get apiservices | grep metrics
```

Expected APIService:

```text
v1beta1.metrics.k8s.io
```

The API version can vary depending on the Metrics Server and Kubernetes versions.

### Verify Metrics Server

Check node resource usage:

```bash
kubectl top nodes
```

Check Pod resource usage:

```bash
kubectl top pods -A
```

Check resource usage in a particular namespace:

```bash
kubectl top pods -n kube-system
```

If these commands display CPU and memory usage, the Metrics Server is working.

**Note:** Metrics may take a short time to become available after installation.

---

## 3. How the Kubernetes Resource Metrics Pipeline Works

The resource metrics pipeline is responsible for collecting and providing CPU and memory usage information.

### Components involved

**1. Container Runtime**

The container runtime, such as containerd or CRI-O, runs the containers on a Kubernetes node.

**2. cAdvisor**

cAdvisor collects and aggregates container resource usage information. It is integrated with the kubelet.

**3. Kubelet**

Kubelet is the node agent responsible for managing Pods and containers on a node.

It exposes resource metrics through endpoints such as:

```text
/metrics/resource
/stats
```

**4. Metrics Server**

Metrics Server collects resource metrics from the kubelet on each node and aggregates them.

**5. Kubernetes API Server**

Metrics Server exposes metrics through the Kubernetes Metrics API:

```text
metrics.k8s.io
```

Clients such as `kubectl top` and HPA can access these metrics through the API.

### Resource Metrics Pipeline Flow

```text
       Container Runtime
               |
               v
            cAdvisor
               |
               v
             Kubelet
               |
               | Resource Metrics
               v
         Metrics Server
               |
               v
        Kubernetes API
               |
          +----+----+
          |         |
          v         v
      kubectl top   HPA
```

### Important points

- cAdvisor collects container resource usage.
- Kubelet exposes resource metrics.
- Metrics Server collects metrics from kubelets.
- Metrics Server provides the Metrics API.
- `kubectl top` displays CPU and memory usage.
- HPA uses metrics to make scaling decisions.

Metrics Server uses the `/metrics/resource` endpoint in versions v0.6.0 and later. Older versions used the Summary API endpoint `/stats/summary`.

Reference:

https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

---

## 4. Find the Pod Using the Most CPU in kube-system

### Step 1: Display CPU and memory usage

```bash
kubectl top pods -n kube-system
```

Example output:

```text
NAME                                      CPU(cores)   MEMORY(bytes)
coredns-xxxxxxxxxx-xxxxx                  2m           18Mi
etcd-controlplane                         25m          120Mi
kube-apiserver-controlplane               45m          250Mi
kube-controller-manager-controlplane     12m          55Mi
kube-scheduler-controlplane               8m           40Mi
```

The CPU usage values are displayed in millicores (`m`).

For example:

```text
1000m = 1 CPU core
100m  = 0.1 CPU core
10m   = 0.01 CPU core
```

### Step 2: Sort Pods by CPU usage

```bash
kubectl top pods -n kube-system --sort-by=cpu
```

The Pods are displayed in ascending order of CPU usage. The last Pod in the output is the one consuming the most CPU.

### Step 3: Save the Pod name to a file

Run this command on the control-plane node:

```bash
kubectl top pods -n kube-system --no-headers | sort -k2 -h | tail -n 1 | awk '{print $1}' > /opt/highest-cpu-pod.txt
```

This command performs the following operations:

- `kubectl top pods`: Gets resource usage of Pods.
- `-n kube-system`: Filters Pods from the kube-system namespace.
- `--no-headers`: Removes the column headings.
- `sort -k2 -h`: Sorts based on the CPU usage column.
- `tail -n 1`: Selects the Pod with the highest CPU usage.
- `awk '{print $1}'`: Extracts the Pod name.
- `>`: Writes the Pod name into a file.

### Step 4: Verify the file

```bash
cat /opt/highest-cpu-pod.txt
```

The file should contain only the name of the Pod consuming the most CPU.

**Note:** This command uses Linux utilities (`sort`, `tail`, and `awk`). It should be executed in the Linux shell of the control-plane node.

---

## 5. Read Kubelet Logs Using journalctl

### What is journalctl?

`journalctl` is a Linux command used to view logs collected by the systemd journal.

In Kubernetes, kubelet commonly runs as a systemd service on worker nodes.

We can use `journalctl` to inspect kubelet logs without using kubectl.

### Step 1: Connect to a worker node

From the control-plane node, connect to a worker node using SSH:

```bash
ssh username@worker-node-ip
```

Use the appropriate username and IP address for your environment.

### Step 2: Check kubelet service status

```bash
sudo systemctl status kubelet
```

This displays the current status of the kubelet service.

### Step 3: Read kubelet logs

```bash
sudo journalctl -u kubelet
```

This displays the logs generated by the kubelet service.

### Step 4: Display the latest logs

```bash
sudo journalctl -u kubelet -n 50
```

Displays the last 50 log entries.

### Step 5: Follow logs in real time

```bash
sudo journalctl -u kubelet -f
```

The `-f` option follows the logs continuously, similar to `tail -f`.

Press `Ctrl + C` to stop following the logs.

### Useful journalctl commands

```bash
# Logs from the current boot
sudo journalctl -u kubelet -b

# Logs generated within the last hour
sudo journalctl -u kubelet --since "1 hour ago"

# Logs with detailed timestamps
sudo journalctl -u kubelet -o short-iso
```

---

## 6. Read kube-apiserver Container Logs Using crictl

### What is crictl?

`crictl` is a command-line utility used to interact with Kubernetes container runtimes that implement the Container Runtime Interface (CRI).

It can be used to inspect Pods, containers, and images directly through the container runtime.

We can use `crictl` to read container logs without using kubectl.

### Step 1: Connect to the control-plane node

```bash
ssh username@controlplane-node-ip
```

The kube-apiserver normally runs as a static Pod on the control-plane node in a kubeadm cluster.

### Step 2: List running containers

```bash
sudo crictl ps
```

This displays the currently running containers.

Example output:

```text
CONTAINER           IMAGE               CREATED          STATE
xxxxxxxxxxxx        kube-apiserver      ...              Running
xxxxxxxxxxxx        etcd                ...              Running
xxxxxxxxxxxx        kube-scheduler      ...              Running
```

### Step 3: Find the kube-apiserver container ID

```bash
sudo crictl ps | grep kube-apiserver
```

Copy the container ID from the output.

### Step 4: Read kube-apiserver logs

```bash
sudo crictl logs <container-id>
```

Replace `<container-id>` with the actual kube-apiserver container ID.

Example:

```bash
sudo crictl logs xxxxxxxxxxxx
```

This displays the logs generated by the kube-apiserver container.

### Step 5: Display only the latest logs

```bash
sudo crictl logs --tail=50 <container-id>
```

Displays the last 50 lines of the container logs.

### Step 6: Follow container logs

```bash
sudo crictl logs -f <container-id>
```

Displays new log entries as they are generated.

Press `Ctrl + C` to stop following the logs.

### Useful crictl commands

```bash
# List running containers
sudo crictl ps

# List all containers, including stopped containers
sudo crictl ps -a

# Inspect a container
sudo crictl inspect <container-id>

# Read container logs
sudo crictl logs <container-id>
```

**Important:** The kube-apiserver container is normally found on the control-plane node, not the worker node. Use `crictl` directly on the node where that container is running.

---

## 7. Difference Between journalctl and crictl

| Command | Purpose | Used for |
|---|---|---|
| `journalctl` | Reads systemd journal logs | Kubelet service logs |
| `crictl` | Interacts with the container runtime | Container logs and inspection |
| `kubectl logs` | Reads container logs through the Kubernetes API | Application and Pod logs |

### Examples

Read kubelet service logs:

```bash
sudo journalctl -u kubelet
```

Read kube-apiserver container logs:

```bash
sudo crictl ps | grep kube-apiserver
sudo crictl logs <container-id>
```

The important difference is that `journalctl` reads service logs from the Linux system journal, whereas `crictl logs` reads logs directly from a container through the container runtime.

---

## 8. Important Commands Summary

### Metrics Server

```bash
kubectl apply -f <metrics-server-manifest.yaml>
kubectl get pods -n kube-system
kubectl get apiservices | grep metrics
kubectl top nodes
kubectl top pods -A
```

### Find the highest CPU-consuming Pod

```bash
kubectl top pods -n kube-system --sort-by=cpu
```

Save its name:

```bash
kubectl top pods -n kube-system --no-headers | sort -k2 -h | tail -n 1 | awk '{print $1}' > /opt/highest-cpu-pod.txt
```

Verify:

```bash
cat /opt/highest-cpu-pod.txt
```

### Kubelet logs

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet
sudo journalctl -u kubelet -n 50
sudo journalctl -u kubelet -f
```

### Kube-apiserver logs

```bash
sudo crictl ps | grep kube-apiserver
sudo crictl logs <container-id>
sudo crictl logs --tail=50 <container-id>
sudo crictl logs -f <container-id>
```

---

## 9. Conclusion

In this task, we learned how to:

1. Install and verify Metrics Server.
2. Understand how resource metrics travel from the container runtime and cAdvisor through kubelet and Metrics Server to the Kubernetes API.
3. Monitor CPU and memory usage using `kubectl top`.
4. Identify the highest CPU-consuming Pod in the `kube-system` namespace and save its name to a file.
5. Read kubelet service logs using `journalctl` on a worker node.
6. Read kube-apiserver container logs directly using `crictl`, without using kubectl.

These commands are useful for Kubernetes monitoring, cluster administration, and troubleshooting.