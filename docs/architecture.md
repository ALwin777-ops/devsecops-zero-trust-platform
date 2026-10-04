# DevSecOps Zero Trust Platform — Architecture

## 1. Project Overview

The DevSecOps Zero Trust Platform is a security-focused engineering project built around a containerized Spring Boot, React, and MySQL application.

The project demonstrates how security controls can be integrated across the software delivery lifecycle and Kubernetes runtime environment.

The implementation combines:

* Source-code security
* Secret detection
* Container hardening
* Container vulnerability scanning
* Kubernetes security
* Kubernetes policy-as-code
* Zero Trust network segmentation
* Runtime secrets management
* Infrastructure and configuration security
* CI/CD security
* Kubernetes monitoring
* Runtime security monitoring and SIEM capabilities

The platform is implemented as a local security lab using Docker, Kubernetes Kind, HashiCorp Vault, Kyverno, Prometheus, Grafana, Wazuh, Trivy, Gitleaks, Checkov, and GitHub Actions.

The project intentionally focuses on DevSecOps and security engineering rather than application feature development.

---

# 2. Architecture Goals

The primary goals of the architecture are:

1. Reduce security weaknesses before deployment.
2. Harden containerized workloads.
3. Apply least-privilege principles to Kubernetes workloads.
4. Reduce unnecessary network communication.
5. Centralize application secrets outside the application image.
6. Enforce Kubernetes security requirements through policy-as-code.
7. Continuously monitor workload health.
8. Provide runtime security visibility through Wazuh.
9. Automate security checks through CI/CD.
10. Produce security evidence that can be reviewed during security assessments.

The architecture follows a defense-in-depth model rather than relying on a single security mechanism.

---

# 3. High-Level Architecture

```text
                         ┌─────────────────────┐
                         │      Developer      │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   GitHub Repository │
                         └──────────┬──────────┘
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │      GitHub Actions CI/CD      │
                    │                                │
                    │  ┌──────────┐   ┌──────────┐ │
                    │  │ Gitleaks │   │ Checkov  │ │
                    │  └──────────┘   └──────────┘ │
                    └───────────────┬────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Security Validation │
                         └──────────┬──────────┘
                                    │
                                    ▼
                ┌────────────────────────────────────────┐
                │          Kubernetes / Kind             │
                │          devsecops-lab                  │
                │                                        │
                │   ┌────────────────────────────────┐   │
                │   │        devsecops namespace      │   │
                │   │                                │   │
                │   │  ┌────────────┐  ┌──────────┐ │   │
                │   │  │  Frontend  │─▶│ Backend  │ │   │
                │   │  │   React    │  │ Spring   │ │   │
                │   │  └────────────┘  └────┬─────┘ │   │
                │   │                       │       │   │
                │   │                       ▼       │   │
                │   │                  ┌─────────┐  │   │
                │   │                  │  MySQL  │  │   │
                │   │                  └─────────┘  │   │
                │   └────────────────────────────────┘   │
                │                                        │
                │              ┌──────────┐              │
                │              │ Kyverno  │              │
                │              └──────────┘              │
                │                                        │
                │              NetworkPolicies           │
                └───────────────────┬────────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                  ┌─────────────┐       ┌─────────────┐
                  │    Vault    │       │ Prometheus  │
                  │   Secrets   │       │  Metrics    │
                  └─────────────┘       └──────┬──────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │   Grafana   │
                                        │  Monitoring │
                                        └─────────────┘


                    Runtime Security Monitoring
                    ────────────────────────────

                  ┌────────────────────────────┐
                  │     Windows Endpoint       │
                  │      ALWIN-WINDOWS         │
                  └─────────────┬──────────────┘
                                │
                                ▼
                       ┌────────────────┐
                       │  Wazuh Agent   │
                       └───────┬────────┘
                               │
                               ▼
                       ┌────────────────┐
                       │ Wazuh Manager  │
                       └───────┬────────┘
                               │
                               ▼
                          ┌──────────┐
                          │ Filebeat │
                          └────┬─────┘
                               │
                               ▼
                       ┌────────────────┐
                       │ Wazuh Indexer  │
                       └───────┬────────┘
                               │
                               ▼
                       ┌────────────────┐
                       │ Wazuh Dashboard│
                       └────────────────┘
```

---

# 4. Platform Components

| Component       | Primary Purpose                                    |
| --------------- | -------------------------------------------------- |
| GitHub          | Source-code and CI/CD repository                   |
| GitHub Actions  | Automated security pipeline                        |
| Gitleaks        | Secret and credential detection                    |
| Checkov         | Kubernetes, Dockerfile and CI/CD security scanning |
| Docker          | Containerization and workload isolation            |
| Trivy           | Vulnerability, Kubernetes and SBOM scanning        |
| Kubernetes Kind | Local Kubernetes environment                       |
| React           | Frontend application                               |
| Spring Boot     | Backend application                                |
| MySQL           | Application database                               |
| HashiCorp Vault | Runtime secret management                          |
| Kyverno         | Kubernetes policy-as-code                          |
| NetworkPolicies | Network segmentation                               |
| Prometheus      | Metrics collection                                 |
| Grafana         | Monitoring and alerting                            |
| Wazuh Agent     | Endpoint security data collection                  |
| Wazuh Manager   | Security analysis and detection                    |
| Filebeat        | Event forwarding                                   |
| Wazuh Indexer   | Security-event indexing and storage                |
| Wazuh Dashboard | Security investigation and visualization           |

---

# 5. Environment Architecture

The main application runs inside a local Kubernetes Kind cluster.

```text
Cluster:
    devsecops-lab

Context:
    kind-devsecops-lab
```

The primary application namespace is:

```text
devsecops
```

Additional security and observability components are deployed into dedicated namespaces:

```text
devsecops
vault
kyverno
monitoring
```

The Wazuh platform is deployed separately in an Ubuntu 24.04.5 WSL environment to keep the resource-heavy SIEM stack separate from the Kubernetes laboratory.

This separation allows the Kubernetes security platform and endpoint monitoring platform to operate independently.

---

# 6. Application Architecture

The application follows a conventional three-tier structure.

```text
                User
                 │
                 ▼
        ┌─────────────────┐
        │ React Frontend  │
        └────────┬────────┘
                 │
                 │ HTTP :8080
                 ▼
        ┌─────────────────┐
        │ Spring Backend  │
        └────────┬────────┘
                 │
                 │ MySQL :3306
                 ▼
        ┌─────────────────┐
        │      MySQL      │
        └─────────────────┘
```

The frontend communicates with the backend.

The backend communicates with MySQL.

The frontend does not directly communicate with MySQL.

This separation is reinforced through Kubernetes NetworkPolicies.

---

# 7. Container Security Architecture

The application is containerized using Docker.

The backend container has been hardened using several Linux container-security controls.

Implemented controls include:

* Dedicated non-root application user
* Explicit UID/GID
* `allowPrivilegeEscalation: false`
* Linux capability dropping
* `no-new-privileges`
* Read-only root filesystem where supported
* Temporary writable filesystem for `/tmp`
* Restricted runtime configuration

The backend application runs using a dedicated application identity rather than the root user.

The frontend container is also configured with an explicit non-root user.

These controls reduce the impact of container compromise.

---

# 8. Container Vulnerability Management

Trivy is used to assess container images and Kubernetes resources.

The project includes security evidence such as:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
mysql-current-trivy.json
final_report_trivy/
```

Software Bill of Materials (SBOM) generation provides visibility into the components contained within application images.

The project also performs Kubernetes security scanning against the Kind cluster.

The completed Kubernetes scan evaluated:

```text
324 / 324 resources
```

with node scanning enabled.

---

# 9. Kubernetes Architecture

The application is deployed into the Kind cluster:

```text
devsecops-lab
```

The main workloads include:

```text
Frontend Deployment
Backend Deployment
MySQL StatefulSet
```

The backend and frontend use dedicated service accounts.

```text
backend-sa
frontend-sa
mysql-sa
```

Security contexts are used to establish workload-level security requirements.

For supported workloads, controls include:

* `runAsNonRoot`
* Explicit user/group IDs
* `allowPrivilegeEscalation: false`
* Seccomp `RuntimeDefault`
* Linux capability restrictions
* Service-account restrictions
* Resource requests and limits

---

# 10. Zero Trust Network Architecture

The Kubernetes environment uses a default-deny network approach.

Instead of allowing unrestricted pod-to-pod communication, required communication paths are explicitly permitted.

The intended application communication flow is:

```text
Frontend
   │
   │ TCP 8080
   ▼
Backend
   │
   │ TCP 3306
   ▼
MySQL
```

The backend also requires controlled communication with Vault.

DNS access is explicitly permitted for workloads that require service discovery.

The resulting model is:

```text
Default
  │
  └── DENY

Explicitly permitted:
  │
  ├── Frontend → Backend : 8080
  ├── Backend → MySQL : 3306
  ├── Backend → Vault
  └── Backend → DNS
```

This limits unnecessary lateral movement between workloads.

If one workload is compromised, NetworkPolicies reduce the attacker's ability to freely communicate with unrelated workloads.

---

# 11. Kubernetes Policy Architecture

Kyverno is used to implement Kubernetes security policies.

The project includes policies covering:

```text
Require runAsNonRoot
Require no privilege escalation
Require seccomp RuntimeDefault
Require resource requests and limits
```

The policies are configured to continuously evaluate workload security posture.

The project uses both enforcement and audit-oriented controls depending on the security requirement.

This provides policy-as-code rather than relying exclusively on manual review.

---

# 12. Secrets Management Architecture

Application secrets are managed through HashiCorp Vault.

The database credentials are stored in Vault under:

```text
secret/data/devsecops/backend
```

The backend authenticates to Vault using Kubernetes authentication.

The authentication chain is:

```text
Backend Pod
     │
     ▼
backend-sa
     │
     ▼
Vault Kubernetes Authentication
     │
     ▼
Vault Role
devsecops-backend
     │
     ▼
Vault Policy
devsecops-backend
     │
     ▼
Vault Secret
secret/data/devsecops/backend
```

Vault Agent Injector then provides the required secret material to the backend workload.

The backend does not need to embed the database credential directly inside the application image.

This provides separation between application deployment and secret storage.

---

# 13. Vault Security Model

The backend Vault role is bound specifically to:

```text
Service Account:
backend-sa

Namespace:
devsecops
```

The role is configured with a limited token lifetime.

The Vault policy provides read access to the required application secret rather than unrestricted Vault access.

This follows the principle of least privilege.

The backend's ability to access Vault is therefore tied to its Kubernetes workload identity.

---

# 14. CI/CD Security Architecture

GitHub Actions provides the automated security pipeline.

The pipeline performs security validation before changes are considered complete.

The current security workflow includes:

```text
Git Push / Pull Request
        │
        ▼
GitHub Actions
        │
        ├───────────────┐
        │               │
        ▼               ▼
   Gitleaks          Checkov
        │               │
        ▼               ▼
Secret Detection   Configuration
                   Security
        │               │
        └───────┬───────┘
                ▼
          Security Result
```

The workflow runs on:

* Pushes to `main`
* Pushes to `security/**`
* Pull requests targeting `main`

The workflow uses read-only repository permissions where applicable.

---

# 15. Gitleaks Architecture

Gitleaks is used to identify potential secrets and credentials committed to the repository.

The repository was scanned across its Git history.

The completed scan produced:

```text
Commits scanned: 19
Approximate repository history size: 187 KB
Detected leaks: 0
```

This provides a source-control security gate against accidental credential exposure.

---

# 16. Checkov Architecture

Checkov is used to identify security misconfigurations in infrastructure and security-related configuration.

The project uses Checkov against:

* Kubernetes manifests
* Dockerfiles
* GitHub Actions workflows

The final local Checkov validation produced:

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

The skipped checks are documented or correspond to intentional test/generated artifacts and environment-specific exceptions.

---

# 17. SBOM Architecture

Software Bill of Materials files are generated for relevant container images.

Current evidence includes:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

The SBOM provides an inventory of software components contained within the image.

This supports:

* Software inventory
* Dependency visibility
* Vulnerability investigation
* Supply-chain analysis
* Security reporting

---

# 18. Monitoring Architecture

Prometheus provides metrics collection for the Kubernetes environment.

Grafana consumes Prometheus metrics and provides visualization and alerting.

The monitoring architecture is:

```text
Kubernetes Workloads
        │
        ▼
   Metrics Sources
        │
        ▼
    Prometheus
        │
        ▼
      Grafana
        │
        ▼
 Security / Operations Alerts
```

The Grafana deployment includes Prometheus and Alertmanager integrations.

---

# 19. Implemented Monitoring Alerts

The project includes workload availability and stability alerts.

### BackendDeploymentReplicasLow

Detects when the backend deployment has fewer available replicas than expected.

Purpose:

* Detect reduced redundancy
* Detect workload availability issues
* Identify deployment failures

---

### BackendPodRestartDetected

Detects backend pod restart activity.

Purpose:

* Identify unexpected crashes
* Identify container instability
* Support operational investigation

---

### BackendDeploymentUnavailable

Detects unavailable backend replicas.

Purpose:

* Identify deployment availability problems
* Provide early warning of service degradation

---

# 20. Runtime Security Architecture

Wazuh provides the runtime security monitoring layer.

The Wazuh deployment consists of:

```text
Windows Endpoint
       │
       ▼
Wazuh Agent
       │
       ▼
Wazuh Manager
       │
       ▼
Filebeat
       │
       ▼
Wazuh Indexer
       │
       ▼
Wazuh Dashboard
```

This follows the standard Wazuh architecture in which agents collect endpoint data, the Wazuh server analyzes events, Filebeat forwards data to the indexer, and the dashboard provides visualization and investigation capabilities.

The Wazuh central components are deployed as an all-in-one laboratory installation, which is appropriate for a small security lab environment rather than a production-scale deployment.

---

# 21. Windows Endpoint Monitoring

A Windows Wazuh agent was enrolled with the following logical identity:

```text
Agent:
ALWIN-WINDOWS
```

The agent successfully connected to the Wazuh manager.

Windows events were collected and processed by the Wazuh platform.

The project also enabled JSON event archiving.

The resulting flow was validated as:

```text
Windows Event
      │
      ▼
Wazuh Agent
      │
      ▼
Wazuh Manager
      │
      ▼
archives.json
      │
      ▼
Filebeat
      │
      ▼
Wazuh Indexer
      │
      ▼
wazuh-archives-*
      │
      ▼
Wazuh Dashboard
```

Wazuh's documentation describes the use of `logall_json` and Filebeat archives forwarding for indexing all collected events in JSON format.

---

# 22. Runtime Security Validation

Runtime event ingestion was validated by generating a Windows application event.

The test demonstrated that an endpoint event could travel through the complete Wazuh data pipeline.

The validation proved:

```text
Windows Event
      ↓
Wazuh Agent
      ↓
Wazuh Manager
      ↓
Wazuh Archive
      ↓
Filebeat
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```

The event was successfully located in the:

```text
wazuh-archives-*
```

index pattern.

This demonstrates end-to-end runtime security telemetry.

---

# 23. Defense-in-Depth Model

The platform uses multiple layers of security.

```text
┌───────────────────────────────────────────┐
│            Source Security                │
│        Gitleaks / Git Security            │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│             CI Security                   │
│       GitHub Actions / Checkov             │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│          Container Security               │
│       Docker Hardening / Trivy             │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│          Kubernetes Security              │
│ Security Context / Seccomp / Capabilities │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│         Network Security                  │
│          NetworkPolicies                  │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│          Secrets Security                 │
│              Vault                       │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│          Policy Security                  │
│             Kyverno                      │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│       Monitoring & Observability          │
│        Prometheus / Grafana               │
└────────────────────┬──────────────────────┘
                     │
┌────────────────────▼──────────────────────┐
│          Runtime Security                 │
│              Wazuh                       │
└───────────────────────────────────────────┘
```

The architecture therefore combines preventive, detective, and monitoring controls.

---

# 24. Security Control Categories

The project can be divided into three broad security categories.

## Preventive Controls

Examples:

* Container hardening
* Kubernetes security contexts
* NetworkPolicies
* Vault access control
* Kyverno policies
* CI security gates
* Secret detection
* Configuration scanning

## Detective Controls

Examples:

* Trivy vulnerability scanning
* Wazuh event monitoring
* Wazuh security analysis
* Prometheus monitoring
* Grafana alerts

## Governance and Evidence

Examples:

* SBOMs
* Checkov reports
* Gitleaks reports
* Trivy reports
* Kubernetes policy reports
* Security exception documentation
* GitHub Actions results

---

# 25. Security Lifecycle

The complete security lifecycle can be represented as:

```text
                    PLAN
                     │
                     ▼
                 DEVELOP
                     │
                     ▼
             SECRET DETECTION
                     │
                     ▼
              CI SECURITY
                     │
                     ▼
           CONTAINER SECURITY
                     │
                     ▼
          KUBERNETES SECURITY
                     │
                     ▼
           ZERO TRUST NETWORK
                     │
                     ▼
          SECRETS MANAGEMENT
                     │
                     ▼
          POLICY ENFORCEMENT
                     │
                     ▼
              MONITORING
                     │
                     ▼
          RUNTIME DETECTION
                     │
                     ▼
              INVESTIGATE
                     │
                     ▼
                IMPROVE
```

This represents the project's DevSecOps approach of integrating security throughout the software lifecycle rather than performing security testing only after deployment.

---

# 26. Architecture Security Principles

## Least Privilege

Applications, service accounts, and Vault identities are given only the permissions required for their function.

## Defense in Depth

Multiple independent security controls are used so that the failure of one control does not result in complete platform compromise.

## Zero Trust

Network communication is explicitly permitted rather than implicitly trusted.

## Shift Left

Security checks are introduced during source control and CI/CD.

## Continuous Validation

Kubernetes workloads and infrastructure are continuously evaluated using policy and monitoring controls.

## Secrets Separation

Sensitive credentials are separated from application source code and managed through Vault.

## Runtime Visibility

Preventive security controls are complemented by runtime telemetry and security monitoring.

## Security as Code

Security configuration is represented through version-controlled configuration, policies, workflows, and manifests.

---

# 27. Environment and Deployment Scope

This project is intentionally implemented as a local security engineering laboratory.

The environment consists of:

```text
Windows Host
    │
    ├── Docker Desktop
    │
    ├── Kubernetes Kind
    │      └── devsecops-lab
    │
    └── Ubuntu 24.04.5 WSL
           └── Wazuh All-in-One
```

The project does not currently include:

* AWS production deployment
* Cloud-hosted Kubernetes
* Cloud OIDC integration
* Production high availability
* Production-scale Wazuh clustering

These capabilities are outside the defined scope of the current project.

---

# 28. Architectural Limitations

This platform is a laboratory implementation and should not be interpreted as a production-ready enterprise deployment without additional engineering.

Known limitations include:

* Kubernetes is running through Kind on a local workstation.
* Wazuh is deployed as an all-in-one laboratory environment.
* Prometheus retention is limited for the local environment.
* Grafana persistence is disabled in the current laboratory configuration.
* The Kubernetes cluster contains a single control-plane node.
* MySQL uses the official MySQL container image and retains root execution because forcing a non-root configuration caused runtime incompatibility in this environment.
* High availability and disaster recovery are not implemented.
* Cloud infrastructure is intentionally outside the current project scope.

These limitations are documented rather than hidden.

---

# 29. Security Exception Philosophy

Security controls are implemented wherever technically compatible with the workload.

When a security control conflicts with the runtime requirements of a component, the project uses a risk-based approach:

1. Attempt the security control.
2. Validate the resulting workload behavior.
3. Identify the incompatibility.
4. Retain compatible compensating controls.
5. Document the exception.
6. Avoid weakening unrelated controls.

The MySQL `runAsNonRoot` requirement is an example of this approach.

The workload retains other applicable security controls, including:

* Seccomp `RuntimeDefault`
* Restricted privilege escalation
* Capability restrictions
* Network segmentation
* Resource controls
* Service-account restrictions

The exception is documented separately in:

```text
docs/security-exceptions.md
```

---

# 30. Architecture Outcome

The completed architecture provides an end-to-end security engineering environment covering:

```text
Source Code
     │
     ▼
Secret Detection
     │
     ▼
CI/CD Security
     │
     ▼
Container Hardening
     │
     ▼
Vulnerability Scanning
     │
     ▼
Kubernetes Security
     │
     ▼
Zero Trust Segmentation
     │
     ▼
Runtime Secrets
     │
     ▼
Policy Enforcement
     │
     ▼
Monitoring
     │
     ▼
Runtime Security
     │
     ▼
Security Investigation
```

The platform demonstrates practical implementation of DevSecOps and Zero Trust principles across both the software delivery pipeline and the runtime environment.

---

# 31. References

* OWASP DevSecOps Guideline
  https://owasp.org/projects/devsecops-guideline

* Wazuh Architecture Documentation
  https://documentation.wazuh.com/current/getting-started/architecture.html

* Wazuh Server Documentation
  https://documentation.wazuh.com/current/getting-started/components/wazuh-server.html

* Wazuh Indexer Documentation
  https://documentation.wazuh.com/current/user-manual/wazuh-indexer/index.html

* Wazuh Index and Archives Documentation
  https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-indices.html

---

# 32. Summary

The DevSecOps Zero Trust Platform integrates security controls across the entire application lifecycle.

The architecture combines:

* Git security
* CI/CD security
* Container hardening
* Vulnerability management
* Kubernetes security
* Network segmentation
* Secrets management
* Policy-as-code
* Monitoring
* Runtime security
* Security evidence

The result is a practical security engineering laboratory demonstrating how preventive, detective, and monitoring controls can work together to protect a containerized Kubernetes application.
