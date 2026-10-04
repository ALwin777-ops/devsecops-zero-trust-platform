# DevSecOps Zero Trust Platform — Security Controls

## 1. Purpose

This document describes the security controls implemented in the DevSecOps Zero Trust Platform.

The purpose is to document:

* What security control was implemented
* Which technology provides the control
* What security risk the control addresses
* How the control is implemented
* What evidence is available
* What limitations or exceptions exist

The controls are organized according to the software delivery lifecycle and runtime security architecture.

---

# 2. Security Control Summary

| ID    | Security Area            | Control                               | Technology            | Status                      |
| ----- | ------------------------ | ------------------------------------- | --------------------- | --------------------------- |
| SC-01 | Source Security          | Secret detection                      | Gitleaks              | Implemented                 |
| SC-02 | CI/CD Security           | Automated security pipeline           | GitHub Actions        | Implemented                 |
| SC-03 | IaC Security             | Kubernetes configuration scanning     | Checkov               | Implemented                 |
| SC-04 | Container Security       | Container configuration hardening     | Docker                | Implemented                 |
| SC-05 | Vulnerability Management | Container vulnerability scanning      | Trivy                 | Implemented                 |
| SC-06 | SBOM                     | Software inventory generation         | Trivy                 | Implemented                 |
| SC-07 | Kubernetes Security      | Non-root workload execution           | Kubernetes            | Implemented where supported |
| SC-08 | Kubernetes Security      | Privilege escalation prevention       | Kubernetes            | Implemented                 |
| SC-09 | Kubernetes Security      | Linux capability restriction          | Kubernetes            | Implemented                 |
| SC-10 | Kubernetes Security      | Seccomp runtime profile               | Kubernetes            | Implemented                 |
| SC-11 | Network Security         | Default-deny network model            | NetworkPolicy         | Implemented                 |
| SC-12 | Network Security         | Explicit application traffic rules    | NetworkPolicy         | Implemented                 |
| SC-13 | Secrets Security         | Centralized secret storage            | HashiCorp Vault       | Implemented                 |
| SC-14 | Secrets Security         | Kubernetes-based Vault authentication | Vault Kubernetes Auth | Implemented                 |
| SC-15 | Secrets Security         | Runtime secret injection              | Vault Agent Injector  | Implemented                 |
| SC-16 | Policy Security          | Kubernetes policy-as-code             | Kyverno               | Implemented                 |
| SC-17 | Monitoring               | Kubernetes metrics                    | Prometheus            | Implemented                 |
| SC-18 | Monitoring               | Security/availability alerting        | Grafana               | Implemented                 |
| SC-19 | Runtime Security         | Endpoint event collection             | Wazuh Agent           | Implemented                 |
| SC-20 | Runtime Security         | Event analysis and indexing           | Wazuh                 | Implemented                 |
| SC-21 | Runtime Security         | Archived event pipeline               | Wazuh/Filebeat        | Implemented                 |
| SC-22 | Evidence                 | Security test artifacts               | Reports/SBOMs         | Implemented                 |

---

# 3. Source Code Security

## SC-01 — Secret Detection

### Objective

Prevent accidental exposure of credentials, API keys, tokens, passwords, and other sensitive information through source control.

### Technology

Gitleaks

### Implementation

Gitleaks was used to scan the repository and its Git history for potential secrets.

The repository was scanned across:

```text
19 commits
```

The completed scan produced:

```text
Detected leaks: 0
```

### Security Benefit

Secret detection reduces the risk of credentials being accidentally committed to the source repository.

This provides an early security control before application code reaches the build and deployment stages.

### Evidence

Repository security evidence includes:

```text
gitleaks-report.json
```

### Status

**Implemented and validated.**

---

# 4. CI/CD Security

## SC-02 — Automated Security Pipeline

### Objective

Automatically execute security checks when changes are introduced into the repository.

### Technology

GitHub Actions

### Implementation

The repository contains a security workflow:

```text
.github/workflows/devsecops-security.yml
```

The workflow performs:

```text
Git Checkout
     |
     +----> Gitleaks
     |
     +----> Checkov
```

The workflow is triggered for:

* Pushes to `main`
* Pushes to `security/**`
* Pull requests targeting `main`

The workflow uses:

```yaml
permissions:
  contents: read
```

This limits the default GitHub Actions repository permission.

### Security Benefit

Automated security checks reduce dependence on manual security validation.

Security testing becomes part of the software delivery process.

### Status

**Implemented.**

---

# 5. Infrastructure and Configuration Security

## SC-03 — Kubernetes and Configuration Scanning

### Objective

Identify insecure infrastructure and configuration settings before deployment.

### Technology

Checkov

### Implementation

Checkov was used to evaluate:

* Kubernetes manifests
* Dockerfiles
* GitHub Actions workflows

Final validation results:

```text
Kubernetes:
    Passed: 263
    Failed: 0
    Skipped: 11

Dockerfile:
    Passed: 113
    Failed: 0
    Skipped: 0

GitHub Actions:
    Passed: 204
    Failed: 0
    Skipped: 0
```

### Security Benefit

Infrastructure scanning identifies security misconfigurations before they become runtime issues.

### Important Scope Note

The repository does not contain Terraform infrastructure code.

The repository folder name includes `Terraforms`, but the actual project does not currently use `.tf` infrastructure files.

The Checkov `terraform_plan` framework was therefore excluded from the final scan.

### Status

**Implemented and validated.**

---

# 6. Container Security

## SC-04 — Container Hardening

### Objective

Reduce the attack surface and potential impact of container compromise.

### Technology

Docker

### Implemented Controls

The backend container uses:

* Dedicated non-root application user
* Explicit UID/GID
* `allowPrivilegeEscalation: false`
* Dropped Linux capabilities
* `no-new-privileges`
* Read-only root filesystem
* Temporary writable filesystem for `/tmp`

The frontend container also uses an explicit non-root user.

### Backend Identity

The backend container uses:

```text
UID: 100
GID: 101
```

### Frontend Identity

The frontend container uses:

```text
UID: 101
```

### Security Benefit

These controls reduce the privileges available to a compromised container.

For example, running as a non-root user limits the ability of an attacker to perform privileged operations inside the container.

A read-only root filesystem further reduces opportunities for persistence or unauthorized modification.

### Status

**Implemented.**

---

# 7. Vulnerability Management

## SC-05 — Container and Kubernetes Vulnerability Scanning

### Objective

Identify known vulnerabilities and security issues in container images and Kubernetes resources.

### Technology

Trivy

### Implementation

Trivy was used for:

* Container image scanning
* Kubernetes security scanning
* Node scanning
* Security assessment

The Kubernetes environment was scanned using:

```text
trivy k8s kind-devsecops-lab --report summary --timeout 15m
```

The completed scan evaluated:

```text
324 / 324 resources
```

Node scanning was enabled.

### Security Benefit

Vulnerability scanning provides visibility into known security weaknesses within the application supply chain and runtime configuration.

### Evidence

Trivy evidence is stored in the repository, including:

```text
final_report_trivy/
mysql-current-trivy.json
```

### Status

**Implemented and validated.**

---

# 8. Software Bill of Materials

## SC-06 — SBOM Generation

### Objective

Maintain visibility into the software components included in application images.

### Technology

Trivy / CycloneDX

### Evidence

The repository contains SBOM files including:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

The backend SBOM uses CycloneDX format.

### Security Benefit

An SBOM supports:

* Software inventory
* Dependency visibility
* Vulnerability investigation
* Supply-chain analysis
* Security reporting

### Important Note

CycloneDX SBOM generation and vulnerability scanning are treated as separate operations in Trivy.

### Status

**Implemented.**

---

# 9. Kubernetes Workload Security

## SC-07 — Non-Root Workload Execution

### Objective

Prevent application workloads from unnecessarily running as the root user.

### Technology

Kubernetes SecurityContext

### Implementation

The backend and frontend workloads are configured to run as non-root users.

Backend:

```text
runAsNonRoot: true
UID: 100
GID: 101
```

Frontend:

```text
runAsNonRoot: true
UID: 101
```

### Security Benefit

Non-root execution reduces the privileges available to an attacker who successfully compromises an application container.

### Limitation

The MySQL workload is an intentional exception.

The official MySQL 8.4 container requires root-level behavior during its initialization/runtime process in this local environment.

An attempt to enforce the same non-root restriction caused:

```text
setgid: Operation not permitted
```

The exception is documented separately in:

```text
docs/security-exceptions.md
```

### Status

**Implemented for supported workloads; documented exception for MySQL.**

---

# 10. Privilege Escalation Prevention

## SC-08 — Prevent Privilege Escalation

### Objective

Prevent processes inside containers from gaining additional privileges.

### Technology

Kubernetes SecurityContext

### Implementation

Application workloads use:

```yaml
allowPrivilegeEscalation: false
```

### Security Benefit

This reduces the ability of processes inside a container to escalate their privileges through Linux privilege mechanisms.

### Status

**Implemented.**

---

# 11. Linux Capability Restriction

## SC-09 — Capability Dropping

### Objective

Reduce unnecessary Linux capabilities available to application containers.

### Technology

Docker / Kubernetes SecurityContext

### Implementation

The backend workload drops unnecessary Linux capabilities.

The container configuration also uses capability restrictions as part of the hardened runtime.

MySQL retains only the capabilities compatible with the official image's runtime behavior.

### Security Benefit

Linux capabilities provide a more granular privilege model than simply treating a process as root.

Removing unnecessary capabilities reduces the attack surface.

### Status

**Implemented.**

---

# 12. Seccomp

## SC-10 — Seccomp RuntimeDefault

### Objective

Restrict the system calls available to containers.

### Technology

Kubernetes Seccomp

### Implementation

Application workloads use:

```yaml
seccompProfile:
  type: RuntimeDefault
```

Kyverno also validates the presence of the required seccomp configuration.

### Security Benefit

Seccomp reduces the set of Linux system calls available to containers and can limit exploitation techniques that depend on dangerous system calls.

### Status

**Implemented and policy-validated.**

---

# 13. Network Security

## SC-11 — Default-Deny Network Model

### Objective

Prevent unrestricted network communication between Kubernetes workloads.

### Technology

Kubernetes NetworkPolicy

### Implementation

A default-deny ingress policy is implemented.

The network architecture follows:

```text
Default:
    DENY

Required traffic:
    Explicitly ALLOW
```

### Security Benefit

A compromised workload cannot automatically communicate with every other workload in the namespace.

This reduces lateral movement opportunities.

### Status

**Implemented.**

---

# 14. Application Network Segmentation

## SC-12 — Explicit Application Traffic Rules

### Objective

Allow only required application communication.

### Technology

Kubernetes NetworkPolicy

### Approved Communication

```text
Frontend → Backend
TCP 8080

Backend → MySQL
TCP 3306

Backend → Vault
Vault API

Backend → DNS
DNS resolution
```

Additional ingress policies control access to the application services.

### Security Benefit

The application follows an explicit communication model rather than unrestricted pod-to-pod connectivity.

### Status

**Implemented.**

---

# 15. Secrets Management

## SC-13 — Centralized Secret Storage

### Objective

Separate sensitive credentials from application deployment configuration.

### Technology

HashiCorp Vault

### Implementation

Backend database credentials are stored in Vault at:

```text
secret/data/devsecops/backend
```

The application does not require the database password to be embedded directly into its container image.

### Security Benefit

Centralized secret storage provides:

* Access control
* Secret separation
* Centralized management
* Reduced credential exposure
* Better auditability

### Status

**Implemented.**

---

# 16. Kubernetes Authentication to Vault

## SC-14 — Workload Identity for Vault

### Objective

Authenticate workloads to Vault using Kubernetes workload identity.

### Technology

Vault Kubernetes Authentication

### Implementation

The backend uses:

```text
Service Account:
backend-sa

Namespace:
devsecops

Vault Role:
devsecops-backend
```

The Vault role is bound to the backend service account.

The role has a limited token lifetime.

### Security Benefit

The backend does not require a static Vault credential embedded in the deployment.

Authentication is tied to the Kubernetes workload identity.

### Status

**Implemented and validated.**

---

# 17. Runtime Secret Injection

## SC-15 — Vault Agent Injector

### Objective

Provide application secrets to the workload at runtime.

### Technology

Vault Agent Injector

### Implementation

Vault Agent Injector annotations are configured on the backend Deployment.

The injector provides the required secret material to the backend pod.

The resulting secret files were validated inside the backend container.

The injected paths included:

```text
/run/secrets/db_user
/run/secrets/db_password
```

The actual secret values are intentionally not stored in this documentation.

### Validation

The backend successfully accessed the database after Vault secret injection.

The backend API returned:

```text
[]
```

from:

```text
/api/user
```

This demonstrated successful application-to-database operation using the Vault-managed credentials.

### Status

**Implemented and validated.**

---

# 18. Kubernetes Policy-as-Code

## SC-16 — Kyverno Security Policies

### Objective

Continuously validate Kubernetes workloads against defined security requirements.

### Technology

Kyverno

### Implemented Policies

The environment includes validation for:

```text
require-run-as-nonroot
require-no-privilege-escalation
require-seccomp-runtime-default
require-resource-requests-limits
```

The policies are configured as ValidatingPolicies.

### Security Benefit

Security requirements become machine-readable policies instead of relying entirely on manual review.

### Status

**Implemented.**

---

# 19. Policy Reporting

Kyverno generates policy reports showing which workloads satisfy or violate individual controls.

For example, the MySQL workload currently reports:

```text
PASS:
    require-no-privilege-escalation
    require-resource-requests-limits
    require-seccomp-runtime-default

FAIL:
    require-run-as-nonroot
```

The single failure is intentional and documented as a security exception.

### Security Benefit

Policy reports provide continuous security evidence and make exceptions visible.

### Status

**Implemented and validated.**

---

# 20. Monitoring

## SC-17 — Kubernetes Metrics

### Objective

Provide operational visibility into Kubernetes workloads.

### Technology

Prometheus

### Implementation

Prometheus collects Kubernetes metrics from the environment.

The monitoring stack runs in:

```text
monitoring
```

namespace.

### Security Benefit

Monitoring provides visibility into:

* Workload availability
* Pod restarts
* Deployment health
* Resource behavior
* Operational anomalies

### Status

**Implemented.**

---

# 21. Grafana Alerting

## SC-18 — Workload Security and Availability Alerts

### Objective

Generate alerts when important workload conditions occur.

### Technology

Grafana

### Implemented Alerts

#### BackendDeploymentReplicasLow

Detects when the number of available backend replicas falls below the expected threshold.

Purpose:

* Detect reduced redundancy
* Detect deployment problems
* Detect potential service degradation

---

#### BackendPodRestartDetected

Detects backend pod restart activity.

Purpose:

* Identify unexpected container crashes
* Detect workload instability
* Support investigation

---

#### BackendDeploymentUnavailable

Detects unavailable backend replicas.

Purpose:

* Detect backend availability problems
* Provide early operational warning

---

# 22. Runtime Security

## SC-19 — Endpoint Security Monitoring

### Objective

Collect endpoint security events for runtime detection and investigation.

### Technology

Wazuh Agent

### Implementation

A Windows Wazuh agent was enrolled using the logical agent identity:

```text
USER-WINDOWSS
```

The agent successfully communicated with the Wazuh manager.

### Security Benefit

Endpoint telemetry provides visibility into security-relevant events occurring outside the Kubernetes application environment.

### Status

**Implemented and validated.**

---

# 23. Wazuh Event Analysis

## SC-20 — Runtime Security Analysis

### Objective

Analyze endpoint events and make them available for security investigation.

### Technology

Wazuh Manager

### Architecture

```text
Windows Endpoint
      |
      v
Wazuh Agent
      |
      v
Wazuh Manager
```

The Wazuh manager receives and processes events from the endpoint.

### Security Benefit

This provides a dedicated runtime security monitoring layer that complements preventive controls.

### Status

**Implemented.**

---

# 24. Wazuh Archive Pipeline

## SC-21 — Full Event Archiving and Indexing

### Objective

Preserve collected endpoint events and make them searchable through the Wazuh dashboard.

### Technologies

* Wazuh Manager
* Filebeat
* Wazuh Indexer
* Wazuh Dashboard

### Implementation

JSON event archiving is enabled.

Events are written to:

```text
/var/ossec/logs/archives/archives.json
```

Filebeat is configured to forward Wazuh archives.

The resulting indexed data is available through:

```text
wazuh-archives-*
```

### Validation

A Windows application event was generated and successfully observed in the Wazuh archive index.

This validated the complete pipeline:

```text
Windows Event
      ↓
Wazuh Agent
      ↓
Wazuh Manager
      ↓
Wazuh Archives
      ↓
Filebeat
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```

### Status

**Implemented and end-to-end validated.**

---

# 25. Security Evidence

## SC-22 — Security Evidence and Reporting

### Objective

Maintain evidence demonstrating that implemented controls were tested.

### Evidence Categories

### Gitleaks

```text
gitleaks-report.json
```

### Checkov

```text
checkov-results.json
checkov-k8s-results.json
checkov-k8s-results.xml
```

### SBOM

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

### Trivy

```text
final_report_trivy/
mysql-current-trivy.json
```

### Kubernetes / Kyverno

Policy reports and workload configuration provide evidence of policy evaluation.

### Grafana

Alert rules provide monitoring evidence.

### Wazuh

The indexed Windows event provides runtime security telemetry evidence.

### Status

**Implemented.**

---

# 26. Defense-in-Depth Mapping

The implemented controls provide multiple layers of protection.

| Security Layer   | Controls                   |
| ---------------- | -------------------------- |
| Source           | Gitleaks                   |
| CI/CD            | GitHub Actions             |
| Configuration    | Checkov                    |
| Container        | Docker hardening           |
| Vulnerability    | Trivy                      |
| Supply Chain     | SBOM                       |
| Workload         | Kubernetes SecurityContext |
| Network          | NetworkPolicies            |
| Secrets          | Vault                      |
| Policy           | Kyverno                    |
| Monitoring       | Prometheus                 |
| Alerting         | Grafana                    |
| Runtime Security | Wazuh                      |

This prevents the security model from depending on a single control.

---

# 27. Preventive Controls

The following controls primarily prevent insecure configurations or unauthorized behavior:

* Secret detection
* CI security checks
* Checkov scanning
* Container hardening
* Non-root execution
* Privilege escalation prevention
* Capability restrictions
* Seccomp
* NetworkPolicies
* Vault access control
* Kyverno policies

---

# 28. Detective Controls

The following controls provide detection or visibility:

* Trivy scanning
* Prometheus metrics
* Grafana alerts
* Wazuh endpoint monitoring
* Wazuh event analysis
* Wazuh event archives

---

# 29. Security Evidence Controls

The following artifacts support security verification:

* Gitleaks report
* Checkov results
* Trivy reports
* SBOM files
* Kyverno PolicyReports
* Grafana alert configurations
* Wazuh indexed events
* CI/CD workflow results

---

# 30. Control Relationships

The security controls are not independent.

They form a layered security pipeline:

```text
                 SOURCE
                   |
                   v
              Gitleaks
                   |
                   v
                  CI
                   |
                   v
                Checkov
                   |
                   v
               Docker
              Hardening
                   |
                   v
                Trivy
                   |
                   v
             Kubernetes
              Security
                   |
        +----------+----------+
        |                     |
        v                     v
 NetworkPolicies           Kyverno
        |                     |
        +----------+----------+
                   |
                   v
                 Vault
                   |
                   v
              Application
                   |
        +----------+----------+
        |                     |
        v                     v
   Prometheus               Wazuh
        |                     |
        v                     v
     Grafana             Runtime Security
```

---

# 31. Security Control Validation Strategy

Each major security control was validated using one or more of the following approaches:

### Static Validation

Used for:

* Gitleaks
* Checkov
* Dockerfile configuration
* Kubernetes manifests
* GitHub Actions configuration

### Runtime Validation

Used for:

* Kubernetes security contexts
* NetworkPolicies
* Vault secret injection
* Application-to-database communication
* Prometheus monitoring
* Grafana alerts
* Wazuh agent communication

### Evidence Validation

Used for:

* Trivy reports
* SBOMs
* Kyverno PolicyReports
* Wazuh archives
* Grafana alert configuration
* CI/CD execution results

---

# 32. Risk Reduction Model

The security architecture reduces risk through multiple stages.

```text
Potential Security Issue
          |
          v
   Source/CI Detection
          |
          v
   Configuration Review
          |
          v
    Container Hardening
          |
          v
   Kubernetes Controls
          |
          v
 Network Segmentation
          |
          v
   Secret Protection
          |
          v
   Policy Validation
          |
          v
      Monitoring
          |
          v
 Runtime Detection
          |
          v
    Investigation
```

This layered approach reduces both the probability and potential impact of successful attacks.

---

# 33. Security Exceptions

Not every security control can be applied identically to every workload.

The primary documented exception is the MySQL non-root requirement.

The MySQL container is currently allowed to run as root because forcing:

```text
runAsNonRoot: true
```

caused an initialization/runtime failure:

```text
setgid: Operation not permitted
```

Rather than disabling security controls globally, the project retains compatible controls and documents the exception.

The complete explanation is available in:

```text
docs/security-exceptions.md
```

---

# 34. Security Design Principles

The implemented controls support the following principles.

## Least Privilege

Workloads and identities receive only the permissions required for their intended function.

## Defense in Depth

Multiple security layers are used to reduce dependence on any single control.

## Zero Trust

Network communication is explicitly allowed based on workload requirements.

## Shift Left

Security checks are introduced during source control and CI/CD.

## Continuous Validation

Kubernetes workloads are continuously evaluated through policy and monitoring.

## Secrets Separation

Sensitive credentials are managed outside the application source and container image.

## Runtime Visibility

Preventive controls are complemented by runtime security monitoring.

## Security as Code

Security controls are represented through version-controlled configuration, policies, manifests, and workflows.

---

# 35. Control Coverage Summary

The platform provides security coverage across:

```text
Source Code
      ✓
CI/CD
      ✓
Container Security
      ✓
Vulnerability Management
      ✓
SBOM
      ✓
Kubernetes Security
      ✓
Network Security
      ✓
Secrets Management
      ✓
Policy Enforcement
      ✓
Monitoring
      ✓
Runtime Security
      ✓
Security Evidence
      ✓
```

---

# 36. Final Assessment

The DevSecOps Zero Trust Platform implements a layered security architecture spanning the software delivery pipeline and runtime environment.

The implementation demonstrates practical use of:

* Gitleaks for secret detection
* Checkov for configuration security
* Trivy for vulnerability and SBOM analysis
* Docker security hardening
* Kubernetes security contexts
* NetworkPolicies for Zero Trust segmentation
* HashiCorp Vault for runtime secrets
* Kyverno for policy-as-code
* Prometheus and Grafana for monitoring
* Wazuh for runtime security monitoring
* GitHub Actions for automated security validation

The controls are supported by concrete evidence including scan results, SBOMs, policy reports, monitoring configurations, CI workflows, and validated Wazuh events.

The architecture intentionally documents limitations and exceptions rather than presenting the laboratory environment as a production deployment.

This approach demonstrates security engineering through implementation, validation, evidence collection, and risk-based decision making.
