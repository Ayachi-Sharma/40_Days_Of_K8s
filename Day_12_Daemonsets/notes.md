# Kubernetes: DaemonSet and CronJob

# Part 1: DaemonSet

## 1. What Is a DaemonSet?

A **DaemonSet** ensures that a copy of a particular Pod runs on each eligible node in a Kubernetes cluster.

DaemonSets are commonly used for node-level services, such as:
- Log collection agents
- Monitoring agents
- Node networking components - weave-net/calico/flannel
- Security agents
- control-plane controller - kube-proxy

## 2. Create a DaemonSet YAML File

Create a file named `daemonset.yaml`.

Example configuration using Nginx:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      app: nginx-daemon
  template:
    metadata:
      labels:
        app: nginx-daemon
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

## 3. Create the DaemonSet

Apply the YAML file:
```powershell
kubectl apply -f daemonset.yaml
```

Expected output:
```text
daemonset.apps/nginx-daemonset created
```

## 4. Verify the DaemonSet

Check its status:
```powershell
kubectl get daemonsets
```

Inspect its details:
```powershell
kubectl describe daemonset nginx-daemonset
```

List the Pods created by the DaemonSet:
```powershell
kubectl get pods -o wide -l app=nginx-daemon
```

Check the cluster nodes:
```powershell
kubectl get nodes
```

The number of DaemonSet Pods depends on the number of **eligible nodes**, not simply the number of Deployments or namespaces. For example, if your cluster has one eligible node, you would normally expect one DaemonSet Pod.

---

# Part 2: Cron Syntax and CronJob

## 1. What Is a CronJob?

A **CronJob** creates Kubernetes Jobs according to a specified schedule.
A Job runs a task to completion. A CronJob schedules these Jobs repeatedly.

Common use cases include:
- Database backups
- Periodic reports
- Scheduled cleanup tasks
- Automated maintenance scripts

## 2. Understand Cron Syntax

A cron expression contains five fields:

```text
* * * * *
┬ ┬ ┬ ┬ ┬
│ │ │ │ └── Day of the week
│ │ │ └──── Month
│ │ └────── Day of the month
│ └──────── Hour
└────────── Minute
```
| Minute | `0–59` | 
| Hour | `0–23` | 
| Day of month | `1–31` |
| Month | `1–12` | 
| Day of week | `0–6` | 

### Common Cron Examples

| `* * * * *` | Every minute |
| `*/5 * * * *` | Every five minutes |
| `0 * * * *` | At the beginning of every hour |
| `0 9 * * *` | Every day at 9:00 AM |
| `0 0 * * 0` | Every Sunday at midnight |

---

## 3. Create the CronJob YAML File

Create a file named `cronjob.yaml`.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kubernetes-cronjob
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: busybox
              image: busybox:1.36
              command:
                - /bin/sh
                - -c
                - 'echo "40daysofkubernetes"'
          restartPolicy: OnFailure
```

**Important:** The command prints the message and exits. The Job completes successfully when the command succeeds.

---

## 4. Create the CronJob

Apply the YAML file:

```powershell
kubectl apply -f cronjob.yaml
```

Expected output:
```text
cronjob.batch/kubernetes-cronjob created
```

Verify:
```powershell
kubectl get cronjobs
```

Inspect the schedule and configuration:
```powershell
kubectl describe cronjob kubernetes-cronjob
```

---

## 5. Verify Job Execution

Because the CronJob runs every five minutes, a new Job will be scheduled at the next matching five-minute mark. You do not necessarily need to wait a full five minutes if you create it just before that mark.

List Jobs:

```powershell
kubectl get jobs
```

List Pods created by the Jobs:
```powershell
kubectl get pods
```

To see the output, first identify a Pod created by the CronJob and then run:

```powershell
kubectl logs <pod-name>
```

Expected log output:
```text
40daysofkubernetes
```

Replace `<pod-name>` with the actual Pod name.

### Useful Commands

Check the most recent Jobs:

```powershell
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

Watch Jobs as they appear:

```powershell
kubectl get jobs -w
```

Press `Ctrl+C` to stop watching.

Check CronJob events and scheduling details:

```powershell
kubectl describe cronjob kubernetes-cronjob
```

**Note:** Kubernetes may retain completed Job and Pod objects for some time. The CronJob’s history limits can be configured if you want to control how many completed Jobs are retained.

---

## 6. Troubleshooting

### CronJob Exists but No Job Has Appeared

Check:

```powershell
kubectl describe cronjob kubernetes-cronjob
kubectl get events --sort-by=.metadata.creationTimestamp
```

Possible causes include waiting for the next scheduled time, a suspended CronJob, or scheduling restrictions.

---

## 7. Clean Up

Delete the CronJob:

```powershell
kubectl delete cronjob kubernetes-cronjob
```
Deleting the CronJob stops future scheduled executions. Existing Jobs and Pods may remain, so delete them separately if you want a complete cleanup.

Delete the sample DaemonSet:
```powershell
kubectl delete daemonset nginx-daemonset
```

Verify:
```powershell
kubectl get cronjobs
kubectl get daemonsets
kubectl get jobs
kubectl get pods
```

---

## Final Summary

1. A DaemonSet maintains a Pod on each eligible node.
2. A Deployment manages a specified number of replicas, while a DaemonSet targets eligible nodes.
3. A CronJob creates Jobs on a schedule.
4. Cron syntax uses five fields: minute, hour, day of month, month, and day of week.
5. `*/5 * * * *` means every five minutes.
6. BusyBox can run a short shell command such as `echo "40daysofkubernetes"`.
7. Use `kubectl get jobs` and `kubectl logs` to verify that scheduled tasks execute successfully.