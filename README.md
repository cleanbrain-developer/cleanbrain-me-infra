# cleanbrain-me-infra

Infrastructure repository for `cleanbrain.me`.

This repository owns the Kubernetes manifests and deployment documentation for services running on the personal Hetzner K3s cluster.

Application source code lives in separate repositories and is not mixed into this repository.

Current application repository:

- [`english-core-speaking`](https://github.com/cleanbrain-developer/english-core-speaking)

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

The production certificate is already issued successfully.

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

Current application hostname:

```text
english-core-speaking.cleanbrain.me
```

The DNS record ultimately resolves directly to the Hetzner VM.

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
```

Avoid redundant names such as:

```text
cleanbrain-me-english-core-speaking-web
```

unless there is a specific reason to use them.

---

# Repository layout

```text
kubernetes/
├── namespaces/
│   └── cleanbrain-me-english-core-speaking.yaml
│
└── apps/
    └── english-core-speaking/
        ├── secret.example.yaml
        ├── rbac.yaml
        ├── httproute.yaml
        │
        ├── postgres/
        │   ├── statefulset.yaml
        │   └── service.yaml
        │
        ├── api/
        │   ├── configmap.yaml
        │   ├── deployment.yaml
        │   └── service.yaml
        │
        └── web/
            ├── deployment.yaml
            └── service.yaml
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

### Google OAuth

Google Cloud Console must contain the production redirect URI:

```text
https://english-core-speaking.cleanbrain.me/api/auth/google/callback
```

The configured application value must exactly match the authorized redirect URI.

---

# GHCR image pull authentication

If the GHCR packages are private, Kubernetes requires a registry pull secret.

Create it once in the application namespace:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<credential-with-read-packages>
```

Both application Deployments reference:

```text
imagePullSecrets:
  - name: ghcr-pull
```

If the packages are later made public, this Secret can be removed along with the corresponding Deployment references.

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

## 3. GHCR pull secret

If GHCR is private:

```bash
kubectl -n cleanbrain-me-english-core-speaking \
  create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<credential-with-read-packages>
```

## 4. Application Secret

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

## 5. PostgreSQL

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

## 6. API

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

## 7. Web

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

## 8. HTTPRoute

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

Application CI/CD lives in:

```text
english-core-speaking/.github/workflows/deploy.yml
```

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

Configure these in the `english-core-speaking` repository.

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

| Component  | CPU request | Memory request | CPU limit | Memory limit |
| ---------- | ----------: | -------------: | --------: | -----------: |
| PostgreSQL |        250m |          256Mi |     1000m |        512Mi |
| API        |        100m |          128Mi |      500m |        256Mi |
| Web        |         50m |           32Mi |      200m |         64Mi |
| **Total**  |    **400m** |      **416Mi** | **1700m** |    **832Mi** |

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
