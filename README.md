# Homelab Infrastructure

Production-grade homelab demonstrating enterprise infrastructure patterns with Kubernetes, networking, and automation.

## 🏗️ Architecture

- **Server 1**: Proxmox hypervisor running production services
- **Server 2**: Bare metal Talos Linux Kubernetes cluster  
- **Networking**: VLAN-segmented network with PfSense routing
- **Management**: Arch Linux control plane for automation

## 🚀 Features

- Multi-cluster Kubernetes (k3s + Talos)
- Infrastructure as Code (Terraform)
- GitOps workflows (ArgoCD)
- Comprehensive monitoring (Prometheus/Grafana)
- Network segmentation and security
- Automated backup and disaster recovery

## 📊 Clusters

### K3s Cluster (Traditional)
- **Purpose**: Stable production services
- **Nodes**: 3x VMs on Proxmox
- **Services**: Monitoring, Git, CI/CD

### Talos Cluster (Bare Metal)
- **Purpose**: Experimental/cutting-edge workloads
- **Nodes**: 1x bare metal server
- **Services**: Advanced applications, testing

## 🛠️ Quick Start

```bash
# Clone repository
git clone https://github.com/eelbros/homelab-infrastructure.git
cd homelab-infrastructure

# Deploy infrastructure
cd terraform/proxmox
terraform init && terraform plan

# Deploy Kubernetes resources
kubectl apply -f kubernetes/k3s-cluster/monitoring/

## 🛠️ Current Status

### Completed:
- Proxmox hypervisor setup with VLAN configuration
- k3s cluster deployment (1 master + 2 worker nodes)
- SSH key authentication across all nodes
- kubectl access from Management laptop (Arch Linux)
- Network segmentation (Management: 192.168.2.0/24, Clients: 10.10.10.0/24, Lab: 10.10.20.0/24, IoT: 10.10.30.0/24)

### Infrastructure:
- **Proxmox Host**: pve (192.168.1.20)
- **k3s-master01**: 10.10.20.20 (control plane)
- **k3s-worker01**: 10.10.20.21
- **k3s-worker02**: 10.10.20.22
- **Template VM**: Clean Ubuntu 22.04 base

## Next Steps:
- [ ] Deploy monitoring stack (Prometheus/Grafana)
- [ ] Add Talos base metal cluster
- [ ] Implement GitOps with ArgoCD
