# Security Exceptions

## 1. Purpose

This document records security controls that could not be fully enforced in the local DevSecOps Zero Trust Platform environment and explains the compensating controls and rationale.

The purpose of documenting an exception is to distinguish between:

* A control that passed validation
* A control that was intentionally not applicable
* A control that could not be enforced because of workload compatibility
* A risk that requires further remediation in a production environment

Security exceptions are documented rather than hidden from the overall security assessment.

---

# 2. Exception Summary

| ID    | Component | Control              | Status    | Reason                                                |
| ----- | --------- | -------------------- | --------- | ----------------------------------------------------- |
| EX-01 | MySQL     | `runAsNonRoot: true` | Exception | MySQL container failed when forced to run as non-root |

There is currently one documented security exception in the project.

---

# 3. EX-01 — MySQL Non-Root Execution

## Affected Component

```text
Namespace: devsecops
Workload: MySQL StatefulSet
Container: mysql
Image: mysql:8.4
```

## Security Control

The project attempts to enforce:

```yaml
securityContext:
  runAsNonRoot: true
```

Running application workloads as non-root is a desired Kubernetes security control because it reduces the privileges available to a compromised application process.

---

# 4. Observed Behavior

When the MySQL workload was configured to run as non-root, the container failed during startup.

The observed error was:

```text
setgid: Operation not permitted
```

The failure prevented the MySQL container from starting correctly.

The issue was therefore treated as a workload compatibility issue rather than simply bypassing the policy without investigation.

---

# 5. Validation Performed

The MySQL container was inspected after removing the incompatible non-root requirement.

The resulting runtime identity was:

```text
uid=0(root)
gid=0(root)
groups=0(root)
```

This confirms that the MySQL container continues to run as root in the local environment.

The exception is therefore a real security deviation and is not represented as a successful `runAsNonRoot` validation.

---

# 6. Kyverno Evidence

Kyverno was configured with a policy requiring non-root execution.

The MySQL PolicyReport showed:

```text
No privilege escalation        PASS
Resource requests/limits       PASS
Seccomp RuntimeDefault         PASS
runAsNonRoot                   FAIL
```

The resulting report contained:

```text
Error: 0
Fail: 1
Pass: 3
```

The `runAsNonRoot` failure corresponds to this documented exception.

Kyverno's `Audit` behavior is designed to report policy violations without blocking the resource, allowing existing workloads and their violations to be observed through PolicyReports.

---

# 7. Compensating Controls

Although the MySQL workload remains root-based, additional security controls remain enabled.

The MySQL workload uses:

```text
allowPrivilegeEscalation: false
Seccomp: RuntimeDefault
Capability restrictions
```

The workload is also protected by Kubernetes NetworkPolicies.

The database is not directly exposed to the frontend.

The intended communication path is:

```text
Frontend
    |
    X
    |
   MySQL

Backend
    |
    | TCP 3306
    v
  MySQL
```

Therefore, the MySQL workload is still subject to multiple layers of defense.

---

# 8. Network Isolation

The database is protected by the project's NetworkPolicy model.

The backend is explicitly permitted to communicate with MySQL on:

```text
TCP/3306
```

Unnecessary application-to-database paths are denied.

In particular:

```text
Frontend → MySQL
```

is not an allowed application path.

This limits the ability of a compromised frontend workload to directly interact with the database.

---

# 9. Secret Protection

Database credentials are not stored directly in the application deployment manifest as the primary secret-management mechanism.

Vault is used to provide database credentials to the backend workload.

The flow is:

```text
Vault
  |
  v
Kubernetes Authentication
  |
  v
backend-sa
  |
  v
Vault Agent Injector
  |
  v
Backend
  |
  v
MySQL
```

The database therefore remains protected by the broader secrets-management architecture even though its container runs as root.

---

# 10. Why the Exception Was Accepted

The project is a local security laboratory rather than a production database platform.

Forcing the MySQL image to run as non-root caused the database workload to fail.

Maintaining a functioning application while retaining other security controls was considered more useful for this project than forcing an incompatible configuration.

The decision was therefore:

```text
Do not force incompatible configuration
            +
Retain compensating controls
            +
Record the exception
            +
Identify production remediation
```

---

# 11. Risk Assessment

### Risk

A root-running database process has greater privileges inside its container than a non-root process.

If the MySQL process were compromised, the attacker could potentially gain greater privileges within the container environment.

### Current Risk Reduction

The risk is reduced through:

* Network isolation
* Backend-only database access
* No direct frontend-to-MySQL path
* Restricted privilege escalation
* Seccomp RuntimeDefault
* Capability restrictions
* Vault-based secret management
* Kubernetes workload isolation
* Runtime monitoring

### Residual Risk

The residual risk is:

```text
MEDIUM — Local laboratory exception
```

The rating reflects the increased privilege of the MySQL process while considering the compensating controls surrounding the workload.

This rating is specific to the project's local laboratory context and should not automatically be reused for a production risk assessment.

---

# 12. Production Remediation

For a production deployment, the preferred approach would be to evaluate a database image or deployment model that supports non-root execution without breaking database initialization and runtime behavior.

Potential remediation activities include:

1. Evaluate a supported non-root MySQL-compatible image.
2. Validate filesystem ownership and initialization requirements.
3. Test database startup and upgrade behavior.
4. Validate persistent-volume permissions.
5. Re-enable:

```yaml
runAsNonRoot: true
```

6. Confirm the workload passes the Kyverno policy.
7. Re-run Checkov and Kubernetes security validation.
8. Perform application regression testing.

The goal is not simply to suppress the policy finding but to make the workload genuinely compatible with non-root execution.

---

# 13. Exception Acceptance Criteria

The exception is considered acceptable for the current laboratory project because:

* The root execution was identified through testing.
* The failure reason was investigated.
* The behavior was reproduced.
* The exception is documented.
* Compensating controls remain enabled.
* Network access is restricted.
* Database credentials are managed through Vault.
* Kyverno continues to report the policy violation.
* The exception is not represented as a successful security control.

---

# 14. Security Principle

The project follows the principle:

```text
A documented security exception
is better than
an undocumented security bypass.
```

The objective is to maintain visibility of residual risk while preserving a functional security laboratory.

Kyverno supports this type of workflow through policy reporting and controlled validation behavior. Policy results can be recorded in PolicyReports, allowing security teams to identify and track non-compliant resources.

---

# 15. Final Assessment

The MySQL `runAsNonRoot` requirement is the only known security exception intentionally retained in the current project scope.

The exception does not invalidate the other implemented controls.

The final security posture is therefore:

```text
MySQL Non-Root
       |
       X
   Exception
       |
       +----------------------+
       |                      |
       v                      v
Network Isolation       Runtime Hardening
       |                      |
       v                      v
Vault Secrets           Seccomp RuntimeDefault
       |                Capability Restrictions
       |                No Privilege Escalation
       +----------+-----------+
                  |
                  v
           Reduced Residual Risk
```

This exception should be revisited if the platform is migrated from the local laboratory environment to a production Kubernetes environment.
