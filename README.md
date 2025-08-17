Here’s a **step-by-step tutorial on Azure Monitor** with commands and examples so you can get hands-on quickly 🚀.

---

# 📊 Azure Monitor Tutorial

Azure Monitor is Microsoft’s **full-stack monitoring service** for collecting, analyzing, and acting on telemetry from Azure and on-premises environments.

---

## 1️⃣ What Azure Monitor Does

* **Collect**: Metrics, logs, traces from VMs, containers, databases, apps.
* **Analyze**: Use Kusto Query Language (KQL) in **Log Analytics**.
* **Visualize**: Dashboards, Workbooks, Grafana, Power BI.
* **Act**: Alerts, Automation Runbooks, Logic Apps, Functions.

---

## 2️⃣ Enable Azure Monitor

### a) For Virtual Machines

```bash
az vm extension set \
  --resource-group MyResourceGroup \
  --vm-name MyVM \
  --name OmsAgentForLinux \
  --publisher Microsoft.EnterpriseCloud.Monitoring
```

### b) For Containers (AKS)

```bash
az aks enable-addons \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<subId>/resourceGroups/<rgName>/providers/Microsoft.OperationalInsights/workspaces/<workspaceName>
```

---

## 3️⃣ Collect Logs & Metrics

* **Metrics**: Lightweight numbers (CPU %, Memory, Disk I/O, etc.).
* **Logs**: Rich event/traces stored in **Log Analytics Workspace**.

Example: Query VM CPU usage with KQL

```kusto
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
```

---

## 4️⃣ Create Alerts

### CLI Example – Alert on CPU > 80%

```bash
az monitor metrics alert create \
  --name HighCPUAlert \
  --resource-group MyResourceGroup \
  --scopes /subscriptions/<subId>/resourceGroups/<rgName>/providers/Microsoft.Compute/virtualMachines/MyVM \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when CPU usage > 80%"
```

---

## 5️⃣ Visualization

* **Azure Portal** → Monitor → Metrics/Workbooks → Create charts.
* **Grafana**: Use **Azure Monitor plugin**.
* **Power BI**: Connect to Log Analytics with KQL queries.

---

## 6️⃣ Automation

* Link alerts to:

  * **Azure Logic Apps** (notify via Teams/Slack)
  * **Azure Functions** (auto-scale, shutdown VMs)
  * **Runbooks** (auto-remediation)

---

## 7️⃣ Best Practices

* Use **Action Groups** for alert routing.
* Separate **Prod & Non-Prod Workspaces**.
* Enable **Diagnostic Settings** for PaaS services (App Service, SQL, Cosmos DB).
* Export logs to **Storage Account** for long-term retention.

---

✅ With this setup, you can monitor **VMs, AKS, Databases, Apps, and Networks** end-to-end.

---

Would you like me to **create a complete project repo** (Terraform + sample KQL queries + alert rules + dashboard JSON) so you can deploy and test Azure Monitor in your own subscription?
