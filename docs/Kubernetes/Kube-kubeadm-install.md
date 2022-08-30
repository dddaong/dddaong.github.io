---
title:  "K8s Installation script for Centos7"
author: DDAONG
date:   2022-08-03
category: "Kubernetes"
layout: post
---

## Kubernetes Installation via Kubeadm - bash script for Centos7

#### CentOS 보안 설정

```bash
sed -i -re 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux;
sed -i -re 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config;
setenforce 0;
#sestatus

systemctl disable firewalld;
systemctl stop firewalld;
#systemctl status firewalld

swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab;
#swapon

modprobe overlay
modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

tee -a /etc/sysctl.d/k8s.conf <<EOF 1>/dev/null
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system;
```

#### CRI Containerd Install

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum -y install containerd.io

### containerd config
sudo mkdir -p /etc/containerd
cat > /etc/containerd/config.toml <<EOF
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2
EOF

systemctl daemon-reload
systemctl enable containerd --now
```

#### Kubernetes Installation

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

kubeadm config images pull

kubeadm init --pod-network-cidr 10.224.0.0/16
```

#### Kubernetes Post-Installation 작업

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

yum -y install bash-completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### Kubernetes CNI Calico Installation

```bash
# Calico
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O

curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -o calico.yaml
kubectl apply -f calico.yaml
```
