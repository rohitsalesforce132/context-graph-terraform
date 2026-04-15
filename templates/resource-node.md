# Resource Node Template

```markdown
---
id: CG-TF-[resource-name]
resource_type: [azurerm_*]
resource_name: [name]
resource_group: [rg-name]
tf_file: [file.tf]
tf_module: [module]
created: [YYYY-MM-DD]
created_by: [team/person]
ticket: [ticket-ref]
monthly_cost: [$amount]
---

# [resource-name] ([resource-type-short])

## Why It Exists
[One paragraph: what is this resource, why was it created, what problem does it solve]

## Owners
- **Primary:** [team]
- **On-call:** [team/rotation]
- **Approver:** [person/team]
- **Cost center:** [code]

## Current Configuration
- **Attribute1:** value
- **Attribute2:** value
- **Attribute3:** value

## Change History

| Date | What Changed | Who | Why | Ticket |
|------|-------------|-----|-----|--------|
| YYYY-MM-DD | [change] | [person] | [reason] | [ref] |

## Dependencies

### This resource is depended on by:
- [resource] — [what it's used for]
- [resource] — [what it's used for]

### This resource depends on:
- [resource] — [why]
- [resource] — [why]

## Incident History

| Incident | Date | What Happened | Impact | Change Made |
|----------|------|--------------|--------|-------------|
| INC-XXX | YYYY-MM-DD | [description] | [impact] | [change/ref] |

## Known Risks
1. [Risk description]
2. [Risk description]

## Drift Sensitivity

| Attribute | Sensitivity | Why |
|-----------|------------|-----|
| [attr] | HIGH/MED/LOW/CRITICAL | [reason] |

## Cost Breakdown

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| [item] | $amount | [notes] |

## Related Context Nodes
- [resource-a](resource-a.md)
- [resource-b](resource-b.md)
```
