## Cost Management and Optimization

Azure Cost Management was configured for the Hybrid Infrastructure lab.

A monthly budget was created at the resource group scope:

```text
Budget name: budget-hybrid-lab
Scope: rg-hybrid-infrastructure
Budget amount: 20 EUR
Reset period: Monthly
```

Budget alerts were configured at:

```text
Actual cost: 50%
Actual cost: 80%
Actual cost: 100%
Forecasted cost: 100%
```

Cost Analysis was reviewed using:

```text
Scope: rg-hybrid-infrastructure
Period: September 2026
Granularity: Daily
Group by: Resource
```

At the time of review:

```text
Actual cost: approximately 4.88 EUR
Forecasted monthly cost: approximately 50.38 EUR
Configured monthly budget: 20 EUR
```

The main observed cost drivers included:

```text
VPN Gateway
Virtual machine storage
Public IP and networking resources
```

The VPN Gateway had already been removed because the external student-network environment prevented successful Site-to-Site VPN operation.

The project therefore applies the following cost controls:

```text
Resource tagging
Resource-group budget
Actual-cost alerts
Forecast-cost alerts
Removal of unused resources
Avoidance of unnecessary paid Defender plans
Avoidance of Azure Bastion Standard
Reuse of an existing Standard Public IP for NAT Gateway
Use of Standard ACR instead of Premium
Temporary resources removed after testing
```

Cost optimization review result:

```text
Resource tagging: Completed
Budget: Configured
Budget alerts: Configured
Cost Analysis: Reviewed
Primary cost drivers: Identified
Unused VPN Gateway: Removed
Cost-conscious architecture decisions: Documented
```

## Azure Policy Governance

Azure Policy was configured to enforce tagging standards across the Hybrid Infrastructure resource group.

The built-in policy definition used was:

```text
Policy definition: Require a tag on resources
Assignment name: require-environment-tag
Scope: rg-hybrid-infrastructure
Required tag: environment
Effect: Deny
```

The policy requires resources within the project scope to include the environment tag.

Before the policy assignment was created, all project resources were tagged with:

```text
environment = lab
```

A manual Azure Policy compliance scan was triggered using:

```bash
az policy state trigger-scan \
  --resource-group rg-hybrid-infrastructure
```

After policy evaluation completed, the assignment reported:

```text
Compliance state: Compliant
Resource compliance: 100%
Compliant resources: 16 of 16
Non-compliant resources: 0
Non-compliant policies: 0
```

This confirmed that the project resources comply with the required tagging standard.

Because the policy uses the Deny effect, future resource deployments or updates within the governed scope can be rejected if the required environment tag is missing.

The resulting governance model is:

```text
rg-hybrid-infrastructure
        |
        v
Azure Policy
require-environment-tag
        |
        v
Required tag:
environment
        |
        v
Existing resources: Compliant
Future untagged resources: Denied
```
Azure Policy governance result:

```text
Resource tagging standard: Defined
Tag enforcement policy: Assigned
Required environment tag: Enforced
Policy evaluation: Completed
Resource compliance: 100%
Non-compliant project resources: 0
Governance baseline: Validated
```

```markdown
## Current Status

Phase 13 Governance and Cost Management has been completed.

```text
Resource tagging: Completed
Standard project tags: Applied
Monthly budget: Configured
Budget alerts: Configured
Cost Analysis: Reviewed
Primary cost drivers: Identified
Azure Policy assignment: Configured
Required environment tag: Enforced
Policy compliance: 100%
Governance baseline: Validated