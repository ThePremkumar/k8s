# Kubernetes Cluster Diagnostic & Administration Log

A structured collection of live terminal traces documenting common node status checks, image pull failures, cordoning operations, replica scaling, and deployment recovery using backup YAML files.

---

## 1. Check Node Status & kubelet Health

Verify all nodes in the cluster and check their status:
```bash
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   14d   v1.21.2
worker   Ready    <none>                 14d   v1.21.2
```

Check the `kubelet` service daemon health status:
```bash
[root@master ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enab>
     Drop-In: /usr/lib/systemd/system/kubelet.service.d
              └─10-kubeadm.conf
     Active: active (running) since Tue 2025-01-21 02:09:38 UTC; 5>
       Docs: https://kubernetes.io/docs/
   Main PID: 1629 (kubelet)
      Tasks: 15 (limit: 2253)
     Memory: 126.3M
        CPU: 10.887s
     CGroup: /system.slice/kubelet.service
```

---

## 2. Describe Worker Node Details

Inspect resource capacities, allocations, conditions, and active pods on the worker node:
```bash
[root@master ~]# kubectl describe node worker
```

**Output:**
```text
Name:               worker
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=worker
                    kubernetes.io/os=linux
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"1a:c0:90:8e:f3:2c"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 192.168.10.154
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Mon, 06 Jan 2025 02:36:32 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  worker
  AcquireTime:     <unset>
  RenewTime:       Tue, 21 Jan 2025 02:18:14 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 21 Jan 2025 02:10:26 +0000   Tue, 21 Jan 2025 02:10:26 +0000   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Tue, 21 Jan 2025 02:15:09 +0000   Mon, 06 Jan 2025 02:36:32 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 21 Jan 2025 02:15:09 +0000   Mon, 06 Jan 2025 02:36:32 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 21 Jan 2025 02:15:09 +0000   Mon, 06 Jan 2025 02:36:32 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 21 Jan 2025 02:15:09 +0000   Mon, 20 Jan 2025 02:01:00 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.10.154
  Hostname:    worker
Capacity:
  cpu:                2
  ephemeral-storage:  8310764Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1947072Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  7659200090
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1844672Ki
  pods:               110
System Info:
  Machine ID:                 ec2d43c1f0bee4f10162af8556decacf
  System UUID:                ec2d43c1-f0be-e4f1-0162-af8556decacf
  Boot ID:                    33e40397-8994-4fb2-a69f-e7cc55b668fc
  Kernel Version:             6.1.119-129.201.amzn2023.x86_64
  OS Image:                   Amazon Linux 2023.6.20241212
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://25.0.6
  Kubelet Version:            v1.21.2
  Kube-Proxy Version:         v1.21.2
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                 ------------  ----------  ---------------  -------------  ---
  besant                      multicontainer                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         12d
  besant                      myapp-deployment-558547cf67-ggd7b    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  besant                      myapp-deployment-558547cf67-kwrfq    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  besant                      myapp-deployment-558547cf67-srfkp    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  besant                      nginx                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         12d
  default                     nginx                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         12d
  kube-flannel                kube-flannel-ds-wh24b                100m (5%)     0 (0%)      50Mi (2%)        0 (0%)         14d
  kube-system                 coredns-558bd4d5db-4nl7g             100m (5%)     0 (0%)      70Mi (3%)        170Mi (9%)     14d
  kube-system                 kube-proxy-9k2qj                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         14d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                200m (10%)  0 (0%)
  memory             120Mi (6%)  170Mi (9%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:
  Type    Reason                   Age                    From        Message
  ----    ------                   ----                   ----        -------
  Normal  Starting                 8m41s                  kubelet     Starting kubelet.
  Normal  NodeAllocatableEnforced  8m40s                  kubelet     Updated Node Allocatable limit across pods
  Normal  NodeHasSufficientPID     8m32s (x7 over 8m41s)  kubelet     Node worker status is now: NodeHasSufficientPID
  Normal  NodeHasSufficientMemory  8m16s (x8 over 8m41s)  kubelet     Node worker status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    8m16s (x8 over 8m41s)  kubelet     Node worker status is now: NodeHasNoDiskPressure
  Normal  Starting                 8m2s                   kube-proxy  Starting kube-proxy.
```

---

## 3. Check Image pull and Pod Events

Testing pod deployment with a non-existent image (`xyz`) in the `besant` namespace:
```bash
[root@master ~]# kubectl  run mypod --image=xyz -n besant
pod/mypod created
```

Check the active pods status:
```bash
[root@master ~]# kubectl get pods -n besant
NAME                                READY   STATUS         RESTARTS   AG                                             E
multicontainer                      2/2     Running        4          12                                             d
myapp-deployment-558547cf67-ggd7b   1/1     Running        1          23                                             h
myapp-deployment-558547cf67-kwrfq   1/1     Running        1          23                                             h
myapp-deployment-558547cf67-srfkp   1/1     Running        1          23                                             h
mypod                               0/1     ErrImagePull   0          2s
nginx                               1/1     Running        2          12                                             d
```

Describe `mypod` to troubleshoot the `ImagePullBackOff` failure:
```bash
[root@master ~]# kubectl describe pod mypod -n besant
```

**Output:**
```text
Name:         mypod
Namespace:    besant
Priority:     0
Node:         worker/192.168.10.154
Start Time:   Tue, 21 Jan 2025 02:19:37 +0000
Labels:       run=mypod
Annotations:  <none>
Status:       Pending
IP:           10.244.1.30
IPs:
  IP:  10.244.1.30
Containers:
  mypod:
    Container ID:
    Image:          xyz
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fmwf6 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  kube-api-access-fmwf6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  86s                default-scheduler  Successfully assigned besant/mypod to worker
  Normal   Pulling    42s (x3 over 86s)  kubelet            Pulling image "xyz"
  Warning  Failed     42s (x3 over 86s)  kubelet            Failed to pull image "xyz": rpc error: code = Unknown desc = Error response from daemon: pull access denied for xyz, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     42s (x3 over 86s)  kubelet            Error: ErrImagePull
  Normal   BackOff    2s (x5 over 84s)   kubelet            Back-off pulling image "xyz"
  Warning  Failed     2s (x5 over 84s)   kubelet            Error: ImagePullBackOff
```

Deploying a functional `nginx` container:
```bash
[root@master ~]# kubectl  run mypod1 --image=nginx -n besant
pod/mypod1 created
```

Check the updated pods status:
```bash
[root@master ~]# kubectl get pods -n besant
NAME                                READY   STATUS             RESTARTS   AGE
multicontainer                      2/2     Running            4          12d
myapp-deployment-558547cf67-ggd7b   1/1     Running            1          23h
myapp-deployment-558547cf67-kwrfq   1/1     Running            1          23h
myapp-deployment-558547cf67-srfkp   1/1     Running            1          23h
mypod                               0/1     ImagePullBackOff   0          2m48s
mypod1                              0/1     Pending            0          8s
nginx                               1/1     Running            2          12d
```

---

## 4. Cordon & Uncordon worker Node

Disable scheduling on the worker node:
```bash
[root@master ~]# kubectl cordon worker
node/worker already cordoned
```

Check node list and scheduling state:
```bash
[root@master ~]# k get nodes
NAME     STATUS                     ROLES                  AGE   VERSION
master   Ready                      control-plane,master   14d   v1.21.2
worker   Ready,SchedulingDisabled   <none>                 14d   v1.21.2
```

Uncordon the worker node to allow pod scheduling:
```bash
[root@master ~]# kubectl uncordon worker
node/worker uncordoned
```

Check status again:
```bash
[root@master ~]# k get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   14d   v1.21.2
worker   Ready    <none>                 14d   v1.21.2
```

Check active pods:
```bash
[root@master ~]# kubectl get pods -n besant
NAME                                READY   STATUS             RESTARTS   AGE
multicontainer                      2/2     Running            4          12d
myapp-deployment-558547cf67-ggd7b   1/1     Running            1          23h
myapp-deployment-558547cf67-kwrfq   1/1     Running            1          23h
myapp-deployment-558547cf67-srfkp   1/1     Running            1          23h
mypod                               0/1     ImagePullBackOff   0          5m16s
mypod1                              0/1     Pending            0          2m36s
nginx                               1/1     Running            2          12d
```

---

## 5. Scale Deployment Replicas

Check deployment size:
```bash
[root@master ~]# kubectl get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           24h
```

Scale down deployment to 1 replica:
```bash
[root@master ~]# kubectl scale deploy myapp-deployment --replicas=1 -n besant
deployment.apps/myapp-deployment scaled
```

Check active deployments and pods during termination:
```bash
[root@master ~]# kubectl get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   1/1     1            1           24h

[root@master ~]# k get pods -n besant
NAME                                READY   STATUS             RESTARTS   AGE
multicontainer                      2/2     Running            4          12d
myapp-deployment-558547cf67-ggd7b   0/1     Terminating        1          24h
myapp-deployment-558547cf67-kwrfq   0/1     Terminating        1          24h
myapp-deployment-558547cf67-srfkp   1/1     Running            1          24h
mypod                               0/1     ImagePullBackOff   0          9m4s
mypod1                              1/1     Running            0          6m24s
nginx                               1/1     Running            2          12d
```

---

## 6. Backup and Re-apply Deployment Manifest

Retrieve deployment list:
```bash
[root@master ~]# k get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           24h
```

Dump the active deployment specification to standard output:
```bash
[root@master ~]# k get deploy -n besant myapp-deployment -oyaml
```

**Output:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"myapp"},"name":"myapp-deployment","namespace":"besant"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"myapp"}},"template":{"metadata":{"labels":{"app":"myapp"}},"spec":{"containers":[{"image":"httpd","name":"myapp","ports":[{"containerPort":80}]}]}}}}
  creationTimestamp: "2025-01-20T02:24:04Z"
  generation: 5
  labels:
    app: myapp
  name: myapp-deployment
  namespace: besant
  resourceVersion: "19382"
  uid: 895e4cb9-4214-473d-83cb-30e1051a8a07
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp
    spec:
      containers:
      - image: httpd
        imagePullPolicy: Always
        name: myapp
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2025-01-21T02:29:29Z"
    lastUpdateTime: "2025-01-21T02:29:29Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2025-01-20T02:24:04Z"
    lastUpdateTime: "2025-01-21T02:33:17Z"
    message: ReplicaSet "myapp-deployment-66975fdfb7" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 5
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

Export deployment to a backup file:
```bash
[root@master ~]# k get deploy -n besant  myapp-deployment -oyaml > savedeploy.yaml
```

List active local files:
```bash
[root@master ~]# ll
total 44
-rw-r--r--. 1 root root  178 Jan 20 02:34 1
-rw-r--r--. 1 root root  335 Jan 21 02:33 deployment.yaml
-rw-r--r--. 1 root root 5816 Jan 20 02:53 k8s_svc.txt
-rw-r--r--. 1 root root   61 Jan  8 02:22 my-namespace.yaml
-rw-r--r--. 1 root root  322 Jan  9 02:17 nginx-pod.yml
-rw-r--r--. 1 root root 2066 Jan 21 02:35 savedeploy.yaml
-rw-r--r--. 1 root root  203 Jan 20 02:47 svc_Hard_code_NodePort.yaml
-rw-r--r--. 1 root root  182 Jan 20 02:40 svc_LoadBalancer.yaml
-rw-r--r--. 1 root root  178 Jan 20 02:34 svc_NodePort.yaml
-rw-r--r--. 1 root root  180 Jan 20 02:44 svc_clusterip.yaml
```

Verify deployment exists:
```bash
[root@master ~]# k get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           24h
```

Delete the deployment:
```bash
[root@master ~]# k delete deploy myapp-deployment -n besant
deployment.apps "myapp-deployment" deleted
```

Verify resource removal:
```bash
[root@master ~]# k get deploy -n besant
No resources found in besant namespace.
```

List local files:
```bash
[root@master ~]# ls
1                  nginx-pod.yml                svc_NodePort.yaml
deployment.yaml    savedeploy.yaml              svc_clusterip.yaml
k8s_svc.txt        svc_Hard_code_NodePort.yaml
my-namespace.yaml  svc_LoadBalancer.yaml
```

Restore deployment using backup YAML:
```bash
[root@master ~]# k apply -f savedeploy.yaml -n besant
deployment.apps/myapp-deployment created
```

Check the deployment status after restore:
```bash
[root@master ~]# k get deploy -n besant
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           3s
```

---

## 7. Print Join Worker Node Command

Generate a token and print the bootstrap command required to join worker nodes:
```bash
[root@master ~]# kubeadm token create --print-join-command
```

**Output:**
```text
kubeadm join 192.168.10.175:6443 --token nm6bh8.4pqr583abi157d8x --discovery-token-ca-cert-hash sha256:4f8cf1eae483ba83d9551620354448390ad0c45dde0fbc40a69b51891f9e1d68
```
