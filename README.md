# OpenCode Grafana

A complete GitOps infrastructure for Kubernetes with ArgoCD, Grafana, VictoriaMetrics, and secret management with OpenBao.

## Architecture

This project provides a complete monitoring and GitOps stack:

- **ArgoCD** - GitOps continuous delivery
- **Grafana** - Visualization and dashboards
- **VictoriaMetrics** - Time-series database for metrics
- **Fluent-bit** - Log collection and forwarding
- **OpenBao** - Secret management (open-source Vault fork)
- **External Secrets Operator** - Kubernetes secret synchronization from OpenBao
- **Traefik** - Ingress controller (via cluster CNI)

## Prerequisites

- Docker
- Kind (Kubernetes in Docker)
- kubectl
- kustomize
- helm

## Quick Start

1. **Create the cluster**:
   ```bash
   kind create cluster --config cluster.yaml
   ```

2. **Deploy infrastructure**:
   ```bash
   kustomize build --enable-helm infrastructure/overlays/local/argocd --load-restrictor=LoadRestrictionsNone | kubectl apply -f -
   ```

3. **Deploy all applications**:
   ```bash
   kustomize build --enable-helm infrastructure/overlays/local --load-restrictor=LoadRestrictionsNone | kubectl apply -f -
   ```

## Access Services

### ArgoCD
- URL: https://argocd.192.168.174.195.sslip.io
- Username: `admin`
- Password: Retrieved from OpenBao via External Secrets Operator (default: `admin`)

### Grafana
- URL: https://grafana.192.168.174.195.sslip.io
- Username: `admin`
- Password: Retrieved from OpenBao via External Secrets Operator (default: `admin`)

### OpenBao
- URL: https://openbao.192.168.174.195.sslip.io
- Token: `dev-only-token` (development only)
- **Note**: This is a development OpenBao instance using dev mode

## Components

### ArgoCD
GitOps continuous delivery tool for Kubernetes. Configured to sync applications from this repository.

### Grafana
Visualization platform with VictoriaMetrics as default datasource. Includes:
- Pre-configured datasource to VictoriaMetrics
- Admin credentials managed via secrets
- IngressRoute for external access

### VictoriaMetrics
Time-series database with:
- Single node deployment (VMSingle)
- 30-day retention period
- VM-Agent for metric scraping
- 30Gi storage

### Fluent-bit
Log collector and forwarder configured to:
- Collect logs from all pods
- Output logs to stdout (for local development)
- Can be configured to send to VictoriaLogs or other destinations

### External Secrets Operator
Manages secrets from external sources with:
- OpenBao integration for secret management
- SecretStore and ExternalSecret CRDs for external secret integration
- Automatic synchronization of secrets to Kubernetes
- Support for various secret providers (AWS Secrets Manager, HashiCorp Vault, etc.)

### OpenBao
Open-source secret management tool (Vault fork) providing:
- Centralized secret storage and management
- KV secrets engine for storing sensitive data
- Dev mode for local development (no persistence)
- UI for managing secrets
- Token-based authentication
- Automatic secret initialization via post-sync hook

## Secret Management

This project uses OpenBao for secret storage, with External Secrets Operator synchronizing secrets to Kubernetes.

### How It Works

1. **OpenBao** stores all secrets in a centralized, secure location
2. **Secrets are initialized** automatically via a post-sync hook job that:
   - Enables KV v2 secrets engines for argocd and grafana
   - Creates initial secrets (admin credentials)
3. **External Secrets Operator** watches OpenBao and syncs secrets to Kubernetes
4. **Applications** use standard Kubernetes secrets that are automatically updated

### Local Development

For local development, OpenBao runs in **dev mode**:
- No persistent storage (secrets are reset on restart)
- Default token: `dev-only-token`
- Automatic initialization via Kubernetes Job
- Secrets initialized in `secret/argocd` and `secret/grafana` paths

### Managing Secrets in OpenBao

Access the OpenBao UI or use the CLI:

```bash
# Login to OpenBao
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="dev-only-token"

# List secrets
vault secrets list

# Read a secret
vault kv get secret/argocd/adminPassword

# Write a new secret
vault kv put secret/myapp \
  username="myuser" \
  password="mypassword"

# Create a new ExternalSecret to sync this secret
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: my-namespace
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: openbao-secret-store
    kind: SecretStore
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: myapp
      property: username
  - secretKey: password
    remoteRef:
      key: myapp
      property: password
EOF
```

### Production Setup

For production, configure OpenBao with:

1. **Proper storage backend** - Consul, S3, etc.
2. **TLS encryption** - Enable TLS for API communication
3. **Strong authentication** - Use approle, Kubernetes auth, or OIDC
4. **Auto-unseal** - Configure with AWS KMS, GCP KMS, etc.
5. **Audit logging** - Enable audit logs for compliance

Example SecretStore for production:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: openbao-secret-store
  namespace: my-namespace
spec:
  provider:
    vault:
      server: "https://openbao.example.com:8200"
      path: "secret"
      version: "v2"
      caProvider:
        type: Secret
        name: openbao-ca
        key: ca.crt
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets-controller
```

## Configuration

### Fluent-bit Output
By default, fluent-bit outputs logs to stdout for local development. To configure a different output:

Edit `infrastructure/base/fluent-bit/values.yaml`:
```yaml
outputs: |
  [OUTPUT]
      Name vmlogs
      Match kube.*
      URL: http://vlogs:9428/insert/jsonline
```

### Grafana Datasources
Grafana is pre-configured with VictoriaMetrics datasource. To add more:

Edit `infrastructure/base/grafana/values.yaml`:
```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: MyDatasource
        type: prometheus
        url: http://my-datasource:9090
        access: proxy
```

## Directory Structure

```
.
├── cluster.yaml                           # Kind cluster configuration
├── infrastructure/
│   ├── base/                              # Base infrastructure components
│   │   ├── argocd/                        # ArgoCD base configuration
│   │   ├── grafana/                       # Grafana base configuration
│   │   ├── fluent-bit/                    # Fluent-bit base configuration
│   │   ├── vm-single/                     # VictoriaMetrics single node
│   │   ├── vm-operator/                   # VictoriaMetrics operator
│   │   ├── cni/                           # CNI configuration
│   │   ├── external-secrets-operator/     # External Secrets Operator
│   │   └── openbao/                      # OpenBao secret management
│   ├── overlays/                         # Environment-specific overlays
│   │   └── local/                        # Local development overlay
│   │       ├── argocd/                    # ArgoCD local configuration
│   │       ├── grafana/                   # Grafana local configuration
│   │       ├── fluent-bit/                # Fluent-bit local configuration
│   │       ├── openbao/                   # OpenBao local configuration
│   │       └── external-secrets-operator/# External secrets local config
│   └── components/                       # Reusable components
│       └── local-secrets/                # Local development secrets
├── init/                                # Initial setup resources
│   └── local/                           # Local init resources
└── mise.toml                            # Mise tool configuration
```

## Troubleshooting

### ArgoCD not syncing
```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```

### Grafana not accessible
```bash
kubectl get pods -n grafana
kubectl logs -n grafana deployment/grafana
```

### Fluent-bit not collecting logs
```bash
kubectl get pods -n fluent-bit
kubectl logs -n fluent-bit daemonset/fluent-bit
```

### External Secrets not syncing
```bash
kubectl get externalsecrets -A
kubectl describe externalsecret <name> -n <namespace>
```

### OpenBao issues

#### Check OpenBao status
```bash
kubectl get pods -n openbao
kubectl logs -n openbao deployment/openbao
```

#### Check OpenBao connection
```bash
kubectl port-forward -n openbao svc/openbao 8200:8200

# Test connection
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="dev-only-token"
vault status
```

#### View secrets in OpenBao
```bash
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="dev-only-token"

# List secrets engines
vault secrets list

# Read ArgoCD secret
vault kv get secret/argocd/adminPassword

# Read Grafana secret
vault kv get secret/grafana/credentials
```

#### Reinitialize OpenBao secrets
```bash
kubectl delete job -n openbao openbao-init
kubectl apply -f infrastructure/overlays/local/openbao/init-job.yaml
```

## Development Workflow

1. Make changes to the repository
2. Commit and push changes
3. ArgoCD automatically detects changes and syncs applications
4. Monitor progress in ArgoCD UI

## Security Considerations

⚠️ **Important Security Notes:**

1. **Local development only**: This configuration is for local development only
2. **Insecure mode**: ArgoCD is configured with `server.insecure: true` - disable for production
3. **OpenBao dev mode**: Running OpenBao in dev mode without persistence and with root token - NOT for production
4. **Default credentials**: Change admin passwords in OpenBao before deploying to production
5. **Secret management**: For production, configure OpenBao with:
   - Proper storage backend (Consul, S3, etc.)
   - TLS encryption
   - Auto-unseal with cloud KMS
   - Strong authentication (approle, Kubernetes auth, OIDC)
6. **Network policies**: Add network policies to restrict pod-to-pod communication
7. **TLS**: Configure proper TLS certificates for all services in production

## Future Improvements

- Add pre-commit hooks for YAML validation
- Implement CI/CD pipelines
- Add backup strategies for VictoriaMetrics and OpenBao
- Configure alerting rules for Prometheus/Grafana
- Add more Grafana dashboards for comprehensive monitoring
- Implement multi-environment support (dev/staging/prod)
- Add network policies for enhanced security
- Configure TLS for all services
- Add OpenBao monitoring and alerting
- Implement secret rotation policies in OpenBao
- Add OpenBao audit logging for compliance
- Configure OpenBao with auto-unseal using cloud KMS
- Add more external secret integrations (AWS Secrets Manager, Azure Key Vault)
- Implement secret versioning and rollback capabilities

