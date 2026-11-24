# Day 39 – Kubernetes Worker Node Troubleshooting

## 1. Introduction

In this task, we intentionally introduce two different failures on Kubernetes worker nodes and troubleshoot them to restore cluster health.

### Objectives

1. Troubleshoot an incorrect kubelet configuration.
2. Recover a worker node after kubelet configuration corruption.
3. Troubleshoot a stopped kubelet service.
4. Restore both worker nodes to the `Ready` state.

### Problems introduced

| Script | Node | Issue |
|---|---|---|
| `problem1.sh` | Worker Node 1 | Incorrect `clientCAFile` path in kubelet configuration |
| `problem2.sh` | Worker Node 2 | Kubelet service stopped |

---

## 2. Initial Cluster Health Check

Before introducing any issues, check the cluster from the control-plane node:

```bash
kubectl get nodes
```

Check detailed node information:

```bash
kubectl get nodes -o wide
```

Both worker nodes should initially be in the `Ready` state.

---

## 3. Problem 1 – Incorrect Kubelet Configuration

### Step 1: Login to Worker Node 1

```bash
ssh username@worker-node1-ip
```

### Step 2: Clone the repository

```bash
git clone <repository-url>
```

Navigate to the Day39 directory:

```bash
cd <repository-name>/CKA-2024/Resources/Day39
```

### Step 3: Execute Problem 1

Do not open the script before executing it.

```bash
chmod +x problem1.sh
./problem1.sh
```

The script changes the kubelet configuration and restarts the kubelet.

### Step 4: Check cluster health

From the control-plane node:

```bash
kubectl get nodes
```

Worker Node 1 may now show `NotReady`.

### Step 5: Troubleshoot kubelet

Login to Worker Node 1 and check the kubelet service:

```bash
sudo systemctl status kubelet
```

Read kubelet logs:

```bash
sudo journalctl -u kubelet -n 50
```

Follow logs in real time:

```bash
sudo journalctl -u kubelet -f
```

The problem is caused by an incorrect certificate authority file path in the kubelet configuration.

### Step 6: Inspect the kubelet configuration

```bash
sudo grep -n "clientCAFile" /var/lib/kubelet/config.yaml
```

The incorrect configuration points to:

```text
/etc/kubernetes/pki/non-existent-ca.crt
```

The expected path introduced by the original script is:

```text
/etc/kubernetes/pki/ca.crt
```

### Step 7: Fix the configuration

Edit the kubelet configuration:

```bash
sudo vi /var/lib/kubelet/config.yaml
```

Find the `clientCAFile` entry and correct it:

```yaml
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
```

Save the file.

**Note:** Preserve the rest of the existing configuration. Do not replace the entire file with this small YAML snippet.

Verify the certificate file exists:

```bash
sudo ls -l /etc/kubernetes/pki/ca.crt
```

### Step 8: Restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Check its status:

```bash
sudo systemctl status kubelet
```

### Step 9: Verify Worker Node 1

From the control-plane node:

```bash
kubectl get nodes
```

Worker Node 1 should return to `Ready` after the kubelet reconnects and reports its status.

---

## 4. Problem 2 – Kubelet Service Stopped

### Step 1: Login to Worker Node 2

```bash
ssh username@worker-node2-ip
```

### Step 2: Clone the repository

```bash
git clone <repository-url>
```

Navigate to the Day39 directory:

```bash
cd <repository-name>/CKA-2024/Resources/Day39
```

### Step 3: Execute Problem 2

Do not open the script before executing it.

```bash
chmod +x problem2.sh
./problem2.sh
```

The script stops the kubelet service:

```bash
sudo systemctl stop kubelet
```

### Step 4: Check cluster health

From the control-plane node:

```bash
kubectl get nodes
```

Worker Node 2 may now show `NotReady`.

### Step 5: Check kubelet status

Login to Worker Node 2:

```bash
sudo systemctl status kubelet
```

The service should show an inactive or stopped state.

### Step 6: Start kubelet

Start the service:

```bash
sudo systemctl start kubelet
```

Check its status:

```bash
sudo systemctl status kubelet
```

Enable kubelet to start automatically during system boot:

```bash
sudo systemctl enable kubelet
```

### Step 7: Verify kubelet logs

```bash
sudo journalctl -u kubelet -n 50
```

Check for errors that could prevent kubelet from registering or communicating with the API server.

### Step 8: Verify Worker Node 2

From the control-plane node:

```bash
kubectl get nodes
```

Worker Node 2 should return to `Ready` after kubelet reconnects.

---

## 5. Final Cluster Health Verification

After fixing both worker nodes, execute the following commands from the control-plane node.

Check all Nodes:

```bash
kubectl get nodes
```

Check detailed Node information:

```bash
kubectl get nodes -o wide
```

Check all Pods:

```bash
kubectl get pods -A
```

Inspect Node conditions if any node remains unhealthy:

```bash
kubectl describe node <node-name>
```

Check recent cluster events:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Expected result: Both worker nodes should be in the `Ready` state.

---

## 6. Important Commands Summary

### Problem 1 – Kubelet Configuration

```bash
# Inspect client CA configuration
sudo grep -n "clientCAFile" /var/lib/kubelet/config.yaml

# Verify CA certificate
sudo ls -l /etc/kubernetes/pki/ca.crt

# Edit configuration
sudo vi /var/lib/kubelet/config.yaml

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Check status and logs
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

### Problem 2 – Kubelet Stopped

```bash
# Check kubelet
sudo systemctl status kubelet

# Start kubelet
sudo systemctl start kubelet

# Enable kubelet at boot
sudo systemctl enable kubelet

# Check logs
sudo journalctl -u kubelet -n 50
```

### Cluster Verification

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl get pods -A
kubectl describe node <node-name>
```

---

## 7. Conclusion

In this task, we practiced troubleshooting two common Kubernetes worker-node failures.

**Problem 1:** An incorrect `clientCAFile` path prevented kubelet from loading its intended configuration correctly. We fixed the path and restarted kubelet.

**Problem 2:** The kubelet service was stopped. We started the service and verified that it was running.

The key learning is that when a Node becomes `NotReady`, checking kubelet status, configuration, and system logs is an important first step in identifying the root cause.