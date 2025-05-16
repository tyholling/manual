# Installing Kubernetes on Ubuntu 25.04

1. Disable swap
   ```
   swapoff -a
   sed -i '/swap/s/^[^#]/# /g' /etc/fstab
   ```
1. Configure DNS
   ```
   systemctl disable --now systemd-resolved
   ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
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
   apt-get install -y software-properties-common curl

   export CRIO_VERSION=v1.32
   curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key \
   | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
   cat << eof > /etc/apt/sources.list.d/cri-o.list
   deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] \
   https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /
   eof

   apt-get update
   apt-get install -y cri-o
   systemctl enable --now crio
   ```
1. Install Kubernetes
   ```
   export KUBERNETES_VERSION=v1.32
   curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key \
   | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   cat << eof > /etc/apt/sources.list.d/kubernetes.list
   deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
   https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /
   eof

   apt-get update
   apt-get install -y kubeadm kubelet kubectl kubernetes-cni
   systemctl enable kubelet
   ```
---
1. Initialize the cluster
   ```
   cat << eof > kubeadm-config.yaml
   apiVersion: kubeadm.k8s.io/v1beta4
   kind: ClusterConfiguration
   networking:
     podSubnet: 10.244.0.0/16
   ---
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   serverTLSBootstrap: true
   eof

   kubeadm init --node-name ubuntu --config kubeadm-config.yaml
   mkdir ~/.kube
   cp -i /etc/kubernetes/admin.conf ~/.kube/config
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
