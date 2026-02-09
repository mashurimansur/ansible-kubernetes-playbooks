# Kubernetes Playbook Setup Guide for Ubuntu 24.04

## Environment Configuration

This playbook has been updated to support:
- **Master Node**: 10.10.10.50 (Ubuntu 24.04)
- **Worker Node**: 10.10.10.51 (Ubuntu 24.04)
- **Kubernetes Version**: 1.29
- **Container Runtime**: Containerd
- **Network Plugin**: Flannel

## Prerequisites

On your local machine:
1. Install Ansible:
   ```bash
   # macOS with Homebrew
   brew install ansible
   
   # Or with pip
   pip install ansible
   ```

2. Ensure SSH access to both VMs:
   ```bash
   # Copy your SSH public key to both VMs
   ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@10.10.10.50
   ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@10.10.10.51
   ```

3. Verify connectivity:
   ```bash
   ssh ubuntu@10.10.10.50 "echo Master OK"
   ssh ubuntu@10.10.10.51 "echo Worker OK"
   ```

## Running the Playbook

### Step 1: Install Common Dependencies
This installs containerd, kubelet, kubeadm, and configures kernel modules on all nodes:

```bash
ansible-playbook -i hosts playbooks/kube-dependencies.yml
```

The playbook will:
- Verify Ubuntu 22.04 or 24.04 OS
- Update APT packages and reboot
- Disable SWAP (Kubernetes requirement)
- Configure kernel modules (overlay, br_netfilter)
- Configure sysctl parameters for networking
- Install containerd and containerd configuration
- Install kubelet and kubeadm
- Reboot again for final configuration

### Step 2: Initialize Master Node
```bash
ansible-playbook -i hosts playbooks/master.yml
```

The playbook will:
- Create kubeadm configuration with systemd cgroup driver
- Initialize the Kubernetes cluster
- Configure kubectl for the ubuntu user
- Deploy Flannel CNI plugin

### Step 3: Join Worker Nodes
```bash
ansible-playbook -i hosts playbooks/workers.yml
```

The playbook will:
- Generate join command from master node
- Wait for master API port (6443) to be reachable
- Join the worker node to the cluster

## Verification

After the playbook runs, verify the cluster:

```bash
# SSH into master
ssh ubuntu@10.10.10.50

# Check nodes
kubectl get nodes

# Check pods across all namespaces
kubectl get pods -A

# Check flannel pod deployment
kubectl get pods -n kube-flannel
```

Expected output should show:
- Master node in Ready state
- Worker node in Ready state (after a few seconds)
- Flannel pods running in kube-flannel namespace

## Troubleshooting

### Issue: Nodes not in Ready state
```bash
# Check kubelet status on affected node
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet -n 100
```

### Issue: Flannel pods not running
```bash
# Check flannel logs
kubectl logs -n kube-flannel -l app=flannel
```

### Issue: Worker cannot join cluster
```bash
# Ensure port 6443 is open on master
sudo ufw allow 6443

# Test connectivity from worker
nc -zv 10.10.10.50 6443
```

### Common Ubuntu 24.04 Issues
- The playbook has been updated to support Ubuntu 24.04 alongside 22.04
- All package repositories (Docker, Kubernetes) support Ubuntu 24.04 noble release

## Key Changes for Ubuntu 24.04 Support

1. **OS Detection**: Updated to accept both Ubuntu 22.04 and 24.04
2. **Package Compatibility**: Docker and Kubernetes repositories fully support Ubuntu 24.04
3. **Worker Join**: Fixed node reference to use hardcoded IP (10.10.10.50) instead of dynamic host facts
4. **Timeout**: Increased wait_for timeout from 1s to 30s for reliable master connectivity

## Inventory File (hosts)

The `hosts` file has been created with your VM configuration:
```ini
[master]
k8s-master-1 ansible_host=10.10.10.50

[workers]
k8s-worker-1 ansible_host=10.10.10.51

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_extra_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_user=ubuntu
```

Adjust `ansible_ssh_private_key_file` if your SSH key is stored elsewhere.

## Additional Resources

- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Kubeadm Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Flannel CNI Plugin](https://github.com/flannel-io/flannel)
- [Containerd Documentation](https://containerd.io/docs/)
