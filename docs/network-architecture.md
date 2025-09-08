# Overview

The homelab network is designed with enterprise-grade segmentation using VLANs to separate different types of traffic and provide security isolation between network segments.

## Physical Infrastructure

### Hardware Components
- **Proxmox Host (pve)**: Primary hypervisor running virtualized infrastructure
- **Talos Server**: Secure API driven Cluster Manager
- **Managed Switch**: VLAN-capable switch with trunk and access ports
- **Wireless Access Point (WAP)**: Multi-SSID support for VLAN assignment
- **ThinkPad T470s**: Management laptop running Arch Linux

### Network Topology

```
Internet
    |
[PfSense Firewall/Router]
    |
[Managed Switch - Trunk Port]
    |
    ├── VLAN 1 (Management Network)
    ├── VLAN 10 (Client Network)
    ├── VLAN 20 (LAB Network)  
    └── VLAN 30 (IoT Network)
     

[Wireless Access Point]
├── Client-WiFi → VLAN 10
├── LAB-WiFi → VLAN 20
├── SmartHome-WiFi → VLAN 30
└── Management WiFi → VLAN 1
```

## Network Segments

### VLAN 1 - Management Network: 192.168.1.0/24 - 192.168.2.0/24
**Purpose**: WAN Interface, Device Management

| Device | IP Address | Role |
|--------|------------|------|
| Internet Modem | 192.168.1.1 | Gateway |
| Proxmox (pve) | 192.168.1.20 | Hypervisor management |
| PfSense | 192.168.1.2 | Firewall/DHCP |
| Pi Hole | 192.168.1.40 | DNS/Ad-blocker |
| Switch Management | 192.168.2.250 | Network device management |
| Wireless AP | 192.168.2.100 | Wireless Access Point with VLAN capability |

- **DHCP Range**: 192.168.1.100-150
- **Access**: Internet, limited homelab access

### VLAN 10 - Client Network: 10.10.10.0/24
**Purpose**: Client Device connection

- **Gateway**: PfSense Client Interface (10.10.10.1)
- **DHCP Range**: 10.10.10.100-200
- **Access**: Internet, Selective Home Network access

### VLAN 20 - LAB Network: 10.10.20.0/24
**Purpose**: Development, testing, and lab infrastructure

| Device | IP Address | Role | VM ID |
|--------|------------|------|-------|
| Gateway | 10.10.20.1 | PfSense LAB interface | - |
| k3s-master01 | 10.10.20.20 | Kubernetes control plane | 105 |
| k3s-worker01 | 10.10.20.21 | Kubernetes worker node | 103 |
| k3s-worker02 | 10.10.20.22 | Kubernetes worker node | 104 |
| Template | - | Ubuntu 22.04 template | 102 |

**Reserved Ranges**:
- 10.10.20.1-19: Infrastructure (gateways, DNS, etc.)
- 10.10.20.20-29: Kubernetes nodes
- 10.10.20.30-99: Additional VMs
- 10.10.20.100-200: DHCP/Dynamic allocation
- 10.10.20.200-250: DHCP/Future Talos cluster

### VLAN 30 - SmartHome Network: 10.30.0.0/24
**Purpose**: IoT devices, home automation

- **Gateway**: PfSense IoT Interface(10.30.0.1)
- **Isolation**: Limited internet access, no inter-VLAN routing
- **DHCP Range**: 10.30.0.100-110

## Kubernetes Internal Networks

### Cluster Network Design
The k3s cluster uses internal network ranges that don't conflict with physical infrastructure:

```
Physical LAB Network: 10.10.20.0/24
├── Node IPs: 10.10.20.20-22
│
Kubernetes Internal Networks:
├── Pod Network (CNI): 10.42.0.0/16
│   ├── Node 1 Pods: 10.42.0.0/24
│   ├── Node 2 Pods: 10.42.1.0/24  
│   └── Node 3 Pods: 10.42.2.0/24
│
├── Service Network: 10.43.0.0/16
│   ├── kubernetes: 10.43.0.1
│   ├── kube-dns: 10.43.0.10
│   └── Applications: 10.43.x.x
│
└── Load Balancer Range: 10.10.20.100-150
    └── External service IPs via MetalLB
```

## Security and Access Control

### Firewall Rules (PfSense)
```
Management → LAB: Allow (admin access)
LAB → Client: Deny (security isolation)
LAB → Internet: Allow (outbound only)
SmartHome → Internet: Limited (specific ports)
SmartHome → Other VLANs: Deny
```

### Inter-VLAN Routing
- **Management ↔ LAB**: Full access for administration
- **LAB → Client**: Blocked for security
- **Client → LAB**: Controlled access for services
- **SmartHome**: Isolated segment

### Network Services

#### DNS Resolution
- **Primary DNS**: PiHole (per VLAN gateway)
- **Secondary DNS**: 8.8.8.8, 8.8.4.4
- **Kubernetes DNS**: CoreDNS (10.43.0.10)

#### DHCP Services
- **Management**: PfSense DHCP
- **Client VLAN**: PfSense DHCP  
- **LAB VLAN**: Static IPs for infrastructure, DHCP for testing
- **SmartHome**: PfSense DHCP

## Future Expansion Plans

### Phase 2: Talos Cluster
- **Bare metal server conversion**
- **IP Range**: 10.10.20.100-110
- **Cross-cluster networking** via MetalLB

### Phase 3: Service Mesh
- **Istio/Linkerd deployment**
- **Cross-cluster service discovery**
- **Advanced traffic management**

### Monitoring and Observability
- **Network monitoring**: LibreNMS/PRTG
- **Flow analysis**: PfSense + Grafana
- **Performance metrics**: Prometheus exporters

## Troubleshooting

### Common Network Issues
1. **VLAN Tagging**: Ensure proper VLAN configuration on switch ports
2. **Routing**: Verify inter-VLAN rules in PfSense
3. **DNS**: Check DNS servers in each VLAN
4. **MTU**: Ensure consistent MTU across VLANs

### Network Validation Commands
```bash
# Test VLAN connectivity
ping 10.10.20.1    # LAB gateway
ping 192.168.1.20  # Proxmox management

# Kubernetes networking
kubectl get nodes -o wide
kubectl get svc -A

# Network troubleshooting
traceroute 8.8.8.8
nslookup kubernetes.default.svc.cluster.local
```

## Network Monitoring

### Key Metrics
- **Bandwidth utilization** per VLAN
- **Inter-VLAN traffic patterns**
- **Kubernetes pod networking performance**  
- **Internet connectivity uptime**

### Alerting Thresholds
- Network utilization > 80%
- Inter-VLAN routing failures
- DNS resolution failures
- Kubernetes network plugin issues

This network architecture provides enterprise-grade segmentation while maintaining flexibility for lab experimentation and future expansion.
