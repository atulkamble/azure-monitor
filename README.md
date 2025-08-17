# Azure Monitor with AMA — VM + DCR + Alerts

End-to-end setup to collect **metrics & logs** from Azure VMs into **Log Analytics**, visualize with KQL, and trigger **alerts**. Uses the **Azure Monitor Agent (AMA)** and **Data Collection Rules (DCR)**.

> If you were using OMS (legacy), this replaces it. AMA is the recommended path.

---

## 📁 Repo Structure (suggested)

```
azure-monitor/
├─ README.md
├─ scripts/
│  ├─ 01_env.sh
│  ├─ 02_workspace.sh
│  ├─ 03_vm_agent_linux.sh
│  ├─ 03_vm_agent_windows.sh
│  ├─ 04_dcr_create.sh
│  ├─ 05_dcr_associate.sh
│  ├─ 06_alerts.sh
│  └─ 99_cleanup.sh
└─ kql/
   ├─ perf_basics.kql
   └─ syslog_recent.kql
```

> You can keep it all in the README or split into the above scripts. Both are provided below.

---

## ✅ Prerequisites

* Azure CLI ≥ 2.45 (you’re on 2.76—good)
* Owner/Contributor on the subscription/resource group
* A **Linux** or **Windows** Azure VM already running

Login and pick the right subscription:

```bash
az login
az account list -o table
az account set --subscription "cc57cd42-dede-4674-b810-a0fbde41504a"
```

---

## 0) Set Environment Variables (edit these)

```bash
# ==== REQUIRED: change to your real values ====
export SUB_ID="cc57cd42-dede-4674-b810-a0fbde41504a"
export RG="myRG"
export LOC="eastus"              # e.g., centralindia, eastus, westeurope
export VM_NAME="myvm"

# workspace name can be anything unique in the RG
export LA_WS="logws-${RG}"
export DCR_NAME="dcr-${RG}"
export DCRA_NAME="dcr-assoc-${VM_NAME}"

# Optional: Action Group for alerts (email)
export ACTION_GRP="ag-${RG}"
export ACTION_EMAIL="<you@example.com>"
```

Sanity:

```bash
az group show -g "$RG" -o table
az vm show -g "$RG" -n "$VM_NAME" -o table
```

---

## 1) Create (or Reuse) Log Analytics Workspace

```bash
az monitor log-analytics workspace create \
  -g "$RG" -n "$LA_WS" -l "$LOC" --sku PerGB2018

export LA_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$LA_WS" --query id -o tsv)
echo "Workspace ID: $LA_ID"
```

---

## 2) Install Azure Monitor Agent on the VM

### Linux VM

```bash
az vm extension set \
  --resource-group "$RG" \
  --vm-name "$VM_NAME" \
  --publisher Microsoft.Azure.Monitor \
  --name AzureMonitorLinuxAgent
```

### Windows VM (if applicable)

```bash
az vm extension set \
  --resource-group "$RG" \
  --vm-name "$VM_NAME" \
  --publisher Microsoft.Azure.Monitor \
  --name AzureMonitorWindowsAgent
```

Check extension status:

```bash
az vm extension list -g "$RG" --vm-name "$VM_NAME" -o table
```

---

## 3) Create a Data Collection Rule (DCR)

This DCR collects **Perf** & **Syslog** (Linux). You can extend it later.

```bash
az monitor data-collection rule create \
  --resource-group "$RG" \
  --location "$LOC" \
  --name "$DCR_NAME" \
  --data-flows "[{\"streams\":[\"Microsoft-Perf\",\"Microsoft-Syslog\"],\"destinations\":[\"logAnalytics\"]}]" \
  --destinations "[{\"type\":\"logAnalytics\",\"workspaceResourceId\":\"$LA_ID\",\"name\":\"logAnalytics\"}]" \
  --stream-declarations '{
    "Microsoft-Perf": {
      "columns": [
        {"name":"TimeGenerated","type":"datetime"},
        {"name":"Computer","type":"string"},
        {"name":"CounterName","type":"string"},
        {"name":"InstanceName","type":"string"},
        {"name":"ObjectName","type":"string"},
        {"name":"CounterValue","type":"real"}
      ]
    },
    "Microsoft-Syslog": {
      "columns": [
        {"name":"TimeGenerated","type":"datetime"},
        {"name":"Computer","type":"string"},
        {"name":"Facility","type":"string"},
        {"name":"SeverityLevel","type":"string"},
        {"name":"SyslogMessage","type":"string"},
        {"name":"ProcessName","type":"string"}
      ]
    }
  }'
```

Grab IDs:

```bash
export DCR_ID=$(az monitor data-collection rule show -g "$RG" -n "$DCR_NAME" --query id -o tsv)
export VM_ID=$(az vm show -g "$RG" -n "$VM_NAME" --query id -o tsv)
echo "DCR: $DCR_ID"
echo "VM:  $VM_ID"
```

---

## 4) Associate DCR to the VM

```bash
az monitor data-collection rule association create \
  --name "$DCRA_NAME" \
  --rule-id "$DCR_ID" \
  --resource "$VM_ID"
```

Verify:

```bash
az monitor data-collection rule association list --resource "$VM_ID" -o table
```

> Data typically appears in **5–10 minutes** in Log Analytics.

---

## 5) Verify Ingestion with KQL

Open **Azure Portal → Monitor → Logs** (select your workspace) and run:

**kql/perf\_basics.kql**

```kusto
Perf
| where TimeGenerated > ago(30m)
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), ObjectName, CounterName, Computer
| order by TimeGenerated desc
```

**kql/syslog\_recent.kql**

```kusto
Syslog
| where TimeGenerated > ago(30m)
| project TimeGenerated, Computer, Facility, SeverityLevel, ProcessName, SyslogMessage
| order by TimeGenerated desc
```

> If you don’t see data after \~15 minutes, see **Troubleshooting** below.

---

## 6) Create Alerts

### 6.1 Action Group (email)

```bash
az monitor action-group create \
  --name "$ACTION_GRP" \
  --resource-group "$RG" \
  --short-name "alerts" \
  --action email Atul "$ACTION_EMAIL"
```

### 6.2 **Metric Alert** — VM CPU > 80% (classic metric)

```bash
az monitor metrics alert create \
  --name "HighCPUAlert" \
  --resource-group "$RG" \
  --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when VM CPU > 80% (5m avg)" \
  --window-size 5m --evaluation-frequency 1m \
  --action "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/microsoft.insights/actionGroups/$ACTION_GRP"
```

> Metric name “**Percentage CPU**” works for Azure VM metrics.

### 6.3 **Log Alert** — High Syslog Severity (errors in last 5m)

Create a saved search (optional) or inline KQL; we’ll do inline:

```bash
LOG_QUERY='Syslog | where TimeGenerated > ago(5m) | where SeverityLevel in ("err","crit","alert","emerg") | summarize count()'

az monitor scheduled-query create \
  --name "SyslogErrorsAlert" \
  --resource-group "$RG" \
  --scopes "$LA_ID" \
  --description "Alert if high-severity syslog entries detected" \
  --action "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/microsoft.insights/actionGroups/$ACTION_GRP" \
  --condition "count > 0" \
  --condition-query "$LOG_QUERY" \
  --window-size 5m \
  --evaluation-frequency 5m
```

---

## 7) (Optional) Enable VM Insights (1-command bootstrap)

This attaches a Microsoft-managed DCR and connects to your workspace:

```bash
az monitor vm enable \
  --resource-group "$RG" \
  --vm-name "$VM_NAME" \
  --workspace "$LA_ID"
```

> Use **Insights → Virtual machines** in the portal for performance maps and dependency views.

---

## 8) (Optional) Send PaaS Diagnostics to Log Analytics

Example: send **Storage Account** logs/metrics to the same workspace.

```bash
# Replace with your Storage Account name
export SA_NAME="<yourstorageacct>"

az monitor diagnostic-settings create \
  --name "diag-to-la" \
  --resource "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA_NAME" \
  --workspace "$LA_ID" \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --logs '[
    {"category":"StorageRead","enabled":true},
    {"category":"StorageWrite","enabled":true},
    {"category":"StorageDelete","enabled":true}
  ]'
```

> You’ll see categories under tables like `AzureDiagnostics` (for many PaaS services) or resource-specific tables.

---

## 9) Visualization (quick pointers)

* **Portal → Monitor → Workbooks**: Start from Gallery → “Performance” / “Virtual Machines”.
* **Metrics Explorer**: Add charts for `Percentage CPU`, `Network In/Out`, disk metrics.
* **Grafana**: Use Azure Monitor data source and query workspace or metrics.

---

## 🔧 Troubleshooting

* **ResourceNotFound**
  Ensure you’re in the correct **subscription** and names are exact:

  ```bash
  az account show -o table
  az vm list -d -o table
  ```

* **No data in `Perf`/`Syslog` after 15m**

  * Check agent extension status:

    ```bash
    az vm extension list -g "$RG" --vm-name "$VM_NAME" -o table
    ```
  * Confirm DCR association:

    ```bash
    az monitor data-collection rule association list --resource "$VM_ID" -o table
    ```
  * On Linux VM, generate test syslog:

    ```bash
    logger -p user.err "AMA test error from $(hostname)"
    ```

* **Wrong region errors with DCR**
  DCR and target resources should be in compatible regions. Keep **DCR** in same/nearby region as VMs.

---

## 🧹 Cleanup

```bash
# Delete alerts
az monitor metrics alert delete -g "$RG" -n "HighCPUAlert" --yes
az monitor scheduled-query delete -g "$RG" -n "SyslogErrorsAlert" --yes

# Remove DCR association
az monitor data-collection rule association delete \
  --name "$DCRA_NAME" --resource "$VM_ID" --yes

# Delete DCR
az monitor data-collection rule delete -g "$RG" -n "$DCR_NAME" --yes

# (Optional) Remove VM extensions manually if needed
# az vm extension delete -g "$RG" --vm-name "$VM_NAME" --name AzureMonitorLinuxAgent
# az vm extension delete -g "$RG" --vm-name "$VM_NAME" --name AzureMonitorWindowsAgent

# Delete workspace (careful—removes data)
# az monitor log-analytics workspace delete -g "$RG" -n "$LA_WS" --yes
```

---

## 📜 Scripts (drop into `scripts/`)

**scripts/01\_env.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

export SUB_ID="<your-subscription-id>"
export RG="<your-resource-group>"
export LOC="centralindia"
export VM_NAME="<your-vm-name>"

export LA_WS="logws-${RG}"
export DCR_NAME="dcr-${RG}"
export DCRA_NAME="dcr-assoc-${VM_NAME}"

export ACTION_GRP="ag-${RG}"
export ACTION_EMAIL="<you@example.com>"
```

**scripts/02\_workspace.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"

az monitor log-analytics workspace create -g "$RG" -n "$LA_WS" -l "$LOC" --sku PerGB2018
export LA_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$LA_WS" --query id -o tsv)
echo "$LA_ID"
```

**scripts/03\_vm\_agent\_linux.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"

az vm extension set \
  --resource-group "$RG" \
  --vm-name "$VM_NAME" \
  --publisher Microsoft.Azure.Monitor \
  --name AzureMonitorLinuxAgent
```

**scripts/03\_vm\_agent\_windows.sh**

```powershell
# PowerShell Core recommended
# az vm extension set --resource-group $env:RG --vm-name $env:VM_NAME --publisher Microsoft.Azure.Monitor --name AzureMonitorWindowsAgent
```

**scripts/04\_dcr\_create.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"
export LA_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$LA_WS" --query id -o tsv)

az monitor data-collection rule create \
  --resource-group "$RG" \
  --location "$LOC" \
  --name "$DCR_NAME" \
  --data-flows "[{\"streams\":[\"Microsoft-Perf\",\"Microsoft-Syslog\"],\"destinations\":[\"logAnalytics\"]}]" \
  --destinations "[{\"type\":\"logAnalytics\",\"workspaceResourceId\":\"$LA_ID\",\"name\":\"logAnalytics\"}]" \
  --stream-declarations '{
    "Microsoft-Perf": { "columns": [
      {"name":"TimeGenerated","type":"datetime"},
      {"name":"Computer","type":"string"},
      {"name":"CounterName","type":"string"},
      {"name":"InstanceName","type":"string"},
      {"name":"ObjectName","type":"string"},
      {"name":"CounterValue","type":"real"} ] },
    "Microsoft-Syslog": { "columns": [
      {"name":"TimeGenerated","type":"datetime"},
      {"name":"Computer","type":"string"},
      {"name":"Facility","type":"string"},
      {"name":"SeverityLevel","type":"string"},
      {"name":"SyslogMessage","type":"string"},
      {"name":"ProcessName","type":"string"} ] }
  }'

az monitor data-collection rule show -g "$RG" -n "$DCR_NAME" -o table
```

**scripts/05\_dcr\_associate.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"

export DCR_ID=$(az monitor data-collection rule show -g "$RG" -n "$DCR_NAME" --query id -o tsv)
export VM_ID=$(az vm show -g "$RG" -n "$VM_NAME" --query id -o tsv)

az monitor data-collection rule association create \
  --name "$DCRA_NAME" \
  --rule-id "$DCR_ID" \
  --resource "$VM_ID"

az monitor data-collection rule association list --resource "$VM_ID" -o table
```

**scripts/06\_alerts.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"

# Create or reuse Action Group
az monitor action-group create \
  --name "$ACTION_GRP" \
  --resource-group "$RG" \
  --short-name "alerts" \
  --action email Default "$ACTION_EMAIL"

export LA_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$LA_WS" --query id -o tsv)
export VM_ID=$(az vm show -g "$RG" -n "$VM_NAME" --query id -o tsv)

# Metric alert
az monitor metrics alert create \
  --name "HighCPUAlert" \
  --resource-group "$RG" \
  --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when VM CPU > 80% (5m avg)" \
  --window-size 5m --evaluation-frequency 1m \
  --action "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/microsoft.insights/actionGroups/$ACTION_GRP"

# Log alert
LOG_QUERY='Syslog | where TimeGenerated > ago(5m) | where SeverityLevel in ("err","crit","alert","emerg") | summarize count()'
az monitor scheduled-query create \
  --name "SyslogErrorsAlert" \
  --resource-group "$RG" \
  --scopes "$LA_ID" \
  --description "Alert if high-severity syslog entries detected" \
  --action "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/microsoft.insights/actionGroups/$ACTION_GRP" \
  --condition "count > 0" \
  --condition-query "$LOG_QUERY" \
  --window-size 5m \
  --evaluation-frequency 5m
```

**scripts/99\_cleanup.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$(dirname "$0")/01_env.sh"

export LA_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$LA_WS" --query id -o tsv)
export VM_ID=$(az vm show -g "$RG" -n "$VM_NAME" --query id -o tsv)

az monitor metrics alert delete -g "$RG" -n "HighCPUAlert" --yes || true
az monitor scheduled-query delete -g "$RG" -n "SyslogErrorsAlert" --yes || true

az monitor data-collection rule association delete --name "$DCRA_NAME" --resource "$VM_ID" --yes || true
az monitor data-collection rule delete -g "$RG" -n "$DCR_NAME" --yes || true

# Optional:
# az vm extension delete -g "$RG" --vm-name "$VM_NAME" --name AzureMonitorLinuxAgent || true
# az monitor log-analytics workspace delete -g "$RG" -n "$LA_WS" --yes || true
```

---

## 🏁 Quick Start (one-liners)

```bash
# 1) Set env
source scripts/01_env.sh

# 2) Workspace
bash scripts/02_workspace.sh

# 3) VM Agent (Linux or Windows)
bash scripts/03_vm_agent_linux.sh
# bash scripts/03_vm_agent_windows.sh

# 4) DCR + Associate
bash scripts/04_dcr_create.sh
bash scripts/05_dcr_associate.sh

# 5) Alerts
bash scripts/06_alerts.sh
```

---

If you want, I can push this as a ready-to-clone repo (**azure-monitor-ama-starter**) with the file structure above. Tell me your preferred repo name, and I’ll tailor the defaults (region, email, etc.) to your setup.
