# Context Queries

> Common questions and how to answer them with the context graph.

## "Why does this resource exist?"

```bash
grep -A20 "^## Why It Exists" nodes/[resource].md
```

Every node has a `## Why It Exists` section explaining the original rationale.

## "Who created/changed this?"

```bash
grep -A10 "^## Change History" nodes/[resource].md
```

Full change history with who, when, why, and ticket references.

## "What depends on this resource?"

```bash
grep -A20 "^### This resource is depended on by" nodes/[resource].md
```

Shows all downstream dependencies — workloads, services, other resources.

## "What happens if I change/delete this?"

Two things to check:

```bash
# 1. Dependencies (blast radius)
grep -A20 "^### This resource is depended on by" nodes/[resource].md

# 2. Known risks
grep -A10 "^## Known Risks" nodes/[resource].md

# 3. Drift sensitivity
grep -A10 "^## Drift Sensitivity" nodes/[resource].md
```

## "Has this been involved in incidents?"

```bash
grep -A10 "^## Incident History" nodes/[resource].md
```

## "Is this drift dangerous?"

```bash
# Check the drift brief
cat drift/[date]-[resource].md

# Check drift sensitivity
grep -A10 "^## Drift Sensitivity" nodes/[resource].md
```

## "What decisions were made about this resource?"

```bash
grep -l "[resource-name]" decisions/*.md
```

## "How much does this cost?"

```bash
grep -A10 "^## Cost Breakdown" nodes/[resource].md
```

## "Who owns this resource?"

```bash
grep -A5 "^## Owners" nodes/[resource].md
```

## Cross-Resource Queries

### "Show me all resources in production"
```bash
grep -l "resource_group: rg-production" nodes/*.md
```

### "Show me all resources involved in INC-001"
```bash
grep -l "INC-001" nodes/*.md
```

### "Show me all HIGH sensitivity resources"
```bash
grep -B1 "HIGH" nodes/*.md | grep "^--$|^nodes" 
```

### "Show me all decisions by engineer-a"
```bash
grep -l "engineer-a" decisions/*.md
```

### "Show me the full dependency chain for prod-cluster"
```bash
cat nodes/aks-prod-cluster.md | sed -n '/depends on:/,/## Incident/p'
# Then follow each dependency:
cat nodes/vnet-production.md | sed -n '/depends on:/,/## Incident/p'
```
