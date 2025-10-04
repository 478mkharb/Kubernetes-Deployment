#  Kubernetes Cluster Setup on Ubuntu 24 (v1.34) with Full CI/CD Ports
This guide sets up a **Kubernetes v1.34 cluster** with **1 master node** and **2 worker nodes** on Ubuntu 24.  
It includes node preparation, container runtime setup, Kubernetes installation, master initialization, joining workers, verification, pod network, test deployment, and full CI/CD firewall rules.

## 1. Prepare All Nodes
```bash
sudo apt update && sudo apt upgrade -y
```
#### Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
#### Load kernel modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
#### Apply sysctl params
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
## Step 2: Verify Node Communication
```bash
echo "Testing connectivity to worker nodes:"
ping -c 3 <WORKER_IP_1>
ping -c 3 <WORKER_IP_2>
```
#### Test port connectivity to Kubernetes API port 6443 on master
```bash
nc -zv <MASTER_IP> 6443
```
## Step 3: Install Container Runtime (containerd)
```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```
## Step 4: Install Kubernetes Components
```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
## Step 5: Update the system
```bash
sudo apt update
```
## Step 6: Install Kubernetes Components (contd.)
```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## Step 7: Initialize Master Node (Master Only)
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
## Step 8: Configure kubectl on Master (run only on master)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Step 9: Install Pod Network (Flannel) (To be run only on master)
#### - Kubernetes pods need to communicate with each other across nodes.
#### - Each pod gets its own IP address, and Kubernetes expects all pods to be able to reach each other.
#### - Flannel is a simple overlay network that allows pods on different nodes to communicate.
#### - Other alternatives: Calico, Weave Net, Cilium — they all provide pod networking, but some add network policies, security, or advanced features.
### Option 1: Flannel (simple, lightweight)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
### Option 2: Calico (advanced, supports network policies)
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
#### *Note: Pick either Flannel or Calico, not both*

## Step 10: Deploy Ingress Controller (NGINX) [Optional for external HTTP/HTTPS access]
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```
### This is required only if you want to expose multiple services externally via Ingress

## Step 11: Join Worker Nodes (run on each worker)
### Paste the kubeadm join command from master here (remove #):
```bash
sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Step 12: Verify Cluster (run on master)
```bash
kubectl get nodes
kubectl get pods -A
```
### All nodes should be Ready, and kube-system pods Running

## Step 13: Verify Cluster (run on master)
```bash
kubectl get nodes
kubectl get pods -A
```
### All nodes should be Ready, and kube-system pods Running

## Step 14: Deploy a Test Application (Optional)
```bash
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc
```
##### Access the app:
##### - NodePort: http://<worker-node-IP>:<NodePort>  (ensure security group allows this)
##### - Or via Ingress Controller if deployed

## Security Group / Firewall Rules (Inbound & Outbound)
### Inbound (traffic coming to nodes)
#### Master Node:
##### TCP 6443  : Kubernetes API server
##### TCP 2379-2380 : etcd server client API
##### TCP 10250 : Kubelet API
##### TCP 10251 : kube-scheduler
##### TCP 10252 : kube-controller-manager
##### TCP 22    : SSH (admins / Jenkins agents)
##### TCP 80,443 : HTTP/HTTPS (apps)
##### TCP 25,465 : SMTP/SMTPS for CI/CD notifications
##### TCP 30000-32767 : NodePort services
##### UDP 8472 : Flannel VXLAN pod network

#### Worker Nodes:
##### TCP 10250 : Kubelet API
##### TCP 22    : SSH
##### TCP 80,443 : HTTP/HTTPS apps
##### TCP 30000-32767 : NodePort services
##### UDP 8472 : Flannel VXLAN
##### ICMP     : Ping/health check

#### Outbound (traffic going out from nodes)
#### All Nodes:
##### TCP 53      : DNS resolution
##### TCP/UDP 80,443 : External APIs, container registry, CI/CD tools
##### TCP 6443    : Master API server
##### TCP 5000    : Docker Registry
##### TCP 8080,50000 : Jenkins web interface and agent communication
##### ICMP       : Ping/health check to other nodes

#### *Notes:*
##### - *Nodes must allow internal communication on Kubernetes ports (API, Kubelet, NodePort, pod network)*
##### - *Open external ports only as needed for apps and CI/CD tools*
##### - *Always run kubeadm init / join as root (sudo)*
##### - *Pod network must be installed before joining workers*
