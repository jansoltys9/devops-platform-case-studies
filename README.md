# DevOps Platform Case Studies

This repository contains concise technical case studies based on practical DevOps, cloud and platform-engineering work.

Each case study focuses on:

- the original technical state
- the problem or limitation
- the selected architecture or migration approach
- implementation responsibilities
- technical challenges
- validation and resulting improvements

The repository complements the broader
[JSAPPINF Platform Portfolio](https://github.com/jansoltys9/jsappinf-platform-portfolio).

## Case Studies

### Kubernetes Routing Migration

[ingress-nginx and NLB to Gateway API and AWS ALB](case-studies/kubernetes-routing-migration/README.md)

A routing architecture refactoring within an AWS EKS platform, introducing Kubernetes Gateway API, AWS Load Balancer Controller and Application Load Balancer integration.

### PostgreSQL → Aurora Migration Series

Two parts of one cross-region lab migration, from environment preparation to new application writes on Aurora.

1. **[Building the Migration Environment](case-studies/postgresql-aurora-migration-environment/README.md)** — Source and target, bootstrap, private networking, privileges and DMS placement.
2. **[DMS Full Load, CDC and Application Cutover](case-studies/postgresql-aurora-dms-cutover/README.md)** — Task failure and troubleshooting, Full Load + CDC, a real application switch, network and database repairs, and validation of new Aurora writes.

## Planned Case Studies

- GitLab container-image pipeline evolution
- Helm chart structure and runtime refactoring
- EKS upgrade and support-lifecycle strategy
- Terraform repository consolidation

Sensitive values, account identifiers, resource IDs and credentials are excluded.
