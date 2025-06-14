# Run a Kubernetes cluster on macOS (arm64)

1. Create virtual machines (nodes)
   - Create one or more nodes for the control plane (e.g. `centos`, `debian`)
   - Create one or more worker nodes (e.g. `fedora`, `ubuntu`)
   - CentOS Stream
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/centos)
     - [Install Kubernetes](https://github.com/tyholling/packer/blob/main/centos/kubelet.sh)
   - Debian
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/debian)
     - [Install Kubernetes](https://github.com/tyholling/packer/blob/main/debian/kubelet.sh)
   - Fedora
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/fedora)
     - [Install Kubernetes](https://github.com/tyholling/packer/blob/main/fedora/kubelet.sh)
   - Ubuntu
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/ubuntu)
     - [Install Kubernetes](https://github.com/tyholling/packer/blob/main/ubuntu/kubelet.sh)
   - The nodes will have IPs in `192.168.64.0/24`, see `/var/db/dhcpd_leases`

1. Initialize the cluster
   * Option 1: Single-node control plane
     ```
     ssh centos
     kubeadm init --pod-network-cidr=10.244.0.0/16
     ```
   * Option 2: Multi-node control plane (high availability)
     - The provided pod manifest uses `192.168.64.64` for the cluster virtual IP

     Initialize the cluster:
     ```
     ssh centos
     wget -P /etc/kubernetes/manifests \
     https://raw.githubusercontent.com/tyholling/deploy/refs/heads/main/kube-vip.yaml
     kubeadm init --control-plane-endpoint 192.168.64.64 \
     --pod-network-cidr=10.244.0.0/16 --upload-certs
     ```
     Add nodes to the control plane:
     ```
     ssh debian
     kubeadm join --control-plane ...
     ```
     Update and deploy `kube-vip` to the control plane:
     ```
     ssh centos sed -i s/super-admin/admin/g /etc/kubernetes/manifests/kube-vip.yaml
     scp centos:/etc/kubernetes/manifests/kube-vip.yaml debian:/etc/kubernetes/manifests/
     ```
1. Add worker nodes
   ```
   ssh fedora
   kubeadm join ...
   ```
   - Note: for Fedora and Ubuntu, after `kubeadm init` or `kubeadm join`:
     ```
     systemctl enable --now systemd-resolved
     ```
1. On macOS:
   - `brew install kubectl helm`
   - Copy `/etc/kubernetes/admin.conf` from a control plane node:
     ```
     mkdir ~/.kube
     scp centos:/etc/kubernetes/admin.conf ~/.kube/config
     ```
1. Install [flannel](https://github.com/flannel-io/flannel)
   - This provides the overlay network to allow pods to communicate
   ```
   kubectl create ns flannel
   helm repo add flannel https://flannel-io.github.io/flannel
   helm install flannel -n flannel flannel/flannel
   ```
1. Install [metallb](https://github.com/metallb/metallb)
   - This provides the load balancer to allow connections to the cluster
   ```
   kubectl create ns metallb
   helm repo add metallb https://metallb.github.io/metallb
   helm install metallb -n metallb metallb/metallb
   ```
   - Configure the IP address pool
     - https://metallb.universe.tf/configuration/#defining-the-ips-to-assign-to-the-load-balancer-services
     - This is the pool of available addresses to assign to load balancer services
     - In this example, one IP is selected: `192.168.64.100`
     - This IP cannot be the same as any of the virtual machines
     ```
     cat << eof > ip-address-pool.yaml
     apiVersion: metallb.io/v1beta1
     kind: IPAddressPool
     metadata:
       name: ip-address-pool
       namespace: metallb
     spec:
       addresses:
       - 192.168.64.100/32
       autoAssign: true
       avoidBuggyIPs: true
     eof
     kubectl apply -f ip-address-pool.yaml
     ```
   - Configure the Layer 2 advertisement
     - https://metallb.universe.tf/configuration/#layer-2-configuration
     ```
     cat << eof > l2-advertisement.yaml
     apiVersion: metallb.io/v1beta1
     kind: L2Advertisement
     metadata:
       name: l2-advertisement
       namespace: metallb
     eof
     kubectl apply -f l2-advertisement.yaml
     ```
1. Install [ingress-nginx](https://github.com/kubernetes/ingress-nginx)
   - This provides the ingress to route external traffic to services
   ```
   kubectl create ns ingress
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm install ingress -n ingress ingress-nginx/ingress-nginx
   ```
1. Test the nginx 404 page:
   ```
   curl 192.168.64.100
   ```
1. Install [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
   - This provides memory and cpu monitoring to support pod autoscaling
   ```
   kubectl create ns metrics
   helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
   helm install metrics -n metrics metrics-server/metrics-server --set args={--kubelet-insecure-tls}
   ```
