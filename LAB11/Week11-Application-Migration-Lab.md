# Week11 Application Migration Lab

## Task 1 – Set Up Tooling for Discovery

**Discovery method**

- Use a **mix** of methods:  
  - **Agentless** for the VMware VMs: `WEB01`, `WEB02`, `APP01`.  
  - **Agent-based** for the physical SQL server: `SQL01`.

**Appliances**

- Deploy **one Azure Migrate appliance** for the **VMware** environment.  
  - It connects to vCenter and discovers `WEB01`, `WEB02`, `APP01`.  
- `SQL01` is physical, so we discover it with **agents**, not another appliance.

**Credentials needed**

- **Software inventory:** OS/domain creds on all four servers so the tool can read installed apps/roles.  
- **SQL discovery:** SQL or Windows account on `SQL01` with rights to list instances and databases.  
- **Dependency mapping:** OS/domain creds on `WEB01`, `WEB02`, `APP01`, `SQL01` so the tool can see ports, processes, and connections.

**Three discovery best practices**

1. Use **agentless** for VMware by default and **agents** only when necessary.  
2. Get all OS and SQL **credentials ready in advance** with the infra and DBA teams.  
3. **Monitor the Azure Migrate project** to make sure data is being collected and discovery runs long enough to capture normal load.

---

## Task 2 – Perform Assessment Planning

**Assessment type**

- Treat this 3-tier app as a **production** workload.

**Target region and performance history**

- **Target region:** Tailwind’s **primary Azure region**.  
- **Performance history:** **1 month** of performance data.

**Sizing approach**

- Use **performance-based sizing**, not “match on-prem VM size”.  
- This lets Azure Migrate right-size CPU, RAM, and disks based on **real usage** instead of current over/under-provisioning.

**Comfort factor, pricing, licensing**

- **Comfort factor:** around **1.3** to give ~30% headroom for spikes.  
- **Pricing model:** **Pay-as-you-go** in the assessment.  
- **Licensing model:** **License-included** for Windows/SQL so the estimated cost already includes licenses.

---

## Task 3 – Dependency Analysis

**Main components**

- `WEB01`, `WEB02` – web/front-end
- `APP01` – application/API 
- `SQL01` – SQL Server 2017
- `LB01` – internal load balancer  
- Nightly backup system  
- Firewalls

**Key dependencies**

1. Users → `LB01` over HTTP/HTTPS.  
2. `LB01` → `WEB01`/`WEB02` over HTTP/HTTPS.  
3. `WEB01`/`WEB02` → `APP01` over HTTP/HTTPS or API ports.  
4. `APP01` → `SQL01` over TCP.  
5. `SQL01` → backup target for nightly backups.  
6. All servers → AD/DNS.  
7. All servers → monitoring system.  
8. All servers → patching/update service.  
9. Firewall rules between web/app/db tiers and for outbound traffic.

**Noise to filter out**

- Generic **system processes** not specific to the app.  
- **Internet endpoints** that are not app dependencies.

**Business requirements**

- **Business criticality:** High-priority internal LOB app. downtime affects daily operations.  
- **Uptime/downtime:** High availability required. only **1-hour** allowed for cutover.  
- **Data classification:** `SQL01` holds **confidential internal data**, so it must stay protected and within controlled networks.  
- **Licensing dependencies:** Uses Windows + SQL Server 2017 licenses. check if any third-party tools are tied to IP/MAC/hostname.  
- **Patching:** Regular patching. avoid disruptive updates during business hours. migration should align with maintenance windows.  
- **Firewall/IP:** Existing rules control tier-to-tier and outbound access. moving to Azure changes IPs, so rules, backup targets, monitoring endpoints, and allowlists must be updated.

---

## Task 4 – Validate Assessment Results with Application Owner

**Recommended VM sizes**

- **WEB tier (`WEB01`, `WEB02`):** small general-purpose VM, e.g. **Standard D2s_v3**.  
- **APP tier (`APP01`):** medium general-purpose VM, e.g. **Standard D4s_v3**.  
- **SQL tier (`SQL01`):** memory-optimized VM, e.g. **Standard E8s_v3**.

**Components to replace/optimize in Azure**

- Replace `LB01` with **Azure Load Balancer** or **Application Gateway**.  
- Move backups to **Azure Backup / VM backups**.  
- For SQL: either stay on a tuned SQL VM or move to **Azure SQL MI/DB** for managed HA and patching.  
- Use **availability sets/zones** or PaaS HA instead of on-prem clustering.

**Validate dependencies**

With the app owner, confirm that the assessment covers:

- User → `LB01` → `WEB01`/`WEB02`  
- `WEB01`/`WEB02` → `APP01`  
- `APP01` → `SQL01`  
- `SQL01` → backups  
- All tiers → AD, DNS, monitoring, patching  
- Required firewall rules

If all are captured, dependency coverage is acceptable.

**SLA and downtime check**

- Design aims for at least the same **availability SLA** as on-prem.  
- Cutover plan keeps the **final outage within 1 hour**, with pre-provisioned resources, final sync, DNS/connection string changes, and a defined rollback.

**SQL migration options**

1. **Rehost:**  
   - Easiest, minimal changes, similar admin model to on-prem.

2. **Azure SQL Managed Instance:**  
   - High compatibility with SQL Server 2017, built-in HA/backups/patching, less admin.

3. **Azure SQL Database:**  
   - Fully managed PaaS. great for scaling, but may need more app changes and feature checks.

---

## Task 5 – Migration Plan

**1. Scope**

- One tightly coupled group: `WEB01`, `WEB02`, `APP01`, `SQL01`, `LB01`, backups, and related firewall rules.  
- **Cutover downtime window:** 1 hour.

### 2. Pre-migration tasks

- Review assessment and confirm VM sizes and SQL target with infra + DBAs.  
- Design target Azure setup: VNet, subnets, NSGs, VMs or SQL PaaS, and load balancer.  
- Double-check dependencies.  
- Ensure all necessary OS and SQL credentials are available.  
- Verify recent backups and document restore steps.  
- Agree on migration date/time and communicate the 1-hour outage.  
- Prepare DNS and connection string changes.

### 3. Migration steps

**Before the downtime window**

- Create Azure VNet, subnets, NSGs, and load balancer.  
- Provision the Azure VMs or SQL PaaS.  
- Install required OS roles, app prerequisites, and SQL.  
- Set up initial DB migration and copy necessary app files/configs.  
- Optionally test in a non-prod or pilot environment.

**During the 1-hour cutover**

1. **Stop new usage**  
   - Announce downtime and drain/stop connections to on-prem `LB01`/web/app.

2. **Final DB cutover**  
   - Stop on-prem app services.  
   - Do final DB sync from `SQL01` to Azure target and bring DB online in Azure.  
   - Ensure no more writes hit on-prem SQL.

3. **Start Azure stack**  
   - Start SQL services or Azure SQL MI/DB.  
   - Update app configs on Azure WEB/APP to use the new SQL endpoint.  
   - Start Azure web and app services and test basic internal paths.

4. **Switch traffic**  
   - Update DNS to point to the Azure load balancer.  
   - Confirm users now hit the Azure environment and monitor health.

### 4. DNS updates

- Identify all app-related DNS records.  
- During cutover, change them from on-prem `LB01`/IPs to the **Azure load balancer** address.  
- Verify name resolution and connectivity from client networks.

### 5. Connection string changes

- Find all references to `SQL01` in app configs and jobs.  
- During cutover, update them to the new SQL endpoint.  
- Restart services so they pick up the new settings.

### 6. Load balancer

- Configure an **Azure** load balancer or Application Gateway: backend pool, health probes, and rules.  
- Verify that both web servers are healthy and traffic is balanced correctly.

### 7. SQL migration considerations

- For **SQL on VM**, check disks/IOPS, HA (availability sets/zones), and backups.  
- For **SQL MI** or **SQL DB**, check feature compatibility, network integration, and performance tier.  
- Ensure the chosen option fits the 1-hour cutover and security/data requirements.

### 8. Post-migration validation

- Confirm users can access the app and run key transactions.  
- Check connections to SQL, AD/DNS, monitoring, backups, and patching services.  
- Compare performance to pre-migration baselines and check logs for errors.  
- Verify backups and scheduled jobs are running in Azure.  
- Get sign-off from the application owner.

### 9. Back-out plan

If the Azure cutover fails and can’t be fixed within 1 hour:

- Decide to **roll back** within the window.  
- Revert DNS to point back to the on-prem `LB01`/environment.  
- Restart on-prem `WEB01`, `WEB02`, `APP01`, and `SQL01` and confirm they work.  
- Treat Azure data as temporary. on-prem remains the source of truth.  
- Document what went wrong and adjust the plan before trying again.

---

## Task 6 – Plan the Migration Waves

### Wave design

- **Wave 1 – Web tier:** Move `WEB01`, `WEB02` and introduce the Azure load balancer.  
  - Stateless, easier to roll back, and lets us test Azure networking early.

- **Wave 2 – App tier:** Move `APP01`.  
  - Depends on SQL but can initially talk to on-prem SQL, letting us test full web+app in Azure before touching data.

- **Wave 3 – SQL tier:** Move `SQL01`.  
  - Most critical and highest risk. must respect the 1-hour downtime and needs careful validation.

### Final migration waves table

| Wave   | Servers                    | Reason                                                                                 |
|--------|----------------------------|----------------------------------------------------------------------------------------|
| Wave 1 | WEB01, WEB02, LB01         | Stateless front end. easy rollback. tests Azure networking and load balancing first    |
| Wave 2 | APP01                      | Depends on WEB tier. medium risk. can be tested with on-prem SQL                      |
| Wave 3 | SQL01                      | Holds critical data. highest risk. must meet 1-hour downtime and be fully validated   |
