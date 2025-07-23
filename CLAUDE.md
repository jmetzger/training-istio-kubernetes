# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a comprehensive Kubernetes Security Training repository containing educational materials, exercises, and practical examples for learning Kubernetes security best practices. The content follows a layered security approach from OS/Kernel to Application level.

## Architecture and Organization

### Content Structure
The repository is organized by security layers and learning progression:

- **`/security/`** - Core security content organized by protection layers
  - `overview/` - Security fundamentals, attack vectors, protection layers
  - `by.layer/pods-container/` - Pod and container security (SecurityContext, capabilities, runAs, sysctls)
  - `cluster/components/` - Cluster component security (kubelet, API server)
  - `admissionController/` - Policy enforcement (OPA Gatekeeper, PSA, image verification)
  - `scanning-containers/` - Image security and vulnerability scanning
  - `explore/` - Hands-on security exercises and hacking sessions

- **`/kubernetes/`** - Core Kubernetes security concepts
  - `rbac/` - Complete RBAC implementation guides and exercises
  - `secrets/` - Secret management practices

- **`/kubectl-examples/`** - Practical command examples and exercises
- **`/istio/`** - Service mesh security implementation
- **`/monitoring/training-stack/`** - Infrastructure setup and deployment

### Learning Progression
Content follows numbered sequences (00-overview.md, 01-*.md, 02-*.md) with consistent patterns:
1. Theoretical overview
2. Practical exercises (exercise.md)
3. Step-by-step walkthroughs (walkthrough.md)
4. Real-world examples and case studies

## Development Commands

### Infrastructure Setup
Training environments use Terraform for DigitalOcean clusters:

```bash
# Deploy training cluster (takes 6-7 minutes)
cd monitoring/training-stack/
terraform init
terraform apply -auto-approve

# If deployment fails, retry or redeploy
terraform apply -auto-approve
# Or full reset:
terraform destroy -auto-approve && terraform apply -auto-approve

# Test ingress-nginx deployment
kubectl -n ingress-nginx get svc
curl <ingress-ip>  # Should return 404
```

### Security Scanning
Key security tools and commands referenced throughout:

```bash
# CIS Benchmark scanning with kube-bench
kubectl apply -f job.yaml
kubectl logs <kube-bench-pod>

# Container image scanning (examples use Trivy)
# Network policy testing
# OPA Gatekeeper policy validation
```

## Key Files and Entry Points

### Essential Reading Order
1. **`/README.md`** - Complete curriculum agenda and navigation
2. **`/security/truth.md`** - Security philosophy: "ongoing process, not safe by default"
3. **`/security/overview/layers-2-protect.md`** - Security layers framework
4. **`/security/checklist/security-checklist.md`** - Comprehensive security checklist

### Critical Security Content
- **`/kubernetes/rbac/`** - Complete RBAC implementation (starting point for access control)
- **`/security/admissionController/`** - Policy enforcement mechanisms
- **`/security/by.layer/pods-container/`** - Container-level security controls
- **`/security/explore/01-hack-session-hostpid.md`** - Practical security exercise

### Training Infrastructure
- **`/monitoring/training-stack/install.md`** - Complete setup instructions
- Uses wildcard domain: `*.tlnx.do.t3isp.de`
- Includes metallb, ingress-nginx, and 3-node clusters

## Content Conventions

### Writing Style
- Direct, practical approach with minimal theory
- Hands-on exercises emphasize real-world scenarios
- References actual security incidents (Tesla breach case study)
- Mixed English/German content (primarily English)

### File Naming
- Sequential numbering for learning progression
- `exercise.md` for hands-on activities
- `walkthrough.md` for step-by-step guides
- `overview.md` for conceptual introductions

### Security Focus Areas
1. **OS/Kernel Layer** - System hardening
2. **Cluster Layer** - Component security, RBAC
3. **Pod/Container Layer** - SecurityContext, capabilities, privileges
4. **Network Layer** - Policies and service mesh
5. **Image Layer** - Scanning and verification
6. **Runtime Layer** - Monitoring and response

## Important Notes

- This is a training repository - content focuses on education and defensive security
- No build processes or compilation required - pure documentation
- Infrastructure examples use DigitalOcean with Terraform
- Exercises assume access to Kubernetes clusters for hands-on learning
- Content covers industry-standard security tools (OPA Gatekeeper, Trivy, kube-bench)