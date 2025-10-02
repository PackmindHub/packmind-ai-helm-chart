# Packmind Helm Chart - Deployment Guide

## Table of Contents

- [Quick Start](#quick-start)
- [Critical: Encryption Key](#critical-encryption-key)
- [Database Configuration](#database-configuration)
- [Environment Variables](#environment-variables)
- [Ingress Configuration](#ingress-configuration)
- [Backup Considerations](#backup-considerations)
- [Additional Secrets](#additional-secrets)
- [Resource Configuration](#resource-configuration)

## Quick Start

### Add Helm Repository

```bash
helm repo add packmind https://packmindHub.github.io/packmind-ai-helm-chart/
helm repo update
```

### Install Packmind

Deploy Packmind with default settings (internal PostgreSQL and Redis):

```bash
helm install packmind packmind/packmind
```

Or install from local chart directory:

```bash
helm install packmind packmind/
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

### API Service Environment Variables

Override environment variables for the API service:

```yaml
api:
  env:
    APP_WEB_URL: "https://local.packmind.acme"
```

### MCP Server Environment Variables

Override environment variables for the MCP server:

```yaml
mcpServer:
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

mcpServer:
  secretEnvVars:
    - name: MCP_EXTERNAL_TOKEN
      secretName: vault-mcp-secret
      key: external-token
      optional: false
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

**Routes:** `/api` → API, `/mcp` → MCP Server, `/` → Frontend

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
  mcp:
    jwtSecretKey: "your-mcp-jwt-secret"
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
