# CCF Blob Connector Accelerator

> **Ask GitHub Copilot**: *"Help me deploy this accelerator https://github.com/robertmoriarty12/CCF-Blob-Connector-Accelerator"* — Copilot will walk you through every step below interactively.

This accelerator is a reference implementation of a Microsoft Sentinel **Codeless Connector Framework (CCF) Blob Connector** using the `StorageAccountBlobContainer` kind. The **ContosoFort** solution included here is a fictional ISV connector built to demonstrate the complete end-to-end pattern — from Azure Blob Storage through Event Grid to a custom Log Analytics table — without writing any code.

Use this accelerator to:
- **Learn** the CCF blob connector pattern before building your own
- **Test** Sentinel CCF blob ingestion in a live environment
- **Adapt** the ContosoFort template into your own product's connector by replacing branding and schema

---

## What is the CCF Blob Connector?

The [Codeless Connector Framework (CCF)](https://learn.microsoft.com/en-us/azure/sentinel/create-codeless-connector) lets ISV partners integrate log sources into Microsoft Sentinel without deploying infrastructure. The `StorageAccountBlobContainer` connector kind uses an event-driven pattern:

1. Your product writes JSON log files to an **Azure Blob container** (ADLS Gen2)
2. **Azure Event Grid** detects each new blob and pushes a `BlobCreated` notification to a **Storage Queue**
3. The **Microsoft Sentinel CCF poller** reads the queue, fetches the blob content, and sends it to a **Data Collection Rule (DCR)**
4. The **DCR** applies a KQL transform and writes rows to your custom **Log Analytics table**

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Your Product / Data Source                                         │
│  (writes JSON log files)                                            │
└───────────────────────────┬─────────────────────────────────────────┘
                            │  Upload blob
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Azure Storage Account (ADLS Gen2 – HNS enabled)                    │
│  ┌─────────────────────────────────┐  ┌────────────────────────┐    │
│  │  Blob Container                 │  │  Storage Queues        │    │
│  │  contosofort-logs/              │  │  sentinel-connector-   │    │
│  │    *.json                       │  │  notification          │    │
│  └─────────────────────────────────┘  │  sentinel-connector-   │    │
│                                       │  dlq                   │    │
│                                       └────────────┬───────────┘    │
└───────────────────────────────────────────────────┼─────────────────┘
                            │ BlobCreated event      │ queue message
                            ▼                        │
              ┌─────────────────────────┐            │
              │  Azure Event Grid       │            │
              │  System Topic +         │────────────┘
              │  Subscription           │
              └─────────────────────────┘
                                                     │
                            ┌────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Microsoft Sentinel CCF Poller                                      │
│  (polls queue → fetches blob → sends to DCR via Log Ingestion API) │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Data Collection Rule (DCR)                                         │
│  KQL transform: source | extend TimeGenerated = now() | project ... │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
                  ContosoFortV1_CL
              (Log Analytics custom table)
```

### CCF Connector Components

| Component | File | Purpose |
|---|---|---|
| Connector Definition | `ContosoFortLog_ConnectorDefinition.json` | Sentinel UI tile, instructions, permissions display |
| Poller Config | `ContosoFortLog_PollerConfig.json` | `StorageAccountBlobContainer` kind, DCR stream config |
| Data Collection Rule | `ContosoFortLog_DCR.json` | Stream schema, KQL transform, workspace destination |
| Table Schema | `ContosoFortLog_Table.json` | Custom Log Analytics table (`ContosoFortV1_CL`) |
| ARM Template | `ContosoFort/Package/mainTemplate.json` | Full solution deployment template (generated) |

### Data Schema — `ContosoFortV1_CL`

| Column | Type | Description |
|---|---|---|
| `TimeGenerated` | datetime | Ingestion timestamp (UTC), set by DCR transform |
| `EventTime` | datetime | Event timestamp from source log |
| `EventType` | string | `ThreatDetected`, `NetworkAlert`, `PolicyViolation`, `Audit` |
| `Severity` | string | `Critical`, `High`, `Medium`, `Low`, `Informational` |
| `Action` | string | `Allow`, `Block`, `Monitor`, `Quarantine` |
| `SourceIP` | string | Source IP address of the network event |
| `DestinationIP` | string | Destination IP address |
| `ThreatName` | string | Threat name or signature (empty if no threat) |
| `RuleID` | string | Identifier of the rule that triggered the event |

---

## Repository Structure

```
Tools/CCF-Blob-Connector-Accelerator/
├── README.md                          ← This file
├── storage-account-deploy.json        ← ARM template: ADLS Gen2 storage account + container
└── ContosoFort/
    ├── SolutionMetadata.json          ← Publisher/partner metadata
    ├── ReleaseNotes.md                ← Solution changelog
    ├── Data/
    │   └── Solution_ContosoFort.json  ← Solution manifest for packaging tool
    ├── Data Connectors/
    │   └── ContosoFortLog_CCF/
    │       ├── ContosoFortLog_ConnectorDefinition.json  ← Connector UI definition
    │       ├── ContosoFortLog_PollerConfig.json         ← StorageAccountBlobContainer config
    │       ├── ContosoFortLog_DCR.json                  ← Data Collection Rule
    │       └── ContosoFortLog_Table.json                ← Custom table schema
    ├── Package/
    │   └── mainTemplate.json          ← Deployable ARM template (deploy this to Sentinel)
    └── Sample Data/
        ├── ContosoFortSampleData.json  ← 3 test events
        └── ContosoFortSampleData2.json ← 4 additional test events
```

---

## Prerequisites

### Azure Permissions

> **You need Owner role** on the storage account subscription. The RBAC role assignments in Step 3 (`Microsoft.Authorization/roleAssignments/write`) require Owner or a custom role with that permission. Contributor alone is not sufficient.

| Permission Needed | Why | Minimum Role |
|---|---|---|
| Create resource groups and storage accounts | Step 1 — deploy storage | Contributor on subscription or RG |
| Deploy ARM templates to Sentinel workspace | Step 2 — deploy solution | Contributor on Sentinel resource group |
| Assign RBAC roles on storage resources | Step 3 — grant queue/blob access | **Owner** on the storage account subscription |
| Create Event Grid system topics and subscriptions | Step 4 — connector auto-provisions | Contributor on storage account RG |
| Write data connectors in Sentinel | Step 4 — connector registration | Microsoft Sentinel Contributor |

**Recommended**: **Owner** on the subscription where the storage account will be deployed, plus **Microsoft Sentinel Contributor** on the Sentinel workspace resource group.

### Tooling

- [ ] **VS Code** with the **GitHub Copilot** extension installed
- [ ] **Azure CLI** — [Install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  - Verify: `az --version`
  - Log in: `az login`
  - Set subscription: `az account set --subscription "<your-subscription-id>"`

### Azure Resources

- [ ] **Azure subscription** — with Owner role (see above)
- [ ] **Microsoft Sentinel workspace** already deployed — [Quickstart: Onboard Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/quickstart-onboard)
- [ ] **Microsoft.EventGrid provider** registered:
  ```bash
  az provider register --namespace Microsoft.EventGrid --wait
  ```
- [ ] **Microsoft.SecurityInsights provider** registered:
  ```bash
  az provider register --namespace Microsoft.SecurityInsights --wait
  ```

### Tenant Consent

- [ ] The Sentinel Service Principal (app ID: `4f05ce56-95b6-4612-9d98-a45c8cc33f9f`) must be consented in your tenant. It appears as an Enterprise Application once you first interact with the connector. If it is not present, a Global Administrator must grant tenant-wide admin consent.

---

## Step 1 — Deploy the Storage Account

The CCF blob connector requires an **Azure Data Lake Storage Gen2** (ADLS Gen2) storage account with **hierarchical namespace enabled**. This is a hard requirement — standard StorageV2 without HNS will not work.

> **Why ADLS Gen2?**  
> The CCF `StorageAccountBlobContainer` connector uses ADLS Gen2 APIs for blob access. See [MS Learn: Azure Storage Blob connector reference](https://learn.microsoft.com/en-us/azure/sentinel/data-connection-rules-reference-azure-storage).

### Option A — Deploy via Azure Portal (ARM template)

Deploy the included ARM template [`storage-account-deploy.json`](./storage-account-deploy.json):

1. Open the [Azure Portal](https://portal.azure.com) and search for **"Deploy a custom template"**
2. Click **Build your own template in the editor**
3. Click **Load file** and upload `storage-account-deploy.json` from this folder (or paste its contents)
4. Click **Save**
5. Fill in the deployment parameters:

   | Parameter | Description | Example |
   |---|---|---|
   | **Subscription** | Your Azure subscription | Visual Studio Enterprise |
   | **Resource Group** | Create new or use existing | `contosofort-ccfblob-rg` |
   | **Region** | Azure region — recommend matching your Sentinel workspace region | `Central US` |
   | **Storage Account Name** | Globally unique, 3–24 lowercase alphanumeric chars | `contosofortlogs1234` |
   | **Location** | Leave as default (inherits resource group region) | *(default)* |
   | **Blob Container Name** | Leave as default `contosofort-logs` | `contosofort-logs` |

6. Click **Review + create** → **Create**

After deployment, note the **Outputs** tab — it shows:
- `storageAccountName` — needed for RBAC assignments in Step 3
- `blobContainerUrl` — paste this into the Sentinel connector UI in Step 4

> 📖 **ARM template deployment guide**: [Quickstart: Create and deploy ARM templates by using the Azure portal](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/quickstart-create-templates-use-the-portal)

### Option B — Deploy via Azure CLI

```bash
# 1. Create a resource group (skip if using an existing one)
az group create --name contosofort-ccfblob-rg --location centralus --output table

# 2. Deploy the storage account (ADLS Gen2, HNS enabled)
az storage account create \
  --name contosofortlogs1234 \
  --resource-group contosofort-ccfblob-rg \
  --location centralus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --output table

# 3. Create the blob container (filesystem)
az storage fs create \
  --name contosofort-logs \
  --account-name contosofortlogs1234 \
  --auth-mode login \
  --output table
```

> **Note**: Use `az storage fs create` (ADLS Gen2 filesystem API) rather than `az storage container create` for HNS-enabled accounts.

### Verify HNS is enabled

```bash
az storage account show \
  --name <storage-account-name> \
  --resource-group <resource-group> \
  --query "{name:name, hns:isHnsEnabled, location:location, provisioningState:provisioningState}" \
  --output table
```

Confirm `Hns` column shows `True` before proceeding.

---

## Step 2 — Deploy the Sentinel Solution

Deploy the ContosoFort connector solution to your Sentinel workspace using `ContosoFort/Package/mainTemplate.json`.

### Look up your workspace details first

```bash
# List all Log Analytics workspaces in your subscription
az monitor log-analytics workspace list \
  --query "[].{name:name, resourceGroup:resourceGroup, location:location}" \
  --output table

# Get the name and location of a specific workspace
az monitor log-analytics workspace show \
  --name <workspace-name> \
  --resource-group <workspace-resource-group> \
  --query "{name:name, location:location, resourceGroup:resourceGroup}" \
  --output table
```

### Deploy via Azure Portal

1. Open the [Azure Portal](https://portal.azure.com) and search for **"Deploy a custom template"**
2. Click **Build your own template in the editor**
3. Click **Load file** and upload `ContosoFort/Package/mainTemplate.json`
4. Click **Save**
5. Fill in the deployment parameters:

   | Parameter | Description | Example |
   |---|---|---|
   | **Subscription** | Subscription where your Sentinel workspace lives | *(your subscription)* |
   | **Resource Group** | Resource group of your Sentinel workspace | `sentinel-rg` |
   | **Workspace** | Your Log Analytics workspace name (from lookup above) | `my-sentinel-ws` |
   | **Workspace Location** | Region of your Log Analytics workspace | `centralus` |

6. Click **Review + create** → **Create**

This deploys:
- The **ContosoFort** data connector definition (UI tile in Sentinel Content Hub)
- The **Data Collection Rule (DCR)** for log transformation
- The **`ContosoFortV1_CL`** custom table schema

> 💡 The connector will appear in Sentinel under **Content Hub → Data Connectors** as:  
> *"ContosoFort (Using Blob Container) (via Codeless Connector Framework)"*

---

## Step 3 — Grant Required RBAC Permissions

> ⚠️ **Known Issue**: The solution deployment template attempts to auto-create the required RBAC role assignments. In testing this auto-assignment failed silently, resulting in a `BLB40011` error on first connect.

> **Sequencing note**: The blob container role (#1 below) can be assigned now. The two **queue roles (#2 and #3) require the queues to exist first** — the queues are created automatically when you click **Connect** in Step 4. If you get `BLB40011` on first connect:
> 1. Run role assignments #2 and #3 below (queues now exist)
> 2. Wait 5 minutes for RBAC to propagate
> 3. Click **Connect** again

The Microsoft Sentinel Service Principal (app ID: `4f05ce56-95b6-4612-9d98-a45c8cc33f9f`) needs three roles on your storage account resources.

### 3a — Get the Service Principal Object ID

```bash
az ad sp show --id "4f05ce56-95b6-4612-9d98-a45c8cc33f9f" --query "id" -o tsv
```

Save this value — it's your `<sp-object-id>`.

### 3b — Assign the Three Required Roles

Replace the placeholder values with your actual values:

```bash
# Variables — fill these in
SP_ID="<sp-object-id>"               # from step 3a
SUB_ID="<subscription-id>"           # az account show --query id -o tsv
RG="<storage-account-resource-group>"
SA="<storage-account-name>"

# 1. Storage Blob Data Contributor — on the blob container
az role assignment create \
  --assignee $SP_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA/blobServices/default/containers/contosofort-logs"

# 2. Storage Queue Data Contributor — on the notification queue
#    NOTE: This queue is created when you click Connect in Step 4.
#    If this command fails with "resource not found", complete Step 4 first, then re-run.
az role assignment create \
  --assignee $SP_ID \
  --role "Storage Queue Data Contributor" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA/queueServices/default/queues/sentinel-connector-notification"

# 3. Storage Queue Data Contributor — on the dead-letter queue
az role assignment create \
  --assignee $SP_ID \
  --role "Storage Queue Data Contributor" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA/queueServices/default/queues/sentinel-connector-dlq"
```

> ⏱️ **Azure RBAC propagation takes 1–5 minutes.** Wait before proceeding to Step 4.

| Role | Scope | Purpose |
|---|---|---|
| Storage Blob Data Contributor | `contosofort-logs` container | Read blob content |
| Storage Queue Data Contributor | `sentinel-connector-notification` queue | Read and delete queue messages |
| Storage Queue Data Contributor | `sentinel-connector-dlq` queue | Write failed messages to DLQ |

> 📖 See: [Azure built-in roles for Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory#azure-built-in-roles-for-blobs)

---

## Step 4 — Connect the Connector in Sentinel

1. Navigate to your **Microsoft Sentinel** workspace in the [Azure Portal](https://portal.azure.com)
2. Go to **Content Hub** → **Data Connectors**
3. Find **ContosoFort (Using Blob Container) (via Codeless Connector Framework)**
4. Click **Open connector page**
5. Fill in the connection fields:

   | Field | Where to Find It | Example |
   |---|---|---|
   | **Blob container URL** | Deployment output from Step 1, or: Storage Account → Containers → right-click → Properties | `https://contosofortlogs1234.blob.core.windows.net/contosofort-logs` |
   | **Storage account resource group name** | Azure Portal → Storage Account → Overview → Resource group | `contosofort-ccfblob-rg` |
   | **Storage account location** | Azure Portal → Storage Account → Overview → Location | `centralus` |
   | **Storage account subscription ID** | Azure Portal → Subscriptions, or `az account show --query id -o tsv` | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
   | **Event Grid topic name** | Leave **empty** on first connect. If a system topic already exists for this storage account, enter its name from Storage Account → Events. | *(leave blank)* |

6. Click **Connect**

When you click Connect, the connector automatically provisions:
- Two storage queues (`sentinel-connector-notification`, `sentinel-connector-dlq`)
- An Event Grid system topic + subscription filtering `Microsoft.Storage.BlobCreated` events
- RBAC role assignments for the Sentinel Service Principal *(may fail — see Step 3)*

**What success looks like**: The connector page shows a green **Connected** status and the toggle switches to the connected state. If the status stays disconnected or you see an error banner, check the troubleshooting table at the bottom.

> If you see **`BLB40011: Access to queue denied`**, complete Step 3 queue role assignments and click **Connect** again.

---

## Step 5 — Upload Sample Data

Trigger your first ingestion by uploading a sample log file. This creates a new blob, which fires a `BlobCreated` Event Grid event, which queues a message for Sentinel to process.

> ⚠️ **Important**: Any blobs that existed in the container **before** the connector was connected (Step 4) will **not** be automatically processed — there was no Event Grid subscription to generate queue messages for them. You must re-upload them after connecting.

### Via Azure CLI

```bash
# First upload
az storage blob upload \
  --account-name <storage-account-name> \
  --container-name contosofort-logs \
  --name "ContosoFortSampleData.json" \
  --file "ContosoFort/Sample Data/ContosoFortSampleData.json" \
  --auth-mode login

# Re-upload to re-trigger ingestion (e.g. if blob already exists, or connector wasn't
# connected on first upload — add --overwrite to fire a new BlobCreated event)
az storage blob upload \
  --account-name <storage-account-name> \
  --container-name contosofort-logs \
  --name "ContosoFortSampleData.json" \
  --file "ContosoFort/Sample Data/ContosoFortSampleData.json" \
  --auth-mode login \
  --overwrite
```

### Via Azure Portal

1. Navigate to your storage account → **Containers** → **contosofort-logs**
2. Click **Upload**
3. Select `ContosoFort/Sample Data/ContosoFortSampleData.json`
4. Click **Upload**

### Confirm the Event Grid event fired (optional)

After uploading, you can check whether a queue message was generated:

```bash
# List messages in the notification queue (peek without consuming)
az storage message peek \
  --queue-name sentinel-connector-notification \
  --account-name <storage-account-name> \
  --auth-mode login \
  --num-messages 5
```

If messages are present, Event Grid is working and Sentinel will process them shortly. An empty queue after ~1 minute means the Event Grid subscription may not be active — verify the connector status in Step 4.

---

## Step 6 — Verify Data in Log Analytics

Allow **~5 minutes** for the CCF poller to detect the queue message, fetch the blob, and ingest the data.

Navigate to **Microsoft Sentinel** → **Logs** and run:

```kql
// All ContosoFort events
ContosoFortV1_CL
| order by TimeGenerated desc
| take 10
```

Additional queries to try:

```kql
// Blocked threat events only
ContosoFortV1_CL
| where Action == "Block"
| project TimeGenerated, EventType, Severity, ThreatName, SourceIP, DestinationIP, RuleID

// Events grouped by severity
ContosoFortV1_CL
| summarize Count = count() by Severity
| order by Count desc

// High and Critical threats in the last hour
ContosoFortV1_CL
| where TimeGenerated > ago(1h)
| where Severity in ("Critical", "High")
| project TimeGenerated, EventType, ThreatName, SourceIP, DestinationIP, Action
```

---

## How the Data Flow Works (End-to-End)

```
Step 1:  You upload a blob to contosofort-logs container
            ↓
Step 2:  Azure Storage fires a Microsoft.Storage.BlobCreated event to the
         Event Grid system topic
            ↓
Step 3:  Event Grid subscription filters BlobCreated events for
         /blobServices/default/containers/contosofort-logs/...
         and pushes a queue message to sentinel-connector-notification
            ↓
Step 4:  Sentinel CCF poller detects the queue message (polls periodically)
            ↓
Step 5:  CCF fetches the blob content using the URL embedded in the queue message
            ↓
Step 6:  CCF parses the JSON (eventsJsonPaths: ["$"] = root array)
         and sends records to the DCR via the Log Ingestion API
            ↓
Step 7:  DCR transformKql runs:
         source | extend TimeGenerated = now()
               | project TimeGenerated, EventTime, EventType, Severity,
                         Action, SourceIP, DestinationIP, ThreatName, RuleID
            ↓
Step 8:  Rows written to ContosoFortV1_CL in Log Analytics
            ↓
Step 9:  Queue message deleted (success) or moved to sentinel-connector-dlq (failure)
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `BLB40011: Access to queue denied` when connecting | RBAC roles not assigned or not propagated | Complete Step 3, wait 5 minutes, click Connect again |
| Connector deploys but data never arrives | Blob was uploaded before connector was connected (no queue message) | Re-upload the blob file after connecting |
| No data after 10+ minutes | RBAC propagation still in progress | Verify role assignments exist: `az role assignment list --assignee <sp-id> --scope <storage-account-id>` |
| `Microsoft.EventGrid provider not registered` error | Provider not registered in subscription | `az provider register --namespace Microsoft.EventGrid --wait` |
| Connector not visible in Sentinel Content Hub | Solution deployed to wrong workspace/subscription | Re-deploy `mainTemplate.json` targeting the correct workspace |
| DLQ has messages | DCR processing failed (malformed JSON, schema mismatch) | Check blob content matches the expected schema (all fields present, correct types) |

---

## Adapting This Accelerator for Your Product

To build your own CCF blob connector based on this template:

1. **Rename** all occurrences of `ContosoFort` → `YourProduct` in all files
2. **Update the schema** in `ContosoFortLog_DCR.json` and `ContosoFortLog_Table.json` to match your log format
3. **Update `transformKql`** in the DCR to map your source fields to your table columns
4. **Update metadata** in `SolutionMetadata.json`:
   - `publisherId` — your Azure Marketplace publisher ID (lowercase)
   - `offerId` — your solution offer ID (lowercase)
   - `support.email` / `support.link` — your support contact
5. **Update the connector UI** in `ContosoFortLog_ConnectorDefinition.json`:
   - Update `descriptionMarkdown` and links to your product docs
   - Update the `customs` section with your product-specific prerequisites
6. **Regenerate `mainTemplate.json`** using the createSolutionV3.ps1 packaging tool:
   ```powershell
   cd Tools/Create-Azure-Sentinel-Solution/V3
   ./createSolutionV3.ps1
   # When prompted, select your Solution_YourProduct.json manifest
   ```

---

## References

| Resource | Link |
|---|---|
| Create a codeless connector for Microsoft Sentinel | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/sentinel/create-codeless-connector) |
| Azure Storage Blob connector (StorageAccountBlobContainer) API reference | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/sentinel/data-connection-rules-reference-azure-storage) |
| Data connector definitions reference | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/sentinel/data-connector-ui-definitions-reference) |
| Data collection rules overview | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview) |
| Structure of a data collection rule | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-structure) |
| Deploy ARM templates via Azure Portal | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/quickstart-create-templates-use-the-portal) |
| Azure Storage Blobs introduction | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction) |
| Create an Azure Storage account | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal) |
| Azure Event Grid overview | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/event-grid/overview) |
| Azure built-in roles for Storage | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory) |
| Cloudflare CCF Blob reference implementation | [GitHub](https://github.com/Azure/Azure-Sentinel/tree/master/Solutions/Cloudflare/Data%20Connectors/CloudflareLog_CCF) |
| Microsoft Sentinel solutions overview | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/sentinel/sentinel-solutions) |

---

> This accelerator was developed and validated against a live Microsoft Sentinel workspace. The RBAC requirements, ADLS Gen2 prerequisite, and Event Grid queue message pattern are based on direct end-to-end testing.
