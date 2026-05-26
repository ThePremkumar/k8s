# Kubernetes Master Node Setup & Initialization

This reference document outlines the exact sequence of commands to prepare hosts (network configuration, SELinux, firewalls, and swap), configure repositories, install packages, and initialize a Kubernetes control plane using `kubeadm`.

---

## 1. Prepare Hostname, Firewall, and SELinux

Run these commands to set the node hostname and define cluster mapping:

```bash
# Set host name
hostnamectl set-hostname master-node

# Configure local hosts resolution mapping
cat <<EOF>> /etc/hosts
10.128.0.27 master-node
10.128.0.29 node-1 worker-node-1
10.128.0.30 node-2 worker-node-2
EOF
```

---

## 2. Disable Security Enforcements & Firewalls

Disable SELinux policies and stop the system firewall:

```bash
# Disable SElinux and update your firewall rules.
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
reboot
```

```bash
# Disable Firewalld
systemctl stop firewalld
systemctl disable firewalld
```

---

## 3. Install and Configure Docker Engine

Install, start, and configure permissions for the container runtime:

```bash
# Install the most recent Docker Engine package.
sudo yum update -y
sudo amazon-linux-extras install docker -y
systemctl enable docker
systemctl daemon-reload && systemctl restart docker 
sudo usermod -a -G docker ec2-user
docker info
```

---

## 4. Kernel Modules & Swap Configuration

Configure networking modules required for bridge filtering and disable swap space:

```bash
# Disable Swap Space
swapoff -a

# Load bridge filtering kernel modules
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
modprobe br_netfilter
modprobe nf_nat
modprobe xt_REDIRECT
modprobe xt_owner
modprobe iptable_nat
modprobe iptable_mangle
modprobe iptable_filter
```

---

## 5. Setup the Kubernetes Repository (CentOS/RHEL)

Manually configure the repository source file:

```bash
# Setup the Kubernetes Repo
# You will need to add Kubernetes repositories manually as they do not come installed by default on CentOS 7.

cat >> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF
```

---

## 6. Install Kubernetes Packages

Install precise versions of `kubelet`, `kubectl`, and `kubeadm` packages, then enable services:

```bash
# Install packages
yum install -y kubelet-1.21.2
yum install -y kubectl-1.21.2
yum install -y kubeadm-1.21.2 

# Reload and enable services
systemctl daemon-reload 
systemctl restart kubelet
systemctl status kubelet
systemctl enable kubelet
```

---

## 7. Initialize Kubernetes Master Node

Ensure swap is turned off, and run the `kubeadm init` bootstrapping command:

```bash
# Disable swap (mandatory for kubeadm init)
swapoff -a

# Initialize Kubernetes master
kubeadm init --control-plane-endpoint "192.168.0.0:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
```

Configure non-root user credentials:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 8. Deploy Pod Network (Flannel)

Apply the Flannel CNI network overlay manifest:

```bash
# Deploy Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
