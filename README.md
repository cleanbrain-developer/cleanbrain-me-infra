# cleanbrain-me-infra

Infrastructure repository for `cleanbrain.me`.

This repository owns the Kubernetes manifests and deployment documentation for services running on the personal Hetzner K3s cluster.

Application source code lives in separate repositories and is not mixed into this repository.

Current application repositories:

- [`english-core-speaking`](https://github.com/cleanbrain-developer/english-core-speaking)
- [`kioti-crm-discount-enhance-demo`](https://github.com/cleanbrain-developer/kioti-crm-discount-enhance-demo) (private repository)
- [`cleanbrain-me-entrance`](https://github.com/cleanbrain-developer/cleanbrain-me-entrance)

---

## Architecture

```text
Internet
  |
  v
Cloudflare DNS
  - DNS Only
  - no Cloudflare proxy
  |
  v
Hetzner VM
  - 2 vCPU
  - 4 GB RAM
  - 40 GB local disk
  |
  v
K3s
  |
  v
Traefik
  |
  v
cleanbrain-me-gateway
  namespace: cleanbrain-me-system
  |
  v
HTTPRoute
  application namespace
  |
  +--> Kubernetes Service
  |      |
  |      v
  |     Pod
  |
  +--> Kubernetes Service
         |
         v
        Pod
```

### Current platform

| Component             | Current implementation                  |
| --------------------- | --------------------------------------- |
| Cloud                 | Hetzner Cloud                           |
| Kubernetes            | Single-node K3s                         |
| Container runtime     | containerd                              |
| External DNS          | Cloudflare DNS Only                     |
| Traffic entry point   | Traefik                                 |
| Kubernetes routing    | Gateway API                             |
| TLS automation        | cert-manager                            |
| Certificate authority | Let's Encrypt                           |
| Storage               | K3s `local-path`                        |
| Container registry    | GitHub Container Registry (GHCR)        |
| Current CD model      | GitHub Actions → SSH → scoped `kubectl` |
| Planned CD model      | ArgoCD / GitOps                         |

The currently observed K3s version is:

```text
v1.36.3+k3s1
```

---

## Networking

### Traefik

Traefik is bundled with K3s and has Kubernetes Gateway API support enabled.

Its internal container entry points are:

```text
HTTP:  8000
HTTPS: 8443
```

The Traefik Kubernetes Service exposes them externally as:

```text
80  -> 8000
443 -> 8443
```

External clients therefore continue to use normal HTTP/HTTPS ports:

```text
http://<hostname>:80
https://<hostname>:443
```

The internal Gateway listeners match Traefik's entry-point ports.

### Gateway

Shared infrastructure Gateway:

```text
Name:
  cleanbrain-me-gateway

Namespace:
  cleanbrain-me-system

GatewayClass:
  traefik

Listeners:
  HTTP  8000
  HTTPS 8443
```

Applications must **not recreate the Gateway**.

Each application creates its own `HTTPRoute` and attaches it to the shared Gateway using a cross-namespace `parentRef`.

The shared Gateway allows application routes from other namespaces.

---

## TLS

TLS is handled by:

```text
cert-manager
  |
  v
Let's Encrypt
```

Existing ClusterIssuers:

```text
cleanbrain-me-letsencrypt-staging
cleanbrain-me-letsencrypt-prod
```

Production services use:

```text
cleanbrain-me-letsencrypt-prod
```

For `english-core-speaking`:

```text
Hostname:
  english-core-speaking.cleanbrain.me

TLS Secret:
  cleanbrain-me-english-core-speaking-tls

TLS Secret Namespace:
  cleanbrain-me-system
```

For `kioti-crm-discount`:

```text
Hostname:
  crm-discount.kioti.cleanbrain.me

TLS Secret:
  cleanbrain-me-kioti-crm-discount-tls

TLS Secret Namespace:
  cleanbrain-me-system
```

For `cleanbrain-me-entrance` (**not yet issued**):

```text
Hostname:
  cleanbrain.me

TLS Secret:
  cleanbrain-me-entrance-tls  (planned name, following the <app>-tls convention)

TLS Secret Namespace:
  cleanbrain-me-system
```

This would be the first certificate for the bare apex domain rather than a
subdomain. Given "ACME HTTP-01 implementation" below (per-hostname
`Certificate` objects driving a Traefik Ingress HTTP-01 solver, not a
Gateway-API-native cert-manager integration), the existing two TLS Secrets
were most likely produced by a `cert-manager.io/v1` `Certificate` object
per hostname rather than anything automatic tied to the Gateway. Before
applying `kubernetes/apps/entrance/httproute.yaml`, as cluster administrator:

1. Inspect what actually produced the existing certs, to confirm the
   pattern instead of assuming it:

   ```bash
   kubectl get certificate -n cleanbrain-me-system
   kubectl get certificate -n cleanbrain-me-system \
     -o yaml <existing-english-core-speaking-or-kioti-cert-name>
   kubectl get gateway cleanbrain-me-gateway -n cleanbrain-me-system -o yaml
   ```

   The Gateway output matters specifically for its HTTPS listener's
   `tls.certificateRefs`: if it's a single-secret-per-listener design, a
   new listener is needed for `cleanbrain.me`; if one listener already
   lists multiple `certificateRefs` (SNI-multiplexed), the new secret only
   needs to be appended to that list. This repository has no tracked
   Gateway manifest to check against -- it must be read from the live
   cluster.

2. Request the certificate, mirroring the existing `Certificate` object's
   `issuerRef`/spec shape found in step 1 (adjust below if it differs):

   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: cleanbrain-me-entrance
     namespace: cleanbrain-me-system
   spec:
     secretName: cleanbrain-me-entrance-tls
     issuerRef:
       name: cleanbrain-me-letsencrypt-prod
       kind: ClusterIssuer
     dnsNames:
       - cleanbrain.me
   EOF

   kubectl get certificate cleanbrain-me-entrance \
     -n cleanbrain-me-system -w
   ```

   Wait for `READY: True` before proceeding -- HTTP-01 issuance requires
   the apex A record to already resolve to this server, which is confirmed
   (see "DNS" above).

3. If step 1 showed the Gateway needs a new listener (rather than an
   append to an existing `certificateRefs` list), add one for
   `cleanbrain.me` referencing `cleanbrain-me-entrance-tls`, matching the
   existing listeners' `port`/`protocol`/`allowedRoutes` shape. This file
   is not tracked in this repository (see "Networking" > "Gateway" above),
   so this is a direct `kubectl edit`/`kubectl apply` against the live
   Gateway object, not a change to a manifest here.

4. Once the Secret exists and the Gateway can serve it, apply
   `kubernetes/apps/entrance/httproute.yaml` and verify with `curl -I
   https://cleanbrain.me`.

This runbook is inferred from this repository's documented pattern, not
independently verified against the live cluster -- correct it once step 1's
actual output is known, and consider committing a tracked `Certificate`
manifest afterward if that turns out to be this repo's actual mechanism
(unlike the Gateway/ClusterIssuer, a per-app `Certificate` object plausibly
belongs alongside each app's other manifests rather than as unmanaged
cluster bootstrap).

Both existing certificates are issued the same way, per-hostname via HTTP-01 -- no
wildcard certificate (which would require a DNS-01 solver and a Cloudflare
API token) has been introduced. See "Naming conventions" below for how
`kioti.cleanbrain.me` is used as a namespace for multiple future services
without adding that complexity.

The production certificates are already issued successfully.

### ACME HTTP-01 implementation

Application traffic uses **Gateway API**.

However, cert-manager's HTTP-01 ACME challenge currently uses the Traefik **Ingress solver**.

During certificate issuance or renewal, cert-manager may temporarily create resources such as:

```text
cm-acme-http-solver-*
```

including an Ingress and solver Pod/Service.

These temporary Ingress resources are only part of certificate validation.

Application traffic continues to use:

```text
Traefik
  -> Gateway API
  -> HTTPRoute
  -> Service
```

---

## DNS

The domain is managed by Cloudflare.

Cloudflare Proxy is disabled.

```text
Cloudflare
  Proxy: DNS Only
      |
      v
Hetzner public IP
```

Current application hostnames:

```text
english-core-speaking.cleanbrain.me
crm-discount.kioti.cleanbrain.me
```

`cleanbrain-me-entrance` targets the bare apex hostname `cleanbrain.me` itself, not a subdomain. An A record for the apex, pointing at the Hetzner public IP, already exists (confirmed 2026-09-09) -- unlike the TLS certificate for that hostname, DNS is not a blocker for this service (see "TLS" above for the remaining open item).

`kioti.cleanbrain.me` is a dedicated subdomain namespace for KIOTI-related
test/demo services (see "Naming conventions" below), covered by a single
wildcard DNS record so that adding another `kioti-*` service later needs no
further DNS changes:

```text
*.kioti.cleanbrain.me   A   <Hetzner public IP>   (Proxy: DNS Only)
```

All records ultimately resolve directly to the Hetzner VM.

---

## Naming conventions

Shared infrastructure resources use the `cleanbrain-me-` prefix.

Examples:

```text
cleanbrain-me-system
cleanbrain-me-gateway
cleanbrain-me-letsencrypt-prod
cleanbrain-me-english-core-speaking-tls
```

Each application receives its own namespace.

For example:

```text
cleanbrain-me-english-core-speaking
```

Inside an application namespace, resource names should remain concise because the namespace already provides the application context.

Preferred:

```text
web
api
postgres
app
```

Avoid redundant names such as:

```text
cleanbrain-me-english-core-speaking-web
```

unless there is a specific reason to use them.

### `kioti.cleanbrain.me` subdomain namespace

`kioti.cleanbrain.me` is not a Kubernetes namespace -- it is a DNS-level
grouping for KIOTI-related test/demo services, kept separate from the bare
`cleanbrain.me` hostnames used by general personal projects
(`english-core-speaking.cleanbrain.me`). Each such service still gets its
own Kubernetes namespace following the normal `cleanbrain-me-<service-name>`
rule (e.g. `cleanbrain-me-kioti-crm-discount`) and its own `HTTPRoute`
attached to the one shared Gateway -- only the hostname sits under
`*.kioti.cleanbrain.me` instead of directly under `cleanbrain.me`.

Rationale: a single wildcard Cloudflare DNS record covers every current and
future `kioti-*` service, so adding one is DNS-free; TLS still uses the
existing per-hostname HTTP-01 flow (no wildcard certificate / DNS-01 solver
introduced). This was chosen over registering a separate root domain for
KIOTI work because it needed no new DNS/registrar setup and no change to the
existing Gateway/cert-manager model, at the cost of not being independently
transferable to a third party the way a separate domain would be.

---

# Repository layout

```text
kubernetes/
├── namespaces/
│   ├── cleanbrain-me-english-core-speaking.yaml
│   ├── cleanbrain-me-kioti-crm-discount.yaml
│   └── cleanbrain-me-entrance.yaml
│
└── apps/
    ├── english-core-speaking/
    │   ├── secret.example.yaml
    │   ├── rbac.yaml
    │   ├── httproute.yaml
    │   │
    │   ├── postgres/
    │   │   ├── statefulset.yaml
    │   │   └── service.yaml
    │   │
    │   ├── api/
    │   │   ├── configmap.yaml
    │   │   ├── deployment.yaml
    │   │   └── service.yaml
    │   │
    │   └── web/
    │       ├── deployment.yaml
    │       └── service.yaml
    │
    ├── kioti-crm-discount/
    │   ├── secret.example.yaml
    │   ├── rbac.yaml
    │   ├── pvc.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── httproute.yaml
    │
    └── entrance/
        ├── rbac.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── httproute.yaml
```

Plain Kubernetes manifests are used initially.

Kustomize and Helm are intentionally not introduced yet because this is currently a small single-node cluster with one main application.

---

# Applications

## english-core-speaking

Source repository:

[`cleanbrain-developer/english-core-speaking`](https://github.com/cleanbrain-developer/english-core-speaking)

### Runtime

```text
Namespace:
  cleanbrain-me-english-core-speaking

Hostname:
  english-core-speaking.cleanbrain.me
```

Application components:

```text
web
api
postgres
```

Routing:

```text
/api/*  -> api:3000
/*      -> web:80
```

Container images:

```text
ghcr.io/cleanbrain-developer/english-core-speaking-api
ghcr.io/cleanbrain-developer/english-core-speaking-web
```

Application deployments should use immutable Git commit SHA image tags.

Example:

```text
ghcr.io/cleanbrain-developer/english-core-speaking-api:<git-sha>
```

The `latest` tag may also be published for convenience, but it is not the preferred production deployment reference.

---

## entrance

Source repository:

[`cleanbrain-developer/cleanbrain-me-entrance`](https://github.com/cleanbrain-developer/cleanbrain-me-entrance)

Landing page for the bare `cleanbrain.me` domain: a static service
directory listing the other `cleanbrain.me` services with links out to
each. No backend, no database, no auth -- see that repository's
`docs/product/scope.md`.

### Runtime

```text
Namespace:
  cleanbrain-me-entrance

Hostname:
  cleanbrain.me
```

Application components:

```text
web
```

Routing:

```text
/*  -> web:80
```

Container image:

```text
ghcr.io/cleanbrain-developer/cleanbrain-me-entrance
```

Application deployments should use immutable Git commit SHA image tags.
The `latest` tag is also published for convenience/bootstrap but is not the
preferred production deployment reference, same as the other apps.

### First-time deployment

Prerequisite: push `cleanbrain-me-entrance`'s `main` branch at least once
first, so `deployment.yaml`'s `:latest` tag exists in GHCR before bootstrap
(same reasoning as `english-core-speaking` -- see that section above). This
has already been done; the first Actions run (`test` + `build-and-push`)
succeeded.

DNS is confirmed (see "DNS" above). **Before running the steps below**,
complete the TLS runbook in "TLS" above -- `cleanbrain-me-entrance-tls`
must exist in `cleanbrain-me-system` and the Gateway must be able to serve
it, or the last `httproute.yaml` step below will apply cleanly but HTTPS
for `cleanbrain.me` won't actually work yet.

## 1. Namespace, RBAC, Deployment, Service, HTTPRoute

```bash
kubectl apply -f \
  kubernetes/namespaces/cleanbrain-me-entrance.yaml

kubectl apply -f \
  kubernetes/apps/entrance/rbac.yaml

kubectl apply -f \
  kubernetes/apps/entrance/deployment.yaml

kubectl -n cleanbrain-me-entrance \
  rollout status deployment/web \
  --timeout=120s

kubectl apply -f \
  kubernetes/apps/entrance/service.yaml

kubectl apply -f \
  kubernetes/apps/entrance/httproute.yaml
```

The GHCR package is public (see "GHCR image pull authentication" below), so
no pull secret is needed here, matching `english-core-speaking`.

## 2. CI ServiceAccount token and deploy-host kubeconfig

This is the third application sharing `/home/deploy/.kube/config` -- follow
"Multi-application kubeconfig on the deploy host" below, not the single-app
numbered steps 1-12 literally (those would overwrite the existing file).
Substituting this app's values from that section's table:

```bash
export DEPLOY_NAMESPACE="cleanbrain-me-entrance"
export DEPLOY_SERVICE_ACCOUNT="ci-deployer"
export DEPLOY_TOKEN_SECRET="ci-deployer-cleanbrain-me-entrance-token"

kubectl get serviceaccount "$DEPLOY_SERVICE_ACCOUNT" -n "$DEPLOY_NAMESPACE"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${DEPLOY_TOKEN_SECRET}
  namespace: ${DEPLOY_NAMESPACE}
  annotations:
    kubernetes.io/service-account.name: ${DEPLOY_SERVICE_ACCOUNT}
type: kubernetes.io/service-account-token
EOF

for i in $(seq 1 30); do
  TOKEN_B64="$(kubectl get secret "$DEPLOY_TOKEN_SECRET" -n "$DEPLOY_NAMESPACE" -o jsonpath='{.data.token}' 2>/dev/null || true)"
  CA_B64="$(kubectl get secret "$DEPLOY_TOKEN_SECRET" -n "$DEPLOY_NAMESPACE" -o jsonpath='{.data.ca\.crt}' 2>/dev/null || true)"
  [ -n "$TOKEN_B64" ] && [ -n "$CA_B64" ] && break
  sleep 1
done
[ -z "$TOKEN_B64" ] || [ -z "$CA_B64" ] && { echo "ServiceAccount token was not populated"; exit 1; }

DEPLOY_TOKEN="$(printf '%s' "$TOKEN_B64" | base64 -d)"
CA_TMP="$(mktemp)"
printf '%s' "$CA_B64" | base64 -d > "$CA_TMP"

# Merges into the EXISTING /home/deploy/.kube/config (run as the deploy
# user, or install with the same ownership afterward) -- does NOT touch
# the cleanbrain-me-k3s cluster entry or current-context, per "Multi-
# application kubeconfig on the deploy host" below.
KUBECONFIG_PATH=/home/deploy/.kube/config

kubectl config set-credentials ci-deployer-cleanbrain-me-entrance \
  --token="$DEPLOY_TOKEN" \
  --kubeconfig="$KUBECONFIG_PATH"

kubectl config set-context ci-deployer-cleanbrain-me-entrance@cleanbrain-me-k3s \
  --cluster=cleanbrain-me-k3s \
  --user=ci-deployer-cleanbrain-me-entrance \
  --namespace="$DEPLOY_NAMESPACE" \
  --kubeconfig="$KUBECONFIG_PATH"

rm -f "$CA_TMP"
unset DEPLOY_TOKEN TOKEN_B64 CA_B64
```

Verify (per steps 7-10 below, with `--context` added):

```bash
sudo -u deploy -H env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth whoami --context=ci-deployer-cleanbrain-me-entrance@cleanbrain-me-k3s
# expect: system:serviceaccount:cleanbrain-me-entrance:ci-deployer

sudo -u deploy -H env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i patch deployment/web -n cleanbrain-me-entrance \
  --context=ci-deployer-cleanbrain-me-entrance@cleanbrain-me-k3s
# expect: yes

sudo -u deploy -H env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i get secrets -n cleanbrain-me-entrance \
  --context=ci-deployer-cleanbrain-me-entrance@cleanbrain-me-k3s
# expect: no
```

## 3. Enable CI deploys

Once the above is verified, set `ENABLE_PRODUCTION_DEPLOY=true` as a
repository variable on `cleanbrain-me-entrance` (Settings -> Secrets and
variables -> Actions -> Variables) and push (or re-run) to trigger a real
deploy through the workflow, confirming `kubectl set image` /
`kubectl rollout status` succeed end to end (step 12 below).

Verify the deployed service the same way as `english-core-speaking`
("Deployment verification" below), substituting the namespace and
hostname; there is no `/api/health` endpoint for this app, so the external
check is just `curl -I https://cleanbrain.me` and a browser load of the
page.

---

## kioti-crm-discount

Source repository:

[`kioti-crm-discount-enhance-demo`](https://github.com/cleanbrain-developer/kioti-crm-discount-enhance-demo) (private)

Internal CRM discount/order tool for KIOTI dealer operations, syncing
Dealer/Product/PaymentTerm/ProgramCode data from a Salesforce org
(`crm-kioti-usa.my.salesforce.com`). Currently a single-page demo/PoC-stage
tool, not a high-traffic production service.

### Runtime

```text
Namespace:
  cleanbrain-me-kioti-crm-discount

Hostname:
  crm-discount.kioti.cleanbrain.me
```

Unlike `english-core-speaking`, this application is a **single container**
with no separate API/web split and no PostgreSQL:

```text
app       -- Next.js app (UI + API routes together)
```

Persistence is a single SQLite file on a PersistentVolumeClaim (`data`,
mounted at `/app/data`), not a database Pod. Because SQLite does not support
concurrent writers across processes, `replicas` must stay at `1` unless the
application moves off SQLite first.

Routing:

```text
/*  -> app:3000
```

Container image:

```text
ghcr.io/cleanbrain-developer/kioti-crm-discount-enhance-demo
```

The `kioti-crm-discount-enhance-demo` GitHub repository is **private**, and
its GHCR packages default to private too -- unlike `english-core-speaking`'s
public packages, this Deployment needs an `imagePullSecrets` entry (see "GHCR
image pull authentication" below). Application deployments should use
immutable Git commit SHA image tags, the same as `english-core-speaking`; a
`latest` tag is also published for convenience/bootstrap but is not the
preferred production deployment reference.

### First-time deployment

Prerequisite: push `kioti-crm-discount-enhance-demo`'s `main` branch at least
once first, so `deployment.yaml`'s `:latest` tag exists in GHCR before
bootstrap (same reasoning as `english-core-speaking` -- see that section
above).

```bash
kubectl apply -f \
  kubernetes/namespaces/cleanbrain-me-kioti-crm-discount.yaml

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/rbac.yaml
```

Create the GHCR pull secret (private packages -- see "GHCR image pull
authentication" below for the exact command), then:

```bash
cp \
  kubernetes/apps/kioti-crm-discount/secret.example.yaml \
  kubernetes/apps/kioti-crm-discount/secret.yaml
# edit secret.yaml with real values, never commit it

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/secret.yaml

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/pvc.yaml

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/deployment.yaml

kubectl -n cleanbrain-me-kioti-crm-discount \
  rollout status deployment/app \
  --timeout=180s

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/service.yaml

kubectl apply -f \
  kubernetes/apps/kioti-crm-discount/httproute.yaml
```

On first start, the container's entrypoint applies the Prisma schema and, if
`/app/data` is empty, seeds the database before the server starts listening
-- expect a slower first rollout than subsequent ones.

Verify the same way as `english-core-speaking` ("Deployment verification"
below), substituting the namespace and hostname.

---

## PostgreSQL

PostgreSQL runs as a single-replica StatefulSet.

```text
postgres StatefulSet
        |
        v
PersistentVolumeClaim
        |
        v
K3s local-path storage
        |
        v
Hetzner local disk
```

Migrating the previous Docker PostgreSQL data is intentionally out of scope.

A fresh database is acceptable.

Persistence **after the Kubernetes migration** is required.

---

# Secrets

Actual production secret values must never be committed.

Template:

```text
kubernetes/apps/english-core-speaking/secret.example.yaml
```

Local production file:

```text
kubernetes/apps/english-core-speaking/secret.yaml
```

`secret.yaml` is gitignored.

Required values currently include:

| Key                    | Purpose                             |
| ---------------------- | ----------------------------------- |
| `POSTGRES_DB`          | PostgreSQL database                 |
| `POSTGRES_USER`        | PostgreSQL user                     |
| `POSTGRES_PASSWORD`    | PostgreSQL password                 |
| `DATABASE_URL`         | API database connection string      |
| `GOOGLE_CLIENT_ID`     | Google OAuth client                 |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret          |
| `GOOGLE_CALLBACK_URL`  | Google OAuth redirect URI           |
| `SESSION_SECRET`       | Application session signing secret  |
| `FRONTEND_ORIGIN`      | CORS and post-login frontend origin |

`DATABASE_URL` must be kept consistent with the PostgreSQL credential values because Kubernetes Secrets do not perform `${VAR}` interpolation.

Recommended session secret generation:

```bash
openssl rand -base64 48
```

### kioti-crm-discount

Template:

```text
kubernetes/apps/kioti-crm-discount/secret.example.yaml
```

Local production file:

```text
kubernetes/apps/kioti-crm-discount/secret.yaml
```

Required values:

| Key                 | Purpose                                     |
| -------------------- | -------------------------------------------- |
| `DATABASE_URL`      | SQLite file path on the mounted PVC (`/app/data/prod.db`) |
| `SF_DOMAIN`         | Salesforce org domain                       |
| `SF_CLIENT_ID`      | Salesforce connected app client ID          |
| `SF_CLIENT_SECRET`  | Salesforce connected app client secret      |
| `SF_USERNAME`       | Salesforce API user                         |
| `SF_PASSWORD`       | Salesforce API user password                |
| `SF_API_VERSION`    | Salesforce REST API version                 |

**Rotate the Salesforce credentials before first production use.** A prior
commit in the application repository (`docker-compose.yml`, now fixed to
read from an untracked `.env` file) hardcoded real values for this org
directly into git history. The repository is private, which limits exposure,
but git history still contains the old values -- treat that client
secret/password as burned and issue new ones from the Salesforce connected
app before relying on this in production.

### Google OAuth

Google Cloud Console must contain the production redirect URI:

```text
https://english-core-speaking.cleanbrain.me/api/auth/google/callback
```

The configured application value must exactly match the authorized redirect URI.

---

# GHCR image pull authentication

## english-core-speaking: public packages

Current policy: both GHCR packages are **public**.

```text
ghcr.io/cleanbrain-developer/english-core-speaking-api
ghcr.io/cleanbrain-developer/english-core-speaking-web
```

K3s pulls them anonymously -- no registry credential, no Kubernetes Secret, and no
`imagePullSecrets` entry on either Deployment. This was confirmed directly from the
`ImagePullBackOff` Kubernetes event seen during first bootstrap (`failed to authorize:
... 403 Forbidden`), which was caused by a stale `imagePullSecrets: [{name: ghcr-pull}]`
reference pointing at a Secret that was never created for packages that don't need one
in the first place -- not by the packages actually being private.

**If a package is ever switched back to private**, anonymous pulls will start failing
with the same `403 Forbidden` shape, and both a pull Secret and an `imagePullSecrets`
reference need to come back (same as the kioti-crm-discount procedure directly below).
Until that happens, don't add either -- an `imagePullSecrets` entry naming a Secret
that doesn't exist blocks pulls outright, public image or not.

## cleanbrain-me-entrance: public packages

Current policy: the GHCR package is **public** (confirmed 2026-09-09).

```text
ghcr.io/cleanbrain-developer/cleanbrain-me-entrance
```

Same as `english-core-speaking`: K3s pulls it anonymously, no registry
credential, no Kubernetes Secret, and no `imagePullSecrets` entry on the
Deployment. `kubernetes/apps/entrance/deployment.yaml` reflects this. If
this package is ever switched back to private, both a pull Secret and an
`imagePullSecrets` reference need to be added, the same as
`kioti-crm-discount` directly below.

## kioti-crm-discount: private package

The `kioti-crm-discount-enhance-demo` source repository is private, and its GHCR
package defaults to private as well:

```text
ghcr.io/cleanbrain-developer/kioti-crm-discount-enhance-demo
```

Unlike `english-core-speaking`, `deployment.yaml` for this app DOES declare
`imagePullSecrets: [{name: ghcr-pull}]`, so the `ghcr-pull` Secret must exist in
`cleanbrain-me-kioti-crm-discount` before the Deployment is applied, or the pod
will `ImagePullBackOff`:

```bash
kubectl -n cleanbrain-me-kioti-crm-discount \
  create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<PAT-with-read:packages-scope>
```

Use a GitHub Personal Access Token scoped to `read:packages` only, not the
`GITHUB_TOKEN` used by the build workflow (that token is short-lived and
scoped to the Actions run, not usable here). If this package is ever made
public, this Secret and the `imagePullSecrets` entry can both be removed,
mirroring `english-core-speaking`'s current setup.

---

# First-time deployment

The first deployment is performed manually using the existing cluster administrator.

The CI deployment identity is **not** responsible for infrastructure bootstrap.

**Prerequisite: push `english-core-speaking`'s `main` branch at least once before
running the steps below.** `api/deployment.yaml` and `web/deployment.yaml` both pin
`:latest`, which only exists in GHCR once `deploy.yml`'s `build-and-push` job has run --
that job runs on every push to `main` regardless of `ENABLE_PRODUCTION_DEPLOY` (only the
separate `deploy`/`kubectl` job is gated by that variable). Bootstrapping against a GHCR
package with no `:latest` tag yet would `ImagePullBackOff`. Confirm the images exist
first: `https://github.com/cleanbrain-developer/english-core-speaking/pkgs/container/english-core-speaking-api`
and `.../english-core-speaking-web`.

## 1. Namespace

```bash
kubectl apply -f \
  kubernetes/namespaces/cleanbrain-me-english-core-speaking.yaml
```

## 2. CI RBAC

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/rbac.yaml
```

This creates the limited `ci-deployer` identity used only for subsequent API/Web image rollouts.

Both GHCR packages are public, so there's no pull secret to create here -- see "GHCR
image pull authentication" above if that ever changes.

## 3. Application Secret

```bash
cp \
  kubernetes/apps/english-core-speaking/secret.example.yaml \
  kubernetes/apps/english-core-speaking/secret.yaml
```

Edit:

```text
kubernetes/apps/english-core-speaking/secret.yaml
```

with the actual production values.

Then:

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/secret.yaml
```

Never commit this file.

## 4. PostgreSQL

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/postgres/
```

Wait for PostgreSQL:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  wait \
  --for=condition=ready \
  pod \
  -l app=postgres \
  --timeout=120s
```

## 5. API

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/api/
```

Wait for rollout:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/api \
  --timeout=120s
```

The current API image applies the application's Prisma migrations and initialization logic during startup.

## 6. Web

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/web/
```

Wait for rollout:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/web \
  --timeout=120s
```

## 7. HTTPRoute

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/httproute.yaml
```

This route attaches to the existing:

```text
cleanbrain-me-system / cleanbrain-me-gateway
```

It does not recreate or modify:

```text
Gateway
ClusterIssuer
cert-manager
TLS Secret
```

---

# Deployment verification

## Kubernetes resources

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  get pods,svc
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  get httproute
```

Inspect HTTPRoute status:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  describe httproute
```

Expected Gateway API conditions include successful route attachment such as:

```text
Accepted=True
ResolvedRefs=True
```

## API health check

Before relying on external DNS/routing:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  port-forward svc/api 3000:3000
```

From another shell:

```bash
curl http://localhost:3000/api/health
```

## External checks

```bash
curl -I \
  https://english-core-speaking.cleanbrain.me
```

```bash
curl \
  https://english-core-speaking.cleanbrain.me/api/health
```

Then verify the service from a real browser:

```text
https://english-core-speaking.cleanbrain.me
```

Test at least:

- page loading;
- API health;
- Google OAuth login;
- authenticated session;
- study queue behavior.

---

# Migration from the old Docker deployment

Do not stop the previous Docker Compose deployment until the Kubernetes version has been verified end to end.

Migration sequence:

```text
1. Build and publish Kubernetes-targeted images
2. Deploy PostgreSQL
3. Deploy API
4. Deploy Web
5. Deploy HTTPRoute
6. Verify Kubernetes internally
7. Verify the new hostname externally
8. Verify Google OAuth
9. Verify the real application workflow
10. Stop the old Docker Compose deployment
11. Remove obsolete Caddy configuration/container
12. Close the old exposed port 3000
```

Do not delete `/opt` or the existing deployment directories before cutover is complete.

Caddy does not exist in the final Kubernetes architecture.

---

# CI/CD

Application CI/CD lives in each application repository, not in this one:

```text
english-core-speaking/.github/workflows/deploy.yml
kioti-crm-discount-enhance-demo/.github/workflows/deploy.yml
```

Both follow the same model (test -> build/push to GHCR -> SSH -> `kubectl set
image` -> `kubectl rollout status`), each serialized under its own
`concurrency.group` and gated by its own repo's `ENABLE_PRODUCTION_DEPLOY`
variable so the two deploy independently. `kioti-crm-discount-enhance-demo`'s
workflow additionally has no `test` job with real checks yet (the app repo
currently has no `test`/`typecheck` npm scripts) -- its `test` job runs
`lint` and `build` only, which still catches type errors since `next build`
type-checks. Add real tests to that repo and wire them in when they exist.

Current pipeline:

```text
Push to main
   |
   v
Tests
   |
   v
Build Web/API images
   |
   v
Push immutable SHA-tagged images to GHCR
   |
   +--> [ENABLE_PRODUCTION_DEPLOY != "true": stop here, job skipped]
   |
   v
SSH to Hetzner as deploy (known_hosts-verified)
   |
   v
kubectl set image
   |
   v
kubectl rollout status
```

The workflow also publishes `latest` for convenience, but production deployment uses the Git commit SHA.

### Bootstrap-safety gate

The `api`/`web` Deployments don't exist until the manual "First-time deployment" steps
above have been run. `kubectl set image deployment/api ...` against a Deployment that
doesn't exist yet fails outright, so the first few pushes to `main` must not attempt the
SSH/`kubectl` deploy job at all.

The `deploy` job in `deploy.yml` is gated on a **repository variable** (not secret):

```text
ENABLE_PRODUCTION_DEPLOY = "true"
```

Left unset (its state before bootstrap), the `deploy` job is skipped -- shown as skipped
in the Actions UI, not failed -- while `test` and `build-and-push` still run normally on
every push to `main`, so images are already built and waiting in GHCR by the time
bootstrap happens. Once "First-time deployment" above has been completed and verified,
set the repository variable to `"true"` (repo Settings -> Secrets and variables ->
Actions -> Variables tab -- distinct from the Secrets tab) and subsequent pushes deploy
normally. This was chosen over a `workflow_dispatch`-only gate because it keeps the
normal push-to-deploy flow as the steady state without a manual trigger every time, and
over a separate bootstrap workflow file because a single boolean condition is the
smallest change that expresses "block one job until an admin flips a switch."

---

## Required GitHub Actions configuration

Configure these in the `english-core-speaking` repository. The same secret
names and variable are required in `kioti-crm-discount-enhance-demo` too --
`HETZNER_SSH_*` values can be the same (same server, same `deploy` Linux
user), but `ENABLE_PRODUCTION_DEPLOY` is a separate per-repo variable and
must be set independently once that app's bootstrap is verified.

### Secrets (Settings -> Secrets and variables -> Actions -> Secrets)

| Secret                    | Purpose                                 |
| ------------------------- | --------------------------------------- |
| `HETZNER_SSH_HOST`        | Hetzner server hostname/IP              |
| `HETZNER_SSH_USER`        | Dedicated `deploy` Linux user           |
| `HETZNER_SSH_PRIVATE_KEY` | Dedicated CI deployment SSH private key |
| `HETZNER_SSH_PORT`        | Optional SSH port; defaults to 22       |
| `HETZNER_SSH_KNOWN_HOSTS` | Output of `ssh-keyscan -p <port> -H <host>`, captured once so the workflow can verify the host key (see below) instead of trusting it blindly |

### Variables (Settings -> Secrets and variables -> Actions -> Variables)

| Variable                   | Purpose                                                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `ENABLE_PRODUCTION_DEPLOY` | `"true"` to allow the `deploy` job to run; unset/anything else skips it (see "Bootstrap-safety gate" above) |

`GITHUB_TOKEN` is automatically provided by GitHub Actions and is used for GHCR publishing.

### Capturing `HETZNER_SSH_KNOWN_HOSTS`

**`ssh-keyscan`'s output must not be trusted on its own.** It just records whatever host
key the server presents over the network at that moment -- the same trust-on-first-use
gap `StrictHostKeyChecking=yes` is meant to close. Treat it as unverified until you've
independently confirmed the fingerprint from the server itself, over a channel that
doesn't depend on the network path `ssh-keyscan` used (e.g. the Hetzner Cloud Console's
web-based VNC/serial console, not another SSH session to the same host/IP).

1. Capture the candidate key:

   ```bash
   ssh-keyscan -p <port> -H <host> > hetzner-known-hosts.txt
   ssh-keygen -lf hetzner-known-hosts.txt
   ```

   This prints a fingerprint per host-key algorithm the server offered, e.g.:

   ```text
   256 SHA256:AbCdEf...  <host> (ED25519)
   ```

2. Independently obtain the server's actual host key fingerprint by logging into the
   Hetzner Cloud Console (web console, not SSH) and running, on the server itself:

   ```bash
   ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   # if you need to cross-check other algorithms too:
   ssh-keygen -lf /etc/ssh/ssh_host_ecdsa_key.pub
   ssh-keygen -lf /etc/ssh/ssh_host_rsa_key.pub
   ```

3. Compare the two fingerprints (algorithm and the `SHA256:...` value) character for
   character. **Only if they match** does `hetzner-known-hosts.txt` represent the real
   server and get pasted into the `HETZNER_SSH_KNOWN_HOSTS` secret. If they don't match,
   stop -- do not save the secret -- and investigate (wrong host/port, DNS pointing
   somewhere unexpected, or an actual MITM) before proceeding.

Paste the full, verified contents of `hetzner-known-hosts.txt` (one or more `-H`
hashed-hostname lines) into the `HETZNER_SSH_KNOWN_HOSTS` secret, then delete the local
file -- it's not sensitive on its own (it's a public key), but there's no reason to
leave it lying around.
`deploy.yml` writes it to the runner's `~/.ssh/known_hosts` and connects with
`StrictHostKeyChecking=yes`, so an unrecognized or changed host key fails the deploy
loudly instead of silently trusting whatever's presented (which is what
`StrictHostKeyChecking=no` -- or an action that defaults to it -- would do). If the
server is ever rebuilt or its host key rotated, re-run the capture and update the
secret; a failed connection with a `REMOTE HOST IDENTIFICATION HAS CHANGED` error is the
expected symptom otherwise, not a bug in the workflow.

The build/publish job should have only the permissions it requires, including:

```yaml
permissions:
  contents: read
  packages: write
```

Production deployment should run only from the intended production branch.

Overlapping production deployments must be serialized using GitHub Actions concurrency.

---

# Current CD model and configuration drift

The current SSH deployment bridge executes commands equivalent to:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  set image deployment/api \
  api=ghcr.io/cleanbrain-developer/english-core-speaking-api:<sha>
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/api \
  --timeout=120s
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  set image deployment/web \
  web=ghcr.io/cleanbrain-developer/english-core-speaking-web:<sha>
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/web \
  --timeout=120s
```

This is intentionally an interim deployment model.

`kubectl set image` modifies live Kubernetes state but does not update the image tag committed in `cleanbrain-me-infra`.

Therefore the following can temporarily diverge:

```text
Git manifest image
!=
currently running image
```

Until ArgoCD is introduced:

- `cleanbrain-me-infra` is the source of truth for base Kubernetes configuration;
- the live Deployment is the source of truth for the currently deployed SHA;
- image-version reconciliation is manual.

Do not blindly re-apply an older Deployment manifest after CI has deployed a newer image, because it may roll the application back to the image tag stored in the manifest.

This limitation disappears when the deployment process moves to GitOps.

---

# CI deployment identity

The GitHub Actions deployment job must never use:

```text
/etc/rancher/k3s/k3s.yaml
```

and must never use:

```text
cluster-admin
system:masters
```

Two independent credentials exist:

```text
SSH credential
  -> authenticates the Linux deploy user

Kubernetes credential
  -> authenticates ci-deployer
```

The SSH credential alone does not grant Kubernetes API privileges.

The Kubernetes identity is:

```text
system:serviceaccount:
cleanbrain-me-english-core-speaking:
ci-deployer
```

Its permissions come from:

```text
kubernetes/apps/english-core-speaking/rbac.yaml
```

---

## RBAC model

The deployment RBAC uses:

```text
ServiceAccount
Role
RoleBinding
```

inside:

```text
cleanbrain-me-english-core-speaking
```

It does not use a `ClusterRole` or `ClusterRoleBinding`.

### Allowed

```text
Deployment/api
  get
  patch

Deployment/web
  get
  patch

Deployments in the application namespace
  list
  watch
```

### Explicitly not granted

```text
Secrets
ConfigMaps
Pods
Pod logs
ReplicaSets
StatefulSets
PVCs
RBAC resources
Namespaces
Gateway resources
cert-manager resources
other namespaces
create operations
delete operations
arbitrary update operations
```

The CI identity cannot modify its own RBAC.

---

# One-time server setup for the deploy user

The `deploy` Linux account exists only as a temporary SSH-based CD bridge before ArgoCD is introduced.

Even if the deployment SSH credential is compromised, Kubernetes API access through the scoped kubeconfig remains limited to what the `ci-deployer` Role allows.

This does **not** mean compromise of the Linux account itself is harmless.

The OS account must also be hardened independently:

```text
no sudo
no Docker group
key-only SSH
restricted file permissions
scoped Kubernetes kubeconfig
```

---

## Token model

Kubernetes recommends TokenRequest-based short-lived ServiceAccount tokens over persistent manually managed ServiceAccount token Secrets.

This setup deliberately uses a persistent ServiceAccount token as a **temporary bridge** because:

1. this is a single personal K3s cluster;
2. the token remains stored only on the Hetzner host;
3. namespace-scoped least-privilege RBAC limits Kubernetes authorization;
4. the complete SSH deployment mechanism will be removed once ArgoCD is introduced.

The token must never be stored in GitHub Secrets or committed to Git.

---

## 1. Apply CI RBAC

From the infrastructure repository:

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/rbac.yaml
```

Verify:

```bash
kubectl get serviceaccount,role,rolebinding \
  -n cleanbrain-me-english-core-speaking
```

Expected objects include:

```text
ci-deployer
```

Do not proceed if the manifest creates a `ClusterRole`, `ClusterRoleBinding`, or grants `cluster-admin`.

---

## 2. Create the Linux deploy user

Create the account if it does not already exist:

```bash
if id deploy >/dev/null 2>&1; then
  echo "deploy user already exists"
else
  useradd \
    --create-home \
    --shell /bin/bash \
    deploy
fi
```

Disable password authentication for the account:

```bash
passwd -l deploy
```

Verify groups:

```bash
id deploy
groups deploy
```

The account must not belong to privileged groups such as:

```text
sudo
wheel
docker
```

---

## 3. Install the CI SSH public key

Create:

```bash
install \
  -d \
  -m 700 \
  -o deploy \
  -g deploy \
  /home/deploy/.ssh
```

Read the dedicated deployment public key:

```bash
read -r -p \
  "Paste the deploy SSH public key: " \
  DEPLOY_SSH_PUBLIC_KEY
```

Install it:

```bash
printf '%s\n' "$DEPLOY_SSH_PUBLIC_KEY" \
  > /home/deploy/.ssh/authorized_keys
```

Set ownership and permissions:

```bash
chown deploy:deploy \
  /home/deploy/.ssh/authorized_keys

chmod 600 \
  /home/deploy/.ssh/authorized_keys
```

Remove the shell variable:

```bash
unset DEPLOY_SSH_PUBLIC_KEY
```

Verify:

```bash
stat \
  -c '%U:%G %a %n' \
  /home/deploy/.ssh \
  /home/deploy/.ssh/authorized_keys
```

Expected:

```text
deploy:deploy 700 /home/deploy/.ssh
deploy:deploy 600 /home/deploy/.ssh/authorized_keys
```

The corresponding private key is stored only in:

```text
HETZNER_SSH_PRIVATE_KEY
```

in GitHub Actions.

---

## 4. Create the temporary ServiceAccount token

Set variables:

```bash
export DEPLOY_NAMESPACE="cleanbrain-me-english-core-speaking"
export DEPLOY_SERVICE_ACCOUNT="ci-deployer"
export DEPLOY_TOKEN_SECRET="ci-deployer-token"
```

Verify the ServiceAccount:

```bash
kubectl get serviceaccount \
  "$DEPLOY_SERVICE_ACCOUNT" \
  -n "$DEPLOY_NAMESPACE"
```

Create the persistent token Secret:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${DEPLOY_TOKEN_SECRET}
  namespace: ${DEPLOY_NAMESPACE}
  annotations:
    kubernetes.io/service-account.name: ${DEPLOY_SERVICE_ACCOUNT}
type: kubernetes.io/service-account-token
EOF
```

Wait for Kubernetes to populate it:

```bash
for i in $(seq 1 30); do
  TOKEN_B64="$(
    kubectl get secret "$DEPLOY_TOKEN_SECRET" \
      -n "$DEPLOY_NAMESPACE" \
      -o jsonpath='{.data.token}' \
      2>/dev/null || true
  )"

  CA_B64="$(
    kubectl get secret "$DEPLOY_TOKEN_SECRET" \
      -n "$DEPLOY_NAMESPACE" \
      -o jsonpath='{.data.ca\.crt}' \
      2>/dev/null || true
  )"

  if [ -n "$TOKEN_B64" ] && [ -n "$CA_B64" ]; then
    break
  fi

  sleep 1
done
```

Validate:

```bash
if [ -z "$TOKEN_B64" ] || [ -z "$CA_B64" ]; then
  echo "ServiceAccount token was not populated"
  exit 1
fi
```

Do not manually add this Secret to the ServiceAccount's `secrets:` field.

---

## 5. Build the scoped kubeconfig

Create temporary files:

```bash
KUBECONFIG_TMP="$(mktemp)"
CA_TMP="$(mktemp)"
```

Decode:

```bash
DEPLOY_TOKEN="$(
  printf '%s' "$TOKEN_B64" |
  base64 -d
)"
```

```bash
printf '%s' "$CA_B64" |
  base64 -d \
  > "$CA_TMP"
```

Configure the local K3s API endpoint:

```bash
kubectl config set-cluster cleanbrain-me-k3s \
  --server=https://127.0.0.1:6443 \
  --certificate-authority="$CA_TMP" \
  --embed-certs=true \
  --kubeconfig="$KUBECONFIG_TMP"
```

Configure the ServiceAccount identity:

```bash
kubectl config set-credentials ci-deployer \
  --token="$DEPLOY_TOKEN" \
  --kubeconfig="$KUBECONFIG_TMP"
```

Configure the context:

```bash
kubectl config set-context ci-deployer@cleanbrain-me-k3s \
  --cluster=cleanbrain-me-k3s \
  --user=ci-deployer \
  --namespace="$DEPLOY_NAMESPACE" \
  --kubeconfig="$KUBECONFIG_TMP"
```

Activate it:

```bash
kubectl config use-context \
  ci-deployer@cleanbrain-me-k3s \
  --kubeconfig="$KUBECONFIG_TMP"
```

---

## 6. Install the deploy kubeconfig

Create:

```bash
install \
  -d \
  -m 700 \
  -o deploy \
  -g deploy \
  /home/deploy/.kube
```

Install:

```bash
install \
  -m 600 \
  -o deploy \
  -g deploy \
  "$KUBECONFIG_TMP" \
  /home/deploy/.kube/config
```

Remove temporary credentials:

```bash
rm -f \
  "$KUBECONFIG_TMP" \
  "$CA_TMP"

unset DEPLOY_TOKEN
unset TOKEN_B64
unset CA_B64
```

Verify:

```bash
stat \
  -c '%U:%G %a %n' \
  /home/deploy/.kube \
  /home/deploy/.kube/config
```

Expected:

```text
deploy:deploy 700 /home/deploy/.kube
deploy:deploy 600 /home/deploy/.kube/config
```

---

## 7. Verify kubeconfig contents

The deploy kubeconfig must not contain admin client credentials.

```bash
if grep -Eq \
  'client-certificate|client-key' \
  /home/deploy/.kube/config; then

  echo "ERROR: client certificate/key found"
  exit 1
else
  echo "OK: token-only kubeconfig"
fi
```

Inspect users:

```bash
sudo -u deploy -H \
  kubectl config get-users \
  --kubeconfig=/home/deploy/.kube/config
```

Inspect contexts:

```bash
sudo -u deploy -H \
  kubectl config get-contexts \
  --kubeconfig=/home/deploy/.kube/config
```

Only the deployment identity/context should exist.

---

## 8. Verify Kubernetes identity

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth whoami
```

Expected identity:

```text
system:serviceaccount:cleanbrain-me-english-core-speaking:ci-deployer
```

---

## 9. Verify allowed operations

All of these must return:

```text
yes
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  get deployment/api \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  patch deployment/api \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  get deployment/web \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  patch deployment/web \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  list deployments \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  watch deployments \
  -n cleanbrain-me-english-core-speaking
```

---

## 10. Verify denied operations

All of these must return:

```text
no
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  get secrets \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  get pods \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  get replicasets \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  patch statefulset/postgres \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  create deployments \
  -n cleanbrain-me-english-core-speaking
```

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  delete deployment/api \
  -n cleanbrain-me-english-core-speaking
```

Verify cross-namespace denial:

```bash
sudo -u deploy -H \
  env KUBECONFIG=/home/deploy/.kube/config \
  kubectl auth can-i \
  patch deployments \
  -n cleanbrain-me-system
```

This must return:

```text
no
```

Do not enable CI deployment if any denied operation unexpectedly returns `yes`.

---

## 11. CI kubeconfig behavior

The remote CI command must explicitly use:

```bash
export KUBECONFIG=/home/deploy/.kube/config
```

Do not rely on implicit K3s kubeconfig discovery.

Example deployment commands:

```bash
export KUBECONFIG=/home/deploy/.kube/config
```

```bash
kubectl set image \
  deployment/api \
  api=<immutable-api-image> \
  -n cleanbrain-me-english-core-speaking
```

```bash
kubectl rollout status \
  deployment/api \
  -n cleanbrain-me-english-core-speaking \
  --timeout=120s
```

```bash
kubectl set image \
  deployment/web \
  web=<immutable-web-image> \
  -n cleanbrain-me-english-core-speaking
```

```bash
kubectl rollout status \
  deployment/web \
  -n cleanbrain-me-english-core-speaking \
  --timeout=120s
```

---

## 12. Real rollout verification

`kubectl auth can-i` verifies authorization policy but does not prove that the exact deployment workflow works end to end.

After the initial `api` and `web` Deployments exist, perform one real deployment using the GitHub Actions production workflow.

Verify that both:

```text
kubectl set image
kubectl rollout status
```

complete without authorization errors.

If `kubectl rollout status` reports a `Forbidden` response for another resource type, inspect the actual API request before changing RBAC.

Do not preemptively grant:

```text
Pods
ReplicaSets
Pod logs
Secrets
```

unless the workflow demonstrably requires them.

---

# SSH hardening

The `deploy` user must not have:

```text
sudo
Docker group membership
password-based login
```

The Kubernetes kubeconfig must remain:

```text
owner: deploy
mode: 0600
```

Verify:

```bash
id deploy
```

```bash
stat \
  -c '%U:%G %a %n' \
  /home/deploy/.kube/config
```

The GitHub Actions deployment workflow should verify the Hetzner SSH host key.

Do not use:

```text
StrictHostKeyChecking=no
```

as the permanent production configuration.

The private deployment SSH key must never be committed or copied to the Hetzner server.

---

# Credential rotation

The persistent ServiceAccount token is a rotatable temporary deployment credential.

For routine rotation:

1. create a second ServiceAccount token Secret;
2. wait for Kubernetes to populate it;
3. create a replacement kubeconfig using the new token;
4. verify `kubectl auth whoami`;
5. verify `kubectl auth can-i`;
6. atomically replace `/home/deploy/.kube/config`;
7. perform a deployment test;
8. delete the previous token Secret.

Never delete the currently active token before the replacement credential has been verified.

---

## Multi-application kubeconfig on the deploy host

The numbered steps above ("1. Apply CI RBAC" through "11. CI kubeconfig
behavior") were written for a single application and, followed literally a
second time for `kioti-crm-discount`, would **overwrite**
`/home/deploy/.kube/config` and destroy `english-core-speaking`'s working
credential -- step 5 builds a brand-new kubeconfig from scratch into a temp
file, and step 6 installs it by copying over the existing file wholesale.

For a second (and any subsequent) application, instead:

1. Substitute app-specific values throughout:

   | Variable                 | english-core-speaking value | kioti-crm-discount value                    | entrance value |
   | ------------------------- | ---------------------------- | --------------------------------------------- | --------------- |
   | `DEPLOY_NAMESPACE`        | `cleanbrain-me-english-core-speaking` | `cleanbrain-me-kioti-crm-discount` | `cleanbrain-me-entrance` |
   | `DEPLOY_SERVICE_ACCOUNT`  | `ci-deployer`                | `ci-deployer`                                 | `ci-deployer` |
   | `DEPLOY_TOKEN_SECRET`     | `ci-deployer-token`          | `ci-deployer-kioti-crm-discount-token`        | `ci-deployer-cleanbrain-me-entrance-token` |
   | kubeconfig user name      | `ci-deployer`                | `ci-deployer-kioti-crm-discount`              | `ci-deployer-cleanbrain-me-entrance` |
   | kubeconfig context name   | `ci-deployer@cleanbrain-me-k3s` | `ci-deployer-kioti-crm-discount@cleanbrain-me-k3s` | `ci-deployer-cleanbrain-me-entrance@cleanbrain-me-k3s` |

   The entrance value's context name matches the literal string already
   hardcoded in `cleanbrain-me-entrance`'s own `.github/workflows/deploy.yml`
   -- keep the two in sync if either ever changes.

   The token Secret and kubeconfig user/context names must differ per app --
   reusing `ci-deployer-token` or the `ci-deployer` user/context name across
   two ServiceAccounts in different namespaces will silently clobber
   whichever was created second.

2. In step 5 ("Build the scoped kubeconfig"), run the `kubectl config
   set-cluster` / `set-credentials` / `set-context` commands with
   `--kubeconfig=/home/deploy/.kube/config` directly (as the `deploy` user,
   or `install`ed with the same ownership afterward) instead of a fresh
   `$KUBECONFIG_TMP`. This **merges** the new cluster/user/context entries
   into the existing file rather than replacing it -- the `cleanbrain-me-k3s`
   cluster entry already exists from the first app's setup and does not need
   to be redefined, only the new user and context.

3. Do **not** run `kubectl config use-context` for the new context. Changing
   `current-context` would silently break `english-core-speaking`'s deploy,
   which relies on it being `ci-deployer@cleanbrain-me-k3s`. Leave
   `current-context` as-is.

4. Because `current-context` cannot be relied on to select the right
   identity once more than one app shares the file, `kioti-crm-discount`'s
   `deploy.yml` (unlike `english-core-speaking`'s) must pass
   `--context=ci-deployer-kioti-crm-discount@cleanbrain-me-k3s` explicitly on
   every `kubectl` invocation in its remote script, rather than relying on
   the implicit current context.

5. Step 11 ("CI kubeconfig behavior")'s `export KUBECONFIG=...` line still
   applies as-is -- both apps share the one kubeconfig file, distinguished by
   `--context`, not by separate files.

Steps 1-4 (RBAC), 7 (kubeconfig contents), 8 (identity), 9-10 (`can-i`
checks), and 12 (real rollout verification) apply per-app unchanged, just
run again with the substituted values and (for step 8/9/10) the new
`--context` flag added to each `kubectl auth ...` command.

---

# Immediate CI credential revocation

If the Kubernetes deployment credential is suspected to be compromised, remove authorization first:

```bash
kubectl delete rolebinding ci-deployer \
  -n cleanbrain-me-english-core-speaking
```

Then delete the persistent token:

```bash
kubectl delete secret ci-deployer-token \
  -n cleanbrain-me-english-core-speaking
```

If the SSH credential is also compromised, remove its public key from:

```text
/home/deploy/.ssh/authorized_keys
```

Rotate the SSH key pair and update:

```text
HETZNER_SSH_PRIVATE_KEY
```

After remediation, restore the declarative RBAC:

```bash
kubectl apply -f \
  kubernetes/apps/english-core-speaking/rbac.yaml
```

Then issue a new token and rebuild the scoped kubeconfig.

---

# Rollback

Manual rollback:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout undo deployment/api
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout undo deployment/web
```

Or explicitly deploy a known-good immutable image:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  set image deployment/api \
  api=ghcr.io/cleanbrain-developer/english-core-speaking-api:<known-good-sha>
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/api \
  --timeout=120s
```

The same process applies to `web`.

---

# Operational inspection

## Workloads

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  get pods
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  get deployment,statefulset,svc
```

## Logs

Administrator/operator:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  logs deployment/api
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  logs deployment/web
```

The CI `ci-deployer` identity intentionally does not have log access.

## Rollout

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/api
```

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  rollout status deployment/web
```

## Pod details

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  describe pod <pod-name>
```

---

# Resource budget

Hetzner VM:

```text
2 vCPU
4 GB RAM
40 GB local disk
```

Current application resource targets:

| Component                     | CPU request | Memory request | CPU limit | Memory limit |
| ------------------------------ | ----------: | -------------: | --------: | -----------: |
| english-core-speaking/postgres |        250m |          256Mi |     1000m |        512Mi |
| english-core-speaking/api      |        100m |          128Mi |      500m |        256Mi |
| english-core-speaking/web      |         50m |           32Mi |      200m |         64Mi |
| kioti-crm-discount/app         |        150m |          256Mi |      500m |        512Mi |
| entrance/web                   |         50m |           32Mi |      200m |         64Mi |
| **Total**                      |    **600m** |      **704Mi** | **2400m** |   **1408Mi** |

The combined CPU **limit** total (2400m) exceeds the box's 2 vCPU (2000m)
capacity, and by a larger margin than before `entrance/web` was added --
this is the "third similarly-sized service" scenario this section already
warned about. This is still expected and not itself a problem -- limits are
ceilings per Pod, not reservations, and all workloads hitting their limit
simultaneously at once is unlikely for these low-traffic apps -- but there
is now no slack left at all for a further service without raising the VM
spec or tightening existing limits. Requests (600m / 704Mi) are what the
scheduler actually reserves and stay comfortably inside budget. Re-check
with `kubectl top nodes` / `kubectl top pods -A` after `entrance/web`'s
first-time deployment, and before adding another `kioti-*` test service or
any other new service under this same namespace strategy.

This leaves capacity for:

```text
K3s control-plane components
containerd
CoreDNS
Traefik
cert-manager
system workloads
```

Disk usage must also be monitored because container images, logs, PostgreSQL data, and K3s state all share the same 40 GB disk.

Useful checks:

```bash
df -h
```

```bash
free -h
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

---

# Planned GitOps migration

The current SSH deployment mechanism is intentionally temporary.

Target:

```text
Git push
   |
   v
GitHub Actions
   |
   +--> test
   |
   +--> build
   |
   +--> GHCR
   |
   v
GitOps desired state
   |
   v
ArgoCD
   |
   v
K3s
```

Once ArgoCD becomes responsible for deployment reconciliation:

- remove the GitHub Actions SSH deployment job;
- remove `HETZNER_SSH_PRIVATE_KEY`;
- remove the CI deployment SSH public key from the server;
- delete the `ci-deployer` RoleBinding;
- delete the persistent ServiceAccount token Secret;
- remove `/home/deploy/.kube/config`;
- retire the temporary push-based CD bridge.

At that point, Git becomes the authoritative source for both base configuration and deployed image versions.

---

# Explicitly out of scope for the current phase

The following are intentionally deferred:

- ArgoCD / GitOps reconciliation;
- Prometheus;
- Grafana;
- Loki;
- service mesh;
- additional Kubernetes operators;
- multi-node K3s;
- PostgreSQL HA;
- migration of the previous Docker PostgreSQL data;
- Caddy.

Caddy is fully replaced in the target architecture by:

```text
Traefik
Gateway API
cert-manager
Let's Encrypt
```

The immediate priority is to complete and validate the production migration of:

```text
english-core-speaking
```

on the existing lightweight single-node K3s cluster.
