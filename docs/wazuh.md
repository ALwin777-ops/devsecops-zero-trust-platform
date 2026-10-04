# Wazuh Runtime Security and SIEM Integration

## 1. Purpose

This document describes the Wazuh runtime security and SIEM integration implemented as part of the DevSecOps Zero Trust Platform.

Wazuh provides the runtime detection and security-monitoring layer that complements the preventive controls implemented across Docker, Kubernetes, Vault, Kyverno, NetworkPolicies, Checkov, Gitleaks, and Trivy.

The Wazuh implementation was designed to demonstrate:

* Endpoint monitoring
* Centralized security event collection
* Event analysis
* Security event archiving
* Log forwarding
* Indexed event storage
* Security-event investigation
* Dashboard-based visibility

The implementation uses a Windows endpoint as the monitored system and a separate Ubuntu WSL environment for the Wazuh central components.

---

# 2. Role of Wazuh in the Project

The overall platform follows a defense-in-depth model.

```text
Source Security
      |
      v
Gitleaks / Checkov
      |
      v
Container Security
      |
      v
Trivy / Docker Hardening
      |
      v
Kubernetes Security
      |
      v
NetworkPolicies / Kyverno
      |
      v
Secrets Security
      |
      v
Vault
      |
      v
Runtime Monitoring
      |
      v
Prometheus / Grafana / Wazuh
```

Wazuh therefore serves primarily as the **runtime security and security-event monitoring layer**.

It does not replace the preventive Kubernetes controls.

Instead, it complements them by providing visibility into security events after workloads and endpoints are running.

---

# 3. Wazuh Architecture

The implemented Wazuh environment follows this logical architecture:

```text
┌───────────────────────────────┐
│       Windows Endpoint        │
│                               │
│       ALWIN-WINDOWS           │
│                               │
│        Wazuh Agent            │
└───────────────┬───────────────┘
                │
                │ Security Events
                ▼
┌───────────────────────────────┐
│       Wazuh Manager/Server    │
│                               │
│       Event Analysis          │
│       Decoders / Rules        │
└───────────────┬───────────────┘
                │
                │ Event Data
                ▼
┌───────────────────────────────┐
│           Filebeat            │
│                               │
│       Log Forwarding          │
└───────────────┬───────────────┘
                │
                │ Indexed Events
                ▼
┌───────────────────────────────┐
│        Wazuh Indexer          │
│                               │
│       Event Storage           │
└───────────────┬───────────────┘
                │
                │ Search / Query
                ▼
┌───────────────────────────────┐
│       Wazuh Dashboard         │
│                               │
│     Investigation / View      │
└───────────────────────────────┘
```

This is consistent with the Wazuh architecture in which agents collect endpoint data, the Wazuh server analyzes events, Filebeat forwards data to the indexer, and the dashboard provides visualization and investigation capabilities.

---

# 4. Deployment Environment

The Wazuh environment was deployed separately from the Kubernetes Kind cluster.

The Wazuh environment runs inside:

```text
Ubuntu 24.04.5 LTS
WSL
```

The WSL distribution is stored on:

```text
D:\WSL\Wazuh-Ubuntu24
```

The central Wazuh components are deployed in an all-in-one configuration.

The deployment contains:

```text
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard
Filebeat
```

This type of all-in-one deployment is appropriate for a laboratory environment and small-scale testing. Wazuh documentation describes all-in-one deployments as suitable for labs and environments with limited monitored endpoints.

---

# 5. Wazuh Version

The implemented environment uses:

```text
Wazuh 4.14.8
```

The central Wazuh components use the same version.

Filebeat is used as the forwarding component between the Wazuh server and Wazuh indexer.

Wazuh documentation specifies compatibility between Wazuh Indexer 4.14.8 and Filebeat OSS 7.10.2.

---

# 6. Windows Endpoint

A Windows endpoint was enrolled into the Wazuh environment.

The logical Wazuh agent name is:

```text
ALWIN-WINDOWS
```

The endpoint acts as the monitored system in the lab.

The Wazuh Agent collects configured security and system events and forwards them to the Wazuh server.

Wazuh agents support Windows and other major operating systems and are designed to collect system and application security data for forwarding to the Wazuh server.

---

# 7. Agent-to-Manager Flow

The Windows endpoint follows this flow:

```text
Windows Event
      |
      v
Wazuh Agent
      |
      v
Wazuh Manager
```

The agent establishes an authenticated connection with the Wazuh server and forwards collected events.

The Wazuh server then processes the received events using its analysis engine.

The Wazuh architecture uses an agent connection service for agent communication and validates agent identity using enrollment credentials.

---

# 8. Wazuh Server / Manager

The Wazuh Manager is the central analysis component.

Its responsibilities include:

* Receiving endpoint events
* Decoding event data
* Applying detection rules
* Generating alerts
* Maintaining agent information
* Providing the Wazuh API
* Forwarding security data through Filebeat

The Wazuh server contains the analysis engine, API, agent enrollment and connection services, and Filebeat integration.

---

# 9. Event Analysis

The Wazuh analysis engine processes received events.

The logical processing flow is:

```text
Raw Event
    |
    v
Decoder
    |
    v
Structured Event
    |
    v
Detection Rules
    |
    v
Alert / Event
```

Decoders extract relevant fields from incoming events.

Rules then determine whether an event matches a known security condition.

Wazuh documents this process as real-time analysis using decoders and rules.

---

# 10. Wazuh Archives

An important part of this project was enabling Wazuh event archiving.

The archive configuration allows events received by the Wazuh server to be stored even when they do not trigger a detection alert.

The primary JSON archive file is:

```text
/var/ossec/logs/archives/archives.json
```

Wazuh documents that `archives.json` stores received events in JSON format and can contain events regardless of whether they trigger alerts.

This is useful for:

* Historical investigation
* Threat hunting
* Event reconstruction
* Troubleshooting
* Security analysis
* Evidence collection

---

# 11. Archive Configuration

The Wazuh manager was configured to enable JSON event archiving.

The relevant configuration concept is:

```xml
<logall_json>yes</logall_json>
```

This causes received events to be written to:

```text
/var/ossec/logs/archives/archives.json
```

Wazuh documentation specifies that `logall_json` enables JSON event archiving and is required when archived events are to be indexed for dashboard visualization.

---

# 12. Why Archive Data Was Enabled

Normal alert data only represents events that match configured detection conditions.

Archive data provides a broader event history.

The distinction can be represented as:

```text
All Received Events
        |
        +--------------------+
        |                    |
        v                    v
   Archive Data          Detection Rules
        |                    |
        v                    v
archives.json             Alerts
```

Therefore:

```text
Alert
≠
All Events
```

Archive data allows events that did not generate an alert to remain available for investigation.

Wazuh specifically documents the `wazuh-archives-*` indices for storing and querying these archived events.

---

# 13. Filebeat Integration

Filebeat provides the forwarding layer between the Wazuh server and Wazuh indexer.

The logical flow is:

```text
Wazuh Manager
      |
      v
Filebeat
      |
      v
Wazuh Indexer
```

The Wazuh server uses Filebeat to send event and alert data to the Wazuh indexer.

---

# 14. Archive Forwarding

Filebeat was configured with archive forwarding enabled.

The relevant configuration concept is:

```yaml
archives:
  enabled: true
```

This allows archived events to be forwarded to the Wazuh indexer.

The resulting flow is:

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
```

Wazuh's current documentation describes this configuration for indexing archived events.

---

# 15. Filebeat Output Validation

Filebeat connectivity was validated using:

```bash
sudo filebeat test output
```

The output confirmed successful communication with the configured indexer endpoint.

This established that the forwarding layer was able to communicate with the Wazuh indexer.

---

# 16. Wazuh Indexer

The Wazuh Indexer stores security data as searchable JSON documents.

It provides the storage and search layer for the Wazuh platform.

The logical flow is:

```text
Filebeat
    |
    v
Wazuh Indexer
    |
    v
Indexed Security Events
```

The Wazuh Indexer is designed as the search and analytics engine for Wazuh security data.

---

# 17. Archive Index Pattern

The archive data was exposed through the following index pattern:

```text
wazuh-archives-*
```

The dashboard uses this pattern to locate archived event indices.

Wazuh documentation specifies `wazuh-archives-*` as the index pattern for archived events and recommends using the `timestamp` field as the time field.

---

# 18. Dashboard Investigation

The Wazuh Dashboard provides the investigation interface.

Archived events can be viewed through:

```text
Explore
    |
    v
Discover
    |
    v
wazuh-archives-*
```

This allows the analyst to search and inspect raw archived security events.

The Wazuh Dashboard is designed for security-event visualization, investigation, and platform management.

---

# 19. End-to-End Validation

The Wazuh pipeline was validated end-to-end.

A test Windows event was generated on the monitored endpoint.

The event used:

```text
Event ID: 998
Source: WazuhTest
Log: APPLICATION
```

The test message was:

```text
Wazuh archive indexed test
```

The event was then traced through the complete pipeline.

---

# 20. Validation Pipeline

The validated flow was:

```text
Windows Endpoint
       |
       | Windows Event
       v
Wazuh Agent
       |
       | Agent Communication
       v
Wazuh Manager
       |
       | Event Processing
       v
archives.json
       |
       | Filebeat
       v
Wazuh Indexer
       |
       | wazuh-archives-*
       v
Wazuh Dashboard
       |
       v
Discover
```

The test event was successfully located in the `wazuh-archives-*` index pattern.

This demonstrated that the complete event pipeline was operational.

---

# 21. Evidence of Successful Integration

The successful validation established:

```text
Windows Event Collection
        ✓
Wazuh Agent Enrollment
        ✓
Agent-to-Manager Communication
        ✓
Wazuh Event Processing
        ✓
JSON Event Archiving
        ✓
Filebeat Archive Forwarding
        ✓
Indexer Storage
        ✓
Dashboard Discovery
        ✓
```

This provides evidence that the Wazuh implementation is not merely installed but operational.

---

# 22. Runtime Security Role

Wazuh provides the runtime detection layer of the project.

The distinction between the major security technologies is:

| Technology       | Primary Role                                   |
| ---------------- | ---------------------------------------------- |
| Gitleaks         | Secret detection                               |
| Checkov          | IaC / configuration security                   |
| Trivy            | Vulnerability and security scanning            |
| Docker hardening | Container runtime hardening                    |
| NetworkPolicy    | Network segmentation                           |
| Vault            | Secret management                              |
| Kyverno          | Kubernetes policy validation                   |
| Prometheus       | Metrics collection                             |
| Grafana          | Monitoring and alerting                        |
| Wazuh            | Runtime security monitoring and event analysis |

This separation prevents Wazuh from being treated as a replacement for the other security controls.

---

# 23. Wazuh and Zero Trust

Wazuh complements the Zero Trust architecture by providing continuous security visibility.

The relationship can be represented as:

```text
Zero Trust Preventive Controls
        |
        v
Restrict Access
        |
        v
Least Privilege
        |
        v
Network Segmentation
        |
        v
Runtime Environment
        |
        v
Wazuh Monitoring
        |
        v
Detect Security Events
        |
        v
Investigate
```

Zero Trust therefore does not end after access is granted.

Runtime activity remains observable.

---

# 24. Wazuh and Defense in Depth

The platform uses Wazuh as one layer in a broader defense-in-depth strategy.

```text
Layer 1
Source Security
        |
Layer 2
CI/CD Security
        |
Layer 3
Container Security
        |
Layer 4
Kubernetes Security
        |
Layer 5
Network Segmentation
        |
Layer 6
Secrets Management
        |
Layer 7
Policy Validation
        |
Layer 8
Monitoring
        |
Layer 9
Runtime Security
```

Wazuh operates primarily at the final runtime-security layer while consuming security telemetry from monitored endpoints.

---

# 25. Resource Considerations

The Wazuh environment is intentionally separated from the Kubernetes Kind cluster.

This was necessary because the complete local security laboratory contains several resource-intensive components.

The environment includes:

```text
Kubernetes Kind
Vault
Kyverno
Prometheus
Grafana
Wazuh Manager
Wazuh Indexer
Wazuh Dashboard
Filebeat
```

Running all components simultaneously can create significant CPU and memory pressure on a development workstation.

Therefore, Wazuh services can be stopped when the Kubernetes environment is being used heavily.

This does not affect the persistent project configuration.

---

# 26. Lab Architecture

The complete local security laboratory can be viewed as two environments:

```text
                  Local Development Machine
                           |
             ┌─────────────┴─────────────┐
             |                           |
             v                           v
     Kubernetes Environment       Wazuh Environment
             |                           |
       Kind Cluster                 Ubuntu WSL
             |                           |
       ┌─────┴─────┐              ┌─────┴─────┐
       |           |              |           |
   Application  Security       Wazuh      Wazuh
   Workloads    Controls       Manager     Indexer
       |           |              |           |
       |           |              +-----+-----+
       |           |                    |
       |           |                 Dashboard
       |           |
       +-----------+
```

The separation demonstrates that runtime security monitoring can exist alongside the Kubernetes security platform without being embedded into the application workload.

---

# 27. Security Event Lifecycle

The complete event lifecycle is:

```text
Event Generated
      |
      v
Event Collected
      |
      v
Event Transported
      |
      v
Event Analyzed
      |
      v
Event Archived
      |
      v
Event Indexed
      |
      v
Event Queried
      |
      v
Security Investigation
```

This provides an auditable path from event generation to analyst visibility.

---

# 28. Security Investigation Workflow

A basic investigation workflow is:

```text
1. Security event occurs
          |
          v
2. Wazuh Agent collects event
          |
          v
3. Wazuh Manager receives event
          |
          v
4. Event is analyzed
          |
          v
5. Event is archived/indexed
          |
          v
6. Analyst searches Dashboard
          |
          v
7. Event context is reviewed
```

This workflow can be extended in a production environment with incident-management integrations and automated response processes.

---

# 29. Current Scope

The Wazuh implementation in this project intentionally focuses on:

* Endpoint enrollment
* Event collection
* Centralized analysis
* Event archiving
* Filebeat forwarding
* Indexing
* Dashboard investigation
* End-to-end validation

The project does not attempt to implement a complete enterprise SOC.

---

# 30. Deliberate Scope Limitations

The current implementation is a local security laboratory.

It does not implement:

* Multi-node Wazuh server clustering
* Multi-node Wazuh indexer clustering
* Production high availability
* Enterprise-scale endpoint fleet management
* Production incident-response orchestration
* Cloud-hosted Wazuh
* Full enterprise SOC integration
* Autonomous AI-based incident response

These are intentionally outside the current project scope.

---

# 31. AI Analyst Consideration

An AI-based analyst layer is not required for the core project.

The current Wazuh implementation already demonstrates:

```text
Collection
Analysis
Archiving
Indexing
Investigation
```

If an AI-assisted analysis layer were introduced in the future, it should be treated as an optional analyst-assistance capability rather than a replacement for Wazuh detection.

A safe conceptual model would be:

```text
Wazuh Event
     |
     v
AI-Assisted Analysis
     |
     +---- Summary
     +---- Context
     +---- Possible MITRE Mapping
     +---- Suggested Investigation
     |
     v
Human Analyst
```

The current project does not require this extension.

---

# 32. Security Value

The Wazuh integration provides several security benefits:

### Centralized Visibility

Security events from the Windows endpoint can be viewed centrally.

### Historical Investigation

Archived events can be searched after the original event occurred.

### Runtime Monitoring

The endpoint can be monitored after deployment.

### Event Correlation

Events can be analyzed through Wazuh's analysis engine.

### Searchable Security Data

Indexed events can be investigated through the dashboard.

### Evidence Collection

Archived events provide supporting evidence for security investigations.

---

# 33. Validation Commands

The following commands were used during the implementation and validation process.

### Check Wazuh service status

```bash
sudo systemctl is-active wazuh-dashboard filebeat wazuh-manager wazuh-indexer
```

### Test Filebeat output

```bash
sudo filebeat test output
```

### Inspect Wazuh archives

```bash
sudo ls -lh /var/ossec/logs/archives/
```

### Inspect archived JSON events

```bash
sudo tail -n 20 /var/ossec/logs/archives/archives.json
```

The commands above should be executed only when the Wazuh environment is running.

---

# 34. Important Operational Consideration

Wazuh archives can grow significantly because they retain events beyond only those that trigger alerts.

Wazuh documentation warns that enabling archive logging increases storage requirements because all received events may be retained.

Therefore, archive retention should be managed appropriately in production environments.

For this project, archive indexing is primarily intended to demonstrate:

```text
Event Collection
+
Event Retention
+
Event Search
+
Security Investigation
```

---

# 35. Wazuh Security Pipeline Summary

The completed Wazuh pipeline is:

```text
                         SECURITY EVENT
                              |
                              v
                    ┌───────────────────┐
                    │ Windows Endpoint  │
                    └─────────┬─────────┘
                              |
                              v
                    ┌───────────────────┐
                    │   Wazuh Agent     │
                    └─────────┬─────────┘
                              |
                              v
                    ┌───────────────────┐
                    │  Wazuh Manager    │
                    │ Analysis Engine   │
                    └─────────┬─────────┘
                              |
                    ┌─────────┴─────────┐
                    |                   |
                    v                   v
             Detection Alerts      archives.json
                                        |
                                        v
                                  ┌───────────┐
                                  │ Filebeat  │
                                  └─────┬─────┘
                                        |
                                        v
                                  ┌───────────┐
                                  │  Indexer  │
                                  └─────┬─────┘
                                        |
                                        v
                                  ┌───────────┐
                                  │ Dashboard │
                                  └─────┬─────┘
                                        |
                                        v
                                  Investigation
```

---

# 36. Final Assessment

The Wazuh milestone is considered successfully implemented for the project's laboratory scope.

The most important validation was the successful end-to-end demonstration:

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
wazuh-archives-*
      ↓
Wazuh Dashboard
      ↓
Discover
```

This demonstrates that the platform can collect, process, archive, index, and investigate runtime security events.

Wazuh therefore provides the runtime detection and security-visibility layer of the broader DevSecOps Zero Trust platform.

---

# 37. Related Documentation

Additional implementation details are available in:

```text
docs/architecture.md
docs/security-controls.md
docs/zero-trust.md
docs/monitoring.md
docs/security-testing.md
docs/security-exceptions.md
docs/project-report.md
```

The Wazuh documentation used as technical reference includes the official architecture, server, dashboard, archive, and indexer documentation.
