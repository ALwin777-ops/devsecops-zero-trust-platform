# DevSecOps Zero Trust Platform

An enterprise-oriented security engineering platform integrating **DevSecOps, Kubernetes security, container security, supply-chain security, and Zero Trust–aligned controls** into a containerized application environment.

The project focuses on embedding security across the software delivery lifecycle, from source-code and dependency analysis through container security, Kubernetes policy enforcement, immutable image deployment, and continuous monitoring.

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
    +-----------------------------+
    |        Security Gates       |
    |                             |
    | Gitleaks  Checkov  Trivy    |
    | npm audit  SBOM              |
    +-------------+---------------+
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
       +----------+----------+
       |          |          |
       v          v          v
   Frontend    Backend     MySQL
                  |
                  |
            NetworkPolicies
                  |
                  v
              Kyverno
                  |
                  v
         Policy Enforcement
                  |
          +-------+-------+
          |               |
          v               v
     Prometheus        Grafana
          |               |
          +------ Alerts--+
```

## Security Capabilities

### DevSecOps & Supply Chain Security

* Gitleaks secret detection
* Checkov Kubernetes/IaC security scanning
* Trivy filesystem and container vulnerability scanning
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
* Read-only root filesystems
* Linux capability reduction
* Privilege-escalation prevention
* RuntimeDefault seccomp profiles
* Isolated temporary writable storage
* Resource requests and limits
* Container health checks

### Kubernetes Security

The Kubernetes environment implements:

* Namespace isolation
* Dedicated ServiceAccounts
* Disabled unnecessary service-account token mounting
* SecurityContexts
* Pod security controls
* Non-root workloads
* Seccomp enforcement
* Resource controls
* Readiness and liveness probes
* PodDisruptionBudgets
* Kubernetes NetworkPolicies

### Network Segmentation

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

### Policy Enforcement

Kyverno provides Kubernetes admission control for security requirements including:

* Non-root execution
* Privilege-escalation prevention
* Seccomp configuration
* Resource requirements
* Container image security

This provides a policy-driven security layer between workload configuration and Kubernetes admission.

### Monitoring & Alerting

Prometheus and Grafana provide Kubernetes monitoring and security-oriented alerting.

Monitoring covers areas such as:

* Workload availability
* Pod restarts
* Node health
* Database availability
* Pending workloads
* Kubernetes resource health

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

The platform implements several Zero Trust–aligned principles:

* Default-deny network access
* Explicit service-to-service communication
* Least-privilege workload execution
* Non-root containers
* Reduced container capabilities
* Disabled unnecessary service-account access
* Admission policy enforcement
* Immutable workload deployment
* Continuous security monitoring

The project currently represents a **Zero Trust–aligned Kubernetes security architecture**. Future identity and workload-authentication controls can further strengthen the model.

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

The platform is designed as a security engineering laboratory and portfolio implementation. Production adoption would require additional controls including centralized identity, production secret management, managed Kubernetes, registry governance, high availability, centralized logging, and disaster recovery.

## Future Security Extensions

Planned extensions include:

* HashiCorp Vault
* Wazuh security monitoring
* OIDC-based workload identity
* Centralized secret management
* Additional cloud-security controls
* Production-oriented deployment validation

These extensions will be introduced where they provide a specific security capability rather than simply increasing the number of security tools.

## Project Origin

The application baseline is derived from the MIT-licensed open-source project:

**dockerized-spring-react-mysql**
by **Jhordy Gavinchu**

The original MIT license and copyright notice are retained in this repository.

This repository extends the application baseline with DevSecOps, Kubernetes security, Zero Trust–aligned controls, monitoring, and security automation.

## Disclaimer

This repository is a security engineering and DevSecOps portfolio project intended for development, experimentation, and validation.

The Kubernetes environment is not intended to represent a production-ready deployment without additional infrastructure, security, reliability, and operational controls.

## License

The original application baseline is distributed under the MIT License.

See the `LICENSE` file for the original copyright notice and license terms.
