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
