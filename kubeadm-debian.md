# Installing Kubernetes on Debian 12

1. Disable swap
   ```
   swapoff -a
   sed -i '/swap/s/^/# /g' /etc/fstab
   ```
1. Configure DNS
   ```
   apt-get remove -y systemd-resolved
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
1. Initialize the cluster
   ```
   kubeadm init --node-name debian --pod-network-cidr=10.244.0.0/16
   mkdir ~/.kube
   cp -i /etc/kubernetes/admin.conf ~/.kube/config
   ```
1. Install Flannel
   ```
   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```
