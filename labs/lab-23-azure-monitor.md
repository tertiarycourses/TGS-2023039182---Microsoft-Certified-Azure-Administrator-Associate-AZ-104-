# Lab 23 — Azure Monitor: Metrics, Logs, KQL, and Alerts

Maps to AZ-104 objective **Monitor and maintain Azure resources → Monitor resources with Azure Monitor** (interpret metrics; configure log settings via diagnostic settings; query and analyze logs with KQL; set up alert rules, action groups, and alert processing rules; configure Insights).

You will create a Log Analytics workspace, route diagnostic data into it, read platform metrics, write real Kusto (KQL) queries against `Heartbeat`, `Perf`, and `AzureActivity`, then wire up an action group, a metric alert, an activity-log alert, and an alert processing rule — the full Azure Monitor loop the exam expects you to know.

Run all steps in the **Azure Portal** (https://portal.azure.com) and **Azure Cloud Shell** (https://shell.azure.com).

**Azure services used:** Azure Monitor (Metrics, Logs, Alerts, Action Groups, Alert Processing Rules, Insights), Log Analytics workspace, a Linux VM and a storage account as monitored resources.

---

## Step 1 — Create the resource group and a Log Analytics workspace

**Portal:** Search **Log Analytics workspaces → + Create**. Set **Resource group** = `rg-az104-lab23`, **Name** = `law-az104-lab23`, **Region** = your region, then **Review + create**. The workspace is where diagnostic logs and Insights data land.

```bash
LOCATION="southeastasia"
RG="rg-az104-lab23"
WS="law-az104-lab23"

az group create --name "$RG" --location "$LOCATION"

# create the Log Analytics workspace (retention defaults to 30 days on the pay-as-you-go SKU)
az monitor log-analytics workspace create \
  --resource-group "$RG" \
  --workspace-name "$WS" \
  --location "$LOCATION" \
  --retention-time 30

# grab the workspace resource ID and GUID for later steps
WS_ID=$(az monitor log-analytics workspace show -g "$RG" -n "$WS" --query id -o tsv)
WS_GUID=$(az monitor log-analytics workspace show -g "$RG" -n "$WS" --query customerId -o tsv)
echo "$WS_ID"; echo "$WS_GUID"
```

```powershell
$RG="rg-az104-lab23"; $WS="law-az104-lab23"; $LOCATION="southeastasia"
New-AzResourceGroup -Name $RG -Location $LOCATION
New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $WS -Location $LOCATION -Sku PerGB2018
```

## Step 2 — Deploy monitored resources (VM + storage account)

**Portal:** Create a small **Virtual machine** (`vm-az104-lab23`, Ubuntu, B1s) and a **Storage account** (`stlab23<unique>`) in `rg-az104-lab23`. These give us real metrics and logs to work with.

```bash
STORAGE="stlab23$RANDOM"

# small Linux VM that will emit metrics and (with the agent) guest logs
az vm create \
  --resource-group "$RG" \
  --name "vm-az104-lab23" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# storage account to generate transaction/capacity metrics and diagnostic logs
az storage account create \
  --resource-group "$RG" \
  --name "$STORAGE" \
  --location "$LOCATION" \
  --sku Standard_LRS
```

## Step 3 — Interpret platform metrics

**Portal:** Open **vm-az104-lab23 → Monitoring → Metrics**. Set **Metric Namespace** = *Virtual Machine Host*, **Metric** = *Percentage CPU*, aggregation *Avg*. Pin the chart to a dashboard. Metrics are numeric time-series sampled at 1-minute granularity and retained 93 days.

```bash
VM_ID=$(az vm show -g "$RG" -n "vm-az104-lab23" --query id -o tsv)

# list the metric definitions available for the VM
az monitor metrics list-definitions --resource "$VM_ID" \
  --query "[].{metric:name.value, unit:unit}" -o table

# read the last hour of Percentage CPU, averaged per 5 minutes
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --aggregation Average \
  --interval PT5M \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --output table
```

```powershell
$vm = Get-AzVM -ResourceGroupName $RG -Name "vm-az104-lab23"
Get-AzMetric -ResourceId $vm.Id -MetricName "Percentage CPU" -AggregationType Average -TimeGrain 00:05:00
```

> Metrics = lightweight numeric telemetry (good for near-real-time alerting). Logs = richer, queryable records in Log Analytics. Know the difference for the exam.

## Step 4 — Configure log settings (diagnostic settings to Log Analytics)

**Portal:** Open the **storage account → Monitoring → Diagnostic settings → + Add diagnostic setting**. Under a sub-resource (e.g. `blob`) tick **StorageRead / StorageWrite / StorageDelete** and metric **Transaction**, then destination **Send to Log Analytics workspace** → `law-az104-lab23`. Repeat for the VM's platform-level activity as needed.

```bash
STORAGE_ID=$(az storage account show -g "$RG" -n "$STORAGE" --query id -o tsv)

# route the blob service logs + transaction metrics into the workspace
az monitor diagnostic-settings create \
  --name "diag-to-law" \
  --resource "$STORAGE_ID/blobServices/default" \
  --workspace "$WS_ID" \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true},{"category":"StorageDelete","enabled":true}]' \
  --metrics '[{"category":"Transaction","enabled":true}]'

# subscription-level Activity Log can also be exported to the workspace
az monitor diagnostic-settings create \
  --name "activity-to-law" \
  --resource "/subscriptions/$(az account show --query id -o tsv)" \
  --workspace "$WS_ID" \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Alert","enabled":true},{"category":"Security","enabled":true}]'
```

> Diagnostic settings are the mechanism that turns platform logs/metrics into queryable Log Analytics tables. Without them, resource logs are not retained.

## Step 5 — Enable VM Insights (and note storage/network Insights)

**Portal:** Open **vm-az104-lab23 → Monitoring → Insights → Enable**, choosing workspace `law-az104-lab23`. This deploys the **Azure Monitor Agent** and a Data Collection Rule (DCR), populating the `Perf`, `Heartbeat`, and `InsightsMetrics` tables and the VM Map. **Storage Insights** and **Network Insights** live under **Monitor → Insights hub** with no agent required.

```bash
# enable VM insights via the CLI extension (installs AMA + associates a DCR)
az extension add --name monitor-control-service --only-show-errors 2>/dev/null || true

# simplest agent-based path: install the Azure Monitor Agent, then associate a DCR in the portal
az vm extension set \
  --resource-group "$RG" \
  --vm-name "vm-az104-lab23" \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --enable-auto-upgrade true
```

> After enabling, wait ~10 minutes for the agent to report before the `Heartbeat`/`Perf` queries below return data.

## Step 6 — Query and analyze logs with KQL

**Portal:** Open **Monitor → Logs**, scope to `law-az104-lab23`, and run the queries below. KQL flows left-to-right through pipes (`|`).

```kusto
// 1) Which machines are checking in, and when did each last report?
Heartbeat
| summarize LastHeartbeat = max(TimeGenerated) by Computer, OSType
| order by LastHeartbeat desc
```

```kusto
// 2) Average CPU per machine over the last hour (from VM insights Perf table)
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

```kusto
// 3) Who changed what? Recent control-plane operations from the Activity Log
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue has_any ("write", "delete", "action")
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, ResourceGroup
| order by TimeGenerated desc
```

```bash
# run KQL from the CLI against the workspace GUID
az monitor log-analytics query \
  --workspace "$WS_GUID" \
  --analytics-query "Heartbeat | summarize LastHeartbeat = max(TimeGenerated) by Computer" \
  --output table
```

## Step 7 — Create an action group

**Portal:** **Monitor → Alerts → Action groups → + Create**. Name `ag-az104-lab23`, short name `az104ag`, add an **Email** action (your address) and optionally an **SMS**/**Webhook**. Action groups define *who/what gets notified* and are reused by many alert rules.

```bash
az monitor action-group create \
  --resource-group "$RG" \
  --name "ag-az104-lab23" \
  --short-name "az104ag" \
  --action email admin angch@tertiaryinfotech.com
```

```powershell
$email = New-AzActionGroupReceiver -Name "admin" -EmailReceiver -EmailAddress "angch@tertiaryinfotech.com"
Set-AzActionGroup -ResourceGroupName $RG -Name "ag-az104-lab23" -ShortName "az104ag" -Receiver $email
```

## Step 8 — Create a metric alert rule

**Portal:** **Monitor → Alerts → + Create → Alert rule**. Scope = `vm-az104-lab23`, **Signal** = *Percentage CPU*, **Condition** = *Average > 80* over 5 min, **Action group** = `ag-az104-lab23`, severity **Sev 3**.

```bash
AG_ID=$(az monitor action-group show -g "$RG" -n "ag-az104-lab23" --query id -o tsv)

az monitor metrics alert create \
  --name "alert-vm-highcpu" \
  --resource-group "$RG" \
  --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 3 \
  --action "$AG_ID" \
  --description "VM CPU sustained above 80%"
```

```powershell
$vmId = (Get-AzVM -ResourceGroupName $RG -Name "vm-az104-lab23").Id
$cond = New-AzMetricAlertRuleV2Criteria -MetricName "Percentage CPU" -TimeAggregation Average -Operator GreaterThan -Threshold 80
$ag = New-AzActionGroup -ActionGroupId (Get-AzActionGroup -ResourceGroupName $RG -Name "ag-az104-lab23").Id
Add-AzMetricAlertRuleV2 -Name "alert-vm-highcpu" -ResourceGroupName $RG -WindowSize 00:05:00 `
  -Frequency 00:01:00 -TargetResourceId $vmId -Condition $cond -ActionGroup $ag -Severity 3
```

## Step 9 — Create an activity-log alert

**Portal:** **Monitor → Alerts → + Create → Alert rule**, scope your subscription, **Signal type = Activity Log**, choose a signal such as *Delete Virtual Machine*, attach `ag-az104-lab23`. These fire on control-plane events rather than metrics.

```bash
SUB_ID=$(az account show --query id -o tsv)

az monitor activity-log alert create \
  --name "alert-vm-deleted" \
  --resource-group "$RG" \
  --scope "/subscriptions/$SUB_ID" \
  --condition category=Administrative and operationName=Microsoft.Compute/virtualMachines/delete \
  --action-group "$AG_ID" \
  --description "Fires when any VM is deleted"
```

## Step 10 — Create an alert processing rule

**Portal:** **Monitor → Alerts → Alert processing rules → + Create**. Scope = `rg-az104-lab23`. Choose **Apply action group** to add notifications to *all* alerts in the RG, or **Suppress notifications** with a schedule (e.g. a maintenance window). Alert processing rules let you add/suppress actions across many alerts without editing each rule.

```bash
# suppress notifications for the whole RG during a maintenance window (example: one-off)
az monitor alert-processing-rule create \
  --name "apr-suppress-maintenance" \
  --resource-group "$RG" \
  --scopes "/subscriptions/$SUB_ID/resourceGroups/$RG" \
  --rule-type RemoveAllActionGroups \
  --description "Suppress all alert notifications for lab RG during maintenance"

# alternative: add the action group to every alert in the RG
az monitor alert-processing-rule create \
  --name "apr-add-ag" \
  --resource-group "$RG" \
  --scopes "/subscriptions/$SUB_ID/resourceGroups/$RG" \
  --rule-type AddActionGroups \
  --action-groups "$AG_ID"
```

> Metric/activity alert **rules** decide *when* to fire; **alert processing rules** post-process the resulting alerts (add or suppress actions) across scopes.

## Clean up

```bash
# alerts, action groups, processing rules, VM, storage and workspace all live in the RG
az group delete --name rg-az104-lab23 --yes --no-wait
```

> The Log Analytics workspace enters a **soft-delete** state for 14 days after deletion (it can be recovered by recreating it with the same name/RG in that window). Deleting the RG removes all lab resources; no Recovery Services vault is used here.

## References

- Azure Monitor overview — https://learn.microsoft.com/azure/azure-monitor/overview
- Diagnostic settings — https://learn.microsoft.com/azure/azure-monitor/essentials/diagnostic-settings
- Log queries and KQL in Azure Monitor — https://learn.microsoft.com/azure/azure-monitor/logs/log-query-overview
- Create action groups and alert rules — https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-overview

## What you learned

- Create a Log Analytics workspace and route platform logs/metrics into it with **diagnostic settings**.
- Read and interpret **platform metrics** (Percentage CPU) versus **logs**, and know their retention differences.
- Enable **VM Insights** (Azure Monitor Agent + DCR) and locate Storage/Network Insights in the Insights hub.
- Write real **KQL** against `Heartbeat`, `Perf`, and `AzureActivity`, including `summarize`, `bin()`, and `render`.
- Build an **action group**, a **metric alert**, and an **activity-log alert**, then govern them with an **alert processing rule**.
- Perform every task via Portal, Azure CLI, and Azure PowerShell.
