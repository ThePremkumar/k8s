# Kubernetes Basic Troubleshooting Guide

This guide covers fundamental troubleshooting steps for diagnosing issues in a Kubernetes cluster, from high-level cluster health checks down to container-level logs, resource constraints, and scheduling anomalies.

---

## 1. Check Cluster Health

Start by confirming the cluster itself is reachable and healthy.

```bash
kubectl cluster-info
kubectl get componentstatuses     # (deprecated but still useful on some clusters)
kubectl get nodes
```

### What to look for:
* API server is reachable.
* All nodes are in `Ready` state.
* No widespread master or control-plane component failures.

---

## 2. Node-Level Troubleshooting

If pods are not scheduling or are frequently restarting, check the nodes.

```bash
kubectl describe node <node-name>
```

### 🔍 Common Issues
* `NotReady` node status.
* `DiskPressure` or `MemoryPressure` alerts.
* Subnet or CNI network issues.
* `kubelet` service not running on node host.

### 📌 Remediation for Unhealthy Nodes
1. Restart the `kubelet` service:
   ```bash
   systemctl restart kubelet
   ```
2. Check kubelet system logs:
   ```bash
   journalctl -u kubelet -f
   ```
3. Validate underlying cloud provider or VM virtual machine health.

---

## 3. Pod Status Investigation

Most deployment and runtime issues surface at the pod level.

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

### 🛑 Common Pod States & Causes

| Status | Likely Cause |
| :--- | :--- |
| `Pending` | No nodes/resources available, or PVC mounting issues. |
| `CrashLoopBackOff` | Application crash, bad configuration, or missing env vars. |
| `ImagePullBackOff` | Incorrect image name/tag, or registry authentication failure. |
| `Error` | Application initialization or startup failure. |

---

## 4. Check Pod Logs

If the pod is running but behaving unexpectedly, inspect its logs.

```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

For checking logs of restarting/crashed containers:
```bash
kubectl logs <pod-name> --previous -n <namespace>
```

### Look for:
* Application runtime crashes or unhandled exceptions.
* Missing config files, volumes, or secrets.
* Database or external service connection failures.
* Security or system permission errors.

---

## 5. Resource Constraints (CPU / Memory)

Pods may fail to start or terminate abruptly due to resource constraints.

```bash
kubectl describe pod <pod-name>
```

### 🔍 Check
* **OOMKilled:** The container was terminated because it exceeded its allocated memory limit.
* **CPU Throttling:** Pod is running slowly due to hard CPU limits.
* **Requests vs. Limits:** Misaligned resources preventing scheduling.

### 💡 Quick Action
* Adjust the CPU/Memory requests and limits in the deployment manifest.

---

## 6. Networking & Service Issues

If the application is running but is unreachable externally or internally:

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service-name>
```

Check the endpoints to ensure pods are registered to the service:
```bash
kubectl get endpoints <service-name>
```

---

## 7. Deployment & Rollout Problems

If a recent application deployment update failed:

```bash
kubectl get deploy -n <namespace>
kubectl rollout status deploy/<deployment-name>
```

To view the rollout history:
```bash
kubectl rollout history deploy/<deployment-name>
```

To rollback to a previous stable revision:
```bash
kubectl rollout undo deploy/<deployment-name>
```

---

## 8. Events: The Fastest Clue

Always check cluster events—they record cluster lifecycle actions and warning logs.

```bash
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

### 📌 Events Reveal
* Image pull failures.
* Scheduler resource bottlenecks.
* Volume mounting and storage controller issues.

---

## 9. Quick "Golden Checklist"

When things break, ask these 5 questions:
1. Is the cluster & node healthy?
2. Is the pod running?
3. Are the logs clean?
4. Are configurations and secrets correct?
5. Can the service reach the network?

---

## 10. Practical Scenario: Cordoning a Worker Node

Below is a trace demonstrating how to cordon, test, and uncordon a worker node.

### 10.1 Check Node Status

```bash
[root@master ~]# k get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   11d   v1.21.2
worker   Ready    <none>                 11d   v1.21.2
```

### 10.2 Cordon the Worker Node

Disable scheduling on the worker node:
```bash
[root@master ~]# k cordon worker
node/worker cordoned
```

Verify status changes to `SchedulingDisabled`:
```bash
[root@master ~]# k get nodes
NAME     STATUS                     ROLES                  AGE   VERSION
master   Ready                      control-plane,master   11d   v1.21.2
worker   Ready,SchedulingDisabled   <none>                 11d   v1.21.2
```

### 10.3 Inspect Scheduling Consequences

If you delete pods, new replicas will remain `Pending` because the master has taints and the worker node is cordoned:

```bash
[root@master ~]# k get pods -n besant
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-845d4d9dff-2c2kt   0/1     Pending   0          62s
nginx-deployment-845d4d9dff-l2csh   0/1     Pending   0          62s
nginx-deployment-845d4d9dff-lnrvd   0/1     Pending   0          62s
nginx-deployment-845d4d9dff-ms2xj   0/1     Pending   0          62s
nginx-deployment-845d4d9dff-wk2f5   0/1     Pending   0          62s
```

Describe a pending pod to see the scheduling events:
```text
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  14s (x5 over 32s)  default-scheduler  0/2 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 1 node(s) were unschedulable.
```

### 10.4 Uncordon the Worker Node

Enable scheduling back on the node:
```bash
[root@master ~]# k uncordon worker
node/worker uncordoned
```

Check nodes status:
```bash
[root@master ~]# k get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   11d   v1.21.2
worker   Ready    <none>                 11d   v1.21.2
```

Observe pods successfully transition to running:
```bash
[root@master ~]# k get pods -n besant
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-845d4d9dff-2c2kt   0/1     Pending   0          2m15s
nginx-deployment-845d4d9dff-l2csh   1/1     Running   0          2m15s
nginx-deployment-845d4d9dff-lnrvd   1/1     Running   0          2m15s
nginx-deployment-845d4d9dff-ms2xj   1/1     Running   0          2m15s
nginx-deployment-845d4d9dff-wk2f5   0/1     Pending   0          2m15s
```

```bash
[root@master ~]# k get pods -n besant
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-845d4d9dff-2c2kt   0/1     ContainerCreating   0          2m22s
nginx-deployment-845d4d9dff-l2csh   1/1     Running             0          2m22s
nginx-deployment-845d4d9dff-lnrvd   1/1     Running             0          2m22s
nginx-deployment-845d4d9dff-ms2xj   1/1     Running             0          2m22s
nginx-deployment-845d4d9dff-wk2f5   0/1     ContainerCreating   0          2m22s
```
