# Context Graph for Terraform Drift

> **`terraform plan` tells you WHAT drifted. Context graphs tell you WHY it was there, WHO put it there, and WHAT BREAKS if you "fix" it.**

---

## The Problem

Terraform drift happens every day in big orgs:

1. Someone changes a resource in the Azure Portal (bypassing TF)
2. Someone runs `terraform apply` with the wrong version
3. An automated process modifies infrastructure outside TF
4. Emergency hotfix changes are never backported to TF state

When you run `terraform plan`, you see:

```
~ resource_group_name = "rg-old" → "rg-new"
~ node_count          = 3 → 5
~ sku_name            = "Standard" → "Premium"
```

**Three lines. Zero context.**

- Why was it `rg-old`? Was that intentional?
- Who changed it to `rg-new`? When?
- Was there an incident that caused this?
- What depends on `rg-old` that will break?
- Is this drift... or is the TF state wrong?

**You're guessing.**

---

## The Solution

**Context Graph for Terraform Drift** — Every Terraform resource has a living context node that tracks:

1. **Why it exists** — the original decision and rationale
2. **Who created it** — person, team, ticket
3. **What changed** — full change history with reasons
4. **What depends on it** — dependency chain
5. **What happens if it changes** — blast radius analysis
6. **Incident history** — every incident that involved this resource

When drift is detected, you don't just see the diff. You see the **full story**.

---

## Example

### Without Context Graph (just `terraform plan`)

```
# terraform plan output
~ azurerm_kubernetes_cluster.prod
    default_node_pool.0.node_count = 3 → 5

Plan: 0 to add, 1 to change, 0 to destroy.
```

Engineer thinks: "Someone scaled it up. I should scale it back to match TF state."

### With Context Graph

```
# terraform plan + context graph
~ azurerm_kubernetes_cluster.prod
    default_node_pool.0.node_count = 3 → 5

📋 CONTEXT:
  Changed: 2026-04-12 13:15 IST by engineer-a
  Reason:  Terraform apply (autoscaling min=3→5)
  Ticket:  INC-001 (Node NotReady + Disk Pressure)
  Decision: DEC-TF-005 — Increase min nodes from 3 to 5
            Rationale: 3 nodes insufficient during disk pressure events
            Precedent: INC-001 (3 NotReady nodes caused 14 pod failures)
            Approved by: SRE lead

⚠️  BLAST RADIUS:
  • 2 additional nodes running ($2,400/month)
  • All services on prod-cluster benefit from higher availability
  • No services will break if reverted to 3

⚠️  WARNING:
  • Reverting to 3 nodes risks repeating INC-001
  • INC-004 also involved insufficient node count
  • TF state is BEHIND reality — state should be updated, not reverted

💡 RECOMMENDATION:
  Update TF state to match current reality (node_count=5)
  Do NOT revert the change.
```

**Same diff. Completely different decision.**

---

## How It Works

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Terraform State ──→ Context Graph Builder ──→ Nodes    │
│       │                    │                    │        │
│       │                    │                    │        │
│  Git History ──────────────┘                    │        │
│       │                                         │        │
│  Incident Replays ─────────────────────────────┘        │
│       │                                                  │
│  Change Correlator ────────────────────────────┘        │
│                                                         │
│  terraform plan ──→ Drift Detector ──→ Context Query    │
│                                          │               │
│                                          ↓               │
│                                    Drift Brief           │
│                                    (Markdown)            │
└─────────────────────────────────────────────────────────┘
```

### Node Schema

Every Terraform resource becomes a context node:

```markdown
---
id: CG-TF-aks-prod-cluster
resource_type: azurerm_kubernetes_cluster
resource_name: prod-cluster
resource_group: rg-production
tf_file: main.tf
tf_module: aks
created: 2025-10-15
created_by: platform-team
ticket: PROJ-1234
monthly_cost: $12,000
---

# prod-cluster (AKS)

## Why It Exists
Production Kubernetes cluster for CAMARA API services.
Provisioned as part of the platform migration from ACS.

## Owners
- **Primary:** platform-team
- **On-call:** sre-oncall
- **Approver:** sre-lead

## Current Configuration
- Location: East US 2
- Node count: 5 (min) / 10 (max)
- SKU: Standard_D4s_v5
- Kubernetes version: 1.28.0
- Network plugin: Azure CNI

## Change History
| Date | What Changed | Who | Why | Ticket |
|------|-------------|-----|-----|--------|
| 2025-10-15 | Created (3 nodes) | platform-team | Initial provisioning | PROJ-1234 |
| 2026-01-10 | k8s upgraded 1.27→1.28 | engineer-a | Security patch | INC-004 |
| 2026-03-05 | autoscaling enabled | engineer-b | Cost optimization | PROJ-2345 |
| 2026-04-12 | node_count 3→5 | engineer-a | INC-001 disk pressure | DEC-TF-005 |

## Dependencies
### This resource is depended on by:
- deployment/payment-api (3 replicas)
- deployment/camara-gateway (5 replicas)
- deployment/monitoring-stack (Prometheus + Grafana)
- ingress/nginx-ingress-controller
- cert-manager/letsencrypt-prod

### This resource depends on:
- azurerm_resource_group.rg-production
- azurerm_virtual_network.vnet-prod
- azurerm_subnet.aks-subnet
- azurerm_log_analytics_workspace.logs

## Incident History
| Incident | What Happened | Impact on This Resource |
|----------|-------------|----------------------|
| INC-001 | Disk pressure, 3 nodes NotReady | node_count increased 3→5 |
| INC-004 | Runtime restart cascading failures | k8s version upgraded |
| INC-006 | Provider timeout | Autoscaling enabled |

## Known Risks
- **Disk accumulation:** Images build up every ~90 days without automated prune (INC-001)
- **Node pool scaling:** Min=5 may be insufficient during peak traffic (10:00-14:00 IST)
- **Version skew:** k8s 1.28 approaching EOL — upgrade needed by Q3 2026

## Drift Sensitivity
- **node_count:** HIGH — direct impact on availability
- **sku_name:** HIGH — cost + performance impact
- **kubernetes_version:** MEDIUM — requires careful upgrade process
- **network_plugin:** CRITICAL — changing this requires cluster rebuild
```

---

## Project Structure

```
context-graph-terraform/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── BLOG.md
├── nodes/                          # Context graph nodes (one per resource)
│   ├── aks-prod-cluster.md
│   ├── aks-dev-cluster.md
│   ├── vnet-production.md
│   ├── rg-production.md
│   ├── storage-logs.md
│   └── keyvault-prod.md
├── drift/                          # Drift briefs (generated per plan)
│   ├── 2026-04-12-node-count.md
│   ├── 2026-04-15-sku-change.md
│   └── template.md
├── decisions/                      # TF-related decision traces
│   ├── DEC-TF-001-node-count.md
│   ├── DEC-TF-002-sku-upgrade.md
│   ├── DEC-TF-005-autoscaling.md
│   └── template.md
├── dependencies/                   # Dependency maps
│   ├── prod-cluster-deps.md
│   └── full-stack-deps.md
├── templates/                      # Templates for new nodes
│   ├── resource-node.md
│   ├── drift-brief.md
│   └── decision-trace.md
├── samples/                        # Sample data
│   └── full-drift-scenario/
│       ├── terraform-plan.txt
│       ├── drift-brief.md
│       ├── context-query.md
│       └── recommendation.md
└── queries/                        # Common context queries
    ├── why-does-this-exist.md
    ├── what-depends-on-this.md
    ├── who-changed-this.md
    └── what-happens-if.md
```

---

## Drift Brief Template

When `terraform plan` shows drift, generate a drift brief:

```markdown
# Drift Brief: [DATE]

## Terraform Plan Diff

[Paste terraform plan output]

## Context

### Resource: [resource-name]
- **Type:** [resource-type]
- **Node:** [link to context node]
- **Owner:** [who owns it]
- **Monthly Cost:** [if known]

### Current State (from context graph)
[Last known configuration and why it's that way]

### Drift Analysis
| Attribute | TF State | Reality | Risk |
|-----------|----------|---------|------|
| [attr] | [val] | [val] | HIGH/MED/LOW |

### Change History
[Recent changes from context node]

### Incident History
[Any incidents involving this resource]

### Dependencies
[What depends on this resource]

## Recommendation

**Action:** [Update TF state | Revert change | Investigate further]

**Reasoning:**
[Why this is the right call, based on context]

**Risk if Reverted:**
[What breaks if you revert to TF state]

**Risk if Accepted:**
[What happens if you accept the drift and update TF]

---

*Generated by Context Graph for Terraform Drift*
```

---

## Common Queries

### "Why does this resource exist?"
```bash
grep -A20 "^## Why It Exists" nodes/[resource].md
```

### "Who changed this last?"
```bash
grep -A5 "^## Change History" nodes/[resource].md
```

### "What depends on this?"
```bash
grep -A10 "^### This resource is depended on by" nodes/[resource].md
```

### "What happens if I change/delete this?"
```bash
grep -A10 "^## Known Risks" nodes/[resource].md
grep -A10 "^### This resource is depended on by" nodes/[resource].md
```

### "Has this resource been involved in incidents?"
```bash
grep -A5 "^## Incident History" nodes/[resource].md
```

---

## Integration with Existing Stack

### From Context Graph SRE
- Decision traces feed into TF resource nodes
- Past incidents linked to specific resources

### From Incident Replay Engine
- Incident replays reference specific TF resources
- TF resource nodes link back to incident replays

### From Change Correlator
- TF applies are a change source
- Correlation analysis includes TF changes

### From Crisis Command Center
- Drift briefs shown during active incidents
- Context graph queries available in dashboard

---

## Roadmap

**Phase 1 (MVP):**
- Resource node schema and templates
- Sample nodes for common Azure resources (AKS, VNet, RG, Storage)
- Drift brief template
- Manual context query examples

**Phase 2:**
- Auto-generate nodes from `terraform state show`
- Auto-generate drift briefs from `terraform plan`
- Dependency graph from `terraform graph`
- Cost attribution from Azure Cost Management

**Phase 3:**
- Git history integration (auto-populate change history)
- Incident replay cross-reference
- Weekly drift report generation
- Compliance audit export

**Phase 4:**
- Blast radius simulation ("what if I change this?")
- Drift risk scoring (some drift is dangerous, some is harmless)
- Predictive drift (resources likely to drift based on patterns)

---

## Why This Matters

**For Manav:**
- He manages Terraform daily — AKS, VNets, resource groups
- Drift happens and causes confusion
- Nobody knows WHY resources are configured the way they are
- Auditors ask questions that take days to answer

**For SRE teams:**
- Drift is scary because you don't know the blast radius
- Context removes the fear
- "Fixing" drift becomes an informed decision, not a guess

**For the ecosystem:**
- `terraform plan` shows WHAT. Context graph shows WHY.
- Together: complete infrastructure intelligence.

---

## Philosophy

> **Infrastructure without context is just config. Infrastructure with context is knowledge.**

Every resource has a story. This project captures those stories.
