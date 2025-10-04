# Kubernetes Cluster Setup on Ubuntu 24

This guide explains how to set up a **Kubernetes cluster (v1.34)** with **1 master** and **2 worker nodes** on Ubuntu 24.

---

## Prerequisites

- All nodes are fresh Ubuntu 24 with internet access
- Nodes can communicate via private IP (VPC/subnet if cloud)
- Disable swap and configure firewall/security groups for intra-node communication

---

## Step 1: Prepare All Nodes (Master & Workers)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

