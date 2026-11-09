# Julcap environments

The manifests in `base/` are the shared Kustomize base. ArgoCD deploys
an overlay for each environment, with independent image tags, replica counts,
ingress hostnames, and API CORS origins.

| Environment | ArgoCD application | Namespace | Hostname | Frontend / API replicas |
| --- | --- | --- | --- | --- |
| Production | `julcap` | `default` | `tools-k8s.julcap.net` | 5 / 1 |
| Staging | `julcap-staging` | `julcap-staging` | `tools-k8s-staging.julcap.net` | 1 / 1 |

Production keeps its existing Application name, Kubernetes resource names,
namespace, and PostgreSQL volume claim. Switching its source to the production
overlay does not move or recreate the database.

Staging has a separate PostgreSQL instance and a new 5Gi volume claim. Service
names resolve within each namespace, so the staging API connects to the staging
database. Namespace separation does not restrict network traffic between
environments; NetworkPolicies are still on the repository roadmap.

## Set up staging

Before merging these manifests into `main` (which triggers automated ArgoCD
sync), prepare staging's secrets:

```sh
kubectl create namespace julcap-staging
```

Create the following Secrets in `julcap-staging` using your secret management
workflow. Do not commit credentials to this repository.

| Secret | Required keys / type |
| --- | --- |
| `julcap-pull` | Registry credentials for `registry.julcap.net`, type `kubernetes.io/dockerconfigjson` |
| `pg-conf` | `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
| `jwt-secret` | `JWT_SECRET` |
| `resend-api-key` | `RESEND_API_KEY` |

Use separate staging credentials, especially the JWT signing key and database
password. The API inherits `EMAIL_PROVIDER=resend` and `EMAIL_FROM=main@julcap.net`;
configure an appropriate staging sender in `overlays/staging/api.yaml` if needed.
The staging API can send real email with the configured Resend credentials.

Point `tools-k8s-staging.julcap.net` at the same Traefik entry point as production
and configure HTTPS in the existing TLS setup. These manifests retain the current
Ingress setup and do not provision certificates. If you choose another hostname,
update both its ingress patch in `overlays/staging/kustomization.yaml` and its
`CORS_ORIGIN` in `overlays/staging/api.yaml`.

ArgoCD's root Application discovers `bootstrap/apps/julcap-staging.yaml` and
creates the staging Application automatically. `CreateNamespace=true` also allows
ArgoCD to create the namespace, but the pods cannot start until the Secrets exist.
Only deploy the overlays; the base is shared configuration.

## Deploy and promote versions

Update `images[].newTag` in the desired environment's `kustomization.yaml`.
Both environments initially use the current production images. To promote a
tested release, copy the frontend and backend tags from staging to production
and commit that change. Changes to base manifests affect both environments.

The frontend must call the API through its environment's `/api` route, or be built
with the matching API URL if that URL is compiled into the image. This repository
does not contain the frontend source or build configuration.

Preview the rendered manifests locally:

```sh
kubectl kustomize apps/julcap/overlays/production
kubectl kustomize apps/julcap/overlays/staging
```

## Add another environment

Copy `overlays/staging` to a new overlay directory, then change its namespace,
hostname, CORS origin, replica counts, and image tags. Add a corresponding ArgoCD
Application under `bootstrap/apps` with a unique Application name, the new overlay
path, and a matching destination namespace. Provision its Secrets and DNS as above.
