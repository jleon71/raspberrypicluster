# Phase 6: Scaling Your Infrastructure (Rocky Linux)

## Overview

Scale your Rocky Linux Raspberry Pi cluster from 1-3 nodes to 10+ nodes efficiently using automation.

**Estimated Time:** Varies (mostly waiting for hardware)

---

## What You'll Do

1. Plan cluster expansion
2. Add new Raspberry Pis using automation
3. Configure load balancing
4. Monitor cluster growth
5. Optimize resource usage
6. Handle node failures

---

## Prerequisites

- Working 1-3 node cluster
- Ansible playbooks from Phase 2
- Terraform (optional but recommended for IaC)

---

## Step 1: Plan Cluster Expansion

### Scaling Checklist

Before adding nodes, consider:

- **Network capacity**: Do you have IP addresses for new nodes?
- **Power requirements**: Each Pi needs 2.5A USB-C supply
- **Storage**: NVMe SSD for better performance (optional)
- **Network bandwidth**: Will your network handle the traffic?
- **Cooling**: Proper ventilation for stability
- **Monitoring**: Can your monitoring stack handle more metrics?

### Recommended Configurations

#### Small Cluster (1-3 nodes)
```
Total Cost: $250-300
Hardware: 3x Pi 4 (4GB)
Master: 1 node
Workers: 2 nodes
Use Case: Development, learning, small workloads
```

#### Medium Cluster (4-10 nodes)
```
Total Cost: $600-800
Hardware: 10x Pi 4 (8GB)
Master: 2 nodes (HA)
Workers: 8 nodes
Use Case: Production workloads, monitoring
```

#### Large Cluster (10+ nodes)
```
Total Cost: $1500+
Hardware: 20+ Pi 4 (8GB), Mix with Pi 5 or other ARM boards
Masters: 3 nodes (HA)
Workers: 17+ nodes
Use Case: Distributed services, edge computing
```

---

## Step 2: Network Planning

### IP Address Management

Create a subnet allocation:

```
192.168.1.0/24 - Raspberry Pi Cluster

Master Nodes:
  192.168.1.100-102 (reserved for 3 HA masters)

Worker Nodes:
  192.168.1.110-150 (40 available addresses)

Services:
  192.168.1.200+ (reserved for services/loadbalancer)
```

---

## Step 3: Automated Node Addition

### 3.1 Update Inventory

Edit `inventory/hosts.ini`:

```ini
[all:vars]
ansible_user=debian
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3

[masters]
rpi-master-1 ansible_host=192.168.1.100 node_type=master
rpi-master-2 ansible_host=192.168.1.101 node_type=master (optional for HA)

[workers]
rpi-worker-1 ansible_host=192.168.1.110 node_type=worker
rpi-worker-2 ansible_host=192.168.1.111 node_type=worker
rpi-worker-3 ansible_host=192.168.1.112 node_type=worker
rpi-worker-4 ansible_host=192.168.1.113 node_type=worker
# ... add more as needed

[k3s_cluster:children]
masters
workers
```

### 3.2 Run Ansible on New Nodes

```bash
# Test connectivity to new nodes
ansible new-node -m ping -i inventory/hosts.ini

# Run common configuration
ansible-playbook playbooks/01-common.yml -i inventory/hosts.ini --limit new-node

# Install K3s
ansible-playbook playbooks/02-k3s.yml -i inventory/hosts.ini --limit new-node

# Verify node joined cluster
ssh debian@192.168.1.100
sudo k3s kubectl get nodes
```

---

## Step 4: Configure High Availability (HA)

For production clusters, set up HA master nodes.

### 4.1 HA Architecture

```
LoadBalancer (Virtual IP)
        |
        +-- Master 1 (192.168.1.100)
        +-- Master 2 (192.168.1.101)
        +-- Master 3 (192.168.1.102)
        |
        +-- Worker 1-N
```

### 4.2 Setup HA with Keepalived (Rocky Linux)

Install keepalived with dnf:

```bash
ansible all -m dnf -a "name=keepalived state=present"
```

Create `playbooks/03-k3s-ha.yml`:

```yaml
---
- name: Setup K3s HA with Keepalived
  hosts: masters
  become: yes

  tasks:
    - name: Install keepalived
      apt:
        name: keepalived
        state: present

    - name: Configure keepalived on first master
      copy:
        dest: /etc/keepalived/keepalived.conf
        content: |
          global_defs {
            router_id K3S_HA
            script_user root root
            enable_script_security
          }

          vrrp_script check_k3s {
            script "/usr/bin/systemctl is-active k3s"
            interval 5
            weight -20
          }

          vrrp_instance k3s {
            state MASTER
            interface eth0
            virtual_router_id 51
            priority 100
            advert_int 1

            authentication {
              auth_type PASS
              auth_pass k3sha
            }

            virtual_ipaddress {
              192.168.1.50
            }

            track_script {
              check_k3s
            }
          }
      when: inventory_hostname == groups['masters'][0]

    - name: Configure keepalived on backup masters
      copy:
        dest: /etc/keepalived/keepalived.conf
        content: |
          global_defs {
            router_id K3S_HA
          }

          vrrp_instance k3s {
            state BACKUP
            interface eth0
            virtual_router_id 51
            priority 90
            advert_int 1

            authentication {
              auth_type PASS
              auth_pass k3sha
            }

            virtual_ipaddress {
              192.168.1.50
            }
          }
      when: inventory_hostname != groups['masters'][0]

    - name: Start keepalived
      service:
        name: keepalived
        state: started
        enabled: yes

    - name: Verify virtual IP
      shell: ip addr show | grep 192.168.1.50
      register: vip_check
      changed_when: false
      failed_when: vip_check.rc != 0

    - name: Update kubeconfig to use virtual IP
      shell: |
        sed -i 's/192.168.1.100:6443/192.168.1.50:6443/g' /etc/rancher/k3s/k3s.yaml
      when: inventory_hostname == groups['masters'][0]
```

Deploy HA:

```bash
# Setup second and third master
ansible-playbook playbooks/02-k3s.yml --limit rpi-master-2
ansible-playbook playbooks/02-k3s.yml --limit rpi-master-3

# Configure HA
ansible-playbook playbooks/03-k3s-ha.yml

# Verify
ssh debian@192.168.1.100
sudo k3s kubectl get nodes
```

---

## Step 5: Load Balancing

### 5.1 MetalLB for Load Balancing

For bare-metal clusters, use MetalLB to assign external IPs.

Create `k3s-manifests/metallb-config.yml`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
  labels:
    app: metallb
    version: v0.13.5

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: metallb
  namespace: kube-system
spec:
  chart: metallb
  repo: https://metallb.universe.tf
  targetNamespace: metallb-system
  set:
    configInline:
      address-pools:
      - name: default
        protocol: layer2
        addresses:
        - 192.168.1.200-192.168.1.250

---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Deploy:

```bash
kubectl apply -f k3s-manifests/metallb-config.yml

# Check load balancer status
kubectl get all -n metallb-system

# Services will now get external IPs
kubectl get svc
```

---

## Step 6: Monitor Cluster Growth

### 6.1 Grafana Dashboard for Cluster Size

In Grafana, create a dashboard showing:

```promql
# Node count
count(kube_node_info)

# Pod count
count(kube_pod_info)

# Total CPU
sum(node_cpu_seconds_total)

# Total Memory
sum(node_memory_MemTotal_bytes)

# Available Resources
sum(kube_allocatable_cpu_cores)
sum(kube_allocatable_memory_bytes)
```

### 6.2 Cluster Capacity Planning

```bash
# Check cluster capacity
kubectl describe nodes | grep -A 2 "Allocated resources"

# Get detailed metrics
kubectl top nodes

# Monitor pod allocation
kubectl describe nodes | grep -E "Name:|Allocated resources" -A 5
```

---

## Step 7: Storage Scaling

### 7.1 Expand Storage for Prometheus/Grafana

```bash
# Check current PVC size
kubectl get pvc -n monitoring

# Update PVC size
kubectl patch pvc prometheus-db-prometheus-0 -n monitoring -p \
  '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# Note: Can only increase, not decrease
```

### 7.2 Distributed Storage (Optional)

For larger clusters, consider distributed storage like Longhorn:

```bash
# Install Longhorn
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace

# Use as StorageClass
kubectl patch storageclass local-path -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass longhorn -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Step 8: Auto-Scaling Configuration

### 8.1 Horizontal Pod Autoscaling

Create `k3s-manifests/hpa-example.yml`:

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-world-hpa
  namespace: applications
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-world

  minReplicas: 2
  maxReplicas: 10

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

Deploy:

```bash
kubectl apply -f k3s-manifests/hpa-example.yml

# Watch autoscaling
kubectl get hpa -w
kubectl describe hpa hello-world-hpa -n applications
```

### 8.2 Cluster Autoscaling (Manual)

K3s doesn't have built-in cluster autoscaling, but you can:

1. Monitor node utilization
2. Add nodes when CPU/Memory usage exceeds 80%
3. Use Ansible to automate additions

---

## Step 9: Node Maintenance and Rotation

### 9.1 Drain Node for Maintenance

```bash
# Gracefully remove node from service
kubectl drain rpi-worker-1 --ignore-daemonsets --delete-emptydir-data

# Do maintenance (upgrade OS, update hardware, etc.)

# Return node to service
kubectl uncordon rpi-worker-1
```

### 9.2 Remove Node from Cluster

```bash
# Delete node
kubectl delete node rpi-worker-1

# On the node itself, reset K3s
ssh debian@192.168.1.110
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

---

## Step 10: Performance Optimization for Large Clusters

### 10.1 Tune K3s for Scale (Rocky Linux)

Update K3s configuration on all nodes with dnf and sysctl:

Create `playbooks/tune-k3s.yml`:

```yaml
---
- name: Tune K3s for Large Clusters
  hosts: k3s_cluster
  become: yes

  tasks:
    - name: Increase file descriptors
      lineinfile:
        path: /etc/security/limits.conf
        line: "{{ item }}"
      loop:
        - "* soft nofile 1000000"
        - "* hard nofile 1000000"
        - "* soft nproc 1000000"
        - "* hard nproc 1000000"

    - name: Tune kernel parameters
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        state: present
      loop:
        - { key: "net.ipv4.ip_forward", value: "1" }
        - { key: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { key: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { key: "vm.max_map_count", value: "262144" }

    - name: Restart K3s to apply
      systemd:
        name: k3s
        state: restarted
      when: "'masters' in group_names"
```

---

## Step 11: Monitoring Large Clusters

### Disable unnecessary metrics

For clusters with 50+ nodes, reduce monitoring overhead:

```yaml
# In Prometheus values
prometheus:
  prometheusSpec:
    # Reduce scrape interval
    scrapeInterval: 60s

    # Disable expensive metrics
    serviceMonitorSelectorNilUsesHelmValues: false
```

---

## Scaling Checklist

- [ ] Plan network IPs and subnets
- [ ] Prepare new hardware (Pis, power supplies, SD cards)
- [ ] Flash Debian images
- [ ] Update Ansible inventory
- [ ] Run Ansible playbooks
- [ ] Verify nodes joined cluster
- [ ] Update Cloudflare DNS (if needed)
- [ ] Update Grafana dashboards
- [ ] Test applications
- [ ] Monitor metrics for 24 hours
- [ ] Optimize resource usage if needed

---

## Troubleshooting Scaled Clusters

### Node not joining

```bash
# Check K3s logs
ssh debian@<new-node-ip>
sudo journalctl -u k3s-agent -n 50

# Verify connectivity to master
ping 192.168.1.100

# Check token
cat /var/lib/rancher/k3s/agent/token
```

### High memory usage on master

```bash
# Check etcd size (only on master)
ssh debian@192.168.1.100
sudo k3s kubectl get etcdctl | head -20

# Compact etcd if needed
sudo k3s kubectl exec -n kube-system -it etcd-rpi-master -- \
  etcdctl compact
```

### Slow API server with many nodes

```bash
# Increase API server resource limits
kubectl edit deployment kube-apiserver -n kube-system

# Or update K3s service
ssh debian@192.168.1.100
sudo nano /etc/systemd/system/k3s.service
# Add: --kube-apiserver-arg=etcd-compaction-interval=5m
```

---

## Next Steps

1. **07_BACKUP_RECOVERY.md** - Backup and disaster recovery
2. Deploy your applications
3. Set up CI/CD pipelines
4. Monitor and optimize continuously

---

**Status:** Infrastructure ready to scale
**Next:** Backup and recovery strategies
