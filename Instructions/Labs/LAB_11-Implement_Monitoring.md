---
lab:
  title: "Lab 11: Implement Monitoring - werkende versie met Heartbeat"
  module: Administer Monitoring
  description: "Configure Azure Monitor, VM insights, Log Analytics queries, alerts, action groups, and alert processing rules."
  duration: 60 minutes
  level: 300
---

# Lab 11 - Implement Monitoring

## Lab introduction

In this lab you deploy a Windows virtual machine, enable Azure Monitor for VM guest telemetry, verify `Heartbeat` data in Log Analytics, and then create and test Azure Monitor alerts.

This version makes the monitoring dependency explicit. The original flow can leave students without a Log Analytics workspace, Azure Monitor Agent, or Data Collection Rule association, which causes the `Heartbeat` and `InsightsMetrics` queries to return no useful result. In this version, students verify monitoring before deleting the virtual machine.

## Estimated timing

60 minutes

## Lab scenario

Your organization has migrated infrastructure to Azure. Administrators must be able to monitor virtual machines, query collected telemetry, and receive notifications when important changes occur.

## Prerequisites

- An Azure subscription where you can create resources.
- Permission to create resource groups, virtual machines, Log Analytics workspaces, Data Collection Rules, alert rules, and action groups.
- The lab ARM template: `az104-11-vm-template.json`.
- Recommended region: `East US`. If quota fails, use another region consistently throughout the lab.
- Use a password that meets Azure VM complexity requirements.

## Important design note

Do not delete `az104-vm0` until after the Log Analytics verification task. `Heartbeat` is produced by the Azure Monitor Agent while the VM is running and associated with a Data Collection Rule. If the VM is deleted before monitoring is fully onboarded, students can end up with no visible heartbeat data.

## Job skills

- Task 1: Deploy the lab infrastructure.
- Task 2: Create a Log Analytics workspace.
- Task 3: Enable VM insights and Azure Monitor Agent for the VM.
- Task 4: Verify Heartbeat and InsightsMetrics data.
- Task 5: Create an alert and action group.
- Task 6: Configure an alert processing rule.
- Task 7: Trigger the alert and confirm it is working.
- Task 8: Clean up resources.

---

# Task 1: Deploy the lab infrastructure

In this task, you deploy the VM and network resources used in the monitoring scenarios.

1. Sign in to the Azure portal:

   `https://portal.azure.com`

1. Search for and select **Deploy a custom template**.

1. Select **Build your own template in the editor**.

1. Select **Load file**.

1. Select the file `az104-11-vm-template.json`.

1. Select **Save**.

1. Complete the deployment with these values.

   | Setting | Value |
   | --- | --- |
   | Subscription | Your Azure subscription |
   | Resource group | `az104-rg11` |
   | Region | `East US` or another region if quota requires it |
   | Username | `localadmin` |
   | Password | A complex password |

1. Select **Review + create**.

1. Select **Create**.

1. Wait until the deployment succeeds.

1. Select **Go to resource group**.

1. Confirm that the resource group contains at least:

   - One virtual machine named `az104-vm0`
   - One virtual network
   - One network interface
   - One public IP address
   - One network security group
   - One storage account for boot diagnostics

1. Open the VM `az104-vm0`.

1. Confirm that the VM status is **Running**. If it is stopped, select **Start** and wait until it is running.

## Checkpoint

The VM `az104-vm0` exists and is running.

---

# Task 2: Create a Log Analytics workspace

In this task, you create the workspace that will receive VM guest telemetry.

1. In the Azure portal, search for and select **Log Analytics workspaces**.

1. Select **Create**.

1. Use these values.

   | Setting | Value |
   | --- | --- |
   | Subscription | Your Azure subscription |
   | Resource group | `az104-rg11` |
   | Name | `az104-law11` |
   | Region | Same region as the VM, for example `East US` |

1. Select **Review + create**.

1. Select **Create**.

1. Wait until deployment finishes.

1. Open the workspace `az104-law11`.

1. In **Overview**, confirm the workspace exists and is in a **Succeeded** state.

## Checkpoint

The resource group `az104-rg11` now contains a Log Analytics workspace named `az104-law11`.

---

# Task 3: Enable VM insights and Azure Monitor Agent

In this task, you connect the VM to the workspace and install the Azure Monitor Agent. This creates or associates a Data Collection Rule.

> Trainer note: Azure portal wording changes over time. The key outcome is not the exact button text. The key outcome is: Azure Monitor Agent installed, VM associated with a VM insights Data Collection Rule, and data sent to `az104-law11`.

1. In the Azure portal, search for and select **Monitor**.

1. In the left menu, select **Virtual Machines**.

1. Select the **Not monitored** tab or equivalent view.

1. Find `az104-vm0`.

1. Select **Configure** or **Enable** for `az104-vm0`.

1. Choose **Azure Monitor Agent** if the portal asks for an agent type.

1. On the **Capabilities** page, select **Customize infrastructure monitoring** if the monitoring options are not already visible.

1. In **Enable detailed metrics**, configure the options exactly as follows.

   | Setting | Value |
   | --- | --- |
   | OpenTelemetry metrics | Unchecked |
   | `[Classic] Log-based metrics` | Checked |
   | Log Analytics workspace | `az104-law11` |

   > Important: Do not leave **OpenTelemetry metrics** selected for this lab. OpenTelemetry metrics are sent to an Azure Monitor workspace, such as `defaultazuremonitorworkspace-*`. The `Heartbeat` query in this lab uses Log Analytics tables, so the VM must send classic log-based data to `az104-law11`.

1. When asked for a Data Collection Rule, select **Create new**. If the portal automatically proposes a new DCR, you can use it as long as the selected Log Analytics workspace is `az104-law11`.

1. Use these values for the Data Collection Rule.

   | Setting | Value |
   | --- | --- |
   | Rule name | `MSVMI-az104-law11` |
   | Subscription | Your Azure subscription |
   | Resource group | `az104-rg11` |
   | Region | Same region as the VM |
   | Destination workspace | `az104-law11` |
   | Performance collection | Enabled |
   | Processes and dependencies / Map | Optional. Not required for this lab |

1. In **Alerts**, turn **Enable recommended alerts** off. You will create the delete alert manually later in this lab.

1. Select **Review + enable**, **Configure**, or **Enable**.

1. Wait until the configuration completes.

1. Stay on the VM insights page and wait a few minutes. The portal may need 5 to 15 minutes before data appears.

## Alternative portal path

If you cannot find the VM insights onboarding page:

1. Open `az104-vm0`.
1. In the VM menu, select **Insights**.
1. Select **Enable** or **Configure monitoring**.
1. Choose workspace `az104-law11`.
1. Enable monitoring with Azure Monitor Agent.

## Validation in the portal

1. Open `az104-vm0`.

1. Select **Extensions + applications**.

1. Confirm that an Azure Monitor Agent extension is installed. For a Windows VM, it is commonly shown as `AzureMonitorWindowsAgent`.

1. In the Azure portal, search for and select **Data Collection Rules**.

1. Open `MSVMI-az104-law11` or the DCR created by the portal.

1. Confirm that `az104-vm0` is listed as an associated resource.

## Optional validation with Azure CLI

Run these commands from Cloud Shell or a local shell authenticated to the subscription.

```bash
az vm extension list \
  --resource-group az104-rg11 \
  --vm-name az104-vm0 \
  --query "[].{name:name,publisher:publisher,type:type,state:provisioningState}" \
  --output table
```

Expected: an Azure Monitor Agent extension is listed.

```bash
VM_ID=$(az vm show -g az104-rg11 -n az104-vm0 --query id -o tsv)
az monitor data-collection rule association list \
  --resource "$VM_ID" \
  --output table
```

Expected: at least one DCR association is listed.

## Checkpoint

The VM has Azure Monitor Agent installed and is associated with a Data Collection Rule that sends data to `az104-law11`.

---

# Task 4: Verify Heartbeat and InsightsMetrics data

In this task, you verify that Log Analytics receives data from the VM. This task is intentionally before the delete-alert test.

1. In the Azure portal, search for and select **Log Analytics workspaces**.

1. Open `az104-law11`.

1. Select **Logs**.

1. Close the welcome or sample-query screen if it appears.

1. Set the query editor to **KQL mode** if needed.

1. Run this query.

```kql
Heartbeat
| where TimeGenerated > ago(30m)
| summarize Count=count(), Latest=max(TimeGenerated) by Computer
| order by Latest desc
```

1. Confirm that the result contains a row for `az104-vm0`.

1. If no rows appear, wait 5 minutes and run the query again.

1. If still no rows appear after 15 minutes, run this diagnostic query.

```kql
search *
| where TimeGenerated > ago(30m)
| summarize Rows=count() by Type
| order by Rows desc
```

1. Confirm whether any data types are being ingested.

1. Run this performance query.

```kql
InsightsMetrics
| where TimeGenerated > ago(30m)
| summarize Count=count(), Latest=max(TimeGenerated) by Computer, Namespace, Name
| order by Latest desc
```

1. If rows are returned, run the chart query.

```kql
InsightsMetrics
| where TimeGenerated > ago(30m)
| where Name == "UtilizationPercentage"
| summarize AvgValue=avg(Val) by bin(TimeGenerated, 5m), Computer
| render timechart
```

## If Heartbeat does not appear

Check these items before continuing:

- `az104-vm0` is running.
- The VM agent status is ready.
- Azure Monitor Agent extension is installed.
- The VM is associated with the VM insights DCR.
- The DCR destination is workspace `az104-law11`.
- You are querying workspace `az104-law11`, not the subscription scope or a different workspace.
- You waited at least 5 to 15 minutes after enabling VM insights.

## Checkpoint

The `Heartbeat` query returns at least one row for `az104-vm0`.

---

# Task 5: Create an alert and action group

In this task, you create an alert for a virtual machine delete event and configure email notification.

1. In the Azure portal, search for and select **Monitor**.

1. Select **Alerts**.

1. Select **Create** and then **Alert rule**.

1. On the **Scope** tab, select your subscription.

1. Select **Apply**.

1. On the **Condition** tab, select **See all signals**.

1. Search for and select **Delete Virtual Machine (Virtual Machines)**.

1. Select **Apply**.

1. Leave the default event level and status selections.

1. Open the **Actions** tab.

1. Select **Use action groups**.

1. Select **Create action group**.

1. On the **Basics** tab, use these values.

   | Setting | Value |
   | --- | --- |
   | Subscription | Your Azure subscription |
   | Resource group | `az104-rg11` |
   | Region | `Global` |
   | Action group name | `Alert the operations team` |
   | Display name | `AlertOpsTeam` |

1. Select **Next: Notifications**.

1. Add a notification with these values.

   | Setting | Value |
   | --- | --- |
   | Notification type | Email/SMS message/Push/Voice |
   | Name | `VM was deleted` |
   | Email | Your email address |

1. Select **OK**.

1. Select **Review + create**.

1. Select **Create**.

1. Return to the alert rule.

1. On the **Details** tab, use these values.

   | Setting | Value |
   | --- | --- |
   | Resource group | `az104-rg11` |
   | Severity | `Sev 3` |
   | Alert rule name | `VM was deleted` |
   | Alert rule description | `A VM in the subscription was deleted` |

1. Select **Review + create**.

1. Select **Create**.

1. Wait until the alert rule is created before continuing.

## Checkpoint

The alert rule `VM was deleted` and the action group `Alert the operations team` exist.

---

# Task 6: Configure an alert processing rule

In this task, you create an alert processing rule that suppresses notifications during a planned maintenance window.

1. In **Monitor**, select **Alerts**.

1. Select **Alert processing rules**.

1. Select **Create**.

1. Select your subscription as the scope.

1. Select **Apply**.

1. Select **Next: Rule settings**.

1. Select **Suppress notifications**.

1. Select **Next: Scheduling**.

1. Configure a maintenance window.

   | Setting | Value |
   | --- | --- |
   | Apply the rule | At a specific time |
   | Start | Today at 10:00 PM |
   | End | Tomorrow at 7:00 AM |
   | Time zone | Your local time zone |

1. Select **Next: Details**.

1. Use these values.

   | Setting | Value |
   | --- | --- |
   | Resource group | `az104-rg11` |
   | Rule name | `Planned Maintenance` |
   | Description | `Suppress notifications during planned maintenance.` |

1. Select **Review + create**.

1. Select **Create**.

## Checkpoint

The alert processing rule `Planned Maintenance` exists.

---

# Task 7: Trigger the delete alert and confirm it is working

In this task, you delete the VM after monitoring has already been verified.

> Important: Complete Task 4 before starting this task. Deleting the VM too early is the most common reason the log query task has no visible result.

1. In the Azure portal, search for and select **Virtual machines**.

1. Select `az104-vm0`.

1. Select **Delete**.

1. Confirm the deletion when prompted.

1. Wait until the VM deletion succeeds.

1. In the Azure portal, open **Monitor**.

1. Select **Alerts**.

1. Wait a few minutes and refresh the alerts list.

1. Confirm that the alert rule `VM was deleted` fired.

1. Check your email for the Azure Monitor alert notification.

## If the alert does not appear

- Confirm the alert rule finished deploying before the VM was deleted.
- Confirm the scope is the subscription or a scope that includes `az104-vm0`.
- Confirm the signal is **Delete Virtual Machine (Virtual Machines)**.
- Wait several minutes; activity log alerts are not always immediate.
- Confirm your alert processing rule is not currently suppressing notifications.

## Checkpoint

The VM delete alert appears in Azure Monitor, and the notification email is received unless it was suppressed by the processing rule.

---

# Task 8: Clean up resources

If you are using your own subscription, delete the lab resources to avoid cost.

1. In the Azure portal, open resource group `az104-rg11`.

1. Select **Delete resource group**.

1. Enter the resource group name to confirm.

1. Select **Delete**.

Optional Azure CLI cleanup:

```bash
az group delete --name az104-rg11 --yes --no-wait
```

---

# Trainer preflight checklist

Use this checklist before class.

1. Confirm the target region has VM quota for `Standard_D2s_v3`.
1. Confirm students have permissions for:
   - Resource group creation
   - VM deployment
   - Log Analytics workspace creation
   - Data Collection Rule creation
   - Role assignments are not normally needed for this lab, but students must be able to enable monitoring on a VM
   - Alert rule and action group creation
1. Confirm the portal can enable VM insights with Azure Monitor Agent.
1. Confirm student tenants do not block Azure Monitor Agent extension deployment by policy.
1. Confirm outbound connectivity from the VM is not blocked. The template's NSG only controls inbound RDP; default outbound connectivity should remain available.
1. Tell students not to delete the VM until after the Heartbeat checkpoint.

---

# Known-good validation commands

These commands are optional but useful for trainer troubleshooting.

```bash
az account show --query "{subscription:name,user:user.name}" -o table
```

```bash
az resource list -g az104-rg11 --query "[].{name:name,type:type,location:location}" -o table
```

```bash
az monitor log-analytics workspace show \
  --resource-group az104-rg11 \
  --workspace-name az104-law11 \
  --query "{name:name,customerId:customerId,provisioningState:provisioningState}" \
  --output table
```

```bash
az vm get-instance-view \
  --resource-group az104-rg11 \
  --name az104-vm0 \
  --query "{power:instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus | [0], vmAgent:instanceView.vmAgent.statuses[0].displayStatus}" \
  --output table
```

```bash
az vm extension list \
  --resource-group az104-rg11 \
  --vm-name az104-vm0 \
  --query "[].{name:name,publisher:publisher,type:type,state:provisioningState}" \
  --output table
```

```bash
VM_ID=$(az vm show -g az104-rg11 -n az104-vm0 --query id -o tsv)
az monitor data-collection rule association list --resource "$VM_ID" -o table
```

```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show -g az104-rg11 -n az104-law11 --query customerId -o tsv)
az monitor log-analytics query \
  --workspace "$WORKSPACE_ID" \
  --analytics-query "Heartbeat | where TimeGenerated > ago(30m) | summarize Count=count(), Latest=max(TimeGenerated) by Computer | order by Latest desc" \
  --output table
```

---

# Why this version works

Azure Monitor automatically has platform metrics and activity logs for Azure VMs, but guest OS telemetry such as heartbeat and VM insights performance data requires extra configuration. For this lab, the required chain is:

1. Log Analytics workspace exists.
1. Azure Monitor Agent is installed on `az104-vm0`.
1. A Data Collection Rule exists for VM insights.
1. The Data Collection Rule is associated with `az104-vm0`.
1. The DCR sends data to the selected Log Analytics workspace.
1. Students query that same workspace.

If any link is missing, `Heartbeat` and `InsightsMetrics` can be empty.

---

# Suggested instructor explanation

"Azure Monitor has two layers here. The VM resource itself always has platform metrics and activity log events. But Task 4 uses Log Analytics tables, which are guest telemetry. Guest telemetry needs Azure Monitor Agent plus a Data Collection Rule that sends data to a Log Analytics workspace. That is why we explicitly create the workspace and verify the DCR association before testing the delete alert."

---

# Reference links

- Original AZ-104 lab: https://github.com/MicrosoftLearning/AZ-104-MicrosoftAzureAdministrator/blob/master/Instructions/Labs/LAB_11-Implement_Monitoring.md
- Monitor virtual machines with Azure Monitor - data collection: https://learn.microsoft.com/azure/azure-monitor/vm/monitor-virtual-machine-data-collection
- Collect guest log data from virtual machines with Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/vm/data-collection
- Monitor Azure virtual machines overview: https://learn.microsoft.com/azure/virtual-machines/monitor-vm
