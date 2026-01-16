# ADR: Secrets Management (SOPS + ESO)

**Status:** Accepted
**Date:** 2024-04-01

## Context

Need secure secrets management for both Terraform and Kubernetes.

## Decision

Implement two-layer secrets architecture:
1. **Terraform secrets:** SOPS + age encryption
2. **Kubernetes secrets:** External Secrets Operator (ESO)

## Rationale

### Why SOPS for Terraform?

| Alternative | Pros | Cons |
|-------------|------|------|
| **SOPS + age** | Git-stored, zero cost | Manual encryption | **Selected** |
| HashiCorp Vault | Enterprise features | Infrastructure overhead |
| AWS Secrets Manager | Managed | Cloud dependency, cost |

### Why ESO for Kubernetes?

| Alternative | Pros | Cons |
|-------------|------|------|
| **ESO** | Backend-agnostic | Additional component | **Selected** |
| Direct SOPS | Simple | No dynamic refresh |
| Sealed Secrets | Git-native | No external backend |

**Key Factor:** ESO allows backend swap without app changes.

## Architecture

```
Terraform Layer:
  terraform.tfvars.enc → SOPS decrypt → terraform apply

Kubernetes Layer:
  secrets namespace → ESO → application namespaces
```

## MVP Configuration

**Zero-cost MVP:**
- SOPS encrypts secrets in git
- ESO with Kubernetes provider (reads from `secrets` namespace)
- No external vault service

**Future enterprise:**
- Swap ESO backend to HashiCorp Vault
- Only `ClusterSecretStore` changes needed

## Consequences

**Positive:** Zero cost, git-stored, backend-agnostic, future-proof
**Negative:** Manual SOPS for Terraform, ESO overhead

## Related

- [SPEC-SECRETS-CONFIGURATION](./SPEC-SECRETS-CONFIGURATION.md)
- [ADR-SECRETS-INFISICAL](./ADR-SECRETS-INFISICAL.md)
