# Engineering DevSecOps Pipelines Across Five GitLab Repositories

> Five different delivery responsibilities, five fit-for-purpose CI baselines, and one read-only scanner-version monitor

![Repository-specific CI controls across Terraform modules, AWS platform, GitOps, Helm and Node.js services, plus an independent version monitor](assets/devsecops-five-repositories.png)

## Context

[JSAPPINF](https://github.com/jansoltys9/jsappinf-platform-portfolio) separates reusable Terraform modules, AWS platform infrastructure, Kubernetes desired state, application Helm charts and Node.js microservices into **five private GitLab repositories**. The practical DevSecOps objective was not to copy one security workflow everywhere. It was to make each pipeline check the types of defects that matter for its repository, while keeping infrastructure changes, image publication and GitOps promotion outside routine security validation.

The solution uses **GitLab Free**, pinned tool versions, reviewable merge-request jobs and a separate local Python inventory that tracks scanner-version drift. The public case study describes the controls and engineering trade-offs; it does not publish private CI configuration, credentials or runtime settings.

**Evidence boundary:** the control descriptions come from the recorded project implementation, supplied job metadata and the locally executed version monitor in November 2026. Detailed CI job definitions were not independently retrieved for every repository during this write-up. Where the exact job name or enforcement policy was not evidenced, the text describes the control's purpose without claiming an unverified blocking gate. Passing CI is not a production security certification.

## Original State

The delivery responsibilities were deliberately separated:

| Repository | Owns | Typical failure to catch before merge |
|---|---|---|
| `terraform-modules` | Reusable Terraform module interfaces | Invalid HCL, broken module validation, leaked credentials, dangerous IaC patterns |
| `jsappinf-platform` | AWS/EKS foundation and platform add-ons | Invalid Terraform composition, unwanted CI infrastructure actions, secrets and IaC risks |
| `jsappinf-gitops` | Argo CD desired state for environments | Risky or inconsistent GitOps configuration and committed secrets |
| `helmchartsappjs` | Reusable Kubernetes application charts | Invalid rendered Kubernetes resources or unsafe configuration |
| `JSAPP` | UI, user, product and order Node.js services | Service contract regressions, exposed secrets, source-code/Dockerfile findings and missing inventory |

The application release path already used service-specific semantic versions. CI validation, container build/scan/push, Helm configuration updates and Argo CD reconciliation have **different owners and triggers**. Security-only changes should not silently deploy workloads or apply Terraform.

## Problem or Limitation

Three recurring problems motivated the work:

1. **Security coverage is domain-specific.** Terraform validation cannot replace a Dockerfile lint or a Kubernetes schema check; a green job in one repository says nothing about another.
2. **A pipeline can be green while findings still need review.** `allow_failure: true` jobs, advisory IaC reports and generated SBOMs must be interpreted differently from blocking gates.
3. **Pinned scanners drift over time.** CI files can contain versions in nested includes; checking the current developer checkout instead of merged `origin/main` can report the wrong baseline, especially with dirty worktrees.

The implementation therefore combined clear repository-level responsibilities, a small number of explicitly enforced security gates and a separate version-monitoring workflow.

## Decision and Responsibility Model

```text
Change → repository-specific MR checks → reviewed merge
                                    |
                      no automatic AWS / Kubernetes mutation

Separate application release → build → image scan → registry push
                             → reviewed chart / GitOps update → Argo CD

Independent local monitor → read origin/main CI definitions
                          → compare pinned versions to releases
                          → JSON review report (no automatic update)
```

A **blocking** check must succeed for the MR to pass. An **advisory** check runs and reports, but can be accepted temporarily while findings are triaged. Neither a successful advisory job nor an SBOM artifact is a claim of zero security issues.

## Implementation

### Pipeline 1 — `terraform-modules`: reusable infrastructure contracts

This repository is a module library, not an AWS environment to be applied by CI. Its baseline covers the reusable module inventory and Terraform syntax/validation for **ten modules**, while retaining separate versioning of modules.

| Control | What it checks | Enforcement / output |
|---|---|---|
| `fmt` | Terraform formatting across the reusable modules; keeps HCL review diffs predictable | Blocking |
| `validate` | `terraform init -backend=false` followed by `terraform validate` for each module; validates configuration without initializing a remote state backend | Blocking; provider downloads may occur, but no AWS deployment |
| `terraform:security-contract` | Ensures expected module/CI boundaries and rejects infrastructure `plan`, `apply` or `destroy` operations in this validation workflow | Blocking |
| `terraform:secrets` | Trivy secret detection on repository content | Blocking |
| `terraform:iac-audit` | Trivy Terraform/IaC analysis, with HIGH/CRITICAL findings captured as reviewable evidence | **Advisory**; findings require triage |

**Why this matters:** a module can validate without being safe to deploy; formatting and `validate` address syntax/contracts, whereas secret and IaC checks cover a different risk class. Conversely, an advisory IaC finding does not grant permission to change infrastructure automatically. One-off `migration-scenarios` were intentionally outside this reusable-module gate.

### Pipeline 2 — `jsappinf-platform`: AWS/EKS platform composition

This repository composes the infrastructure stack and platform runtime. The CI baseline validates the Terraform source **without invoking a real AWS plan, apply or destroy**.

| Control | What it checks | Enforcement / output |
|---|---|---|
| Terraform format | Consistent HCL and reviewable changes | CI validation |
| Terraform init / validate | Initializes the platform stack in validation mode and verifies the configuration contract | CI validation; not a deployment |
| Security contract | Prevents unsafe CI responsibility expansion, such as an unreviewed infrastructure lifecycle action | CI security check |
| Trivy secret scan | Detects credential-shaped material committed to the repository | CI security check |
| IaC audit | Flags potentially risky infrastructure configuration (for example, security-hardening concerns) | Findings documented for follow-up, not automatically "fixed" |

The baseline's merge-request run had five successful CI jobs, while **separate IaC findings** were explicitly recorded for a later hardening review. They are not presented as remediated merely because the pipeline passed. This distinction is particularly important with ephemeral DEV infrastructure and sensitive local `.tfvars` that must never be published.

### Pipeline 3 — `jsappinf-gitops`: desired-state configuration

This repository stores environment manifests, Argo CD Applications and Helm values. Its security baseline is concerned with **what would be declared as desired state**, not with applying it to a live cluster during a code scan.

| Control area | Practical verification purpose | Evidence / boundary |
|---|---|---|
| Repository configuration checks | Catch malformed or inconsistent GitOps definitions before they are reconciled by Argo CD | GitOps validation belongs here; exact job-specific rule set is not reproduced in this public review |
| Trivy scanning | Inspect tracked configuration for issues detectable by the configured scanner | The merged CI source was found and `aquasec/trivy:0.76.2` was inventoried |
| Change review | Keep DEV/STAGE/PROD values and application promotion changes traceable | Argo CD sync and cluster readiness are separate runtime checks |

**Why this matters:** passing a repository scan does not prove that Argo CD will report `Healthy` after reconciliation. Network reachability, secrets resolution and cluster-specific CRDs require runtime verification. The DevSecOps work did not claim to exercise those dependencies.

### Pipeline 4 — `helmchartsappjs`: Kubernetes application packaging

Helm charts are executable deployment templates. Source YAML that appears valid can still produce invalid Kubernetes manifests after templating, so chart validation belongs at this layer.

| Control area | What it is intended to catch | Evidence / boundary |
|---|---|---|
| Helm/chart validation | Chart structure, templates and values-related configuration errors | Repository-specific CI validation; this review does not claim a complete values/cluster matrix |
| `kubeconform` | Rendered Kubernetes API resource schema problems, subject to available schemas and CRD support | Tool detected in merged CI source; schema checks are not live-cluster tests |
| Trivy scanning | Security issues in chart/configuration content covered by the CI scan | `aquasec/trivy:0.76.2` pin detected |
| Release boundary | Prevent validation jobs from silently changing GitOps-managed chart versions | Version promotion remains a separate reviewed action |

This repository had **seven CI-related tracked files** in the monitor inventory. That is a file inventory, not a count of executed validation jobs. Helm lint/schema checks complement rather than replace functional deployment tests.

### Pipeline 5 — `JSAPP`: four Node.js microservices

The application repository contains UI, user, product and order services. The DevSecOps baseline was **additive**: it introduced dedicated security jobs alongside the existing service, DB lifecycle and release-helper validation without replacing the semantic-version release process.

| Job / control | What it does | MR enforcement |
|---|---|---|
| `validate_service_merge_request` (per service) | Checks the changed service against the repository's MR/contract validation rules | Blocking in the supplied MR |
| `validate_db_lifecycle` | Tests/validates DB lifecycle helpers and expected behavior | Blocking |
| `validate_release_helpers` | Checks scripts used in semantic-version release handling | Blocking |
| `devsecops:gitleaks` | Scans Git history for credential-shaped material; `--redact` prevents raw values in scanner output | **Blocking** (`allow_failure: false`) |
| `devsecops:semgrep` | Runs configured static-analysis rules against application source; produces JSON report for review | **Advisory** (`allow_failure: true`) during rollout |
| `devsecops:hadolint` | Lints application Dockerfiles against Docker build/container best practices | **Advisory** (`allow_failure: true`) during rollout |
| `devsecops:sbom` | Uses Trivy to generate a **CycloneDX SBOM** of project dependencies and artifacts | **Blocking** (`allow_failure: false`); downloadable artifact |

Pinned scanner images include Gitleaks `8.28.0`, Semgrep `1.178.0`, Hadolint `2.12.0-alpine` and Trivy `0.76.2`. The repository also contains an older Trivy `0.76.1` reference in its separate CI definitions; the monitor reports both for review rather than silently treating all occurrences as the same active job.

The **release path** remains distinct from these merge-request checks: semantic-version validation → service image build → image scan → approved push to ECR → separate Helm/GitOps version update → Argo CD reconciliation. An MR security run does not itself mean a container was built, pushed or deployed.

Four additional **local `npm audit`** evaluations (one per service) passed at the time of assessment. They are point-in-time dependency evidence, not a substitute for continually maintained CI security gates.

### Example: investigating a Gitleaks detection without weakening the scanner

The history scan detected a `generic-api-key` pattern in an HTTP request example for `POST /orders`. Targeted inspection located the **`Idempotency-Key`** header rather than a GitLab access token. Idempotency keys identify repeated requests; they are not inherently authentication credentials. The original masked value's provenance was not independently established.

The fix preserved the API documentation with a clearly illustrative `Idempotency-Key: order-create-example-001`, then limited the historical Gitleaks exception to **one fingerprint**. No file-wide, rule-wide or history-wide ignore was used. Local Git-history and working-tree scans subsequently passed, as did the GitLab Gitleaks job. The decision was deliberately narrow: if independent evidence later identifies a real exposed credential, the historical exception would not replace revocation or incident response.

### Central tool-version monitor — how it works

A separate Python utility is the maintenance layer for these **five different pipelines**. It is an **inventory and review aid**, not a security scanner or an automated update bot.

| Step | Implementation | Why it matters |
|---|---|---|
| 1. Discover repositories | Uses a configured list of the five local Git repositories | Restricts the inventory to the relevant project |
| 2. Select source | Prefers `refs/remotes/origin/main`; flags fallback to local `main` or `HEAD` | Prevents dirty feature branches from being mistaken for merged CI |
| 3. Read CI definitions | Uses `git ls-tree` and `git show` to inspect tracked files, including nested CI configs and includes | Does not need to check out or modify the source tree |
| 4. Extract tools and pins | Recognizes images for Terraform, Trivy, Gitleaks, Semgrep, Hadolint and other configured tools | Shows what versions are referenced across repositories |
| 5. Optionally compare releases | Uses public GitHub release metadata with `--online` | Identifies **newer-version review candidates**, not proven vulnerabilities |
| 6. Report | Prints per-repository inventory and writes JSON when `--output` is provided | Makes evidence reusable for an upgrade backlog |

The command used in the local maintenance workspace was:

```bash
mkdir -p "$HOME/Project/_local-reports/devsecops-version-monitor"
py -3.12 "$HOME/Project/_devsecops_batch_package/today-closeout/scripts/monitor_versions.py" \
  --base "$HOME/Project" \
  --online \
  --output "$HOME/Project/_local-reports/devsecops-version-monitor/versions.json"
```

The inventory also reports each worktree as `CLEAN` or `DIRTY` **without changing it**. It does not run `git fetch`: `origin/main` is the *locally known* remote-tracking ref, so freshness of that ref must be managed separately. Release lookups may be unavailable or rate-limited; in that case the upstream result is **unverified**, not a safe-to-upgrade decision.

#### Example version-review report

During the 19 November 2026 run, all five repositories were discovered and the monitor surfaced six version-review items:

| Pinned example | Newer release observed | Required action |
|---|---|---|
| Terraform `1.17.0` (module and platform CI) | `1.17.2` | Review compatibility and validate both pipelines before updating |
| Trivy `0.76.1` (one JSAPP reference) | `0.76.2` | Identify the active CI reference before removing or updating the older pin |
| Hadolint `2.12.0-alpine` | `2.15.1` | Review rule changes and assess Dockerfile findings |
| Semgrep `1.178.0` | `1.186.0` | Review scanner/rule compatibility and CI output |
| Gitleaks `8.28.0` | `8.30.1` | Revalidate history-scan behavior and the fingerprint exception |

These are **observations from that date**, not a live current-version check. Changing a tool version is a separate reviewed change with its own test evidence. The monitor does **not** create branches, edit CI files, open merge requests, schedule itself, alter credentials or deploy infrastructure.

## Validation

The five repository baselines were merged through their respective review workflows. For the JSAPP security pipeline, exported GitLab job metadata confirmed that **Gitleaks, Semgrep, Hadolint, SBOM and the existing merge-request validations executed successfully**. The SBOM artifact was present. Semgrep and Hadolint were still advisory, so their status alone does not establish that the source has no warnings.

The local version-monitor run found tracked CI files in all five repositories and produced a JSON inventory with review candidates. No AWS or Kubernetes deployment was performed as part of the validation described here.

**Evidence limits:** full advisory scanner reports and the detailed contents of the generated CycloneDX SBOM were not independently reviewed in this case study. A successful job proves execution under its configured conditions, not future vulnerability absence. Live DEV runtime and DAST testing are separate engineering scopes.

## Result

The practical outcome is a **repository-specific security review layer** instead of a generic, one-size-fits-all pipeline:

- Infrastructure modules and AWS platform CI distinguish Terraform correctness, secrets and IaC-risk reporting.
- GitOps and Helm pipelines focus on configuration and Kubernetes deployment contracts, without running clusters during source validation.
- Application CI adds history secret scanning, static analysis, Dockerfile checks and SBOM creation while preserving service validation and release boundaries.
- A read-only monitor identifies potential scanner-version drift across all five merged CI configurations for later human review.

## Current Status and Limits

**Implemented:** baseline CI controls in five repositories; Gitleaks/Trivy SBOM blocking and Semgrep/Hadolint advisory in JSAPP; a local, read-only version monitor.

**Not claimed:** automated scanner upgrades, a scheduled central version-monitor job, validated SBOM contents, zero outstanding IaC warnings, guaranteed Kubernetes runtime health, production penetration testing or full DAST coverage. No public GitHub repository contains the private CI source or runtime credentials.

## Technologies

GitLab CI/CD, Git, Terraform, Trivy, Gitleaks, Semgrep CE, Hadolint, CycloneDX, Node.js/npm, Docker, Helm, kubeconform, Argo CD and AWS EKS (platform context, not a runtime exercised by this case study).

## Key Lessons

- **Different codebases need different gates.** Terraform module validation, Kubernetes manifest validation and application source scanning answer different questions.
- **Explain the control, not merely the tool name.** A scan only makes sense when its failure policy, report and engineering owner are known.
- **Advisory is not invisible.** A green pipeline may still carry reviewable IaC or static-analysis findings.
- **`BLOCKED` is not `PASS`.** A local scanner that could not download did not inspect the source.
- **Monitor the merged baseline.** Read CI definitions from `origin/main`, record stale/fallback refs and keep developer worktrees untouched.
- **Keep upgrades and deployments as deliberate decisions.** Version drift should open a review, never silently alter infrastructure or releases.
