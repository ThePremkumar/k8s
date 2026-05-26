# Complete Kubernetes Administration & Reference Guide

This comprehensive reference document contains detailed commands, network port rules, initialization procedures, YAML templates, and troubleshooting steps for setting up and managing a multi-node Kubernetes cluster.

---

## 1. Master & Worker Port Requirements

To ensure proper cluster communication, configure your host firewalls or security groups to open the following inbound ports.

### Master Nodes
* **TCP Inbound 2379 – 2380:** etcd server client API
* **TCP Inbound 6443:** Kubernetes API server
* **TCP Inbound 10250:** Kubelet API
* **TCP Inbound 10251:** kube-scheduler
* **TCP Inbound 10252:** kube-controller-manager

### Worker Nodes
* **TCP Inbound 10250:** Kubelet API
* **TCP Inbound 30000–32767:** NodePort Services (Default port range)

---

## 2. Master Node Bootstrapping (Ubuntu)

Follow these steps on your master node to install Docker, configure Kubernetes packages, initialize the control plane, and set up local authentication.

### Step 2.1: Initial System Preparation
```bash
# Become root user
sudo su

# Add IP-hostname entry to hosts file
echo -e "\n$(hostname -i) \t $(hostname)\n" >> /etc/hosts

cat /etc/hosts

# Update package lists
apt-get update
```

### Step 2.2: Install Docker Engine (Recommended version 18.06)
```bash
apt-get install -y apt-transport-https
curl -s https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
apt update
apt install -y docker-ce=18.06.0~ce~3-0~ubuntu
usermod -aG docker ubuntu
```

### Step 2.3: Install Kubernetes Components
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt-get update

apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubelet kubeadm kubectl
```

### Step 2.4: Initialize Cluster Control Plane
```bash
# Initialize Kubernetes:
# kubeadm init --apiserver-advertise-address=$(hostname -i) --pod-network-cidr=10.244.0.0/16
kubeadm init --pod-network-cidr=10.244.0.0/16
```
> [!NOTE]
> Note down the join token returned at the end of the initialization output. Example:
> `kubeadm join 172.31.44.222:6443 --token u3emp1.v1apk68iwuhpugt4 --discovery-token-ca-cert-hash sha256:bfadbe7f4ad864567cb75bc96e877ff91902320866d29e1c75dab9ecd1ea745f`

### Step 2.5: Configure Local Non-Root kubectl Access
```bash
su - ubuntu
docker info

# Granting k8s access to a normal user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

cat $HOME/.kube/config
```

### Step 2.6: CNI Network Fabric Deployment (Flannel)
Verify initial pods are up before applying network fabric:
```bash
kubectl get pods --all-namespaces
```

Deploy Flannel L3 network fabric overlay:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Verify CNI deployment and check active nodes:
```bash
# Get pods in all namespaces
kubectl get pods --all-namespaces

# Get all nodes
kubectl get nodes
```

---

## 3. Worker Node Setup & Integration

Execute these commands on your worker hosts to prepare them and join the cluster control plane.

### Step 3.1: Install Docker & Kubernetes on Worker
```bash
# Become root user
sudo su

# Add IP-hostname entry to hosts file
echo -e "\n$(hostname -i) \t $(hostname)\n" >> /etc/hosts

# Update package lists
apt-get update

# Install Docker
apt-get install -y apt-transport-https
curl -s https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
apt update
apt install -y docker-ce=18.06.0~ce~3-0~ubuntu
usermod -aG docker ubuntu
docker info

# Install Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubelet kubeadm kubectl
```

### Step 3.2: Join the Cluster
Use the token recorded from the `kubeadm init` output to join:
```bash
kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

### Step 3.3: (Optional) Generate Join Token on Master Node
If you do not have the token, generate the correct command dynamically from the master host:
```bash
token=$(kubeadm token list | sed '1d' | head -1| awk '{print $1}')
hash=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | awk '{print $2}')
ipaddress=$(hostname -i)
echo -e "\n\nkubeadm join $ipaddress:6443 --token $token --discovery-token-ca-cert-hash sha256:$hash\n\n";
unset token hash
```

---

## 4. Verification & Basic Object Management

Verify node connectivity and test the deployment of Pods, Replication Controllers, and Services.

### Step 4.1: Cluster Verification
```bash
# Get all nodes
kubectl get nodes -o wide

# Get pods in all namespaces
kubectl get pods --all-namespaces -o wide
```

### Step 4.2: Deploying a Pod
```bash
git clone https://github.com/Kalki5/temp
cd temp
ll
```

Create a custom pod manifest file:
```bash
vim nginx-pod.yml
```

**`nginx-pod.yml`:**
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

Deploy, verify, and describe the pod:
```bash
kubectl create -f nginx-pod.yml

kubectl get pods
kubectl get pods -o wide
kubectl describe pods nginx
curl <POD_IP>
```

Expose the pod internally:
```bash
kubectl expose pod nginx --port=80 --name=nginx-http
kubectl get svc
curl <SVC_IP>
```

Clean up resources:
```bash
kubectl delete svc nginx-http
kubectl delete -f nginx-pod.yml
kubectl get svc
kubectl get pod
```

---

### Step 4.3: Deploying a ReplicationController
Create a ReplicationController manifest:
```bash
vim nginx-rs.yml
```

**`nginx-rs.yml`:**
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

Deploy and check the status:
```bash
kubectl create -f nginx-rs.yml
kubectl get rc -o wide
kubectl describe rc nginx
kubectl get pods
kubectl get pods -o wide
```

---

### Step 4.4: Deploying a Service (LoadBalancer)
Create a Service manifest:
```bash
vim nginx-svc.yml
```

**`nginx-svc.yml`:**
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

Deploy and verify service:
```bash
kubectl create -f nginx-svc.yml
kubectl get svc
```

To diagnose processes inside containers:
```bash
# Kill nginx containers
htop
```

Deploying and exposing an Apache service using imperative commands:
```bash
kubectl create deployment apache --image=httpd
kubectl get deployments
kubectl create service nodeport apache --tcp=80:80
kubectl get svc
```

---

## 5. Control Plane Node Taints & Node Operations

Kubernetes masters are tainted by default to prevent scheduling workloads. Below are commands to manage taints and reset cluster states.

### Step 5.1: Describe and Check Master Taints
```bash
kubectl describe nodes $(hostname)
kubectl describe nodes $(hostname) | grep Taints
```

### Step 5.2: Remove the Taint (Allow Pods on Master)
```bash
# Remove the taint from master node (i.e. Allow pods to be scheduled on the master)
kubectl taint nodes $(hostname) node-role.kubernetes.io/master-
```

### Step 5.3: Re-apply the Master Taint
```bash
# Add the taint back again
kubectl taint nodes $(hostname) $(hostname)=DoNotSchedulePods:NoSchedule
```

### Step 5.4: Reset Cluster Node (Revert kubeadm init)
```bash
# To reset "kubeadm init"
kubectl drain $(hostname) --delete-local-data --force --ignore-daemonsets
kubectl delete node $(hostname)
sudo kubeadm reset
```

Inspect Flannel network interface configuration:
```bash
cat /run/flannel/subnet.env
```

---

## 6. Kubernetes Dashboard & Monitoring Add-ons

Install and configure the Kubernetes Web Dashboard UI alongside Heapster for container monitoring.

### Step 6.1: Deploy Kubernetes Dashboard
```bash
vim kubernetes-dashboard.yaml
```

**`kubernetes-dashboard.yaml`:**
```yaml
# ------------------- Dashboard Secret ------------------- #
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
# ------------------- Dashboard Service Account ------------------- #
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
# ------------------- Dashboard Role & Role Binding ------------------- #
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
# ------------------- Dashboard Deployment ------------------- #
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
# ------------------- Dashboard Service ------------------- #
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

Apply and check Dashboard:
```bash
kubectl create -f kubernetes-dashboard.yaml
kubectl get deployment -n kube-system -o wide
```

---

### Step 6.2: Deploy Heapster Monitoring Add-on
```bash
vim heapster.yaml
```

**`heapster.yaml`:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: gcr.io/google_containers/heapster-amd64:v1.4.2
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes.summary_api:''?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250&insecure=true
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
```

Apply and verify Heapster:
```bash
kubectl create -f heapster.yaml
kubectl get -f heapster.yaml
```

Assign stats permissions to system heapster:
```bash
kubectl edit clusterrole system:heapster
```
*Verify the apiGroups block contains:*
```yaml
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
```

---

### Step 6.3: Dashboard Admin Service Account Creation
```bash
vim kubernetes-dashboard-admin.yaml
```

**`kubernetes-dashboard-admin.yaml`:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

Apply manifest and retrieve token:
```bash
kubectl create -f kubernetes-dashboard-admin.yaml

kubectl -n kube-system get secret | grep admin-user

kubectl -n kube-system describe secret <SECRET_NAME>

# Alternative dynamic retrieval:
# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

Access Dashboard externally using proxy or port-forward:
```bash
# kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'
# kubectl -n kube-system edit service kubernetes-dashboard
# kubectl -n kube-system get service kubernetes-dashboard
# netstat -plnt

kubectl get svc -n kube-system
```

---

### Step 6.4: Alternate Deployment Trace (Web UI v1.10.1)
```bash
kubectl get pods --all-namespaces

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Create admin user:
```bash
vim dashboard-adminuser.yaml
```

**`dashboard-adminuser.yaml`:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

Apply admin user manifest:
```bash
kubectl apply -f dashboard-adminuser.yaml
```

Create rolebinding:
```bash
vim dashboad-rolebinding.yaml
```

**`dashboad-rolebinding.yaml`:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

Apply rolebinding:
```bash
kubectl apply -f  dashboad-rolebinding.yaml
```

Start the proxy server to allow external connection:
```bash
kubectl proxy --address='0.0.0.0' --accept-hosts='.*'
```
