```bash
# Kubernetes Cluster Setup on Ubuntu 24 - Full Continuous Script

# ----------------------
# Step 1: Prepare All Nodes
# ----------------------
sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# ----------------------
# Step 2: Install Container Runtime (containerd)
# ----------------------
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# ----------------------
# Step 3: Install Kubernetes Components
# ----------------------
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# ----------------------
# Step 4: Initialize Master Node (run only on master)
# ----------------------
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# Copy the kubeadm join command displayed at the end for worker nodes

# ----------------------
# Step 5: Configure kubectl on Master (run only on master)
# ----------------------
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# ----------------------
# Step 6: Install Pod Network (Flannel) (run only on master)
# ----------------------
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# ----------------------
# Step 7: Join Worker Nodes (run on each worker)
# ----------------------
# Paste the kubeadm join command from master here:
# sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
# If the token expired, regenerate on master:
# kubeadm token create --print-join-command

# ----------------------
# Step 8: Verify Cluster (run on master)
# ----------------------
kubectl get nodes
kubectl get pods -A
# All nodes should be Ready, and kube-system pods Running

# ----------------------
# Step 9: Deploy a Test Application (Optional)
# ----------------------
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc
# Access the app via browser: http://<worker-node-IP>:<NodePort>

# ----------------------
# Notes
# ----------------------
# Always run kubeadm init and kubeadm join as root (sudo)
# Ensure security groups/firewall allow traffic between nodes (VPC CIDR)
# Keep all nodes at compatible Kubernetes versions (same minor version as master)
# Pod network must be installed before joining workers
