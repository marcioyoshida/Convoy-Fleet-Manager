# How do multi-tenant cost allocation and chargebac

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional operating system, accounting for who consumed what compute resource is notoriously difficult, often relying on crude per-user CPU hour estimates or missing entirely. In a Kubernetes-backed shell architecture, multi-tenant cost allocation and chargeback tracking become precise, data-driven operations because every command, pipeline, and storage volume is explicitly metered and tagged at the infrastructure API layer.

Here is how cost allocation and chargeback tracking are handled across these shell pipelines:

## 1. Granular Metering: From Ephemeral Pods to User Identity

Because shell execution units are translated into discrete Kubernetes objects (Pods, Jobs, PVCs), resource consumption is never anonymous.

The Identity Link: Every shell session injects metadata labels into the compiled manifests—capturing the authenticated user ID, team/tenant tag, project code, and session timestamp (e.g., owner: alice@company.com, team: data-science, pipeline-id: pipe-8493).

Real-Time Metering: Open-source telemetry engines (such as Kubecost or OpenCost) continuously scrape Kubernetes metrics API and cloud provider billing APIs, attributing every CPU millicore, megabyte of RAM, network egress byte, and storage gigabyte to the exact user or team tag that spawned it.

## 2. Allocating Shared and Dynamic Pipeline Costs

Pipelines introduce accounting challenges that simple server allocation misses: shared resources, idle time, and intermediate streaming storage.

Shared Intermediate Storage: When a pipeline pipes data across stages via ephemeral volumes (emptyDir) or object storage, the cost of that temporary storage and network transfer is automatically calculated and attributed to the parent pipeline session.

Idle vs. Active Time: If a user allocates a dedicated workspace pod or a persistent spark cluster that sits idle, the chargeback engine distinguishes between active compute time (when processes are running) and idle reservation time, allowing finance teams to enforce strict timeout policies or bill accordingly.

Shared Cluster Overhead: System-level components (like the shell gateway, ingress controllers, and cluster management planes) are amortized across all active tenants using proportional usage models.

## 3. Automated Guardrails and Budget Enforcement

Chargeback shouldn't just be an unpleasant surprise at the end of the month; it must be tied directly to runtime governance.

Namespace Cost Budgets: Each user or tenant namespace has a financial ceiling defined alongside their ResourceQuota.

Proactive Throttling: If a tenant's pipeline usage approaches their daily or monthly budget cap, the shell platform can trigger automated alerts, require manager approval for heavy jobs, or temporarily restrict low-priority batch execution until the billing cycle resets.

## 4. Exporting to Enterprise Financial Systems

To operationalize chargeback within an enterprise, raw cluster telemetry is translated into standard financial outputs:

Showback vs. Chargeback: Teams can run "showback" reports to see their exact resource consumption without direct billing, or configure "chargeback" models that automatically generate invoice line items mapped to internal cost centers or cloud billing accounts (AWS Cost Allocation Tags, GCP Labels).
