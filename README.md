# DevSecOps Zero Trust Platform

An enterprise-oriented security engineering platform integrating **DevSecOps, Kubernetes security, container security, supply-chain security, secrets management, monitoring, and Zero Trust-aligned controls** into a containerized application environment.

The project focuses on embedding security across the software delivery lifecycle, from source-code and dependency analysis through container security, Kubernetes policy enforcement, immutable image deployment, secrets management, and continuous monitoring.

## Architecture

```text
Developer
    |
    v
Git Repository
    |
    v
Security CI/CD Pipeline
    |
    +--------------------------------+
    |         Security Gates         |
    |                                |
    | Gitleaks  Checkov  Trivy       |
    | npm audit  SBOM                |
    +----------------+---------------+
                     |
                     v
                Docker Build
                     |
                     v
              Container Scan
                     |
                     v
                   GHCR
                     |
              Immutable Digest
                     |
                     v
             Kubernetes / Kind
                     |
        +------------+------------+
        |            |            |
        v            v            v
    Frontend      Backend       MySQL
                     |
              +------+------+
              |             |
              v             v
           Kyverno      HashiCorp Vault
              |             |
              v             v
       Policy Controls   Secrets Injection
              |             |
              +------+------+
                     |
                     v
              NetworkPolicies
                     |
                     v
             Prometheus / Grafana
                     |
                     v
                   Alerts
```

## Security Capabilities

### DevSecOps & Supply Chain Security

* Gitleaks secret detection
* Checkov Kubernetes/IaC security scanning
* Trivy filesystem and container vulnerability scanning
* Trivy Kubernetes cluster scanning
* npm dependency auditing
* CycloneDX SBOM generation
* Security gates within CI/CD
* GitHub Container Registry integration
* Immutable container image deployment

The CI/CD pipeline follows a **build, scan, generate SBOM, and publish** workflow. Kubernetes deployments reference registry-backed images using SHA-256 digests rather than mutable application tags.

### Container Security

Application containers are hardened using:

* Non-root execution
* Dedicated application identities
* Read-only root filesystems where supported
* Linux capability reduction
* Privilege-escalation prevention
* RuntimeDefault seccomp profiles
* Isolated temporary writable storage
* Resource requests and limits
* Container health checks

Security scanning is performed using Trivy, with vulnerability findings reviewed before deployment.

## Kubernetes Security

The Kubernetes environment implements:

* Namespace isolation
* Dedicated ServiceAccounts
* SecurityContexts
* Non-root workloads
* Seccomp enforcement
* Resource controls
* Readiness and liveness probes
* PodDisruptionBudgets
* Kubernetes NetworkPolicies
* Kyverno policy controls
* Kubernetes security scanning with Trivy
* IaC/Kubernetes configuration scanning with Checkov

The backend intentionally uses its ServiceAccount token because it is required for **Vault Kubernetes authentication**. This is treated as an explicit architecture requirement rather than unnecessary token mounting.

## Network Segmentation

The application follows a default-deny network model with explicitly permitted communication paths.

```text
Frontend
    |
    | 8080
    v
Backend
    |
    | 3306
    v
MySQL
```

Unnecessary service-to-service communication is restricted through Kubernetes NetworkPolicies.

For example, frontend-to-MySQL communication is not permitted directly. The intended application path is:

```text
Frontend -> Backend -> MySQL
```

## Policy Enforcement

Kyverno provides Kubernetes policy controls for security requirements including:

* Non-root execution
* Privilege-escalation prevention
* Seccomp configuration
* Resource requirements
* Container image security

These policies provide a policy-driven security layer between workload configuration and Kubernetes admission.

## Secrets Management with HashiCorp Vault

HashiCorp Vault is integrated into the Kubernetes environment for application secret management.

The backend uses:

* Vault Kubernetes authentication
* Dedicated Kubernetes ServiceAccount
* Vault policy with least-privilege read access
* Vault Kubernetes authentication role
* Vault Agent Injector
* File-based secret injection
* Spring Boot ConfigTree integration

The runtime secret flow is:

```text
Kubernetes ServiceAccount
          |
          v
   Vault Kubernetes Auth
          |
          v
      Vault Policy
          |
          v
       Vault KV v2
          |
          v
    Vault Agent Injector
          |
          v
      /run/secrets/
          |
          v
   Spring Boot ConfigTree
          |
          v
       HikariCP
          |
          v
        MySQL
```

Database credentials are not stored directly in the Kubernetes Deployment manifest.

The backend is authorized only to read the required Vault secret path:

```text
secret/data/devsecops/backend
```

The Vault integration was validated by confirming successful authentication, secret injection, Spring Boot configuration loading, database connection, and application startup.

## Checkov Security Scanning

Checkov is used to validate Kubernetes and infrastructure security configuration.

The Kubernetes configuration was scanned with **Checkov 3.3.16**.

Latest project scan result:

```text
Passed checks: 261
Failed checks: 0
Skipped checks: 10
```

`CKV_K8S_38` is intentionally excluded from the scan because the backend requires its ServiceAccount token for Vault Kubernetes authentication.

The exclusion represents a documented architecture requirement rather than disabling the broader Kubernetes security scan.

## Trivy Kubernetes Security Scanning

The Kind Kubernetes environment is also scanned using Trivy.

The cluster scan included:

* Kubernetes workloads
* Kubernetes configuration
* Node-related resources
* Security configuration

The completed cluster scan evaluated the full discovered resource set in the development environment.

## SBOM

CycloneDX SBOM generation is used to provide software component visibility.

A backend image SBOM is generated as:

```text
backend-sbom.json
```

The SBOM provides a machine-readable inventory of application and operating-system components contained in the image.

## Monitoring & Alerting

Prometheus and Grafana provide Kubernetes monitoring and security-oriented alerting.

Monitoring covers areas such as:

* Workload availability
* Pod restarts
* Node health
* Database availability
* Pending workloads
* Kubernetes resource health

Configured Grafana alerts include conditions such as:

* Backend deployment replicas becoming unavailable
* Backend pod restart detection

Grafana provides dashboards and alerting for operational and security-relevant events.

## Immutable Deployment Model

Application images are published to GitHub Container Registry and deployed to Kubernetes using immutable SHA-256 digests.

```text
Source Code
     |
     v
Security Checks
     |
     v
Container Build
     |
     v
Trivy + SBOM
     |
     v
GHCR
     |
     v
SHA-256 Digest
     |
     v
Kubernetes
```

This provides a controlled relationship between the artifact that passes the CI/CD security pipeline and the artifact deployed to Kubernetes.

## Zero Trust Alignment

The platform implements several Zero Trust-aligned principles:

* Default-deny network access
* Explicit service-to-service communication
* Least-privilege workload execution
* Non-root containers
* Reduced container capabilities
* Explicit ServiceAccount usage
* Admission policy enforcement
* Immutable workload deployment
* Centralized secrets management
* Continuous security monitoring

The project currently represents a **Zero Trust-aligned Kubernetes security architecture** rather than a complete enterprise Zero Trust implementation.

Additional enterprise controls such as centralized identity, workload identity, production-grade logging, managed infrastructure, high availability, and disaster recovery would be required for a production deployment.

## Technology Stack

| Area                     | Technologies              |
| ------------------------ | ------------------------- |
| Containers               | Docker                    |
| Orchestration            | Kubernetes, Kind          |
| CI/CD                    | GitHub Actions            |
| Container Registry       | GitHub Container Registry |
| Secret Scanning          | Gitleaks                  |
| IaC Security             | Checkov                   |
| Vulnerability Management | Trivy                     |
| SBOM                     | CycloneDX                 |
| Secrets Management       | HashiCorp Vault           |
| Policy Enforcement       | Kyverno                   |
| Monitoring               | Prometheus                |
| Visualization & Alerting | Grafana                   |
| Application              | Spring Boot, React, MySQL |

## Kubernetes Environment

The current development environment is built using:

* Kubernetes
* Kind
* kubectl
* Helm
* Kyverno
* Prometheus
* Grafana
* HashiCorp Vault
* Trivy
* Checkov

The platform is designed as a security engineering laboratory and portfolio implementation.

Production adoption would require additional controls including:

* Centralized identity and access management
* Production-grade secrets management
* Managed Kubernetes
* Registry governance
* High availability
* Centralized logging
* Security information and event management
* Disaster recovery
* Production observability
* Infrastructure redundancy

## Future Security Extensions

Planned extensions include:

* Wazuh security monitoring
* Additional cloud-security controls
* Centralized security event correlation
* Production-oriented deployment validation
* Additional Kubernetes security policies

Future extensions will be introduced where they provide a specific security capability rather than simply increasing the number of security tools.

## Project Origin

The application baseline is derived from the MIT-licensed open-source project:

**dockerized-spring-react-mysql**
by **Jhordy Gavinchu**

The original MIT license and copyright notice are retained in this repository.

This repository extends the application baseline with DevSecOps, Kubernetes security, Zero Trust-aligned controls, secrets management, monitoring, and security automation.

## Disclaimer

This repository is a security engineering and DevSecOps portfolio project intended for development, experimentation, and validation.

The Kubernetes environment is not intended to represent a production-ready deployment without additional infrastructure, security, reliability, and operational controls.

## License

The original application baseline is distributed under the MIT License.

See the `LICENSE` file for the original copyright notice and license terms.
