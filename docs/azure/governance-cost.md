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
