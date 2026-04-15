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
Production Kubernetes cluster for CAMARA API services. Provisioned as part of the platform migration from Azure Container Service (ACS) in October 2025. Houses all production workloads: payment-api, camara-gateway, monitoring stack, and ingress controllers.

## Owners
- **Primary:** platform-team
- **On-call:** sre-oncall (rotates weekly)
- **Approver:** sre-lead
- **Cost center:** CC-PROD-001

## Current Configuration
- **Location:** East US 2
- **Node count:** 5 (min) / 10 (max)
- **SKU:** Standard_D4s_v5 (4 vCPU, 16GB RAM)
- **Kubernetes version:** 1.28.0
- **Network plugin:** Azure CNI
- **Network policy:** Calico
- **DNS service:** CoreDNS
- **Load balancer:** Standard SKU
- **Monitoring:** Azure Monitor + Prometheus + Grafana

## Change History

| Date | What Changed | Who | Why | Ticket/Ref |
|------|-------------|-----|-----|------------|
| 2025-10-15 | Created (3 nodes, k8s 1.26) | platform-team | ACS → AKS migration | PROJ-1234 |
| 2025-11-20 | k8s upgraded 1.26→1.27 | engineer-a | Quarterly version bump | PROJ-1345 |
| 2025-12-10 | node SKU changed D2s→D4s | engineer-b | Memory pressure in dev | PROJ-1456 |
| 2026-01-10 | k8s upgraded 1.27→1.28 | engineer-a | Security patch (post INC-004) | INC-004 |
| 2026-03-05 | autoscaling enabled (3-10) | engineer-b | Cost optimization | PROJ-2345 |
| 2026-04-12 | min nodes 3→5 | engineer-a | Post INC-001 (disk pressure) | DEC-TF-005 |

## Dependencies

### This resource is depended on by:

**Workloads (pods running on this cluster):**
- deployment/payment-api (3 replicas) — payment processing
- deployment/camara-gateway (5 replicas) — CAMARA API gateway
- deployment/camara-qod-api (2 replicas) — Quality of Data API
- deployment/camara-sim-swap (2 replicas) — SIM swap detection
- deployment/monitoring-prometheus (1 replica) — metrics collection
- deployment/monitoring-grafana (1 replica) — dashboards
- ingress/nginx-ingress-controller (2 replicas) — external routing
- cert-manager/letsencrypt-prod (1 replica) — TLS certificates

**External consumers:**
- AT&T internal API gateway → routes to camara-gateway
- Partner integrations → consume CAMARA APIs
- Monitoring dashboards → Grafana shows cluster health

### This resource depends on:

- `azurerm_resource_group.rg-production` — container RG
- `azurerm_virtual_network.vnet-prod` — network fabric
- `azurerm_subnet.aks-subnet` — IP range for nodes
- `azurerm_subnet.aks-services-subnet` — IP range for services
- `azurerm_log_analytics_workspace.logs-prod` — diagnostic logs
- `azurerm_container_registry.acr-prod` — container images

## Incident History

| Incident | Date | What Happened | Impact on This Resource | Change Made |
|----------|------|--------------|----------------------|-------------|
| INC-001 | 2026-04-12 | Disk pressure, 3 nodes NotReady | node_count increased 3→5 | DEC-TF-005 |
| INC-002 | 2026-04-13 | Provider timeout on CAMARA API | No infra change | — |
| INC-004 | 2026-01-10 | Runtime restart cascading failures | k8s version upgraded 1.27→1.28 | PROJ-1345 |
| INC-006 | 2026-03-05 | Provider timeout (repeat) | Autoscaling enabled | PROJ-2345 |

## Known Risks

1. **Disk accumulation:** Container images build up every ~90 days without automated prune. Caused INC-001. Weekly prune cron added but needs monitoring.
2. **Node pool scaling:** Min=5 may be insufficient during peak traffic (10:00-14:00 IST). Autoscaling max=10 but scale-up takes 3-5 minutes.
3. **Version skew:** k8s 1.28 approaching EOL — upgrade to 1.29 needed by Q3 2026. Coordinated with platform team.
4. **SKU lock-in:** Standard_D4s_v5 is the only SKU used. No burstable or spot instances for cost optimization.
5. **Single region:** Cluster is in East US 2 only. DR cluster in Central US exists but is not actively synced.

## Drift Sensitivity

| Attribute | Sensitivity | Why |
|-----------|------------|-----|
| node_count | HIGH | Direct impact on availability and cost |
| sku_name | HIGH | Cost + performance impact, requires node rotation |
| kubernetes_version | MEDIUM | Requires careful upgrade process + testing |
| network_plugin | CRITICAL | Changing requires full cluster rebuild |
| network_policy | HIGH | Affects pod-to-pod communication |
| lb_sku | MEDIUM | Affects external connectivity |
| addon_oms_agent | LOW | Monitoring only, no workload impact |
| private_cluster | CRITICAL | Changing this requires cluster rebuild |

## Cost Breakdown

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| 5x Standard_D4s_v5 nodes | $8,000 | $1,600/node/month |
| Load Balancer (Standard) | $500 | Fixed |
| Public IPs (3) | $90 | $30/IP/month |
| Azure CNI IP addresses | $200 | 50 IPs reserved |
| Log Analytics ingestion | $1,200 | ~50GB/day |
| Azure Monitor metrics | $500 | Custom metrics |
| **Total** | **$10,490** | + ACR + Storage separately |

## Related Context Nodes

- [vnet-production](vnet-production.md) — Network fabric
- [rg-production](rg-production.md) — Resource group
- [storage-logs](storage-logs.md) — Diagnostic storage
- [keyvault-prod](keyvault-prod.md) — Secrets and certificates
- [acr-prod](acr-prod.md) — Container registry
