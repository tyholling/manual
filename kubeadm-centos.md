# Installing Kubernetes on CentOS Stream 9

1. Disable swap
   ```
   swapoff -a
   sed -i '/swap/s/^/# /g' /etc/fstab
   ```
1. Disable SELinux
   ```
   setenforce 0
   sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   ```
1. Disable firewall
   ```
   systemctl disable --now firewalld
   ```
1. Load kernel modules
   ```
   cat << eof > /etc/modules-load.d/kubernetes.conf
   br_netfilter
   overlay
   eof
   systemctl restart systemd-modules-load
   ```
1. Configure kernel parameters
   ```
   cat << eof > /etc/sysctl.d/kubernetes.conf
   net.ipv4.ip_forward = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   eof
   sysctl --system
   ```
1. Install CRI-O
   ```
   export CRIO_VERSION=v1.32
   cat << eof > /etc/yum.repos.d/cri-o.repo
   [cri-o]
   name=CRI-O
   baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
   eof

   dnf install -y cri-o
   systemctl enable --now crio
   ```
1. Install Kubernetes
   ```
   export KUBERNETES_VERSION=v1.32
   cat << eof > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
   eof

   dnf install -y kubeadm kubelet kubectl kubernetes-cni
   systemctl enable kubelet
   ```
---
1. Initialize the cluster
   ```
   kubeadm init --node-name centos --pod-network-cidr=10.244.0.0/16
   mkdir ~/.kube
   cp -i /etc/kubernetes/admin.conf ~/.kube/config
   ```
1. Install Flannel
   ```
   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```
1. Configure DNS
   ```
   systemctl enable --now systemd-resolved
   ```
---
1. Join the cluster
   ```
   kubeadm join ...
   ```
1. Configure DNS
   ```
   systemctl enable --now systemd-resolved
   ```
