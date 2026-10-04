# Security Testing and Validation

## 1. Purpose

This document records the security testing and validation activities performed for the DevSecOps Zero Trust Platform.

The objective was to verify that the implemented security controls operate as intended rather than only confirming that the relevant tools are installed.

Testing was performed across multiple layers:

```text
Source Code
    |
    v
CI/CD
    |
    v
Container Images
    |
    v
Kubernetes
    |
    v
Network Controls
    |
    v
Secrets Management
    |
    v
Policy Enforcement
    |
    v
Monitoring
    |
    v
Runtime Security
```

The testing strategy follows the DevSecOps principle of integrating security checks into development and deployment workflows and continuously validating security controls.

---

# 2. Testing Scope

The security validation covered:

* Gitleaks secret detection
* Checkov configuration scanning
* Trivy Kubernetes scanning
* Trivy container/image scanning
* SBOM generation
* Docker security hardening
* Kubernetes security contexts
* Kubernetes NetworkPolicies
* Vault authentication
* Vault secret injection
* Kyverno policy validation
* Prometheus/Grafana monitoring
* Grafana alert testing
* Wazuh endpoint monitoring
* Wazuh archive pipeline
* GitHub Actions security pipeline
* Application/API connectivity
* Documented MySQL security exception

The testing focused on the security controls introduced as part of the DevSecOps platform.

---

# 3. Testing Methodology

The project used a combination of:

```text
Automated Scanning
        +
Configuration Validation
        +
Runtime Validation
        +
Functional Security Checks
        +
Manual Verification
```

This is preferable to relying on a single security scanner.

Automated security testing provides repeatable validation, while manual verification is used to confirm that the implemented controls actually behave as expected. OWASP recommends combining automated security testing with manual expert testing where appropriate.

---

# 4. Test Environment

The primary Kubernetes environment is:

```text
Cluster: devsecops-lab
Context: kind-devsecops-lab
Platform: Kubernetes Kind
```

Relevant namespaces include:

```text
devsecops
vault
kyverno
monitoring
```

The Wazuh environment runs separately in:

```text
Ubuntu 24.04.5 LTS
WSL
```

The monitored Windows endpoint is registered as:

```text
ALWIN-WINDOWS
```

---

# 5. Security Testing Summary

| Test Area                              | Tool / Method         | Result                               |
| -------------------------------------- | --------------------- | ------------------------------------ |
| Secret detection                       | Gitleaks              | PASS                                 |
| Kubernetes configuration               | Checkov               | PASS                                 |
| Dockerfile security                    | Checkov               | PASS                                 |
| GitHub Actions security                | Checkov               | PASS                                 |
| Kubernetes vulnerability/security scan | Trivy                 | Completed                            |
| SBOM generation                        | Trivy CycloneDX       | Completed                            |
| Container hardening                    | Manual validation     | PASS                                 |
| Security contexts                      | kubectl validation    | PASS                                 |
| Network segmentation                   | NetworkPolicies       | PASS                                 |
| Vault authentication                   | Kubernetes Auth       | PASS                                 |
| Vault secret injection                 | Vault Agent           | PASS                                 |
| Kyverno policies                       | PolicyReports         | PASS with documented MySQL exception |
| Prometheus metrics                     | Runtime validation    | PASS                                 |
| Grafana alerts                         | Scaling test          | PASS                                 |
| Wazuh agent                            | Endpoint validation   | PASS                                 |
| Wazuh archive pipeline                 | End-to-end event test | PASS                                 |
| GitHub Actions                         | CI pipeline           | PASS                                 |
| Application connectivity               | Backend API test      | PASS                                 |

---

# 6. Gitleaks Secret Detection

## Objective

Identify credentials or other sensitive information accidentally committed to the Git repository.

## Tool

```text
Gitleaks
```

## Test Scope

The repository history was scanned.

The tested repository contained approximately:

```text
19 commits
~187 KB
```

## Result

```text
Leaks detected: 0
```

Result:

```text
PASS
```

## Evidence

The generated report is:

```text
gitleaks-report.json
```

The successful scan demonstrates that no detected secrets were present in the scanned repository history.

---

# 7. Checkov Kubernetes Testing

## Objective

Validate Kubernetes configuration against security and configuration best practices.

## Tool

```text
Checkov 3.3.16
```

## Test

The Kubernetes manifests were scanned using Checkov.

The final result was:

```text
Passed: 263
Failed: 0
Skipped: 11
```

## Result

```text
PASS
```

The skipped checks represent intentionally excluded or context-specific controls.

The project does not claim that skipped checks are equivalent to passed checks.

---

# 8. Checkov Dockerfile Testing

Dockerfiles were independently evaluated for security configuration.

Final result:

```text
Passed: 113
Failed: 0
Skipped: 0
```

Result:

```text
PASS
```

The testing included security properties such as:

* Non-root execution
* Container configuration
* Port configuration
* Runtime hardening

---

# 9. Checkov GitHub Actions Testing

The GitHub Actions workflow was also scanned.

Final result:

```text
Passed: 204
Failed: 0
Skipped: 0
```

Result:

```text
PASS
```

The workflow includes:

```text
Gitleaks
Checkov
```

and uses restricted repository permissions:

```yaml
permissions:
  contents: read
```

This reduces unnecessary permissions granted to the CI workflow.

---

# 10. Checkov Terraform Scope

Terraform was not part of the final implementation scope.

The repository does not contain Terraform configuration required for this project.

Therefore:

```text
terraform_plan
```

was excluded from the final Checkov command.

This is a scope decision rather than a claim that Terraform security was tested.

---

# 11. Trivy Kubernetes Testing

## Objective

Scan the Kubernetes environment for security findings and configuration issues.

## Tool

```text
Trivy
```

## Command

The Kubernetes cluster was scanned using:

```powershell
trivy k8s kind-devsecops-lab --report summary --timeout 15m
```

## Result

The scan evaluated:

```text
324 / 324 resources
```

Node scanning was enabled.

Result:

```text
COMPLETED
```

This provided broad visibility into the Kubernetes environment.

---

# 12. Trivy Container Testing

Container images were also evaluated using Trivy.

The project includes images for:

```text
Backend
Frontend
MySQL
```

The testing was used to identify vulnerabilities and security-relevant findings within the container environment.

The objective was not simply to produce a scanner report but to incorporate container security into the overall DevSecOps lifecycle.

---

# 13. SBOM Generation

Software Bills of Materials were generated using Trivy.

The project contains:

```text
backend-sbom.json
frontend-sbom.json
mysql-sbom.json
```

The SBOM format used includes CycloneDX output.

The SBOM provides a machine-readable inventory of software components contained within the relevant images.

This supports software supply-chain visibility.

OWASP identifies SBOMs as an important component of software supply-chain security.

---

# 14. Docker Security Hardening Validation

The application containers were manually inspected to confirm security hardening.

The backend uses:

```text
UID 100
GID 101
```

The frontend uses:

```text
UID 101
```

Security controls include:

```text
Non-root execution
No privilege escalation
Capability restrictions
No-new-privileges
Read-only root filesystem where applicable
Temporary writable filesystem for required paths
```

Result:

```text
PASS
```

---

# 15. Kubernetes Security Context Validation

The Kubernetes workloads were inspected using Kubernetes commands to confirm that security settings were actually applied to running workloads.

The backend was verified to use:

```text
runAsNonRoot: true
```

with:

```text
UID: 100
GID: 101
```

The workload also uses:

```text
allowPrivilegeEscalation: false
Seccomp RuntimeDefault
```

Result:

```text
PASS
```

---

# 16. Kubernetes Service Account Validation

Dedicated service accounts were configured for application workloads.

Relevant service accounts include:

```text
backend-sa
frontend-sa
mysql-sa
```

The backend service account is especially important because it is used as the Kubernetes identity for Vault authentication.

This provides a clear identity boundary between workloads.

Result:

```text
PASS
```

---

# 17. NetworkPolicy Validation

Network segmentation was validated against the intended application architecture.

The primary allowed paths are:

```text
Frontend → Backend : 8080
Backend → MySQL    : 3306
Backend → Vault    : Vault API
Backend → DNS      : DNS
```

Unnecessary communication is denied by default.

The architecture therefore follows:

```text
Default Deny
     +
Explicit Allow
```

Result:

```text
PASS
```

---

# 18. Network Segmentation Model

The intended application flow is:

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

The frontend does not require direct database access.

The backend communicates with Vault for secret management.

This was used as the baseline for NetworkPolicy configuration.

---

# 19. Vault Authentication Testing

## Objective

Verify that the backend can authenticate to Vault using its Kubernetes workload identity.

The configured Vault role is:

```text
devsecops-backend
```

The role is associated with:

```text
backend-sa
```

in:

```text
devsecops
```

The Vault policy is:

```text
devsecops-backend
```

The policy provides restricted access to the backend secret.

Result:

```text
PASS
```

---

# 20. Vault Secret Injection Testing

The backend deployment was configured with Vault Agent Injector.

The injected secret files were verified inside the backend container:

```text
/run/secrets/db_user
/run/secrets/db_password
```

The files were confirmed to exist.

Secret values were intentionally not displayed or included in project documentation.

Result:

```text
PASS
```

This confirms that the Vault-to-workload injection mechanism was operational.

---

# 21. Vault-to-Application Validation

The backend application was tested after Vault integration.

The backend API was queried internally:

```text
/api/user
```

The response was:

```text
[]
```

The response demonstrated successful application operation after the secret-injection configuration was applied.

The test therefore validated the application path:

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

Result:

```text
PASS
```

---

# 22. Kyverno Policy Validation

Kyverno was used to validate Kubernetes security policies.

The configured policies include:

```text
require-run-as-nonroot
require-no-privilege-escalation
require-seccomp-runtime-default
require-resource-requests-limits
```

The policies were confirmed to be ready and operational.

Result:

```text
PASS
```

---

# 23. Kyverno PolicyReport Testing

The MySQL workload produced the following PolicyReport outcome:

```text
No privilege escalation        PASS
Resource requests/limits       PASS
Seccomp RuntimeDefault         PASS
runAsNonRoot                   FAIL
```

Summary:

```text
Error: 0
Fail: 1
Pass: 3
```

The single failure was intentional and documented.

---

# 24. MySQL Security Exception Validation

The MySQL workload could not be forced to run as non-root in the local environment.

Attempting to enforce the requirement resulted in:

```text
setgid: Operation not permitted
```

The workload therefore retains compatible security controls while documenting the non-root exception.

The exception is not hidden.

It is explicitly documented in:

```text
docs/security-exceptions.md
```

This is important because a security assessment should distinguish between:

```text
PASS
```

and:

```text
EXCEPTION / ACCEPTED RISK
```

rather than incorrectly treating every control as passed.

---

# 25. Prometheus Validation

Prometheus was validated as the metrics collection layer.

The monitoring environment contains:

```text
Prometheus
Prometheus Operator
kube-state-metrics
Node Exporter
```

Kubernetes workload metrics were available for querying.

Result:

```text
PASS
```

---

# 26. Grafana Alert Validation

The following alerts were configured:

```text
BackendDeploymentReplicasLow
BackendPodRestartDetected
BackendDeploymentUnavailable
```

The alert group is:

```text
DevSecOps Kubernetes
```

with:

```text
Kubernetes Alerts
```

The configured evaluation interval is:

```text
1 minute
```

---

# 27. Backend Replica Test

The backend Deployment was intentionally scaled from:

```text
2 replicas
```

to:

```text
1 replica
```

and then restored to:

```text
2 replicas
```

This was used to validate the replica availability monitoring logic.

The test demonstrated that changes to the Kubernetes workload state were visible to the monitoring system.

Result:

```text
PASS
```

---

# 28. Backend Restart Monitoring

The project also includes:

```text
BackendPodRestartDetected
```

This provides visibility into unexpected backend pod restart activity.

The alert is designed to identify runtime instability requiring investigation.

The alert complements replica-availability monitoring by providing a different signal:

```text
Replica Availability
        +
Restart Activity
```

---

# 29. Wazuh Agent Validation

The Windows endpoint was enrolled into Wazuh.

The registered logical agent name is:

```text
ALWIN-WINDOWS
```

The agent was confirmed to communicate with the Wazuh environment.

Result:

```text
PASS
```

---

# 30. Wazuh Archive Validation

Wazuh JSON event archiving was enabled.

The archive file is:

```text
/var/ossec/logs/archives/archives.json
```

The configuration was validated by confirming that events were written to the archive.

Result:

```text
PASS
```

---

# 31. Wazuh Filebeat Validation

Filebeat archive forwarding was enabled.

The configured archive forwarding capability was validated.

Filebeat connectivity was tested using:

```bash
sudo filebeat test output
```

The output confirmed successful communication with the configured Wazuh indexer.

Result:

```text
PASS
```

---

# 32. Wazuh End-to-End Event Test

A test Windows event was generated:

```text
Event ID: 998
Log: APPLICATION
Source: WazuhTest
```

Test message:

```text
Wazuh archive indexed test
```

The event was then traced through the Wazuh pipeline.

---

# 33. Wazuh Pipeline Validation

The complete validated path was:

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
archives.json
      |
      v
Filebeat
      |
      v
Wazuh Indexer
      |
      v
wazuh-archives-*
      |
      v
Wazuh Dashboard
      |
      v
Discover
```

The generated event was successfully found in the `wazuh-archives-*` index pattern.

Result:

```text
PASS
```

This is one of the strongest runtime-validation results in the project because it verifies the complete data pipeline.

---

# 34. GitHub Actions Validation

The repository contains the security workflow:

```text
.github/workflows/devsecops-security.yml
```

The workflow performs:

```text
Checkout
   |
   v
Gitleaks
   |
   v
Checkov
```

The workflow is configured for:

```text
main
security/**
pull_request → main
```

Repository permissions are restricted to:

```yaml
permissions:
  contents: read
```

The workflow itself was also scanned using Checkov.

Result:

```text
PASS
```

---

# 35. CI Security Model

The CI security model is:

```text
Developer
    |
    v
Git Push / Pull Request
    |
    v
GitHub Actions
    |
    +---- Gitleaks
    |
    +---- Checkov
    |
    v
Security Validation
```

This shifts selected security checks earlier into the software-delivery lifecycle.

OWASP recommends integrating security activities into CI/CD rather than treating security as a separate late-stage activity.

---

# 36. Security Testing Evidence

The project contains several generated security artifacts.

Relevant evidence includes:

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

These artifacts provide evidence of automated security scanning and supply-chain analysis.

---

# 37. Testing Matrix

The final testing matrix is:

| ID   | Control              | Validation            | Result           |
| ---- | -------------------- | --------------------- | ---------------- |
| T-01 | Secret detection     | Gitleaks              | PASS             |
| T-02 | Kubernetes security  | Checkov               | PASS             |
| T-03 | Dockerfile security  | Checkov               | PASS             |
| T-04 | CI workflow security | Checkov               | PASS             |
| T-05 | Kubernetes scanning  | Trivy                 | COMPLETED        |
| T-06 | Container scanning   | Trivy                 | COMPLETED        |
| T-07 | SBOM                 | Trivy CycloneDX       | COMPLETED        |
| T-08 | Container hardening  | Manual validation     | PASS             |
| T-09 | Security contexts    | kubectl               | PASS             |
| T-10 | Service accounts     | kubectl               | PASS             |
| T-11 | Network segmentation | NetworkPolicies       | PASS             |
| T-12 | Vault authentication | Kubernetes Auth       | PASS             |
| T-13 | Vault injection      | Vault Agent           | PASS             |
| T-14 | Application/DB path  | API test              | PASS             |
| T-15 | Kyverno policies     | PolicyReports         | PASS / Exception |
| T-16 | Prometheus           | Metrics validation    | PASS             |
| T-17 | Grafana alerts       | Scaling test          | PASS             |
| T-18 | Wazuh agent          | Agent validation      | PASS             |
| T-19 | Wazuh archives       | Archive validation    | PASS             |
| T-20 | Wazuh pipeline       | End-to-end event test | PASS             |
| T-21 | GitHub Actions       | Workflow validation   | PASS             |

---

# 38. Tests Not Performed

The following activities were not part of the final project validation scope:

* Full external penetration test
* Full authenticated web application penetration test
* Production-scale DAST assessment
* Full SAST program
* Enterprise cloud security assessment
* Kubernetes multi-cluster security assessment
* Production HA/DR validation
* Enterprise Wazuh cluster testing
* Autonomous AI-based security response
* Production incident-response exercise

These should not be represented as completed tests.

---

# 39. Why This Distinction Matters

A security project should distinguish between:

```text
Configured
```

and:

```text
Validated
```

It should also distinguish between:

```text
Automated Scan
```

and:

```text
Full Security Assessment
```

For example:

```text
Checkov PASS
```

means the scanned configuration passed the relevant Checkov checks.

It does not mean:

```text
The entire application is vulnerability-free.
```

Similarly:

```text
Trivy scan completed
```

does not mean:

```text
No possible security vulnerabilities exist.
```

Security testing provides evidence and risk reduction, not an absolute guarantee of security.

---

# 40. Testing Limitations

The main limitations of the validation environment are:

### Local Environment

The platform runs locally using Kubernetes Kind.

### Limited Scale

The environment does not represent a production Kubernetes cluster.

### Single Control Plane

The Kind cluster uses a single control-plane node.

### Wazuh Laboratory Deployment

Wazuh is deployed as an all-in-one laboratory environment.

### Monitoring Retention

Prometheus retention is limited to approximately 24 hours.

### MySQL Exception

The MySQL workload has a documented non-root execution exception.

### No Full Application Penetration Test

The documentation does not claim a complete penetration test of the inherited Spring/React application.

---

# 41. Security Testing Philosophy

The project follows a layered testing philosophy:

```text
Detect
   |
   v
Validate
   |
   v
Harden
   |
   v
Re-test
   |
   v
Document
```

A finding is not considered fully addressed simply because a configuration was changed.

The relevant control should be revalidated where practical.

This makes the project evidence-driven rather than configuration-driven.

---

# 42. DevSecOps Testing Lifecycle

The testing lifecycle can be represented as:

```text
Developer Change
      |
      v
Secret Scan
      |
      v
Configuration Scan
      |
      v
Container Scan
      |
      v
Kubernetes Validation
      |
      v
Policy Validation
      |
      v
Runtime Monitoring
      |
      v
Security Event Detection
      |
      v
Investigation
```

This reflects the broader DevSecOps objective of detecting security issues as early as possible while continuing security monitoring throughout the lifecycle.

---

# 43. Final Security Testing Assessment

The project has undergone multi-layer security validation across source control, CI/CD, containers, Kubernetes, networking, secrets, policy enforcement, monitoring, and runtime security.

The strongest validated controls include:

```text
Gitleaks
    ↓
0 detected secrets

Checkov
    ↓
0 failed checks in final scanned frameworks

Trivy
    ↓
324 / 324 Kubernetes resources scanned

Vault
    ↓
Successful secret injection

Kyverno
    ↓
Security policies operational

Grafana
    ↓
Backend availability test completed

Wazuh
    ↓
End-to-end archive event pipeline validated
```

The testing demonstrates that the security controls are operational within the project's local laboratory environment.

It does not claim that the platform is completely vulnerability-free or equivalent to a production enterprise security assessment.

---

# 44. Final Result

The final security-testing outcome can be summarized as:

```text
Source Security
      ✓

CI/CD Security
      ✓

Container Security
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

Documented Exceptions
      ✓
```

The combination of automated scanning, configuration validation, runtime testing, and documented exceptions provides evidence that the DevSecOps security controls were implemented and tested rather than merely configured.

---

# 45. Related Documentation

Additional information is available in:

```text
docs/architecture.md
docs/security-controls.md
docs/zero-trust.md
docs/wazuh.md
docs/monitoring.md
docs/security-exceptions.md
docs/project-report.md
```

The security-testing document should be read together with these documents to understand the architecture, individual controls, Zero Trust implementation, runtime monitoring, and documented limitations.
