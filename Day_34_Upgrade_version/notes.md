# Kubernetes Node Management and Cluster Upgrade

# PART 1: Cordon, Drain and Uncordon

## 1. What is Cordon?

Cordon is a Kubernetes operation that marks a Node as unschedulable.

It prevents Kubernetes from scheduling new Pods on that Node.

Important:
- Existing Pods continue running.
- New Pods will not be scheduled on that Node.
- The Node remains available for existing workloads.
- Useful during maintenance and upgrades.

### Command

kubectl cordon <node-name>

Example:

kubectl cordon worker-node-1

Check Node status:

kubectl get nodes

Example output:

NAME             STATUS
master-node      Ready
worker-node-1    Ready,SchedulingDisabled

SchedulingDisabled means the Node is cordoned.

### Why Do We Use Cordon?

Suppose we need to perform maintenance on a worker Node.

Before starting maintenance, we don't want Kubernetes to schedule new Pods on that Node.

We use cordon to prevent new Pod scheduling.

---

## 2. What is Drain?

Drain is a Kubernetes operation that safely evicts workload Pods from a Node.

It prepares a Node for maintenance by removing eligible Pods from it.

Important:
- Drain first marks the Node as unschedulable.
- Existing eligible Pods are evicted.
- Pods managed by Deployments or ReplicaSets can be recreated on other available Nodes.
- DaemonSet-managed Pods are not removed by default.
- The Node becomes available for maintenance.

### Command

kubectl drain <node-name> --ignore-daemonsets

Example:

kubectl drain worker-node-1 --ignore-daemonsets

### Explanation

- `kubectl drain`: Evicts eligible Pods from the specified Node.
- `worker-node-1`: Node to be drained.
- `--ignore-daemonsets`: Allows drain to proceed without attempting to delete DaemonSet-managed Pods.

### Important Points

1. Drain automatically cordons the Node.
2. Pods managed by Deployments and ReplicaSets can be recreated on other Nodes.
3. DaemonSet Pods are not evicted by default.
4. Pods using local storage or unmanaged Pods may prevent drain from completing.
5. PodDisruptionBudgets (PDBs) may restrict voluntary Pod eviction.

### What is PodDisruptionBudget (PDB)?

A PodDisruptionBudget controls how many replicas of an application can be voluntarily disrupted at a time.

For example:

If an application has 3 replicas and its PDB requires at least 2 available replicas, Kubernetes will prevent an eviction that would violate that requirement.

Check PDBs:

kubectl get pdb -A

### Difference Between Cordon and Drain

| Cordon | Drain |
|---|---|
| Marks Node unschedulable | Evicts eligible Pods |
| Prevents new Pods | Removes existing eligible Pods |
| Existing Pods continue running | Eligible Pods are moved/recreated elsewhere |
| Used to stop scheduling | Used before maintenance |

Important:

Drain does not literally move existing Pods to another Node. Kubernetes creates replacement Pods through their controllers, and those Pods may be scheduled on other Nodes.

---

## 3. What is Uncordon?

Uncordon is a Kubernetes operation that marks a Node as schedulable again.

It allows Kubernetes to schedule new Pods on that Node.

After maintenance or an upgrade, we use uncordon to bring the Node back into normal scheduling.

### Command

kubectl uncordon <node-name>

Example:

kubectl uncordon worker-node-1

Check Node status:

kubectl get nodes

Example output:

NAME             STATUS
master-node      Ready
worker-node-1    Ready

The SchedulingDisabled status is removed.

### Difference Between Cordon, Drain and Uncordon

| Command | Purpose |
|---|---|
| cordon | Prevent new Pod scheduling |
| drain | Evict eligible existing Pods |
| uncordon | Allow new Pod scheduling |

### Complete Node Maintenance Flow

1. Cordon the Node.
2. Drain the Node.
3. Perform maintenance or upgrade.
4. Verify the Node is healthy.
5. Uncordon the Node.

Commands:

kubectl cordon worker-node-1

kubectl drain worker-node-1 --ignore-daemonsets

# Perform maintenance

kubectl uncordon worker-node-1

---

# PART 2: Kubernetes Cluster Upgrade

## 4. What is Kubernetes Cluster Upgrade?

Kubernetes Cluster Upgrade means upgrading the Kubernetes components to a newer version.

Kubernetes releases new versions containing:
- New features.
- Bug fixes.
- Security updates.
- Performance improvements.
- API changes and deprecations.

A cluster upgrade ensures that the Kubernetes environment runs on a supported and updated version.

### Components Involved in a Kubernetes Upgrade

1. kube-apiserver
2. kube-controller-manager
3. kube-scheduler
4. kubelet
5. kube-proxy
6. kubectl
7. kubeadm
8. etcd
9. CoreDNS

Note:

Not all components follow exactly the same versioning scheme. etcd and CoreDNS have their own versions, while Kubernetes components generally follow Kubernetes release versions.

---

## 5. Kubernetes Release Version Format

Kubernetes uses Semantic Versioning.

Format:

MAJOR.MINOR.PATCH

Example:

1.31.2

Explanation:

1. MAJOR (1)

   Represents the major version.

2. MINOR (31)

   Represents the minor release.

3. PATCH (2)

   Represents bug fixes and security updates within that minor release.

### Example

1.30.0
1.30.1
1.30.2
1.31.0
1.31.1

Here:

- 1.30.0 is the initial release of the 1.30 minor version.
- 1.30.2 is a later patch release.
- 1.31.0 is the next minor version.

### Important Kubernetes Versioning Rule

Kubernetes cluster upgrades should be performed one minor version at a time.

Example:

1.30.x → 1.31.x → 1.32.x

Do not directly upgrade:

1.30.x → 1.32.x

Upgrade to the latest appropriate patch release of the next minor version first.

Note:

The package version may also contain a Debian package revision, for example:

1.31.2-1.1

Here:
- 1.31.2 is the Kubernetes version.
- -1.1 is the package revision.

---

## 6. Kubernetes Cluster Upgrade Architecture

A Kubernetes cluster generally contains:

### Control Plane Node

Responsible for managing the cluster.

Important components:

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd
- kubelet
- kubeadm
- kubectl

### Worker Node

Responsible for running application workloads.

Important components:

- kubelet
- kube-proxy
- Container Runtime
- CNI Plugin
- kubeadm (used during kubeadm-based upgrades)

### Upgrade Order

1. Upgrade the control plane node.
2. Upgrade worker nodes.
3. Verify all components and Nodes.

For multiple control plane nodes, upgrade them one at a time.

---

# PART 3: Kubernetes Cluster Upgrade Practical

## 7. Task Details

Objective:

Upgrade the Kubernetes cluster to a higher version using kubeadm.

Requirements:

1. Log in to the cluster and check its health.
2. Upgrade Kubernetes to a newer version by following the readme.md instructions.
3. Upgrade the control plane node.
4. Upgrade all worker nodes.
5. Upgrade kubelet, kubectl and kubeadm.
6. Verify that all Nodes and components are healthy.

Important:

The commands below assume a kubeadm-based cluster running on Ubuntu/Debian with APT package management.

Use the exact target Kubernetes version specified by your assignment and the available package repository.

---

## 8. Step 1: Check Cluster Health

Before upgrading, make sure the cluster is healthy.

### Check Nodes

kubectl get nodes -o wide

Expected:

All Nodes should be Ready.

### Check Pods

kubectl get pods -A

Check that important system Pods are running.

### Check System Components

kubectl get pods -n kube-system

Important components include:

- CoreDNS
- kube-proxy
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd

Some control plane components may run as static Pods.

### Check Current Kubernetes Version

kubectl version

Check Node versions:

kubectl get nodes -o wide

The VERSION column displays the kubelet version reported by each Node.

### Check Cluster Information

kubectl cluster-info

### Check kubeadm Version

On the control plane:

kubeadm version

### Before Starting the Upgrade

- Make sure all Nodes are healthy.
- Make sure system Pods are running.
- Check available disk space.
- Back up important data.
- Take an etcd snapshot if using self-managed etcd.
- Review the upgrade instructions and release notes.

---

## 9. Step 2: Understand Kubernetes Version Compatibility

Before upgrading, understand Kubernetes version compatibility.

Important rules:

1. kubeadm should be upgraded to the target minor version before using it to upgrade the control plane.
2. kubelet must not be newer than kube-apiserver.
3. kubelet may be up to three minor versions older than kube-apiserver, subject to supported upgrade paths.
4. kubectl should be within one minor version of kube-apiserver.
5. Upgrade one minor version at a time.

Example:

Current Cluster Version: 1.30.x

Target Version: 1.31.x

Upgrade path:

1.30.x → 1.31.x

---

# PART 4: Upgrade Control Plane Node

## 10. Step 3: Find Available kubeadm Versions

Log in to the control plane Node.

Update package information:

sudo apt update

List available kubeadm versions:

sudo apt-cache madison kubeadm

Example output:

kubeadm | 1.31.2-1.1
kubeadm | 1.31.1-1.1
kubeadm | 1.30.2-1.1

Select the required target version.

For example:

1.31.2-1.1

Note:

The example versions are illustrative. Choose a version available in your configured repository and appropriate for your cluster.

---

## 11. Step 4: Upgrade kubeadm

First, unhold kubeadm:

sudo apt-mark unhold kubeadm

Update package information:

sudo apt-get update

Install the target version:

sudo apt-get install -y kubeadm='1.31.2-1.1'

Hold kubeadm again:

sudo apt-mark hold kubeadm

Verify:

kubeadm version

Expected version:

v1.31.2

---

## 12. Step 5: Check Upgrade Plan

Run:

sudo kubeadm upgrade plan

This command displays:

- Current cluster version.
- Available Kubernetes versions.
- Upgrade recommendations.
- Component upgrade information.

Review the output before proceeding.

---

## 13. Step 6: Upgrade Control Plane Components

Run:

sudo kubeadm upgrade apply v1.31.2

Replace the version with your actual target version.

This command upgrades the control plane components and applies the required cluster configuration changes.

It also handles Kubernetes-managed addon upgrades, such as CoreDNS and kube-proxy, according to the selected Kubernetes version and configuration.

Important:

- The command may take some time.
- Follow the prompts and verify the upgrade result.
- Do not skip required intermediate minor versions.
- etcd may also be upgraded if the cluster configuration and upgrade plan require it.

### What Happens During the Control Plane Upgrade?

The following components are upgraded as applicable:

1. kube-apiserver
2. kube-controller-manager
3. kube-scheduler
4. etcd
5. CoreDNS
6. kube-proxy

Note:

etcd and CoreDNS have independent version numbers. Do not expect their version numbers to match the Kubernetes version.

---

## 14. Step 7: Drain the Control Plane Node

Before upgrading the kubelet and kubectl on the control plane Node, drain it.

First, identify the Node name:

kubectl get nodes

Drain:

kubectl drain <control-plane-node> --ignore-daemonsets

Example:

kubectl drain master-node --ignore-daemonsets

Explanation:

- Prevents new Pods from being scheduled on the Node.
- Evicts eligible workload Pods.
- Keeps DaemonSet-managed Pods running.

Note:

Draining a control plane Node can be affected by PodDisruptionBudgets, unmanaged Pods, and local storage.

During a single-control-plane upgrade, Kubernetes API management operations may be temporarily unavailable while the API server is down.

Existing application containers generally continue running, although workloads may be affected by the upgrade.

---

## 15. Step 8: Upgrade kubelet and kubectl

Unhold the packages:

sudo apt-mark unhold kubelet kubectl

Update package information:

sudo apt-get update

Install the target versions:

sudo apt-get install -y kubelet='1.31.2-1.1' kubectl='1.31.2-1.1'

Hold the packages again:

sudo apt-mark hold kubelet kubectl

Verify versions:

kubelet --version

kubectl version --client

---

## 16. Step 9: Restart kubelet

Reload the systemd configuration:

sudo systemctl daemon-reload

Restart kubelet:

sudo systemctl restart kubelet

Check kubelet status:

sudo systemctl status kubelet

Make sure the service is active.

---

## 17. Step 10: Uncordon the Control Plane Node

After upgrading and verifying the Node, allow it to schedule Pods again.

Run:

kubectl uncordon <control-plane-node>

Example:

kubectl uncordon master-node

Check:

kubectl get nodes

The control plane Node should return to Ready status without SchedulingDisabled.

---

# PART 5: Upgrade Worker Nodes

## 18. Step 11: Connect to Worker Node

Log in to the worker Node through SSH.

Example:

ssh username@worker-node-ip

Check the current versions:

kubeadm version

kubelet --version

kubectl version --client

Note:

kubectl does not necessarily need to be installed on every worker Node. Follow the assignment's installation requirements and your cluster's existing configuration.

---

## 19. Step 12: Upgrade kubeadm on Worker Node

Update package information:

sudo apt-get update

Unhold kubeadm:

sudo apt-mark unhold kubeadm

Install the target version:

sudo apt-get install -y kubeadm='1.31.2-1.1'

Hold kubeadm:

sudo apt-mark hold kubeadm

Verify:

kubeadm version

---

## 20. Step 13: Upgrade Worker Node Configuration

Run:

sudo kubeadm upgrade node

This command updates the local kubeadm-managed Node configuration.

Important:

Unlike the control plane, a worker Node does not use:

sudo kubeadm upgrade apply

That command is used for upgrading the control plane.

---

## 21. Step 14: Drain Worker Node

From a machine where kubectl has administrative cluster access, run:

kubectl drain <worker-node> --ignore-daemonsets

Example:

kubectl drain worker-node-1 --ignore-daemonsets

This safely evicts eligible application Pods so that the worker Node can be upgraded.

---

## 22. Step 15: Upgrade kubelet and kubectl

On the worker Node:

Unhold the packages:

sudo apt-mark unhold kubelet kubectl

Update package information:

sudo apt-get update

Install the target versions:

sudo apt-get install -y kubelet='1.31.2-1.1' kubectl='1.31.2-1.1'

Hold the packages:

sudo apt-mark hold kubelet kubectl

Note:

If kubectl is not installed on the worker Node, upgrade it only if required by your environment.

---

## 23. Step 16: Restart kubelet

Run:

sudo systemctl daemon-reload

sudo systemctl restart kubelet

Check:

sudo systemctl status kubelet

Make sure kubelet is active.

---

## 24. Step 17: Uncordon Worker Node

From the control plane or another administrative machine:

kubectl uncordon <worker-node>

Example:

kubectl uncordon worker-node-1

Check:

kubectl get nodes

Repeat the same process for every remaining worker Node, one at a time.

---

# PART 6: Verify the Kubernetes Cluster Upgrade

## 25. Step 18: Verify Node Status

Run:

kubectl get nodes -o wide

Expected:

All Nodes should show:

STATUS: Ready

Verify that the VERSION column displays the expected kubelet version.

## 26. Step 19: Verify Kubernetes Components

Check the client and server versions:

kubectl version

Check kubeadm:

kubeadm version

Check kubelet on each Node:

kubelet --version

Check kubectl:

kubectl version --client

## 27. Step 20: Verify System Pods

Run:

kubectl get pods -n kube-system

Check that the important system Pods are healthy.

Pay special attention to:

- CoreDNS
- kube-proxy
- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler

## 28. Step 21: Verify Deployments and Applications

Check all Pods:

kubectl get pods -A

Check Deployments:

kubectl get deployments -A

Check Services:

kubectl get svc -A

Make sure application workloads are running properly after the upgrade.

---

# PART 7: Important Upgrade Concepts

## 29. Why Do We Upgrade One Minor Version at a Time?

Kubernetes supports specific version-skew and upgrade policies.

Skipping minor versions can cause compatibility problems.

Example:

Correct:

1.29.x → 1.30.x → 1.31.x

Incorrect:

1.29.x → 1.31.x

Always check the official Kubernetes upgrade documentation for supported upgrade paths.

## 30. What Happens When the Control Plane Is Down?

During a single-control-plane upgrade, the API server may be temporarily unavailable.

During this time:

- Existing Pods generally continue running.
- Existing containers generally continue executing.
- New API requests may fail.
- kubectl management operations may not work.
- New scheduling and controller operations may be interrupted.

Once the control plane becomes healthy again, normal management operations resume.

In a highly available cluster, other control plane instances can continue serving API requests if configured correctly.

## 31. Why Do We Drain Nodes Before Upgrading?

Draining helps avoid disrupting application workloads during Node maintenance.

It:

- Prevents new Pods from being scheduled.
- Evicts eligible existing workload Pods.
- Allows controllers to recreate workloads on other available Nodes.
- Provides a safer maintenance process.

## 32. Why Do We Uncordon Nodes After Upgrading?

After the Node upgrade is completed, uncordon allows the scheduler to place new Pods on that Node again.

Without uncordon, the Node remains unschedulable.

---

# PART 8: Quick Revision Commands

### Node Management

Cordon:

kubectl cordon <node-name>

Drain:

kubectl drain <node-name> --ignore-daemonsets

Uncordon:

kubectl uncordon <node-name>

### Cluster Upgrade

Check versions:

kubectl version

Check Nodes:

kubectl get nodes -o wide

Check available kubeadm versions:

sudo apt-cache madison kubeadm

Check upgrade plan:

sudo kubeadm upgrade plan

Upgrade control plane:

sudo kubeadm upgrade apply v1.31.2

Upgrade worker configuration:

sudo kubeadm upgrade node

Restart kubelet:

sudo systemctl daemon-reload

sudo systemctl restart kubelet

Check kubelet:

sudo systemctl status kubelet

### Verify Components

kubeadm version

kubelet --version

kubectl version --client

kubectl get pods -n kube-system

kubectl get nodes -o wide

---

# PART 9: Interview Questions

1. What is cordon in Kubernetes?
2. What is drain in Kubernetes?
3. What is uncordon?
4. What is the difference between cordon and drain?
5. Why do we drain a Node before upgrading it?
6. What happens to DaemonSet Pods during drain?
7. What is a PodDisruptionBudget?
8. What is kubeadm?
9. What is the difference between kubeadm upgrade apply and kubeadm upgrade node?
10. Why should Kubernetes be upgraded one minor version at a time?
11. What is the difference between kubelet, kubeadm, and kubectl?
12. What happens to existing Pods when the control plane is unavailable?
13. Why do we restart kubelet after upgrading it?
14. Why do we uncordon a Node after upgrading?
15. What is the role of etcd during a Kubernetes upgrade?
16. Why does CoreDNS have a different version from Kubernetes?
17. How do you verify whether a Kubernetes cluster upgrade was successful?
18. What is Kubernetes version skew?

---

# PART 10: Practical Assignment Checklist

1. Log in to the Kubernetes cluster.
2. Verify that all Nodes are healthy.
3. Check current Kubernetes component versions.
4. Check available kubeadm versions.
5. Select the target Kubernetes version.
6. Upgrade kubeadm on the control plane.
7. Run kubeadm upgrade plan.
8. Upgrade control plane components using kubeadm upgrade apply.
9. Drain the control plane Node.
10. Upgrade kubelet and kubectl.
11. Restart kubelet.
12. Uncordon the control plane Node.
13. Connect to the first worker Node.
14. Upgrade kubeadm on the worker.
15. Run kubeadm upgrade node.
16. Drain the worker Node.
17. Upgrade kubelet and kubectl.
18. Restart kubelet.
19. Uncordon the worker Node.
20. Repeat for remaining worker Nodes.
21. Verify all Nodes are Ready.
22. Verify Kubernetes component versions.
23. Verify CoreDNS and other system Pods.
24. Verify application workloads.

---

# Conclusion

Cordon, Drain, and Uncordon are important Kubernetes Node management operations used during maintenance and upgrades.

Kubernetes cluster upgrades involve upgrading the control plane first, followed by worker Nodes.

The kubeadm tool manages cluster upgrades, kubelet manages Node-level operations, and kubectl is used to communicate with the Kubernetes API Server.

Always upgrade one minor version at a time, drain Nodes before maintenance, restart kubelet after upgrading it, and uncordon Nodes after successful verification.