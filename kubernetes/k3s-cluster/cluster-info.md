# k3s Cluster Documentation

## Cluster Overview

A production-ready 3-node Kubernetes cluster running k3s on Ubuntu 22.04 VMs, designed for learning SRE practices and running containerized applications.

## Architecture

### Cluster Topology
```
k3s Cluster (10.10.20.0/24)
├── k3s-master01 (10.10.20.20)
│   ├── Control Plane Components
│   ├── etcd (embedded)
│   ├── API Server
│   ├── Scheduler
│   ├── Controller Manager
│   └── Can run workloads
│
├── k3s-worker01 (10.10.20.21)  
│   ├── kubelet
│   ├── kube-proxy
│   ├── containerd
│   └── Workload execution
│
└── k3s-worker02 (10.10.20.22)
    ├── kubelet
    ├── kube-proxy  
    ├── containerd
    └── Workload execution
```

### Node Specifications

| Node | VM ID | IP Address | Role | CPU | Memory | Storage |
|------|-------|------------|------|-----|--------|---------|
| k3s-master01 | 105 | 10.10.20.20 | Control Plane | 2 cores | 4GB | 32GB |
| k3s-worker01 | 103 | 10.10.20.21 | Worker | 2 cores | 4GB | 32GB |
| k3s-worker02 | 104 | 10.10.20.22 | Worker | 2 cores | 4GB | 32GB |
| k3s-template | 102 | - | Template | 2 cores | 4GB | 32GB |

## Installation Details

### Master Node Installation
```bash
# Installed on k3s-master01 (10.10.20.20)
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-cidr=10.42.0.0/16 \
  --service-cidr=10.43.0.0/16 \
  --cluster-dns=10.43.0.10
```

### Worker Node Installation
```bash
# Installed on both worker nodes
curl -sfL https://get.k3s.io | K3S_URL=https://10.10.20.20:6443 \
  K3S_TOKEN=<node-token> sh -
```

### Configuration Parameters

#### Network Configuration
- **Cluster CIDR**: `10.42.0.0/16` - Pod networking range
- **Service CIDR**: `10.43.0.0/16` - Service IP range  
- **Cluster DNS**: `10.43.0.10` - CoreDNS service IP
- **CNI**: Flannel (default k3s CNI)

#### Built-in Components
- **Container Runtime**: containerd
- **Ingress Controller**: Traefik
- **Load Balancer**: ServiceLB (basic)
- **DNS**: CoreDNS
- **Metrics**: metrics-server

## Cluster Status

### Node Information
```bash
# Check cluster nodes
kubectl get nodes -o wide

# Expected output:
NAME           STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   
k3s-master01   Ready    control-plane,master   1d    v1.28.x+k3s1   10.10.20.20   
k3s-worker01   Ready    <none>                 1d    v1.28.x+k3s1   10.10.20.21   
k3s-worker02   Ready    <none>                 1d    v1.28.x+k3s1   10.10.20.22   
```

### System Pods
```bash
# Check system components
kubectl get pods -A

# Core system pods:
NAMESPACE     NAME                             READY   STATUS
kube-system   coredns-6799fbcd5-xxxxx          1/1     Running
kube-system   local-path-provisioner-xxxxx     1/1     Running  
kube-system   metrics-server-xxxxx             1/1     Running
kube-system   traefik-xxxxx                    1/1     Running
```

## Networking

### Pod Network (CNI)
- **Plugin**: Flannel (VXLAN backend)
- **CIDR**: 10.42.0.0/16
- **Per-node subnets**: /24 blocks automatically assigned

### Service Network
- **CIDR**: 10.43.0.0/16  
- **kubernetes service**: 10.43.0.1
- **CoreDNS service**: 10.43.0.10

### Ingress and Load Balancing
- **Ingress Controller**: Traefik (built-in)
- **Load Balancer**: ServiceLB (metallb alternative planned)
- **External Access**: NodePort services on all nodes

## Storage

### Default Storage Class
- **Provisioner**: rancher.io/local-path
- **Type**: Local host path storage
- **Mount Path**: `/opt/local-path-provisioner` on each node

### Storage Configuration
```bash
# Check storage classes
kubectl get storageclass

# Default storage class: local-path (RWO)
# Suitable for: databases, stateful applications
# Limitations: Node-local, no replication
```

## Security

### RBAC Configuration
- **Default**: k3s comes with RBAC enabled
- **Admin Access**: Available via kubeconfig
- **Service Accounts**: Automatically created per namespace

### Network Policies
- **Current**: No network policies implemented
- **Future**: Implement Calico/Cilium for micro-segmentation

### Certificate Management
- **CA Certificates**: Auto-generated during installation
- **Location**: `/var/lib/rancher/k3s/server/tls/`
- **Rotation**: Manual process (planned automation)

## Access and Management

### kubectl Configuration
```bash
# From management laptop (Arch Linux)
export KUBECONFIG=~/.kube/k3s-config

# Alternative: Copy to default location
cp ~/.kube/k3s-config ~/.kube/config
```

### SSH Access to Nodes
```bash
# Direct SSH access configured
ssh k3s-master01  # 10.10.20.20
ssh k3s-worker01  # 10.10.20.21  
ssh k3s-worker02  # 10.10.20.22
```

### Service Management
```bash
# On master node
sudo systemctl status k3s
sudo systemctl restart k3s

# On worker nodes  
sudo systemctl status k3s-agent
sudo systemctl restart k3s-agent
```

## Planned Deployments

### Monitoring Stack
- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards
- **AlertManager**: Alert routing and management
- **Node Exporter**: Host metrics

### GitOps Workflow
- **ArgoCD**: Continuous deployment
- **Git Repository**: Source of truth for configurations
- **Automated Sync**: Git-to-cluster synchronization

### Additional Services
- **Container Registry**: Harbor or private registry
- **Secrets Management**: Sealed Secrets or External Secrets Operator
- **Backup Solution**: Velero for cluster backups

## Operations and Maintenance

### Regular Maintenance Tasks
```bash
# Update k3s (on master first, then workers)
curl -sfL https://get.k3s.io | sh -

# Check cluster health
kubectl get nodes
kubectl get pods -A
kubectl top nodes  # Requires metrics-server

# Clean up unused resources
kubectl delete pods --field-selector=status.phase=Succeeded -A
```

### Backup Procedures
```bash
# Backup cluster state (on master)
sudo cp -r /var/lib/rancher/k3s/server/db /backup/k3s-db-$(date +%Y%m%d)

# Export all resources
kubectl get all -A -o yaml > cluster-backup-$(date +%Y%m%d).yaml
```

### Log Management
```bash
# k3s logs (systemd)
sudo journalctl -u k3s -f        # Master
sudo journalctl -u k3s-agent -f  # Workers

# Pod logs
kubectl logs -n kube-system <pod-name>
kubectl logs -f deployment/traefik -n kube-system
```

## Troubleshooting

### Common Issues and Solutions

#### Node Not Ready
```bash
# Check node status
kubectl describe node <node-name>

# Check k3s service
ssh <node> "sudo systemctl status k3s-agent"

# Check network connectivity
kubectl get nodes -o wide
ping <node-ip>
```

#### Pod Networking Issues
```bash
# Check CNI configuration
ssh k3s-master01 "sudo cat /var/lib/rancher/k3s/server/manifests/flannel.yaml"

# Test pod-to-pod connectivity
kubectl run test-pod --image=busybox -it --rm -- sh
# Inside pod: ping <another-pod-ip>
```

#### Storage Issues
```bash
# Check storage class
kubectl get storageclass

# Check PVC status
kubectl get pvc -A

# Check local-path-provisioner logs
kubectl logs -n kube-system deployment/local-path-provisioner
```

### Performance Monitoring
```bash
# Resource usage
kubectl top nodes
kubectl top pods -A

# Cluster information
kubectl cluster-info
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Upgrade Strategy

### k3s Version Upgrades
1. **Test in development**: Use template VM for testing
2. **Backup cluster**: Export configurations and data
3. **Upgrade master**: Apply new k3s version
4. **Verify master**: Ensure control plane stability  
5. **Upgrade workers**: One node at a time
6. **Validate cluster**: Run smoke tests

### Application Updates
- **Rolling updates**: Default Kubernetes strategy
- **Blue-green deployments**: For critical applications
- **Canary deployments**: Gradual rollout with monitoring

## Metrics and Monitoring

### Key Performance Indicators
- **Node resource utilization**: CPU, Memory, Disk
- **Pod scheduling efficiency**: Pod distribution across nodes
- **Network latency**: Inter-pod and external communication
- **Storage performance**: I/O metrics and capacity usage

### Alerting Scenarios
- Node becomes NotReady
- High resource utilization (>80%)
- Pod crash loops or frequent restarts
- etcd performance degradation
- Certificate expiration warnings

This k3s cluster serves as the foundation for learning container orchestration, DevOps practices, and Site Reliability Engineering principles in a production-like environment.
