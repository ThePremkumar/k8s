# How to Use Nginx Ingress Controller

This guide explains how to configure and deploy a Kubernetes Ingress using the Nginx Ingress Controller.

---

## 1. Creating a Kubernetes Ingress

First, let’s create two separate services to demonstrate how Ingress routes requests. We will run two web applications that output slightly different responses using the `http-echo` container.

### Step 1.1: Create Apple Application Manifest

```bash
vi apple.yaml
```

**`apple.yaml`:**
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: apple-app
  labels:
    app: apple
spec:
  containers:
    - name: apple-app
      image: hashicorp/http-echo
      args:
        - "-text=apple"

---

kind: Service
apiVersion: v1
metadata:
  name: apple-service
spec:
  selector:
    app: apple
  ports:
    - port: 5678 # Default port for image
```

### Step 1.2: Create Banana Application Manifest

```bash
vi banana.yaml
```

**`banana.yaml`:**
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: banana-app
  labels:
    app: banana
spec:
  containers:
    - name: banana-app
      image: hashicorp/http-echo
      args:
        - "-text=banana"

---

kind: Service
apiVersion: v1
metadata:
  name: banana-service
spec:
  selector:
    app: banana
  ports:
    - port: 5678 # Default port for image 
```

---

## 2. Deploy the Applications

Apply the manifests to create the pods and services in your cluster:

```bash
kubectl apply -f apple.yaml
kubectl apply -f banana.yaml
```

---

## 3. Declare the Ingress Rules

We will declare an Ingress resource to route requests starting with `/apple` to the `apple-service`, and requests starting with `/banana` to the `banana-service`.

```bash
vi ingress.yaml
```

**`ingress.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /apple
        pathType: Prefix
        backend:
          service:
            name: apple-service
            port:
              number: 5678
      - path: /banana
        pathType: Prefix
        backend:
          service:
            name: banana-service
            port:
              number: 5678
```

### Deploy the Ingress

```bash
kubectl create -f ingress.yaml
```

---

## 4. Verification and Testing

Check the services in the `ingress-nginx` namespace:

```bash
kubectl get svc -n ingress-nginx
```

**Output:**
```text
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.102.93.210   <pending>     80:32435/TCP,443:32454/TCP   64m
```

Configure `/etc/hosts` to point `example.com` to your Ingress controller IP:

```bash
vi /etc/hosts
```

### Test Routing via Domain Name

```bash
curl -kL http://example.com/apple
```
*Output:* `apple`

```bash
curl -kL http://example.com/banana
```
*Output:* `banana`

### Test Routing via NodePort

```bash
curl -k example.com:32435/apple
```
*Output:* `apple`

```bash
curl -k example.com:32435/banana
```
*Output:* `banana`
