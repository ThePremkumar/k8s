# Kubernetes Service

Expose an application running in your cluster behind a single outward-facing endpoint, even when the workload is split across multiple backends.

In Kubernetes, a Service is a method for exposing a network application that is running as one or more Pods in your cluster.

---

## Service Types

In Kubernetes, there are different ways to expose applications running on pods within a cluster depending on the desired access level:

* **ClusterIP:**
  * The default service type.
  * Only accessible from within the Kubernetes cluster.
  * Useful for internal pod-to-pod communication.

* **NodePort:**
  * Exposes a service on a specific port on every node in the cluster.
  * Allows external access by connecting to any node's IP address and the designated NodePort.

* **LoadBalancer:**
  * Creates a dedicated external load balancer managed by the underlying cloud provider.
  * Provides a single publicly accessible IP address to access the service, with traffic distributed across pods.

---

## Hands-on Service Configuration Trace

Below is a step-by-step console session demonstrating how to manage Deployments and expose them using various Service types in the `besant` namespace.

### 1. Check Initial Cluster State

Check cluster nodes:
```bash
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   13d   v1.21.2
worker   Ready    <none>                 13d   v1.21.2
```

Check namespaces:
```bash
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
besant            Active   12d
default           Active   13d
kube-flannel      Active   13d
kube-node-lease   Active   13d
kube-public       Active   13d
kube-system       Active   13d
```

Check existing pods in the `besant` namespace:
```bash
[root@master ~]# kubectl get pods -n besant
NAME             READY   STATUS    RESTARTS   AGE
multicontainer   2/2     Running   2          11d
nginx            1/1     Running   1          11d
```

---

### 2. Deploy a Sample Application

Create a Deployment manifest file:
```bash
[root@master ~]# cat deployment.yaml
```

**`deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx
        ports:
        - containerPort: 80
```

Apply the deployment in the `besant` namespace:
```bash
[root@master ~]# kubectl create -f deployment.yaml  -n besant
deployment.apps/myapp-deployment created
```

Verify deployment and pods:
```bash
[root@master ~]# k get pods -n besant
NAME                                READY   STATUS    RESTARTS   AGE
multicontainer                      2/2     Running   2          11d
myapp-deployment-558547cf67-6bt76   1/1     Running   0          24s
myapp-deployment-558547cf67-9cdpl   1/1     Running   0          24s
myapp-deployment-558547cf67-whfjd   1/1     Running   0          24s
nginx                               1/1     Running   1          11d

[root@master ~]# k get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           2m28s
```

```bash
[root@master ~]# k get pods -n besant
NAME                                READY   STATUS    RESTARTS   AGE
multicontainer                      2/2     Running   2          11d
myapp-deployment-558547cf67-ggd7b   1/1     Running   0          75s
myapp-deployment-558547cf67-kwrfq   1/1     Running   0          75s
myapp-deployment-558547cf67-srfkp   1/1     Running   0          75s
nginx                               1/1     Running   1          11d
```

Check initial services in the `besant` namespace:
```bash
[root@master ~]# k get svc -n besant
No resources found in besant namespace.
```

---

### 3. Expose as ClusterIP (Internal Access)

Show ClusterIP manifest file:
```bash
[root@master ~]# cat svc_clusterip.yaml
```

**`svc_clusterip.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply the ClusterIP Service:
```bash
[root@master ~]# k create -f svc_clusterip.yaml -n besant
service/myapp-service created
```

Check service list:
```bash
[root@master ~]# k get svc -n besant
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
myapp-service   ClusterIP   10.111.235.83   <none>        80/TCP    7s
```

Verify internal accessibility via cluster IP:
```bash
[root@master ~]# curl 10.111.235.83
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

---

### 4. Expose as NodePort (External Node IP Access)

Show NodePort manifest file:
```bash
[root@master ~]# cat svc_NodePort.yaml
```

**`svc_NodePort.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

Apply the NodePort Service:
```bash
[root@master ~]# kubectl create -f svc_NodePort.yaml  -n besant
service/myapp-service created
```

Check service list:
```bash
[root@master ~]# kubectl get svc -n besant
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp-service   NodePort   10.101.76.156   <none>        80:30327/TCP   39s
```

---

### 5. Expose as LoadBalancer (Cloud Managed Access)

Show LoadBalancer manifest file:
```bash
[root@master ~]# cat svc_LoadBalancer.yaml
```

**`svc_LoadBalancer.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lbmyapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

Apply the LoadBalancer Service:
```bash
[root@master ~]# k create -f svc_LoadBalancer.yaml -n besant
service/lbmyapp-service created
```

Check service list:
```bash
[root@master ~]# k get svc -n besant
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
lbmyapp-service   LoadBalancer   10.109.54.159   <pending>     80:32031/TCP   7s
myapp-service     NodePort       10.101.76.156   <none>        80:30327/TCP   5m40s
```

---

### 6. Expose as Hardcoded NodePort (Static NodePort Assignment)

Show hardcoded NodePort manifest file:
```bash
[root@master ~]# cat svc_Hard_code_NodePort.yaml
```

**`svc_Hard_code_NodePort.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hc-myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
  type: NodePort
```

Apply the hardcoded NodePort Service:
```bash
[root@master ~]# k create -f svc_Hard_code_NodePort.yaml -n besant
service/hc-myapp-service created
```

Check the final list of active services:
```bash
[root@master ~]# k get svc -n besant
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
clmyapp-service    ClusterIP      10.106.100.85    <none>        80/TCP         3m16s
hc-myapp-service   NodePort       10.108.228.220   <none>        80:30007/TCP   7s
lbmyapp-service    LoadBalancer   10.109.54.159    <pending>     80:32031/TCP   6m42s
myapp-service      NodePort       10.101.76.156    <none>        80:30327/TCP   12m
```
