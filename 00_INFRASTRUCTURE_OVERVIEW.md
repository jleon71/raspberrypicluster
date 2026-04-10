# Raspberry Pi 4 Cluster Infrastructure - Complete Guide

## Project Overview

This guide provides a complete, free infrastructure automation framework for managing a Raspberry Pi 4 cluster running:
- **Rocky Linux 9** (ARM64/aarch64, enterprise-ready)
- **K3s** (lightweight Kubernetes)
- **Grafana** (monitoring and visualization)
- **Cloudflare Agent** (DNS and network connectivity)

**Alternative OS:** This guide originally supported Debian but has been updated to use Rocky Linux. See troubleshooting section if you need Debian version.

**Cost:** Completely free (uses open-source tools and free tiers)
**Scalability:** Designed to grow from 1-3 devices to 10+ with minimal changes

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                 Control Machine                         │
│         (Your laptop/desktop running Linux/Mac)         │
│  - Ansible (configuration management)                   │
│  - kubectl (Kubernetes management)                       │
│  - Git (version control)                                │
└────────────┬────────────────────────────────────────────┘
             │ SSH & API calls
    ┌────────┴──────────────────────────┐
    │                                   │
┌───▼──────────────────┐    ┌──────────▼─────────────────┐
│  Raspberry Pi 1      │    │  Raspberry Pi 2            │
│  (Master Node)       │    │  (Worker Node)             │
│  - Debian 12         │    │  - Debian 12               │
│  - K3s Master        │    │  - K3s Agent               │
│  - Grafana           │    │  - Prometheus Node Exp.    │
│  - Cloudflare Agent  │    │  - Cloudflare Agent        │
└──────────────────────┘    └────────────────────────────┘
         │                           │
         └───────────────────┬───────┘
                             │
                    Cloudflare Network
                    (DNS, routing, security)
```

---

## Free Tools & Services Used

### Infrastructure Automation
- **Ansible** - Infrastructure as Code (IaC) with dnf and firewalld
- **Docker/Containerd** - Container runtime (K3s uses containerd)
- **K3s** - Lightweight Kubernetes distribution (free, minimal resources)
- **dnf** - Rocky Linux package manager (replaces apt)
- **firewalld** - Rocky Linux firewall management

### Monitoring & Observability
- **Grafana** - Open-source dashboards and visualization
- **Prometheus** - Metrics collection and storage
- **Node Exporter** - Hardware metrics from Raspberry Pis

### Networking & DNS
- **Cloudflare Agent** - Free DNS, DDoS protection, and edge connectivity
- **firewalld** - Advanced firewall with rule management
- **SELinux** - Optional enhanced security (permissive by default)

### Version Control & Documentation
- **Git/GitHub** - Free repository for your infrastructure code
- **Markdown** - Documentation

---

## Prerequisites

### Hardware
- Raspberry Pi 4 with 4GB+ RAM (8GB recommended)
- 32GB+ microSD card (or USB SSD)
- USB-C power supply (2.5A minimum)
- Ethernet or good WiFi connection
- Optional: USB SSD for better performance

### Software on Control Machine
- Linux, macOS, or WSL2 on Windows
- Python 3.8+
- Ansible 2.10+
- kubectl
- Git

### Accounts (All Free)
- Cloudflare account (free tier)
- GitHub account (free) - optional but recommended

---

## Quick Start Timeline

| Phase | Duration | Tasks |
|-------|----------|-------|
| **Phase 1: Foundation** | 1-2 hours | Install Debian, basic Ansible setup |
| **Phase 2: K3s Cluster** | 1-2 hours | Deploy K3s master and nodes |
| **Phase 3: Monitoring** | 1-2 hours | Set up Prometheus and Grafana |
| **Phase 4: Networking** | 30-60 min | Configure Cloudflare Agent |
| **Phase 5: Automation** | 1-2 hours | Create Ansible playbooks for scaling |

**Total: 5-9 hours for complete setup**

---

## What Makes This Infrastructure Scalable

1. **Automated Provisioning** - Ansible playbooks can deploy to new devices automatically
2. **Kubernetes Native** - K3s handles scheduling and orchestration
3. **Declarative Configuration** - All settings in Git for version control
4. **Modular Design** - Each component (K3s, Grafana, Cloudflare) is independent
5. **Low Resource Footprint** - K3s and Grafana are lightweight enough for Pi hardware

---

## Next Steps

This guide is organized into the following sections:

1. **01_DEBIAN_INSTALLATION.md** - Initial OS setup
2. **02_ANSIBLE_SETUP.md** - Control machine configuration
3. **03_K3S_DEPLOYMENT.md** - Kubernetes cluster creation
4. **04_PROMETHEUS_GRAFANA.md** - Monitoring setup
5. **05_CLOUDFLARE_AGENT.md** - DNS and network configuration
6. **06_SCALING_GUIDE.md** - Adding more Raspberry Pis
7. **07_BACKUP_RECOVERY.md** - Data protection strategies
8. **ansible/** - Playbooks directory (ready-to-use Ansible code)

---

## Key Principles

- **Infrastructure as Code (IaC)** - Everything is version-controlled
- **Declarative** - Describe desired state, not steps
- **Idempotent** - Running playbooks multiple times is safe
- **Automated** - Minimal manual intervention after setup
- **Free & Open Source** - No vendor lock-in or licensing costs

---

## Cost Analysis

| Component | Cost | Notes |
|-----------|------|-------|
| Raspberry Pi 4 (8GB) | ~$75 each | One-time hardware |
| MicroSD Card (256GB) | ~$20-30 | One-time storage |
| Power Supply | ~$10 | One-time |
| Cloudflare Agent | **FREE** | Free tier |
| K3s | **FREE** | Open source |
| Grafana | **FREE** | Open source, no license needed |
| Prometheus | **FREE** | Open source |
| Ansible | **FREE** | Open source |
| **Total (for 3 Pis)** | **~$280-300** | All software free forever |

---

## Support & Resources

- **K3s Docs:** https://docs.k3s.io
- **Ansible Docs:** https://docs.ansible.com
- **Grafana Docs:** https://grafana.com/docs/
- **Cloudflare Docs:** https://developers.cloudflare.com

---

**Last Updated:** 2026-04-06
**Status:** Ready for implementation.
