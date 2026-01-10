# Talent Mesh Secrets Management

## Overview

This document describes the secrets management architecture for Talent Mesh, using Infisical for self-service secrets management with External Secrets Operator for Kubernetes synchronization.

> **Architecture Decision:** See [ADR-039: Secrets Management - Infisical](/docs/09-adrs/ADR-039-SECRETS-MANAGEMENT-INFISICAL.md) (supersedes ADR-013)
>
> **Note:** SOPS + age is still used for Terraform bootstrap secrets only.

---

## Architecture

### Two-Layer Model

```mermaid
flowchart TB
    subgraph layer1["LAYER 1: Infrastructure (Terraform)"]
        TF_ENC[terraform.tfvars.enc<br/>SOPS + age encrypted]
        TF_ENC -->|sops decrypt| TF_APPLY[terraform apply]
        TF_APPLY --> CONTABO[Contabo VPS]
        TF_APPLY --> K3S[K3s cluster]
        TF_APPLY --> DNS[DNS failover]
    end

    subgraph layer2["LAYER 2: Kubernetes (Runtime)"]
        K8S_ENC[k8s/secrets/*.enc.yaml<br/>SOPS + age encrypted]
        K8S_ENC -->|sops decrypt<br/>kubectl apply| SECRETS_NS[secrets namespace<br/>K8s Secrets]
        SECRETS_NS -->|ESO reads| ESO[External Secrets Operator<br/>ClusterSecretStore]
        ESO -->|ESO creates| APP_NS[talent-mesh namespace<br/>Application Secrets]
    end
```

---

## Technology Stack

| Component | Purpose | Cost |
|-----------|---------|------|
| **SOPS** | Encrypt/decrypt secrets | Free |
| **age** | Encryption algorithm | Free |
| **External Secrets Operator** | Sync secrets to K8s | Free |
| **Kubernetes provider** | ESO backend (MVP) | Free |
| **HashiCorp Vault** | ESO backend (Future) | Free (self-hosted) |

---

## Layer 1: Infrastructure Secrets (SOPS)

### What's Protected

| Secret | File | Purpose |
|--------|------|---------|
| Contabo API credentials | `terraform.tfvars.enc` | Terraform Contabo provider authentication |
| SSH private key | `~/.ssh/talent-mesh.key` | VM access |
| SSH public key | `terraform.tfvars.enc` | VM provisioning |
| Domain credentials | `terraform.tfvars.enc` | DNS management |

### Workflow

```bash
# 1. Decrypt secrets
sops --decrypt terraform.tfvars.enc > terraform.tfvars

# 2. Run terraform
terraform apply

# 3. Clean up
rm terraform.tfvars  # NEVER commit unencrypted

# Or use helper script
./scripts/terraform-apply.sh apply
```

### File Locations

```
terraform/
├── environments/contabo-europe/
│   ├── terraform.tfvars.enc    # Encrypted (committed)
│   ├── terraform.tfvars        # Decrypted (NEVER commit)
│   └── terraform.tfvars.example # Template (committed)
└── .gitignore                   # Excludes *.tfvars, allows *.tfvars.enc
```

---

## Layer 2: Kubernetes Secrets (ESO)

### Architecture

```yaml
# 1. Master secrets in "secrets" namespace (SOPS encrypted in git)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: secrets
stringData:
  url: postgresql://user:pass@host:5432/db

---
# 2. ClusterSecretStore (reads from secrets namespace)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: main-store
spec:
  provider:
    kubernetes:
      remoteNamespace: secrets

---
# 3. ExternalSecret (syncs to app namespace)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: talent-mesh
spec:
  secretStoreRef:
    name: main-store
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: DATABASE_URL
    remoteRef:
      key: postgres-credentials
      property: url
```

### Secret Types

| Secret | Namespace | Used By |
|--------|-----------|---------|
| `postgres-credentials` | secrets | All services needing DB |
| `mongodb-credentials` | secrets | Assessment, User services |
| `redis-credentials` | secrets | All services |
| `linkedin-oauth` | secrets | Auth service |
| `jwt-signing-key` | secrets | Auth service |

### Workflow

```bash
# 1. Decrypt and apply master secrets
sops --decrypt k8s/secrets/base-secrets.enc.yaml | kubectl apply -f -

# 2. Verify secrets exist
kubectl get secrets -n secrets

# 3. Deploy app with ExternalSecret
kubectl apply -f k8s/apps/auth-service/

# 4. ESO automatically creates secrets in app namespace
kubectl get secrets -n talent-mesh
```

---

## Key Management

### Age Keypair

The age keypair is the **master key** for all SOPS-encrypted secrets.

| Key | Location | Committed? | Backup Required? |
|-----|----------|------------|------------------|
| Public key | `.sops.yaml`, `.age-public-key` | Yes | No (in git) |
| Private key | `~/.config/sops/age/keys.txt` | **NEVER** | **YES!** |

### Backup Strategy

1. **Password Manager**: Store private key in 1Password/Bitwarden
2. **Physical Backup**: Print and store in safe
3. **Recovery Test**: Periodically verify you can decrypt

```bash
# Test decryption
sops --decrypt terraform/environments/contabo-europe/terraform.tfvars.enc | head -5
```

### Key Rotation

1. Generate new age keypair
2. Update `.sops.yaml` with new public key
3. Re-encrypt all secrets with new key
4. Securely destroy old private key

```bash
# Generate new key
age-keygen -o ~/.config/sops/age/keys-new.txt

# Update .sops.yaml with new public key
# Then re-encrypt all files
for f in $(find . -name "*.enc" -o -name "*.enc.yaml"); do
    sops rotate --in-place "$f"
done
```

---

## Migration to Enterprise Vault

When scaling to enterprise, only the `ClusterSecretStore` changes:

```yaml
# MVP: Kubernetes provider
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: main-store
spec:
  provider:
    kubernetes:
      remoteNamespace: secrets

---
# ENTERPRISE: Vault provider (ONLY THIS CHANGES)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: main-store
spec:
  provider:
    vault:
      server: "https://vault.talent-mesh.io"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
```

**Key insight:** All `ExternalSecret` resources remain unchanged!

---

## Security Best Practices

### Do's

- Use SOPS for all secrets in git
- Backup age private key securely
- Use helper scripts for terraform
- Rotate secrets every 90 days
- Use separate secrets per environment

### Don'ts

- NEVER commit unencrypted secrets
- NEVER log secret values
- NEVER share age private key via Slack/email
- NEVER hardcode secrets in code
- NEVER use same secrets across environments

---

## File Reference

```
talent-mesh/
├── .sops.yaml                    # SOPS configuration (age public key)
├── .age-public-key               # Age public key (safe to share)
├── scripts/
│   ├── setup-sops.sh             # Install SOPS + age
│   ├── encrypt-secrets.sh        # Encrypt helper
│   ├── decrypt-secrets.sh        # Decrypt helper
│   └── terraform-apply.sh        # Terraform with auto-decrypt
├── terraform/
│   └── environments/contabo-europe/
│       ├── terraform.tfvars.enc  # Encrypted terraform secrets
│       └── terraform.tfvars.example
└── k8s/
    ├── infrastructure/
    │   └── external-secrets/
    │       ├── cluster-secret-store.yaml
    │       └── helmrelease.yaml
    └── secrets/
        ├── base-secrets.enc.yaml # Encrypted K8s secrets
        └── README.md
```

---

## Troubleshooting

### Cannot decrypt

```bash
# Check age key exists
cat ~/.config/sops/age/keys.txt

# Verify key matches .sops.yaml public key
grep "age1" ~/.config/sops/age/keys.txt
grep "age1" .sops.yaml

# Force use specific key
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops --decrypt file.enc
```

### ESO not syncing

```bash
# Check ESO pods
kubectl get pods -n external-secrets

# Check ClusterSecretStore status
kubectl get clustersecretstores

# Check ExternalSecret status
kubectl get externalsecrets -A
kubectl describe externalsecret <name> -n <namespace>
```

### SOPS config not found

```bash
# Ensure .sops.yaml exists in project root
cat .sops.yaml

# Or specify config explicitly
sops --config /path/to/.sops.yaml --encrypt file
```

---

## References

- [ADR-013: Secrets with SOPS and ESO](/docs/09-adrs/ADR-013-SECRETS-SOPS-ESO.md)
- [INFRASTRUCTURE_SETUP.md](/docs/08-operations/INFRASTRUCTURE_SETUP.md)
- [SOPS Documentation](https://github.com/getsops/sops)
- [age Documentation](https://github.com/FiloSottile/age)
- [External Secrets Operator](https://external-secrets.io/)

---

*Document Version: 1.1*
*Last Updated: 2026-01-07*
*Owner: Platform Team*
