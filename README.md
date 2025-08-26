# Readit – .NET App Migration to Azure (Completed)

This repository documents a **completed end‑to‑end migration** of a sample .NET application suite (“Readit”) to **Microsoft Azure**.  
The solution includes web apps, a shopping cart on AKS, a Weather API on Linux, serverless order processing, observability, security with Key Vault & Managed Identity, and a basic DR setup.

![Architecture](docs/images/architecture.png)

---

## Solution Overview

**Workloads**
- **Catalog App** (.NET, IIS on Windows VM)
- **Inventory App** (.NET, Azure App Service)
- **Weather API** (Node.js on Linux VM)
- **Shopping Cart** (containerized, AKS)
- **Order Processing** (Azure Functions – Cosmos DB & Storage)
- **Data Layer**: Azure SQL, Cosmos DB (NoSQL), Azure Storage, Azure Cache for Redis
- **Networking**: VNets, subnets, NSGs, VNet peering, Application Gateway
- **Security**: Microsoft Entra ID (Azure AD), Managed Identity, Key Vault, Private Endpoints
- **Observability**: Application Insights, Log Analytics, Container Insights
- **DR**: Secondary App Service slot for Inventory with Traffic Manager/Front Door failover

**Public entry**: Application Gateway (WAF optional) fronts Catalog VM and Inventory App.  
**Private reachability**: Private Endpoints for SQL/Key Vault/Storage/Cosmos; VNet Integration for App Service; peering for AGW→Catalog VM and VNet-to-VNet traffic.

---

## Repository Layout

```
readit-azure-portfolio/
├─ docs/
│  └─ images/architecture.png
├─ infra/             # (optional) Bicep/Terraform (not committed here)
├─ k8s/               # AKS manifests for cart (Service/Deployment/Ingress if used)
├─ src/               # App code if included later
├─ scripts/           # Helper scripts (az/kubectl)
├─ .github/workflows/ # CI examples (optional)
├─ .gitignore
├─ LICENSE
└─ README.md
```

---

## Deploy Summary

### 1) Catalog App → Windows VM (IIS)
1. Published locally:
   ```bash
   dotnet publish -o publish
   ```
2. Provisioned **Windows VM**, installed **.NET 8 Hosting Bundle**, enabled **IIS**.
3. Copied `publish/` to `C:\apps\catalog`, created IIS site **Catalog** binding **HTTP:8080**.
4. NSG inbound rule for port **8080** (dev) and restriction to trusted sources.
5. AGW backend pool later targets the VM’s **private IP**.

### 2) Weather API → Linux VM (private)
1. Provisioned **Linux VM** in **weather-subnet** (separate VNet for demo).
2. Installed Node.js + npm, then:
   ```bash
   sudo apt update
   sudo apt install -y git nodejs npm
   git clone https://github.com/memilavi/weatherAPI.git
   cd weatherAPI
   npm install
   npm start
   ```
3. Assigned private IP (e.g., `10.1.0.4`) and allowed ingress only from the app network via NSG.

### 3) Shopping Cart → AKS
1. Connected to cluster:
   ```bash
   az login
   az aks get-credentials --resource-group catalog-vm_group --name cart-aks
   kubectl get nodes
   ```
2. Built cart image and pushed to **Azure Container Registry (ACR)**.
3. Deployed manifests:
   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```
4. Created a **Private Endpoint** from AKS VNet to ACR for private pull (no public access).

### 4) Application Gateway (AGW)
1. Provisioned AGW in `readit-appgw-vnet/gw-subnet`, public IP **readit-ip**.
2. Backend pools: **Catalog VM** (private IP) and **Inventory App Service**.
3. **Service Endpoint** (`Microsoft.Web`) to restrict Inventory App direct access.  
   App Service **Access Restrictions** allow only AGW subnet.
4. Peering: **AGW VNet ↔ Catalog VNet** to reach the VM privately.

### 5) Azure SQL + EF Migrations + Private Endpoint
1. Created DB `readit-db` (DTU/perf tier appropriate for demo).
2. Executed EF Core migrations:
   ```bash
   dotnet tool install --global dotnet-ef
   dotnet ef migrations add -c BookContext InitialCreate
   dotnet ef database update
   ```
3. Added **Private Endpoint** to the app VNet; removed public firewall rules.

### 6) Inventory App → App Service (with VNet Integration)
1. Deployed Inventory to **App Service**.
2. Configured connection strings via **Configuration** (no secrets in code).
3. Enabled **VNet Integration** so Inventory reaches SQL/Cosmos/Storage via private endpoints.

### 7) Serverless Order Processing (Functions) + Event Grid
1. Created **Function App** (Consumption). Implemented:
   - `ProcessOrderCosmos` → upserts order docs in **Cosmos DB**.
   - `ProcessOrderStorage` → writes order artifacts to **Storage** (Blob/Queue).
2. **Storage Integration**: Cart posts order JSON to container **`neworders`**.
3. **Event Grid**: Subscribed the Storage account **BlobCreated** event to the Function endpoint (EventGridTrigger).  
   The end-to-end flow: **Cart → Blob (neworders) → Event Grid → Functions → Cosmos/Storage**.
4. Functions deployed via VS Code Azure extension.

### 8) Redis Cache Integration
1. Provisioned **Azure Cache for Redis** (small, dev tier).
2. Updated Catalog/Cart to cache book & cart items via StackExchange.Redis.
3. Connection settings retrieved securely from **Key Vault** at runtime.

### 9) Identity & Access (Microsoft Entra ID)
1. **Managed Identity** enabled on **Inventory App Service** and **Function App**.
2. **App Service Authentication** (EasyAuth) configured with Entra ID; only signed‑in users allowed.
3. Inventory uses **Microsoft.Identity.Web** to validate tokens and protect controllers (JWT bearer).
4. SQL access via AAD:
   ```sql
   CREATE USER [<inventory-app>] FROM EXTERNAL PROVIDER;
   ALTER ROLE db_datareader ADD MEMBER [<inventory-app>];
   ALTER ROLE db_datawriter ADD MEMBER [<inventory-app>];
   ```
5. Role-based authorization in app for Admin vs Reader actions.

### 10) Secrets & Keys (Azure Key Vault)
1. Created **Key Vault**; stored SQL connection string, Redis key, Storage/Cosmos keys.
2. Granted **Key Vault Secrets User** to app identities; apps use `DefaultAzureCredential` to fetch secrets.
3. Removed plain secrets from App Settings; committed only `*.example` files.

### 11) Monitoring & Observability
1. **Application Insights** enabled for App Service, Functions, VMs (agent), and AKS (Container Insights).
2. Distributed tracing (Activity/OpenTelemetry) correlates an order through all components.
3. Azure Monitor dashboards + alerts:
   - Error rate & dependency failures
   - Function failures
   - AKS pod restarts / CPU
   - AGW backend health

### 12) Security Posture
- NSGs restricted to necessary sources/ports only.
- App Service accessible only via AGW (Access Restrictions).
- Private Endpoints for data services; no public keys in repo.
- **Defender for Cloud** enabled; addressed high‑severity recommendations.

### 13) Disaster Recovery (DR)
- **Secondary**: Inventory App deployed in paired region App Service.
- **Traffic Manager/Front Door** used for endpoint health probe and failover.
- Verified failover by simulating primary outage; Inventory served from secondary.

---

## End-to-End Test (Smoke)

```bash
# 1) Place an order from Catalog/Cart (fronted by AGW).
# 2) Verify Blob created in 'neworders' container.
# 3) Confirm Functions processed event:
#    - Cosmos DB has the order document
#    - Storage or Queue contains the processed artifact
# 4) Check Inventory reads data from SQL and shows updated stock.
# 5) Inspect App Insights traces for the full order journey.
```

---

## Local Development Notes
- Use **Docker Compose** for Redis and local SQL Server Developer.
- Use **Azurite** for Storage; optional Cosmos DB Emulator.
- Keep secrets in `appsettings.Development.json` (not committed).

---

## Clean‑Up
- Delete RGs to avoid charges.
- Remove any lingering public endpoints or NSG rules used during provisioning.

---
