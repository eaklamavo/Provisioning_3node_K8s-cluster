#### START ####

## Intalling kubeadm and creating a cluster with kubeadm.

## Set hostnames to  master, worker1 & worker2 on nodes accordingly for easy identification.

sudo hostnamectl set-hostname master #on master server
sudo hostnamectl set-hostname worker1 #on worker1 server
sudo hostnamectl set-hostname worker2 #on worker2 server
 
## On all nodes, set up the hosts file to enable all the nodes to reach each other using these hostnames:

# Get private ips of each node and add this to host files on each server

sudo vi /etc/hosts

<master node server private_ip address> master
<worker1 node server private_ip address> worker1
<worker2 node server private_ip address> worker2

# Log out of all 3 servers for changes to take effect.

# Load some kernel modules and modify some system settings on all nodes.

cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay

sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# Install and configure containerd on all nodes.

sudo apt-get update && sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd

sudo systemctl status containerd #to check status after restart

# On all nodes, disable swap.

sudo swapoff -a

# Install kubeadm, kubelet, and kubectl on all nodes.

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00

sudo apt-mark hold kubelet kubeadm kubectl

# On the control plane node only, initialize the cluster and set up kubectl access.

sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.23.0

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify master node server/cluster Is working

kubectl get nodes

# Install on master node server, the Calico network add-on.

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Create a join token on master node & get the join command to paste on worker nodes.

kubeadm token create --print-join-command


# Copy the join command from the control plane node. Run it on each worker nodes.

sudo kubeadm join <generated token>

# Verify all nodes in your cluster are ready, may take a few moments for all of the nodes to enter the READY state.

kubectl get nodes

#### END ####