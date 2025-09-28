# Lab 2 – Cloud Deployment Models (IaaS vs PaaS)

## On-Premise Setup
```mermaid
flowchart LR
  %% On-Premise Setup (realistic DNS placement)

  B[Browser] --> FW[Firewall] --> AGW[Application Gateway]

  %% DNS side query
  B -.-> DNS[DNS Server]
  DNS -. resolves domain .-> B

  %% Two identical app pathways
  subgraph pathA[Pathway A]
    direction TB
    UIA[React UI A] --> WSA[Flask Web Server A]
  end

  subgraph pathB[Pathway B]
    direction TB
    UIB[React UI B] --> WSB[Flask Web Server B]
  end

  AGW --> UIA
  AGW --> UIB

  %% Load Balancer distributes across app servers
  WSA --> LB[Load Balancer]
  WSB --> LB

  %% Database setup with primary/replica and replication
  LB --> DBVIP[DB Virtual IP / Endpoint]
  DBVIP --> DB1[(Postgres Primary DB)]
  DBVIP --> DB2[(Postgres Replica DB)]
  DB1 <--> DB2
```

## IAAS Setup
```mermaid

flowchart LR
  %% IaaS Deployment (Azure Virtual Network with Azure-specific components)

  B[Browser] --> FW[Azure Firewall] --> AGW[Azure Application Gateway]

  %% DNS side query
  B -.-> DNS[Azure DNS]
  DNS -. resolves domain .-> B

  %% Azure Virtual Network grouping
  subgraph VNET[Azure Virtual Network]
    AGW

    %% Two VM pathways
    subgraph pathA[Pathway A]
      direction TB
      VMA[Azure VM: React UI + Flask Web Server A]
    end

    subgraph pathB[Pathway B]
      direction TB
      VMB[Azure VM: React UI + Flask Web Server B]
    end

    AGW --> VMA
    AGW --> VMB

    %% Load Balancer
    VMA --> LB[Azure Load Balancer]
    VMB --> LB

    %% Database setup on VMs
    LB --> DBVIP[DB Virtual IP / Endpoint]
    DBVIP --> DB1[(Azure VM: Postgres Primary DB)]
    DBVIP --> DB2[(Azure VM: Postgres Replica DB)]
    DB1 <--> DB2
  end

```
In an IaaS setup, the application is deployed on virtual machines within an Azure Virtual Network. Traffic is routed from the browser (via Azure DNS) through the Azure Firewall and Azure Application Gateway to VMs that host the React UI and Flask web server, managed under an Azure Load Balancer. The database layer is built on Azure VMs running PostgreSQL, configured with a primary-replica architecture accessed through a database endpoint. This model gives full control of servers and databases but requires more responsibility for patching, scaling, and maintenance.

## PAAS Setup

```mermaid
flowchart LR
  %% PaaS Deployment (Azure)

  B[Browser] --> FW[Azure Firewall] --> AFD[Azure Front Door]

  %% DNS side query
  B -.-> DNS[Azure DNS]
  DNS -. resolves domain .-> B

  %% Azure App Service hosting React + Flask
  AFD --> AppSvc[Azure App Service: React UI + Flask Web Server]

  %% Managed database 
  AppSvc --> DB[Azure Database for PostgreSQL - Managed Service]

```
In a PaaS setup, the application relies on managed services instead of self-hosted VMs. Requests move from the browser (via Azure DNS) through the Azure Firewall and Azure Front Door to Azure App Service, which runs both the React UI and Flask web server without the need to manage underlying infrastructure. The backend uses Azure Database for PostgreSQL (Managed Service), which handles replication, backups, and high availability automatically. This model reduces operational overhead and lets developers focus on application functionality rather than infrastructure management.