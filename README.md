# RPi K3s Homelab

A 5-node Raspberry Pi K3s cluster managed with Ansible. This repository contains everything needed to bootstrap, configure, and operate the cluster — from bare-metal Pi setup through to running production-grade workloads.

## Hardware

| Node | Model | IP (LAN) | IP (Tailscale) | Role |
|------|-------|----------|-----------------|------|
| rpi1 | Raspberry Pi 4 | 192.168.0.101 | 100.x.x.x | Control plane |
| rpi2 | Raspberry Pi 4 | 192.168.0.103 | 100.x.x.x | Worker |
| rpi3 | Raspberry Pi 4 | 192.168.0.106 | 100.x.x.x | Worker |
| rpi4 | Raspberry Pi 4 | 192.168.0.104 | 100.x.x.x | Worker |
| rpi5 | Raspberry Pi 4 | 192.168.0.105 | 100.x.x.x | Worker |

- SSH user: `rpi`
- Gateway / DNS: `192.168.0.1`
- MetalLB address pool: `192.168.0.200–220`

## Architecture

```
                        ┌─────────────┐
                        │   Router     │
                        │ 192.168.0.1 │
                        └──────┬──────┘
                               │ LAN
            ┌──────────────────┼──────────────────┐
            │                  │                  │
     ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
     │    rpi1      │   │    rpi2      │   │    rpi3      │
     │ Control Plane│   │   Worker     │   │   Worker     │
     │ .101         │   │ .103         │   │ .106         │
     └──────────────┘   └──────────────┘   └──────────────┘
            │                  │                  │
     ┌──────┴──────┐   ┌──────┴──────┐           │
     │    rpi4      │   │    rpi5      │           │
     │   Worker     │   │   Worker     │           │
     │ .104         │   │ .105         │           │
     └──────────────┘   └──────────────┘           │
                                                   │
     ┌──────────────────────────────────────────────┘
     │
     │  Tailscale mesh VPN (100.x) — fallback when LAN is down
     └─────────────────────────────────────────────
```

All nodes run Tailscale for out-of-LAN access. The `tailscale_inventory.ini` provides a fallback inventory when the router or LAN is unreachable.

## Cluster Components

```
┌─────────────────────────────────────────────────────────┐
│                     K3s Cluster                          │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  MetalLB     │  │ ingress-nginx│  │ cert-manager  │   │
│  │ .200-.220    │  │ (DaemonSet) │  │ + local CA    │   │
│  └─────────────┘  └─────────────┘  └──────────────┘   │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  Longhorn    │  │ Prometheus   │  │   Grafana     │   │
│  │ (storage)    │  │ + exporters  │  │               │   │
│  └─────────────┘  └─────────────┘  └──────────────┘   │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  MinIO       │  │   Harbor     │  │  OpenChoreo   │   │
│  │ (operator +  │  │ (registry)   │  │  (Thunder)    │   │
│  │  tenant)     │  │              │  │               │   │
│  └─────────────┘  └─────────────┘  └──────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  OpenBao (Vault) ← External Secrets Operator     │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Networking

| Component | Purpose | Access |
|-----------|---------|--------|
| **MetalLB** | Bare-metal LoadBalancer (L2) | Pool `192.168.0.200–220` |
| **ingress-nginx** | Ingress controller (DaemonSet, default class) | Via MetalLB IPs |
| **cert-manager** | Automated TLS certificates | Self-signed local CA (`*.k3s.local`) |

### Storage

| Component | Purpose | Details |
|-----------|---------|---------|
| **Longhorn** | Distributed block storage | Default StorageClass, 2 replicas, reclaimPolicy: Delete |
| **local-path** | Built-in K3s storage | Used by Prometheus/Grafana (no replication) |

### Monitoring

| Component | Purpose | Access |
|-----------|---------|--------|
| **kube-prometheus-stack** | Prometheus + Grafana + node-exporter | `prometheus.k3s.local`, `grafana.k3s.local` |
| **Prometheus** | Metrics, 7d retention, 10Gi PVC | Port 9090 |
| **Grafana** | Dashboards, 5Gi persistence | Port 80 |

### Registries & Object Storage

| Component | Purpose | Access |
|-----------|---------|--------|
| **Harbor** | Container image registry | `harbor.k3s.local` (HTTP, Longhorn-backed) |
| **MinIO** | S3-compatible object storage | `minio.k3s.local`, `minio-console.k3s.local` |

### Platform

| Component | Purpose | Access |
|-----------|---------|--------|
| **OpenChoreo (Thunder)** | API platform / developer portal | `thunder.openchoreo.192-168-0-202.nip.io` |
| **OpenBao** | Vault-compatible secrets engine | Internal (`openbao.openbao.svc:8200`) |
| **External Secrets** | Syncs K8s secrets from OpenBao | ClusterSecretStore in `openbao` namespace |

## Documentation

A contents page for the `ansible/` directory — every folder, what it contains, and what each file does — is in:

- [ansible/DOCS.md](ansible/DOCS.md)

## Repository Layout

```
rpi-k3s/
├── README.md                          # This file
├── .gitignore                         # Secrets, state, editor artifacts
└── ansible/
    ├── ansible.cfg                    # Ansible defaults (inventory, SSH, caching)
    │
    ├── # Inventories
    ├── inventory.ini                  # LAN IPs, group [rpi]
    ├── k3s_inventory.ini              # Adds [control_plane] and [workers] groups
    ├── tailscale_inventory.ini        # Tailscale 100.x fallback, group [raspberry_pis]
    │
    ├── # Ansible playbooks (run against the Pis via ansible-playbook)
    ├── playbooks/
    │   ├── static_ips.yml             # Assign permanent static IPs via NetworkManager
    │   ├── enable_cgroups.yml         # Enable memory cgroups in cmdline.txt (K3s requirement)
    │   ├── enable_cgroups_2.yml       # Older variant — prefer enable_cgroups.yml
    │   ├── install_k3s.yml            # Install K3s server + agents, verify cluster
    │   ├── uninstall_k3s.yml          # Full K3s teardown and reboot
    │   ├── disable_k3s_defaults.yml   # Disable Traefik + ServiceLB (replaced by MetalLB/nginx)
    │   ├── install_helm.yml           # Install Helm 3 on control plane
    │   ├── install_metallb.yml        # Install MetalLB v0.15.2 native manifests
    │   ├── setup_ftp.yml              # vsftpd + chroot-jailed FTP users (LAN only)
    │   └── cluster_health.yml         # Read-only diagnostics (hostname, disk, RAM, temp, etc.)
    │
    ├── # Kubernetes manifests (kubectl apply -f on rpi1)
    ├── manifests/
    │   ├── metallb-config.yml         # IPAddressPool + L2Advertisement
    │   ├── ingress-nginx.yml          # nginx ingress controller (HelmChart CR)
    │   ├── cert-manager.yml           # cert-manager (HelmChart CR)
    │   ├── local-ca.yml               # Self-signed CA + ClusterIssuer
    │   ├── longhorn.yml               # Longhorn storage (HelmChart CR)
    │   ├── monitoring.yml             # kube-prometheus-stack (HelmChart CR)
    │   ├── prometheus-ingress.yml     # Ingress: prometheus.k3s.local
    │   ├── grafana-ingress.yml        # Ingress: grafana.k3s.local
    │   ├── minio-operator.yml         # MinIO operator (HelmChart CR)
    │   ├── minio-tenant.yml           # MinIO tenant (1 server, 10Gi Longhorn)
    │   ├── minio-tls.yml              # TLS certificate for MinIO
    │   ├── minio-ingress.yml          # Ingress: minio.k3s.local (HTTP)
    │   ├── minio-ingress-tls.yml      # Ingress: minio.k3s.local (HTTPS)
    │   ├── harbor.yml                 # Harbor registry (HelmChart CR)
    │   ├── openchoreo-secret.yml      # OpenChoreo bootstrap CA
    │   ├── openbao-secrets.yml        # External Secrets → OpenBao ClusterSecretStore
    │   ├── thunder-values.yaml        # Helm values for OpenChoreo Thunder
    │   └── register-backstage-client.yaml # One-shot Job: register Backstage OAuth2 client
    │
    ├── # Smoke tests (kubectl apply → verify → delete)
    ├── tests/
    │   ├── metallb-test.yml           # nginx LoadBalancer — verify MetalLB assigns IP
    │   ├── ingress-test.yml           # nginx + Ingress at test.k3s.local
    │   ├── storage-test.yml           # busybox writing to local-path PVC
    │   └── longhorn-test.yml          # busybox writing to Longhorn PVC
    │
    ├── # Dokploy (Docker Swarm) playbooks
    ├── Dokploy/
    │   ├── install-dokploy.yml            # Docker + Swarm + Dokploy on the Pis
    │   └── install-dokploy-from-scratch.yml # Clean-slate Docker/Dokploy install over Tailscale
    │
    └── DOCS.md                       # Contents page — folders + every file (see Documentation above)
```

## Prerequisites

- Raspberry Pi OS (64-bit) on all nodes
- SSH key-based auth for user `rpi`
- Ansible installed on your control machine
- Tailscale installed on all nodes + your workstation (for fallback access)

## Bootstrapping the Cluster

Run all commands from the `ansible/` directory.

### 1. Assign Static IPs

Uses the Tailscale inventory (run from anywhere with VPN access):

```sh
ansible-playbook -i tailscale_inventory.ini playbooks/static_ips.yml
```

Sets permanent IPs via NetworkManager on each Pi (runs one at a time).

### 2. Enable cgroups

Required for K3s on Raspberry Pi OS:

```sh
ansible-playbook -i inventory.ini playbooks/enable_cgroups.yml
```

Appends `cgroup_enable=memory cgroup_memory=1` to `cmdline.txt` and reboots.

### 3. Install K3s

Bootstraps the control plane on rpi1, then joins all workers:

```sh
ansible-playbook -i k3s_inventory.ini playbooks/install_k3s.yml
```

### 4. Disable K3s defaults

Disables the built-in Traefik and ServiceLB (replaced by MetalLB + ingress-nginx):

```sh
ansible-playbook -i k3s_inventory.ini playbooks/disable_k3s_defaults.yml
```

### 5. Install Helm

```sh
ansible-playbook -i k3s_inventory.ini playbooks/install_helm.yml
```

### 6. Install MetalLB

```sh
ansible-playbook -i k3s_inventory.ini playbooks/install_metallb.yml
```

Then apply the IP pool configuration on rpi1:

```sh
k3s kubectl apply -f manifests/metallb-config.yml
```

### 7. Install ingress-nginx

```sh
k3s kubectl apply -f manifests/ingress-nginx.yml
```

## Applying Workloads

All Kubernetes manifests are applied on the control plane (rpi1) via `kubectl`:

```sh
k3s kubectl apply -f <manifest>.yml
```

There is no Ansible playbook that applies them automatically — this is intentional for flexibility.

### Recommended apply order

Some components depend on others. Apply in this order:

```
1. manifests/cert-manager.yml      # Certificate management
2. manifests/local-ca.yml          # Self-signed CA + ClusterIssuer
3. manifests/longhorn.yml          # Distributed storage
4. manifests/monitoring.yml        # Prometheus + Grafana
5. manifests/prometheus-ingress.yml # Expose Prometheus
6. manifests/grafana-ingress.yml   # Expose Grafana
7. manifests/minio-operator.yml    # MinIO operator
8. manifests/minio-tenant.yml      # MinIO tenant
9. manifests/minio-tls.yml         # TLS cert for MinIO
10. manifests/minio-ingress-tls.yml # Expose MinIO (HTTPS)
11. manifests/harbor.yml           # Container registry
12. manifests/openbao-secrets.yml  # External Secrets store
13. manifests/openchoreo-secret.yml # OpenChoreo CA
14. manifests/thunder-values.yaml  # OpenChoreo Thunder (helm install)
```

## Smoke Tests

Validate individual components after installation:

```sh
# MetalLB — verify an IP is assigned
k3s kubectl apply -f tests/metallb-test.yml

# Ingress — verify host-based routing
k3s kubectl apply -f tests/ingress-test.yml

# Local-path storage — verify PVC write
k3s kubectl apply -f tests/storage-test.yml

# Longhorn storage — verify distributed PVC write
k3s kubectl apply -f tests/longhorn-test.yml
```

Clean up after testing:

```sh
k3s kubectl delete -f tests/metallb-test.yml
k3s kubectl delete -f tests/ingress-test.yml
k3s kubectl delete -f tests/storage-test.yml
k3s kubectl delete -f tests/longhorn-test.yml
```

## Cluster Health Check

Run diagnostics across all nodes (hostnames, IPs, connectivity, disk, RAM, CPU, temperature, routing):

```sh
ansible-playbook -i inventory.ini playbooks/cluster_health.yml
```

## Tearing Down

Full K3s removal across all nodes (uninstalls, removes data, reboots):

```sh
ansible-playbook -i k3s_inventory.ini playbooks/uninstall_k3s.yml
```

## Troubleshooting

### Node not joining cluster

- Check that cgroups are enabled: `ssh rpi@<ip> "cat /proc/cgroups | grep memory"`
- Verify K3s server is running on rpi1: `ssh rpi@192.168.0.101 "sudo systemctl status k3s"`
- Check agent logs on the worker: `ssh rpi@<ip> "sudo journalctl -u k3s-agent -f"`

### MetalLB not assigning IPs

- Verify MetalLB pods are running: `k3s kubectl get pods -n metallb-system`
- Check the IPAddressPool: `k3s kubectl get ipaddresspool -n metallb-system -o yaml`
- Ensure no conflicts with existing LoadBalancer services

### Ingress not routing

- Verify ingress-nginx pod is running: `k3s kubectl get pods -n ingress-nginx`
- Check that `nginx` is the default IngressClass: `k3s kubectl get ingressclass`
- DNS: add `192.168.0.101 <host>.k3s.local` to `/etc/hosts` or your router's DNS

### Longhorn volumes stuck

- Check node disks: `k3s kubectl get nodes -l longhorn.io/node --show-labels`
- Verify sufficient disk space: run `playbooks/cluster_health.yml`
- Check Longhorn manager logs: `k3s kubectl logs -n longhorn-system -l app=longhorn-manager`

### Accessing the cluster from your workstation

**LAN:**
```sh
export KUBECONFIG=/path/to/k3s.yaml
# or copy from rpi1:
scp rpi@192.168.0.101:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

**Tailscale (off-LAN):**
```sh
# Use the Tailscale IP of rpi1 (see tailscale_inventory.ini)
ssh rpi@100.x.x.x "k3s kubectl get nodes"
```

## Notes

- The server IP `192.168.0.101` is hardcoded in `playbooks/install_k3s.yml`.
- `enable_cgroups_2.yml` is an older variant; prefer `enable_cgroups.yml` (both in `playbooks/`).
- Secrets are committed in `manifests/thunder-values.yaml` and `manifests/register-backstage-client.yaml` — rotate these if the repo is ever made public.
