[README.md](https://github.com/user-attachments/files/32009058/README.md)
# Hybrid Infrastructure Project

> **Windows Server & Microsoft Azure — Enterprise Hybrid Infrastructure Lab**

![Status](https://img.shields.io/badge/status-final%20documentation-blue)
![Platform](https://img.shields.io/badge/platform-Windows%20Server%20%7C%20Azure-blue)
![Container](https://img.shields.io/badge/container-AKS%20%7C%20ACR-blue)
![Monitoring](https://img.shields.io/badge/monitoring-Prometheus%20%7C%20Grafana-orange)
![Security](https://img.shields.io/badge/security-Key%20Vault%20%7C%20Workload%20Identity-green)
![License](https://img.shields.io/badge/license-Educational-lightgrey)

## Overview

This project implements a small-scale **enterprise hybrid infrastructure lab** combining an on-premises Windows Server environment with Microsoft Azure.

The project demonstrates how infrastructure can be:

- centrally managed
- integrated with cloud identity
- containerized and orchestrated
- monitored and alerted
- secured using identity and network controls
- protected with backup
- governed with policy and tagging
- cost-monitored
- validated through controlled failure tests

The focus is the infrastructure surrounding the workload rather than application development itself.

> **Important:** Some hybrid connectivity features were configured successfully but could not remain operational because of external network restrictions in the student environment. These limitations are documented rather than represented as successful production connectivity.

---

## Architecture

```text
                              Internet
                                 |
                +----------------+----------------+
                |                                 |
                v                                 v
        +---------------+                 +------------------+
        | Microsoft     |                 | Azure Public     |
        | Entra ID      |                 | Ingress          |
        +-------+-------+                 +--------+---------+
                ^                                  |
                | Cloud Sync                       v
                |                         +------------------+
+---------------+-------------+           | Azure Kubernetes |
| On-Premises Infrastructure  |           | Service (AKS)    |
|                             |           | hybrid-api-helm  |
| VMware Workstation          |           | 2-5 replicas     |
|  - TW-DC01                  |           +--------+---------+
|  - TW-DC02                  |                    |
|  - TW-CLIENT                |                    v
|  - TW-FW01 / pfSense        |           +------------------+
+---------------+-------------+           | Azure Container  |
                |                         | Registry (ACR)   |
                |                         +------------------+
                |
                | Site-to-Site VPN configuration
                | (connectivity limited by external network)
                v
        +---------------------+
        | Azure VNet          |
        | 10.10.0.0/16        |
        |                     |
        | snet-workload       |
        | snet-management     |
        | snet-aks            |
        | GatewaySubnet       |
        +----------+----------+
                   |
        +----------+------------------------------------+
        |                    |                          |
        v                    v                          v
  Azure Key Vault      Recovery Services Vault   NAT Gateway
        |                    |                          |
        v                    v                          v
 Workload Identity       VM Backup                Explicit Egress
```

Detailed architecture documentation:

- [Architecture](diagrams/architecture.md)

---

## Technology Stack

| Area | Technology |
|---|---|
| Virtualization | VMware Workstation |
| On-Premises OS | Windows Server |
| Directory Services | Active Directory Domain Services |
| DNS | Windows DNS |
| DHCP | Windows DHCP |
| Firewall / Routing | pfSense |
| Cloud Identity | Microsoft Entra ID |
| Hybrid Identity | Microsoft Entra Cloud Sync |
| Cloud Platform | Microsoft Azure |
| Networking | Azure Virtual Network |
| Hybrid Connectivity | Site-to-Site VPN configuration |
| Containers | Docker |
| Container Registry | Azure Container Registry |
| Orchestration | Azure Kubernetes Service |
| Package Management | Helm |
| Metrics | Prometheus |
| Visualization | Grafana |
| Security | Azure RBAC, NSG, Defender for Cloud |
| Secrets | Azure Key Vault |
| Workload Authentication | Microsoft Entra Workload Identity |
| Backup | Azure Backup |
| Governance | Azure Policy |
| Cost Management | Azure Cost Management |

---

## On-Premises Infrastructure

```text
Network: 10.0.0.0/24
Gateway: 10.0.0.1

TW-DC01:   10.0.0.5
TW-DC02:   10.0.0.6
TW-CLIENT: 10.0.0.50
TW-FW01:   pfSense gateway/firewall
```

Active Directory domain:

```text
techwork.local
```

High availability is provided through two Domain Controllers:

```text
TW-DC01
TW-DC02
```

---

## Azure Network

```text
VNet: vnet-hybrid-prod
Address space: 10.10.0.0/16

snet-workload     10.10.1.0/24
snet-management   10.10.2.0/24
snet-aks          10.10.3.0/24
GatewaySubnet     10.10.10.0/24
```

Azure workload VM:

```text
VM: tw-app-01
Private IP: 10.10.1.4
Subnet: snet-workload
```

A NAT Gateway was attached to `snet-workload` to provide explicit outbound connectivity without assigning a public IP directly to the VM.

---

## Hybrid Connectivity

A Site-to-Site IPsec VPN architecture was configured between pfSense and Azure.

```text
On-Premises network: 10.0.0.0/24
Azure network:       10.10.0.0/16
IKE version:         IKEv2
Encryption:          AES256
Integrity:           SHA256
DH group:            14
```

IKE traffic was observed, but the tunnel could not complete because the external student-network environment restricted required connectivity.

The Azure VPN Gateway was later removed to avoid unnecessary cost because the environmental limitation could not be changed.

Detailed documentation:

- [Local Network Gateway](docs/azure/local-network-gateway.md)
- [Site-to-Site VPN](docs/azure/site-to-site-vpn.md)
- [VPN Troubleshooting](docs/azure/vpn-troubleshooting.md)

---

## Hybrid Identity

Hybrid identity was implemented using **Microsoft Entra Cloud Sync**.

```text
On-Premises domain: techwork.local
Provisioning Agent: TW-DC02
Scoped OU: AzureSync
Authentication: Password Hash Synchronization
```

A Provision on Demand test was successfully completed for an in-scope Active Directory user.

Continuous Cloud Sync later entered quarantine because the external environment blocked outbound connectivity required by the provisioning agent to Azure Service Bus.

Detailed documentation:

- [Hybrid Identity](docs/azure/hybrid-identity.md)

---

## Azure Kubernetes Service

```text
Cluster: aks-hybrid-prod
Region: Poland Central
Kubernetes: 1.35.x
Node count: 2
Networking: Azure CNI Overlay
```

Container registry:

```text
Registry: acrhybridronak01
Image: hybrid-api:v1
```

AKS accesses ACR using managed identity with the `AcrPull` role.

### Helm Deployment

```text
Release: hybrid-api-helm
Service type: ClusterIP
Minimum replicas: 2
Maximum replicas: 5
CPU HPA target: 50%
```

Ingress:

```text
Ingress class: webapprouting.kubernetes.azure.com
Public IP: 20.215.101.192
```

Application response:

```json
{
  "message": "Hybrid Infrastructure API is running",
  "environment": "Azure Kubernetes Service",
  "status": "healthy"
}
```

Detailed documentation:

- [AKS](docs/azure/aks.md)

---

## High Availability and Resilience

Controlled failure tests validated:

```text
Pod deletion and self-healing: PASSED
HPA scale-up: PASSED
HPA scale-down recovery: PASSED
Ingress availability during Pod failure: PASSED
Node scheduling failure simulation: PASSED
Pod rescheduling to healthy node: PASSED
```

During the ingress availability test, all 10 HTTP requests remained successful while one application Pod was deleted and recreated.

Detailed documentation:

- [Failure Tests](docs/testing/failure-tests.md)
- [HA Tests](docs/testing/ha-test.md)

---

## Monitoring and Alerting

Prometheus and Grafana were deployed using `kube-prometheus-stack`.

Monitoring includes:

- AKS node CPU and memory
- Pod CPU and memory
- Kubernetes resource requests and limits
- application workload metrics
- HPA behavior under load

Grafana was accessed through local port forwarding.

### Alerting

```text
Alert: hybrid-api-high-cpu
Threshold: > 80%
Evaluation: 1 minute
Pending period: 2 minutes
Severity: warning
Environment: lab
```

The alert was validated through:

```text
Normal -> Pending -> Firing
```

Webhook delivery was also successfully tested.

Detailed documentation:

- [Monitoring](docs/azure/monitoring.md)

---

## Security

### Azure Key Vault

```text
Vault: kv-hybrid-ronak01
Authorization: Azure RBAC
Soft delete: Enabled
Network default action: Deny
```

### Workload Identity

```text
Kubernetes ServiceAccount: hybrid-api-sa
Managed Identity: id-hybrid-api
Federated Credential: fic-hybrid-api
Key Vault role: Key Vault Secrets User
```

A temporary AKS Pod successfully authenticated using a projected federated token and retrieved a test secret from Azure Key Vault.

### RBAC

```text
id-hybrid-api:
  Key Vault Secrets User
  Scope: Key Vault only

AKS kubelet identity:
  AcrPull
  Scope: ACR only
```

### Network Security

SSH access to `tw-app-01` is restricted by NSG to the approved management public IP.

The AKS-managed NSG was not manually modified.

### Container Registry

```text
ACR admin user: Disabled
AKS authentication: Managed Identity
AKS role: AcrPull
Public network access: Enabled
SKU: Standard
```

### Defender for Cloud

Microsoft Defender for Cloud recommendations were reviewed. Paid Defender plans were not enabled to avoid unnecessary student-subscription cost.

Detailed documentation:

- [Security](docs/azure/security.md)

---

## Backup and Recovery

```text
Recovery Services Vault: rsv-hybrid-prod
Backup policy: policy-daily-vm-backup
Frequency: Daily
Retention: 30 days
Instant Restore: 2 days
```

On-demand backup was executed and recovery points were successfully created.

### File Recovery Test

Successful validation included:

```text
Recovery script generation
Recovery script transfer to VM
Linux recovery prerequisites
NAT-based outbound connectivity
HTTPS 443 connectivity
Azure Backup TCP 3260 connectivity
Secure TCP tunnel startup
```

The final recovery volume mount did not complete because iSCSI discovery failed during target authentication.

Detailed documentation:

- [Backup and Recovery](docs/azure/backup-recovery.md)

---

## Governance and Cost Management

### Resource Tagging

```text
environment = lab
project = hybrid-infrastructure
owner = ronak
managed-by = manual
cost-center = student-lab
```

### Azure Policy

```text
Policy: Require a tag on resources
Required tag: environment
Compliance: 100%
Compliant resources: 16 of 16
Non-compliant resources: 0
```

### Budget

```text
Budget: budget-hybrid-lab
Amount: 20 EUR
Reset period: Monthly

Actual cost alerts: 50%, 80%, 100%
Forecasted cost alert: 100%
```

Detailed documentation:

- [Governance and Cost Management](docs/azure/governance-cost.md)

---

## Known Limitations

### Site-to-Site VPN

The VPN configuration was completed, but the tunnel could not fully establish because of restrictions in the external student-network environment.

### Entra Cloud Sync

Provision on Demand succeeded, but continuous provisioning later entered quarantine because required outbound Service Bus connectivity was blocked.

### File Recovery

Azure Backup created valid recovery points, but the Linux File Recovery test stopped at iSCSI target authentication and the recovery volume was not mounted.

### Production Networking

Some services remain publicly reachable because the project uses cost-conscious lab SKUs. Production recommendations include Private Endpoints, private DNS, stronger ingress restrictions, and centralized logging.

---

## Repository Structure

```text
Hybride-Infrastruktur/
|
├── README.md
├── diagrams/
│   └── architecture.md
├── docs/
│   ├── azure/
│   │   ├── aks.md
│   │   ├── backup-recovery.md
│   │   ├── governance-cost.md
│   │   ├── hybrid-identity.md
│   │   ├── local-network-gateway.md
│   │   ├── monitoring.md
│   │   ├── security.md
│   │   ├── site-to-site-vpn.md
│   │   ├── vnet.md
│   │   ├── vpn-gateway.md
│   │   └── vpn-troubleshooting.md
│   ├── network/
│   ├── on-premises/
│   └── testing/
│       ├── failure-tests.md
│       └── ha-test.md
└── screenshots/
    └── on-premises/
```

---

## Project Status

| Phase | Status |
|---|---|
| Planning | ✅ Completed |
| On-Premises Infrastructure | ✅ Completed |
| VMware / pfSense | ✅ Completed |
| Active Directory | ✅ Completed |
| Domain Client | ✅ Completed |
| Azure Virtual Network | ✅ Completed |
| Site-to-Site VPN Configuration | ⚠️ Configured / Environmental limitation |
| Hybrid Identity | ⚠️ Provisioning validated / Continuous sync limited |
| ACR / AKS Application | ✅ Completed |
| High Availability | ✅ Completed |
| Monitoring and Alerting | ✅ Completed |
| Security Hardening | ✅ Completed |
| Backup and Recovery | ⚠️ Backup validated / File restore limitation documented |
| Governance and Cost Management | ✅ Completed |
| Failure Testing | ✅ Completed |
| Final Documentation | 🟡 In Progress |
| Presentation | 🟡 In Progress |

---

## Key Outcomes

```text
Two-domain-controller on-premises Active Directory
Domain-joined Windows client
pfSense routing and VPN configuration
Azure network segmentation
Microsoft Entra Cloud Sync configuration
Container image build and ACR integration
Two-node AKS deployment
Helm-managed application deployment
Horizontal Pod Autoscaling
Prometheus and Grafana monitoring
Grafana alerting and webhook delivery
Azure Key Vault
Microsoft Entra Workload Identity
Least-privilege Azure RBAC
Key Vault network restrictions
Azure VM Backup
NAT Gateway explicit outbound connectivity
Resource tagging
Azure Policy enforcement
Cost budgets and alerts
Controlled Kubernetes failure testing
```

The final result is a **documented, monitored, secured, governed, cost-aware, and resilience-tested hybrid infrastructure lab**.

---

## Security Notice

No passwords, API keys, VPN pre-shared keys, SAS URLs, temporary recovery passwords, certificates, or other sensitive credentials should be committed to this repository.

---

## License

This project is intended for **educational and portfolio purposes**.
