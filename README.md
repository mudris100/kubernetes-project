# Homelab Kubernetes Platform

A fully automated, production-inspired homelab that provisions virtual machines, bootstraps a Kubernetes cluster, and deploys containerized applications using GitOps. This project demonstrates end-to-end DevOps practices across three distinct layers: Infrastructure as Code, Configuration Management, and Cloud-Native GitOps.

---

## Architecture Overview

```
  Internet User
       │
       ▼
┌─────────────────┐
│  Cloudflare CDN │  (DDoS protection, TLS termination)
└────────┬────────┘
         │ encrypted tunnel (no open ports)
         ▼
┌─────────────────────────────────────────────────────────┐
│                 Proxmox VE Host                         │
│                                                         │
│  ┌──────────────┐   ┌─────────────┐   ┌──────────────┐  │
│  │  k8s-master  │   │ k8s-worker1 │   │ k8s-worker2  │  │
│  │192.168.88.210│   │192.168.88.211   │192.168.88.212│  │
│  │   4GB RAM    │   │   2GB RAM   │   │   2GB RAM    │  │
│  └──────┬───────┘   └─────────────┘   └──────────────┘  │
│         │  Kubernetes (kubeadm) + Flannel CNI           │
│         │  MetalLB (L2) + NGINX Ingress                 │
│         │  Argo CD + Sealed Secrets + cloudflared       │
└─────────────────────────────────────────────────────────┘
         ▲
         │ Argo CD polls every ~3 min
┌─────────────────────────────────────────────────────────┐
│                    Fedora Workstation                   │
│             git push  ──►  GitLab Repository            │
└─────────────────────────────────────────────────────────┘
```

**Layer 1 — Provisioning (Terraform):** Spins up VMs on Proxmox VE using Cloud-Init templates with static IP assignment.

**Layer 2 — Configuration (Ansible):** Hardens the OS, installs the container runtime, bootstraps the Kubernetes cluster, installs Argo CD, MetalLB, NGINX Ingress, and Sealed Secrets controller. Ansible runs once — never again.

**Layer 3 — GitOps (Argo CD):** Owns all post-bootstrap cluster state. Watches this Git repository and automatically reconciles any drift between Git and the live cluster.

---

## Tech Stack

| Category | Tool |
|---|---|
| Virtualization | Proxmox VE |
| IaC | Terraform |
| Configuration Management | Ansible + Ansible Vault |
| Orchestration | Kubernetes (kubeadm) |
| Container Runtime | containerd (systemd cgroup) |
| Networking | Flannel CNI, MetalLB (Layer 2) |
| Ingress | NGINX Ingress Controller |
| GitOps | Argo CD |
| Secrets Management | Ansible Vault + Sealed Secrets |
| Public Exposure | Cloudflare Tunnel (cloudflared) |
| Applications | Spring Boot (Java), PostgreSQL, Redis |

---

## Repository Structure

```
.
├── terraform/
│   └── proxmox/              # VM provisioning (Proxmox provider)
│       ├── main.tf
│       ├── variables.tf
│       └── providers.tf
│
├── ansible/                  # One-time cluster bootstrap
│   ├── bootstrap-cluster.yml # OS hardening, containerd, kubeadm
│   ├── deploy-infra.yml      # Argo CD, MetalLB, Ingress, Sealed Secrets, bootstrap secrets
│   ├── deploy-apps.yml       # Blog + cloudflared Argo CD Applications
│   ├── inventory.ini
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── vault.yml         # Ansible Vault (encrypted secrets)
│   └── roles/
│       ├── common/           # OS hardening, dependencies
│       ├── master/           # kubeadm init, CNI setup
│       ├── workers/          # kubeadm join
│       └── argocd/           # Argo CD installation
│
└── k8s/                      # GitOps source of truth (watched by Argo CD)
    ├── apps/
    │   ├── blog/             # Spring Boot blog application
    │   │   ├── blog-deployment.yml
    │   │   ├── blog-secrets.yml      # SealedSecret (safe to commit)
    │   │   ├── blog-service.yml
    │   │   └── ingress.yml
    │   └── cloudflared/      # Cloudflare Tunnel
    │       └── deployment.yml
    ├── infra/                # Cluster infrastructure config
    │   ├── metallb-pool.yml
    │   └── proxy-headers.yml
    └── argocd/               # Argo CD Application manifests
        ├── blog-app.yaml
        ├── cloudflared-app.yml
        ├── infra-app.yaml
        └── sealed-secrets-app.yaml
```

---

## GitOps Workflow

Git is the single source of truth — the cluster state always converges to what is defined in the `k8s/` directory.

```
git push
   │
   ▼
Argo CD detects diff (polls every ~3 min)
   │
   ├── k8s/apps/blog/*        ──► blog-app           (Deployment, Service, Ingress, SealedSecret)
   ├── k8s/apps/cloudflared/* ──► cloudflared-app    (Deployment)
   ├── k8s/infra/*            ──► infra-app           (MetalLB pool, proxy headers)
   └── (Helm registry)        ──► sealed-secrets-app  (Sealed Secrets controller)
```

All Argo CD Applications are configured with automated sync:

```yaml
syncPolicy:
  automated:
    prune: true      # removes resources deleted from Git
    selfHeal: true   # reverts manual kubectl changes automatically
```

**Day-to-day workflow — no Ansible, no kubectl needed:**

1. Update a manifest in `k8s/` (e.g. bump image tag, change replica count)
2. `git push`
3. Argo CD auto-syncs within 3 minutes

**Rolling updates with zero downtime** are configured on the blog deployment:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # spin up 1 extra pod before terminating old
    maxUnavailable: 0  # never terminate a pod until new one is Ready
```

Combined with a `readinessProbe` on `/actuator/health`, the old pod stays alive and serving traffic until the new pod passes its health check — eliminating 502 errors during deployments.

---

## Cloudflare Tunnel

The blog is exposed to the internet via a **Cloudflare Tunnel** — no open ports, no port forwarding, no static public IP needed. The `cloudflared` pod runs inside the cluster and maintains an outbound-only encrypted connection to Cloudflare's edge. All public traffic flows through Cloudflare's CDN before reaching the cluster.

```
Internet User ──► Cloudflare Edge ──► cloudflared pod ──► NGINX Ingress ──► blog-app
```

This means:
- **No ports exposed** on the Proxmox host or router
- **Free TLS** — Cloudflare handles HTTPS termination
- **DDoS protection** included by default

The tunnel token is an external credential issued by Cloudflare's dashboard. It is stored in **Ansible Vault** and injected into the cluster as a Kubernetes Secret during bootstrap — before Argo CD is running. It does not go through Sealed Secrets.

---

## Secrets Management

Two complementary secrets mechanisms are used depending on when the secret is needed:

```
Ansible Vault                         Sealed Secrets
──────────────────────────────────    ──────────────────────────────────
Bootstrap-time secrets that must      Application secrets committed to
exist BEFORE Argo CD can function:    Git and synced BY Argo CD:

  - GitLab SSH key (repo access)        - DB_USER / DB_PASSWORD
  - Cloudflare tunnel token             - REDIS_PASSWORD

Applied once by Ansible.              Encrypted locally with kubeseal,
Never stored in Git.                  safe to commit. Auto-synced.
```

### Ansible Vault secrets (bootstrap)

Stored in `ansible/group_vars/vault.yml`, encrypted with `ansible-vault`. Ansible applies these directly to the cluster as Kubernetes Secrets during `deploy-infra.yml`:

```bash
# Edit vault
ansible-vault edit ansible/group_vars/vault.yml

# Run playbook with vault password
ansible-playbook -i inventory.ini deploy-infra.yml --ask-vault-pass
```

### Sealed Secrets (application credentials)

Encrypted client-side with the cluster's public key. The encrypted `SealedSecret` is committed to Git — Argo CD syncs it, and the in-cluster controller decrypts it into a standard Kubernetes Secret.

```bash
kubectl create secret generic blog-secrets \
  --from-literal=DB_USER=youruser \
  --from-literal=DB_PASSWORD=yourpassword \
  --from-literal=REDIS_PASSWORD=yourpassword \
  --dry-run=client -o yaml \
| kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > k8s/apps/blog/blog-secrets.yml

git add k8s/apps/blog/blog-secrets.yml
git commit -m "Update blog secrets"
git push
```

> **Important:** Back up the Sealed Secrets controller private key after cluster setup. Without it, sealed secrets cannot be decrypted if the cluster is rebuilt.
> ```bash
> kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key \
>   -o yaml > sealed-secrets-master-key.yaml
> # Store this file somewhere safe — NOT in Git
> ```

---

## Getting Started

### Prerequisites

- Proxmox VE host with a Cloud-Init VM template
- Fedora Workstation (or any Linux machine) with:
  ```bash
  sudo dnf install kubectl ansible terraform
  ```
- `kubeseal` CLI installed:
  ```bash
  curl -OL https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.37.0/kubeseal-0.37.0-linux-amd64.tar.gz
  tar -xvzf kubeseal-0.37.0-linux-amd64.tar.gz kubeseal
  sudo install -m 755 kubeseal /usr/local/bin/kubeseal
  ```

### Step 1 — Provision VMs

```bash
cd terraform/proxmox
terraform init
terraform apply
```

Creates 3 VMs (1 master, 2 workers) with static IPs on Proxmox.

### Step 2 — Bootstrap the Cluster

```bash
cd ansible
ansible-playbook -i inventory.ini bootstrap-cluster.yml
```

Installs containerd, kubeadm, initializes the control plane, joins workers, deploys Flannel CNI.

### Step 3 — Copy kubeconfig locally

```bash
mkdir -p ~/.kube
scp ubuntu@192.168.88.210:/etc/kubernetes/admin.conf ~/.kube/config
kubectl get nodes  # verify
```

### Step 4 — Deploy Infrastructure & Argo CD

```bash
ansible-playbook -i inventory.ini deploy-infra.yml --ask-vault-pass
```

This single command:
- Installs Argo CD
- Installs MetalLB, NGINX Ingress Controller, Sealed Secrets controller
- Injects bootstrap secrets from Ansible Vault (GitLab SSH key, Cloudflare tunnel token)
- Registers `infra-app`, `sealed-secrets-app`, and `cloudflared-app` with Argo CD

After this step Argo CD is running and managing infrastructure. Ansible's job is almost done.

### Step 5 — Create and Commit Application Secrets

```bash
kubectl create secret generic blog-secrets \
  --from-literal=DB_USER=youruser \
  --from-literal=DB_PASSWORD=yourpassword \
  --from-literal=REDIS_PASSWORD=yourpassword \
  --dry-run=client -o yaml \
| kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > k8s/apps/blog/blog-secrets.yml

git add k8s/apps/blog/blog-secrets.yml && git commit -m "Add blog secrets" && git push
```

### Step 6 — Deploy Blog Application

```bash
ansible-playbook -i inventory.ini deploy-apps.yml --ask-vault-pass
```

Registers `blog-app` with Argo CD. From this point on, all blog updates are done via `git push` only.

### Step 7 — Access Argo CD UI

```bash
kubectl get svc argocd-server -n argocd  # get LoadBalancer IP

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## Nodes

| Node | Role | IP | RAM |
|---|---|---|---|
| k8s-master | Control Plane | 192.168.88.210 | 4GB |
| k8s-worker1 | Worker | 192.168.88.211 | 2GB |
| k8s-worker2 | Worker | 192.168.88.212 | 2GB |

---

## Key Design Decisions

**Ansible bootstraps once, Git owns everything after.** Ansible is scoped to cluster initialization only. Once Argo CD is running, all cluster state is managed through Git — including third-party controllers like Sealed Secrets, which Argo CD installs from the official Helm registry.

**Two-phase Ansible deployment.** `deploy-infra.yml` installs the Sealed Secrets controller and injects bootstrap secrets first. Only after application secrets are encrypted and committed to Git does `deploy-apps.yml` register the blog — avoiding a chicken-and-egg problem where the app would crash waiting for secrets that don't exist yet.

**Two secrets mechanisms for two different lifecycles.** Ansible Vault handles bootstrap secrets that must exist before Argo CD starts (GitLab SSH key, Cloudflare tunnel token). Sealed Secrets handles application credentials that live in Git and are synced by Argo CD after bootstrap. Each tool is used where it fits — not one mechanism forced onto everything.

**Cloudflare Tunnel over port forwarding.** The cluster has no open inbound ports. `cloudflared` maintains an outbound-only connection to Cloudflare's edge, which proxies public traffic into the cluster. This eliminates the need for a static public IP, router port forwarding, or a cloud load balancer.

**No files are copied to the master node.** All `kubernetes.core.k8s` tasks read manifests locally on the Ansible controller using `lookup('file', ...)`, keeping cluster nodes stateless with respect to configuration files.

**Versioned image tags over `latest`.** Every Docker image is pushed with a unique tag (e.g. `p1`, `p2`, `p3`). Using `latest` would prevent Argo CD from detecting a diff and auto-syncing on new image pushes.

**`terraform.tfstate` is excluded from Git** — it may contain sensitive infrastructure details and should be stored securely (e.g. Terraform Cloud, S3 backend) in a production environment.

