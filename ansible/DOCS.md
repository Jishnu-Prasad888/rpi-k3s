# Ansible — Contents & File Reference

This is the contents page for the `ansible/` directory. It describes each folder, what it contains, and what every file does.

All commands below assume you run them from the `ansible/` directory.

---

## Contents

- [Folder overview](#folder-overview)
- [`playbooks/`](#playbooks--ansible-playbooks)
- [`manifests/`](#manifests--kubernetes-manifests)
- [`tests/`](#tests--smoke-tests)
- [`Dokploy/`](#dokploy--docker-swarm-playbooks)
- [Support files (root of `ansible/`)](#support-files-root-of-ansible)

---

## Folder overview

| Folder | What it contains | How you run it |
|--------|------------------|----------------|
| [`playbooks/`](#playbooks--ansible-playbooks) | Ansible playbooks that run **against the Pis** (OS / cluster install, config, diagnostics) | `ansible-playbook -i <inventory> playbooks/<file>.yml` |
| [`manifests/`](#manifests--kubernetes-manifests) | Kubernetes manifests applied **inside the cluster** (HelmChart CRs, Ingresses, certs, secrets, Helm values) | `k3s kubectl apply -f manifests/<file>.yml` (on rpi1) or `helm install` for `thunder-values.yaml` |
| [`tests/`](#tests--smoke-tests) | Smoke-test manifests — verify a component is working, then delete | `k3s kubectl apply -f tests/<file>.yml` → check → `k3s kubectl delete -f tests/<file>.yml` |
| [`Dokploy/`](#dokploy--docker-swarm-playbooks) | Playbooks for the Docker Swarm + Dokploy deployment (uses 100.x Tailscale addresses) | `ansible-playbook -i k3s_inventory.ini Dokploy/<file>.yml` |
| *(root)* | Config + inventories + this doc | — |

Inventories you can pass to playbooks:

| Inventory | Purpose |
|-----------|---------|
| `inventory.ini` | LAN IPs (192.168.0.x), group `[rpi]` |
| `k3s_inventory.ini` | LAN IPs with `[control_plane]` and `[workers]` groups |
| `tailscale_inventory.ini` | Tailscale 100.x fallback, group `[raspberry_pis]` (used by `static_ips.yml`) |
| `tailscale.ini` | Tailscale 100.x inventory with `[control_plane]` / `[workers]` groups (older variant of the above) |

---

## `playbooks/` — Ansible playbooks

Playbooks that configure the Raspberry Pi hosts themselves. They are run with `ansible-playbook` and need root (handled by `ansible.cfg`).

### Bootstrapping (OS / cluster setup)

| File | What it does | Inventory / hosts |
|------|--------------|-------------------|
| `static_ips.yml` | Assigns permanent static IPs (`.101`, `.103`, `.106`, `.104`, `.105`) via NetworkManager. Runs **one node at a time** (`serial: 1`) to avoid address conflicts. | `tailscale_inventory.ini` → `[raspberry_pis]` |
| `enable_cgroups.yml` | Appends `cgroup_enable=memory cgroup_memory=1` to `cmdline.txt` (detects `/boot/firmware` vs `/boot`) and reboots. **Required before K3s.** | `inventory.ini` → `[rpi]` |
| `enable_cgroups_2.yml` | **Older variant** of `enable_cgroups.yml` — only targets `/boot/firmware/cmdline.txt`. Prefer `enable_cgroups.yml`. | `inventory.ini` → `[rpi]` |
| `install_k3s.yml` | Installs the K3s server on rpi1 (with specific flags), then installs agents on all workers and verifies the cluster. Server IP `.101` is hardcoded here. | `k3s_inventory.ini` → `[rpi]` (server on `[control_plane]`) |
| `uninstall_k3s.yml` | Complete teardown: stops `k3s`/`k3s-agent`, removes packages, deletes `/var/lib/rancher/k3s` data, and reboots. | `k3s_inventory.ini` → `[rpi]` |

### Configuration

| File | What it does | Inventory / hosts |
|------|--------------|-------------------|
| `disable_k3s_defaults.yml` | Writes `/etc/rancher/k3s/config.yaml` with `disable: traefik` and `disable: servicelb` so MetalLB + ingress-nginx take over. | `k3s_inventory.ini` → `[control_plane]` |
| `install_helm.yml` | Downloads the official `get-helm-3` script and installs Helm on the control plane. | `k3s_inventory.ini` → `[control_plane]` |
| `install_metallb.yml` | Applies the MetalLB **v0.15.2** native manifests via `k3s kubectl` and waits for the controller rollout. Pool still needs `manifests/metallb-config.yml`. | `k3s_inventory.ini` → `[control_plane]` |

### Diagnostics

| File | What it does | Inventory / hosts |
|------|--------------|-------------------|
| `cluster_health.yml` | Read-only checks across all nodes: hostname, IP, disk, RAM, CPU, temperature, uptime, routing. Safe to run anytime. | `inventory.ini` → `[rpi]` |

---

## `manifests/` — Kubernetes manifests

Applied **inside the cluster** on rpi1 (`k3s kubectl apply -f manifests/<file>.yml`) — or `helm install` for `thunder-values.yaml`. See the recommended apply order in the root README.

### Networking

| File | What it does |
|------|--------------|
| `metallb-config.yml` | MetalLB `IPAddressPool` (`192.168.0.200–220`) + `L2Advertisement`. Required after `playbooks/install_metallb.yml`. |
| `ingress-nginx.yml` | Namespace + HelmChart CR installing ingress-nginx (DaemonSet, default IngressClass). |
| `prometheus-ingress.yml` | Ingress `prometheus.k3s.local` → `kube-prometheus-stack-prometheus:9090`. |
| `grafana-ingress.yml` | Ingress `grafana.k3s.local` → `kube-prometheus-stack-grafana:80`. |
| `minio-ingress.yml` | Ingress `minio.k3s.local` (HTTP). |
| `minio-ingress-tls.yml` | Ingress `minio.k3s.local` with TLS backed by the `minio-tls` certificate. |

### Certificates (cert-manager)

| File | What it does |
|------|--------------|
| `cert-manager.yml` | HelmChart CR installing cert-manager (CRDs enabled). |
| `local-ca.yml` | Self-signed `ClusterIssuer` + `k3s-local-ca` Certificate — the local CA for `*.k3s.local`. |
| `minio-tls.yml` | Certificate for `minio.k3s.local` and `minio-console.k3s.local`, signed by the local CA. |
| `openchoreo-secret.yml` | `selfsigned-bootstrap` ClusterIssuer + `openchoreo-ca` Certificate (ECDSA) for OpenChoreo. |

### Storage

| File | What it does |
|------|--------------|
| `longhorn.yml` | HelmChart CR installing Longhorn; default StorageClass with 2 replicas, `reclaimPolicy: Delete`. |
| `minio-tenant.yml` | MinIO `Tenant` CR: 1 server pool with a Longhorn-backed volume. |
| `minio-operator.yml` | HelmChart CR installing the MinIO operator (prerequisite for the tenant). |

### Monitoring

| File | What it does |
|------|--------------|
| `monitoring.yml` | Namespace + HelmChart CR for `kube-prometheus-stack` (Prometheus 7d retention / 10Gi, Grafana 5Gi). |

### Registry / secrets platform

| File | What it does |
|------|--------------|
| `harbor.yml` | HelmChart CR installing Harbor registry (clusterIP, HTTP) — Longhorn-backed persistence. |
| `openbao-secrets.yml` | ServiceAccount + `ClusterSecretStore` ("default") wiring External Secrets Operator to OpenBao (`openbao.openbao.svc:8200`) via Kubernetes auth. |

### OpenChoreo (Thunder)

| File | What it does |
|------|--------------|
| `thunder-values.yaml` | Helm **values file** for the OpenChoreo Thunder chart (`helm install`), not a kubectl manifest. |
| `register-backstage-client.yaml` | One-shot `Job` in the `thunder` namespace that registers the Backstage OAuth2 client. |

---

## `tests/` — Smoke tests

Quick manifests to prove a component works. Pattern: `kubectl apply` → inspect → `kubectl delete`.

| File | What it verifies |
|------|------------------|
| `metallb-test.yml` | MetalLB assigns an external IP: `nginx` Deployment exposed via a LoadBalancer Service. |
| `ingress-test.yml` | Host-based routing: `nginx` Deployment + Service + Ingress at `test.k3s.local`. |
| `storage-test.yml` | `local-path` storage: PVC + busybox Pod that writes a file. |
| `longhorn-test.yml` | Longhorn storage: 2Gi PVC + busybox Pod that writes a file. |

---

## `Dokploy/` — Docker Swarm playbooks

Dokploy / Docker Swarm deployment on the Pis (Tailscale-based). These run against the same hosts, e.g. `ansible-playbook -i k3s_inventory.ini Dokploy/install-dokploy.yml`.

| File | What it does |
|------|--------------|
| `install-dokploy.yml` | Installs Docker on all Pis, initializes a Docker Swarm on rpi1, joins the workers, then installs Dokploy on rpi1. |
| `install-dokploy-from-scratch.yml` | Clean-slate variant: removes any existing Docker/Dokploy install first, reinstalls Docker, initializes the Swarm **over Tailscale** (uses the 100.x address as advertise/listen), joins workers, and installs Dokploy. |

---

## Support files (root of `ansible/`)

| File | Purpose |
|------|---------|
| `ansible.cfg` | Ansible defaults: default inventory, `rpi` user, `become: true` (sudo, root), JSON fact cache at `.cache/facts`, SSH pipelining. |
| `inventory.ini` | LAN inventory, all 5 Pis in group `[rpi]`. |
| `k3s_inventory.ini` | LAN inventory plus `[control_plane]` (rpi1) and `[workers]` (rpi2–rpi5) groups. |
| `tailscale_inventory.ini` | Tailscale fallback inventory (`100.x.x.x` placeholders), group `[raspberry_pis]` — used by `playbooks/static_ips.yml`. |
| `tailscale.ini` | Tailscale inventory with `[control_plane]` / `[workers]` groups and real 100.x addresses (older variant). |
| `.cache/` | Generated Ansible fact cache — git-ignored, do not commit. |
| `DOCS.md` | This file. |