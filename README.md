# Packmind Helm Chart - Deployment Guide

Packmind documentation is available [here](https://packmindhub.github.io/packmind/).

## Table of Contents

- [Version Selection](#version-selection)
- [Quick Start](#quick-start)
- [Critical: Encryption Key](#critical-encryption-key)
- [Database Configuration](#database-configuration)
- [Environment Variables](#environment-variables)
- [Ingress Configuration](#ingress-configuration)
- [Backup Considerations](#backup-considerations)
- [Additional Secrets](#additional-secrets)
- [Resource Configuration](#resource-configuration)
- [DNS Configuration (Self-Hosted Kubernetes)](#dns-configuration-self-hosted-kubernetes)

## Version Selection

Packmind supports two versions: **OSS** (default) and **Enterprise**.

### OSS Version (Default)

No additional configuration required. Uses standard images:
- `packmind/api:X.X.X`
- `packmind/frontend:X.X.X`

### Enterprise Version

Set `global.version` to `enterprise`:

```yaml
global:
  version: "enterprise"
```

This automatically uses enterprise images:
- `packmind/api:X.X.X-enterprise`
- `packmind/frontend:X.X.X-enterprise`

## Quick Start

### Add Helm Repository

```bash
helm repo add packmind https://packmindHub.github.io/packmind-ai-helm-chart/
helm repo update
```

### Install Packmind

Deploy Packmind with default settings (internal PostgreSQL and Redis):

```bash
helm install packmind-ai packmind/packmind-ai
```

Or install from local chart directory:

```bash
helm install packmind-ai ./packmind
```

Access at: `http://packmind.local/` (update `/etc/hosts` for local testing)

## Critical: Encryption Key

**⚠️ The encryption key protects sensitive data (Git platform API tokens, etc.) stored in the database.**

### Default Behavior (Recommended)

By default, the chart auto-generates a secure 32-character encryption key that persists across upgrades:

```yaml
secrets:
  encryptionKeyGeneration: true  # default
```

**Important:** Back up this secret immediately after first deployment to prevent data loss.

### Manual Key (Alternative)

Provide your own encryption key:

```yaml
secrets:
  encryptionKeyGeneration: false
  api:
    encryptionKey: "your-secure-32-char-key-here"
```

## Database Configuration

### Scenario 1: Internal PostgreSQL (Default)

```yaml
postgresql:
  enabled: true
  persistence:
    size: 2Gi
```

**⚠️ Backup Required:** Persistent volumes must be backed up regularly

### Scenario 2: External Database (Hardcoded URI)

```yaml
postgresql:
  enabled: false
  external:
    databaseUrl: 'postgres://user:password@host:5432/packmind'
```

⚠️ **Not recommended for production** (credentials in values file)

### Scenario 3: External Database (Secret Reference)

```yaml
postgresql:
  enabled: false
  external:
    existingSecret: 'packmind-db-secret'
    existingSecretKey: 'database-url'
```

Create the secret:

```bash
kubectl create secret generic packmind-db-secret \
  --from-literal=database-url="postgres://user:password@host:5432/packmind"
```

✅ **Recommended for production**

## Environment Variables

All the environment variables are available [here](https://packmindhub.github.io/packmind/gs-install-self-hosted#configure-deployment-and-environment-variables).

### API Service Environment Variables

Override environment variables for the API service:

```yaml
api:
  env:
    APP_WEB_URL: "https://local.packmind.acme"
```

### Using External Secrets for Environment Variables

For production environments, use `secretEnvVars` to reference secrets managed by external systems (Vault, External Secrets Operator, etc.):

```yaml
api:
  secretEnvVars:
    - name: THIRD_PARTY_API_KEY
      secretName: vault-managed-secret
      key: api-key
      optional: false
    - name: OPTIONAL_TOKEN
      secretName: external-service-secret
      key: token
      optional: true
```

## Ingress Configuration

### Production with NGINX + Let's Encrypt

```yaml
ingress:
  enabled: true
  className: "nginx"
  defaultHost: "app.example.com"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "32m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "15"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  tls:
    - secretName: packmind-tls
      hosts:
        - app.example.com
```

**Prerequisites:**
- NGINX Ingress Controller installed
- cert-manager installed with Let's Encrypt ClusterIssuer configured

**Routes:** `/api` → API, `/` → Frontend

## Backup Considerations

### Internal Databases

When using internal PostgreSQL and Redis, ensure regular backups of:
- `packmind-postgresql-data` PVC 
- `packmind-redis-data` PVC

### External Databases

Backups are managed by your external database provider.

## Additional Secrets

### JWT Secrets

```yaml
secrets:
  api:
    jwtSecretKey: "your-api-jwt-secret"
    openaiApiKey: "your-openai-api-key-here"
```

### Private Docker Registry

```yaml
dockerRegistry:
  enabled: true
  existingSecret: "packmind-registry-secret"
```

Or create inline:

```bash
kubectl create secret docker-registry packmind-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=email@example.com
```

To inject an OpenAI API Key, you'll have to create the secret first

```bash
kubectl create secret generic my-openai-secret \
  --from-literal=open-ai-key="your-openai-api-key-here"
```

And then include in your `values.yaml`:

```yaml
api:
  secretEnvVars:
    - name: OPEN_AI_KEY
      secretName: my-openai-secret
      key: open-ai-key
      optional: false
```

## DNS Configuration (Self-Hosted Kubernetes)

The API image is Alpine-based (musl libc). Under Kubernetes' default pod DNS
(`options ndots:5` plus cluster/corporate `search` domains), musl can fail to
resolve external hosts such as `api.github.com`, producing
`getaddrinfo ENOTFOUND` and breaking GitHub App registration. This happens
because musl aborts the search-domain list on the first non-`NXDOMAIN` reply
instead of falling back to the absolute name.

If you hit this, set `ndots:"1"` on the API pod so names with at least one dot
resolve as absolute first:

```yaml
api:
  dnsConfig:
    options:
      - name: ndots
        value: "1"
```

Kubernetes merges these options onto the pod's default `resolv.conf`, so
`dnsPolicy` does not need to change. `api.dnsPolicy` is also exposed if you
need to override it (e.g. `"None"`, `"Default"`, `"ClusterFirst"`). Both keys
are empty by default, so nothing is rendered unless you opt in.

See [PackmindHub/packmind#388](https://github.com/PackmindHub/packmind/issues/388)
for details.