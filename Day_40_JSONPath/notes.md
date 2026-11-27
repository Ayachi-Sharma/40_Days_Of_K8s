# Kubernetes – JSON, JSONPath and kubectl Queries

## 1. Introduction

In this task, we learn JSON fundamentals, Kubernetes JSONPath expressions, and how to retrieve and sort Kubernetes resource information using `kubectl`.

### Objectives

1. Understand JSON structure and syntax.
2. Understand JSONPath expressions used with kubectl.
3. Sort Running Pods by creation timestamp.
4. Sort Nodes by CPU capacity using custom columns.
5. Count schedulable Nodes, excluding Nodes with `NoSchedule` taints, and save the count to a file.

---

## 2. JSON Fundamentals

### What is JSON?

JSON stands for **JavaScript Object Notation**.

It is a lightweight data-interchange format used to store and exchange structured information.

Kubernetes API objects can be represented in JSON format.

### JSON Example

```json
{
  "name": "nginx",
  "namespace": "default",
  "replicas": 3,
  "running": true,
  "ports": [80, 443],
  "labels": {
    "app": "nginx"
  }
}
```

### Important JSON Data Types

| Data Type | Description                       | Example           |
|-----------|-----------------------------------|-------------------|
| String    | Text enclosed in double quotes    | `"nginx"`         |
| Number    | Integer or decimal                | `3`, `10.5`       |
| Boolean   | True or false                     | `true`            |
| Object    | Collection of key-value pairs     | `{"app":"nginx"}` |
| Array     | Ordered collection of values      | `[80,443]`        |
| Null      | Represents an empty value         | `null`            |

### JSON Structure

- **Object:** Contains key-value pairs enclosed in `{ }`.
- **Array:** Contains multiple values enclosed in `[ ]`.
- **Key:** Identifies a value in an object.
- **Value:** Stores the associated data.

---

## 3. JSON in Kubernetes

Kubernetes API objects contain structured information.

For example, retrieve Nodes in JSON format:

```bash
kubectl get nodes -o json
```

Retrieve Pods in JSON format:

```bash
kubectl get pods -A -o json
```

Retrieve a particular Node:

```bash
kubectl get node <node-name> -o json
```

Common Kubernetes JSON fields:

| Field                         | Description                       |
|-------------------------------|-----------------------------------|
| `.metadata.name`              | Resource name                     |
| `.metadata.namespace`         | Namespace                         |
| `.metadata.creationTimestamp` | Creation time                     |
| `.spec`                       | Desired configuration             |
| `.status`                     | Current resource status           |
| `.status.capacity.cpu`        | Total CPU capacity of a Node      |
| `.spec.taints`                | Node taints                       |
| `.spec.unschedulable`         | Whether scheduling is disabled    |

---

## 4. JSONPath Fundamentals

### What is JSONPath?

JSONPath is a query language used to extract specific information from JSON documents.

In Kubernetes, JSONPath is supported by `kubectl` to retrieve selected fields without displaying the entire JSON object.

General syntax:

```bash
kubectl get <resource> -o jsonpath='<expression>'
```

### Common JSONPath Expressions

| Expression            | Purpose                                       |
|-----------------------|-----------------------------------------------|
| `.metadata.name`      | Retrieves resource name                       |
| `.metadata.namespace` | Retrieves namespace                           |
| `.items[*]`           | Selects all items                             |
| `.items[0]`           | Selects the first item                        |
| `[*]`                 | Selects all elements in an array              |
| `..name`              | Searches recursively for fields named name    |
| `?()`                 | Filters objects based on a condition          |
| `range` and `end`     | Iterates through a list                       |

### Examples

Get all Node names:

```bash
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
```

Get the creation timestamp of all Pods:

```bash
kubectl get pods -A -o jsonpath='{.items[*].metadata.creationTimestamp}'
```

Get all Pod names with a new line after each name:

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

Get all Node names and their CPU capacity:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\n"}{end}'
```

**Note:** Kubernetes JSONPath supports additional syntax such as `range`, `end`, and filters. It does not support every feature of standard JSONPath, including regular expressions.

---

## 5. Practical Task 1 – Sort Running Pods by Creation Timestamp

### Requirement

Sort all Running Pods across all namespaces based on their creation timestamp.

### Command

```bash
kubectl get pods -A --field-selector=status.phase=Running --sort-by=.metadata.creationTimestamp
```

### Explanation

- `kubectl get pods`: Retrieves Pods.
- `-A`: Includes all namespaces.
- `--field-selector=status.phase=Running`: Selects only Running Pods.
- `--sort-by=.metadata.creationTimestamp`: Sorts Pods by creation timestamp in ascending order.

The oldest Running Pod appears first, and the newest appears last.

To display selected columns:

```bash
kubectl get pods -A --field-selector=status.phase=Running --sort-by=.metadata.creationTimestamp -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,CREATED:.metadata.creationTimestamp
```

---

## 6. Practical Task 2 – Sort Nodes by CPU Capacity

### Requirement

Sort Nodes based on CPU capacity and display only two columns:

- NODE
- CPU

### Command

```bash
kubectl get nodes --sort-by=.status.capacity.cpu -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu
```

### Explanation

- `kubectl get nodes`: Retrieves all Nodes.
- `--sort-by=.status.capacity.cpu`: Sorts Nodes using their CPU capacity field.
- `-o custom-columns`: Displays selected fields in a custom table.
- `NODE:.metadata.name`: Displays the Node name.
- `CPU:.status.capacity.cpu`: Displays the total CPU capacity.

Example output:

```text
NODE                CPU
worker-node-1       2
worker-node-2       4
control-plane       4
```

The values above are illustrative.

**Note:** `capacity.cpu` represents the total CPU capacity, not the currently available CPU. Kubernetes represents CPU capacity as a resource quantity. For numeric sorting across multi-digit values, use a numeric-aware sorting method if the displayed order is not numeric.

---

## 7. Practical Task 3 – Count Schedulable Nodes Excluding NoSchedule Taints

### Requirement

Find the number of schedulable Nodes, excluding:

1. Nodes marked unschedulable.
2. Nodes having a taint with the effect `NoSchedule`.

Save the final count to a file.

### Step 1: Check all Nodes

```bash
kubectl get nodes
```

### Step 2: Inspect Node taints

```bash
kubectl describe nodes
```

Or inspect taints directly:

```bash
kubectl get nodes -o custom-columns=NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable,TAINTS:.spec.taints
```

A `NoSchedule` taint prevents Pods that do not tolerate that taint from being scheduled on the Node.

Example:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

### Step 3: Check jq availability

The following solution uses `jq` to filter the JSON returned by Kubernetes.

```bash
jq --version
```

If jq is not installed, install it using your Linux distribution's package manager.

### Step 4: Count eligible Nodes and save the result

```bash
kubectl get nodes -o json | jq '[.items[] | select((.spec.unschedulable // false) == false and ([.spec.taints[]?.effect] | index("NoSchedule") == null))] | length' > schedulable-node-count.txt
```

### Explanation

- `kubectl get nodes -o json`: Retrieves all Nodes as JSON.
- `.items[]`: Iterates through every Node.
- `.spec.unschedulable // false`: Treats an absent `unschedulable` field as `false`.
- `select(...)`: Keeps only Nodes satisfying the conditions.
- `.spec.taints[]?.effect`: Retrieves taint effects, if present.
- `index("NoSchedule") == null`: Excludes Nodes having a `NoSchedule` taint.
- `length`: Counts the remaining Nodes.
- `>`: Writes the count into the output file.

### Step 5: Verify the file

```bash
cat schedulable-node-count.txt
```

Example output:

```text
2
```

The number above is only an example; your cluster may return a different count.

**Important:** This task deliberately excludes every Node with a `NoSchedule` taint, even if a particular workload could tolerate that taint. `PreferNoSchedule` is not excluded by this filter.

---

## 8. Important Commands Summary

### JSON and JSONPath

```bash
kubectl get nodes -o json
kubectl get pods -A -o json
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### Task 1 – Sort Running Pods

```bash
kubectl get pods -A --field-selector=status.phase=Running --sort-by=.metadata.creationTimestamp
```

### Task 2 – Sort Nodes by CPU

```bash
kubectl get nodes --sort-by=.status.capacity.cpu -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu
```

### Task 3 – Count Schedulable Nodes

```bash
kubectl get nodes -o json | jq '[.items[] | select((.spec.unschedulable // false) == false and ([.spec.taints[]?.effect] | index("NoSchedule") == null))] | length' > schedulable-node-count.txt
```

Verify:

```bash
cat schedulable-node-count.txt
```

---

## 9. Conclusion

In this task, we learned:

1. JSON objects, arrays, keys, values, and data types.
2. How Kubernetes resources are represented in JSON.
3. How to extract resource information using JSONPath.
4. How to sort Running Pods by creation timestamp.
5. How to sort Nodes by CPU capacity using custom columns.
6. How to filter Nodes based on scheduling status and taints.
7. How to save command output into a file.