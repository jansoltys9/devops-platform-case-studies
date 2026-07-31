# Kubernetes Routing Migration

## ingress-nginx and NLB to Gateway API and AWS ALB

## Context

This case study describes the refactoring of the application-routing layer in an AWS EKS platform.

The work is part of the broader
[JSAPPINF Platform Portfolio](https://github.com/jansoltys9/jsappinf-platform-portfolio).

The goal was not only to replace one Kubernetes routing resource with another, but to establish clearer responsibility boundaries between AWS infrastructure, Kubernetes runtime configuration and GitOps delivery.

## Original Architecture

```text
Internet
→ AWS Network Load Balancer
→ ingress-nginx
→ Kubernetes Ingress
→ Kubernetes Service
```

The original model was functional, but it introduced an additional proxy layer inside the cluster and made several responsibilities less explicit.

These included:

- AWS load-balancer integration
- Kubernetes routing
- controller IAM permissions
- TLS and DNS integration
- application exposure rules
- deployment and synchronization order

## Target Architecture

```text
Internet
→ Route 53
→ CloudFront
→ AWS WAF
→ Application Load Balancer
→ Kubernetes Gateway API
→ HTTPRoute
→ Kubernetes Service
```

Long-running CloudFront and WAF exposure remains intentionally disabled in the development environment.

The routing model, application flow and responsibility boundaries have been defined and validated.

## Architecture Decision

The refactored design uses:

- Kubernetes Gateway API for routing contracts
- HTTPRoute for application and API routing
- AWS Load Balancer Controller for AWS integration
- Application Load Balancer for Layer 7 routing
- Terraform for AWS prerequisites and IAM
- Helm for reusable Kubernetes packaging
- Argo CD for GitOps-managed runtime state

## Responsibility Model

### Terraform

Terraform owns:

- AWS infrastructure
- EKS prerequisites
- IAM and IRSA
- AWS Load Balancer Controller prerequisites
- DNS and certificate prerequisites
- Argo CD bootstrap

### Helm

Helm owns:

- reusable Kubernetes templates
- deployment packaging
- default chart structure

### Argo CD and GitOps

GitOps owns:

- Gateway resources
- HTTPRoute resources
- environment-specific Helm values
- application desired state
- synchronization order

AWS Load Balancer Controller translates the Kubernetes routing configuration into AWS ALB resources.

## Main Implementation Areas

The migration included:

- removing the ingress-nginx routing dependency
- replacing NLB-based ingress exposure with ALB integration
- configuring AWS Load Balancer Controller through IRSA
- installing and validating Gateway API CRDs
- defining Gateway and HTTPRoute resources
- validating subnet and security-group discovery
- separating AWS foundation deployment from Kubernetes runtime deployment
- preparing application and API path-based routing
- aligning Route 53, ACM, CloudFront and WAF responsibilities

## Technical Challenges

The main challenges were not limited to Gateway API syntax.

They included:

- controller IAM permissions
- IRSA trust configuration
- controller startup dependencies
- Gateway API CRD installation order
- subnet discovery and tagging
- security-group connectivity
- Terraform bootstrap ordering
- separation between Terraform, Helm and GitOps ownership

## Validation

Validated areas included:

- AWS Load Balancer Controller deployment
- IAM and IRSA integration
- Gateway API CRD availability
- Gateway and HTTPRoute ownership
- ALB-oriented routing architecture
- Terraform and Kubernetes runtime separation
- GitOps synchronization responsibilities

## Result

The resulting design provides:

- native AWS ALB integration
- clearer routing ownership
- support for application and API path-based routing
- a clean integration path for CloudFront and AWS WAF
- reusable GitOps-managed Gateway API resources
- better separation between Terraform, Helm and Argo CD

## Current Status

Validated:

- Gateway API platform direction
- AWS Load Balancer Controller integration
- ALB-based routing responsibilities
- HTTPRoute ownership model
- Terraform, Helm and GitOps boundaries

Planned evolution:

- unified application entry point
- API path-based routing
- long-running CloudFront and WAF integration
- Cognito authentication integration

## Key Lesson

The main improvement did not come only from replacing an Ingress object with an HTTPRoute.

The most important part was defining explicit ownership boundaries between AWS infrastructure, Kubernetes runtime configuration and GitOps delivery.
