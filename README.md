# DevSecOps Zero Trust Platform

An enterprise-oriented DevSecOps and Zero Trust security platform built around a containerized web application and Kubernetes.

The project focuses on integrating security throughout the software delivery lifecycle, from source-code and container security to Kubernetes policy enforcement, vulnerability management, infrastructure-as-code scanning, monitoring, and security-focused CI/CD.

## Project Origin

The application baseline in this repository is derived from the MIT-licensed open-source project:where 

**dockerized-spring-react-mysql** by **Jhordy Gavinchu**

The original MIT license and copyright notice are retained in this repository.

This repository extends that application baseline with additional DevSecOps, Kubernetes security, Zero Trust, monitoring, and security automation work.

## Security Engineering Implemented

### Source & Supply Chain Security

* Gitleaks secret scanning
* Trivy vulnerability scanning
* CycloneDX SBOM generation
* Checkov Kubernetes/IaC security scanning

### Container Security

* Non-root application containers
* Reduced Linux capabilities
* `no-new-privileges`
* Read-only root filesystem where applicable
* Temporary writable filesystem using `tmpfs` / `emptyDir`
* Container vulnerability remediation
* Container health checks

### Kubernetes Security

* Kubernetes deployments and services
* Namespace isolation
* Kubernetes NetworkPolicies
* Default-deny network segmentation
* Pod security hardening
* SecurityContext configuration
* Seccomp `RuntimeDefault`
* Service account hardening
* Disabled unnecessary service-account token mounting
* Resource requests and limits
* Startup, readiness, and liveness probes
* PodDisruptionBudgets

### Policy Enforcement

Kyverno is used to validate Kubernetes security requirements including:

* Non-root execution
* Privilege-escalation prevention
* Seccomp configuration
* Resource requirements
* Container image tag controls

### Monitoring & Alerting

* Prometheus
* Grafana
* Kubernetes security and availability metrics
* Grafana dashboard provisioning
* Kubernetes alert rules
* Backend availability monitoring
* Pod restart detection
* Node health monitoring
* MySQL availability monitoring
* Pending workload detection

## Current Architecture

```text
Developer
    |
    v
Git Repository
    |
    v
Security-focused CI/CD
    |
    +------------------+
    |                  |
    v                  v
Gitleaks            Checkov
    |                  |
    +--------+---------+
             |
             v
          Trivy
             |
             v
        Docker Images
             |
             v
      Kubernetes / Kind
             |
     +-------+-------+
     |       |       |
     v       v       v
 Frontend Backend  MySQL
     |       |
     |       |
     +---NetworkPolicies---+
             |
             v
          Kyverno
             |
             v
     Policy Validation
             |
             +----------------+
             |                |
             v                v
        Prometheus         Grafana
             |                |
             +-------> Alerts
```

## Kubernetes Environment

The current development environment uses:

* Kubernetes
* Kind
* kubectl
* Helm
* Kyverno
* Prometheus
* Grafana

The project is designed as a local security engineering laboratory and portfolio implementation. Production deployments would require additional controls such as managed Kubernetes, registry-backed immutable images, centralized identity, production secret management, and external monitoring.

## Security Pipeline

The planned CI/CD security workflow will integrate:

1. Secret scanning
2. IaC security scanning
3. Filesystem vulnerability scanning
4. Container image scanning
5. SBOM generation
6. Security gates
7. Build validation

## Future Security Extensions

Planned extensions include:

* HashiCorp Vault for centralized secret management and credential rotation
* Wazuh for security monitoring and detection
* OIDC-based identity and authentication
* Additional cloud-security controls
* Production-oriented CI/CD deployment validation

These components will be introduced where they provide a specific security function rather than simply increasing the number of tools.

## Disclaimer

This repository is primarily a security engineering and DevSecOps laboratory/portfolio project. The Kubernetes environment is intended for development and validation rather than production deployment.

## License

The original application baseline is distributed under the MIT License. See the `LICENSE` file for the original copyright notice and license terms.
