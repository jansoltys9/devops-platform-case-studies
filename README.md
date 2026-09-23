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

### PostgreSQL → Aurora Migration Environment

[Building a Cross-Region PostgreSQL to Aurora Migration Environment](case-studies/postgresql-aurora-migration-environment/README.md)

Preparing source PostgreSQL, private cross-region networking, Aurora administration and AWS DMS prerequisites for a running application migration.

### AWS DMS Full Load, CDC and Application Cutover

[From Green DMS Endpoints to a Working Application](case-studies/postgresql-aurora-dms-cutover/README.md)

A lab migration through replication troubleshooting, a real application cutover, network and sequence/default repairs, and successful new application writes on Aurora.

## Planned Case Studies

- GitLab container-image pipeline evolution
- Helm chart structure and runtime refactoring
- EKS upgrade and support-lifecycle strategy
- Terraform repository consolidation

Sensitive values, account identifiers, resource IDs and credentials are excluded.
