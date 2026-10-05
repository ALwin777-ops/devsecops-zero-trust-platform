# Threat Model — DevSecOps Zero Trust Platform

## 1. System Scope

### 1.1 Purpose

This threat model evaluates the security architecture of the DevSecOps and Zero Trust Kubernetes platform implemented in this repository.

The objective is to identify realistic threats against application workloads, secrets, data flows, Kubernetes resources, CI/CD components, and security monitoring infrastructure, and to map those threats to implemented security controls and validation evidence.

The threat model follows the STRIDE methodology and is based on the actual architecture and security controls implemented in the project.

### 1.2 System Under Analysis

The system consists of:

- A containerized Spring Boot backend
- A React frontend
- A MySQL database
- Kubernetes workloads running on a local Kind cluster
- Ingress-NGINX for application ingress
- Kubernetes NetworkPolicies for workload segmentation
- Kyverno admission policies
- HashiCorp Vault for application secret management
- Kubernetes ServiceAccount-based workload identity
- Prometheus and Grafana for monitoring and alerting
- Wazuh for security event collection and detection
- GitHub Actions for CI/CD security checks
- Gitleaks for secret scanning
- Checkov for Kubernetes, Dockerfile, and GitHub Actions security analysis
- Trivy for container and Kubernetes vulnerability scanning
- SBOM generation for container images

### 1.3 Primary Security Objectives

The threat model focuses on protecting:

- Confidentiality of application secrets and database credentials
- Integrity of application images, Kubernetes manifests, and security policies
- Availability of application workloads and supporting security services
- Isolation between application components
- Kubernetes workload identities and authorization boundaries
- Security telemetry and audit visibility
- Software supply-chain integrity
- Prevention of container privilege escalation and unauthorized access

### 1.4 Threat-Model Boundaries

The primary trust boundaries considered are:

1. Developer and source-control environment to CI/CD
2. CI/CD pipeline to container artifacts
3. External traffic to Kubernetes ingress
4. Frontend to backend
5. Backend to MySQL
6. Backend to Vault
7. Kubernetes workloads to the Kubernetes control plane
8. Application workloads to monitoring and security infrastructure

### 1.5 Environment

The implementation is a security engineering and validation lab rather than a production deployment.

The Kubernetes environment runs on a local Kind cluster. Vault uses file storage and is configured without high availability. These characteristics are explicitly treated as environmental limitations and residual risks rather than being represented as production-grade infrastructure.

### 1.6 Out of Scope

The following are outside the scope of this threat model:

- Production cloud infrastructure deployment
- Cloud-provider IAM architecture
- Production Kubernetes high-availability design
- Physical infrastructure security
- Organization-wide identity governance
- Business continuity and disaster recovery planning
- Application business-logic threat modeling beyond the security controls implemented in this project
- Third-party SaaS security assessments
- Production TLS certificate management
- Full enterprise compliance assessment

### 1.7 Threat Modeling Approach

The model uses:

- Data-flow and trust-boundary analysis to decompose the system
- STRIDE for threat identification
- Risk-based prioritization
- Mapping of threats to implemented preventive, detective, and monitoring controls
- Validation evidence from Kubernetes, Vault, CI/CD, scanning, monitoring, and Wazuh testing

The model is intended to be maintained as the system architecture and security controls evolve.

## 2. Assets

Assets are the resources that require protection from unauthorized access, modification, disclosure, or disruption.

### 2.1 Critical Assets

| ID | Asset | Security Importance |
|---|---|---|
| A-01 | Vault-managed application secrets | Contains database credentials used by the backend application |
| A-02 | MySQL application data | Contains persistent application data and requires confidentiality and integrity |
| A-03 | Kubernetes workload identity | Includes the `backend-sa` ServiceAccount and its trust relationship with Vault |
| A-04 | Vault authorization configuration | Includes the Kubernetes auth role and policy controlling backend access to secrets |

### 2.2 High-Value Assets

| ID | Asset | Security Importance |
|---|---|---|
| A-05 | Backend container image | Executes application code and is a primary workload security boundary |
| A-06 | Frontend container image | Serves the application frontend and handles user-facing traffic |
| A-07 | Kubernetes manifests | Define workloads, services, security contexts, NetworkPolicies, and other security controls |
| A-08 | Kyverno security policies | Enforce workload security requirements at admission |
| A-09 | CI/CD workflow | Controls automated security validation of repository changes |
| A-10 | Source code repository | Contains application code, infrastructure definitions, policies, and security configuration |
| A-11 | Wazuh security telemetry | Provides security event visibility and detection data |
| A-12 | Kubernetes control plane | Controls workload scheduling, API access, and cluster state |

### 2.3 Monitoring and Security Evidence

| ID | Asset | Security Importance |
|---|---|---|
| A-13 | Prometheus monitoring data | Provides workload and infrastructure health metrics |
| A-14 | Grafana alert configuration | Defines detection and alerting for workload availability and restart conditions |
| A-15 | Trivy vulnerability results | Provides vulnerability assessment evidence for images and Kubernetes resources |
| A-16 | SBOM artifacts | Provide software-component inventory and supply-chain visibility |
| A-17 | Gitleaks scan results | Provide evidence that repository history was checked for exposed secrets |
| A-18 | Checkov scan results | Provide security validation evidence for Kubernetes, Dockerfile, and GitHub Actions configurations |

### 2.4 Asset Security Priorities

The highest-priority assets are the assets that could enable an attacker to move from a compromised workload to sensitive data or additional infrastructure.

The primary high-impact attack paths considered by this threat model are:

1. Compromise of the backend workload leading to unauthorized database access
2. Compromise or misuse of Kubernetes workload identity leading to unauthorized Vault access
3. Exposure of Vault-managed credentials leading to database compromise
4. Tampering with source code or CI/CD workflows leading to malicious container artifacts
5. Privilege escalation within a container leading to broader workload or node compromise
6. Compromise of Kubernetes configuration leading to weakened security controls or unauthorized workload access
7. Loss or manipulation of security telemetry reducing the ability to detect malicious activity

### 2.5 Asset Protection Goals

| Asset Category | Primary Protection Goal |
|---|---|
| Secrets | Confidentiality |
| Database | Confidentiality + Integrity + Availability |
| Workload identity | Integrity + Authorization |
| Container images | Integrity |
| Source code | Integrity |
| Kubernetes configuration | Integrity |
| Security policies | Integrity + Enforcement |
| Monitoring data | Integrity + Availability |
| Security telemetry | Integrity + Availability |
| Scan/SBOM evidence | Integrity + Traceability |



## 3. Architecture and Data Flow

### 3.1 High-Level Architecture

The platform consists of application, infrastructure, security-control, monitoring, and CI/CD layers.

```text
                         ┌──────────────────────┐
                         │   Developer / User    │
                         └──────────┬───────────┘
                                    │
                                    │ Source / Requests
                                    ▼
                         ┌──────────────────────┐
                         │   GitHub Repository   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │   GitHub Actions      │
                         │ Gitleaks / Checkov    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │   Container Images    │
                         │   Digest-pinned       │
                         └──────────┬───────────┘
                                    │
                             Trivy / SBOM
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         KIND KUBERNETES CLUSTER                            │
│                                                                           │
│  ┌───────────────┐      ┌────────────────┐      ┌─────────────────────┐  │
│  │ Ingress-NGINX │─────▶│ React Frontend │─────▶│ Spring Boot Backend │  │
│  └───────────────┘      └────────────────┘      └──────────┬──────────┘  │
│                                                             │             │
│                                      ┌──────────────────────┼──────────┐  │
│                                      │                      │          │  │
│                                      ▼                      ▼          │  │
│                               ┌──────────────┐       ┌──────────────┐ │  │
│                               │    MySQL     │       │    Vault     │ │  │
│                               │ Application  │       │   Secrets    │ │  │
│                               │    Data      │       │              │ │  │
│                               └──────────────┘       └──────────────┘ │  │
│                                                                           │
│  Security Controls:                                                      │
│  • NetworkPolicies                                                       │
│  • Kyverno admission policies                                            │
│  • Container security contexts                                           │
│  • Kubernetes ServiceAccounts                                             │
│                                                                           │
│  Monitoring:                                                             │
│  • Prometheus                                                            │
│  • Grafana                                                               │
└───────────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────────────┐
                    │       Wazuh Security         │
                    │ Agent → Manager → Indexer     │
                    │          → Dashboard          │
                    └──────────────────────────────┘
```

### 3.2 Primary Data Flows

| ID    | Source               | Destination       | Data / Interaction                                 | Security Control                      |
| ----- | -------------------- | ----------------- | -------------------------------------------------- | ------------------------------------- |
| DF-01 | Developer            | GitHub            | Source code, manifests, policies                   | Repository access controls            |
| DF-02 | GitHub               | GitHub Actions    | Repository changes                                 | `contents: read` workflow permissions |
| DF-03 | GitHub Actions       | Security scanners | Source and configuration analysis                  | Gitleaks + Checkov                    |
| DF-04 | Container images     | Trivy             | Vulnerability analysis                             | Trivy scanning                        |
| DF-05 | Container images     | SBOM generation   | Software component inventory                       | CycloneDX SBOM                        |
| DF-06 | External client      | Ingress-NGINX     | HTTP application traffic                           | Ingress boundary                      |
| DF-07 | Ingress-NGINX        | Frontend          | HTTP traffic                                       | NetworkPolicy                         |
| DF-08 | Frontend             | Backend           | API requests                                       | NetworkPolicy                         |
| DF-09 | Backend              | MySQL             | Database queries                                   | Backend-to-MySQL NetworkPolicy        |
| DF-10 | Backend              | Vault             | Kubernetes-authenticated secret request            | Vault Kubernetes Auth + NetworkPolicy |
| DF-11 | Vault                | Backend           | Database credentials through Vault Agent injection | Vault policy + injector               |
| DF-12 | Kubernetes workloads | Prometheus        | Metrics                                            | Monitoring configuration              |
| DF-13 | Prometheus           | Grafana           | Monitoring data                                    | Prometheus datasource                 |
| DF-14 | Windows endpoint     | Wazuh Manager     | Security events                                    | Wazuh agent                           |
| DF-15 | Wazuh Manager        | Wazuh Indexer     | Security telemetry/archive data                    | Wazuh archive pipeline                |

### 3.3 Vault Secret Flow

The backend secret flow is modeled separately because database credentials represent a critical asset.

```text
Backend Pod
    │
    │ Kubernetes workload identity
    ▼
backend-sa
    │
    ▼
Vault Kubernetes Auth
    │
    │ Role binding
    ▼
devsecops-backend
    │
    │ Policy
    ▼
devsecops-backend policy
    │
    │ read
    ▼
secret/data/devsecops/backend
    │
    │ Vault Agent Injector
    ▼
/run/secrets/
    ├── db_user
    └── db_password
    │
    ▼
Spring Boot Backend
    │
    ▼
MySQL
```

The Vault role is restricted to the `backend-sa` ServiceAccount in the `devsecops` namespace, with a one-hour token TTL and an explicit Kubernetes audience.

The associated Vault policy grants read access only to:

```text
secret/data/devsecops/backend
```

This creates an explicit identity and authorization chain rather than relying solely on network location.

### 3.4 Container Security Boundary

The backend workload represents a security boundary between application code and the underlying container runtime.

The backend container is configured with:

* `runAsNonRoot: true`
* `runAsUser: 10001`
* `runAsGroup: 10001`
* `allowPrivilegeEscalation: false`
* All Linux capabilities dropped
* `readOnlyRootFilesystem: true`
* `seccompProfile: RuntimeDefault`
* Writable temporary storage isolated to `/tmp`

These controls reduce the potential impact of application compromise and restrict common container privilege-escalation paths.

### 3.5 Network Segmentation

The application namespace uses Kubernetes NetworkPolicies to restrict communication between workloads.

The primary permitted application flows are:

* Ingress-NGINX → Frontend
* Ingress-NGINX → Backend
* Frontend → Backend
* Backend → MySQL
* Backend → Vault
* Backend → DNS

A default-deny ingress policy is applied to the `devsecops` namespace.

Backend egress is additionally restricted to required DNS, MySQL, and Vault destinations.

The threat model recognizes that frontend and MySQL workloads do not currently have the same broad egress restrictions as the backend. This is treated as a residual risk rather than being represented as complete network isolation.

### 3.6 Trust Boundary Overview

| ID    | Boundary                                | Security Significance             |
| ----- | --------------------------------------- | --------------------------------- |
| TB-01 | Developer → GitHub                      | Source-code integrity boundary    |
| TB-02 | GitHub → GitHub Actions                 | CI/CD execution boundary          |
| TB-03 | CI/CD → Container Artifact              | Software supply-chain boundary    |
| TB-04 | External Client → Ingress               | Application entry boundary        |
| TB-05 | Ingress → Application Workloads         | Kubernetes ingress boundary       |
| TB-06 | Frontend → Backend                      | Application service boundary      |
| TB-07 | Backend → MySQL                         | Data-store authorization boundary |
| TB-08 | Backend → Vault                         | Secret-management trust boundary  |
| TB-09 | Workloads → Kubernetes Control Plane    | Cluster authorization boundary    |
| TB-10 | Workloads → Monitoring/Security Systems | Security telemetry boundary       |

### 3.7 Architecture Modeling Principle

The architecture is modeled around data flows and trust boundaries rather than simply listing deployed technologies.

Each boundary represents a point where authentication, authorization, isolation, integrity, confidentiality, or monitoring controls may be required.

This architecture model provides the foundation for the STRIDE threat analysis in the following section.



## 4. Trust Boundaries and Attack Surface

The attack surface is evaluated from the perspective of an attacker attempting to enter the system, cross trust boundaries, access protected assets, or move from a compromised component to another component.

The analysis considers both external and internal attack paths. Internal communication is not automatically treated as trusted; each boundary is evaluated according to the identity, authorization, network controls, and security assumptions applied at that point.

### 4.1 External Attack Surface

The primary externally reachable application path is:

```text
External Client
      |
      v
Ingress-NGINX
      |
      +------> React Frontend
      |
      +------> Spring Boot Backend
```

The external attack surface includes:

* HTTP application traffic entering through Ingress-NGINX
* Frontend HTTP requests
* Backend API requests
* User-supplied application input reaching the frontend and backend
* Kubernetes ingress configuration controlling external application exposure

The primary security boundary is the transition from an untrusted external client to the Kubernetes application workloads.

Implemented controls include:

* Ingress-NGINX as the controlled application entry point
* Kubernetes NetworkPolicies restricting ingress to application workloads
* Container security contexts
* Trivy vulnerability scanning
* Checkov configuration analysis
* Gitleaks secret scanning
* Wazuh security monitoring

### 4.2 Internal Application Attack Surface

After an attacker reaches an application workload, the following internal paths become relevant:

```text
Frontend
   |
   v
Backend
   |
   +------> MySQL
   |
   +------> Vault
```

The frontend-to-backend boundary is protected through an explicit NetworkPolicy.

The backend-to-MySQL boundary is restricted through a NetworkPolicy allowing the backend workload to communicate with MySQL on the required database port.

The backend-to-Vault boundary is restricted through a NetworkPolicy and additionally protected through Vault Kubernetes authentication and authorization.

The principal security concern is preventing compromise of one workload from automatically providing unrestricted access to other workloads or sensitive infrastructure.

### 4.3 Kubernetes Control-Plane Attack Surface

Kubernetes introduces an additional high-value trust boundary between application workloads and the Kubernetes control plane.

A compromised workload may attempt to:

* Access the Kubernetes API
* Abuse its ServiceAccount identity
* Obtain credentials or tokens
* Discover cluster resources
* Modify Kubernetes resources
* Escalate privileges through excessive permissions
* Use a compromised workload as a pivot toward other cluster components

The project reduces this risk through:

* Dedicated Kubernetes ServiceAccounts
* Explicit workload identity
* Kyverno admission policies
* Kubernetes security contexts
* NetworkPolicies
* Least-privilege Vault Kubernetes authentication
* Container privilege restrictions

The Kubernetes control plane remains a high-value asset and is therefore treated as a critical trust boundary even though this project does not implement a production Kubernetes control-plane security architecture.

### 4.4 Vault Attack Surface

Vault represents a high-value secrets-management boundary.

The primary attack path is:

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
devsecops-backend Policy
     |
     v
secret/data/devsecops/backend
```

Access is restricted to the `backend-sa` ServiceAccount in the `devsecops` namespace.

The associated policy grants read access only to the required secret path.

The role uses a one-hour token TTL and an explicit Kubernetes audience.

This limits the blast radius of a compromised workload identity compared with unrestricted access to the Vault instance.

However, compromise of the backend workload remains a high-impact scenario because the backend is intentionally authorized to retrieve the database credentials required by the application.

### 4.5 CI/CD and Software Supply-Chain Attack Surface

The software supply-chain boundary is:

```text
Developer
    |
    v
GitHub Repository
    |
    v
GitHub Actions
    |
    v
Container Images
    |
    +------> Trivy
    |
    +------> SBOM
    |
    v
Kubernetes Deployment
```

An attacker who gains the ability to modify repository code, workflow files, manifests, or container build inputs may attempt to introduce malicious code or weaken security controls.

The project applies several controls to this boundary:

* GitHub Actions security workflow
* `contents: read` workflow permissions
* Gitleaks secret scanning
* Checkov analysis
* Trivy vulnerability scanning
* SBOM generation
* Digest-pinned container images
* Kubernetes admission controls through Kyverno

The supply-chain boundary remains important because compromise before deployment can result in a trusted deployment carrying malicious code.

### 4.6 Monitoring and Security Telemetry Attack Surface

The platform also contains security monitoring infrastructure:

```text
Kubernetes Workloads
        |
        v
   Prometheus
        |
        v
      Grafana


Windows Endpoint
        |
        v
   Wazuh Agent
        |
        v
   Wazuh Manager
        |
        v
   Wazuh Indexer
        |
        v
 Wazuh Dashboard
```

Monitoring and telemetry are treated as security-sensitive assets because loss or manipulation of monitoring data can reduce detection capability.

The project therefore considers:

* Prometheus availability and monitoring data
* Grafana alert configuration
* Wazuh event collection
* Wazuh archive data
* Security scan results

as part of the defensive attack surface.

### 4.7 Trust Boundary and Attack Surface Matrix

| ID    | Trust Boundary                          | Potential Attacker Capability                         | Primary Asset at Risk          | Existing Controls                                 | Residual Exposure                                                  |
| ----- | --------------------------------------- | ----------------------------------------------------- | ------------------------------ | ------------------------------------------------- | ------------------------------------------------------------------ |
| TB-01 | External Client → Ingress               | Send malicious or unexpected application traffic      | Frontend / Backend             | Ingress-NGINX, NetworkPolicies                    | Application-level vulnerabilities remain possible                  |
| TB-02 | Ingress → Application Workloads         | Attempt unauthorized workload access                  | Application workloads          | NetworkPolicies                                   | Application authorization remains important                        |
| TB-03 | Frontend → Backend                      | Manipulate API requests or exploit backend weaknesses | Backend / Application data     | NetworkPolicy                                     | Backend application vulnerabilities remain possible                |
| TB-04 | Backend → MySQL                         | Abuse compromised backend access                      | MySQL data                     | NetworkPolicy, restricted application path        | Backend compromise could expose authorized database access         |
| TB-05 | Backend → Vault                         | Abuse workload identity to obtain secrets             | Vault secrets                  | Kubernetes Auth, Vault policy, TTL, NetworkPolicy | Compromised backend identity may retrieve authorized secrets       |
| TB-06 | Workload → Kubernetes Control Plane     | Abuse ServiceAccount or Kubernetes API access         | Cluster state / control plane  | ServiceAccounts, Kyverno, security contexts       | Production RBAC/API hardening is outside project scope             |
| TB-07 | Developer → GitHub                      | Modify source, manifests, or security configuration   | Source repository              | Repository controls, Gitleaks                     | Repository-account compromise remains possible                     |
| TB-08 | GitHub → GitHub Actions                 | Modify or abuse CI/CD execution                       | CI/CD integrity                | Restricted workflow permissions, Checkov          | CI/CD platform remains a high-value target                         |
| TB-09 | CI/CD → Container Artifact              | Introduce malicious or vulnerable image content       | Container images               | Checkov, Trivy, SBOM, digest pinning              | A compromised trusted build path may bypass downstream assumptions |
| TB-10 | Workloads → Monitoring/Security Systems | Disrupt or manipulate telemetry                       | Monitoring / security evidence | Prometheus, Grafana, Wazuh                        | Monitoring infrastructure requires independent hardening           |

### 4.8 Attack Surface Prioritization

The attack surface is prioritized according to potential impact and the ability of an attacker to reach sensitive assets.

#### Critical

* Backend → Vault
* Backend → MySQL
* Workload → Kubernetes control plane
* GitHub → GitHub Actions
* CI/CD → Container Artifact

These boundaries can potentially enable access to secrets, application data, cluster resources, or trusted software artifacts.

#### High

* External Client → Ingress
* Frontend → Backend
* Developer → GitHub
* Workloads → Monitoring/Security Systems

These boundaries represent important entry points or security-control dependencies.

#### Medium

* Monitoring data flows between Prometheus and Grafana
* Wazuh telemetry and archive flows
* SBOM and vulnerability-report storage

These remain security-relevant but generally have lower direct impact on application confidentiality than the critical boundaries.

### 4.9 Attack Surface Management Principle

The project treats every externally reachable interface, workload-to-workload communication path, identity transition, secret-access path, CI/CD transition, and security-monitoring connection as an attack-surface element.

The objective is not to eliminate every attack surface but to ensure that each important boundary has an explicit security control, a defined trust assumption, and a documented residual risk.

Changes to application architecture, Kubernetes permissions, NetworkPolicies, Vault authorization, CI/CD workflows, or exposed interfaces should trigger a review of this threat model.


## 5. STRIDE Threat Analysis

The STRIDE analysis evaluates threats against the components, data flows, trust boundaries, and assets identified in the preceding sections.

Each threat is evaluated according to:

* Attack scenario
* Affected asset or boundary
* Potential impact
* Implemented security controls
* Validation evidence
* Residual risk

### 5.1 Spoofing

Spoofing threats involve an attacker attempting to impersonate a legitimate user, workload, service, or trusted identity.

For this platform, the most significant spoofing scenarios involve Kubernetes workload identity, Vault authentication, CI/CD identity, and application-facing identities.

#### S-01: Kubernetes ServiceAccount Identity Spoofing

**Threat:**
An attacker who compromises a workload may attempt to obtain or reuse Kubernetes credentials associated with a trusted ServiceAccount in order to impersonate that workload.

**Attack Scenario:**

```text
Compromised Workload
        |
        | Attempts to obtain/reuse credentials
        v
Kubernetes ServiceAccount Identity
        |
        v
Kubernetes API / Vault Authentication
```

A successful identity compromise could allow an attacker to act with the permissions associated with the compromised workload identity.

**Affected Assets:**

* A-03 Kubernetes workload identity
* A-04 Vault authorization configuration
* A-12 Kubernetes control plane

**Affected Boundary:**

* TB-06 Workload → Kubernetes Control Plane
* TB-08 Backend → Vault

**Risk:** High

**Implemented Controls:**

* Dedicated `backend-sa` ServiceAccount
* Vault Kubernetes authentication bound specifically to `backend-sa`
* Vault role restricted to the `devsecops` namespace
* Explicit Kubernetes audience configured for the Vault role
* One-hour Vault token TTL
* Least-privilege Vault policy
* Container `runAsNonRoot`
* `allowPrivilegeEscalation: false`
* All Linux capabilities dropped
* Read-only root filesystem
* Kyverno security policies

**Validation Evidence:**

The Vault Kubernetes authentication role was verified to contain:

```text
bound_service_account_names = [backend-sa]
bound_service_account_namespaces = [devsecops]
audience = https://kubernetes.default.svc.cluster.local
token_ttl = 1h
```

The associated Vault policy grants only read access to:

```text
secret/data/devsecops/backend
```

**Residual Risk:**

A fully compromised backend workload may still be able to use its legitimate workload identity and obtain the secrets that the backend is intentionally authorized to access.

The implemented controls therefore reduce the scope and lifetime of the identity rather than eliminating the impact of a complete backend compromise.

---

#### S-02: Vault Role Impersonation

**Threat:**
An attacker may attempt to authenticate to Vault while presenting a Kubernetes identity that should not be trusted by the `devsecops-backend` role.

**Attack Scenario:**

```text
Attacker-controlled workload
        |
        | Forged/unauthorized Kubernetes identity
        v
Vault Kubernetes Auth
        |
        X
devsecops-backend role
```

If Vault accepted an unauthorized identity, the attacker could potentially obtain database credentials.

**Affected Assets:**

* A-01 Vault-managed application secrets
* A-04 Vault authorization configuration

**Risk:** High

**Implemented Controls:**

* Kubernetes authentication
* ServiceAccount-specific role binding
* Namespace-specific role binding
* Explicit Kubernetes audience
* One-hour token TTL
* Least-privilege Vault policy
* Backend-to-Vault NetworkPolicy

**Validation Evidence:**

The configured role was verified to restrict authentication to:

```text
ServiceAccount: backend-sa
Namespace: devsecops
```

The policy was verified to provide only:

```text
read
```

access to:

```text
secret/data/devsecops/backend
```

**Residual Risk:**

If an attacker fully compromises the legitimate `backend-sa` identity, Vault will correctly recognize that identity as trusted. The remaining risk is therefore dependent on protecting the workload and its Kubernetes credentials.

---

#### S-03: CI/CD Identity Compromise

**Threat:**
An attacker who compromises a developer account, repository credentials, or CI/CD execution context may attempt to impersonate a trusted repository contributor or modify the security pipeline.

**Attack Scenario:**

```text
Compromised Developer / Repository Identity
                  |
                  v
          GitHub Repository
                  |
                  v
           GitHub Actions
                  |
                  v
       Trusted Build / Security Pipeline
```

Successful impersonation could allow unauthorized source-code, manifest, workflow, or security-configuration changes.

**Affected Assets:**

* A-09 CI/CD workflow
* A-10 source repository
* A-05 backend image
* A-06 frontend image

**Affected Boundaries:**

* TB-01 Developer → GitHub
* TB-02 GitHub → GitHub Actions
* TB-03 CI/CD → Container Artifact

**Risk:** High

**Implemented Controls:**

* GitHub Actions security pipeline
* Restricted workflow `contents: read` permission
* Gitleaks secret scanning
* Checkov security analysis
* Trivy scanning
* SBOM generation
* Digest-pinned container images

**Residual Risk:**

Compromise of a legitimate developer or repository-maintainer identity remains an important supply-chain threat. Repository-level authentication and organizational identity controls are outside the scope of this local security engineering project.

---

#### S-04: Application User or Session Identity Spoofing

**Threat:**
An attacker may attempt to impersonate a legitimate application user through compromised credentials, session information, or weaknesses in application-level authentication.

**Affected Assets:**

* Application data
* Backend API
* MySQL application data

**Risk:** Medium

**Project Boundary:**

Application business-logic authentication and authorization are outside the primary scope of this project.

The threat model therefore recognizes application-level identity spoofing as a residual application-security concern rather than claiming that the Kubernetes security controls prevent it.

**Residual Risk:**

An attacker who successfully authenticates as another application user could potentially perform actions available to that identity.

Application-level authentication, session management, authorization, and business-logic security require separate application-security testing.

---

### 5.1.1 Spoofing Risk Summary

| ID   | Spoofing Threat                             | Primary Asset                     | Risk   | Main Controls                                          | Residual Risk                                        |
| ---- | ------------------------------------------- | --------------------------------- | ------ | ------------------------------------------------------ | ---------------------------------------------------- |
| S-01 | Kubernetes ServiceAccount identity spoofing | Workload identity / control plane | High   | Dedicated SA, security context, Kyverno                | Compromised workload may use its legitimate identity |
| S-02 | Vault role impersonation                    | Vault secrets                     | High   | K8s Auth, SA binding, namespace binding, audience, TTL | Legitimate compromised identity remains trusted      |
| S-03 | CI/CD identity compromise                   | Source / CI/CD / images           | High   | GitHub Actions, Gitleaks, Checkov, Trivy               | Repository identity compromise remains possible      |
| S-04 | Application user/session spoofing           | Application data                  | Medium | Outside primary project scope                          | Requires separate application-security controls      |

### 5.1.2 Spoofing Conclusion

The platform provides strong workload-level identity controls for the backend-to-Vault trust boundary. The Vault authentication model uses explicit ServiceAccount and namespace binding, an explicit Kubernetes audience, a limited token lifetime, and a least-privilege policy.

The primary residual spoofing risk is compromise of a legitimate identity rather than unrestricted acceptance of arbitrary identities.

Application-level identity threats and enterprise repository identity governance remain outside the project's primary scope and should be addressed through dedicated application-security and organizational identity controls.


### 5.2 Tampering

Tampering threats involve an attacker modifying trusted code, configuration, artifacts, policies, or data without authorization.

For this platform, the primary tampering risks are concentrated around the software supply chain, Kubernetes configuration, security policies, container images, and secrets-management configuration.

#### T-01: Source Code or Manifest Tampering

**Threat:**
An attacker with unauthorized repository access may modify application source code, Kubernetes manifests, security configuration, or infrastructure definitions.

**Attack Scenario:**

```text
Attacker
   |
   v
GitHub Repository
   |
   +----> Application Source
   |
   +----> Kubernetes Manifests
   |
   +----> Security Policies
   |
   v
CI/CD Pipeline
```

Unauthorized changes could introduce vulnerable code, weaken security controls, expose secrets, or modify workload configuration.

**Affected Assets:**

* A-07 Kubernetes manifests
* A-08 Kyverno security policies
* A-10 Source code repository
* A-05 Backend container image
* A-06 Frontend container image

**Affected Boundary:**

* TB-01 Developer → GitHub
* TB-02 GitHub → GitHub Actions

**Risk:** High

**Implemented Controls:**

* Gitleaks secret scanning
* Checkov configuration analysis
* GitHub Actions security pipeline
* Repository version control
* Kubernetes admission policies through Kyverno
* Trivy scanning of Kubernetes resources and container images

**Validation Evidence:**

The repository was scanned with Gitleaks across its commit history with no detected secrets.

Checkov validation completed with:

```text
Kubernetes:       263 passed, 0 failed, 11 skipped
Dockerfile:       113 passed, 0 failed, 0 skipped
GitHub Actions:   204 passed, 0 failed, 0 skipped
```

**Residual Risk:**

A compromised legitimate repository contributor may still introduce intentional malicious changes. Code review, branch protection, signed commits, and enterprise repository governance are outside the primary scope of this project.

---

#### T-02: CI/CD Workflow Tampering

**Threat:**
An attacker may modify the GitHub Actions workflow to bypass or weaken security checks.

**Attack Scenario:**

```text
Modified Workflow
       |
       v
GitHub Actions
       |
       +----> Security checks weakened
       |
       +----> Malicious build behavior
       |
       v
Container Artifact
```

A compromised workflow could potentially disable scanners, alter build inputs, or introduce malicious steps.

**Affected Assets:**

* A-09 CI/CD workflow
* A-05 Backend image
* A-06 Frontend image

**Risk:** High

**Implemented Controls:**

* GitHub Actions security workflow
* `permissions: contents: read`
* Checkov scanning of GitHub Actions configuration
* Version-controlled workflow configuration

**Validation Evidence:**

Checkov completed GitHub Actions analysis with:

```text
204 passed
0 failed
0 skipped
```

**Residual Risk:**

The workflow itself remains a high-value supply-chain target. A sufficiently privileged repository compromise could modify both application code and CI/CD configuration.

---

#### T-03: Container Image Tampering

**Threat:**
An attacker may attempt to replace, modify, or introduce malicious content into a trusted container image.

**Attack Scenario:**

```text
Source / Build Process
        |
        v
Container Image
        |
   [Tampering]
        |
        v
Kubernetes Workload
```

A compromised image could contain malicious code, vulnerable dependencies, backdoors, or altered startup behavior.

**Affected Assets:**

* A-05 Backend container image
* A-06 Frontend container image

**Risk:** High

**Implemented Controls:**

* Trivy image scanning
* SBOM generation
* Digest-pinned deployment images
* Checkov Dockerfile analysis
* CI/CD security pipeline
* Gitleaks scanning

**Validation Evidence:**

The deployed backend image is referenced using an immutable image digest rather than only a mutable tag.

Container images were also scanned using Trivy, and CycloneDX SBOM artifacts were generated for the project images.

**Residual Risk:**

A malicious artifact produced through a compromised trusted build process may still be treated as a legitimate artifact. Stronger production supply-chain controls such as image signing, provenance verification, and admission-time signature verification are outside the current project scope.

---

#### T-04: Kubernetes Configuration Tampering

**Threat:**
An attacker with sufficient Kubernetes authorization may modify deployments, services, NetworkPolicies, or security contexts to weaken the platform.

**Attack Scenario:**

```text
Unauthorized Kubernetes Access
          |
          v
Kubernetes API
          |
          +----> Deployment modification
          |
          +----> NetworkPolicy modification
          |
          +----> SecurityContext modification
          |
          +----> Service modification
```

An attacker could attempt to remove security controls, expose workloads, or deploy a privileged workload.

**Affected Assets:**

* A-07 Kubernetes manifests
* A-08 Kyverno policies
* A-12 Kubernetes control plane

**Risk:** High

**Implemented Controls:**

* Kyverno admission policies
* Kubernetes security contexts
* NetworkPolicies
* Dedicated ServiceAccounts
* Checkov Kubernetes analysis
* Trivy Kubernetes scanning

**Validation Evidence:**

Checkov Kubernetes analysis completed with:

```text
263 passed
0 failed
11 skipped
```

Trivy Kubernetes scanning evaluated:

```text
324 / 324 resources
```

The project also validated Kyverno policies for:

* Non-root execution
* Privilege-escalation restrictions
* RuntimeDefault seccomp
* Resource requests and limits

**Residual Risk:**

An attacker possessing sufficiently privileged Kubernetes credentials could potentially modify resources in ways not prevented by the project's current admission policies. Production RBAC hardening and control-plane security are outside the project scope.

---

#### T-05: Kyverno Policy Tampering

**Threat:**
An attacker may attempt to modify or disable admission policies to bypass workload security requirements.

**Attack Scenario:**

```text
Attacker
   |
   v
Kubernetes Policy Configuration
   |
   X
Kyverno Enforcement
   |
   v
Insecure Workload
```

If admission controls were successfully weakened, insecure workloads could potentially be introduced into the cluster.

**Affected Assets:**

* A-08 Kyverno security policies
* A-12 Kubernetes control plane

**Risk:** High

**Implemented Controls:**

* Kyverno admission policies
* Checkov Kubernetes configuration scanning
* Version-controlled policy definitions
* Kubernetes authorization boundaries
* Security policy validation

**Validation Evidence:**

The project validated Kyverno policies for:

* `runAsNonRoot`
* `allowPrivilegeEscalation`
* `seccompProfile: RuntimeDefault`
* Resource requests and limits

The MySQL workload has a documented `runAsNonRoot` exception because of local image/runtime compatibility. The remaining security controls continue to be evaluated.

**Residual Risk:**

An attacker with sufficient authorization to modify Kyverno policy resources could weaken admission controls. Protecting policy-management privileges is therefore a critical production requirement.

---

#### T-06: Vault Authorization Configuration Tampering

**Threat:**
An attacker may attempt to modify the Vault role or policy to grant unauthorized access to secrets.

**Attack Scenario:**

```text
Unauthorized Vault Administration
          |
          v
Vault Role / Policy
          |
          X
Least-Privilege Boundary
          |
          v
Sensitive Secrets
```

A successful modification could change which identities can authenticate or what secrets they can read.

**Affected Assets:**

* A-01 Vault-managed secrets
* A-04 Vault authorization configuration

**Risk:** High

**Implemented Controls:**

* Kubernetes authentication
* ServiceAccount-specific role binding
* Namespace-specific role binding
* Least-privilege Vault policy
* Explicit Kubernetes audience
* One-hour token TTL
* Backend-to-Vault NetworkPolicy

**Validation Evidence:**

The `devsecops-backend` Vault role was verified to bind only:

```text
ServiceAccount: backend-sa
Namespace: devsecops
```

The policy was verified to provide only:

```text
read
```

access to:

```text
secret/data/devsecops/backend
```

**Residual Risk:**

The local Vault deployment uses file storage and does not provide high availability. Vault administrative compromise or unauthorized policy modification remains a high-impact scenario.

---

### 5.2.1 Tampering Risk Summary

| ID   | Tampering Threat                   | Primary Asset              | Risk | Main Controls                              | Residual Risk                                            |
| ---- | ---------------------------------- | -------------------------- | ---- | ------------------------------------------ | -------------------------------------------------------- |
| T-01 | Source/manifests tampering         | Source / K8s configuration | High | Gitleaks, Checkov, CI pipeline, Kyverno    | Legitimate contributor compromise                        |
| T-02 | CI/CD workflow tampering           | CI/CD / images             | High | Restricted permissions, Checkov            | Workflow remains a supply-chain target                   |
| T-03 | Container image tampering          | Container images           | High | Trivy, SBOM, digest pinning                | Trusted build compromise may produce malicious artifacts |
| T-04 | Kubernetes configuration tampering | Cluster configuration      | High | Kyverno, NetworkPolicies, Checkov, Trivy   | Privileged Kubernetes identity remains high impact       |
| T-05 | Kyverno policy tampering           | Security policies          | High | Kyverno, Checkov, authorization boundaries | Policy-admin compromise                                  |
| T-06 | Vault authorization tampering      | Vault secrets / policies   | High | K8s Auth, least privilege, NetworkPolicy   | Vault administrative compromise                          |

### 5.2.2 Tampering Conclusion

The platform applies multiple integrity controls across the software supply chain and Kubernetes runtime.

Source and configuration changes are subjected to automated security analysis through Gitleaks and Checkov. Container artifacts are evaluated with Trivy and represented through SBOMs, while deployed images use immutable digests. Kubernetes workloads are further protected by Kyverno admission policies and NetworkPolicies.

The principal residual risk is compromise of a trusted administrative or repository identity. If an attacker obtains sufficient privileges to modify CI/CD workflows, Kubernetes resources, security policies, or Vault authorization, the attacker may be able to tamper with trusted security boundaries.

Production environments should therefore supplement these controls with stronger repository governance, protected branches, signed artifacts, provenance verification, strict Kubernetes RBAC, and protected security-policy administration.


### 5.3 Repudiation

Repudiation threats involve an attacker performing an action and subsequently denying that the action occurred, or manipulating security evidence so that the activity cannot be reliably attributed.

For this platform, repudiation is primarily evaluated against Kubernetes activity, CI/CD activity, application/security events, and Wazuh security telemetry.

The project provides monitoring and evidence collection capabilities, but the local lab environment is not treated as a fully tamper-proof enterprise audit platform.

#### R-01: Kubernetes Activity Without Sufficient Attribution

**Threat:**

An attacker or compromised identity may perform unauthorized Kubernetes actions and attempt to deny responsibility or obscure the activity.

**Attack Scenario:**

```text
Attacker / Compromised Identity
             |
             v
       Kubernetes API
             |
             v
      Cluster Changes
             |
             X
       Audit Evidence
```

If appropriate audit information is unavailable, it may be difficult to determine which identity performed a sensitive action.

**Affected Assets:**

* A-12 Kubernetes control plane
* A-07 Kubernetes configuration
* A-08 Kyverno security policies

**Risk:** High

**Implemented Controls:**

* Dedicated Kubernetes ServiceAccounts
* Kubernetes authorization boundaries
* Kyverno admission policies
* Prometheus monitoring
* Grafana alerting
* Wazuh security monitoring

**Residual Risk:**

The project does not implement a production-grade centralized Kubernetes audit-log retention and tamper-evident archival architecture.

Production environments should enable and centrally protect Kubernetes audit logs with appropriate retention and access controls.

---

#### R-02: CI/CD Activity Attribution

**Threat:**

An attacker using a compromised repository or developer identity may modify source code, manifests, or workflows and subsequently deny having made the changes.

**Attack Scenario:**

```text
Compromised Repository Identity
             |
             v
       GitHub Repository
             |
             v
      Source / Workflow Change
             |
             v
        CI/CD Execution
```

**Affected Assets:**

* A-09 CI/CD workflow
* A-10 Source code repository
* A-05 Backend image
* A-06 Frontend image

**Affected Boundaries:**

* TB-01 Developer → GitHub
* TB-02 GitHub → GitHub Actions
* TB-03 CI/CD → Container Artifact

**Risk:** High

**Implemented Controls:**

* Git history
* GitHub Actions workflow
* Security scanning through Gitleaks and Checkov
* Trivy scanning
* SBOM generation

**Validation Evidence:**

The repository maintains version-controlled history, allowing changes to source code and configuration to be reviewed against previous repository states.

Security scanning is integrated into the GitHub Actions workflow.

**Residual Risk:**

Repository history alone does not provide complete non-repudiation. A sufficiently privileged repository administrator may be able to alter repository history, modify workflows, or otherwise affect the evidence available within the repository.

Enterprise environments should supplement repository history with protected branches, centralized audit logs, identity monitoring, and appropriate administrative controls.

---

#### R-03: Security Event Log Manipulation

**Threat:**

An attacker may attempt to modify, delete, or prevent collection of security events in order to hide malicious activity.

**Attack Scenario:**

```text
Windows Endpoint
      |
      v
 Wazuh Agent
      |
      v
 Wazuh Manager
      |
      v
 Wazuh Indexer
      |
      X
Tampering / Deletion
```

If security telemetry is successfully removed or manipulated, incident investigation and detection capability may be reduced.

**Affected Assets:**

* A-11 Wazuh security telemetry
* Security investigation evidence

**Risk:** High

**Implemented Controls:**

* Wazuh Agent
* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard
* Wazuh archive collection
* Filebeat archive forwarding
* Centralized security-event indexing

**Validation Evidence:**

A Windows security event was generated and successfully observed through the Wazuh archive pipeline and indexed for investigation.

The project also validated the Wazuh archive flow from endpoint event generation through collection and indexing.

**Residual Risk:**

The local Wazuh environment is not considered an immutable or independently protected evidence store. An attacker who gains sufficient privileges over the endpoint or Wazuh infrastructure may be able to interfere with telemetry.

Production deployments should protect security telemetry using centralized access controls, restricted administrative privileges, appropriate retention, and tamper-evident or write-protected storage where required.

---

#### R-04: Application Activity Without Adequate Audit Evidence

**Threat:**

An application user or attacker may perform sensitive actions without sufficient application-level audit information to reconstruct the activity.

**Affected Assets:**

* Application data
* Backend API
* MySQL application data

**Risk:** Medium

**Project Boundary:**

Detailed application business-event auditing is outside the primary scope of this DevSecOps infrastructure security project.

Prometheus and Grafana primarily provide operational and availability monitoring rather than complete application audit trails.

Wazuh provides security telemetry, but it should not be interpreted as automatically providing complete application-level accountability for every business action.

**Residual Risk:**

A production application should define and centrally collect security-relevant events such as authentication attempts, authorization failures, sensitive data modifications, administrative actions, and other security-critical operations.

Sensitive values such as passwords, access tokens, and database credentials should not be written to logs. OWASP specifically recommends protecting log data and avoiding direct logging of credentials and other sensitive secrets.

---

### 5.3.1 Repudiation Risk Summary

| ID   | Repudiation Threat               | Primary Asset         | Risk   | Main Controls                         | Residual Risk                                   |
| ---- | -------------------------------- | --------------------- | ------ | ------------------------------------- | ----------------------------------------------- |
| R-01 | Kubernetes activity attribution  | Cluster configuration | High   | ServiceAccounts, Kyverno, monitoring  | No production-grade centralized audit archive   |
| R-02 | CI/CD activity attribution       | Source / CI/CD        | High   | Git history, CI security pipeline     | Privileged repository users can affect evidence |
| R-03 | Security telemetry manipulation  | Wazuh evidence        | High   | Wazuh Agent/Manager/Indexer, archives | Local telemetry is not immutable                |
| R-04 | Application activity attribution | Application data      | Medium | Wazuh / monitoring                    | Full application audit logging is outside scope |

### 5.3.2 Repudiation Conclusion

The platform provides multiple sources of security and operational evidence through Git history, CI/CD security checks, Prometheus/Grafana monitoring, and Wazuh telemetry.

These controls improve visibility and support investigation, but they are not represented as providing absolute non-repudiation.

The primary residual risk is that the local security engineering environment does not provide an independently administered, immutable, tamper-evident audit repository.

Production environments should strengthen accountability through centralized audit logging, protected administrative identities, controlled log access, appropriate retention, and tamper-evident or write-protected storage for high-value security evidence.


### 5.4 Information Disclosure

Information disclosure occurs when sensitive information is exposed to an unauthorized user, workload, process, or external party.

For this platform, the primary confidentiality concerns are application secrets, database data, Kubernetes resources, source-code and CI/CD information, and security telemetry.

#### I-01: Vault Secret Disclosure

**Threat:**

An attacker who compromises a workload or obtains unauthorized access to Vault may attempt to retrieve application secrets such as database credentials.

**Attack Scenario:**

```text
Compromised Backend Pod
        |
        v
   backend-sa
        |
        v
Vault Kubernetes Auth
        |
        v
devsecops-backend Role
        |
        v
devsecops-backend Policy
        |
        v
Database Credentials
```

**Affected Assets:**

* A-01 Vault-managed application secrets
* A-03 Kubernetes workload identity
* A-04 Vault authorization configuration

**Risk:** High

**Implemented Controls:**

* Dedicated `backend-sa` ServiceAccount
* Vault Kubernetes authentication
* Vault role restricted to `backend-sa`
* Vault role restricted to the `devsecops` namespace
* Explicit Kubernetes authentication audience
* One-hour Vault token TTL
* Dedicated `devsecops-backend` policy
* Read-only access to the exact application secret path
* NetworkPolicy restricting backend communication with Vault
* Vault Agent Injector
* Container running as non-root
* All Linux capabilities dropped
* Privilege escalation disabled
* Read-only root filesystem

**Validation Evidence:**

The Vault Kubernetes authentication role was verified to be bound to `backend-sa` in the `devsecops` namespace with a one-hour token TTL and the expected Kubernetes audience.

The `devsecops-backend` Vault policy was verified to provide read access only to the required application secret path.

Vault Agent injection was validated on the backend workload.

**Residual Risk:**

A sufficiently compromised backend workload may still retrieve secrets that its legitimate identity is authorized to access.

The local Vault deployment also uses file storage without HA, which is appropriate for this security engineering lab but is not equivalent to a production-grade highly available secrets-management architecture.

Production environments should additionally implement strong administrative access controls, secret rotation, protected Vault storage, HA, and centralized audit monitoring.

---

#### I-02: Database Data Disclosure

**Threat:**

An attacker who compromises the backend workload or gains unauthorized network access may attempt to retrieve sensitive application data from MySQL.

**Attack Scenario:**

```text
External Attacker
      |
      v
Application Vulnerability
      |
      v
Backend Workload
      |
      v
MySQL :3306
      |
      v
Application Data
```

**Affected Assets:**

* A-02 MySQL application data
* Backend API
* Database credentials

**Risk:** High

**Implemented Controls:**

* Backend-to-MySQL NetworkPolicy
* Default-deny ingress for the `devsecops` namespace
* Backend database credentials managed through Vault
* Dedicated backend ServiceAccount
* Backend container hardening
* Network segmentation between frontend, backend, and database
* MySQL workload security controls where compatible with the local runtime

**Validation Evidence:**

The backend-to-MySQL communication path was explicitly permitted through NetworkPolicy while unrelated ingress paths were denied.

The application successfully communicated with MySQL through the intended backend path.

**Residual Risk:**

The backend is intentionally authorized to access the application database. Therefore, compromise of the backend can still provide an attacker with the level of database access granted to the application.

The current local environment also does not implement the full database confidentiality architecture expected in a production environment, such as comprehensive database encryption and enterprise database access governance.

---

#### I-03: Container, Configuration, or Secret-File Disclosure

**Threat:**

An attacker who gains access to a container may attempt to read configuration files, mounted secrets, temporary files, or other filesystem content.

**Affected Assets:**

* A-01 Vault-managed secrets
* Application configuration
* Backend container filesystem

**Risk:** High

**Implemented Controls:**

* Backend container runs as non-root
* Fixed non-root UID/GID
* `allowPrivilegeEscalation: false`
* All capabilities dropped
* `readOnlyRootFilesystem: true`
* Isolated writable `/tmp`
* Vault Agent secret injection
* Secrets are not embedded directly into the container image
* Gitleaks scanning of repository history

**Validation Evidence:**

The backend deployment was verified with non-root execution, privilege escalation disabled, all capabilities dropped, read-only root filesystem, and isolated temporary storage.

Gitleaks validation reported no detected secrets in the scanned repository history.

**Residual Risk:**

A compromised backend process may still access files and secrets available to its own authorized security context.

Application code should therefore minimize secret exposure and avoid logging credentials, access tokens, or database connection strings. OWASP specifically recommends that these sensitive values be removed, masked, sanitized, or otherwise protected from application logs.

---

#### I-04: Kubernetes API or Resource Information Disclosure

**Threat:**

An attacker may attempt to obtain Kubernetes resource information, workload metadata, ServiceAccount information, or other cluster data through a compromised workload or unauthorized Kubernetes access.

**Affected Assets:**

* A-03 Kubernetes workload identity
* A-07 Kubernetes manifests
* A-12 Kubernetes control plane

**Risk:** High

**Implemented Controls:**

* Dedicated ServiceAccounts
* Explicit backend ServiceAccount assignment
* NetworkPolicies
* Kyverno security policies
* Non-root workload execution
* Seccomp `RuntimeDefault`
* Privilege escalation disabled
* Capability dropping
* Kubernetes security scanning with Trivy
* Checkov Kubernetes security scanning

**Validation Evidence:**

Checkov Kubernetes scanning completed with:

* 263 passed
* 0 failed
* 11 skipped

Trivy Kubernetes scanning evaluated 324/324 cluster resources.

**Residual Risk:**

The local Kind cluster is a security engineering environment rather than a production Kubernetes platform.

Production deployments should additionally implement strict Kubernetes RBAC, protected control-plane access, hardened API-server configuration, centralized audit logging, and strong administrative identity controls.

---

#### I-05: CI/CD, Source-Code, and Security-Artifact Disclosure

**Threat:**

An attacker who gains unauthorized access to the source repository or CI/CD environment may obtain source code, manifests, security configuration, vulnerability reports, SBOM information, or other information useful for further attacks.

**Affected Assets:**

* A-07 Kubernetes manifests
* A-08 Kyverno policies
* A-09 CI/CD workflow
* A-10 source repository
* A-15 Trivy results
* A-16 SBOM
* A-17 Gitleaks results
* A-18 Checkov results

**Risk:** Medium

**Implemented Controls:**

* GitHub repository access controls
* GitHub Actions workflow with `contents: read`
* Gitleaks secret scanning
* Checkov IaC and workflow scanning
* Trivy vulnerability scanning
* CycloneDX SBOM generation
* Digest-pinned backend image
* Git-based change history

**Validation Evidence:**

The repository was scanned with Gitleaks without detected secrets.

Checkov validation produced:

* Kubernetes: 263 passed, 0 failed, 11 skipped
* Dockerfile: 113 passed, 0 failed, 0 skipped
* GitHub Actions: 204 passed, 0 failed, 0 skipped

**Residual Risk:**

Security reports and SBOMs may contain useful information about software versions, vulnerabilities, configuration, or architecture.

Repository compromise or unauthorized access to CI/CD administration can therefore expose information useful for targeted attacks.

Production environments should apply strong repository access controls, protected branches, MFA, restricted CI/CD administration, appropriate artifact access controls, and controlled distribution of security reports.

---

#### I-06: Security Telemetry and Log Disclosure

**Threat:**

An attacker with access to monitoring or logging infrastructure may obtain sensitive operational information from security telemetry, application logs, or monitoring data.

**Affected Assets:**

* A-11 Wazuh security telemetry
* A-13 Prometheus data
* A-14 Grafana alerts
* Security investigation evidence

**Risk:** Medium

**Implemented Controls:**

* Wazuh centralized security telemetry
* Wazuh archive collection
* Filebeat archive forwarding
* Wazuh Indexer
* Grafana monitoring
* Prometheus metrics
* Security-event investigation through indexed telemetry

**Validation Evidence:**

A Windows test event was successfully generated and observed through the Wazuh archive pipeline and indexed for investigation.

Prometheus and Grafana monitoring were also validated through Kubernetes workload and availability alerts.

**Residual Risk:**

Monitoring and logging systems can themselves become sources of information disclosure if access is not properly restricted.

Logs may contain technical information about hosts, workloads, routes, errors, or internal infrastructure. Sensitive values should not be logged unnecessarily.

OWASP recommends protecting collected logs against unauthorized access and specifically avoiding direct logging of passwords, access tokens, database connection strings, and other sensitive secrets.

Production deployments should implement role-based access to monitoring platforms, centralized log-access controls, appropriate retention, secure transport, and protected storage.

---

### 5.4.1 Information Disclosure Risk Summary

| ID   | Information Disclosure Threat      | Primary Asset                 | Risk   | Main Controls                                   | Residual Risk                                                                |
| ---- | ---------------------------------- | ----------------------------- | ------ | ----------------------------------------------- | ---------------------------------------------------------------------------- |
| I-01 | Vault secret disclosure            | Vault secrets                 | High   | Vault auth, least privilege, TTL, NetworkPolicy | Compromised authorized workload may retrieve permitted secrets               |
| I-02 | Database data disclosure           | MySQL data                    | High   | NetworkPolicy, Vault, workload isolation        | Backend compromise can expose authorized database access                     |
| I-03 | Container/configuration disclosure | Container/secrets             | High   | Non-root, read-only rootfs, capability dropping | Compromised process may access its authorized filesystem/secrets             |
| I-04 | Kubernetes resource disclosure     | Kubernetes control plane      | High   | ServiceAccounts, Kyverno, Checkov, Trivy        | Local Kind environment lacks full production control-plane hardening         |
| I-05 | CI/CD/source disclosure            | Repository/security artifacts | Medium | GitHub controls, Gitleaks, Checkov, Trivy       | Repository or CI/CD compromise can expose security-relevant information      |
| I-06 | Telemetry disclosure               | Wazuh/Prometheus/Grafana      | Medium | Centralized monitoring and logging              | Local monitoring infrastructure requires stronger production access controls |

### 5.4.2 Information Disclosure Conclusion

The platform applies multiple confidentiality controls across secrets, workloads, network communication, source code, CI/CD, and security telemetry.

The strongest confidentiality control is the combination of Vault-based secret management, workload identity, least-privilege secret authorization, NetworkPolicies, and hardened backend containers.

However, security controls do not eliminate the impact of a fully compromised authorized workload. A compromised backend may still access the resources required for its legitimate function, including authorized database data and application secrets.

The main residual risks are therefore associated with the local lab environment, authorized workload compromise, monitoring-data access, and the absence of production-grade centralized access governance.

Production environments should strengthen confidentiality through strict RBAC, protected administrative identities, secret rotation, encrypted communications and storage where appropriate, controlled security-artifact access, and centralized protection of monitoring and audit data.


### 5.5 Denial of Service

Denial of Service (DoS) occurs when an attacker causes a system, workload, security service, or supporting infrastructure to become unavailable or significantly degraded.

For this platform, DoS analysis focuses on application workloads, Kubernetes resources, security infrastructure, and resource exhaustion within the local security engineering environment.

#### D-01: Application Request Flooding

**Threat:**

An attacker may send excessive requests to the exposed application or backend API, consuming CPU, memory, network connections, or application resources and reducing service availability.

**Attack Scenario:**

```text
External Attacker
      |
      v
Ingress-NGINX
      |
      v
React Frontend
      |
      v
Spring Boot Backend
      |
      v
Resource Exhaustion
```

**Affected Assets:**

* Backend API
* Frontend application
* Application availability
* A-02 MySQL application data availability

**Risk:** High

**Implemented Controls:**

* Ingress-NGINX
* Kubernetes workload isolation
* NetworkPolicies
* Kubernetes resource requests and limits
* Prometheus monitoring
* Grafana availability alerts
* Backend replica monitoring

**Validation Evidence:**

Checkov Kubernetes scanning validated the configured Kubernetes resources and resource-management controls.

Prometheus and Grafana alerts were configured and tested for backend deployment availability and pod restart conditions.

**Residual Risk:**

The project does not implement a dedicated production-grade API rate-limiting or DDoS mitigation layer.

An attacker capable of generating sufficiently high request volumes may still exhaust application or infrastructure resources.

Production deployments should implement appropriate rate limiting, request quotas, connection limits, upstream protection, and DDoS mitigation according to the application's expected traffic profile.

---

#### D-02: Kubernetes Workload Resource Exhaustion

**Threat:**

A malicious or compromised workload may consume excessive CPU or memory and affect application availability or other workloads running on the Kubernetes node.

**Attack Scenario:**

```text
Compromised Workload
        |
        v
Excessive CPU / Memory
        |
        v
Kubernetes Node Resources
        |
        v
Workload Degradation
```

**Affected Assets:**

* Kubernetes workloads
* Backend availability
* Frontend availability
* Kind cluster resources

**Risk:** High

**Implemented Controls:**

* Kubernetes resource requests and limits
* Checkov Kubernetes scanning
* Prometheus monitoring
* Grafana alerts
* Kubernetes workload isolation
* Container security hardening
* Kyverno resource-policy validation

**Validation Evidence:**

The Kyverno policy set includes validation for resource requests and limits.

Checkov Kubernetes scanning completed with:

* 263 passed
* 0 failed
* 11 skipped

Grafana availability and restart alerts were also tested against the backend deployment.

**Residual Risk:**

The local Kind cluster runs on shared host resources. Resource exhaustion at the host or Docker layer can therefore affect multiple workloads simultaneously.

Production Kubernetes environments should additionally use node-level capacity planning, quotas, LimitRanges, autoscaling where appropriate, workload prioritization, and dedicated resource governance.

---

#### D-03: Database Resource Exhaustion

**Threat:**

An attacker may abuse backend functionality or database operations to consume excessive MySQL connections, CPU, memory, storage, or query-processing capacity.

**Affected Assets:**

* A-02 MySQL application data availability
* Backend API
* MySQL workload

**Risk:** High

**Implemented Controls:**

* Backend-to-MySQL NetworkPolicy
* Backend workload isolation
* Kubernetes resource controls
* Vault-managed database credentials
* Prometheus/Grafana workload monitoring

**Validation Evidence:**

NetworkPolicy validation confirmed that database communication is restricted to the intended backend workload.

Kubernetes resource-policy validation was performed through Kyverno and Checkov.

**Residual Risk:**

The project does not implement comprehensive database connection pooling limits, query-cost controls, database-specific workload governance, or production database resource isolation.

A compromised backend may still consume database resources through its legitimate database access.

Production deployments should implement database connection limits, query timeouts, workload-specific quotas, appropriate indexing, and database monitoring.

---

#### D-04: Security Infrastructure Availability Loss

**Threat:**

An attacker may disrupt Vault, Wazuh, Prometheus, Grafana, or other security infrastructure, reducing the platform's ability to provide secrets, monitoring, detection, or investigation capabilities.

**Attack Scenario:**

```text
Attacker
   |
   +----> Vault
   |
   +----> Prometheus / Grafana
   |
   +----> Wazuh
   |
   v
Security-Control Degradation
```

**Affected Assets:**

* A-01 Vault-managed secrets
* A-11 Wazuh telemetry
* A-13 Prometheus data
* A-14 Grafana alerts
* Security monitoring capability

**Risk:** High

**Implemented Controls:**

* Kubernetes workload isolation
* NetworkPolicies
* Kyverno security policies
* Prometheus monitoring
* Grafana alerting
* Wazuh centralized telemetry
* Vault authentication and authorization controls

**Validation Evidence:**

The security services were deployed and validated as separate components.

Wazuh archive collection and indexing were successfully tested.

Prometheus and Grafana monitoring were validated using Kubernetes availability and restart alerts.

Vault Kubernetes authentication and secret injection were validated for the backend workload.

**Residual Risk:**

The local environment does not provide high availability for all security infrastructure components.

Vault specifically uses file storage with HA disabled in this lab environment.

Loss of a security service may therefore affect the corresponding security capability until the service is restored.

Production environments should use appropriate HA architectures, backup and recovery procedures, capacity planning, and independent monitoring for critical security infrastructure.

---

#### D-05: Kubernetes Control-Plane or Host Resource Exhaustion

**Threat:**

Excessive workload or infrastructure resource consumption may exhaust the local Kubernetes control-plane or Docker host resources, causing cluster-wide degradation.

**Affected Assets:**

* A-12 Kubernetes control plane
* Kind cluster
* Application workloads
* Security infrastructure

**Risk:** High

**Implemented Controls:**

* Kubernetes resource requests and limits
* Checkov validation
* Kyverno resource-policy validation
* Prometheus monitoring
* Grafana alerting
* Container resource isolation

**Validation Evidence:**

Kubernetes resources were scanned with Trivy and Checkov.

Trivy evaluated 324/324 Kubernetes resources.

Resource requests and limits were validated through Kyverno policy controls and Checkov.

**Residual Risk:**

Kind is a local Kubernetes environment running on the developer workstation. Host-level resource exhaustion remains outside the application's Kubernetes security boundary.

Production environments should use dedicated node capacity, cluster-level resource quotas, autoscaling where appropriate, node monitoring, and capacity planning.

---

### 5.5.1 Denial of Service Risk Summary

| ID   | Denial of Service Threat           | Primary Asset          | Risk | Main Controls                                 | Residual Risk                                       |
| ---- | ---------------------------------- | ---------------------- | ---- | --------------------------------------------- | --------------------------------------------------- |
| D-01 | Application request flooding       | Backend/API            | High | Ingress, resources, monitoring, alerts        | No dedicated production rate limiting or DDoS layer |
| D-02 | Kubernetes workload exhaustion     | Kubernetes workloads   | High | Resource limits, Kyverno, Checkov, monitoring | Shared local Kind/host resources                    |
| D-03 | Database resource exhaustion       | MySQL                  | High | NetworkPolicy, resource controls, Vault       | Limited database-specific resource governance       |
| D-04 | Security infrastructure disruption | Vault/Wazuh/Monitoring | High | Isolation, NetworkPolicies, monitoring        | Local services lack full HA                         |
| D-05 | Control-plane/host exhaustion      | Kubernetes cluster     | High | Resource controls, Trivy, Checkov, monitoring | Kind depends on developer workstation resources     |

### 5.5.2 Denial of Service Conclusion

The platform implements several availability-oriented controls through Kubernetes resource governance, NetworkPolicies, Kyverno, Prometheus, Grafana, and workload monitoring.

These controls reduce the likelihood and impact of resource exhaustion but do not constitute a complete production DDoS protection architecture.

The most significant residual risks are application request flooding, database resource exhaustion, shared local host resources, and the lack of full high availability for security infrastructure.

Production environments should complement the implemented controls with rate limiting, quotas, connection limits, capacity planning, autoscaling where appropriate, high-availability architectures, backup and recovery procedures, and dedicated DDoS or upstream traffic-protection mechanisms where required.


### 5.6 Elevation of Privilege

Elevation of Privilege occurs when an attacker gains permissions beyond those intended for their identity, workload, or security context.

For this platform, the primary privilege-escalation concerns involve container privileges, Kubernetes identities, Vault authorization, CI/CD administration, and security-policy administration.

#### E-01: Container Privilege Escalation

**Threat:**

An attacker who compromises an application process may attempt to gain additional Linux or container privileges and escape the intended security context.

**Attack Scenario:**

```text
Compromised Application Process
          |
          v
Privilege Escalation Attempt
          |
          +----> Root privileges
          |
          +----> Additional Linux capabilities
          |
          +----> Restricted filesystem access
          |
          v
Broader Container / Host Impact
```

**Affected Assets:**

* Backend workload
* Kubernetes node
* A-12 Kubernetes control plane
* Application data

**Risk:** High

**Implemented Controls:**

* `runAsNonRoot: true`
* Explicit non-root UID/GID
* `allowPrivilegeEscalation: false`
* All Linux capabilities dropped
* `seccompProfile: RuntimeDefault`
* `readOnlyRootFilesystem: true`
* Isolated writable `/tmp`
* Kyverno security-policy validation
* Kubernetes Pod Security warning configuration
* Container image hardening

**Validation Evidence:**

The backend deployment was verified to run as UID/GID `10001`, with privilege escalation disabled, all capabilities dropped, a read-only root filesystem, and `RuntimeDefault` seccomp.

Kyverno policies validate the relevant workload security controls.

Kubernetes Restricted Pod Security guidance similarly requires non-root execution, disabled privilege escalation, appropriate seccomp profiles, and dropped capabilities for restricted workloads.

**Residual Risk:**

The project uses the runtime-provided `RuntimeDefault` seccomp profile rather than a workload-specific custom profile.

The local Kind environment also does not provide the complete host-level isolation and hardening expected from a production Kubernetes platform.

Production workloads handling highly sensitive data may require additional host hardening, stronger workload isolation, custom seccomp/AppArmor profiles where justified, and dedicated node security controls.

---

#### E-02: Kubernetes ServiceAccount Privilege Escalation

**Threat:**

A compromised workload may attempt to abuse its Kubernetes ServiceAccount or Kubernetes API access to obtain permissions beyond those required by the application.

**Attack Scenario:**

```text
Compromised Backend Pod
        |
        v
backend-sa
        |
        v
Kubernetes API
        |
        v
Unauthorized Resource Access
        |
        v
Privilege Escalation
```

**Affected Assets:**

* A-03 Kubernetes workload identity
* A-07 Kubernetes manifests
* A-12 Kubernetes control plane

**Risk:** High

**Implemented Controls:**

* Dedicated `backend-sa`
* Dedicated `frontend-sa`
* Dedicated `mysql-sa`
* Explicit ServiceAccount assignment
* Workload isolation through NetworkPolicies
* Kyverno policy enforcement
* Non-root container execution
* Container capability restrictions
* Kubernetes security scanning through Checkov and Trivy

**Validation Evidence:**

The backend deployment explicitly uses `backend-sa`.

The ServiceAccount configuration was reviewed during Vault Kubernetes-authentication validation.

Checkov Kubernetes scanning completed with:

* 263 passed
* 0 failed
* 11 skipped

Trivy Kubernetes scanning evaluated 324/324 Kubernetes resources.

**Residual Risk:**

The project does not implement a comprehensive production Kubernetes RBAC model because detailed cluster-wide identity governance is outside the project's scope.

A compromised workload identity could still perform any Kubernetes actions granted to that identity.

Production deployments should implement strict RBAC with narrowly scoped Roles and RoleBindings, disable unnecessary ServiceAccount permissions, and restrict access to sensitive namespaces and resources.

Kubernetes guidance recommends strict RBAC and least-privilege controls for protecting privileged cluster areas.

---

#### E-03: Vault Authorization Privilege Escalation

**Threat:**

An attacker may attempt to abuse the backend workload identity or Vault configuration to obtain secrets beyond the application's intended authorization boundary.

**Attack Scenario:**

```text
Compromised Backend
       |
       v
backend-sa
       |
       v
Vault Kubernetes Auth
       |
       v
devsecops-backend Role
       |
       v
devsecops-backend Policy
       |
       X
Unauthorized Secret Path
```

**Affected Assets:**

* A-01 Vault-managed secrets
* A-03 Kubernetes workload identity
* A-04 Vault authorization configuration

**Risk:** High

**Implemented Controls:**

* Vault Kubernetes authentication
* Role bound specifically to `backend-sa`
* Namespace restriction to `devsecops`
* Explicit Kubernetes authentication audience
* One-hour token TTL
* Dedicated Vault policy
* Read-only secret capability
* Exact secret-path authorization
* Backend-to-Vault NetworkPolicy
* Vault Agent Injector

**Validation Evidence:**

The `devsecops-backend` Vault role was verified to include:

* `backend-sa`
* `devsecops` namespace
* Explicit Kubernetes audience
* One-hour token TTL
* `devsecops-backend` policy

The policy was verified to grant read access only to:

`secret/data/devsecops/backend`

Vault Agent secret injection was successfully validated for the backend workload.

**Residual Risk:**

A compromised backend can still access secrets explicitly authorized to its identity.

The project therefore limits the blast radius of a compromised workload rather than attempting to make the workload completely incapable of accessing secrets.

Production deployments should additionally implement privileged Vault administration controls, audit monitoring, secret rotation, protected storage, and separation of administrative and application identities.

---

#### E-04: Kyverno Security-Policy Administration Compromise

**Threat:**

An attacker who gains administrative access to Kyverno policies may weaken or disable security controls and subsequently deploy workloads with excessive privileges.

**Attack Scenario:**

```text
Compromised Administrator
          |
          v
Kyverno Policy Modification
          |
          v
Security Control Bypass
          |
          v
Privileged Workload Deployment
```

**Affected Assets:**

* A-08 Kyverno policies
* Kubernetes workloads
* A-12 Kubernetes control plane
* Security-policy integrity

**Risk:** High

**Implemented Controls:**

* Kyverno security policies
* Policy validation for non-root execution
* Policy validation for privilege escalation
* Policy validation for seccomp
* Policy validation for resource requests and limits
* Git-based policy version history
* Checkov scanning
* Trivy Kubernetes scanning

**Validation Evidence:**

The implemented Kyverno policies were validated against workload security requirements.

Checkov Kubernetes scanning completed with 263 passed checks, 0 failed checks, and 11 skipped checks.

Git history provides change tracking for security-policy configuration.

**Residual Risk:**

An identity with sufficient Kubernetes administrative privileges may still modify or disable security policies.

The local project does not implement a complete production-grade separation of duties for security-policy administration.

Production environments should protect Kyverno administration using strict RBAC, protected change-management workflows, code review, restricted policy administrators, and independent monitoring of security-policy modifications.

---

#### E-05: CI/CD Privilege Escalation

**Threat:**

An attacker who compromises a developer account, repository administrator, workflow, or CI/CD process may attempt to gain greater control over source code, container images, deployment configuration, or security controls.

**Attack Scenario:**

```text
Compromised Developer / Repository Identity
                  |
                  v
           CI/CD Workflow
                  |
        +---------+---------+
        |                   |
        v                   v
 Source / Manifests    Container Artifacts
        |                   |
        +---------+---------+
                  |
                  v
          Kubernetes Deployment
                  |
                  v
        Security-Control Bypass
```

**Affected Assets:**

* A-05 Backend image
* A-06 Frontend image
* A-07 Kubernetes manifests
* A-08 Kyverno policies
* A-09 CI/CD workflow
* A-10 Source repository

**Risk:** High

**Implemented Controls:**

* GitHub Actions security pipeline
* Gitleaks scanning
* Checkov scanning
* Trivy scanning
* SBOM generation
* Digest-pinned backend image
* Git-based change history
* CI workflow permission limited to `contents: read`

**Validation Evidence:**

Checkov GitHub Actions scanning completed with:

* 204 passed
* 0 failed
* 0 skipped

Gitleaks scanning reported no detected repository secrets.

The backend deployment uses a digest-pinned container image.

**Residual Risk:**

A sufficiently privileged repository administrator or compromised CI/CD identity may still modify workflows, source code, manifests, or release processes.

The project does not implement enterprise-grade artifact signing, provenance verification, protected deployment environments, or complete separation of CI/CD administration.

Production environments should use protected branches, mandatory reviews, MFA, restricted workflow permissions, signed artifacts, provenance verification, and tightly controlled deployment identities.

---

#### E-06: Kubernetes Security-Policy or Administrative Privilege Abuse

**Threat:**

A compromised Kubernetes administrator may intentionally or accidentally grant excessive permissions, weaken workload security, or modify critical cluster resources.

**Affected Assets:**

* A-07 Kubernetes manifests
* A-08 Kyverno policies
* A-12 Kubernetes control plane
* Cluster security posture

**Risk:** High

**Implemented Controls:**

* Kyverno policy enforcement
* Kubernetes workload security contexts
* NetworkPolicies
* Dedicated ServiceAccounts
* Checkov Kubernetes scanning
* Trivy Kubernetes scanning
* Git-based configuration history
* Pod Security configuration

**Validation Evidence:**

Kubernetes configuration was evaluated through Checkov and Trivy.

The implemented security policies enforce or validate non-root execution, privilege escalation restrictions, seccomp, and resource requirements.

The backend workload was verified against the intended hardened security context.

**Residual Risk:**

Administrative Kubernetes privileges inherently provide significant control over cluster security.

The project does not attempt to eliminate the trust placed in a fully privileged cluster administrator.

Production environments should separate cluster administration from security-policy administration where practical, enforce strict RBAC, use protected administrative identities, monitor privileged actions, and maintain independent audit records.

---

### 5.6.1 Elevation of Privilege Risk Summary

| ID   | Elevation of Privilege Threat             | Primary Asset         | Risk | Main Controls                                     | Residual Risk                                                 |
| ---- | ----------------------------------------- | --------------------- | ---- | ------------------------------------------------- | ------------------------------------------------------------- |
| E-01 | Container privilege escalation            | Backend workload      | High | Non-root, seccomp, capabilities, read-only rootfs | Local runtime and host hardening remain limited               |
| E-02 | ServiceAccount privilege escalation       | Kubernetes identity   | High | Dedicated SAs, Kyverno, NetworkPolicies           | Full production RBAC is outside scope                         |
| E-03 | Vault authorization escalation            | Vault secrets         | High | K8s auth, exact policy, namespace binding, TTL    | Compromised authorized workload retains permitted access      |
| E-04 | Kyverno policy administration compromise  | Security policies     | High | Kyverno, Checkov, Git history                     | Policy administrators remain highly trusted                   |
| E-05 | CI/CD privilege escalation                | Source / artifacts    | High | Gitleaks, Checkov, Trivy, permissions             | Privileged repository/CI identities remain high-value targets |
| E-06 | Kubernetes administrative privilege abuse | Cluster control plane | High | Kyverno, RBAC boundaries, scanning, monitoring    | Fully privileged administrators retain broad control          |

### 5.6.2 Elevation of Privilege Conclusion

The platform implements multiple layers of defense against privilege escalation across containers, Kubernetes identities, Vault authorization, CI/CD, and security-policy administration.

The strongest workload-level controls are non-root execution, disabled privilege escalation, dropped Linux capabilities, `RuntimeDefault` seccomp, read-only container filesystems, and Kyverno policy validation. These controls align closely with Kubernetes Restricted Pod Security recommendations.

The project also applies least-privilege principles to the backend's Vault identity by restricting the Kubernetes ServiceAccount, namespace, authentication audience, token lifetime, and secret path.

The main residual risk is administrative trust. A sufficiently privileged Kubernetes, Vault, repository, CI/CD, or security-policy administrator can potentially weaken or bypass the controls protecting the platform.

Production environments should therefore strengthen privilege separation through strict Kubernetes RBAC, protected administrative identities, separation of security-policy administration, protected CI/CD workflows, artifact signing and provenance, privileged-action monitoring, and independent audit records.


## 6. Threat → Risk → Control → Validation Mapping

This section maps the principal threats identified during the STRIDE analysis to the security controls implemented by the platform and the evidence used to validate those controls.

The purpose is to demonstrate that identified threats are not only documented, but are connected to concrete security controls and measurable validation activities.

### 6.1 Principal Threat-to-Control Mapping

| Threat ID | Threat / Attack Path                               | Risk   | Implemented Security Controls                                                                                              | Validation Evidence                                                   | Residual Risk                                                           |
| --------- | -------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| S-01      | Kubernetes workload identity spoofing              | High   | Dedicated ServiceAccounts, Vault Kubernetes authentication, namespace binding, explicit audience, short-lived Vault tokens | Vault role verification; backend ServiceAccount verification          | A compromised authorized workload can still use its legitimate identity |
| S-02      | Vault role impersonation                           | High   | Kubernetes auth, `backend-sa` binding, namespace restriction, audience restriction, 1h TTL, least-privilege Vault policy   | Vault role and policy inspection; secret injection validation         | Compromised backend retains access to secrets authorized for its role   |
| T-01      | Source/manifests tampering                         | High   | Git history, Gitleaks, Checkov, GitHub Actions security pipeline                                                           | Gitleaks scan; Checkov scan; repository history                       | Privileged repository identities can still modify trusted source        |
| T-03      | Container image tampering                          | High   | Trivy, SBOM generation, digest-pinned backend image, CI security scanning                                                  | Trivy image scanning; CycloneDX SBOM; digest verification             | Production artifact signing/provenance is outside scope                 |
| T-04      | Kubernetes configuration tampering                 | High   | Checkov, Trivy, Kyverno, Git-based configuration history                                                                   | Checkov Kubernetes results; Trivy Kubernetes scan; Kyverno validation | Privileged Kubernetes administrators can modify trusted configuration   |
| T-05      | Kyverno policy tampering                           | High   | Kyverno policies, Checkov, Git history, policy validation                                                                  | Kyverno policy validation; Checkov; Git history                       | Security-policy administrators remain highly trusted                    |
| R-01      | Kubernetes activity without sufficient attribution | High   | Dedicated ServiceAccounts, Kyverno, Prometheus/Grafana, Wazuh                                                              | Monitoring and security telemetry validation                          | No independent immutable Kubernetes audit archive                       |
| R-03      | Security telemetry manipulation                    | High   | Wazuh Agent/Manager/Indexer, archive collection, Filebeat                                                                  | Windows event successfully observed through Wazuh archive/indexing    | Local telemetry infrastructure is not immutable                         |
| I-01      | Vault secret disclosure                            | High   | Vault least privilege, Kubernetes auth, TTL, NetworkPolicy, Agent Injector                                                 | Vault role/policy verification and secret injection                   | Authorized compromised workload can retrieve permitted secrets          |
| I-02      | Database data disclosure                           | High   | Backend-to-MySQL NetworkPolicy, Vault credentials, workload isolation                                                      | NetworkPolicy and backend-to-MySQL validation                         | Backend compromise can expose authorized database access                |
| I-04      | Kubernetes resource disclosure                     | High   | ServiceAccounts, Kyverno, Checkov, Trivy, workload hardening                                                               | Checkov: 263 passed / 0 failed / 11 skipped; Trivy: 324/324 resources | Full production Kubernetes control-plane hardening is outside scope     |
| I-05      | CI/CD and source disclosure                        | Medium | Repository access controls, Gitleaks, Checkov, Trivy, SBOM                                                                 | Gitleaks and Checkov validation; security pipeline                    | Repository compromise may expose architecture/security information      |
| D-01      | Application request flooding                       | High   | Ingress-NGINX, resource controls, Prometheus, Grafana alerts                                                               | Resource-policy validation; availability/restart alert testing        | No dedicated production rate-limiting or DDoS layer                     |
| D-02      | Kubernetes workload resource exhaustion            | High   | Resource requests/limits, Kyverno, Checkov, Prometheus/Grafana                                                             | Kyverno resource-policy validation; Checkov                           | Shared Kind/host resources remain a limitation                          |
| D-04      | Security infrastructure disruption                 | High   | Kubernetes isolation, NetworkPolicies, monitoring, workload controls                                                       | Vault, Wazuh, Prometheus/Grafana validation                           | Local security infrastructure lacks full HA                             |
| E-01      | Container privilege escalation                     | High   | Non-root, UID/GID restriction, no privilege escalation, dropped capabilities, read-only rootfs, seccomp                    | Backend security-context inspection; Kyverno validation               | Local runtime/host hardening is limited                                 |
| E-02      | ServiceAccount privilege escalation                | High   | Dedicated ServiceAccounts, Kyverno, NetworkPolicies, workload hardening                                                    | ServiceAccount inspection; Checkov; Trivy                             | Comprehensive production RBAC is outside scope                          |
| E-03      | Vault authorization escalation                     | High   | Kubernetes auth, exact policy, namespace binding, audience, TTL, NetworkPolicy                                             | Vault role/policy inspection; injection validation                    | Authorized workload retains permitted access                            |
| E-04      | Kyverno policy administration compromise           | High   | Kyverno, Checkov, Git history, protected policy workflow recommendations                                                   | Policy validation; Checkov; Git history                               | Privileged administrators may weaken security policies                  |
| E-05      | CI/CD privilege escalation                         | High   | Gitleaks, Checkov, Trivy, restricted workflow permissions, digest pinning                                                  | Checkov GitHub Actions scan; Gitleaks; image validation               | Privileged repository/CI identities remain high-value targets           |
| E-06      | Kubernetes administrative privilege abuse          | High   | Kyverno, workload security controls, NetworkPolicies, scanning                                                             | Checkov; Trivy; policy validation                                     | Fully privileged administrators retain broad control                    |

### 6.1.1 Control Coverage by Security Objective

The implemented controls provide defense in depth across the primary security objectives of the platform.

| Security Objective       | Primary Controls                                                       | Validation                                             |
| ------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------------ |
| Confidentiality          | Vault, NetworkPolicies, non-root containers, read-only root filesystem | Vault role/policy validation; NetworkPolicy validation |
| Integrity                | Kyverno, Checkov, Trivy, Gitleaks, digest-pinned images, Git history   | Security scans and policy validation                   |
| Availability             | Kubernetes resource controls, Prometheus, Grafana, workload monitoring | Alert testing and resource-policy validation           |
| Workload Isolation       | NetworkPolicies, ServiceAccounts, container hardening                  | NetworkPolicy and deployment inspection                |
| Identity & Authorization | ServiceAccounts, Vault Kubernetes auth, least-privilege Vault policy   | Vault role and ServiceAccount validation               |
| Privilege Restriction    | Non-root, no privilege escalation, dropped capabilities, seccomp       | Backend security-context validation; Kyverno           |
| Supply-Chain Security    | Gitleaks, Checkov, Trivy, SBOM, digest pinning                         | CI/security scans and SBOM generation                  |
| Security Monitoring      | Wazuh, Prometheus, Grafana                                             | Wazuh archive/index validation; monitoring alert tests |

### 6.1.2 Validation Evidence Summary

The principal security controls were validated using multiple independent mechanisms rather than relying solely on configuration review.

**Secret and identity security**

* Vault Kubernetes authentication was verified.
* `backend-sa` was verified as the identity bound to the Vault role.
* The Vault role was restricted to the `devsecops` namespace.
* The Kubernetes authentication audience was explicitly configured.
* The Vault token TTL was configured for one hour.
* The Vault policy was restricted to the required application secret path.
* Vault Agent secret injection was validated.

**Container and Kubernetes security**

* Backend container execution as non-root was verified.
* Privilege escalation was disabled.
* Linux capabilities were dropped.
* The backend filesystem was configured as read-only.
* `RuntimeDefault` seccomp was configured.
* Kubernetes NetworkPolicies were validated.
* Kyverno security policies were validated.
* Trivy Kubernetes scanning evaluated 324/324 resources.

**Security scanning**

Checkov validation produced:

* Kubernetes: 263 passed, 0 failed, 11 skipped
* Dockerfile: 113 passed, 0 failed, 0 skipped
* GitHub Actions: 204 passed, 0 failed, 0 skipped

Gitleaks repository scanning reported no detected secrets.

Trivy image scanning and CycloneDX SBOM generation were also completed for the project workloads.

**Security monitoring**

Wazuh archive collection was validated by generating a Windows test event and confirming that the event passed through the archive and indexing pipeline.

Prometheus and Grafana availability and restart alerts were tested against the backend deployment.

### 6.1.3 Residual Risk Principle

The presence of a security control does not imply that the associated threat has been eliminated.

The platform uses defense-in-depth controls to reduce the likelihood and potential impact of identified threats. Residual risk remains where:

* an attacker compromises an already-authorized workload;
* a privileged administrator is compromised;
* local security infrastructure is disrupted;
* production-grade high availability is not implemented;
* enterprise identity governance is outside scope;
* artifact signing and provenance are not implemented;
* comprehensive Kubernetes RBAC is outside scope;
* immutable independent audit storage is not implemented.

These residual risks are intentionally documented rather than represented as solved problems.

The threat model should be reviewed whenever significant changes are made to application architecture, Kubernetes permissions, Vault authorization, NetworkPolicies, CI/CD workflows, security policies, or monitoring infrastructure.


## 7. Residual Risks

The platform implements multiple layers of security controls, but the security engineering environment does not eliminate all threats.

Residual risks represent threats that remain partially mitigated, depend on trusted administrative identities, or are outside the scope of the local Kubernetes security platform.

### 7.1 Local Kubernetes Environment

**Risk:**

The platform runs on a local Kind Kubernetes cluster hosted on a developer workstation.

**Impact:**

Host compromise, Docker runtime compromise, or workstation resource exhaustion could affect the Kubernetes cluster and multiple security controls simultaneously.

**Current Mitigation:**

* Container security hardening
* Kubernetes security policies
* NetworkPolicies
* Kyverno
* Trivy
* Checkov
* Resource requests and limits
* Prometheus/Grafana monitoring

**Residual Risk:**

The local environment does not provide the isolation, redundancy, and infrastructure-level protections expected from a production Kubernetes platform.

**Production Recommendation:**

Use dedicated Kubernetes infrastructure with hardened nodes, controlled administrative access, cluster-level monitoring, backup/recovery procedures, and appropriate high-availability architecture.

---

### 7.2 Vault High Availability and Storage

**Risk:**

The local Vault deployment uses file storage and does not provide HA.

**Impact:**

Vault service or storage failure could temporarily affect application secret retrieval.

**Current Mitigation:**

* Kubernetes authentication
* Least-privilege Vault policy
* Short-lived tokens
* Dedicated ServiceAccount
* NetworkPolicy
* Vault Agent Injector

**Residual Risk:**

The local Vault deployment is not equivalent to a production-grade highly available secrets-management platform.

**Production Recommendation:**

Use an appropriate production Vault storage backend, HA architecture, protected backups, administrative access controls, audit logging, and secret-rotation procedures.

---

### 7.3 Authorized Workload Compromise

**Risk:**

A compromised backend workload may continue to use its legitimate identity and access resources authorized for the application.

**Impact:**

An attacker may retrieve authorized Vault secrets or access application database data.

**Current Mitigation:**

* Dedicated `backend-sa`
* Vault Kubernetes authentication
* Namespace restriction
* Explicit authentication audience
* One-hour token TTL
* Least-privilege Vault policy
* Backend NetworkPolicies
* Container hardening

**Residual Risk:**

Least privilege limits the blast radius but cannot prevent a compromised workload from using permissions intentionally granted to it.

**Production Recommendation:**

Use workload identity isolation, continuous runtime monitoring, secret rotation, database authorization boundaries, application-layer authorization, and additional workload isolation where required.

---

### 7.4 Kubernetes RBAC and Administrative Trust

**Risk:**

A highly privileged Kubernetes administrator may modify workloads, policies, NetworkPolicies, or other security-sensitive resources.

**Impact:**

An administrator with sufficient privileges could weaken or bypass multiple security controls.

**Current Mitigation:**

* Dedicated ServiceAccounts
* Kyverno policies
* Kubernetes security contexts
* NetworkPolicies
* Checkov
* Trivy
* Git-based configuration history

**Residual Risk:**

The project does not implement a complete enterprise Kubernetes RBAC and separation-of-duties architecture.

**Production Recommendation:**

Implement narrowly scoped Roles and RoleBindings, protected administrative identities, MFA, privileged-action monitoring, separation of duties, and independent audit logging.

---

### 7.5 Security-Policy Administration

**Risk:**

An attacker who compromises an identity capable of modifying Kyverno or other security policies may weaken the controls protecting the cluster.

**Impact:**

Security-policy modification could permit deployment of workloads with excessive privileges or insecure configurations.

**Current Mitigation:**

* Kyverno validation
* Checkov scanning
* Git-based policy history
* Security-policy validation

**Residual Risk:**

An administrator with sufficient privileges may still modify or disable security policies.

**Production Recommendation:**

Protect security-policy administration with strict RBAC, mandatory code review, protected branches, restricted administrators, change approval, and independent monitoring.

---

### 7.6 Frontend and MySQL Egress Controls

**Risk:**

The current NetworkPolicy configuration provides stronger egress restrictions for the backend than for the frontend and MySQL workloads.

**Impact:**

If either workload is compromised, unrestricted or less-restricted outbound communication may provide additional opportunities for lateral movement or data exfiltration.

**Current Mitigation:**

* Default-deny ingress
* Explicit frontend-to-backend communication
* Explicit backend-to-MySQL communication
* Backend egress restriction
* Backend-to-Vault restriction
* DNS restrictions for backend

**Residual Risk:**

Frontend and MySQL workloads do not currently have the same broad egress restrictions as the backend.

This is intentionally documented as a residual risk rather than being represented as fully mitigated.

**Production Recommendation:**

Apply workload-specific default-deny egress policies and explicitly allow only required destinations such as DNS, approved application services, monitoring endpoints, or backup systems.

---

### 7.7 MySQL Non-Root Exception

**Risk:**

The MySQL workload does not currently satisfy the same `runAsNonRoot` enforcement used for the hardened backend workload because enforcing the setting caused the local MySQL image/runtime to fail with a `setgid: Operation not permitted` error.

**Impact:**

The MySQL workload has a weaker container privilege baseline than the backend.

**Current Mitigation:**

* `allowPrivilegeEscalation: false`
* Capabilities dropped
* `RuntimeDefault` seccomp
* Resource controls
* NetworkPolicy isolation
* Kyverno validation of the other applicable controls

**Residual Risk:**

The local MySQL workload remains an exception to the `runAsNonRoot` policy.

**Production Recommendation:**

Use a database image/runtime configuration that supports non-root execution and enforce the restricted security baseline consistently before production deployment.

---

### 7.8 Application-Level Security

**Risk:**

Infrastructure security controls cannot prevent all vulnerabilities within application code or business logic.

**Impact:**

Application vulnerabilities such as authorization flaws, injection vulnerabilities, insecure session handling, or business-logic weaknesses could still be exploited through legitimate application interfaces.

**Current Mitigation:**

* Network segmentation
* Container hardening
* Secret management
* CI/CD security scanning
* Trivy
* Gitleaks
* Checkov

**Residual Risk:**

The project does not claim to provide complete application-security assurance.

Detailed application security testing remains a separate concern.

**Production Recommendation:**

Perform dedicated SAST, DAST, API security testing, dependency management, authentication/authorization testing, secure code review, and penetration testing appropriate to the application.

---

### 7.9 Supply-Chain Provenance

**Risk:**

Vulnerability scanning and SBOM generation do not by themselves prove that a container artifact originated from an authorized build process.

**Impact:**

A compromised build or artifact distribution process could potentially introduce an unauthorized image even if the image does not contain a known vulnerability.

**Current Mitigation:**

* GitHub Actions security pipeline
* Gitleaks
* Checkov
* Trivy
* CycloneDX SBOM
* Digest-pinned backend image

**Residual Risk:**

The project does not currently implement full cryptographic artifact signing and provenance verification.

**Production Recommendation:**

Implement signed container images, trusted build provenance, verification at deployment time, protected registries, and controlled release workflows.

---

### 7.10 Immutable Security Evidence

**Risk:**

Security telemetry and audit evidence stored within the local environment may be modified or deleted by an attacker with sufficient privileges.

**Impact:**

Incident investigation and forensic reconstruction may be impaired.

**Current Mitigation:**

* Wazuh Agent
* Wazuh Manager
* Wazuh Indexer
* Wazuh archive collection
* Filebeat archive forwarding
* Prometheus/Grafana monitoring

**Residual Risk:**

The project does not provide an independently administered immutable or write-protected audit repository.

**Production Recommendation:**

Forward high-value security logs to protected centralized storage with controlled administrative access, appropriate retention, tamper-evident mechanisms, and independent monitoring.

---

### 7.11 Residual Risk Summary

| ID    | Residual Risk                     | Impact | Current Status               | Production Direction                               |
| ----- | --------------------------------- | ------ | ---------------------------- | -------------------------------------------------- |
| RR-01 | Local Kind environment            | High   | Accepted for lab             | Hardened production Kubernetes infrastructure      |
| RR-02 | Vault file storage / no HA        | High   | Accepted for lab             | HA Vault with protected storage                    |
| RR-03 | Authorized workload compromise    | High   | Partially mitigated          | Stronger workload isolation and runtime monitoring |
| RR-04 | Kubernetes administrative trust   | High   | Partially mitigated          | Strict RBAC and separation of duties               |
| RR-05 | Security-policy administration    | High   | Partially mitigated          | Protected policy administration                    |
| RR-06 | Frontend/MySQL egress             | Medium | Residual                     | Workload-specific egress restrictions              |
| RR-07 | MySQL non-root exception          | Medium | Accepted lab exception       | Production-compatible non-root database image      |
| RR-08 | Application-level vulnerabilities | High   | Outside infrastructure scope | Dedicated application security testing             |
| RR-09 | Artifact provenance               | High   | Partially mitigated          | Signed artifacts and provenance verification       |
| RR-10 | Immutable security evidence       | High   | Partially mitigated          | Protected centralized audit storage                |

### 7.12 Residual Risk Management Principle

Residual risks are intentionally documented rather than hidden or represented as fully solved.

The security posture of the platform should therefore be understood as **defense in depth within a controlled security engineering environment**, rather than as a claim of complete production security.

Any future architecture change affecting workload identity, Kubernetes permissions, NetworkPolicies, Vault authorization, CI/CD, security policies, monitoring, or exposed application interfaces should trigger a review of the relevant residual risks.

Residual risks should be reassessed before production deployment based on the actual threat model, deployment architecture, business requirements, and organizational security controls.


## 8. Security Assumptions

The threat model is based on a defined set of security assumptions about the platform, its identities, infrastructure, administrative boundaries, and operating environment.

These assumptions establish the conditions under which the documented security controls and residual-risk assessments are considered valid.

### 8.1 Trusted Administrative Control

The threat model assumes that highly privileged administrators are controlled identities and are not intentionally malicious.

This includes:

* Kubernetes cluster administrators
* Vault administrators
* GitHub repository administrators
* CI/CD administrators
* Security-policy administrators

A compromise of one of these identities is treated as a high-impact threat and is documented as residual risk where applicable.

---

### 8.2 Source Repository Integrity

The threat model assumes that the Git repository is the authoritative source for application configuration, Kubernetes manifests, security policies, and CI/CD configuration.

Git history and CI/CD security checks provide change visibility, but repository access control remains a required security dependency.

Unauthorized repository access or administrator compromise may undermine source integrity and security-policy integrity.

---

### 8.3 CI/CD Execution Environment

The threat model assumes that the GitHub Actions execution environment is trusted for the duration of a security-pipeline execution.

The security pipeline is expected to execute the configured Gitleaks and Checkov checks against repository content.

The model does not assume that CI/CD compromise is impossible.

A compromised repository or sufficiently privileged CI/CD identity remains a supply-chain risk.

---

### 8.4 Kubernetes Control Plane

The threat model assumes that the Kubernetes control plane and API server are functioning according to their intended security model.

Kubernetes authorization and admission controls are therefore treated as foundational security mechanisms.

A fully compromised Kubernetes control plane or highly privileged cluster administrator is outside the primary workload threat boundary and is treated as a high-impact residual risk.

---

### 8.5 ServiceAccount Identity

The threat model assumes that Kubernetes ServiceAccounts are the intended workload identities for cluster-integrated security functions.

The backend is expected to use `backend-sa` rather than relying on a shared application identity.

The Vault authorization model depends on the integrity of this workload identity.

If an attacker compromises the backend workload, the attacker may inherit the permissions legitimately assigned to `backend-sa`.

---

### 8.6 Vault Authentication and Authorization

The threat model assumes that Vault Kubernetes authentication correctly validates the Kubernetes workload identity and applies the configured role and policy.

The backend's Vault access is intentionally restricted through:

* `backend-sa`
* `devsecops` namespace
* Explicit Kubernetes authentication audience
* One-hour token TTL
* `devsecops-backend` policy
* Read-only access to the required secret path

The model does not assume that Vault administration is immune to compromise.

---

### 8.7 NetworkPolicy Enforcement

The threat model assumes that Kubernetes NetworkPolicies are enforced by the cluster networking implementation.

NetworkPolicies therefore form part of the workload isolation boundary.

The model specifically recognizes that NetworkPolicy coverage is not uniform across all workloads. Backend egress is more tightly restricted than frontend and MySQL egress.

This difference is treated as a documented residual risk.

---

### 8.8 Container Runtime Security

The threat model assumes that the Kubernetes/container runtime correctly enforces configured security properties such as:

* Non-root execution
* Disabled privilege escalation
* Dropped capabilities
* Read-only root filesystem
* Seccomp `RuntimeDefault`

These controls reduce container privilege but do not provide a guarantee of complete host isolation.

---

### 8.9 Security Policy Integrity

The threat model assumes that Kyverno policies are maintained as security-sensitive configuration and are subject to controlled modification.

The model assumes that policy changes should be reviewed and validated before being trusted.

A sufficiently privileged administrator who can modify or disable security policies remains a trusted administrative dependency.

---

### 8.10 Monitoring and Security Telemetry

The threat model assumes that Prometheus, Grafana, and Wazuh provide operational and security telemetry according to their configured functions.

Monitoring is treated as a detection and investigation capability rather than a preventative control for every threat.

The model does not assume that locally stored telemetry is immutable.

Loss or manipulation of monitoring infrastructure is therefore documented as a residual risk.

---

### 8.11 Application Trust Boundary

The threat model assumes that the application itself remains within the infrastructure security boundary but does not assume that application code is free of vulnerabilities.

Infrastructure controls such as NetworkPolicies, Vault, container hardening, and Kubernetes policies are therefore treated as defense-in-depth protections.

Application-level vulnerabilities remain possible and are documented as outside the primary infrastructure threat-model scope.

---

### 8.12 Local Laboratory Environment

The threat model assumes that the platform is operated as a controlled security engineering laboratory using a local Kind Kubernetes cluster.

The local environment is not treated as equivalent to a production Kubernetes deployment.

Known environmental limitations include:

* Kind running on a developer workstation
* Vault file storage
* Vault HA disabled
* Local Wazuh deployment
* Shared host resources
* Limited production-grade administrative separation

These limitations are explicitly represented as residual risks rather than hidden assumptions.

---

### 8.13 Security Scanning Integrity

The threat model assumes that Gitleaks, Checkov, Trivy, and SBOM generation execute against the intended repository, images, and Kubernetes resources.

Security scanners are treated as supporting controls rather than absolute guarantees.

A clean scan result means that the scanner did not identify the tested class of issue under its configured rules and scope; it does not prove that the system is free from all vulnerabilities.

---

### 8.14 Secret Handling

The threat model assumes that application secrets are intended to be retrieved through Vault rather than embedded directly into container images or source code.

The platform therefore treats Vault as the authoritative secret-management boundary for the backend's database credentials.

Secret exposure through application logs, debugging output, source-code changes, or administrative access remains a potential threat.

---

### 8.15 Assumption Management Principle

Security assumptions should be reviewed whenever the architecture, deployment model, identity model, or trust boundaries change.

An assumption that becomes invalid may invalidate part of the associated threat assessment.

Examples include:

* moving from Kind to production Kubernetes;
* introducing additional ServiceAccounts;
* changing Vault authentication;
* expanding Kubernetes permissions;
* modifying NetworkPolicies;
* changing CI/CD permissions;
* introducing new externally exposed services;
* changing security-policy administration;
* introducing new monitoring or logging infrastructure.

The threat model should therefore be maintained as a living security artifact rather than treated as a one-time document.



## 9. Validation Evidence

The security controls described in this threat model were validated through configuration inspection, security scanning, Kubernetes policy evaluation, workload testing, secret-management validation, monitoring tests, and security-telemetry verification.

The following evidence represents the validation performed within the security engineering environment.

### 9.1 Repository and Secret Scanning

**Control:** Gitleaks repository secret scanning

**Objective:**

Identify accidentally committed credentials, tokens, passwords, private keys, and other detectable secrets within repository history.

**Validation:**

The repository history was scanned using Gitleaks.

**Result:**

* 19 commits scanned
* Approximately 187 KB of repository history evaluated
* 0 detected leaks

**Security Relevance:**

Supports mitigation of:

* T-01 Source tampering
* I-03 Configuration/secret disclosure
* I-05 CI/CD/source disclosure
* E-05 CI/CD privilege escalation

**Limitation:**

A clean Gitleaks result does not prove that no secret exists anywhere in the system. Runtime-generated secrets and secrets outside the scanner's detection scope may still exist.

---

### 9.2 Checkov Infrastructure and CI/CD Scanning

**Control:** Checkov

**Objective:**

Identify insecure Kubernetes, container, and CI/CD configuration patterns.

**Validation Results:**

**Kubernetes:**

* 263 passed
* 0 failed
* 11 skipped

**Dockerfile:**

* 113 passed
* 0 failed
* 0 skipped

**GitHub Actions:**

* 204 passed
* 0 failed
* 0 skipped

**Security Relevance:**

Supports validation of:

* Kubernetes security configuration
* Container hardening
* CI/CD security configuration
* Security-policy configuration
* Resource and workload controls

**Limitation:**

Skipped checks are not equivalent to passed checks, and static configuration analysis cannot detect every runtime or application-level vulnerability.

---

### 9.3 Trivy Kubernetes Security Scanning

**Control:** Trivy Kubernetes scanning

**Objective:**

Evaluate Kubernetes resources and cluster configuration for security-relevant findings.

**Validation:**

The Kind cluster was scanned using Trivy Kubernetes scanning.

**Result:**

* 324/324 Kubernetes resources evaluated
* Node scanning enabled

**Security Relevance:**

Supports validation of:

* T-04 Kubernetes configuration tampering
* I-04 Kubernetes resource disclosure
* E-02 ServiceAccount privilege escalation
* E-06 Kubernetes administrative privilege abuse

**Limitation:**

Trivy scanning provides security findings based on its configured scanners and rules. It does not prove that the Kubernetes cluster is completely secure.

---

### 9.4 Software Bill of Materials

**Control:** CycloneDX SBOM generation

**Objective:**

Provide software-component inventory information for containerized workloads.

**Validation:**

SBOM artifacts were generated for the project workloads, including:

* `backend-sbom.json`
* `frontend-sbom.json`
* `mysql-sbom.json`

The backend image SBOM was generated using CycloneDX-compatible output.

**Security Relevance:**

Supports:

* Software supply-chain visibility
* Component inventory
* Vulnerability-management workflows
* Incident investigation
* Artifact traceability

**Limitation:**

An SBOM identifies software components but does not by itself establish artifact authenticity or provenance.

---

### 9.5 Container Security Validation

**Control:** Hardened backend container security context

**Objective:**

Reduce the impact of container compromise and prevent unnecessary privilege escalation.

**Validated Configuration:**

* `runAsNonRoot: true`
* UID `10001`
* GID `10001`
* `allowPrivilegeEscalation: false`
* Linux capabilities dropped
* `readOnlyRootFilesystem: true`
* `seccompProfile: RuntimeDefault`
* Isolated writable `/tmp`

**Security Relevance:**

Supports:

* E-01 Container privilege escalation
* I-03 Container/configuration disclosure
* D-02 Workload resource exhaustion
* E-06 Administrative privilege abuse

**Limitation:**

These controls reduce container privileges but do not guarantee complete host isolation.

---

### 9.6 Kubernetes Network Segmentation Validation

**Control:** Kubernetes NetworkPolicies

**Objective:**

Restrict workload-to-workload and workload-to-service communication to intended paths.

**Validated Policies:**

* `default-deny-ingress`
* `allow-frontend-to-backend`
* `allow-backend-to-mysql`
* `allow-backend-to-vault`
* `allow-ingress-to-frontend`
* `allow-ingress-to-backend`
* `backend-egress-restriction`

**Validated Communication Paths:**

```text
Ingress
   |
   +----> Frontend
   |
   +----> Backend
            |
            +----> MySQL
            |
            +----> Vault
            |
            +----> DNS
```

**Security Relevance:**

Supports:

* Workload isolation
* Lateral-movement reduction
* Database access restriction
* Vault access restriction
* Network-level defense in depth

**Limitation:**

Backend egress has stronger restrictions than frontend and MySQL egress. This difference remains a documented residual risk.

---

### 9.7 Kyverno Policy Validation

**Control:** Kyverno security-policy validation

**Objective:**

Continuously validate workload security properties against defined policies.

**Validated Policy Areas:**

* Non-root execution
* Privilege-escalation restriction
* Seccomp `RuntimeDefault`
* Resource requests and limits

**Security Relevance:**

Supports:

* E-01 Container privilege escalation
* E-02 ServiceAccount privilege escalation
* T-05 Kyverno policy tampering
* D-02 Workload resource exhaustion

**Known Exception:**

The MySQL workload currently does not satisfy the same `runAsNonRoot` enforcement because enforcing the setting caused the local MySQL image/runtime to fail with:

`setgid: Operation not permitted`

Other applicable security controls remain enforced.

This is documented as a local lab compatibility exception rather than a production recommendation.

---

### 9.8 Vault Authentication and Authorization Validation

**Control:** Vault Kubernetes authentication

**Objective:**

Ensure that backend secret access is restricted to the intended workload identity and secret path.

**Validated Role Configuration:**

* Role: `devsecops-backend`
* ServiceAccount: `backend-sa`
* Namespace: `devsecops`
* Kubernetes audience: explicitly configured
* Token TTL: 1 hour
* Policy: `devsecops-backend`

**Validated Policy:**

The policy provides read access only to:

`secret/data/devsecops/backend`

**Secret Injection:**

Vault Agent Injector was validated on the backend deployment.

**Security Relevance:**

Supports:

* S-01 ServiceAccount identity spoofing
* S-02 Vault role impersonation
* I-01 Vault secret disclosure
* E-03 Vault authorization escalation

**Limitation:**

A compromised backend workload can still use its legitimate authorization to retrieve secrets permitted to its identity.

---

### 9.9 Kubernetes Workload Identity Validation

**Control:** Dedicated Kubernetes ServiceAccounts

**Objective:**

Avoid unnecessary reliance on shared or default workload identities.

**Validated ServiceAccounts:**

* `backend-sa`
* `frontend-sa`
* `mysql-sa`

The backend Deployment was verified to explicitly use:

`backend-sa`

**Security Relevance:**

Supports:

* Workload identity isolation
* Vault Kubernetes authentication
* Least-privilege authorization
* Privilege-escalation mitigation

---

### 9.10 Prometheus and Grafana Monitoring Validation

**Control:** Prometheus and Grafana

**Objective:**

Detect workload availability and restart conditions.

**Validated Alerts:**

* `BackendDeploymentReplicasLow`
* `BackendPodRestartDetected`
* `BackendDeploymentUnavailable`

**Validation:**

Backend replica availability was deliberately changed and restored to validate the monitoring and alerting workflow.

**Security Relevance:**

Supports:

* D-01 Application availability
* D-02 Workload resource/availability monitoring
* D-04 Security infrastructure monitoring
* R-01 Kubernetes activity visibility

**Limitation:**

Prometheus and Grafana provide monitoring and alerting rather than complete security audit logging.

---

### 9.11 Wazuh Security Telemetry Validation

**Control:** Wazuh endpoint-to-index telemetry pipeline

**Objective:**

Validate security-event collection, archival, forwarding, and investigation capability.

**Validated Flow:**

```text
Windows Endpoint
       |
       v
Wazuh Agent
       |
       v
Wazuh Manager
       |
       v
Archive Collection
       |
       v
Filebeat
       |
       v
Wazuh Indexer
       |
       v
Investigation / Dashboard
```

**Validation Event:**

A Windows test event was generated using:

`eventcreate /T INFORMATION /ID 998 /L APPLICATION /SO WazuhTest /D "Wazuh archive indexed test"`

The event was successfully observed through the Wazuh archive pipeline and indexed for investigation.

**Security Relevance:**

Supports:

* R-03 Security telemetry manipulation
* I-06 Telemetry disclosure
* D-04 Security infrastructure availability
* Security investigation capability

**Limitation:**

The local Wazuh deployment is not an independently administered immutable evidence store.

---

### 9.12 Image and Artifact Integrity Validation

**Control:** Digest-pinned container image

**Objective:**

Reduce the risk of silently changing the backend container image reference.

**Validation:**

The backend Kubernetes Deployment uses a digest-pinned container image rather than relying only on a mutable image tag.

**Security Relevance:**

Supports:

* T-03 Container image tampering
* E-05 CI/CD privilege escalation
* Supply-chain integrity

**Limitation:**

Digest pinning provides immutable image reference semantics but does not by itself prove who built or signed the image.

Artifact signing and provenance verification remain production recommendations.

---

### 9.13 Validation Coverage Summary

| Control Area             | Validation Method                      | Result                                   |
| ------------------------ | -------------------------------------- | ---------------------------------------- |
| Repository secrets       | Gitleaks                               | 0 detected leaks                         |
| Kubernetes configuration | Checkov                                | 263 passed / 0 failed / 11 skipped       |
| Dockerfile security      | Checkov                                | 113 passed / 0 failed / 0 skipped        |
| GitHub Actions           | Checkov                                | 204 passed / 0 failed / 0 skipped        |
| Kubernetes cluster       | Trivy                                  | 324/324 resources evaluated              |
| SBOM                     | CycloneDX                              | SBOM artifacts generated                 |
| Container hardening      | Deployment inspection / Kyverno        | Hardened backend configuration validated |
| Network segmentation     | NetworkPolicy inspection/testing       | Intended communication paths validated   |
| Policy enforcement       | Kyverno                                | Security-policy validation operational   |
| Secret management        | Vault role/policy/injection validation | Backend secret flow validated            |
| Workload identity        | ServiceAccount inspection              | Dedicated identities validated           |
| Monitoring               | Prometheus/Grafana                     | Availability/restart alerts tested       |
| Security telemetry       | Wazuh archive/index pipeline           | Windows test event indexed successfully  |
| Image integrity          | Digest inspection                      | Backend image reference pinned by digest |

### 9.14 Evidence Integrity Principle

Validation evidence should be interpreted according to the scope and limitations of each test.

A successful security scan, policy evaluation, or configuration check demonstrates that the tested control operated as expected under the tested conditions.

It does not establish that the entire platform is free from vulnerabilities.

The threat model therefore treats validation results as evidence supporting the effectiveness of specific controls rather than as proof of absolute security.

Future changes to the platform should trigger re-validation of the affected controls and an update to this evidence section where the validation results change.



## 10. Production Hardening Recommendations

The platform was developed and validated as a security engineering and DevSecOps laboratory environment. The following recommendations describe additional controls that should be considered before deploying an equivalent architecture into a production environment.

These recommendations are intentionally documented rather than implemented as part of this project because the current project scope is a local Kubernetes security validation environment.

### 10.1 Production Kubernetes Platform

Replace the local Kind cluster with a production-supported Kubernetes platform providing:

* Highly available control-plane components
* Protected and encrypted control-plane communication
* Hardened worker nodes
* Controlled administrative access
* Centralized audit logging
* Defined backup and recovery procedures
* Strong separation between platform administrators and application operators

Production Kubernetes RBAC should follow least privilege and avoid unnecessary cluster-wide permissions.

The Kubernetes control plane should be treated as a high-value trust boundary because compromise of administrative Kubernetes privileges can undermine workload isolation and security-policy enforcement.

---

### 10.2 Vault High Availability and Storage Protection

The current environment uses Vault file storage with HA disabled.

A production deployment should use:

* Highly available Vault architecture
* Production-supported storage backend
* Encrypted storage
* Protected Vault administrative access
* Automated backup and recovery procedures
* Secret rotation and lifecycle management
* Strong separation of Vault administration from application administration
* Monitoring of Vault authentication and authorization events

The backend's least-privilege Kubernetes authentication model should be retained when moving to production.

---

### 10.3 Workload Egress Restrictions

The current platform provides stronger egress restrictions for the backend than for the frontend and MySQL workloads.

Production deployments should evaluate default-deny egress and explicitly allow only required destinations for each workload.

For example:

* Frontend → only required external services
* Backend → required database, Vault, DNS, and approved external services
* MySQL → only required replication, backup, monitoring, or management paths

This reduces opportunities for command-and-control communication, data exfiltration, and lateral movement.

---

### 10.4 MySQL Non-Root Execution

The current MySQL workload contains a documented `runAsNonRoot` compatibility exception because enforcing the setting caused the local workload to fail with:

`setgid: Operation not permitted`

For production, the database workload should use a database image and runtime configuration that supports non-root execution while maintaining required functionality.

The exception should therefore be treated as a lab compatibility limitation rather than an accepted production security baseline.

---

### 10.5 Kubernetes Policy Administration

Kyverno policies should be protected as security-sensitive infrastructure.

Production recommendations include:

* Restricting Kyverno administrative permissions
* Requiring peer review for policy changes
* Protecting policy repositories and deployment pipelines
* Monitoring policy modifications
* Preventing unauthorized disabling or deletion of security policies
* Testing policy changes before production rollout

A compromised policy administrator can potentially weaken multiple workload security controls simultaneously.

---

### 10.6 Container and Runtime Hardening

The existing backend container hardening should be retained and extended where practical.

Production workloads should additionally consider:

* Minimal and regularly patched base images
* Image vulnerability scanning
* Digest-pinned images
* Runtime security monitoring
* Read-only filesystems where compatible
* Dropped Linux capabilities
* Non-root execution
* Restricted privilege escalation
* Seccomp and appropriate runtime security profiles
* Workload-specific resource limits

Container hardening should be treated as defense in depth rather than as a replacement for application security.

---

### 10.7 Software Supply-Chain Integrity

The current platform provides secret scanning, IaC scanning, container scanning, SBOM generation, and digest-pinned image references.

Production environments should additionally consider:

* Signed container images
* Artifact signature verification before deployment
* Build provenance and attestations
* Protected build environments
* Restricted CI/CD permissions
* Mandatory pull-request review
* Protected branches
* Strong repository authentication and MFA
* Separation of build and deployment privileges
* Controlled release promotion

OWASP identifies CI/CD and software supply-chain infrastructure as important security targets and recommends protecting source control, pipeline configuration, build environments, and artifact integrity.

---

### 10.8 Application Security

Infrastructure security controls do not eliminate vulnerabilities within the application itself.

Before production deployment, the application should undergo appropriate security testing, including where applicable:

* Secure code review
* SAST
* DAST
* API security testing
* Authentication and authorization testing
* Input-validation testing
* Dependency vulnerability management
* Business-logic abuse-case testing
* Session-management testing
* Security regression testing

Application vulnerabilities remain an explicit residual risk in this threat model.

---

### 10.9 Production Ingress and Transport Security

Production ingress should provide appropriate transport and application-layer protections, including:

* TLS for external communication
* Managed certificate lifecycle
* Secure TLS configuration
* Appropriate security headers
* Authentication and authorization controls
* Request-size and timeout controls
* Rate limiting where appropriate
* DDoS protection appropriate to the deployment environment
* Restricted administrative endpoints

These controls should complement the existing Kubernetes network segmentation rather than replace it.

---

### 10.10 Monitoring, Detection, and Incident Response

The current platform provides Prometheus/Grafana monitoring and Wazuh security telemetry.

Production environments should extend this capability with:

* Centralized security logging
* Kubernetes audit-log collection
* Protected log storage
* Tamper-resistant or immutable security evidence
* Alert correlation
* Defined incident-response procedures
* Alert ownership and escalation
* Security-event retention requirements
* Time synchronization across security infrastructure
* Regular detection testing

Monitoring should support both detection and investigation while avoiding the exposure of credentials, tokens, or other sensitive secret material.

---

### 10.11 Backup and Disaster Recovery

Production deployments should define and regularly test recovery procedures for:

* Application data
* Database backups
* Kubernetes configuration
* Vault data and configuration
* Security telemetry
* Critical CI/CD configuration
* Security policies

Backups should be protected against unauthorized modification and deletion.

Recovery objectives should be defined according to the business requirements of the production system.

---

### 10.12 Identity and Administrative Security

Production administrative access should use strong identity controls, including:

* MFA
* Least-privilege administrative roles
* Separation of duties
* Privileged-access management where appropriate
* Short-lived administrative credentials
* Administrative activity logging
* Periodic access reviews
* Removal of unused accounts and permissions

Administrative compromise remains one of the highest-impact residual risks because privileged identities can bypass multiple technical security boundaries.

---

### 10.13 Security Testing and Continuous Validation

Security validation should continue after initial deployment.

Recommended activities include:

* Re-running security scans after significant configuration changes
* Periodic container and dependency scanning
* Kubernetes configuration reviews
* Network-policy validation
* Kyverno policy testing
* Vault authorization testing
* CI/CD security reviews
* Periodic threat-model review
* Penetration testing appropriate to the application
* Incident-response and detection exercises

OWASP recommends treating threat modeling as a maintained process rather than a one-time activity and emphasizes reviewing whether mitigations can actually be tested.

---

### 10.14 Production Readiness Summary

| Area                 | Current Lab State                              | Production Recommendation                               |
| -------------------- | ---------------------------------------------- | ------------------------------------------------------- |
| Kubernetes           | Local Kind                                     | HA production Kubernetes                                |
| Vault                | File storage, HA disabled                      | HA Vault with protected storage                         |
| Workload identity    | Dedicated ServiceAccounts                      | Strict production RBAC and identity governance          |
| Network segmentation | NetworkPolicies implemented                    | Default-deny ingress and workload-specific egress       |
| Container security   | Hardened backend                               | Extend hardening across all compatible workloads        |
| MySQL                | Non-root exception                             | Production-compatible non-root database image           |
| Policy enforcement   | Kyverno                                        | Protected policy administration and change control      |
| Supply chain         | Gitleaks, Checkov, Trivy, SBOM, digest pinning | Signing, provenance, protected builds                   |
| Monitoring           | Prometheus/Grafana                             | Centralized monitoring and security operations          |
| Security telemetry   | Wazuh                                          | Protected centralized and immutable evidence            |
| Application security | Infrastructure-focused                         | Full application security testing                       |
| Ingress              | Lab configuration                              | TLS, rate limiting, DDoS and application-layer controls |
| Recovery             | Lab environment                                | Tested backup and disaster recovery                     |
| Threat model         | Documented                                     | Continuously maintained                                 |

### 10.15 Production Hardening Principle

The controls implemented in this project establish a layered security baseline for a DevSecOps Kubernetes environment.

Production readiness should not be interpreted as simply enabling additional security tools. It requires strengthening the underlying trust boundaries, identities, administrative controls, availability, supply-chain integrity, application security, monitoring, and recovery capabilities.

The production recommendations in this section therefore represent the next security maturity stage rather than claims about the current laboratory environment.

The threat model should be reviewed and updated whenever significant changes are made to architecture, deployment environment, workload identity, network connectivity, CI/CD privileges, security policies, or security infrastructure.
