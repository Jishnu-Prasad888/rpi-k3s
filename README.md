# RPi K3s Cluster

Ansible playbooks and Kubernetes manifests for a 5-node Raspberry Pi K3s cluster.

## Nodes

| Node | IP | Role |
|------|----|------|
| rpi1 | 192.168.0.101 | Control plane |
| rpi2 | 192.168.0.103 | Worker |
| rpi3 | 192.168.0.106 | Worker |
| rpi4 | 192.168.0.104 | Worker |
| rpi5 | 192.168.0.105 | Worker |

SSH user: `rpi`. Tailscale fallback inventory lives in `tailscale_inventory.ini` (used when the router / LAN is down).

## Layout

Everything lives in `ansible/`:

- **Inventories** — `inventory.ini` (LAN IPs, groups `rpi`), `k3s_inventory.ini` (adds `control_plane` / `workers` groups), `tailscale_inventory.ini` (VPN fallback).
- **Playbooks** — Ansible-driven setup for the hosts.
- **Kubernetes manifests** — `*.yml` files that are `kubectl apply`'d against the cluster.

## Usage

Run from `ansible/`:

```sh
ansible-playbook -i k3s_inventory.ini <playbook>.yml
```

Apply a manifest on the control plane (rpi1):

```sh
k3s kubectl apply -f <manifest>.yml
```

## Install order

1. `static_ips.yml` — assign permanent static IPs via NetworkManager (needs tailscale inventory).
2. `enable_cgroups.yml` — add `cgroup_enable=memory cgroup_memory=1` to `cmdline.txt` and reboot (required for K3s on Pi).
3. `install_k3s.yml` — bootstrap server on rpi1, join workers, verify.
4. `disable_k3s_defaults.yml` — disable Traefik + ServiceLB (`/etc/rancher/k3s/config.yaml`), restart.
5. `install_helm.yml` — install Helm on the control plane.
6. `install_metallb.yml` + `metallb-config.yml` — MetalLB with L2 pool `192.168.0.200-220`.
7. `ingress-nginx.yml` — nginx ingress controller (LoadBalancer, default class).

## Components

| File | What it does |
|------|--------------|
| `cert-manager.yml` | cert-manager via HelmChart |
| `local-ca.yml` | Self-signed CA + `k3s-local-ca-issuer` ClusterIssuer |
| `monitoring.yml` | kube-prometheus-stack (Prometheus + Grafana, local-path storage) |
| `prometheus-ingress.yml` / `grafana-ingress.yml` | Ingress for `prometheus.k3s.local` / `grafana.k3s.local` |
| `longhorn.yml` | Longhorn storage (default class, 2 replicas) |
| `minio-operator.yml` | MinIO operator |
| `minio-tenant.yml` | MinIO tenant (1 server, 10Gi on Longhorn) |
| `minio-tls.yml` + `minio-ingress-tls.yml` | TLS cert + ingress for `minio.k3s.local` |
| `minio-ingress.yml` | Plain HTTP ingress (pre-TLS) |
| `harbor.yml` | Harbor registry (Longhorn-backed, HTTP clusterIP) |
| `openchoreo-secret.yml` | OpenChoreo CA (self-signed bootstrap) |
| `openbao-secrets.yml` | External Secrets ClusterSecretStore → OpenBao (Vault) |
| `thunder-values.yaml` | Helm values for Thunder (OpenChoreo gate), Postgres-backed |
| `cluster_health.yml` | Reports hostname, IP, connectivity, disk, RAM, CPU, temp, routing for all Pis |
| `uninstall_k3s.yml` | Nuke K3s from all nodes and reboot |

## Smoke tests

- `metallb-test.yml` — nginx LoadBalancer (checks MetalLB assigns an IP).
- `ingress-test.yml` — nginx deployment + ingress at `test.k3s.local`.
- `storage-test.yml` — busybox pod writing to a local-path PVC.
- `longhorn-test.yml` — busybox pod writing to a Longhorn PVC.

## Notes

- Manifests are applied with `kubectl` on rpi1; there's no role/playbook that applies them automatically.
- `enable_cgroups_2.yml` is an older variant of the cgroups playbook; prefer `enable_cgroups.yml`.
- Server IP `192.168.0.101` is hardcoded in `install_k3s.yml`.
