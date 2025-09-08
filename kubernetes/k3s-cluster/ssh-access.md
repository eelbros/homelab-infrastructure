# SSH Access Documentation

## Overview

Secure Shell (SSH) access is configured across all homelab infrastructure using public key authentication, eliminating the need for password-based authentication and providing secure, automated access for administration and automation.

## SSH Architecture

### Key Management Strategy
- **Centralized key management**: Single key pair for all homelab infrastructure
- **Management laptop**: Arch Linux ThinkPad T470s holds private key
- **Infrastructure nodes**: Hold corresponding public key
- **No password authentication**: Keys-only for enhanced security

### Access Topology
```
Arch Linux Laptop (Management)
├── Private Key: ~/.ssh/homelab_key
├── SSH Config: ~/.ssh/config
│
└── Remote Access To:
    ├── Proxmox Host (pve): 192.168.1.20
    ├── k3s-master01: 10.10.20.20
    ├── k3s-worker01: 10.10.20.21
    ├── k3s-worker02: 10.10.20.22
    └── Future Talos nodes: 10.10.20.100+
```

## SSH Key Configuration

### Key Generation
```bash
# Generated on Arch Linux laptop
ssh-keygen -t ed25519 -f ~/.ssh/homelab_key -C "homelab-infrastructure"

# Key details:
# Algorithm: Ed25519 (modern, secure, fast)
# Key size: 256-bit (equivalent to 3072-bit RSA)
# Location: ~/.ssh/homelab_key (private), ~/.ssh/homelab_key.pub (public)
```

### Key Specifications
- **Algorithm**: Ed25519
- **Purpose**: Authentication for homelab infrastructure
- **Passphrase**: Configured for additional security
- **Comment**: "homelab-infrastructure"

## SSH Client Configuration

### SSH Config File (~/.ssh/config)
```bash
# k3s Kubernetes Cluster
Host k3s-master01
    HostName 10.10.20.20
    User k3s-admin
    IdentityFile ~/.ssh/homelab_key
    StrictHostKeyChecking no

Host k3s-worker01
    HostName 10.10.20.21
    User k3s-admin
    IdentityFile ~/.ssh/homelab_key
    StrictHostKeyChecking no

Host k3s-worker02
    HostName 10.10.20.22
    User k3s-admin
    IdentityFile ~/.ssh/homelab_key
    StrictHostKeyChecking no

# Proxmox Infrastructure
Host pve
    HostName 192.168.1.20
    User root
    IdentityFile ~/.ssh/homelab_key

# Future Talos Cluster
Host talos-01
    HostName 10.10.20.100
    User core
    IdentityFile ~/.ssh/homelab_key

# Global SSH Settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    UserKnownHostsFile ~/.ssh/known_hosts
    IdentitiesOnly yes
```

### SSH Agent Configuration
```bash
# Start SSH agent and add key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/homelab_key

# Verify loaded keys
ssh-add -l

# Auto-load on shell startup (add to ~/.bashrc)
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/homelab_key 2>/dev/null
fi
```

## Node Configuration

### User Account Setup
Each node has a dedicated user account for SSH access:

#### k3s Nodes
- **Username**: `k3s-admin`
- **Home Directory**: `/home/k3s-admin`
- **Sudo Access**: Full administrative privileges
- **Shell**: bash

#### Proxmox Host  
- **Username**: `root`
- **Home Directory**: `/root`
- **Access Level**: Full system administrator

### Public Key Distribution

#### Initial Key Distribution
```bash
# Method 1: Using ssh-copy-id (requires password once)
ssh-copy-id -i ~/.ssh/homelab_key.pub k3s-admin@10.10.20.20

# Method 2: Manual copy (from management laptop)
ssh k3s-admin@10.10.20.20 "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
scp ~/.ssh/homelab_key.pub k3s-admin@10.10.20.20:~/.ssh/authorized_keys
ssh k3s-admin@10.10.20.20 "chmod 600 ~/.ssh/authorized_keys"
```

#### Authorized Keys Files
Location on each node:
```bash
# k3s nodes
/home/k3s-admin/.ssh/authorized_keys

# Proxmox host
/root/.ssh/authorized_keys
```

### File Permissions
Critical for SSH security:
```bash
# On management laptop
chmod 700 ~/.ssh
chmod 600 ~/.ssh/homelab_key
chmod 644 ~/.ssh/homelab_key.pub
chmod 600 ~/.ssh/config

# On remote nodes
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## SSH Access Patterns

### Daily Operations
```bash
# Direct access using host aliases
ssh k3s-master01    # Connect to master node
ssh k3s-worker01    # Connect to worker node
ssh pve             # Connect to Proxmox

# Command execution without login
ssh k3s-master01 "kubectl get nodes"
ssh pve "qm list"

# File transfers
scp local-file.txt k3s-master01:~/
scp k3s-master01:~/remote-file.txt ./
```

### Automation and Scripting
```bash
# Parallel command execution
for node in k3s-master01 k3s-worker01 k3s-worker02; do
    echo "=== $node ==="
    ssh $node "hostname && uptime"
done

# Batch configuration updates
ansible-playbook -i inventory/hosts.yml site.yml

# Remote kubectl operations
ssh k3s-master01 "kubectl get pods -A"
```

## Security Configuration

### SSH Daemon Configuration
Standard security hardening applied to all nodes:

```bash
# /etc/ssh/sshd_config key settings
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password  # Proxmox only
Protocol 2
MaxAuthTries 3
LoginGraceTime 30
```

### Network Security
- **Firewall**: SSH access restricted to management network
- **Port**: Standard port 22 (can be changed for additional security)
- **Source restriction**: Only management laptop subnet allowed

### Key Rotation Strategy
```bash
# Planned key rotation (quarterly)
# 1. Generate new key pair
ssh-keygen -t ed25519 -f ~/.ssh/homelab_key_new

# 2. Distribute new public key
for host in k3s-master01 k3s-worker01 k3s-worker02; do
    ssh-copy-id -i ~/.ssh/homelab_key_new.pub $host
done

# 3. Update SSH config to use new key
# 4. Remove old keys from authorized_keys files
# 5. Delete old key pair
```

## Troubleshooting

### Common Issues and Solutions

#### Permission Denied Errors
```bash
# Check key permissions
ls -la ~/.ssh/homelab_key*

# Fix permissions if incorrect
chmod 600 ~/.ssh/homelab_key
chmod 644 ~/.ssh/homelab_key.pub

# Verify SSH agent has the key
ssh-add -l
```

#### Connection Timeouts
```bash
# Test network connectivity
ping 10.10.20.20

# Test SSH service on remote host
nmap -p 22 10.10.20.20

# Check SSH service status (on remote node)
sudo systemctl status ssh
```

#### Key Authentication Failures
```bash
# Debug SSH connection
ssh -v k3s-master01

# Check authorized_keys on remote host
ssh k3s-master01 "cat ~/.ssh/authorized_keys"

# Compare with local public key
cat ~/.ssh/homelab_key.pub
```

#### SSH Agent Issues
```bash
# Kill existing agents
pkill ssh-agent

# Start fresh agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/homelab_key

# Verify agent is working
ssh-add -l
```

### Connection Verification
```bash
# Test all configured hosts
for host in k3s-master01 k3s-worker01 k3s-worker02 pve; do
    echo "Testing $host..."
    ssh -o ConnectTimeout=5 $host "echo 'Connection successful'"
done
```

## Monitoring and Logging

### SSH Connection Monitoring
```bash
# Check SSH connection logs (on remote hosts)
sudo journalctl -u ssh -f

# Monitor authentication attempts
sudo tail -f /var/log/auth.log | grep ssh
```

### Access Patterns
- **Primary access**: Management laptop during business hours
- **Emergency access**: Direct console access via Proxmox
- **Automated access**: Ansible playbooks for configuration management

## Future Enhancements

### Planned Improvements
1. **Bastion Host**: Implement jump host for additional security layer
2. **SSH Certificates**: Move from keys to certificate-based authentication
3. **Multi-Factor Authentication**: Add TOTP for critical systems
4. **Session Recording**: Implement session audit logging
5. **Key Management**: Centralized key distribution via Vault

### Integration Points
- **Ansible**: Automated configuration management
- **GitOps**: Secure repository access for infrastructure as code
- **Monitoring**: SSH connection metrics and alerting
- **Backup**: Secure backup script execution

This SSH configuration provides secure, efficient access to all homelab infrastructure while maintaining enterprise-grade security practices and enabling automation workflows essential for SRE operations.
