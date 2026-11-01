# Task: Install a Multi-Node Kubernetes Cluster Using kubeadm (v1.29)

## 1. Objective

Set up a multi-node Kubernetes cluster using `kubeadm` on virtual machines.

The cluster should consist of:
- **1 Master (Control Plane) Node**
- **2 Worker Nodes**

Install the following Kubernetes components and add-ons:
- kubeadm
- kubelet
- kubectl
- Calico (Container Network Interface)

**Expected Result:** All three nodes should successfully join the cluster and show `Ready` status.

---

## 2. Cluster Architecture

| Node | Role | Purpose |
|---|---|---|
| Master | Control Plane | Manages the Kubernetes cluster |
| Worker-1 | Worker | Runs application workloads |
| Worker-2 | Worker | Runs application workloads |

All three virtual machines must have network connectivity with each other.

## 3. Prerequisites

Before starting the installation:

- Create three virtual machines.
- Assign unique hostnames to each VM.
- Configure static IP addresses or ensure stable IP addresses.
- Ensure all nodes can communicate with each other.
- Configure firewall rules to allow required Kubernetes communication.
- Disable swap on all nodes.
- Install and configure a container runtime such as containerd.

## 4. Prepare All Nodes

Perform the following steps on the master and both worker nodes.

### Step 1: Set Hostnames

On the master node:

```bash
sudo hostnamectl set-hostname master
```

On Worker-1:

```bash
sudo hostnamectl set-hostname worker1
```

On Worker-2:

```bash
sudo hostnamectl set-hostname worker2
```

### Step 2: Configure Hostname Resolution

Edit `/etc/hosts` on all three nodes:

```bash
sudo vi /etc/hosts
```

Add the IP addresses and hostnames:

```text
<MASTER-IP>   master
<WORKER1-IP>  worker1
<WORKER2-IP>  worker2
```

### Step 3: Disable Swap

Kubernetes requires swap to be disabled unless explicitly configured otherwise.

```bash
sudo swapoff -a
```

To disable swap permanently, comment out the swap entry in `/etc/fstab`.

### Step 4: Load Required Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Create a configuration file:

```bash
sudo vi /etc/modules-load.d/k8s.conf
```

Add:

```text
overlay
br_netfilter
```

### Step 5: Configure Network Parameters

Create a sysctl configuration file:

```bash
sudo vi /etc/sysctl.d/k8s.conf
```

Add:

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

Apply the settings:

```bash
sudo sysctl --system
```

## 5. Install Container Runtime

Install and configure containerd on all three nodes.

After installing containerd, generate its default configuration:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Configure the systemd cgroup driver:

```bash
sudo vi /etc/containerd/config.toml
```

Set:

```toml
SystemdCgroup = true
```

Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify:

```bash
sudo systemctl status containerd
```

## 6. Install Kubernetes Components

Install the following components on all three nodes:

- kubeadm: Initializes and manages the Kubernetes cluster.
- kubelet: Runs on every node and manages Pods and containers.
- kubectl: Command-line tool used to interact with the Kubernetes cluster.

Install Kubernetes version **1.29** using the appropriate package repository for your Linux distribution.

Verify the installed versions:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

Ensure that all nodes use compatible Kubernetes 1.29 packages.

## 7. Initialize the Master Node

Perform this step only on the master node.

Initialize the Kubernetes control plane:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

The Pod CIDR must match the Calico IP pool configuration.

After successful initialization, configure kubectl access for the current user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the cluster:

```bash
kubectl get nodes
```

Initially, the master node may show `NotReady` because the network plugin has not yet been installed.

## 8. Install Calico Network Plugin

Calico provides networking between Kubernetes Pods and supports NetworkPolicy enforcement.

Install a Calico release compatible with Kubernetes 1.29.

Apply the Calico manifest or use the Tigera Operator installation method according to the selected Calico release's documentation.

For a manifest-based installation, the command format is:

```bash
kubectl apply -f <CALICO-MANIFEST-URL>
```

Ensure the Calico Pod network configuration uses the same Pod CIDR specified during `kubeadm init`.

Verify Calico Pods:

```bash
kubectl get pods -n calico-system
```

Depending on the installation method, Calico components may also appear in the `kube-system` namespace.

Wait until the network components are running.

## 9. Join Worker Nodes

After initializing the master node, `kubeadm` displays a `kubeadm join` command.

It looks similar to:

```bash
sudo kubeadm join <MASTER-IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Run this command on **Worker-1** and **Worker-2**.

This connects both worker nodes to the master and adds them to the Kubernetes cluster.

If the join command is no longer available, generate a new one on the master:

```bash
kubeadm token create --print-join-command
```

## 10. Verify the Cluster

Run the following commands on the master node.

### Check Nodes

```bash
kubectl get nodes -o wide
```

Expected result:

| NAME | STATUS | ROLE |
|---|---|---|
| master | Ready | control-plane |
| worker1 | Ready | worker |
| worker2 | Ready | worker |

### Check System Pods

```bash
kubectl get pods -A
```

Ensure that the Kubernetes system components and Calico Pods are running successfully.

### Check Node Details

```bash
kubectl describe nodes
```

Check the node conditions and confirm that the nodes report `Ready`.

## 11. Final Outcome

The multi-node Kubernetes cluster was configured using kubeadm with one control-plane node and two worker nodes.

The Kubernetes components kubeadm, kubelet, and kubectl were installed, and Calico was configured as the cluster networking plugin.

**Final verification:** All three nodes should be in `Ready` status.

## 12. Important Commands Summary

| Command | Purpose |
|---|---|
| `kubeadm init` | Initialize the control plane |
| `kubeadm join` | Join a worker node to the cluster |
| `kubeadm token create --print-join-command` | Generate a worker join command |
| `kubectl get nodes` | Display cluster nodes |
| `kubectl get pods -A` | Display Pods across all namespaces |
| `kubectl get nodes -o wide` | Display detailed node information |
| `kubectl describe nodes` | Inspect node conditions and details |