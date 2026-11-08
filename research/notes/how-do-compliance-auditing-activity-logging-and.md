# How do compliance auditing, activity logging, and

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional POSIX environment, compliance auditing and activity tracking rely on local tools like history files, auditd, and /var/log/secure. These local mechanisms break down in a distributed cluster where workloads are ephemeral and containerized.

In this Kubernetes-backed shell architecture, auditing, activity logging, and security monitoring shift from local host files to centralized API audit streams, runtime eBPF telemetry, and immutable cloud storage.

## 1. Tracking User Commands (The API Audit Trail)

When a user types a command in this shell, it doesn't just write to a local .bash_history file that can be easily tampered with or deleted.

The Translation Layer as the Auditor: Because every command compiles into a Kubernetes API request (creating a Pod, Job, or PVC), the Kubernetes API Server's Audit Log acts as the primary system audit trail.

What is Captured: The API audit log records every transaction with cryptographic certainty:

Who: The authenticated user identity (e.g., alice@company.com via OIDC).

What: The exact high-level action and compiled manifest (e.g., "Created a Pod running grep error").

When & Where: Exact timestamps, source IP/session gateway, and target namespace.

## 2. Monitoring Secret Access (Safe Audit Logging)

Compliance frameworks (like SOC 2, HIPAA, or PCI-DSS) require strict tracking of who accesses sensitive configuration and encryption keys.

API-Level Access Tracking: Whenever a shell session mounts a Secret or injects an environment variable into a process, the Kubernetes API audit log records the read request against that specific secret object.

Automatic Redaction: To prevent compliance violations from leaking credentials into log management tools (like Datadog, Splunk, or Elastic), Kubernetes API audit policies automatically redact secret payload values. The logs record that a secret was accessed by a specific user or pod, but never expose the plaintext credential string in the log stream.

## 3. Runtime Security & Anomaly Detection (Replacing auditd)

Knowing what command a user started is only half the battle; compliance and security monitoring must track what the process actually does inside its isolated container namespace at runtime.

eBPF-Based Runtime Monitoring (e.g., Falco): Instead of running heavyweight agents inside every container, node-level eBPF (Extended Berkeley Packet Filter) probes intercept system calls (execve, open, connect) directly at the host kernel level.

Detecting Malicious Behavior: If a process spawned by a user's pipeline attempts unauthorized actions—such as reading /etc/shadow, modifying system binaries, or establishing an unexpected outbound network connection—the monitoring engine instantly flags a security violation, triggers an alert, and can automatically terminate the offending pod.

## 4. Centralized Compliance and Immutable Storage

To satisfy enterprise compliance mandates, ephemeral logs must be safely exported away from the cluster before pods or nodes disappear:

Log Forwarding: Tools like Fluentbit or Vector daemonsets stream API audit logs and container output out of the cluster in real-time.

Immutable Storage (WORM): Logs are landed in cloud object storage configured with Write-Once-Read-Many (WORM) policies (such as AWS S3 Object Lock), ensuring audit trails cannot be altered or deleted by malicious actors or compromised user sessions.
