# Kubernetes Cluster Setup on Ubuntu 24

This guide explains how to set up a **Kubernetes cluster (v1.34)** with **1 master** and **2 worker nodes** on Ubuntu 24.

```bash
# -------------------------------
# Step 1: Prepare All Nodes
# -------------------------------
sudo apt update && sudo apt upgrade -y
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# -------------------------------
# Step 2: Install containerd
# -------------------------------
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# -------------------------------
# Step 3: Install Kubernetes Components
# -------------------------------
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# -------------------------------
# Step 4: Initialize Master Node
# -------------------------------
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# -------------------------------
# Step 5: Configure kubectl on Master
# -------------------------------
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# -------------------------------
# Step 6: Install Pod Network (Flannel)
# -------------------------------
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# -------------------------------
# Step 7: Join Worker Nodes
# -------------------------------
# On each worker node, run the following command (replace <MASTER_IP>, <token>, <hash>):
# sudo kubeadm join <MASTER_IP>:6443 --token <token> \
#     --discovery-token-ca-cert-hash sha256:<hash>

# If token expired, regenerate on master:
# kubeadm token create --print-join-command

# -------------------------------
# Step 8: Verify Cluster
# -------------------------------
kubectl get nodes
kubectl get pods -A

