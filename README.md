# Run a Kubernetes cluster on macOS (arm64)

1. Create virtual machines (nodes)
   - Create one node for the control plane (for example, `centos`)
     - On the control plane node, initialize the cluster
   - Create one or more worker nodes (for example, `debian`,`fedora`,`ubuntu`)
     - On each worker node, join the cluster
   - CentOS Stream
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/centos)
     - [Install Kubernetes](kubeadm-centos.md)
   - Debian
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/debian)
     - [Install Kubernetes](kubeadm-debian.md)
   - Fedora
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/fedora)
     - [Install Kubernetes](kubeadm-fedora.md)
   - Ubuntu
     - [Virtual machine](https://github.com/tyholling/packer/tree/main/ubuntu)
     - [Install Kubernetes](kubeadm-ubuntu.md)
   - The nodes will have IPs in 192.168.64.0/24, see `/var/db/dhcpd_leases`

1. On macOS:
   - `brew install kubectl helm`
   - Copy `/etc/kubernetes/admin.conf` from the control plane node to `~/.kube/config`

1. Install [flannel](https://github.com/flannel-io/flannel)
   ```
   kubectl create ns flannel
   helm repo add flannel https://flannel-io.github.io/flannel/
   helm install flannel -n flannel flannel/flannel
   ```
1. Install [metallb](https://github.com/metallb/metallb)
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
   ```
   kubectl create ns nginx
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm install nginx -n nginx ingress-nginx/ingress-nginx
   ```
1. Test the nginx 404 page:
   ```
   curl 192.168.64.100
   ```
