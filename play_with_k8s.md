# Playing with Kubernetes: Commands & Concepts

This comprehensive cheat sheet contains practical commands, multi-node configuration procedures, YAML templates, and testing steps for namespaces, pods, services, replication controllers, deployments, and node taints.

---

## 1. Cluster Status

Quickly check pods in all namespaces and check node readiness status:

```bash
# Get pods in all namespaces
kubectl get pods --all-namespaces

# Get all nodes
kubectl get nodes
```

---

## 2. Namespace Operations

Namespaces isolate resources within a cluster. Here are two ways to create, view, and delete namespaces.

### Method A: Creating Namespace via YAML Manifest

1. Create a namespace manifest:
   ```bash
   vi my-namespace.yaml 
   ```
2. **`my-namespace.yaml`:**
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: mynamespace
   ```
3. Apply the manifest:
   ```bash
   kubectl create -f my-namespace.yaml
   ```

### Method B: Creating Namespace Imperatively

To create a namespace in a single line command:
```bash
kubectl create namespace <insert-namespace-name-here>
```

### Viewing and Deleting Namespaces

To list all namespaces:
```bash
kubectl get namespaces
```

To delete a namespace:
```bash
kubectl delete namespaces <insert-some-namespace-name>
```

---

## 3. Pod Operations

Pods are the basic deployable units. Below are instructions for running pods imperatively and declaratively.

### 3.1 Imperative: Run a Pod in One Line

Run Nginx in the `ns` namespace:
```bash
[root@master ~]# kubectl run mypod --image=nginx -n ns
pod/mypod created
```

Check the status:
```bash
[root@master ~]# kubectl get pods -n ns
NAME    READY   STATUS    RESTARTS   AGE
mypod   1/1     Running   0          3s
```

Check details with wide output:
```bash
[root@master ~]# kubectl get pods -n ns -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
mypod   1/1     Running   0          8s    10.244.1.8   worker   <none>           <none>
```

Test access inside cluster:
```bash
[root@master ~]# curl 10.244.1.8
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
```

---

### 3.2 Declarative: Run a Pod via YAML

1. Create a pod manifest:
   ```bash
   vi nginx-pod.yml
   ```
2. **`nginx-pod.yml`:**
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx
     labels:
       apps: nginx
   spec:
     containers:
     - name: nginx
       image: nginx
       imagePullPolicy: IfNotPresent
       ports:
       - containerPort: 80
   ```
3. Apply the pod configuration:
   ```bash
   kubectl create -f nginx-pod.yml
   ```

### 3.3 Diagnostic Pod Check

```bash
# List the pods
kubectl get pods 

kubectl get pods -o wide

kubectl describe pods nginx

curl <POD_IP>
```

---

## 4. Exposing Pods using Services

Services provide stable endpoints to reach pods. Here are the different ways to expose the pod.

### 4.1 Imperative: Exposing Pod Internally (ClusterIP)
```bash
kubectl expose pod nginx --port=80 --name=nginx-http

kubectl get svc

curl <SVC_IP>
```

### 4.2 Imperative: Exposing Pod Externally (NodePort)
Note that NodePort values must be between `30000-32767` by default:
```bash
kubectl expose pod nginx --type=NodePort

kubectl get svc

curl <SVC_IP>

kubectl delete svc nginx-http
```

### 4.3 Declarative: Exposing Pod as LoadBalancer
To create an external load balancer, add `type: LoadBalancer` to your Service manifest:

1. Create a service manifest:
   ```bash
   vi nginx-svc.yml
   ```
2. **`nginx-svc.yml`:**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
     labels:
       apps: nginx
   spec:
     ports:
     - protocol: TCP
       port: 80
       targetPort: 80
     selector:
       apps: nginx
     type: LoadBalancer
   ```
3. Apply and verify:
   ```bash
   kubectl create -f nginx-svc.yml

   kubectl delete -f nginx-svc.yml
   ```

---

### 4.4 Declarative: Exposing Pod as static NodePort

You can assign a specific static NodePort within the range `30000-32767`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    apps: nginx
  ports:
  - nodePort: 30007
    port: 80
    protocol: TCP
    targetPort: 80
```

Apply and clean up resources:
```bash
kubectl create -f nginx-svc.yml

kubectl delete -f nginx-pod.yml

kubectl get svc
kubectl get pod
```

---

## 5. Replication Controller Operations

Replication Controllers ensure that a specified number of pod replicas are running at any given time.

1. Create a Replication Controller manifest:
   ```bash
   vi nginx-rs.yml
   ```
2. **`nginx-rs.yml`:**
   ```yaml
   apiVersion: v1
   kind: ReplicationController
   metadata:
     name: nginx
   spec:
     replicas: 3
     selector:
       apps: nginx
     template:
       metadata:
         name: nginx
         labels:
           apps: nginx
       spec:
         containers:
         - name: nginx
           image: nginx
           imagePullPolicy: IfNotPresent
           ports:
           - containerPort: 80
   ```
3. Apply and check details:
   ```bash
   kubectl create -f nginx-rs.yml

   kubectl get rc -o wide

   kubectl describe rc nginx

   kubectl get pods

   kubectl get pods -o wide
   ```

Exposing the ReplicationController via a LoadBalancer service:

1. Edit the service manifest:
   ```bash
   vi nginx-svc.yml
   ```
2. **`nginx-svc.yml`:**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
     labels:
       apps: nginx
   spec:
     ports:
     - protocol: TCP
       port: 80
       targetPort: 80
     selector:
       apps: nginx
     type: LoadBalancer
   ```
3. Create and verify:
   ```bash
   kubectl create -f nginx-svc.yml

   kubectl get svc
   ```

---

## 6. Deployments

Deployments provide declarative updates for Pods and ReplicaSets.

### 6.1 Create Deployment Imperatively
```bash
kubectl create deployment nginx --image=nginx
```

### 6.2 Create Deployment Declaratively

1. Create a deployment manifest:
   ```bash
   vi deploymnet.yaml
   ```
2. **`deploymnet.yaml`:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
     labels:
       app: nginx
   spec:
     replicas: 1
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
           image: nginx:latest
           ports:
           - containerPort: 80
   ```

*Reference template:* [Guestbook Frontend Deployment](https://github.com/kubernetes/examples/blob/master/guestbook/frontend-deployment.yaml)

---

### 6.3 Scaling Deployments

To scale a deployment imperatively to 3 replicas:
```bash
kubectl scale deploy --replicas=3 nginx
```

---

### 6.4 Connecting Ingress to Deployment Services

Now, in order to create an Ingress, we need to create a Service first. Our `svc.yml` looks as follows:

1. Create a service manifest:
   ```bash
   vi svc.yml 
   ```
2. **`svc.yml`:**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
       labels:
           app: nginx
       name: nginx-svc
   spec:
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
           app: nginx
       type: ClusterIP
   ```
3. Apply service:
   ```bash
   k apply -f svc.yaml
   ```

**Ingress Controllers:** Setup and bind Ingress path-rules to your Service targets.

---

## 7. Master Taints and Administration Commands

### 7.1 Master Node Cordoning & Untainting
```bash
# Remove the taint from master node (i.e. Allow pods to be scheduled on the master)
kubectl taint nodes $(hostname) node-role.kubernetes.io/master-

# Add the taint back again
kubectl taint nodes $(hostname) $(hostname)=DoNotSchedulePods:NoSchedule
```

### 7.2 kubeadm Reset Operations
```bash
# To reset "kubeadm init"
kubectl drain $(hostname) --delete-local-data --force --ignore-daemonsets
kubectl delete node $(hostname)
sudo kubeadm reset
```

Inspect Flannel Network CNI configurations:
```bash
cat /run/flannel/subnet.env
```
