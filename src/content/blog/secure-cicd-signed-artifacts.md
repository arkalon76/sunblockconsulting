---
title: 'Secure CI/CD: How to Deploy Signed Artifacts Across All Environments'
description: 'Learn how to build, sign, and deploy reproducible code artifacts securely. This guide covers container signing, artifact integrity verification, and best practices for immutable deployment artifacts across dev, staging, and production.'
pubDate: 'Jan 09 2026'
heroImage: '/cicd-signing-hero.png'
heroImageAlt: 'Secure CI/CD pipeline diagram showing cryptographic signing and verification across build, staging, and production environments'
---

In modern software delivery, the question isn't just "does the code work?" but "can we prove this is the exact code we tested?" Deploying signed artifacts ensures your software hasn't been tampered with between your CI pipeline and production—giving you security, speed, and repeatability.

## What Are Signed Artifacts?

**Signed artifacts** are software packages, container images, or binaries that have been cryptographically signed to prove their authenticity and integrity. When you sign an artifact, you create a mathematical proof that the code came from a trusted source and hasn't been modified since it was built. This is fundamental to software supply chain security.

## Why Signed Artifacts Matter

Every time you deploy code that wasn't cryptographically verified, you're taking a risk. Supply chain attacks, compromised build servers, and configuration drift can all introduce vulnerabilities. Signed artifacts solve this by creating a cryptographically verifiable chain of custody from build to deployment.

### The Core Benefits

- **Security**: Cryptographic signatures prove code authenticity and integrity
- **Speed**: Build once, deploy everywhere—no rebuilding per environment
- **Repeatability**: Same artifact in dev, staging, and production guarantees consistency
- **Auditability**: Full traceability of what was deployed and when

## Building Reproducible Artifacts

The first step is ensuring your build process creates identical outputs given identical inputs. This is called **reproducible builds**—a cornerstone of any DevSecOps CI/CD pipeline.

### Key Principles

1. **Pin all dependencies** with exact versions and checksums
2. **Use immutable base images** with specific digests, not floating tags
3. **Eliminate timestamps** and random values from build outputs
4. **Version your build tools** to ensure consistent compilation

```dockerfile
# Bad: Uses floating tag
FROM node:18

# Good: Uses immutable digest
FROM node:18@sha256:abc123...
```

## Signing Your Artifacts

Once you have a reproducible build, the next step is signing it. The two most common approaches are **Sigstore/Cosign** for containers and **GPG/PGP** for traditional packages.

### Container Signing with Cosign

[Cosign](https://github.com/sigstore/cosign) from the Sigstore project has become the de facto standard for container signing. It's keyless, meaning you can sign using your CI provider's OIDC identity.

```bash
# Generate a keypair (for key-based signing)
cosign generate-key-pair

# Sign an image with a key
cosign sign --key cosign.key ghcr.io/your-org/your-app:v1.0.0

# Keyless signing with OIDC (recommended for CI)
cosign sign ghcr.io/your-org/your-app:v1.0.0
```

The keyless approach is powerful because the signature includes the identity of the signer (your CI job) and a timestamp, all recorded in a tamper-proof transparency log.

### Generating and Storing Signatures

For production use, implement signing in your CI pipeline. In GitHub Actions, the OIDC subject follows the format `repo:org/repo:ref:refs/heads/main`, allowing you to scope trust to specific repositories and branches:

```yaml
# GitHub Actions example
- name: Sign Container Image
  run: |
    cosign sign \
      --oidc-issuer https://token.actions.githubusercontent.com \
      ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

## Verification at Deploy Time

Signing is only half the story. You must **verify** signatures before deployment to complete artifact integrity verification.

### Policy Enforcement with Admission Controllers

In Kubernetes, use an admission controller like **Kyverno** or **Gatekeeper** to enforce signature verification. For more on Kubernetes security, see our [DevSecOps services](/services):

```yaml
# Kyverno policy example
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign-signature
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/your-org/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "repo:your-org/your-repo:ref:refs/heads/main"
```

This policy ensures that **only pods with verified signatures from your CI pipeline can run in your cluster**.

## Implementing Immutable Deployment Artifacts Across Environments

The beauty of signed artifacts is environment promotion without rebuilding:

```text
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Build    │────▶│   Staging   │────▶│ Production  │
│   + Sign    │     │   Verify    │     │   Verify    │
└─────────────┘     └─────────────┘     └─────────────┘
     │                    │                    │
     ▼                    ▼                    ▼
  Same SHA256         Same SHA256         Same SHA256
  Same Signature      Same Signature      Same Signature
```

### Environment-Specific Configuration

Use environment variables or external configuration (like ConfigMaps or Secrets) for environment-specific settings. The artifact itself remains unchanged:

```yaml
# The deployment references the same signed image
image: ghcr.io/your-org/app@sha256:abc123...
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url
```

## Supply Chain Security with SBOMs

Complement your signing strategy with **Software Bills of Materials (SBOMs)**. These provide a complete inventory of your dependencies:

```bash
# Generate SBOM with Syft
syft ghcr.io/your-org/app:v1.0.0 -o spdx-json > sbom.json

# Attach SBOM as an attestation
cosign attest --predicate sbom.json \
  --type spdxjson \
  ghcr.io/your-org/app:v1.0.0
```

Now you have both proof of authenticity (signature) and proof of contents (SBOM).

## Best Practices Checklist

Before going live, ensure you have:

- [ ] **Reproducible builds** verified by building twice and comparing outputs
- [ ] **Signing integrated into CI** with keyless OIDC when possible
- [ ] **Admission controllers** enforcing verification in all environments
- [ ] **SBOM generation** for dependency transparency
- [ ] **Transparency log monitoring** for unauthorized signing attempts
- [ ] **Key rotation plan** for key-based signing implementations
- [ ] **Incident response procedure** for compromised signatures

## Conclusion

Secure artifact deployment isn't about adding friction—it's about **removing uncertainty**. When every deployment is cryptographically verified, you gain confidence that what you tested is exactly what you're running.

Start small: sign your container images and verify them in staging. Then expand to production enforcement and SBOM attestations. Each step brings you closer to a truly secure software supply chain.

---

*Need help implementing secure CI/CD pipelines? [Contact Sunblock Consulting](/contact) for expert guidance on DevSecOps and supply chain security.*
