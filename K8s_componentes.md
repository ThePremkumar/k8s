# Kubernetes Components

Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.

It groups containers that make up an application into logical units for easy management and discovery. Kubernetes builds upon 15 years of experience of running production workloads at Google, combined with best-of-breed ideas and practices from the community.

Kubernetes is hosted by the Cloud Native Computing Foundation (CNCF).

---

## 1. The Control Plane (Master)

The Control plane is made up of the kube-apiserver, kube-scheduler, cloud-controller-manager, and kube-controller-manager. Kube-proxy and kubelets live on each node, talking to the API and managing the workload of each node.

As the control plane handles most of Kubernetes’ ‘decision making’, nodes which have these components running generally don’t have any user containers running - these are normally named as master nodes.

### 1.1 kube-apiserver
The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane.

The main implementation of a Kubernetes API server is `kube-apiserver`. `kube-apiserver` is designed to scale horizontally—that is, it scales by deploying more instances. You can run several instances of `kube-apiserver` and balance traffic between those instances.

### 1.2 etcd
etcd is a distributed key-value store and the primary datastore of Kubernetes. It stores and replicates the Kubernetes cluster state.

Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.

> [!WARNING]
> If your Kubernetes cluster uses etcd as its backing store, make sure you have a backup plan for those data.
> You can find in-depth information about etcd in the official documentation.

### 1.3 kube-scheduler
Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.

### 1.4 kube-controller-manager
Control plane component that runs controller processes. Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.

The following controllers are present within this manager:
* **Node controller:** Responsible for noticing and responding when nodes go down.
* **Replication controller:** Responsible for maintaining replications of objects in the cluster (such as replicasets).
* **Endpoints controller:** Populates the Endpoints object (that is, joins Services & Pods).
* **Service Account & Token controllers:** Create default accounts and API access tokens for new namespaces.

### 1.5 cloud-controller-manager
A Kubernetes control plane component that embeds cloud-specific control logic. The cloud controller manager lets you link your cluster into your cloud provider's API, and separates out the components that interact with that cloud platform from components that only interact with your cluster.

The cloud-controller-manager only runs controllers that are specific to your cloud provider. If you are running Kubernetes on your own premises, or in a learning environment inside your own PC, the cluster does not have a cloud controller manager.

As with the `kube-controller-manager`, the `cloud-controller-manager` combines several logically independent control loops into a single binary that you run as a single process. You can scale horizontally (run more than one copy) to improve performance or to help tolerate failures.

The following controllers can have cloud provider dependencies:
* **Node controller:** For checking the cloud provider to determine if a node has been deleted in the cloud after it stops responding.
* **Route controller:** For setting up routes in the underlying cloud infrastructure.
* **Service controller:** For creating, updating, and deleting cloud provider load balancers.

---

## 2. Node Components

Node components run on every node, maintaining running pods and providing the Kubernetes runtime environment.

A Node represents a virtual or physical machine depending on your environment. Each node contains the components necessary to run pods. There are two distinct classifications of nodes: Master and Worker nodes.
* **Worker Nodes:** Worker nodes will have an instance of kubelet, kube-proxy, and a certain container runtime (such as Docker, containerd) running on each node, and are used to run user-defined containers. These nodes are managed by the control plane.
* **Master Nodes:** A master node (or control-plane node) will have the control plane binaries bootstrapped and is responsible for components within the control plane, such as running etcd and kube-apiserver. In order to achieve high availability and establish etcd quorum, there are normally 3 or more master nodes in a cluster.

### 2.1 kubelet
An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.

The kubelet functions as an agent within nodes and is responsible for running the pod lifecycle within each node. Its functionality is watching for new or changed pod specifications (PodSpecs) from master nodes and ensuring that pods within the node that it resides in are running, healthy, and match the pod specification. The kubelet doesn't manage containers which were not created by Kubernetes.

### 2.2 kube-proxy
kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept.

It maintains network rules on nodes, which allow network communication to your Pods from network sessions inside or outside of your cluster. It is also used to load balance traffic across services. Kube-proxy uses the operating system packet filtering layer if there is one and it's available. Otherwise, kube-proxy forwards the traffic itself.

### 2.3 Container Runtime
The container runtime is the software that is responsible for running containers.

Kubernetes supports container runtimes such as containerd, CRI-O, and any other implementation of the Kubernetes CRI (Container Runtime Interface).

---

## 3. Pods

Pods are the smallest, most basic deployable objects in Kubernetes. A Pod represents a single instance of a running process in your cluster. Pods contain one or more containers, such as Docker containers. When a Pod runs multiple containers, the containers are managed as a single entity and share the Pod's resources.

---

## 4. Addons

Addons use Kubernetes resources (DaemonSet, Deployment, etc.) to implement cluster features. Because these are providing cluster-level features, namespaced resources for addons belong within the `kube-system` namespace.

### 4.1 DNS
While the other addons are not strictly required, all Kubernetes clusters should have cluster DNS, as many examples rely on it.

Cluster DNS is a DNS server, in addition to the other DNS server(s) in your environment, which serves DNS records for Kubernetes services. Containers started by Kubernetes automatically include this DNS server in their DNS searches.

### 4.2 Web UI (Dashboard)
Dashboard is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage and troubleshoot applications running in the cluster, as well as the cluster itself.

### 4.3 Container Resource Monitoring
Container Resource Monitoring records generic time-series metrics about containers in a central database, and provides a UI for browsing that data.

### 4.4 Cluster-level Logging
A cluster-level logging mechanism is responsible for saving container logs to a central log store with a search/browsing interface.

---

## What's Next?
* Learn about Nodes
* Learn about Controllers
* Learn about kube-scheduler
* Read etcd's official documentation
