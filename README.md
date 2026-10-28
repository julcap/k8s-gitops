# k8s-gitops

GitOps repository for a three tier web application (frontend, backend API, PostgreSQL) running on a self built Kubernetes cluster. Git is the single source of truth: ArgoCD watches this repository and keeps the cluster in sync with it.

## The platform

* **Cluster:** kubeadm on Proxmox VMs (1 control plane, 2 workers), built by hand
* **Container runtime:** containerd with the systemd cgroup driver
* **Networking:** Calico (Tigera operator)
* **Storage:** local path provisioner as the default StorageClass
* **Ingress:** Traefik, installed with Helm
* **Images:** private Harbor registry, tags pinned to Git commit SHAs
* **GitOps:** ArgoCD with automated sync, self heal and prune
* **Secrets:** HashiCorp Vault via External Secrets Operator (planned)

## Repository layout

```
k8s-gitops/
├── README.md
└── apps/
    └── julcap/          # the application: database, API, frontend, ingress
```

## How changes flow

1. A change is committed to this repository.
2. ArgoCD detects the new commit and applies it to the cluster.
3. Manual changes made directly in the cluster are reverted automatically (self heal).

## Secrets

No secret values are stored in this repository. Workloads reference Kubernetes Secrets by name only. The plan is to source those Secrets from Vault through External Secrets Operator, so the manifests committed here contain references, never values.

## Roadmap

* [x] Cluster baseline: networking, storage, private registry
* [x] Application running with probes, resource limits and secrets
* [x] External traffic through Traefik and an Ingress
* [ ] GitOps with ArgoCD (app of apps)
* [ ] Secrets from Vault via External Secrets Operator
* [ ] Hardening: NetworkPolicies, non root containers, PodDisruptionBudgets, ArgoCD Projects
