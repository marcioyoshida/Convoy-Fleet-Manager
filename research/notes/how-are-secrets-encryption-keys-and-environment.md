# How are secrets, encryption keys, and environment

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional POSIX shell, managing secrets and configuration variables is often loose and vulnerable: files like .env sit insecurely on disk, permissions are controlled by file bits (chmod 600), and exporting sensitive values in bash history exposes them to anyone with shell access.

In a Kubernetes-backed shell architecture, managing environment variables, secrets, and encryption keys must shift from local filesystem handling to centralized, policy-driven cloud-native secret engines and API-injected runtime memory.

Here is how secrets and configurations are securely handled across this architecture:

## 1. Environment Variables (export VAR=value → ConfigMaps)

For non-sensitive configuration data (e.g., feature flags, log levels, non-secret endpoints), the POSIX export command maps directly to Kubernetes ConfigMaps.

How it works: When a user types export APP_ENV=production in their shell session, the session controller captures the variable. When a pipeline step or process is compiled and launched, the shell injects these variables into the pod specification under envFrom or env pointing to a session-scoped ConfigMap.

Immutability: Just like local environment variables, these values persist for the duration of the session or process scope without ever writing plaintext configuration files to disk.

## 2. Secrets Management (.env → Kubernetes Secrets & KMS)

By default, standard Kubernetes Secrets are merely base64-encoded strings stored in etcd, which provides zero true encryption at rest unless explicitly configured. A production-grade shell architecture must enforce stringent security standards:

Encryption at Rest: The underlying Kubernetes cluster must have etcd encryption enabled using a cloud provider's Key Management Service (KMS) provider (such as AWS KMS, GCP KMS, or HashiCorp Vault).

Zero-Disk Policy: When a user passes a secret to a process (e.g., passing a database password via shell arguments or secure pipes), the shell injects it directly into the container's environment or mounts it as an in-memory tmpfs volume (emptyDir with medium: Memory). The secret never touches a physical block device or persistent volume.

## 3. Enterprise Integration (Vault & External Secrets Operators)

Hardcoding secrets inside cluster manifests or shell scripts is an anti-pattern. Enterprise applications require dynamic, short-lived credentials (like database credentials that rotate every hour).

External Secrets Integration: The shell acts as a secure translator that binds enterprise secret stores (like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault) to the user's active session.

How it works: When a command references a secret reference (e.g., db-password@vault), the shell compiler uses an External Secrets Operator to fetch the credential dynamically and securely inject it into the runtime process memory at launch time, avoiding storage in etcd altogether.

## 4. Session Scoping & Access Control

In a multi-tenant shell environment, preventing users from reading each other's secrets is paramount:

| Traditional POSIX Security | Kubernetes-Backed Shell Equivalent | How It's Enforced |

| chmod 600 .env  (File ownership) | Namespaced RBAC & Secret Policies | Secrets are locked to specific Kubernetes namespaces; user A's session cannot query user B's secret store. |

| Reading /proc/*/environ  (Process inspection) | Isolated Container Namespaces | Containers cannot inspect neighbor processes or their environment variables due to strict container boundary isolation. |

| Bash History Leakage | Ephemeral Command Sanitization | Sensitive commands containing secrets are automatically scrubbed or masked in session audit logs. |
