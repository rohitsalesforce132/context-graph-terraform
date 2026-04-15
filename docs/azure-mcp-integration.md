# Azure MCP Integration

> **Live infrastructure context** — pull resource state directly from Azure using Model Context Protocol.

---

## Overview

Instead of manually writing context graph nodes, Azure MCP queries your live Azure environment and auto-generates context nodes with real data.

```
Azure MCP ──→ Query Live State ──→ Generate Context Nodes ──→ Compare with TF State ──→ Drift Brief
```

## Setup

### Prerequisites

```bash
# 1. Azure CLI (already installed)
az --version  # v2.85.0

# 2. mcporter (already installed)
mcporter --version  # v0.7.3

# 3. Authenticate with Azure
az login
az account set --subscription "your-subscription-id"

# 4. Install Azure MCP server
npm install -g @azure/mcp
```

### Configure mcporter

Add Azure MCP to your mcporter config:

```bash
mcporter config add azure --command "npx @azure/mcp" --transport stdio
```

Or manually create `config/mcporter.json`:

```json
{
  "servers": {
    "azure": {
      "command": "npx",
      "args": ["@azure/mcp"],
      "transport": "stdio"
    }
  }
}
```

### Verify

```bash
# List available Azure MCP tools
mcporter list azure --schema

# Test a query
mcporter call azure.resources_list
```

---

## Available Azure MCP Tools

### Resource Queries

```bash
# List all resources in a resource group
mcporter call azure.resources_list --args '{"resourceGroupName": "rg-production"}'

# Get specific resource details
mcporter call azure.resources_show --args '{"resourceId": "/subscriptions/.../resourceGroups/rg-production/providers/Microsoft.ContainerService/managedClusters/prod-cluster"}'

# List all resource groups
mcporter call azure.resourcegroups_list
```

### AKS Queries

```bash
# List all AKS clusters
mcporter call azure.aks_list

# Get cluster details
mcporter call azure.aks_show --args '{"resourceGroupName": "rg-production", "clusterName": "prod-cluster"}'

# List node pools
mcporter call azure.aks_nodepool_list --args '{"resourceGroupName": "rg-production", "clusterName": "prod-cluster"}'
```

### Monitor Queries

```bash
# Get activity logs (who changed what)
mcporter call azure.monitor_activitylog_list --args '{"resourceGroupName": "rg-production"}'

# Get metrics
mcporter call azure.monitor_metrics_list --args '{"resourceUri": "/subscriptions/.../providers/Microsoft.ContainerService/managedClusters/prod-cluster"}'
```

### Cost Queries

```bash
# Get cost data
mcporter call azure.costmanagement_query --args '{"scope": "/subscriptions/...", "timeframe": "MonthToDate"}'
```

---

## Auto-Generating Context Nodes

### Process

1. **Query Azure** for resource state
2. **Compare** with existing context node (if any)
3. **Generate** or **update** context node with live data
4. **Flag drift** if Azure state ≠ TF state ≠ context node

### Node Generation from Azure

For each resource in a resource group:

```bash
# Step 1: List all resources
mcporter call azure.resources_list --args '{"resourceGroupName": "rg-production"}' --output json

# Step 2: For each resource, get details
mcporter call azure.resources_show --args '{"resourceId": "<resource-id>"}' --output json

# Step 3: Get activity log for recent changes
mcporter call azure.monitor_activitylog_list --args '{"resourceGroupName": "rg-production", "caller": "any"}' --output json

# Step 4: Generate context node from the collected data
```

### Generated Node Format

Azure MCP data maps directly to context node fields:

| Azure MCP Field | Context Node Field |
|----------------|-------------------|
| `name` | `resource_name` |
| `type` | `resource_type` |
| `resourceGroup` | `resource_group` |
| `location` | `location` |
| `tags.owner` | `owners.primary` |
| `tags.costCenter` | `cost_center` |
| `createdTime` | `created` |
| `changedTime` | `last_changed` |

### Example: Auto-Generated AKS Node

```markdown
---
id: CG-TF-aks-prod-cluster
resource_type: azurerm_kubernetes_cluster
resource_name: prod-cluster
resource_group: rg-production
location: eastus2
created: 2025-10-15T00:00:00Z
last_changed: 2026-04-12T13:15:00Z
source: azure-mcp
live: true
---

# prod-cluster (AKS) — Live from Azure

## Live Configuration (from Azure MCP)
- Location: East US 2
- Node count: 5 (current)
- SKU: Standard_D4s_v5
- Kubernetes version: 1.28.0
- Status: Running
- Provisioning State: Succeeded

## TF State (from terraform state)
- Node count: 3 (min)
- SKU: Standard_D4s_v5
- Kubernetes version: 1.28.0

## ⚠️ Drift Detected
- node_count: TF=3, Azure=5
- See: drift/2026-04-12-node-count.md

## Activity Log (last 7 days)
[Auto-populated from Azure Monitor activity log]

## Tags
- owner: platform-team
- cost-center: CC-PROD-001
- environment: production
```

---

## Drift Detection with Azure MCP

### Automated Flow

```
Every 6 hours (or on-demand):

1. Azure MCP → Query all resource states
2. Terraform → terraform state show (or terraform plan)
3. Context Graph → Read existing nodes
4. Compare: Azure vs TF vs Context Graph
5. If mismatch → Generate drift brief
```

### Manual Drift Check

```bash
# 1. Get current Azure state
mcporter call azure.aks_show --args '{"resourceGroupName": "rg-production", "clusterName": "prod-cluster"}' --output json

# 2. Get TF state
terraform state show azurerm_kubernetes_cluster.prod

# 3. Compare
# Look for differences in: node_count, sku, version, tags, network config

# 4. Read context graph for "why"
cat nodes/aks-prod-cluster.md | grep -A20 "Change History"
```

---

## Integration Architecture

```
┌─────────────────────────────────────────────────────────┐
│           CONTEXT GRAPH FOR TERRAFORM DRIFT             │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Azure MCP   │  │ Terraform    │  │ Incident     │  │
│  │ (live state)│  │ State (code) │  │ Replays      │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                │                  │           │
│         └────────────────┼──────────────────┘           │
│                          ↓                              │
│                  ┌──────────────┐                       │
│                  │ Context      │                       │
│                  │ Builder      │                       │
│                  └──────┬───────┘                       │
│                         ↓                               │
│         ┌───────────────┼───────────────┐              │
│         ↓               ↓               ↓              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│   │ Nodes    │   │ Drift    │   │ Decision │         │
│   │ (live)   │   │ Briefs   │   │ Traces   │         │
│   └──────────┘   └──────────┘   └──────────┘         │
└─────────────────────────────────────────────────────────┘
```

---

## Azure MCP Tools Reference

| Tool | What It Does | Context Graph Use |
|------|-------------|-------------------|
| `resources_list` | List all resources | Discover resources to create nodes for |
| `resources_show` | Get resource details | Populate node configuration |
| `resourcegroups_list` | List resource groups | Organize nodes by RG |
| `aks_list` | List AKS clusters | AKS-specific node generation |
| `aks_show` | Get AKS details | AKS configuration + node pools |
| `monitor_activitylog_list` | Activity log | Auto-populate change history |
| `monitor_metrics_list` | Metrics | Health scoring |
| `costmanagement_query` | Cost data | Cost attribution in nodes |

---

## Next Steps

1. **Run `az login`** to authenticate with Azure
2. **Install `@azure/mcp`**: `npm install -g @azure/mcp`
3. **Configure mcporter**: `mcporter config add azure --command "npx @azure/mcp" --transport stdio`
4. **Test**: `mcporter list azure --schema`
5. **Query your resources**: `mcporter call azure.resources_list`
6. **Generate context nodes** from the output
7. **Compare with TF state** for drift detection
