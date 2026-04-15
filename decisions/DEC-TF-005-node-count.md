# DEC-TF-005: Increase AKS Node Count from 3 to 5

**Date:** 2026-04-12
**Resource:** azurerm_kubernetes_cluster.prod
**Attribute:** default_node_pool.min_count
**Change:** 3 → 5
**Decided By:** engineer-a (on-call), approved by sre-lead
**Confidence:** HIGH

---

## What Changed

AKS production cluster minimum node count increased from 3 to 5.

## Why

### The Trigger
INC-001 was a P1 incident where 3 worker nodes went NotReady simultaneously due to disk pressure. With all 3 worker nodes down, there were **zero healthy worker nodes** to run workloads. 14 pods entered CrashLoopBackOff. API error rate spiked to 5.2%.

### The Reasoning
1. Current minimum of 3 nodes provides NO buffer during failures
2. If any nodes go down, there's no capacity to absorb the displaced pods
3. During disk pressure events, multiple nodes can fail simultaneously (not independent failures)
4. 5 minimum nodes means: if 3 go down, 2 remain → critical workloads survive

### The Alternatives Considered

| Alternative | Considered? | Chosen? | Reason |
|------------|-------------|---------|--------|
| Keep min=3 | Yes | No | INC-001 proved 3 is insufficient |
| Increase to 4 | Yes | No | Only 1 node buffer, still risky |
| **Increase to 5** | **Yes** | **Yes** | **2 node buffer, reasonable cost** |
| Increase to 7 | Yes | No | Cost not justified at this time |

### Precedent

| Incident | Node Count | What Happened | Lesson |
|----------|-----------|--------------|--------|
| INC-004 (2026-01-10) | 3 | 2 nodes went down, 1 remained | Barely survived |
| INC-001 (2026-04-12) | 3 | 3 nodes went down, 0 remained | Complete outage |
| **DEC-TF-005** | **5** | **2 node buffer** | **Prevents repeat** |

## Cost Impact

- Additional 2 nodes: 2 × Standard_D4s_v5 = 2 × $1,600 = **$3,200/month**
- Justified by: avoided incident cost (INC-001 estimated $XX,XXX in revenue impact)

## Blast Radius

**Affected workloads:** All production workloads (positive impact — more capacity)
**Cost impact:** +$3,200/month
**No negative impact:** This is an addition, not a removal

## Approval

- **Proposed by:** engineer-a (on-call during INC-001)
- **Approved by:** sre-lead (verbal approval during incident)
- **Documented:** DEC-TF-005 in context graph
- **TF state update:** Pending (drift brief generated)

## Follow-Up

- [x] Apply node count change (done during INC-001)
- [x] Document decision in context graph
- [ ] Update TF code to reflect min_count=5
- [ ] Update context graph node for aks-prod-cluster
- [ ] Monitor cost impact for 30 days
- [ ] Consider autoscaling optimization (min=5, max=10 → min=5, max=15?)
