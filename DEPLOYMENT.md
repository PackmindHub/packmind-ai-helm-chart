# Packmind Helm Chart - Deployment Guide

## Quick Start

Deploy Packmind with default settings (internal PostgreSQL and Redis):

```bash
helm install packmind .
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
- `packmind-postgresql-data` PVC (default: 2Gi)
- `packmind-redis-data` PVC (default: 8Gi)
- `packmind-api-secrets` (contains encryption key)

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

## Resource Configuration

### Production Sizing

```yaml
api:
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
```

## Health Checks

All services include HTTP health checks:

- **API:** `/api/v0/healthcheck` on port 3000
- **MCP Server:** `/mcp/healthcheck` on port 3001
- **Frontend:** `/` on port 8080

## Troubleshooting

```bash
# Check all resources
kubectl get all -l app.kubernetes.io/instance=packmind

# View API logs
kubectl logs -l app.kubernetes.io/component=api -f

# Check encryption key
kubectl get secret packmind-api-secrets -o jsonpath='{.data.encryption-key}' | base64 -d

# Test database connectivity
kubectl exec deployment/packmind-api -- env | grep DATABASE_URL

# Port forward for local access
kubectl port-forward svc/packmind-frontend 8080:80
```

## Upgrading

```bash
# Upgrade
helm upgrade packmind . -f your-values.yaml

# Verify
kubectl rollout status deployment/packmind-api
```

**Note:** The encryption key persists across upgrades automatically. Ensure you have backed up critical secrets before upgrading.
