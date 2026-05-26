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

### SELinux Deep Dive & Verification (`sestatus`)

Security-Enhanced Linux (SELinux) is a Linux kernel security module that provides access control security policies. In a Kubernetes cluster, SELinux must be set to **Permissive** or **Disabled** because:
* Kubernetes requires host filesystem access for container runtimes to manage pods, mount volumes, and handle networking.
* Default SELinux policies often block these container-to-host operations, leading to container start failures or permission denied errors.

#### Checking SELinux Status

Before modifying any configuration, check the current state of SELinux on your host using the `sestatus` command:

```bash
sestatus
```

**Common Output Fields:**
* **SELinux status:** Indicates if it is `enabled` or `disabled`.
* **Current mode:** The active mode of the system (e.g., `enforcing`, `permissive`, or `disabled`).
* **Mode from config file:** The mode that will be loaded upon the next system reboot.

#### Understanding SELinux Modes

| Mode | Behavior | Impact on Kubernetes |
| :--- | :--- | :--- |
| **Enforcing (`enforcing`)** | Security policy is strictly enforced. Unauthorized actions are blocked and logged. | **Blocks Kubernetes** from performing vital system tasks. |
| **Permissive (`permissive`)** | Security policy is not enforced. Actions are allowed, but violations are logged. | **Recommended for K8s installation/troubleshooting** without fully disabling security. |
| **Disabled (`disabled`)** | SELinux is completely turned off. No policies are loaded, and no violations are logged. | **Allowed by K8s**, but completely turns off host security features. |

> [!NOTE]
> Setting SELinux to `permissive` allows you to successfully initialize `kubeadm` and deploy containers while logging any potential issues to `/var/log/audit/audit.log` for security auditing.

#### Manual Persistent Configuration (`/etc/selinux/config`)

To persistently disable SELinux or set it to permissive, you can manually edit the configuration file `/etc/selinux/config` (which is symlinked to `/etc/sysconfig/selinux`) using a text editor like `vi`:

```bash
sudo vi /etc/selinux/config
```

##### 📄 File State - BEFORE Editing

```text
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
# See also:
# https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-selinux/#getting-started-with-selinux-selinux-states-and-modes
#
# NOTE: In earlier Fedora kernel builds, SELINUX=disabled would also
# fully disable SELinux during boot. If you need a system with SELinux
# fully disabled instead of SELinux running with no policy loaded, you
# need to pass selinux=0 to the kernel command line. You can use grubby
# to persistently set the bootloader to boot with selinux=0:
#
#    grubby --update-kernel ALL --args selinux=0
#
# To revert back to SELinux enabled:
#
#    grubby --update-kernel ALL --remove-args selinux
#
SELINUX=permissive
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

##### 📄 File State - AFTER Editing (Disabled)

```text
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
# See also:
# https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-selinux/#getting-started-with-selinux-selinux-states-and-modes
#
# NOTE: In earlier Fedora kernel builds, SELINUX=disabled would also
# fully disable SELinux during boot. If you need a system with SELinux
# fully disabled instead of SELinux running with no policy loaded, you
# need to pass selinux=0 to the kernel command line. You can use grubby
# to persistently set the bootloader to boot with selinux=0:
#
#    grubby --update-kernel ALL --args selinux=0
#
# To revert back to SELinux enabled:
#
#    grubby --update-kernel ALL --remove-args selinux
#
#SELINUX=permissive
SELINUX=disabled
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
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
