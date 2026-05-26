# Kubernetes Deployments

A Deployment manages a set of Pods to run an application workload, usually one that doesn't maintain state. It provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

---

## Use Cases

Typical use cases for Deployments include:
* Create a Deployment to rollout a ReplicaSet. The ReplicaSet creates Pods in the background. Check the status of the rollout to see if it succeeds or not.
* Declare the new state of the Pods by updating the `PodTemplateSpec` of the Deployment. A new ReplicaSet is created and the Deployment manages moving the Pods from the old ReplicaSet to the new one at a controlled rate. Each new ReplicaSet updates the revision of the Deployment.
* Rollback to an earlier Deployment revision if the current state of the Deployment is not stable. Each rollback updates the revision of the Deployment.
* Scale up the Deployment to facilitate more load.
* Pause the rollout of a Deployment to apply multiple fixes to its `PodTemplateSpec` and then resume it to start a new rollout.
* Use the status of the Deployment as an indicator that a rollout has stuck.
* Clean up older ReplicaSets that you don't need anymore.

---

## Creating a Deployment

The following is an example of a Deployment. It creates a ReplicaSet to bring up three Nginx Pods.

### Step 1: Create the Manifest

```bash
vi my-deploy.yaml
```

**`my-deploy.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Manifest Details:
* A Deployment named `nginx-deployment` is created, indicated by the `.metadata.name` field. This name will become the basis for the ReplicaSets and Pods which are created later.
* The Deployment creates a ReplicaSet that creates three replicated Pods, indicated by the `.spec.replicas` field.
* The `.spec.selector` field defines how the created ReplicaSet finds which Pods to manage. In this case, you select a label that is defined in the Pod template (`app: nginx`).
* **Note on matchLabels:** The `.spec.selector.matchLabels` field is a map of `{key,value}` pairs. A single `{key,value}` in the `matchLabels` map is equivalent to an element of `matchExpressions`, whose key field is `"key"`, the operator is `"In"`, and the values array contains only `"value"`. All requirements must be satisfied in order to match.
* The `.spec.template` field indicates:
  * The Pods are labeled `app: nginx` using the `.metadata.labels` field.
  * The Pod template's specification indicates that the Pods run one container, `nginx`, which runs the `nginx` Docker Hub image at version `1.14.2`.
  * The container is named `nginx` using the `.spec.containers[0].name` field.

### Step 2: Apply the Deployment

Before you begin, make sure your Kubernetes cluster is up and running. Follow the steps given below to create the above Deployment:

```bash
kubectl apply -f my-deployment.yaml
```

---

## Checking Deployment Status

Run `kubectl get deployments` to check if the Deployment was created. If the Deployment is still being created, the output is similar to the following:

```bash
kubectl get deployments
```
**Output:**
```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```

### Understanding the Fields:
* **NAME:** Lists the names of the Deployments in the namespace.
* **READY:** Displays how many replicas of the application are available to your users. It follows the pattern `ready/desired`.
* **UP-TO-DATE:** Displays the number of replicas that have been updated to achieve the desired state.
* **AVAILABLE:** Displays how many replicas of the application are available to your users.
* **AGE:** Displays the amount of time that the application has been running.

Run the command again a few seconds later. The output should update:
```bash
kubectl get deployments
```
**Output:**
```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           18s
```

To see the labels automatically generated for each Pod, run:
```bash
kubectl get pods --show-labels
```
**Output:**
```text
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
```

---

## Scaling Deployments

You can scale up or down the number of deployment replicas.

### Scale to 0 Replicas
```bash
[root@master ~]# k scale deploy nginx-deployment --replicas=0 -n ns
deployment.apps/nginx-deployment scaled
```

Check the pods status:
```bash
[root@master ~]# k get pods -n ns
NAME                                READY   STATUS        RESTARTS   AGE
multi-c                             2/2     Running       2          24h
nginx                               1/1     Running       1          23h
nginx-deployment-66b6c48dd5-2svng   0/1     Terminating   0          2m31s
nginx-deployment-66b6c48dd5-gnqrt   0/1     Terminating   0          2m31s
nginx-deployment-66b6c48dd5-tkfr5   0/1     Terminating   0          2m31s
nginx-deployment-66b6c48dd5-wmq9r   0/1     Terminating   0          95s
nginx-deployment-66b6c48dd5-ww6z2   0/1     Terminating   0          95s
nginx-duplicate                     1/1     Running       1          23h
```

```bash
[root@master ~]# k get pods -n ns
NAME              READY   STATUS    RESTARTS   AGE
multi-c           2/2     Running   2          24h
nginx             1/1     Running   1          23h
nginx-duplicate   1/1     Running   1          23h
```

Verify deployment state:
```bash
[root@master ~]# k get deploy -n ns
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/0     0            0           8m38s
```

### Scale to 1 Replica
```bash
[root@master ~]# k scale deploy nginx-deployment --replicas=1 -n ns
deployment.apps/nginx-deployment scaled
```

Check deploy and pods status:
```bash
[root@master ~]# k get deploy -n ns
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           9m9s

[root@master ~]# k get pods -n ns
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-lpcww   1/1     Running   0          8s
```

### Scale to 3 Replicas
```bash
[root@master ~]# k scale deploy nginx-deployment --replicas=3 -n ns
deployment.apps/nginx-deployment scaled
```

Check status:
```bash
[root@master ~]# k get deploy -n ns
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           9m38s

[root@master ~]# k get pods -n ns
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-blnkm   1/1     Running   0          6s
nginx-deployment-66b6c48dd5-lpcww   1/1     Running   0          36s
nginx-deployment-66b6c48dd5-z479r   1/1     Running   0          6s
```

---

## Updating a Deployment

> [!NOTE]
> A Deployment's rollout is triggered if and only if the Deployment's Pod template (that is, `.spec.template`) is changed—for example, if the labels or container images of the template are updated. Scaling does not trigger a rollout.

### Update Container Image
Let's update the Nginx Pods to use the `httpd` image instead of the `nginx:1.14.2` image:

```bash
[root@master ~]# kubectl set image deploy nginx-deployment nginx=httpd -n ns
deployment.apps/nginx-deployment image updated
```

Watch the rolling update:
```bash
[root@master ~]# watch kubectl get pods -n ns
```

Check the updated pods details:
```bash
[root@master ~]# k get pods -n ns -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-6f6dbc8579-6rjhp   1/1     Running   0          26s   10.244.1.40   worker   <none>           <none>
nginx-deployment-6f6dbc8579-g88wd   1/1     Running   0          26s   10.244.1.39   worker   <none>           <none>
nginx-deployment-6f6dbc8579-nxr6j   1/1     Running   0          24s   10.244.1.42   worker   <none>           <none>
nginx-deployment-6f6dbc8579-spfrv   1/1     Running   0          26s   10.244.1.38   worker   <none>           <none>
nginx-deployment-6f6dbc8579-zmjrv   1/1     Running   0          24s   10.244.1.41   worker   <none>           <none>
```

Verify service is responding with the new image contents:
```bash
[root@master ~]# curl 10.244.1.41
<html><body><h1>It works!</h1></body></html>
```

Check rollout status:
```bash
[root@master ~]# kubectl rollout status deployment/nginx-deployment -n ns
deployment "nginx-deployment" successfully rolled out
```

---

## Rolling Back to a Previous Revision

If the rollout was unstable or done in error, you can roll back the Deployment from the current version to the previous version:

```bash
[root@master ~]# kubectl rollout undo deployment/nginx-deployment -n ns
deployment.apps/nginx-deployment rolled back
```

Watch the rollback progress:
```bash
[root@master ~]# watch kubectl get pods -n ns
```

Verify pods are running the previous configuration:
```bash
[root@master ~]# k get pods -n ns -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-845d4d9dff-cc66g   1/1     Running   0          23s   10.244.1.46   worker   <none>           <none>
nginx-deployment-845d4d9dff-dg666   1/1     Running   0          23s   10.244.1.47   worker   <none>           <none>
nginx-deployment-845d4d9dff-fpph2   1/1     Running   0          25s   10.244.1.44   worker   <none>           <none>
nginx-deployment-845d4d9dff-kw9m4   1/1     Running   0          25s   10.244.1.45   worker   <none>           <none>
nginx-deployment-845d4d9dff-s5rvm   1/1     Running   0          25s   10.244.1.43   worker   <none>           <none>
```

Verify the image content returned (back to Nginx):
```bash
[root@master ~]# curl 10.244.1.45
```
**Output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
