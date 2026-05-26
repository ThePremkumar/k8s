# Kubernetes Ingress Example

This document demonstrates how to configure and test a basic Kubernetes Ingress using a sample application and a custom local domain.

---

## 1. Deploying a Sample Application

To test your Ingress controller, you need a sample application. Here we will deploy an Nginx web server:

```bash
kubectl run myapp --image=nginx
```

---

## 2. Accessing the Application

In order to allow users to access your new sample application, you need a Service that exposes a port on your app. We will use a `NodePort` Service.

To create a `NodePort` Service for your application, run:

```bash
kubectl expose pod myapp --port=80 --name myapp-service --type=NodePort
```

To view the Service’s details:

```bash
kubectl get svc | grep myapp-service
```

### Understanding the Service Access
This tells you that your new demo application is routing port `80` (Nginx's default port) to your host machine's designated node port. To confirm that the application is accessible locally, you can use `curl` or visit `http://localhost:<NodePort>` (e.g. `http://localhost:32017`).

At this point, you have an application running and an Ingress controller running, but the two aren’t connected yet. This allows you to access the application locally, but in a production environment, you don’t want users accessing endpoints using the cluster’s node IP addresses. 

To provide secure access to external users, you’ll need to set up a custom domain and connect your Ingress controller to your Service.

---

## 3. Adding a Custom Domain

Giving your application a custom domain will make the address easier to remember and ensure that even if you redeploy your Kubernetes cluster to a different set of IP addresses, the application will still be available at the same domain.

To avoid purchasing and configuring a real domain for this demonstration, you can add your desired domain (`my-app.com` in this case) to your local `/etc/hosts` file. Run the following command using your `ClusterIP` and domain of choice:

```bash
echo '[ClusterIP] my-app.com' | sudo tee -a /etc/hosts
```

To get your service details:

```bash
kubectl get svc
```

**Output Example:**
```text
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        35h
myapp-service   NodePort    10.98.119.165   <none>        80:31831/TCP   3m35s
```

If you don’t remember your `ClusterIP` from above, you can retrieve it by running:

```bash
kubectl describe svc/myapp-service | grep IP: | awk '{print $2;}'
```

> [!NOTE]
> Be sure to replace `[ClusterIP]` with your actual Service Cluster IP (e.g., `10.98.119.165`).

---

## 4. Connecting the Ingress to your Service

Next, create/update your Ingress configuration file (`ingress.yaml`) with the host (`my-app.com`) and appropriate annotations:

```bash
vi ingress.yaml
```

### Ingress Manifest (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
       - path: /
         pathType: Prefix
         backend:
           service:
             name: myapp-service
             port:
               number: 80
```
