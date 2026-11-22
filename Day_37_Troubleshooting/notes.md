# Kubernetes Application Troubleshooting – Example Voting App

## 1. Introduction

In this task, we deploy a sample distributed voting application on a Kubernetes cluster and troubleshoot the intentional errors introduced in its Kubernetes manifests.

Repository:

https://github.com/piyushsachdeva/example-voting-app

The repository is a modified fork of Docker's Example Voting App. It contains intentional configuration errors to demonstrate real-world Kubernetes application troubleshooting.

### Objectives

1. Clone the Git repository.
2. Deploy the application using Kubernetes manifests.
3. Identify Pods that are not running correctly.
4. Troubleshoot the application using Kubernetes commands.
5. Identify and fix configuration errors.
6. Verify that the complete application works correctly.

---

## 2. Understanding the Application Architecture

The Example Voting App consists of five main components.

| Component | Purpose |
|---|---|
| Vote | Frontend application where users submit votes |
| Redis | Stores incoming votes temporarily in a queue |
| Worker | Processes votes from Redis and stores them in PostgreSQL |
| Database (DB) | PostgreSQL database that stores votes |
| Result | Frontend application that displays voting results |

### Application Workflow

```text
                USER
                  |
          +-------+-------+
          |               |
          v               v
       VOTE APP        RESULT APP
          |               ^
          v               |
        REDIS             |
          |               |
          v               |
        WORKER            |
          |               |
          v               |
       POSTGRESQL --------+
         DATABASE
```

### How the application works

1. A user opens the Vote application and selects an option.
2. The Vote application sends the vote to Redis.
3. Redis stores the vote in its queue.
4. The Worker reads the vote from Redis.
5. The Worker stores the processed vote in PostgreSQL.
6. The Result application reads the voting results from PostgreSQL and displays them.

**Important:** All five components must communicate correctly for the complete application to work.

---

## 3. Clone the Git Repository

### Step 1: Login to the Kubernetes cluster

Verify that your Kubernetes cluster is accessible:

```bash
kubectl get nodes
```

All expected nodes should be in the `Ready` state.

Check the current Kubernetes context:

```bash
kubectl config current-context
```

### Step 2: Clone the repository

```bash
git clone https://github.com/piyushsachdeva/example-voting-app
```

### Step 3: Navigate to the cloned directory

```bash
cd example-voting-app
```

### Step 4: Explore the repository

```bash
ls
```

Check the Kubernetes manifests:

```bash
ls k8s-specifications/
```

The `k8s-specifications` directory contains the Kubernetes YAML manifests used to deploy the application.

---

## 4. Deploy the Application

Apply all Kubernetes manifests from the directory:

```bash
kubectl create -f k8s-specifications/
```

This command creates the Kubernetes resources defined in the YAML files.

The resources include Deployments and Services for the different application components.

**Note:** The command creates resources in the current namespace. If no namespace is configured, Kubernetes uses the `default` namespace.

### Verify the created resources

Check Pods:

```bash
kubectl get pods
```

Check Deployments:

```bash
kubectl get deployments
```

Check Services:

```bash
kubectl get services
```

Check all resources:

```bash
kubectl get all
```

At this stage, some Pods may not be in the `Running` or `Ready` state because the repository intentionally contains errors.

Do not immediately delete or recreate everything. First investigate the failures.

---

## 5. Kubernetes Application Troubleshooting

Troubleshooting should follow a systematic approach.

### Step 1: Check Pod status

```bash
kubectl get pods
```

Common Pod statuses:

| Status | Meaning |
|---|---|
| Pending | Pod has not been scheduled or is waiting for required resources |
| ContainerCreating | Container is being created |
| Running | Pod is running, but it may not be Ready |
| CrashLoopBackOff | Container repeatedly crashes and restarts |
| ImagePullBackOff | Kubernetes cannot pull the container image |
| ErrImagePull | Image pull operation failed |
| CreateContainerConfigError | Container configuration is invalid |

Check Pods with additional details:

```bash
kubectl get pods -o wide
```

This displays the node on which each Pod is running, along with its IP address.

### Step 2: Describe the failing Pod

```bash
kubectl describe pod <pod-name>
```

Example:

```bash
kubectl describe pod vote-xxxxxxxxxx-xxxxx
```

The `describe` command provides useful information such as:

- Container state
- Restart count
- Image details
- Environment variables
- Volume mounts
- Scheduling information
- Events

Pay special attention to the **Events** section at the bottom of the output.

Events often reveal the reason for a Pod failure.

### Step 3: Check container logs

```bash
kubectl logs <pod-name>
```

For example:

```bash
kubectl logs <pod-name>
```

If a Pod contains multiple containers:

```bash
kubectl logs <pod-name> -c <container-name>
```

If a container has restarted, inspect its previous logs:

```bash
kubectl logs <pod-name> --previous
```

Logs help identify application-level errors such as:

- Connection refused
- Database authentication failures
- Redis connection failures
- Missing environment variables
- Application startup errors

### Step 4: Check Deployment status

```bash
kubectl get deployments
```

Describe a Deployment:

```bash
kubectl describe deployment <deployment-name>
```

Check rollout status:

```bash
kubectl rollout status deployment/<deployment-name>
```

This helps identify whether the Deployment successfully created its desired number of replicas.

---

## 6. Troubleshooting Common Kubernetes Configuration Errors

The repository contains intentional errors. The actual mistakes should be identified by inspecting the manifests and comparing them with the Pod status, events, and logs.

The following are common configuration problems to check.

### A. Incorrect Image Name or Tag

A Deployment may reference an incorrect container image.

Check:

```bash
kubectl describe pod <pod-name>
```

Look for errors such as:

```text
ErrImagePull
ImagePullBackOff
```

Inspect the image configured in the Deployment:

```bash
kubectl get deployment <deployment-name> -o yaml
```

Check whether the image name and tag are valid.

### B. Incorrect Environment Variables

Applications need correct environment variables to communicate with other components.

For example, the Vote application needs to connect to Redis.

Check the Deployment configuration:

```bash
kubectl describe deployment vote
```

Look for environment variables containing service hostnames, ports, and other application configuration.

Compare the configured values with the actual Kubernetes Service names.

### C. Incorrect Service Configuration

A Service provides stable network access to Pods.

Check Services:

```bash
kubectl get services
```

Describe a Service:

```bash
kubectl describe service <service-name>
```

Check the Service selector and its endpoints:

```bash
kubectl get endpoints <service-name>
```

On newer Kubernetes versions, EndpointSlices can also be inspected:

```bash
kubectl get endpointslices
```

If a Service has no endpoints, check:

- Whether matching Pods exist.
- Whether the Service selector matches the Pod labels.
- Whether the target Pods are Ready.

### D. Incorrect Port Configuration

Check the Service configuration:

```bash
kubectl get service <service-name> -o yaml
```

Understand the difference:

| Field | Meaning |
|---|---|
| port | Port exposed by the Service |
| targetPort | Port on which the application container listens |
| nodePort | Port exposed on each node for a NodePort Service |

The Service's `targetPort` must match the application's listening port.

### E. Incorrect Database Configuration

The Worker and Result applications depend on PostgreSQL.

Check the database Pod:

```bash
kubectl get pods
```

Inspect its logs:

```bash
kubectl logs <db-pod-name>
```

Check the database Service:

```bash
kubectl get service db
```

Verify database-related environment variables and credentials in the application manifests.

### F. Incorrect Redis Configuration

The Vote application and Worker depend on Redis.

Check the Redis Pod:

```bash
kubectl get pods
```

Check its logs:

```bash
kubectl logs <redis-pod-name>
```

Check the Redis Service:

```bash
kubectl get service redis
```

Verify that the application components are using the correct Redis Service hostname and port.

---

## 7. Fixing Errors in Kubernetes Manifests

Once an error is identified, inspect the corresponding YAML file.

For example:

```bash
cat k8s-specifications/<file-name>.yaml
```

Edit the manifest:

```bash
vi k8s-specifications/<file-name>.yaml
```

Or open the repository in VS Code:

```bash
code .
```

Make the necessary correction.

### Apply the updated manifest

For an existing resource, use:

```bash
kubectl apply -f k8s-specifications/<file-name>.yaml
```

Check whether the resource has updated:

```bash
kubectl get pods
```

If the change updates a Deployment's Pod template, Kubernetes normally creates a new ReplicaSet and replaces the old Pods through a rolling update.

Check the rollout:

```bash
kubectl rollout status deployment/<deployment-name>
```

**Important:** Do not blindly modify every manifest. Identify the actual error first and then apply the smallest necessary correction.

---

## 8. Verify the Complete Application

After fixing the errors, verify all application components.

### Check Pods

```bash
kubectl get pods
```

All expected application Pods should be in the `Running` state and show the expected Ready containers.

### Check Deployments

```bash
kubectl get deployments
```

The READY count should match the desired replica count for each Deployment.

### Check Services

```bash
kubectl get services
```

Verify that all expected Services exist.

### Check Service endpoints

```bash
kubectl get endpoints
```

Confirm that Services have endpoints for their matching ready Pods.

### Test application access

The upstream Docker Example Voting App documentation describes these default NodePort endpoints:

- Vote application: port `31000`
- Result application: port `31001`

The modified fork may change these settings, so inspect the actual Service configuration first:

```bash
kubectl get services -o wide
```

If the Services use these NodePorts and your cluster exposes them to your machine, access:

```text
http://<node-ip>:31000
http://<node-ip>:31001
```

For a local kind cluster, access depends on the cluster's port mappings and networking configuration. You may use port forwarding instead:

```bash
kubectl port-forward service/vote 8080:80
```

Then open:

```text
http://localhost:8080
```

Use the actual Service port shown by `kubectl get service vote`.

For the Result application:

```bash
kubectl port-forward service/result 8081:80
```

Then open:

```text
http://localhost:8081
```

Again, replace `80` with the actual Service port if it is different.

### Functional testing

1. Open the Vote application.
2. Submit a vote.
3. Open the Result application.
4. Verify that the submitted vote appears in the results.
5. Check Pod logs if the vote is not processed or displayed.

---

## 9. Useful Troubleshooting Commands

### General cluster inspection

```bash
kubectl get nodes
kubectl get pods -o wide
kubectl get deployments
kubectl get services
kubectl get all
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Pod troubleshooting

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

### Deployment troubleshooting

```bash
kubectl describe deployment <deployment-name>
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
```

### Service troubleshooting

```bash
kubectl describe service <service-name>
kubectl get endpoints <service-name>
kubectl get endpointslices
```

### Inspect Kubernetes configuration

```bash
kubectl get deployment <deployment-name> -o yaml
kubectl get service <service-name> -o yaml
```

---

## 10. Cleanup

After completing the troubleshooting exercise, remove the application resources if you no longer need them.

From inside the cloned repository directory:

```bash
kubectl delete -f k8s-specifications/
```

Verify that the resources have been deleted:

```bash
kubectl get all
```

This removes the resources defined in the manifests. It does not delete the Git repository from your machine.

To remove the cloned repository itself, navigate to its parent directory and delete the folder.

---

## 11. Conclusion

In this task, we deployed a distributed voting application on Kubernetes and practiced troubleshooting intentional configuration errors.

We learned how to:

1. Clone a Git repository and deploy Kubernetes manifests.
2. Understand communication between multiple application components.
3. Inspect Pod status, events, and container logs.
4. Troubleshoot Deployments and Services.
5. Identify incorrect images, environment variables, selectors, and port configurations.
6. Update Kubernetes manifests and verify rollouts.
7. Test application connectivity and functionality.

**Key learning:** Kubernetes troubleshooting should be systematic. Start by checking the resource status, inspect events and logs, identify the root cause, fix the configuration, and verify the application again.

Reference repository:

https://github.com/piyushsachdeva/example-voting-app

Original Docker Example Voting App:

https://github.com/dockersamples/example-voting-app