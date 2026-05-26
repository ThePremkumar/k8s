# Nginx Ingress Controller for Kubernetes

Nginx Ingress Controller for Kubernetes provides enterprise-grade delivery services for Kubernetes applications, with benefits for users of both open-source Nginx and Nginx Plus.

With the Nginx Ingress Controller, you get basic load balancing, SSL/TLS termination, support for URI rewrites, and upstream SSL/TLS encryption.

---

## Installation

### Step 1: Clone nginx controller repository

```bash
git clone https://github.com/srinisbook/kuberntes-nginx-controller.git
```

### Step 2: Install nginx controller

```bash
cd kuberntes-nginx-controller
kubectl apply -f nginx-ingress.yaml
```

---

## Verification

### Check the pods:

```bash
kubectl get pods -n ingress-nginx
```

**Output:**
```text
NAME                                        READY     STATUS    RESTARTS   AGE
nginx-ingress-controller-65fd579494-f4nfv   1/1       Running   0          70s
```

### Check the services:

```bash
kubectl get services -n ingress-nginx
```

**Output:**
```text
NAME          TYPE         CLUSTER-IP   EXTERNAL-IP      PORT(S) AGE
ingress-nginx LoadBalancer 10.12.10.200 34.69.178.130 80:30903/TCP,443:30605/TCP 2m47s
```

---

**Reference:**
[medium.com/avmconsulting-blog/nginx-ingress-controller-in-kubernetes-8eb14f737f7b](https://medium.com/avmconsulting-blog/nginx-ingress-controller-in-kubernetes-8eb14f737f7b)
