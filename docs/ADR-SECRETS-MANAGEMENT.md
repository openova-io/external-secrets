# ADR: Secrets Management with ESO + PushSecrets

**Status:** Accepted
**Date:** 2024-04-01
**Updated:** 2026-01-16

## Context

Need secure secrets management that:
- Never stores secrets in Git (no SOPS)
- Works across multiple regions
- Supports auto-generation of complex secrets
- Is fully declarative and GitOps-compatible

## Decision

Use **External Secrets Operator (ESO)** with **100% PushSecrets** architecture. Kubernetes Secrets are the source of truth, pushed to external backends.

**Critical:** SOPS is completely eliminated. No secrets in Git, ever.

## Architecture

```mermaid
flowchart LR
    subgraph Bootstrap["Bootstrap (One-time)"]
        Wizard[Bootstrap Wizard]
        User[Operator Input]
    end

    subgraph K8s["Kubernetes (Source of Truth)"]
        Gen[ESO Generators]
        KS[K8s Secrets]
        PS[PushSecrets]
    end

    subgraph Backends["Secrets Backends"]
        V1[Vault Region 1]
        V2[Vault Region 2]
    end

    User -->|"Initial creds"| Wizard
    Wizard -->|"Create"| KS
    Gen -->|"Auto-generate"| KS
    KS --> PS
    PS -->|"Push"| V1
    PS -->|"Push"| V2
```

## Key Principles

| Principle | Implementation |
|-----------|----------------|
| No secrets in Git | SOPS eliminated, interactive bootstrap |
| K8s is source of truth | All secrets originate as K8s Secrets |
| Push, not pull | PushSecrets push to backends |
| Multi-region sync | Push to both Vaults simultaneously |
| Auto-generation | ESO Generators create complex secrets |

## Secret Flow

### Initial Bootstrap

```mermaid
sequenceDiagram
    participant Op as Operator
    participant Wiz as Bootstrap Wizard
    participant K8s as Kubernetes
    participant V1 as Vault R1
    participant V2 as Vault R2

    Op->>Wiz: Enter initial credentials
    Wiz->>K8s: Create K8s Secrets
    K8s->>K8s: ESO PushSecret triggers
    K8s->>V1: Push secrets
    K8s->>V2: Push secrets (parallel)
    Note over V1,V2: Both Vaults synchronized
```

### Auto-Generated Secrets

```mermaid
sequenceDiagram
    participant Gen as ESO Generator
    participant KS as K8s Secret
    participant PS as PushSecret
    participant V1 as Vault R1
    participant V2 as Vault R2

    Gen->>KS: Generate complex password
    KS->>PS: Trigger PushSecret
    PS->>V1: Push secret
    PS->>V2: Push secret
```

## Configuration

### ESO Password Generator

```yaml
apiVersion: generators.external-secrets.io/v1alpha1
kind: Password
metadata:
  name: db-password-generator
  namespace: databases
spec:
  length: 32
  digits: 6
  symbols: 4
  noUpper: false
  allowRepeat: true
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: databases
spec:
  refreshInterval: "0"  # Generate once, never refresh
  target:
    name: db-credentials
    creationPolicy: Owner
  dataFrom:
    - sourceRef:
        generatorRef:
          apiVersion: generators.external-secrets.io/v1alpha1
          kind: Password
          name: db-password-generator
```

### PushSecret to Multiple Backends

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-db-credentials
  namespace: databases
spec:
  refreshInterval: 1h
  secretStoreRefs:
    - name: vault-region1
      kind: ClusterSecretStore
    - name: vault-region2
      kind: ClusterSecretStore
  selector:
    secret:
      name: db-credentials
  data:
    - match:
        secretKey: password
        remoteRef:
          remoteKey: databases/db-credentials
          property: password
```

### ClusterSecretStore for Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-region1
spec:
  provider:
    vault:
      server: "https://vault-r1.<domain>"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-region2
spec:
  provider:
    vault:
      server: "https://vault-r2.<domain>"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

## Supported Backends

| Backend | Type | Notes |
|---------|------|-------|
| Vault Self-Hosted | Self-hosted | Recommended, full control |
| HCP Vault | Managed | HashiCorp Cloud |
| Infisical | Managed | Developer-friendly |
| AWS Secrets Manager | Managed | If AWS chosen |
| GCP Secret Manager | Managed | If GCP chosen |
| Azure Key Vault | Managed | If Azure chosen |

## Why No SOPS?

| SOPS Approach | PushSecrets Approach |
|---------------|---------------------|
| Secrets encrypted in Git | No secrets in Git |
| Manual age key management | Vault handles encryption |
| Decrypt before apply | K8s Secret is source |
| Risk of leaked decrypted files | Secrets never on disk |

**Decision:** Interactive bootstrap is simpler and more secure than SOPS.

## Bootstrap Procedure

1. **Operator enters credentials** in wizard (terminal input)
2. **Wizard creates K8s Secrets** directly via kubectl
3. **PushSecrets sync** to Vault backends
4. **Wizard displays Vault unseal keys** (save offline!)
5. **No secrets ever written to files or Git**

## Generator Types

| Generator | Use Case |
|-----------|----------|
| Password | Database passwords, API keys |
| UUID | Unique identifiers |
| ECRAuthorizationToken | AWS ECR tokens |
| GCRAccessToken | GCP GCR tokens |
| ACRAccessToken | Azure ACR tokens |

## Consequences

**Positive:**
- No secrets in Git (eliminates leak risk)
- Auto-generation of complex secrets
- Multi-region sync via single PushSecret
- Backend-agnostic (swap without app changes)

**Negative:**
- Requires bootstrap for initial secrets
- ESO operator dependency
- Vault/backend operational overhead

## Migration from SOPS

If migrating from SOPS-based setup:

1. Create K8s Secrets from decrypted SOPS files
2. Create PushSecrets to sync to Vault
3. Verify secrets in Vault
4. Delete SOPS-encrypted files from Git
5. Delete local decrypted files

## Related

- [ADR-VAULT](../../vault/docs/ADR-VAULT.md)
- [SPEC-SECRETS-CONFIGURATION](./SPEC-SECRETS-CONFIGURATION.md)
