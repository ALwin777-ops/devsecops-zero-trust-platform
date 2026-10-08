# DevSecOps Zero Trust Security Platform

## Comprehensive Security Project Report

---

## 1. Executive Summary

This project implements a security-focused DevSecOps and Zero Trust platform around a containerized Spring Boot, React, and MySQL application.

The original application was treated as an inherited application baseline. The primary engineering objective was not application feature development, but the implementation and validation of security controls across the software delivery and runtime environments.

The project integrates:

* Git secret detection
* CI/CD security scanning
* Docker container hardening
* Kubernetes security contexts
* Kubernetes NetworkPolicies
* Kyverno policy enforcement
* HashiCorp Vault secret management
* Trivy vulnerability and Kubernetes scanning
* Software Bill of Materials generation
* Prometheus monitoring
* Grafana alerting
* Wazuh runtime security monitoring
* Security testing and evidence collection
* Documented security exceptions

The resulting architecture follows a defense-in-depth model:

```text
                    Developer
                        |
                        v
                  Git Repository
                        |
                        v
                GitHub Actions
                 /           \
                v             v
           Gitleaks        Checkov
                \             /
                 \           /
                  v         v
                  Security Gate
                        |
                        v
                 Kubernetes Kind
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
    Frontend         Backend          MySQL
        |               |
        |               +------> Vault
        |
        +------> Backend only

        Kubernetes Security Layer
        ├── NetworkPolicies
        ├── Kyverno
        ├── Security Contexts
        └── Service Accounts

        Monitoring Layer
        ├── Prometheus
        └── Grafana

        Runtime Security Layer
        └── Wazuh
```

The project demonstrates how multiple security controls can be combined into a practical DevSecOps and Zero Trust laboratory environment.

OWASP describes DevSecOps as embedding security into DevOps activities and recommends detecting security issues as early as possible while continuing to detect them throughout the lifecycle.

---

# 2. Project Objectives

The main objectives were:

1. Secure the inherited containerized application.
2. Introduce security controls into CI/CD.
3. Reduce container privilege.
4. Implement Kubernetes workload hardening.
5. Apply Zero Trust network segmentation.
6. Centralize application secrets using Vault.
7. Enforce security policies using Kyverno.
8. Scan Kubernetes and container environments.
9. Generate SBOMs for software inventory.
10. Monitor Kubernetes health using Prometheus and Grafana.
11. Implement runtime security monitoring using Wazuh.
12. Validate security controls through repeatable testing.
13. Document security exceptions and residual risk.
14. Produce evidence suitable for technical review and interview discussion.

---

# 3. Project Scope

## Included

The final project scope includes:

```text
DevSecOps
Kubernetes Security
Container Security
Zero Trust Networking
Secrets Management
Policy Enforcement
Security Scanning
Monitoring
Runtime Security
CI/CD Security
Security Documentation
```

## Excluded

The following were intentionally outside the final scope:

```text
AWS production deployment
Cloud security architecture
OIDC cloud integration
Production ZTNA
Service mesh / mTLS
Enterprise PAM
Kubernetes multi-cluster HA
Production disaster recovery
Autonomous AI security response
```

The project therefore should be evaluated as a **local DevSecOps and Kubernetes security laboratory**, not as a complete enterprise cloud-security platform.

---

# 4. Application Architecture

The underlying application uses a three-tier architecture:

```text
React Frontend
      |
      | HTTP
      v
Spring Boot Backend
      |
      | MySQL protocol
      v
MySQL Database
```

The security architecture was implemented around these application components rather than replacing them.

---

# 5. Container Security

Container security was implemented before deployment into Kubernetes.

The backend container runs using:

```text
UID: 100
GID: 101
```

The frontend container runs using:

```text
UID: 101
```

Container hardening includes:

* Non-root execution
* No privilege escalation
* Capability restrictions
* `no-new-privileges`
* Read-only root filesystem where applicable
* Temporary writable filesystem for required paths
* Restricted runtime configuration

The objective is to reduce the impact of a compromised application process.

---

# 6. Kubernetes Security

The application is deployed into a Kind Kubernetes cluster:

```text
Cluster: devsecops-lab
Context: kind-devsecops-lab
```

Primary application namespace:

```text
devsecops
```

Additional namespaces include:

```text
vault
kyverno
monitoring
```

Kubernetes security controls include:

* Dedicated service accounts
* Security contexts
* Non-root execution for supported workloads
* Seccomp RuntimeDefault
* Privilege-escalation restrictions
* Capability restrictions
* NetworkPolicies
* Kyverno policy validation

---

# 7. Kubernetes Service Accounts

Dedicated service accounts are used for application workloads:

```text
backend-sa
frontend-sa
mysql-sa
```

The backend service account also provides the Kubernetes identity used for Vault authentication.

This creates an explicit workload identity instead of relying on a shared identity model.

---

# 8. Zero Trust Network Architecture

The Kubernetes network was designed using a default-deny model.

The basic principle is:

```text
Deny by Default
       +
Explicitly Allow Required Traffic
```

Required traffic includes:

```text
Frontend → Backend       TCP/8080
Backend → MySQL          TCP/3306
Backend → Vault          Vault API
Backend → DNS            DNS
```

Unnecessary paths are denied.

For example:

```text
Frontend → MySQL
```

is not permitted.

This limits lateral movement and reduces unnecessary trust relationships between application components.

---

# 9. Vault Secrets Management

HashiCorp Vault provides centralized secret management for the backend application.

The relevant secret path is:

```text
secret/data/devsecops/backend
```

The Vault role is:

```text
devsecops-backend
```

The role is associated with:

```text
backend-sa
```

in the:

```text
devsecops
```

namespace.

The backend uses Kubernetes authentication to obtain access to the appropriate Vault policy.

---

# 10. Vault Agent Injection

Vault Agent Injector is used to provide secrets to the backend workload.

The high-level flow is:

```text
Backend Pod
    |
    v
backend-sa
    |
    v
Kubernetes Auth
    |
    v
Vault Role
    |
    v
Vault Policy
    |
    v
Backend Secret
    |
    v
Vault Agent Injector
    |
    v
/run/secrets/
```

The injected files include:

```text
/run/secrets/db_user
/run/secrets/db_password
```

Secret values are deliberately excluded from project documentation and evidence.

---

# 11. Vault Application Validation

Vault integration was not considered complete merely because the Vault configuration existed.

The injected secret files were verified inside the backend container.

The application was then tested through its internal API:

```text
/api/user
```

The API returned:

```text
[]
```

This demonstrated that the backend continued to operate after moving the database credentials into the Vault-based secret-management workflow.

The validated path was:

```text
Vault
  |
  v
Secret Injection
  |
  v
Backend
  |
  v
MySQL
  |
  v
API
```

---

# 12. Kyverno Policy Enforcement

Kyverno provides Kubernetes policy-as-code.

The implemented policies include:

```text
require-run-as-nonroot
require-no-privilege-escalation
require-seccomp-runtime-default
require-resource-requests-limits
```

These policies provide centralized validation of workload security properties.

The policy layer complements Checkov because:

```text
Checkov
    |
    v
Configuration / IaC Analysis

Kyverno
    |
    v
Kubernetes Policy Enforcement / Validation
```

---

# 13. Kyverno Policy Exception

The MySQL workload has one documented exception.

The workload could not successfully run with:

```yaml
runAsNonRoot: true
```

because the container failed with:

```text
setgid: Operation not permitted
```

The exception is documented in:

```text
docs/security-exceptions.md
```

Other security controls remain enabled for the MySQL workload.

The PolicyReport recorded:

```text
No privilege escalation       PASS
Resource requests/limits      PASS
Seccomp RuntimeDefault        PASS
runAsNonRoot                  FAIL
```

The project deliberately reports this as an exception rather than claiming complete compliance.

---

# 14. Trivy Security Scanning

Trivy was used for Kubernetes and container security scanning.

The Kubernetes environment was scanned using:

```powershell
trivy k8s kind-devsecops-lab --report summary --timeout 15m
```

The scan evaluated:

```text
19 / 19 devsecops resources
```


Trivy was also used for container-related security analysis and SBOM generation.

---

# 15. Software Bill of Materials

SBOMs were generated for:

```text
backend
frontend
mysql
```

Artifacts include:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

CycloneDX was used as the SBOM format.

The SBOMs provide a machine-readable inventory of software components and support software supply-chain visibility.

OWASP notes that SBOMs can help track dependencies and identify which applications are affected when a vulnerability is associated with a particular component.

---

# 16. Gitleaks

Gitleaks was used to detect accidentally committed secrets.

The repository history tested contained approximately:

```text
19 commits
~187 KB
```

Final result:

```text
Detected leaks: 0
```

Evidence:

```text
gitleaks-report.json
```

This provides a source-control security baseline.

---

# 17. Checkov

Checkov was used to scan:

* Kubernetes configuration
* Dockerfiles
* GitHub Actions

Final Kubernetes result:

```text
Passed: 263
Failed: 0
Skipped: 11
```

Final Dockerfile result:

```text
Passed: 113
Failed: 0
Skipped: 0
```

Final GitHub Actions result:

```text
Passed: 204
Failed: 0
Skipped: 0
```

Terraform scanning was not included because Terraform was not part of the final project implementation.

---

# 18. CI/CD Security Pipeline

The security workflow is:

```text
.github/workflows/devsecops-security.yml
```

Pipeline flow:

```text
Git Push / Pull Request
          |
          v
      Checkout
          |
          v
       Gitleaks
          |
          v
        Checkov
          |
          v
   Security Validation
```

The workflow triggers on:

```text
main
security/**
Pull Requests → main
```

Repository permissions are restricted to:

```yaml
permissions:
  contents: read
```

This follows the principle of minimizing unnecessary CI/CD permissions.

CI/CD systems are themselves security-sensitive infrastructure because repositories, automation systems, credentials, build processes, and runners can become attack paths. OWASP specifically recommends securing the CI/CD pipeline as part of the overall security architecture.

---

# 19. Prometheus Monitoring

Prometheus was deployed in:

```text
monitoring
```

The monitoring environment includes:

```text
Prometheus
Prometheus Operator
kube-state-metrics
Node Exporter
Alertmanager
```

Prometheus collects Kubernetes workload and infrastructure metrics.

Prometheus retention in this laboratory environment is approximately:

```text
24 hours
```

---

# 20. Grafana Monitoring

Grafana provides visualization and alert management.

Grafana version:

```text
13.2.2-distroless
```

The project contains a:

```text
DevSecOps Kubernetes
```

alert folder and:

```text
Kubernetes Alerts
```

alert group.

---

# 21. Grafana Alerts

The configured backend alerts include:

### BackendDeploymentReplicasLow

Detects when available backend replicas fall below the configured threshold.

### BackendPodRestartDetected

Detects backend restart activity.

### BackendDeploymentUnavailable

Detects unavailable backend replicas.

The alerts provide operational visibility into application availability and runtime instability.

---

# 22. Monitoring Validation

The backend Deployment was deliberately scaled down and then restored.

The test sequence included:

```text
2 replicas
    ↓
1 replica
    ↓
2 replicas
```

This was used to validate that Kubernetes workload state changes were reflected in the monitoring system.

The test confirmed that the monitoring layer can observe application availability changes.

---

# 23. Wazuh Runtime Security

Wazuh provides the runtime security monitoring layer.

The Wazuh environment is deployed separately from Kubernetes in:

```text
Ubuntu 24.04.5 WSL
```

The Wazuh installation is an all-in-one laboratory deployment containing:

```text
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard
Filebeat
```

Wazuh version:

```text
4.14.8
```

---

# 24. Windows Endpoint Monitoring

A Windows endpoint agent is enrolled with the logical name:

```text
USER-WINDOWSS
```

The runtime event path is:

```text
Windows
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

This adds a runtime security monitoring layer that is separate from Kubernetes infrastructure metrics.

---

# 25. Wazuh Archive Pipeline

Wazuh JSON event archiving was enabled.

Archive location:

```text
/var/ossec/logs/archives/archives.json
```

Filebeat archive forwarding was also enabled.

The archive index pattern used for investigation is:

```text
wazuh-archives-*
```

Filebeat output connectivity was validated successfully.

---

# 26. Wazuh End-to-End Validation

A Windows test event was generated with:

```text
Event ID: 998
Source: WazuhTest
Log: APPLICATION
```

The event was successfully observed through the Wazuh archive pipeline and located in the Wazuh Dashboard Discover interface.

The test therefore validated:

```text
Windows Event
      ↓
Wazuh Agent
      ↓
Wazuh Manager
      ↓
archives.json
      ↓
Filebeat
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```

This demonstrates that the runtime security telemetry pipeline is operational.

---

# 27. Prometheus/Grafana vs Wazuh

These technologies serve different purposes.

| Technology   | Primary Role               |
| ------------ | -------------------------- |
| Prometheus   | Metrics collection         |
| Grafana      | Visualization and alerting |
| Alertmanager | Alert management           |
| Wazuh        | Security monitoring / SIEM |
| Vault        | Secrets management         |
| Kyverno      | Kubernetes policy          |
| Trivy        | Security scanning          |
| Checkov      | Configuration/IaC security |
| Gitleaks     | Secret detection           |

Prometheus and Grafana should therefore not be described as SIEM tools.

Wazuh provides the runtime security-monitoring function in this project.

---

# 28. Security Testing

The project used multiple forms of validation:

```text
Automated Scanning
        +
Configuration Validation
        +
Runtime Validation
        +
Functional Testing
        +
Manual Verification
```

Major results include:

```text
Gitleaks
0 detected secrets

Checkov
0 failed checks in final scanned frameworks

Trivy
19 / 19 devsecops resources scanned

Vault
Secret injection validated

Kyverno
Policies operational

Grafana
Replica availability test completed

Wazuh
End-to-end archive event validated
```

The testing methodology and detailed evidence are documented separately in:

```text
docs/security-testing.md
```

OWASP describes verification as including architecture validation, security-control verification, automated security testing, and manual security testing/penetration testing.

---

# 29. Security Testing vs Penetration Testing

This project should not be represented as a complete application penetration test.

The security validation primarily focused on:

```text
DevSecOps Controls
Kubernetes Security
Container Security
Network Security
Secrets Management
Policy Enforcement
Runtime Monitoring
CI/CD Security
```

The inherited application was not rewritten or subjected to a complete external penetration-test engagement as part of this project.

Therefore, the project demonstrates **security engineering and validation**, rather than claiming a full web application VAPT assessment.

This distinction is important for technical accuracy.

---

# 30. Security Control Coverage

The major security-control layers are:

```text
Layer 1 — Source
    Gitleaks

Layer 2 — CI/CD
    GitHub Actions
    Checkov

Layer 3 — Containers
    Docker hardening
    Trivy

Layer 4 — Kubernetes
    Security Contexts
    Service Accounts
    Kyverno

Layer 5 — Network
    NetworkPolicies

Layer 6 — Secrets
    Vault
    Kubernetes Auth
    Vault Agent

Layer 7 — Monitoring
    Prometheus
    Grafana

Layer 8 — Runtime Security
    Wazuh
```

This layered design reduces dependence on any single security mechanism.

---

# 31. Defense-in-Depth Model

The platform can be summarized as:

```text
                 SECURITY LAYERS

          +-------------------------+
          |       Gitleaks          |
          +-------------------------+
                    |
          +-------------------------+
          |      GitHub Actions     |
          |        Checkov          |
          +-------------------------+
                    |
          +-------------------------+
          |   Container Hardening   |
          |        Trivy            |
          +-------------------------+
                    |
          +-------------------------+
          |      Kubernetes         |
          | Kyverno + Security      |
          |       Contexts          |
          +-------------------------+
                    |
          +-------------------------+
          |    NetworkPolicies      |
          +-------------------------+
                    |
          +-------------------------+
          |         Vault           |
          +-------------------------+
                    |
          +-------------------------+
          | Prometheus + Grafana    |
          +-------------------------+
                    |
          +-------------------------+
          |         Wazuh           |
          +-------------------------+
```

The architecture follows a defense-in-depth approach in which preventive, detective, and evidence-producing controls operate together.

---

# 32. Zero Trust Model

The project applies practical Zero Trust principles within the Kubernetes laboratory.

## Verify Explicitly

Workloads use explicit service identities and policies.

## Least Privilege

Network communication and Vault access are restricted to required paths.

## Assume Breach

NetworkPolicies and workload isolation reduce lateral movement opportunities.

## Continuous Validation

Kyverno, Prometheus/Grafana, and Wazuh provide ongoing visibility into workload and security state.

The project does not claim to implement every component of an enterprise Zero Trust architecture.

---

# 33. Assume-Breach Scenario

Consider a compromised frontend workload.

Without segmentation:

```text
Compromised Frontend
        |
        +------> Backend
        |
        +------> MySQL
        |
        +------> Other Services
```

With the implemented NetworkPolicies:

```text
Compromised Frontend
        |
        +------> Backend
        |
        X------> MySQL
        |
        X------> Unrelated Services
```

The attacker must therefore operate within explicitly allowed network paths.

This demonstrates practical micro-segmentation rather than simply labeling the architecture "Zero Trust."

---

# 34. Security Evidence

The project retains evidence artifacts including:

```text
gitleaks-report.json
checkov-results.json
checkov-k8s-results.json
checkov-k8s-results.xml

backend-sbom.json
frontend-sbom.json
mysql-sbom.json

final_report_trivy
```

Additional evidence is available through:

```text
Kubernetes PolicyReports
Grafana alerts
Wazuh archives
Wazuh Discover
Vault configuration
GitHub Actions workflow
```

The evidence supports repeatability and technical review.

---

# 35. Security Exceptions

The project has one intentionally documented security exception:

```text
MySQL runAsNonRoot
```

Reason:

```text
setgid: Operation not permitted
```

Compensating controls include:

* No privilege escalation
* Seccomp RuntimeDefault
* Capability restrictions
* Network isolation
* Backend-only database access
* Vault-based secret management
* Runtime monitoring

The complete exception analysis is documented in:

```text
docs/security-exceptions.md
```

---

# 36. Project Limitations

The current platform has several laboratory limitations.

## Kubernetes

```text
Single Kind control plane
Local development environment
No production HA
```

## Monitoring

```text
Limited Prometheus retention
Grafana persistence disabled
No production notification integrations
```

## Vault

```text
Local laboratory deployment
No production HA/DR
```

## Wazuh

```text
All-in-one deployment
No distributed SOC architecture
No production HA
```

## Database

```text
MySQL non-root exception
```

## Cloud

```text
No production cloud deployment
```

These limitations are intentionally documented instead of being presented as production capabilities.

---

# 37. Security Architecture Maturity

The project can be viewed as a practical intermediate security-engineering laboratory.

It demonstrates:

```text
Security Automation
        +
Container Security
        +
Kubernetes Security
        +
Secrets Management
        +
Policy as Code
        +
Network Segmentation
        +
Monitoring
        +
Runtime Security
```

It does not attempt to reproduce the complete security architecture of a large enterprise.

OWASP's DevSecOps guidance similarly presents security controls as a set of activities that can be progressively introduced into a delivery pipeline according to the architecture and maturity of the environment.

---

# 38. Overall Security Results

| Security Area                | Result                      |
| ---------------------------- | --------------------------- |
| Secret scanning              | PASS                        |
| CI/CD security               | PASS                        |
| Kubernetes configuration     | PASS                        |
| Dockerfile security          | PASS                        |
| Container hardening          | PASS                        |
| Kubernetes security contexts | PASS                        |
| Network segmentation         | PASS                        |
| Vault authentication         | PASS                        |
| Vault secret injection       | PASS                        |
| Kyverno policies             | PASS + documented exception |
| Kubernetes scanning          | COMPLETED                   |
| SBOM generation              | COMPLETED                   |
| Prometheus monitoring        | PASS                        |
| Grafana alerting             | PASS                        |
| Wazuh endpoint monitoring    | PASS                        |
| Wazuh archive pipeline       | PASS                        |
| Security documentation       | COMPLETE                    |

---

# 39. Key Technical Achievements

The project demonstrates practical experience with:

```text
Docker
Kubernetes
Kind
Helm
Kyverno
HashiCorp Vault
Prometheus
Grafana
Wazuh
Trivy
Checkov
Gitleaks
GitHub Actions
NetworkPolicies
Kubernetes SecurityContexts
SBOM
Zero Trust Architecture
Runtime Security Monitoring
```

More importantly, the project demonstrates the ability to integrate these technologies into one security architecture rather than operating each tool independently.

---

# 40. What This Project Demonstrates Professionally

From a cybersecurity engineering perspective, the project demonstrates experience in:

### DevSecOps

Integrating security checks into CI/CD.

### Container Security

Hardening container runtime configurations and scanning images.

### Kubernetes Security

Applying security contexts, service identities, NetworkPolicies, and Kyverno policies.

### Secrets Management

Using Vault and Kubernetes authentication instead of relying solely on static application configuration.

### Cloud-Native Security

Understanding security boundaries between containers, workloads, namespaces, and services.

### Security Monitoring

Using Prometheus/Grafana for operational monitoring and Wazuh for security telemetry.

### Security Validation

Testing implemented controls and retaining evidence.

### Risk Management

Documenting a real exception rather than suppressing the finding.

---

# 41. Project Completion Status

The implementation phase is considered substantially complete.

Completed areas:

```text
Docker Security                         ✓
Gitleaks                                ✓
Trivy                                   ✓
SBOM                                    ✓
Kubernetes Security                     ✓
NetworkPolicies                         ✓
Kyverno                                 ✓
Vault                                   ✓
Prometheus                              ✓
Grafana                                 ✓
Wazuh                                   ✓
Checkov                                 ✓
GitHub Actions                          ✓
Security Testing                        ✓
Security Exceptions                     ✓
Architecture Documentation              ✓
```

The remaining work is primarily project presentation and repository polish rather than another major security-control implementation.

---

# 42. Recommended Future Enhancements

The following are possible future improvements, but are deliberately outside the current frozen scope:

```text
Production Kubernetes deployment
Cloud security architecture
Multi-cluster Kubernetes
Vault HA
Wazuh distributed architecture
Long-term monitoring storage
Artifact signing
SAST
SCA
DAST
mTLS / service mesh
Production alert integrations
Formal penetration testing
```

These should be treated as future roadmap items rather than missing requirements for the current laboratory.

---

# 43. Why AI Is Not Required

An AI security agent was considered during the project.

It is not required for the current architecture.

The existing Wazuh deployment already provides:

```text
Event Collection
       +
Event Analysis
       +
Search
       +
Investigation
```

Adding an AI agent would introduce another component without being necessary to demonstrate the core security architecture.

If AI is introduced in a future version, an appropriate model would be:

```text
Wazuh Alert
     |
     v
AI-Assisted Analysis
     |
     +---- Summary
     +---- Severity Context
     +---- Possible MITRE Mapping
     +---- Investigation Suggestions
     |
     v
Human Analyst
```

The AI should remain advisory rather than autonomously executing security actions.

---

# 44. Final Assessment

The DevSecOps Zero Trust Security Platform successfully demonstrates a layered security architecture around a containerized application.

The project combines:

```text
Preventive Controls
        +
Detective Controls
        +
Policy Controls
        +
Monitoring
        +
Security Evidence
```

The strongest aspect of the implementation is not any individual tool, but the integration between the controls.

For example:

```text
Gitleaks
   ↓
Protect source-control secrets

Checkov
   ↓
Validate configuration

Trivy
   ↓
Scan Kubernetes / containers

Kyverno
   ↓
Validate Kubernetes policy

NetworkPolicies
   ↓
Restrict communication

Vault
   ↓
Protect application secrets

Prometheus/Grafana
   ↓
Monitor infrastructure state

Wazuh
   ↓
Monitor security events
```

This creates a layered DevSecOps security model rather than a collection of unrelated security tools.

---

# 45. Final Project Statement

This project demonstrates the design, implementation, hardening, monitoring, and validation of a Kubernetes-based DevSecOps security platform using practical open-source security technologies.

The platform provides:

```text
Secure Source Control
        +
Secure CI/CD
        +
Secure Containers
        +
Secure Kubernetes
        +
Zero Trust Networking
        +
Centralized Secrets
        +
Policy Enforcement
        +
Security Scanning
        +
Observability
        +
Runtime Security
```

The implementation is intentionally positioned as a **local security engineering laboratory**.

It does not claim production-level availability, enterprise-scale SOC capabilities, or complete application penetration testing.

Instead, it demonstrates the ability to design security controls, implement them across multiple layers, validate their operation, identify residual risk, and document the resulting security posture.

That combination of implementation, validation, and risk documentation is the primary outcome of the project.

---

# 46. Documentation Set

The complete project documentation consists of:

```text
docs/
├── architecture.md
├── security-controls.md
├── zero-trust.md
├── wazuh.md
├── monitoring.md
├── security-testing.md
├── security-exceptions.md
└── project-report.md
```

Together these documents provide:

```text
Architecture
    ↓
Security Controls
    ↓
Zero Trust Design
    ↓
Runtime Security
    ↓
Monitoring
    ↓
Testing
    ↓
Exceptions
    ↓
Final Project Report
```

---

# 47. References

* OWASP DevSecOps Guideline
* OWASP Developer Guide — Secure Development
* OWASP Developer Guide — Verification
* OWASP CI/CD Security Cheat Sheet
* OWASP Web Security Testing Guide
* OWASP Secrets Management guidance
* OWASP Software Supply Chain Security guidance
* Kubernetes documentation
* Kyverno documentation
* HashiCorp Vault documentation
* Trivy documentation
* Checkov documentation
* Gitleaks documentation
* Prometheus documentation
* Grafana documentation
* Wazuh documentation

The project should use the official documentation of each technology as the authoritative reference for implementation-specific behavior.
