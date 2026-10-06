# Prometheus and Grafana Monitoring

## 1. Purpose

This document describes the monitoring and alerting layer implemented in the DevSecOps Zero Trust Platform.

The monitoring stack provides operational visibility into the Kubernetes environment and helps detect workload availability and runtime health issues.

The implementation uses:

* Prometheus
* Grafana
* Alertmanager
* Kubernetes metrics
* kube-state-metrics
* Node Exporter

The monitoring layer complements the security controls implemented through Kubernetes, Vault, Kyverno, Trivy, Checkov, Gitleaks, and Wazuh.

---

# 2. Monitoring Objectives

The monitoring implementation was designed to provide visibility into:

* Kubernetes workload health
* Deployment availability
* Pod restarts
* Replica availability
* Node-level metrics
* Kubernetes object state
* Security-relevant operational conditions
* Application availability

The primary objective is to identify conditions that require investigation before they become larger operational or security problems.

---

# 3. Monitoring Architecture

The implemented monitoring architecture is:

```text
                    Kubernetes Cluster
                           |
          ┌────────────────┼────────────────┐
          |                |                |
          v                v                v
   kube-state-metrics   Node Exporter   Application/
          |                |             K8s Metrics
          |                |                |
          └────────────────┼────────────────┘
                           |
                           v
                    ┌─────────────┐
                    │ Prometheus  │
                    │             │
                    │ Time Series │
                    └──────┬──────┘
                           |
              ┌────────────┴────────────┐
              |                         |
              v                         v
       ┌─────────────┐          ┌─────────────┐
       │   Grafana   │          │ Alertmanager │
       │ Dashboards   │          │              │
       │ Alert Rules  │          │ Notifications│
       └─────────────┘          └─────────────┘
```

Prometheus collects and stores metrics as time-series data and supports querying and alerting over those metrics.

---

# 4. Prometheus

Prometheus is the primary metrics collection and storage component.

It collects numerical measurements from monitored targets and stores them as time-series data.

Prometheus uses a pull-based model to scrape metrics from HTTP endpoints and can use service discovery to identify monitoring targets.

In this project, Prometheus is responsible for collecting Kubernetes infrastructure and workload metrics.

---

# 5. Kubernetes Monitoring

The Kubernetes monitoring stack collects information about:

```text
Nodes
Pods
Deployments
ReplicaSets
StatefulSets
Containers
Resource Usage
Pod Restarts
Workload Availability
```

This allows the platform to determine whether workloads are operating as expected.

For example:

```text
Deployment Desired Replicas
            |
            v
Deployment Available Replicas
            |
            v
Availability Assessment
            |
            v
Alert if Required
```

---

# 6. kube-state-metrics

kube-state-metrics exposes Kubernetes object state as Prometheus metrics.

This provides visibility into objects such as:

* Deployments
* Pods
* ReplicaSets
* StatefulSets
* Nodes
* Jobs
* Other Kubernetes resources

This is particularly useful for the project's deployment availability alerts.

For example, Prometheus can query deployment replica metrics to determine whether the expected number of backend replicas is available.

---

# 7. Node Exporter

Node Exporter provides host-level metrics that can be collected by Prometheus.

The monitoring stack has Node Exporter enabled.

This provides additional infrastructure-level visibility into the Kubernetes environment.

The monitoring architecture therefore covers both:

```text
Kubernetes Object State
        +
Infrastructure Metrics
```

---

# 8. Monitoring Namespace

The monitoring stack is deployed in the:

```text
monitoring
```

namespace.

The monitoring environment includes components such as:

```text
Prometheus
Prometheus Operator
Alertmanager
Grafana
kube-state-metrics
Node Exporter
```

The components work together to provide metrics collection, visualization, and alerting.

---

# 9. Prometheus Service

The Prometheus service is available internally through the Kubernetes service:

```text
monitoring-kube-prometheus-prometheus.monitoring:9090
```

This service is also configured as the primary Prometheus data source for Grafana.

---

# 10. Grafana

Grafana provides the visualization and alert-management interface.

The deployed Grafana version is:

```text
13.2.2-distroless
```

Grafana connects to Prometheus as a data source and queries its time-series data.

Grafana's Prometheus data source supports metrics, querying, visualization, and alerting.

---

# 11. Grafana Data Sources

The monitoring environment contains the following relevant Grafana data sources:

```text
Prometheus
    |
    v
monitoring-kube-prometheus-prometheus.monitoring:9090
```

and:

```text
Alertmanager
    |
    v
monitoring-kube-prometheus-alertmanager.monitoring:9093
```

The Prometheus data source provides the metrics used by dashboards and alert rules.

The Alertmanager data source provides access to Alertmanager-related alerting resources.

Grafana supports Prometheus and Alertmanager as integrated data sources.

---

# 12. Grafana Alert Folder

The project uses the Grafana folder:

```text
DevSecOps Kubernetes
```

This provides a logical location for the project's Kubernetes security and availability alerts.

The alert group is:

```text
Kubernetes Alerts
```

The configured evaluation interval is:

```text
1m
```

This means the relevant alert group is evaluated approximately once per minute.

Grafana documents that evaluation intervals determine how frequently alert queries are evaluated.

---

# 13. Alerting Architecture

The monitoring alert flow can be represented as:

```text
Kubernetes
    |
    v
Metrics
    |
    v
Prometheus
    |
    v
PromQL Query
    |
    v
Alert Condition
    |
    v
Alert Rule
    |
    v
Alert State
    |
    v
Grafana / Alertmanager
```

The alerting architecture separates metric collection from alert evaluation and notification handling.

Prometheus-native alerting uses rules to generate alerts and Alertmanager to handle alert management such as grouping, silencing, inhibition, and notifications.

---

# 14. BackendDeploymentReplicasLow

The first major alert is:

```text
BackendDeploymentReplicasLow
```

Its purpose is to detect insufficient backend availability.

The condition is based on:

```text
Available Backend Replicas < 2
```

for the configured duration.

The intended healthy state is:

```text
Desired Replicas = 2
Available Replicas = 2
```

If the available replicas fall below the expected level, the alert indicates a potential availability problem.

---

# 15. Why Replica Monitoring Matters

Multiple backend replicas provide basic workload redundancy.

For example:

```text
Healthy State

Backend Pod 1    Running
Backend Pod 2    Running
       |
       v
Available Replicas = 2
```

If one replica becomes unavailable:

```text
Backend Pod 1    Running
Backend Pod 2    Failed
       |
       v
Available Replicas = 1
```

The monitoring system can then detect the reduction in availability.

This helps identify:

* Pod failures
* Scheduling problems
* CrashLoopBackOff conditions
* Resource issues
* Deployment problems
* Unexpected workload loss

---

# 16. BackendPodRestartDetected

The second alert is:

```text
BackendPodRestartDetected
```

This alert monitors backend pod restart activity.

Unexpected restarts can indicate:

* Application crashes
* Container failures
* Resource pressure
* Configuration problems
* Dependency failures
* Runtime instability

The alert therefore provides an early indication that the backend workload may require investigation.

---

# 17. BackendDeploymentUnavailable

The third alert is:

```text
BackendDeploymentUnavailable
```

This alert monitors whether the backend Deployment has unavailable replicas.

The objective is to identify a situation in which the Kubernetes Deployment is not maintaining its expected available workload.

This provides another view of application availability in addition to the replica-count alert.

---

# 18. Alert Summary

The project's primary Grafana alerts are:

| Alert                          | Purpose                                         |
| ------------------------------ | ----------------------------------------------- |
| `BackendDeploymentReplicasLow` | Detects insufficient available backend replicas |
| `BackendPodRestartDetected`    | Detects backend pod restart activity            |
| `BackendDeploymentUnavailable` | Detects unavailable backend deployment replicas |

These alerts focus on practical Kubernetes availability and runtime-health conditions.

---

# 19. Alert Evaluation

The alert group uses:

```text
Evaluation Interval: 1 minute
```

The alert condition is evaluated repeatedly against the latest Prometheus metrics.

The general process is:

```text
Metric Collection
      |
      v
Prometheus
      |
      v
PromQL Evaluation
      |
      v
Condition Met?
    /     \
   No      Yes
   |        |
   v        v
Normal    Pending/
          Alerting
```

Alert evaluation frequency and pending periods are important because they determine how quickly an alert responds and how resistant it is to short-lived conditions.

---

# 20. Backend Scaling Validation

The monitoring configuration was tested using the backend Deployment.

The backend workload was scaled from:

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

This was used to validate the replica-availability monitoring logic.

The validation demonstrated that the monitoring system could observe changes in backend replica availability.

---

# 21. Validation Flow

The test can be represented as:

```text
Normal State
Backend = 2 replicas
       |
       v
Healthy

       ↓

Scale Backend
2 → 1 replica
       |
       v
Available Replicas < 2
       |
       v
Alert Condition

       ↓

Restore Backend
1 → 2 replicas
       |
       v
Healthy State Restored
```

This demonstrates that the alert logic is connected to actual Kubernetes workload state rather than being configured only as a static example.

---

# 22. Prometheus Query Model

Prometheus uses PromQL to query its time-series data.

A simplified example for the backend deployment is:

```text
kube_deployment_status_replicas_available{
  namespace="devsecops",
  deployment="backend"
}
```

The monitoring logic can then compare the returned value against the desired availability threshold.

For example:

```text
Available Replicas < 2
```

This allows the alert to respond to changes in the actual Kubernetes state.

---

# 23. Monitoring and Zero Trust

Monitoring is an important supporting layer of the Zero Trust architecture.

Zero Trust controls restrict access, but monitoring provides visibility into the resulting environment.

The relationship is:

```text
Zero Trust Controls
       |
       v
Restricted Environment
       |
       v
Continuous Monitoring
       |
       v
Detect Unexpected State
       |
       v
Investigate
```

Prometheus and Grafana therefore complement the preventive security controls.

---

# 24. Monitoring and Wazuh

Prometheus/Grafana and Wazuh serve different purposes.

### Prometheus + Grafana

Primarily focused on:

```text
Metrics
Availability
Performance
Workload Health
Infrastructure State
Alerting
```

### Wazuh

Primarily focused on:

```text
Security Events
Endpoint Monitoring
Log Analysis
Detection
Security Investigation
Event Archiving
```

The distinction can be summarized as:

```text
Prometheus / Grafana
        |
        v
"What is the system doing?"

Wazuh
        |
        v
"What security events are happening?"
```

This is why Grafana and Prometheus should not be described as the project's SIEM.

---

# 25. Monitoring vs SIEM

Prometheus is a monitoring and alerting toolkit that stores numerical measurements as time-series data.

Wazuh provides security-event collection, analysis, and investigation capabilities.

Therefore:

| Capability                   | Prometheus/Grafana |               Wazuh |
| ---------------------------- | -----------------: | ------------------: |
| Time-series metrics          |                Yes |                  No |
| Kubernetes availability      |                Yes |         Not primary |
| Pod restart monitoring       |                Yes |         Not primary |
| Infrastructure metrics       |                Yes | Limited/not primary |
| Security event analysis      |                 No |                 Yes |
| Endpoint security monitoring |                 No |                 Yes |
| Security event archiving     |                 No |                 Yes |
| SIEM functionality           |                 No |                 Yes |
| Security investigation       |            Limited |                 Yes |

This separation is important when describing the project in interviews.

---

# 26. Monitoring and Incident Investigation

Prometheus/Grafana can provide the initial operational signal.

For example:

```text
BackendPodRestartDetected
        |
        v
Backend instability observed
        |
        v
Investigate workload
        |
        +---- Kubernetes events
        +---- Pod logs
        +---- Deployment state
        +---- Resource usage
        +---- Wazuh security events
```

This creates a bridge between operational monitoring and security investigation.

---

# 27. Monitoring as Defense in Depth

The monitoring layer provides another security control after deployment.

The project therefore follows:

```text
Build
  |
  v
Scan
  |
  v
Harden
  |
  v
Deploy
  |
  v
Monitor
  |
  v
Detect
  |
  v
Investigate
```

Monitoring ensures that security does not end when a workload is deployed.

---

# 28. Monitoring Stack Components

The monitoring environment includes:

| Component           | Purpose                                              |
| ------------------- | ---------------------------------------------------- |
| Prometheus          | Metrics collection and time-series storage           |
| Grafana             | Visualization and alert management                   |
| Alertmanager        | Alert handling and notification infrastructure       |
| kube-state-metrics  | Kubernetes object-state metrics                      |
| Node Exporter       | Node-level metrics                                   |
| Prometheus Operator | Kubernetes-native management of Prometheus resources |

Together these components provide Kubernetes observability.

---

# 29. Prometheus Retention

The project uses:

```text
Prometheus retention: 24 hours
```

This is appropriate for the local development laboratory but is not intended to represent production retention requirements.

A production monitoring system would normally require retention based on:

* Operational requirements
* Compliance
* Capacity
* Incident-investigation requirements
* Cost
* Historical trend requirements

---

# 30. Grafana Persistence

Grafana persistence is disabled in the current laboratory configuration.

This means the environment is optimized for local experimentation rather than production durability.

The monitoring configuration itself remains reproducible through the project's configuration files.

A production deployment would normally use persistent storage for Grafana state where required.

---

# 31. Monitoring Limitations

The current monitoring implementation has several deliberate limitations:

* Local Kind cluster
* Single control-plane node
* Limited Prometheus retention
* Grafana persistence disabled
* No production notification integration
* No long-term metrics storage
* No multi-cluster monitoring
* No high-availability monitoring architecture

These limitations are appropriate for the scope of the local DevSecOps security laboratory.

---

# 32. Operational Security Value

The monitoring stack provides visibility into conditions that could affect security.

For example:

```text
Unexpected Pod Restart
        |
        v
Potential Application Failure
        |
        +---- Configuration Issue
        +---- Dependency Failure
        +---- Resource Exhaustion
        +---- Possible Exploitation
```

Monitoring therefore acts as an early-warning mechanism.

It does not determine the cause by itself.

Additional investigation is required to distinguish between operational failures and security incidents.

---

# 33. Monitoring Evidence

The implementation provides evidence through:

```text
Prometheus Metrics
       |
       v
Grafana Queries
       |
       v
Grafana Alert Rules
       |
       v
Backend Scaling Test
```

The backend replica test provides concrete validation that the monitoring layer responds to changes in Kubernetes workload state.

---

# 34. Monitoring Architecture in the Complete Platform

The complete security platform can be represented as:

```text
                         GitHub
                           |
                           v
                    GitHub Actions
                           |
                 ┌─────────┴─────────┐
                 |                   |
             Gitleaks             Checkov
                 |                   |
                 └─────────┬─────────┘
                           |
                           v
                    Kubernetes Kind
                           |
       ┌───────────────────┼───────────────────┐
       |                   |                   |
       v                   v                   v
   Frontend             Backend              MySQL
       |                   |                   |
       |                   +------ Vault ------+
       |                   |
       +---- NetworkPolicies / Kyverno --------+
                           |
                           v
                     Prometheus
                           |
                           v
                       Grafana
                           |
                           v
                    Alerting / Monitoring

Separate Runtime Security Environment
                           |
                           v
                     Windows Endpoint
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

# 35. Monitoring Lifecycle

The monitoring lifecycle is:

```text
Collect
  ↓
Store
  ↓
Query
  ↓
Visualize
  ↓
Evaluate
  ↓
Alert
  ↓
Investigate
```

Prometheus performs the collection, storage, and querying functions.

Grafana provides visualization and alert-management capabilities.

Alertmanager provides alert handling and notification functionality where configured.

---

# 36. Final Monitoring Assessment

The Prometheus and Grafana monitoring layer is successfully implemented for the project's local Kubernetes security laboratory.

The implementation provides:

```text
Kubernetes Metrics
       +
Workload Health
       +
Replica Monitoring
       +
Restart Detection
       +
Availability Alerting
       +
Grafana Visualization
```

The backend scaling test demonstrated that workload-state changes can be observed by the monitoring system and used as an alert condition.

The monitoring stack therefore provides the **observability and operational-alerting layer** of the DevSecOps Zero Trust platform.

It complements, rather than replaces, Wazuh's runtime security monitoring and SIEM capabilities.

---

# 37. Related Documentation

Additional implementation details are available in:

```text
docs/architecture.md
docs/security-controls.md
docs/zero-trust.md
docs/wazuh.md
docs/security-testing.md
docs/security-exceptions.md
docs/project-report.md
```
