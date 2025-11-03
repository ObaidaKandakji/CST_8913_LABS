# ShopPro International – Azure Migration


## 1) Architecture Mapping

| Scenario component | Azure service(s) used in the estimate |
|---|---|
| Web front-end (10 VMs per region) | Virtual Machines (**D4as v5**, Linux); **Application Gateway (WAF v2)**; **Azure Front Door Standard** (global entry/CDN); **Standard Load Balancer** |
| API back-end (50 microservices on ~20 VMs) | Virtual Machines (**D2as v5**, Linux) with autoscale assumption captured as average running count |
| Payment processing | VMs included within the D4as v5 count; compliance handled by configuration/controls (encryption, private networking, policy, Defender) |
| Primary database (5 TB SQL) | **Azure SQL Managed Instance** (General Purpose, Gen5, 16 vCores) |
| Analytics database (10 TB NoSQL) | **Azure Cosmos DB for NoSQL**, autoscale |
| Data warehouse (15 TB) | **Azure Synapse Analytics - Dedicated SQL pool** |
| Data analytics & ML | **NC4as T4 v3** GPU VMs |
| Backup & DR | **Azure Backup** (GRS), **Azure Site Recovery** |
| Monitoring & security | **Azure Monitor (Log Analytics)**, **Microsoft Defender for Cloud (CSPM)** |


---

## 2) Monthly Cost Estimate

| Item | Monthly cost |
|---|---:|
| Web VMs - 30 × **D4as v5** (730 h) | **$3,766.80** |
| API VMs - 12 × **D2as v5** (730 h) | **$753.36** |
| GPU/ML - 4 × **NC4as T4 v3** (365 h) | **$767.96** |
| **Application Gateway (WAF v2)** | **$474.11** |
| **Standard Load Balancer** | **$59.75** |
| **Azure Front Door Standard** (North America zone) | **$559.80** |
| **SQL Managed Instance** (GP, Gen5, 16 vCores) | **$2,945.61** |
| **Cosmos DB (NoSQL)** - autoscale **30k RU/s**, **100% average utilization**, 1 region | **$3,504.00** |
| **Azure Backup** - 10 protected instances × **15 TB**, GRS, 30 daily backups | **$12,248.42** |
| **Azure Site Recovery** - 48 instances (to customer-owned sites) | **$768.00** |
| **Azure Monitor** - Auxiliary logs **15 GB/day** | **$22.50** |
| **Microsoft Defender for Cloud** - CSPM (500 billable resources) | **$2,555.00** |
| **Synapse Dedicated SQL pool** - **DW500c**, **730 h**, storage **15 TB** (+ GRS DR) | **$7,546.62** |

**Monthly total: `$35,971.93`**

---

## 3) Migration Cost (one-time)

Azure Migrate doesn't have a cost to it

---

## 4) Management Cost 

- **Azure Monitor (Auxiliary logs 15 GB/day):** **$22.50/month**  
- **Defender for Cloud - CSPM:** **$2,555.00/month**  

---

## 5) Cost Optimization Strategy

1) **VM reservations/savings plan**  
   - VM compute across web + API + GPU lines = **$5,288.12/month**.  
   - **1-year reservation (~41%)** → **save ≈ $2,168/month**.  
   - **3-year reservation (~62%)** → **save ≈ $3,279/month**.

2) **SQL Managed Instance - Azure Hybrid Benefit**  
   - The MI line shows **$1,167.60/month** for SQL license.  
   - Enabling **Azure Hybrid Benefit** removes that charge → **save $1,167.60/month**.

3) **Synapse Dedicated SQL pool - reduce runtime**  
   - Compute rate shown: **$7.55/hour** at **730 hours**.  
   - Scheduling **~480 hours/month (~16 h/day)** saves **(730−480) × $7.55 = $1,887.50/month**; storage/DR unchanged.

4) **Cosmos DB autoscale - realistic average utilization**  
   - Current configuration uses **100% average** → **$3,504.00/month**.  
   - Setting **~35% average** with the same RU/s cap yields **≈ $1,226/month** → **save ≈ $2,278/month**.

5) **Front Door - reduce edge→origin pulls**  
   - Current edge - origin component ≈ **$102.40**.  
   - Improving cache hit and halving origin traffic saves **≈ $51/month**.

6) **Backups - right-size and tier**  
   - The configured policy (10 × 15 TB, GRS, daily retention) drives **$12k+/month**.  
   - Reducing protected-instance sizes, adjusting retention, and tiering older backups to Cool/Archive significantly lowers this line.

**Illustrative “quick-toggle” optimization:**  
Applying (1) one-year VM reservations, (2) MI AHB, (3) Synapse 480 h, (4) Cosmos 35% avg, (5) Front Door origin −50% gets you a reduction of **≈ $7,552/month**, This mean **optimized monthly run rate ≈ `$28,419.90`**.

---

## 6) Future Growth & Budget Plan

Using the configured total as the baseline and applying a generic **+15% per year** growth factor:

| Year | Monthly estimate |
|---|---:|
| **Year 1 (baseline)** | **$35,971.93** |
| **Year 2 (+15%)** | **$41,367.72** |
| **Year 3 (+15%)** | **$47,572.88** |

Applying the optimization above lowers Year-1 to **≈ $28,419.90/month**


