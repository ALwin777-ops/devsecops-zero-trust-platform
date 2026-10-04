# DevSecOps Zero Trust Security Platform

A practical **DevSecOps and Kubernetes security engineering platform** that applies container security, CI/CD security controls, Kubernetes hardening, Zero Trust-aligned network segmentation, secrets management, policy enforcement, monitoring, and endpoint security to a containerized application environment.

The project uses a local **Kind Kubernetes cluster** and a separate **Wazuh deployment in Ubuntu WSL** to demonstrate security controls across the development, deployment, runtime, and monitoring lifecycle.

> **Scope:** This is a local security engineering laboratory and portfolio project. It is not presented as a production-ready enterprise deployment or a complete Zero Trust architecture.

---

## Architecture

```text
                           Developer
                               |
                               v
                         Git Repository
                               |
                               v
                    GitHub Actions Pipeline
                               |
                 +-------------+-------------+
                 |                           |
                 v                           v
             Gitleaks                    Checkov
          Secret Scanning            Security Scanning
                 |                           |
                 +-------------+-------------+
                               |
                               v
                         Docker Images
                               |
                         +-----+-----+
                         |           |
                         v           v
                       Trivy       SBOM
                     Scanning   CycloneDX
                         |
                         v
                 Kubernetes / Kind
                 devsecops-lab
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Frontend        Backend         MySQL
          |              |
          |              +-------> Vault
          |              |        Kubernetes Auth
          |              |
          +----------> Backend
                         |
                  NetworkPolicies
                         |
              +----------+----------+
              |                     |
              v                     v
          Prometheus             Kyverno
              |                 Policy Controls
              v
           Grafana
              |
           Alerts


        Separate Security Monitoring Environment
        -----------------------------------------

       Windows Endpoint
       ALWIN-WINDOWS
              |
              v
        Wazuh Agent
              |
              v
        Wazuh Manager
              |
              v
          Filebeat
              |
              v
       Wazuh Indexer
              |
              v
       Wazuh Dashboard
```

---

## Security Capabilities

### DevSecOps & CI/CD Security

The project integrates security checks into the GitHub Actions workflow, including:

* Gitleaks secret detection
* Checkov security configuration scanning
* Dockerfile security scanning
* Kubernetes configuration scanning
* GitHub Actions workflow scanning
* Security-focused repository validation

The workflow is triggered for:

* `main`
* `security/**`
* Pull requests targeting `main`

The workflow uses read-only repository permissions:

```yaml
permissions:
  contents: read
```

The project intentionally focuses on security validation rather than implementing a full artifact publishing or production deployment pipeline.

OWASP recommends integrating security activities into CI/CD so security issues can be identified earlier in the development lifecycle.

---

## Security Scanning Results

### Gitleaks

Gitleaks was used to scan repository history for exposed secrets.

Final validation:

```text
Commits scanned: 19
Repository size: ~187 KB
Leaks detected: 0
```

Evidence:

```text
gitleaks-report.json
```

---

### Checkov

Checkov 3.3.16 was used to validate Kubernetes, Dockerfile, and GitHub Actions security configuration.

Final results:

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

Terraform plan scanning was not applicable because the final project does not contain Terraform configuration.

Intentional exclusions include generated files and dedicated Kyverno test manifests.

---

### Trivy

Trivy was used for Kubernetes security scanning.

Final cluster validation:

```text
Resources scanned: 324 / 324
Node scanning: enabled
```

Command used:

```text
trivy k8s kind-devsecops-lab --report summary --timeout 15m
```

Trivy was also used to support container security analysis and SBOM generation.

---

## Software Bill of Materials

CycloneDX SBOMs were generated for the relevant container images:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

The SBOMs provide machine-readable component inventories that can support vulnerability and dependency visibility.

SBOMs are a recognized software supply-chain security practice for maintaining visibility into application components and dependencies.

---

# Container Security

Application containers were hardened using Kubernetes and container-level security controls.

Implemented controls include:

* Non-root execution where supported
* Dedicated runtime identities
* `allowPrivilegeEscalation: false`
* Linux capability reduction
* `no-new-privileges`
* RuntimeDefault seccomp
* Read-only root filesystem where applicable
* Writable temporary storage through `tmpfs`
* Resource requests and limits
* Container health probes

### Application identities

Backend:

```text
UID: 100
GID: 101
```

Frontend:

```text
UID: 101
```

The backend and frontend workloads run with hardened security contexts.

---

# Kubernetes Security

The Kubernetes environment is based on:

```text
Kind
Kubernetes
kubectl
Helm
```

Cluster:

```text
devsecops-lab
```

Context:

```text
kind-devsecops-lab
```

Namespaces include:

```text
devsecops
vault
kyverno
monitoring
```

Security controls include:

* Dedicated ServiceAccounts
* SecurityContexts
* Non-root workloads
* Seccomp RuntimeDefault
* Privilege-escalation prevention
* Capability restrictions
* Resource controls
* Readiness/liveness probes
* NetworkPolicies
* Kyverno policies
* Trivy Kubernetes scanning

---

# Zero Trust Network Segmentation

The application implements a **default-deny network model**.

Expected application flow:

```text
Frontend
    |
    | TCP 8080
    v
Backend
    |
    | TCP 3306
    v
MySQL
```

The frontend is not permitted to communicate directly with MySQL.

Additional controlled communication includes:

```text
Backend -> Vault
Backend -> DNS
```

Unnecessary workload-to-workload communication is blocked through Kubernetes NetworkPolicies.

This provides a practical demonstration of:

* Explicit communication paths
* Least-privilege connectivity
* Default-deny networking
* Reduced lateral movement
* Assume-breach principles

The project describes this as **Zero Trust-aligned Kubernetes security**, not as a complete enterprise Zero Trust implementation.

---

# Kyverno Policy Enforcement

Kyverno provides Kubernetes policy-as-code controls.

Implemented policies include:

```text
require-run-as-nonroot
require-no-privilege-escalation
require-seccomp-runtime-default
require-resource-requests-limits
```

The policies provide continuous validation of workload security configuration.

Kyverno was configured with audit-oriented validation for the security controls used in this local laboratory.

---

# Secrets Management with HashiCorp Vault

HashiCorp Vault is integrated with Kubernetes for runtime secret management.

The backend uses:

* Kubernetes authentication
* Dedicated `backend-sa`
* Vault role `devsecops-backend`
* Least-privilege Vault policy
* Vault Agent Injector
* File-based secret injection

Secret path:

```text
secret/data/devsecops/backend
```

The runtime flow is:

```text
Backend Pod
     |
     v
backend-sa
     |
     v
Vault Kubernetes Authentication
     |
     v
devsecops-backend Role
     |
     v
Vault Policy
     |
     v
Secret
     |
     v
Vault Agent Injector
     |
     v
/run/secrets/
     |
     v
Spring Boot
     |
     v
MySQL
```

The injected files include:

```text
/run/secrets/db_user
/run/secrets/db_password
```

The secret values are not stored directly in the Kubernetes Deployment manifest.

Validation confirmed that:

```text
Vault authentication
        ↓
Secret injection
        ↓
Spring Boot configuration
        ↓
Database connection
        ↓
Application API
```

was functioning correctly.

---

# Kubernetes Runtime Policy Exception

One documented exception exists for the MySQL workload.

The `mysql:8.4` container could not be forced to run as a non-root user in the current local configuration because enforcing:

```text
runAsNonRoot: true
```

resulted in:

```text
setgid: Operation not permitted
```

The workload therefore retains its required runtime identity while other security controls remain enabled.

The exception is documented in:

```text
docs/security-exceptions.md
```

The MySQL workload retains:

* Privilege-escalation prevention
* Seccomp RuntimeDefault
* Capability restrictions
* Network isolation
* Backend-only database access
* Vault-managed credentials

This is explicitly treated as a **local laboratory compatibility exception**, not as a recommended production configuration.

---

# Monitoring & Alerting

Prometheus and Grafana provide Kubernetes monitoring and alerting.

Components include:

* Prometheus
* Alertmanager
* Grafana
* kube-state-metrics
* Node Exporter
* Prometheus Operator

Configured security/availability alerts include:

```text
BackendDeploymentReplicasLow
BackendPodRestartDetected
BackendDeploymentUnavailable
```

The monitoring configuration was validated by scaling the backend workload down and restoring it, allowing the alerting path to be exercised.

Prometheus/Grafana are used for **metrics, observability, and alerting**.

They are not treated as a SIEM.

---

# Wazuh Security Monitoring

Wazuh was implemented as a separate runtime security monitoring environment.

Environment:

```text
Ubuntu 24.04.5 WSL
Wazuh 4.14.8
```

The Wazuh environment includes:

* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard
* Filebeat

A Windows endpoint was enrolled as:

```text
ALWIN-WINDOWS
```

The event pipeline is:

```text
Windows Event
     |
     v
Wazuh Agent
     |
     v
Wazuh Manager
     |
     v
Archives
     |
     v
Filebeat
     |
     v
Wazuh Indexer
     |
     v
Wazuh Dashboard
```

Wazuh archives were enabled and validated using:

```text
/var/ossec/logs/archives/archives.json
```

An application event was generated on the Windows endpoint and subsequently verified in the Wazuh Dashboard using:

```text
wazuh-archives-*
```

This validated the complete endpoint-event-to-dashboard pipeline.

Wazuh provides the project's **runtime security monitoring / SIEM layer**, complementing the Kubernetes-focused Prometheus and Grafana monitoring.

---

# GitHub Actions Security Pipeline

The final security workflow is located at:

```text
.github/workflows/devsecops-security.yml
```

The workflow performs:

```text
Repository Checkout
        |
        v
     Gitleaks
        |
        v
     Checkov
```

The workflow is intentionally focused on security validation and does not claim production artifact publishing or registry deployment.

---

# Technology Stack

| Area                    | Technologies              |
| ----------------------- | ------------------------- |
| Application             | Spring Boot, React, MySQL |
| Containers              | Docker                    |
| Orchestration           | Kubernetes                |
| Local Cluster           | Kind                      |
| CI/CD                   | GitHub Actions            |
| Secret Scanning         | Gitleaks                  |
| Configuration Security  | Checkov                   |
| Kubernetes Scanning     | Trivy                     |
| SBOM                    | CycloneDX                 |
| Secrets Management      | HashiCorp Vault           |
| Policy Enforcement      | Kyverno                   |
| Metrics                 | Prometheus                |
| Monitoring / Alerting   | Grafana, Alertmanager     |
| Runtime Security / SIEM | Wazuh                     |
| Windows Monitoring      | Wazuh Agent               |
| Linux Environment       | Ubuntu WSL                |

---

# Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── devsecops-security.yml
│
├── backend/
├── frontend/
├── grafana/
├── k8s/
│
├── docs/
│   ├── architecture.md
│   ├── monitoring.md
│   ├── project-report.md
│   ├── security-controls.md
│   ├── security-exceptions.md
│   ├── security-testing.md
│   ├── wazuh.md
│   └── zero-trust.md
│
├── backend-sbom.json
├── frontend-sbom.json
├── mysql-sbom.json
├── gitleaks-report.json
├── checkov-results.json
├── README.md
└── LICENSE
```

---

# Documentation

Detailed documentation is available in the `docs/` directory:

* [Architecture](docs/architecture.md)
* [Security Controls](docs/security-controls.md)
* [Zero Trust Architecture](docs/zero-trust.md)
* [Wazuh](docs/wazuh.md)
* [Monitoring](docs/monitoring.md)
* [Security Testing](docs/security-testing.md)
* [Security Exceptions](docs/security-exceptions.md)
* [Project Report](docs/project-report.md)

The documentation records implemented controls, validation evidence, limitations, and security exceptions.

---

# Security Testing Summary

The project was validated across multiple security layers:

```text
                    Security Validation
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
   Repository          Kubernetes          Runtime
       |                   |                   |
    Gitleaks          Checkov / Trivy       Wazuh
       |                   |                   |
       +-------------------+-------------------+
                           |
                           v
                    Vault / Kyverno
                           |
                           v
                  Prometheus / Grafana
```

Key validation results:

```text
Gitleaks
  19 commits
  0 leaks

Checkov Kubernetes
  263 passed
  0 failed
  11 skipped

Checkov Dockerfile
  113 passed
  0 failed

Checkov GitHub Actions
  204 passed
  0 failed

Trivy Kubernetes
  324 / 324 resources scanned

Vault
  Authentication + injection validated

Kyverno
  Security policies validated
  MySQL exception documented

Wazuh
  Windows event → Dashboard pipeline validated

Grafana
  Backend availability/restart alerting validated
```

---

# Project Scope and Limitations

This project intentionally focuses on **security engineering and DevSecOps**, rather than application feature development.

The environment is a local laboratory using:

* Kind
* Docker
* Kubernetes
* Ubuntu WSL
* Wazuh

It does not claim to provide:

* Production-grade Kubernetes
* Multi-node high availability
* Disaster recovery
* Cloud deployment
* Enterprise IAM
* Enterprise ZTNA
* Service-mesh-based mTLS
* Production SIEM infrastructure
* Full application penetration testing
* Full production DAST program
* Autonomous AI security response

The project's purpose is to demonstrate practical security engineering across the software delivery and runtime lifecycle.

---

# Project Outcome

The completed platform demonstrates defense-in-depth across:

```text
Source
  ↓
Secret Detection
  ↓
CI/CD Security
  ↓
Container Hardening
  ↓
Kubernetes Security
  ↓
Network Segmentation
  ↓
Policy Enforcement
  ↓
Secrets Management
  ↓
Runtime Monitoring
  ↓
Security Alerting
```

The result is a practical DevSecOps and Kubernetes security laboratory demonstrating how multiple defensive controls can work together rather than relying on a single security tool.

---

# Project Origin

The application baseline is derived from the MIT-licensed open-source project:

**dockerized-spring-react-mysql**
by **Jhordy Gavinchu**

The original MIT license and copyright notice are retained in this repository.

This repository extends the application baseline with DevSecOps, Kubernetes security, Zero Trust-aligned controls, secrets management, monitoring, runtime security monitoring, and security automation.

---

# Disclaimer

This repository is a **security engineering and DevSecOps portfolio project** intended for development, experimentation, learning, and validation.

The Kubernetes and Wazuh environments are laboratory deployments and should not be considered production-ready without additional infrastructure, security, reliability, operational, and governance controls.

---

# License

The original application baseline is distributed under the MIT License.

See [`LICENSE`](LICENSE) for the original copyright notice and license terms.
