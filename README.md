# azure-monitor-foundations

# Production Linux Monitoring with Azure Monitor

## Project Overview

This project demonstrates how to build a basic production-style monitoring solution for a Linux virtual machine using Azure Monitor.

The environment collects performance data and Linux system logs from a virtual machine and sends them to a Log Analytics workspace for analysis.

The project also uses alerts, Action Groups, KQL queries, and Azure Workbooks to monitor system health and investigate issues.

## Technologies Used

- Microsoft Azure
- Azure Monitor
- Azure Monitor Agent
- Log Analytics Workspace
- Data Collection Rules
- Kusto Query Language (KQL)
- Azure Monitor Alerts
- Action Groups
- Azure Workbooks
- Linux / Ubuntu
- Nginx
- Azure CLI

## Architecture

```text
Linux VM
   |
   v
Azure Monitor Agent
   |
   v
Data Collection Rule
   |
   +------------------+
   |                  |
   v                  v
Performance Data    Syslog
   |                  |
   +--------+---------+
            |
            v
   Log Analytics Workspace
            |
            v
           KQL
            |
     +------+------+
     |             |
     v             v
   Alerts       Workbooks
     |
     v
Action Group
```

## Project Objectives

The goal of this project was to learn how to:

- Deploy a Linux virtual machine
- Install the Azure Monitor Agent
- Configure a managed identity for the VM
- Create a Log Analytics workspace
- Create a Data Collection Rule
- Associate a DCR with a virtual machine
- Collect Linux performance counters
- Collect Linux Syslog data
- Query monitoring data using KQL
- Monitor agent health using Heartbeat
- Create CPU alerts
- Configure Action Groups
- Simulate monitoring incidents
- Build an Azure Workbook

## Resources Created

The project used the following Azure resources:

```text
Resource Group:
rg-production-monitoring

Virtual Machine:
vm-production-web

Log Analytics Workspace:
law-production-monitoring

Data Collection Rule:
dcr-linux-production

Action Group:
ag-production-operations
```

The resources were deployed in:

```text
Canada Central
```

## 1. Create the Resource Group

```bash
RG2="rg-production-monitoring"
LOCATION2="canadacentral"

az group create \
  --name "$RG2" \
  --location "$LOCATION2"
```

## 2. Create a Log Analytics Workspace

```bash
LAW2="law-production-monitoring"

az monitor log-analytics workspace create \
  --resource-group "$RG2" \
  --workspace-name "$LAW2" \
  --location "$LOCATION2"
```

Retrieve the workspace resource ID:

```bash
LAW2_ID=$(az monitor log-analytics workspace show \
  --resource-group "$RG2" \
  --workspace-name "$LAW2" \
  --query id \
  --output tsv)
```

## 3. Create the Linux VM

```bash
VM2="vm-production-web"

az vm create \
  --resource-group "$RG2" \
  --name "$VM2" \
  --location "$LOCATION2" \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --assign-identity
```

The managed identity is required so the Azure Monitor Agent can authenticate correctly.

## 4. Install Nginx

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts "
sudo apt-get update -y
sudo apt-get install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
"
```

Verify Nginx:

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts "systemctl status nginx --no-pager"
```

## 5. Install Azure Monitor Agent

```bash
az vm extension set \
  --resource-group "$RG2" \
  --vm-name "$VM2" \
  --publisher Microsoft.Azure.Monitor \
  --name AzureMonitorLinuxAgent \
  --enable-auto-upgrade true
```

Verify the extension:

```bash
az vm extension show \
  --resource-group "$RG2" \
  --vm-name "$VM2" \
  --name AzureMonitorLinuxAgent \
  --output table
```

## 6. Create the Data Collection Rule

The Data Collection Rule defines what data Azure Monitor Agent should collect.

The project collects:

- CPU performance
- Memory utilization
- Disk information
- Linux Syslog events

Example DCR configuration:

```json
{
  "location": "canadacentral",
  "kind": "Linux",
  "properties": {
    "dataSources": {
      "performanceCounters": [
        {
          "name": "linuxPerformance",
          "streams": [
            "Microsoft-Perf"
          ],
          "samplingFrequencyInSeconds": 60,
          "counterSpecifiers": [
            "\\Processor(*)\\% Processor Time",
            "\\Memory\\% Used Memory",
            "\\Memory\\Used Memory MBytes",
            "\\Logical Disk(*)\\% Free Space"
          ]
        }
      ],
      "syslog": [
        {
          "name": "linuxSyslog",
          "streams": [
            "Microsoft-Syslog"
          ],
          "facilityNames": [
            "auth",
            "authpriv",
            "cron",
            "daemon",
            "kern",
            "syslog",
            "user"
          ],
          "logLevels": [
            "Warning",
            "Error",
            "Critical",
            "Alert",
            "Emergency"
          ]
        }
      ]
    }
  }
}
```

Create the DCR:

```bash
az monitor data-collection rule create \
  --resource-group "$RG2" \
  --name "dcr-linux-production" \
  --location "$LOCATION2" \
  --rule-file dcr-linux.json
```

## 7. Associate the DCR with the VM

Retrieve the resource IDs:

```bash
VM2_ID=$(az vm show \
  --resource-group "$RG2" \
  --name "$VM2" \
  --query id \
  --output tsv)
```

```bash
DCR_ID=$(az monitor data-collection rule show \
  --resource-group "$RG2" \
  --name "dcr-linux-production" \
  --query id \
  --output tsv)
```

Create the association:

```bash
MSYS_NO_PATHCONV=1 az monitor data-collection rule association create \
  --name "production-vm-monitoring" \
  --resource "$VM2_ID" \
  --rule-id "$DCR_ID"
```

Verify:

```bash
MSYS_NO_PATHCONV=1 az monitor data-collection rule association list-by-resource \
  --resource "$VM2_ID" \
  --output table
```

## 8. Verify Azure Monitor Agent Heartbeat

In the Log Analytics workspace, open:

```text
Logs
```

Run:

```kusto
Heartbeat
| where Category == "Azure Monitor Agent"
| where TimeGenerated > ago(2h)
| project TimeGenerated,
          Computer,
          Category,
          Version,
          _ResourceId
| order by TimeGenerated desc
```

Heartbeat data confirms that the Azure Monitor Agent is communicating with Azure Monitor.

## 9. Query Performance Data

```kusto
Perf
| where TimeGenerated > ago(30m)
| project TimeGenerated,
          Computer,
          ObjectName,
          CounterName,
          CounterValue
| order by TimeGenerated desc
```

Average values can be summarized with:

```kusto
Perf
| where TimeGenerated > ago(1h)
| summarize AvgValue=avg(CounterValue)
    by bin(TimeGenerated, 5m), CounterName
```

## 10. Query Linux Syslog

```kusto
Syslog
| where TimeGenerated > ago(1h)
| project TimeGenerated,
          Computer,
          SeverityLevel,
          ProcessName,
          SyslogMessage
| order by TimeGenerated desc
```

Find errors:

```kusto
Syslog
| where TimeGenerated > ago(24h)
| where SeverityLevel in ("err", "crit", "alert", "emerg")
| order by TimeGenerated desc
```

## 11. Simulate a Syslog Incident

Generate a test error:

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts '
logger -p daemon.err "MONITORING-LAB: Production web service health check failed"
'
```

Query the generated event:

```kusto
Syslog
| where SyslogMessage contains "MONITORING-LAB"
```

## 12. Create an Action Group

```bash
ACTION_GROUP2="ag-production-operations"
ALERT_EMAIL="your-email@example.com"
```

```bash
az monitor action-group create \
  --resource-group "$RG2" \
  --name "$ACTION_GROUP2" \
  --short-name "ProdOps" \
  --action email Operations "$ALERT_EMAIL" usecommonalertschema
```

## 13. Create a CPU Alert

```bash
ACTION_GROUP2_ID=$(az monitor action-group show \
  --resource-group "$RG2" \
  --name "$ACTION_GROUP2" \
  --query id \
  --output tsv)
```

Create the alert:

```bash
az monitor metrics alert create \
  --resource-group "$RG2" \
  --name "production-high-cpu" \
  --scopes "$VM2_ID" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action "$ACTION_GROUP2_ID" \
  --description "Production VM CPU above 80 percent."
```

## 14. Simulate High CPU

Install and run `stress-ng`:

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts "
sudo apt-get update -y
sudo apt-get install stress-ng -y
nohup stress-ng --cpu 2 --timeout 10m >/tmp/cpu-test.log 2>&1 &
"
```

The CPU load should cause Azure Monitor metrics to increase and may trigger the configured alert.

## 15. Build an Azure Workbook

A basic workbook was created using Azure Monitor Workbooks.

The workbook contains:

- Performance trends
- Syslog events by severity
- VM heartbeat status

Example performance query:

```kusto
Perf
| where TimeGenerated > ago(1h)
| summarize AvgValue=avg(CounterValue)
    by bin(TimeGenerated, 5m), CounterName
```

Example Syslog query:

```kusto
Syslog
| where TimeGenerated > ago(24h)
| summarize Events=count()
    by SeverityLevel
```

Example Heartbeat query:

```kusto
Heartbeat
| summarize LastHeartbeat=max(TimeGenerated)
    by Computer
```

## Troubleshooting

### No Heartbeat Data

Verify that the VM has a managed identity:

```bash
az vm identity show \
  --resource-group "$RG2" \
  --name "$VM2"
```

Verify Azure Monitor Agent:

```bash
az vm extension show \
  --resource-group "$RG2" \
  --vm-name "$VM2" \
  --name AzureMonitorLinuxAgent
```

Check the agent service:

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts "
sudo systemctl status azuremonitoragent --no-pager
"
```

### DCR Association Error

If Git Bash modifies Azure resource IDs, use:

```bash
MSYS_NO_PATHCONV=1
```

before the Azure CLI command.

Example:

```bash
MSYS_NO_PATHCONV=1 az monitor data-collection rule association create \
  --name "production-vm-monitoring" \
  --resource "$VM2_ID" \
  --rule-id "$DCR_ID"
```

### Perf Table Is Empty

Verify the DCR includes:

```text
Microsoft-Perf
```

and confirm the DCR is associated with the VM.

```bash
MSYS_NO_PATHCONV=1 az monitor data-collection rule association list-by-resource \
  --resource "$VM2_ID"
```

### Syslog Table Is Empty

Generate a test error:

```bash
az vm run-command invoke \
  --resource-group "$RG2" \
  --name "$VM2" \
  --command-id RunShellScript \
  --scripts '
logger -p daemon.err "MONITORING TEST ERROR"
'
```

Then query:

```kusto
Syslog
| where SyslogMessage contains "MONITORING TEST ERROR"
```

## Skills Learned

This project provided hands-on experience with:

- Azure Monitor
- Azure Monitor Agent
- Log Analytics
- Data Collection Rules
- KQL
- Linux performance monitoring
- Syslog monitoring
- Metric alerts
- Action Groups
- Azure Workbooks
- Incident simulation
- Azure CLI
- Monitoring troubleshooting

## Azure DevOps Integration

Azure Monitor can integrate with several Azure DevOps technologies.

```text
GitHub / Azure Repos
        |
        v
Azure Pipelines
        |
        v
Application Deployment
        |
        v
Azure Monitor
        |
   +----+----+
   |         |
 Metrics    Logs
   |         |
   +----+----+
        |
        v
Alerts / Workbooks
```

Monitoring can be used after deployments to verify application and infrastructure health.

It can also integrate with:

- Terraform for Monitoring as Code
- AKS and Container Insights
- Application Insights
- Microsoft Sentinel
- Defender for Cloud
- Azure Policy
- Logic Apps
- Azure Automation

## Cleanup

Delete the project resource group when the lab is complete:

```bash
az group delete \
  --name "$RG2" \
  --yes \
  --no-wait
```

## Conclusion

This project demonstrated how to build a basic production-style Linux monitoring solution using Azure Monitor.

The project covered the full monitoring workflow:

```text
Collect
   ↓
Store
   ↓
Query
   ↓
Visualize
   ↓
Alert
   ↓
Respond
```

It provides a foundation for more advanced monitoring and security technologies such as AKS monitoring, Application Insights, Microsoft Sentinel, Defender for Cloud, and automated incident response.