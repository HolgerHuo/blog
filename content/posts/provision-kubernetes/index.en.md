---
title: 'Provisioning a Highly-Available Production Kubernetes Cluster'
date: 2025-11-12T01:27:23+08:00
description: This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using kubeadmin, containerd, kube-vip, cilium, MetalLB, and OpenEBS.
summary: This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using kubeadmin, containerd, kube-vip, cilium, MetalLB, and OpenEBS.
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

This documentation will guide you through setting up a production-level High-Available multi-node Kubernetes cluster using `kubeadmin`, `containerd`, `kube-vip`, `cilium`, `MetalLB`, and `OpenEBS`. If you are using a managed Kubernetes service (K8s-as-a-Service) or already have an existing cluster, feel free to skip this section.

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

To prepare the host OS for Kubernetes installation, we need to disable swap, firewall, and SELinux, and enable IP forwarding.

```bash
# Disable swap
sudo swapoff -a 
sudo sed -i '/^[^#].*\s\+swap\s\+.*$/d' /etc/fstab # Also check systemd.swap
# Disable firewall
sudo systemctl disable --now firewalld
# Disable SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
# Enable forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
vm.nr_hugepages = 1024
EOF
sudo sysctl --system
sudo dnf install epel-release -y
sudo dnf install wget htop btop curl vim nano git jq -y
sudo dnf update -y
```

## Install `containerd`

`containerd` is a high-performance container runtime that is widely used in production Kubernetes clusters. We will install and configure `containerd` as the container runtime for our Kubernetes cluster.

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install containerd.io -y
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/    SystemdCgroup = false/    SystemdCgroup = true/' /etc/containerd/config.toml
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
sudo yum install -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes
sudo systemctl enable --now kubelet
```

Before proceeding with the Kubernetes setup, it is recommended to reboot the system to use the latest kernel version.

```bash
echo "exclude=containerd containerd.io kernel*" | sudo tee -a /etc/yum.conf
sudo reboot
```

## Setup Kubernetes

To setup Kubernetes cluster, we will use `kubeadm`, the official tool for bootstrapping Kubernetes clusters. First, we will setup `kube-vip` using static pod. This provides VirtualIP for control plane nodes, so that we can access the cluster even if one of the control plane nodes goes down. Then we will bootstrap the cluster and join other nodes. Finally we will setup Cilium CNI and MetalLB for networking and load balancing, OpenEBS for storage, and other usefull addons.

Starting now, we should use `root` user.

```bash
sudo -i
```

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

Now that we've setup the cluster, we can interact with the cluster from a local client. To proceed, you need to have `helm` installed on your local client.

```bash
dnf install -y helm
```

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
    --set ipam.mode=kubernetes \
    --set hubble.relay.enabled=true \
    --set hubble.peerService.clusterDomain=cluster.local

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

## OpenEBS for Storage

{{< alert >}}
OpenEBS is very heavy and generally eats up more then 4 GiB of memory on each control node. If you don't need such distributed storage solution, consider using nfs and host path provisioner.
{{< /alert >}}

To achieve HA persistent storage, OpenEBS requires at least 3 nodes, each will run etcd and operator to run OpenEBS.

### Prepare Storage Nodes

#### Login to Storage Nodes

Enable `nvme_tcp` kernel module.

```bash
sudo mkdir -p /etc/modules-load.d/
echo "nvme_tcp" | sudo tee /etc/modules-load.d/openebs.conf
sudo modprobe nvme_tcp
```

#### Label host as Storage Nodes

```bash
kubectl label node <node_name> openebs.io/engine=mayastor
```

### Install OpenEBS

Please update values according to your needs. 

```bash
helm repo add openebs https://openebs.github.io/openebs
helm install openebs \
    --namespace openebs openebs/openebs \
    --create-namespace \
    --set loki.enabled=false --set alloy.enabled=false --set minio.enabled=false
```

### Setup Storage Class

Now you should already have `openebs-hostpath` and `openebs-single-replica` storage classes available. But `openebs-single-replica` still doesn't have a storage pool yet.


#### Setup Mayastor Disk Pool

```bash
kubectl get sc # show predefined storage classes
ls -l /dev/disk/by-id/ # show available block devices
cat <<EOF | kubectl create -f -
apiVersion: "openebs.io/v1beta3"
kind: DiskPool
metadata:
  name: pool-on-node-1
  namespace: openebs
spec:
  node: workernode-1-hostname
  disks: ["aio:///dev/disk/by-id/<id>"]
EOF
```

#### Create Storage Class Using Mayastor

```bash
cat <<EOF | kubectl create -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mayastor-1
parameters:
  protocol: nvmf
  repl: "1"
provisioner: io.openebs.csi-mayastor
EOF
```

## csi-driver-nfs

If you have an existing NFS server, you can use it as a storage backend for your Kubernetes cluster by installing the `csi-driver-nfs` CSI driver.

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system \
    --version 4.12.0 \
    --set externalSnapshotter.enabled=true \
    --set controller.runOnControlPlane=true \
    --set controller.replicas=2
```

### Create Storage Class

```bash
cat <<EOF | kubectl create -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.default.svc.cluster.local
  share: /
  # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
  # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
  # csi.storage.k8s.io/provisioner-secret-namespace: "default"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1
EOF
```

With that, you have successfully set up a production-level High-Available multi-node Kubernetes cluster using `kubeadmin`, `containerd`, `cilium`, `MetalLB`, `OpenEBS`, and `kube-vip`. You can now deploy your applications and services on the cluster.

## Other Useful Add-ons

### cert-manager

`cert-manager` is a powerful tool for managing TLS certificates in Kubernetes. It automates the process of obtaining, renewing, and managing certificates from various sources such as Let's Encrypt, HashiCorp Vault, and more.

```bash
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

