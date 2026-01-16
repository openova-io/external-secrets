# SPEC: Secrets Management Configuration

## Overview

Two-layer secrets architecture for infrastructure and Kubernetes.

## SOPS Configuration

### Key Setup

```bash
# Generate age key
age-keygen -o ~/.config/sops/age/keys.txt

# Public key in .sops.yaml
creation_rules:
  - path_regex: \.tfvars$
    age: age1...public-key...
```

### Encrypt/Decrypt

```bash
# Encrypt
sops --encrypt terraform.tfvars > terraform.tfvars.enc

# Decrypt
sops --decrypt terraform.tfvars.enc > terraform.tfvars

# Edit in-place
sops terraform.tfvars.enc
```

## External Secrets Operator

### ClusterSecretStore (MVP)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: main-store
spec:
  provider:
    kubernetes:
      remoteNamespace: secrets
      server:
        caProvider:
          type: ConfigMap
          name: kube-root-ca.crt
          namespace: kube-system
          key: ca.crt
```

### ExternalSecret Template

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: <service>-secrets
  namespace: <tenant>-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: main-store
    kind: ClusterSecretStore
  target:
    name: <service>-secrets
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: <tenant>-postgres
        property: url
    - secretKey: API_KEY
      remoteRef:
        key: <tenant>-api-keys
        property: main
```

## Secret Types

| Secret | Layer | Storage | Rotation |
|--------|-------|---------|----------|
| Cloud credentials | Terraform | SOPS | On compromise |
| SSH keys | Terraform | SOPS | On compromise |
| Database passwords | K8s | ESO | 90 days |
| API keys | K8s | ESO | On compromise |
| JWT signing keys | K8s | ESO | 30 days |
| TLS certificates | K8s | cert-manager | Auto |

## Critical Backup

The age private key is the ONLY secret requiring manual backup:
- Location: `~/.config/sops/age/keys.txt`
- Backup: Password manager + physical copy

**Warning:** Losing this key makes all SOPS secrets unrecoverable.

## Related

- [ADR-SECRETS-MANAGEMENT](./ADR-SECRETS-MANAGEMENT.md)
