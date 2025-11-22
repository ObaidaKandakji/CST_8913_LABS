### 1. Company overview

CloudMed Solutions Inc. is a healthcare SaaS provider. Its main product, **MedConnect**, offers:

- Secure telehealth sessions  
- EMR management  
- AI-driven health analytics  

It serves hospitals and clinics in **Canada, the US, and Europe**, with workloads hosted in **Canada Central** and **West Europe** Azure regions. The overall design is based on Microsoft’s Azure Cloud Adoption Framework (CAF) landing zone and Zero Trust guidance.

---

### 1.2 Why Zero Trust in Azure

CloudMed handles highly sensitive PHI/PII through internet-facing services used from many locations and device types. A Zero Trust approach in Azure helps to:

- Protect EMR and telehealth data from breaches and ransomware  
- Secure remote access from untrusted networks and mixed device types  
- Limit the blast radius if an account, device, or integration is compromised  
- Support multi-region, multi-tenant operations with strong isolation  

---

### 1.3 Key compliance and operational drivers

**Compliance (HIPAA, GDPR, PIPEDA):**

- Strict access control and least privilege for PHI/PII  
- Encryption at rest and in transit  
- Detailed logging, monitoring, and auditability  
- Regional data residency and controlled cross-border transfers  

**Operational:**

- Centralized governance and guardrails across subscriptions and regions  
- Consistent policies for security, tagging, and cost management  
- Scalable, resilient multi-region architecture for MedConnect  
- Secure DevOps with just-in-time, role-based access and strong identity for users and automation  
- Unified monitoring and incident response across identity, network, and workloads  

---

### 2. Governance and identity

#### 2.1 Management group hierarchy

`Tenant root`  
└── **CloudMed**  
&emsp;├── **Platform (platform landing zone)**  
&emsp;│&emsp;├── Sub: cm-mgmt-shared-services  
&emsp;│&emsp;└── Sub: cm-security-monitoring  
&emsp;├── **Production (application landing zones)**  
&emsp;│&emsp;├── Sub: cm-medconnect-prod-ca-central  
&emsp;│&emsp;└── Sub: cm-medconnect-prod-weu  
&emsp;└── **Development (application landing zones)**  
&emsp;&emsp;├── Sub: cm-medconnect-dev-test  
&emsp;&emsp;└── Sub: cm-sandboxes  

- **CloudMed**: Organization-wide governance scope (baseline policies, RBAC model, global initiatives).  
- **Platform**: Platform landing zone with shared services such as identity, connectivity, security, monitoring, and management.  
- **Production**: Application landing zones for all production MedConnect workloads, separated by region.  
- **Development**: Application landing zones for non-production workloads (dev, test, QA, sandboxes), kept separate from production.

This hierarchy is consistent with CAF landing zone guidance and lets CloudMed apply global controls at the **CloudMed** level, platform guardrails at **Platform**, and workload-specific controls at **Production** and **Development**.

---

#### 2.2 Governance model – RBAC

RBAC is applied at management group, subscription, and resource group levels. Access is assigned to groups instead of individual users, and roles are scoped as narrowly as possible. This follows the CAF enterprise access model and subscription democratization: the platform team sets up and protects the environment, and application teams get controlled access inside it.

- **Admins (Cloud / Platform / Security admins)**  
  - Scope: CloudMed, Platform management group, and management/platform subscriptions.  
  - Typical roles:
    - `Owner` or `User Access Administrator` only for a small break-glass group.  
    - `Contributor` or a custom “Platform Admin” role for normal operations.  
  - Responsibilities:
    - Manage subscriptions, policies, networking, and shared services.  
    - Onboard new landing zones and applications.

- **DevOps / Application teams**  
  - Scope: Specific **Production** and **Development** subscriptions or resource groups for their applications.  
  - Typical roles:
    - `Contributor` in their own resource groups or subscriptions.  
    - `Reader` on shared Platform resources they need (for example, shared vNETs or key vaults).  
  - Responsibilities:
    - Deploy and manage application resources through CI/CD.  
    - No permission to change tenant-wide policies or billing.

- **Finance / Cost management**  
  - Scope: CloudMed and all child management groups and subscriptions.  
  - Typical roles:
    - `Cost Management Reader` and `Reader`.  
  - Responsibilities:
    - View consumption, budgets, and cost reports.  
    - No permission to create or modify resources.

Where needed, **custom roles** are used to lock down sensitive actions even more. **Privileged Identity Management (PIM)** is used for just-in-time elevation to privileged roles, so high-level permissions are temporary and audited. Identity roles in Entra ID are kept separate from Azure RBAC roles to distinguish tenant-level control from resource-level access.

---

#### 2.3 Governance model – Azure Policy

Azure Policy keeps the landing zones consistent and compliant across governance, security, and management:

- **Tagging policies**  
  - Require tags such as:
    - `Environment` (Prod / Dev / Test)  
    - `CostCenter`  
    - `Owner`  
    - `DataClassification` (for example, PHI, Confidential, Internal)  
    - `Application` (for example, MedConnect)  
  - Non-compliant resources are denied or remediated.

- **Allowed regions**  
  - At the **CloudMed** or **Production / Development** management group level:
    - Only `Canada Central` and `West Europe` are allowed for PHI workloads.  
    - Other regions can be allowed only for non-PHI workloads such as sandboxes.

- **Resource consistency and security baselines**  
  - Enforce:
    - Encryption at rest for storage, SQL, and disks.  
    - Required diagnostic settings (logging to Log Analytics or a SIEM).  
    - Only approved SKUs and resource types (for example, no public IPs on the data tier).  
    - Secure defaults for networking (NSGs, private endpoints, firewall configurations).  
  - **Policy initiatives** (for example, “CloudMed-Prod-Security-Baseline”) are assigned at the Production management group so all workloads inherit the same Zero Trust guardrails and baseline controls.

---

#### 2.4 Identity and access – Azure Entra ID & Conditional Access

Azure Entra ID is used as the central identity provider and control plane for:

- CloudMed staff (admins, DevOps, finance).  
- Service principals and managed identities for automation and workloads.  
- B2B identities for partners, with very limited and scoped access.

Key identity and Conditional Access practices:

- **Group-based access**  
  - All RBAC assignments go to Entra ID groups (for example, `cm-platform-admins`, `cm-devops-medconnect`, `cm-finance-readers`).  
  - Groups are mapped to roles and scopes in Azure.

- **Conditional Access policies**  
  - Require **MFA** for all admin, DevOps, and finance roles.  
  - Require **compliant or hybrid-joined devices** for privileged access.  
  - Block or challenge sign-ins from high-risk locations or legacy protocols.  
  - Apply stricter rules for high-privilege roles (for example, only from trusted networks or secure virtual desktops).

- **Privileged Identity Management (PIM)**  
  - Just-in-time elevation for high-privilege directory roles (for example, Global Administrator, Security Administrator) and Azure roles (for example, Owner, Contributor on critical scopes).  
  - Approvals, MFA, and time limits are used so elevated access is temporary and logged.

- **Zero Trust alignment**  
  - Identity becomes the main control point: every admin and workload identity is authenticated and authorized based on risk and device posture.  
  - Least privilege, just-in-time access, and Conditional Access together support “verify explicitly” and “assume breach” for the landing zone.

---

### 3. Network architecture

#### 3.1 Hub-and-spoke and Zero Trust

CloudMed uses a **hub-and-spoke** vNet design in each region (Canada Central, West Europe), following Microsoft’s recommended network topology:

- **Hub vNet** (in a connectivity or platform subscription): hosts shared connectivity and security services only.  
- **Spoke vNets** (application landing zones):
  - **App spoke** – web and mobile front end.
  - **API spoke** – backend APIs and integration services.
  - **Data spoke** – databases and storage, accessed via private endpoints.

Spokes are only peered with the hub, not with each other. All inter-spoke and internet-bound traffic is routed through the hub so it can be inspected and logged. This supports Zero Trust by:

- Removing implicit trust between tiers and spokes.  
- Reducing public exposure by using private IPs, private endpoints, and WAF at the edge.  
- Requiring explicit allow rules for all flows and using micro-segmentation inside each vNet.

---

#### 3.2 Hub components

Each **hub vNet** includes:

- **Azure Firewall**  
  - Single egress and east–west inspection point.  
  - Controls outbound and inter-spoke traffic through network and application rules and threat intelligence.

- **Azure Bastion**  
  - Provides secure RDP/SSH to VMs without public IPs.  
  - Works with Entra ID, RBAC, and just-in-time access for administrators.

- **DNS (Private DNS and optional DNS proxy)**  
  - Private DNS zones for internal services and private endpoints (SQL, Storage, Key Vault).  
  - Central name resolution and DNS logging.

- **Log Analytics / SIEM**  
  - A central workspace that collects logs from Firewall, NSGs, Bastion, and other platform components.  
  - Used for monitoring, threat detection, and compliance reporting.

Overall, the hub acts as a shared connectivity and security layer for all spokes.

---

#### 3.3 Spoke isolation: App, API, Data

Each workload tier uses its own **spoke vNet**, which is further split into subnets with NSGs:

- **App spoke**
  - Subnet for web and mobile front-end workloads.
  - Receives inbound traffic only from WAF/App Gateway or Front Door.
  - Sends outbound traffic through the Firewall and can only reach the allowed API endpoints.

- **API spoke**
  - Subnets for backend APIs and integration components.
  - Accepts traffic only from the App spoke and specific platform components.
  - No direct internet ingress; outbound traffic goes through the Firewall.

- **Data spoke**
  - Subnets for private endpoints for Azure SQL, Storage, Key Vault, and possibly data processing workloads.
  - Only API subnets and required platform services can reach these private endpoints.
  - The App spoke cannot reach the data tier directly.

This enforces the path **App -> API -> Data**, blocks direct App -> Data communication, and limits lateral movement between workloads.

---

#### 3.4 East–west control, private endpoints, subnet separation

- **East–west traffic**
  - Spokes peer only with the hub vNet, never directly with each other.
  - User-defined routes (UDRs) in the spokes send inter-spoke and internet traffic to **Azure Firewall**.
  - The Firewall explicitly allows required flows (such as App -> API, API -> Data) and denies other traffic.

- **Private endpoints**
  - Databases and storage are reachable only via **Private Endpoints** in dedicated subnets.
  - Private DNS zones map PaaS FQDNs to private IPs.
  - NSGs and Firewall rules are used to control which subnets can use these endpoints, combined with identity-based controls.

- **Subnet separation**
  - Each vNet is split into subnets based on function (for example, frontend, backend, private endpoints, management).  
  - NSGs on each subnet implement least-privilege rules and block unnecessary lateral movement.  
  - This creates clear boundaries between management, app, API, and data paths, which supports the “assume breach” mindset.

---

### 4. Zero Trust controls

These controls apply Zero Trust principles across identity, devices, network, data, applications, and monitoring.

#### 4.1 Verify explicitly

Access is always checked based on identity, device, and context:

- **Azure Entra ID** is the central identity provider for admins, DevOps, and automation.  
- **MFA** is required for all privileged roles through Conditional Access.  
- **Conditional Access** evaluates sign-in risk, device compliance, and location before allowing access to the Azure portal, Bastion, or management endpoints.  
- **Managed identities** are used so applications and APIs authenticate to services using Entra ID instead of static secrets.

---

#### 4.2 Least privilege access

Permissions are minimized and time-bound wherever possible:

- **RBAC via groups**, not individual users, with carefully scoped roles:
  - Platform admins at the Platform management group and platform subscriptions.
  - DevOps roles limited to their own application resource groups or subscriptions.
  - Finance roles limited to read-only cost views.
- **No standing “Owner” access** except for break-glass accounts.  
- **Azure AD PIM** is used for on-demand elevation to privileged roles (for example, Owner, Contributor, Security Admin), with approvals, MFA, and time-limited access.  
- **Custom roles** are created where built-in roles are too broad.

---

#### 4.3 Assume breach

The design assumes that a breach can happen and focuses on limiting impact:

- **Network segmentation** using hub-and-spoke, separate App/API/Data spokes, and strict NSG and Firewall rules.  
- **Private endpoints and no public data tier** so Azure SQL, Storage, and Key Vault are only reachable over Private Link.  
- **Encryption everywhere**:
  - Data at rest in SQL, Storage, managed disks, and backups.
  - Data in transit protected with TLS for all application and API endpoints.
- **Central logging and monitoring**:
  - Firewall, NSG, Bastion, and other platform logs are sent to Log Analytics or a SIEM.
  - Alerts are configured for unusual authentication events, suspicious east–west traffic, and policy violations.
- **Policy guardrails**:
  - Azure Policy blocks or flags non-compliant configurations (for example, public IPs where not allowed, resources in the wrong region, or missing tags).

---

#### 4.4 Design examples

1. **Bastion for admin access**  
   - Admin RDP/SSH to VMs is only done through Azure Bastion, so VMs do not need public IPs.  
   - Access goes through Entra ID, MFA, RBAC, and optionally PIM, which supports verify explicitly, least privilege, and assume breach.

2. **Private Link for SQL and Storage**  
   - Azure SQL and Storage accounts are exposed only via Private Endpoints inside the Data spoke.  
   - NSGs and the Firewall control which subnets can reach these endpoints, which helps limit damage if something is compromised.

3. **Policy to block public IPs on data and core workloads**  
   - Azure Policy denies creation of public IPs on data-tier resources and key platform services.  
   - This lowers the external attack surface and enforces secure defaults.

4. **Just-in-time admin roles with PIM**  
   - Cloud and security admins elevate to higher roles only when needed, and only for a short time.  
   - This reduces standing privileges and improves auditing.

5. **Firewall-enforced east–west flows**  
   - UDRs send App -> API -> Data traffic through Azure Firewall, which uses allow-lists for the expected flows.  
   - Unexpected lateral traffic is blocked and logged for investigation.

---

### 5. Monitoring, compliance, and cost

#### 5.1 Monitoring and security

- **Azure Monitor and a central Log Analytics workspace** in the platform/management subscription per region.  
  - Collects platform logs (Activity Logs), resource logs (Firewall, NSG, App Services, SQL, Storage, Key Vault), and metrics through diagnostic settings.  
- **Defender for Cloud** enabled on all subscriptions:  
  - Provides secure score, baseline hardening, and regulatory compliance views (for example, HIPAA/HITRUST and GDPR).  
  - Detects threats across VMs, PaaS, SQL, Storage, and network flows.  
- **Azure Monitor alerts and dashboards**:  
  - Alerts are set up for availability, performance issues, and security events (for example, many firewall denies, repeated failed logins, PIM activations).  
  - Workbooks show operational, security, and SLA information.

Entra ID sign-in and audit logs are also connected to the central workspace so identity-related events can be correlated with network and workload logs.

---

#### 5.2 Compliance enforcement and reporting

- **Azure Policy and Policy Initiatives** at the CloudMed and Production/Development management groups:  
  - Enforce allowed regions, required encryption, diagnostic settings, SKU restrictions, and rules preventing public IPs where they are not allowed.  
  - Require key tags like Environment, CostCenter, Owner, DataClassification, and Application.  
- **Policy compliance reports**:  
  - Used by security and compliance teams to find non-compliant resources and track remediation.  
  - Can be exported to Log Analytics or Event Hub for long-term analysis and reporting.  
- **Defender for Cloud regulatory compliance dashboards**:  
  - Map technical controls to frameworks such as HIPAA and GDPR, and support PIPEDA reporting through mapped and custom controls.

---

#### 5.3 Cost management and optimization

- **Tags for cost allocation and governance**:  
  - Standard tags include `Environment`, `CostCenter`, `Application`, `Owner`, and `BusinessUnit`.  
  - Policy makes sure all billable resources are tagged so costs can be attributed correctly.

- **Azure Cost Management + Billing**:  
  - **Budgets** are defined per subscription, environment, and key applications such as MedConnect production.  
  - **Budget alerts** send notifications (for example, emails or Action Groups) when actual or forecasted costs cross certain thresholds.  
  - Cost anomaly detection and recommendations are used to find unusual spending or optimization opportunities.

- **Cost alerts and dashboards**:  
  - Dashboards show spending by management group or subscription, grouped by region, service, and tags.  
  - Alerts are used to detect sudden spikes in usage or cost.

- **Governance and cost controls together**:  
  - Policies restrict very expensive SKUs or unnecessary services, especially in Dev/Test.  
  - Reserved instances, savings plans, and autoscaling are used where appropriate to reduce costs.

---

### 6. Conceptual diagram

```text
+=================================================================================+
|      Identity, Governance, CAF Landing Zones & Zero Trust Principles            |
|---------------------------------------------------------------------------------|
| - Azure Entra ID, Conditional Access, MFA, PIM                                  |
| - CAF landing zones: Platform + Application landing zones                       |
| - Azure Policy / Initiatives (regions, tags, encryption, IP rules)              |
| - Defender for Cloud, Cost Management + Budgets                                 |
| [Zero Trust: Verify Explicitly, Least Privilege, Assume Breach]                |
+=================================================================================+

Management groups (tenant-wide governance, CAF resource organization)

Tenant Root
└─ MG: CloudMed
   ├─ MG: Platform (Platform landing zone)
   │   └─ Subs: cm-mgmt-shared, cm-security-monitoring
   ├─ MG: Production (Application landing zones)
   │   ├─ Sub: cm-medconnect-prod-ca-central
   │   └─ Sub: cm-medconnect-prod-weu
   └─ MG: Development (Application landing zones)
       └─ Subs: cm-medconnect-dev-test, cm-sandboxes

---------------------------------------------------------------------------------
Region: Canada Central (CAF network topology & connectivity design area)
---------------------------------------------------------------------------------

                     +-------------------------------------------+
                     |        Hub vNet (ca-central-hub)          |
                     |          (Connectivity / Platform)        |
                     |-------------------------------------------|
                     |  - Azure Firewall                         |
                     |  - Azure Bastion                          |
                     |  - DNS / Private DNS                      |
                     |  - Diagnostics → Log Analytics / SIEM     |
                     |  [ZT: Inspect, broker, log all traffic]   |
                     +--------------------+----------------------+
                                          ^
                                          |  vNet peering + UDRs
                                          |  (all egress & inter-spoke
                                          |   flows via Firewall)
        +---------------------------------+--------------------------------+
        |                                 |                                |
        v                                 v                                v
+-------------------------+    +-------------------------+      +-------------------------+
|  App Spoke vNet         |    |  API Spoke vNet        |      |  Data Spoke vNet        |
|  (Application LZ)       |    |  (Application LZ)      |      |  (Application LZ)       |
|-------------------------|    |------------------------|      |------------------------|
| - snet-app-frontend     |    | - snet-api-backend     |      | - snet-data-pe         |
| - snet-app-private-ep   |    | - snet-api-integrations|      | - snet-data-processing |
|                         |    |                        |      |  (SQL, Storage, KV via |
| Inbound: from WAF/FD    |    | Only from App &        |      |   Private Endpoints)   |
| Outbound: via Firewall  |    | platform components    |      | No public endpoints    |
| [ZT: Internet edge,     |    | [ZT: Middle tier,      |      | [ZT: Data isolation,   |
|      minimal exposure]  |    |      whitelisted flows]|      |      private-only]     |
+-------------------------+    +-------------------------+      +-------------------------+

                 App  ───▶  API  ───▶  Data
      (no direct App ───▶ Data, no spoke-to-spoke peering)

---------------------------------------------------------------------------------
Monitoring & compliance overlay (CAF management, security, and governance areas)
---------------------------------------------------------------------------------

+-----------------------------+
|   Monitoring & Analytics    |
|-----------------------------|
| - Azure Monitor /           |
|   Log Analytics workspace   |
| - Azure Monitor alerts      |
| - Defender for Cloud        |
| - Optional Sentinel SIEM    |
+-----------------------------+

- Governance (MGs, Policy, RBAC, PIM): [ZT: Verify Explicitly, Least Privilege]
- Network (hub-and-spoke, Firewall, NSGs, private endpoints): [ZT: Assume Breach]
- Data tier (private-only, encrypted, no public IPs): [ZT: Assume Breach, Least Privilege]
- Identity (Entra ID, Conditional Access, MFA): [ZT: Verify Explicitly]
```
### 7. Summary and recommendations

CloudMed’s Azure landing zone uses clear management groups, strong identity controls with Entra ID and MFA, and a hub-and-spoke network to protect patient data. App, API, and data tiers are separated into different spokes, with private endpoints and firewall rules to control traffic. Azure Policy, Defender for Cloud, and central logging help keep the environment secure, compliant, and ready to grow across regions and new workloads.

**Future improvements**

1. **Automation and GitOps**  
   Use Infrastructure as Code (such as Bicep or Terraform) and CI/CD pipelines to deploy landing zones, policies, and RBAC in a repeatable way.

2. **Cost and performance optimization**  
   Add regular reviews for rightsizing, reserved instances or savings plans, and automatic shutdowns in non-production, supported by dashboards and cost alerts.

