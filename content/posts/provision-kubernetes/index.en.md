---
title: 'Provisioning a Highly-Available Production Kubernetes Cluster'
date: 2025-11-12T01:27:23+08:00
description: This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using kubeadmin, containerd, kube-vip, cilium, and MetalLB.
summary: This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using kubeadmin, containerd, kube-vip, cilium, and MetalLB.
author: holger
categories:
  - Programming
tags:
  - k8s
  - cloud
  - linux
  - devops
  - kubernetes
keywords:
  - k8s
  - cloud
  - linux
  - devops
  - kubernetes
feature: kubernetes-cluster-architecture.svg
featureAlt: 
images:
  - kubernetes-cluster-architecture.svg
slug: provision-kubernetes
type: posts
showComments: true
isCJKLanguage: false
draft: false
showTableOfContents: true
---

This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using `kubeadmin`, `containerd`, `kube-vip`, `cilium`, and `MetalLB`. If you are using a managed Kubernetes service (K8s-as-a-Service) or already have an existing cluster, feel free to skip this section.

## Prerequisites

- A minimum of 3 nodes are required for HA control plane
- At least 2 CPU Cores and 2 GB of RAM per node
- Full network connectivity between all machines in the cluster (public or private network is fine)
- Unique hostname, MAC address, and product_uuid for every node
- RHEL 10 (All recent distros should work but this tutorial is tailored for RHEL)

{{< alert "circle-info">}}
Using MAAS, cloud-init, ansible or similar bare-metal provision tools is strongly recommended to automate and accelerate setup process.
{{< /alert >}}

## Setting Up Host OS

To prepare the host OS for Kubernetes installation, we need to disable swap, firewall, and SELinux, enable IP forwarding, and load the overlay kernel module.

```bash
# Disable swap
sudo swapoff -a # Also check /etc/fstab and systemd.swap
# Disable firewall
sudo systemctl disable --now firewalld
# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
# Enable forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system
# Load overlay module
sudo mkdir -p /etc/modules-load.d/
echo overlay | sudo tee /etc/modules-load.d/k8s.conf 
sudo modprobe overlay
sudo dnf install epel-release -y
sudo dnf update -y
```

## Install `containerd`

`containerd` is a high-performance container runtime that is widely used in production Kubernetes clusters. We will install and configure `containerd` as the container runtime for our Kubernetes cluster.

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install containerd.io -y
echo "exclude=containerd containerd.io kernel*" | sudo tee -a /etc/yum.conf
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/^[[:space:]]*SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
EOF
```
## Install Kubernetes

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
sudo yum install -y kubelet kubeadm kubectl helm --setopt=disable_excludes=kubernetes
sudo systemctl enable --now kubelet
```

Before proceeding with the Kubernetes setup, it is recommended to reboot the system to use the latest kernel version.

```bash
sudo reboot
```

## Setup Kubernetes

To setup Kubernetes cluster, we will use `kubeadm`, the official tool for bootstrapping Kubernetes clusters. First, we will setup `kube-vip` using static pod. This provides VirtualIP for control plane nodes, so that we can access the cluster even if one of the control plane nodes goes down. Then we will bootstrap the cluster and join other nodes. Finally we will setup Cilium CNI and MetalLB for networking and load balancing.

### kube-vip

On all control plane nodes, we will setup `kube-vip` as a static pod. Replace the `VIP` value with your desired Virtual IP.

```bash
export VIP=<vip>
export INTERFACE=<inter-node-network-interface>
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml

# Only on bootstrap node, see https://github.com/kube-vip/kube-vip/issues/684
sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml && systemctl restart kubelet
```

This sets up `kube-vip` as a static pod using ARP mode so that `kubelet` will manage its lifecycle. 

### Bootstrap Cluster

```bash
kubeadm config images pull \
    --cri-socket unix:///var/run/containerd/containerd.sock
kubeadm init \
    --control-plane-endpoint vip.k8s.example.com:6443 \
    --cri-socket unix:///var/run/containerd/containerd.sock \
    --upload-certs \
    --service-cidr 172.18.0.0/16 \
    --pod-network-cidr 172.19.0.0/16 \
    --service-dns-domain cluster.local \
    --skip-phases=addon/kube-proxy
```

Now, kubeadm has initialized the cluster by installing control plane components through static pods, generating certificates for component/pod communications, and setting up `kubeconfig` for cluster access. 

`kube-vip` patch can now be reverted

```bash
sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml && systemctl restart kubelet
```

You can copy the `kubeconfig` file to your home directory for easier access.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join Other Nodes

For additional Control Plane nodes, repeat the Host OS setup and the kube-vip setup (without the initial patch). For Worker nodes, only the Host OS setup is required.

Then, use the kubeadm join command provided in the output of kubeadm init to add the remaining control plane and worker nodes.

### Untaint Control Plane Nodes (Optional)

By default, control plane nodes are tainted and won't run user workloads. To allow workloads to run on them, remove the taint:

```bash
# kubectl taint nodes <key>- : remove taint
# kubectl taint nodes <key>=<value> : add taint

kubectl taint nodes --all node-role.kubernetes.io/control-plane-
# To re-taint: kubectl taint nodes <node-name> node-role.kubernetes.io/control-plane:NoSchedule
```

## Networking and Load Balancing

### Setting Up Cilium CNI

We will use Cilium as the CNI plugin for the cluster. Cilium is a powerful and flexible CNI plugin that provides advanced networking features such as network policies, load balancing, and service mesh capabilities.


```bash
helm repo add cilium https://helm.cilium.io/
API_SERVER_IP=<vip> # replace with your api server vip or FQDN
API_SERVER_PORT=6443
helm install cilium cilium/cilium --version 1.18.3 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=${API_SERVER_PORT} \
    --set ipam.mode=kubernetes

# Verify installation
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement
```

### Install MetalLB

We will use MetalLB as the load balancer for the cluster. MetalLB is a popular load balancer for bare-metal Kubernetes clusters that provides network load balancing using standard protocols such as BGP and ARP.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```
Next, we need to configure MetalLB with a pool of IP addresses that it can use for load balancing. Replace the `address-pool` values with your desired IP range.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: public
spec:
    addresses:
      - "192.168.20.0/24"
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: public
  namespace: metallb-system
spec:
  ipAddressPools:
    - public
EOF
```

With that, you have successfully set up a production-level High-Available multi-node Kubernetes cluster using `kubeadmin`, `containerd`, `cilium`, `MetalLB`, and `kube-vip`. You can now deploy your applications and services on the cluster.
