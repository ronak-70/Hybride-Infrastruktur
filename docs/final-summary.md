# Final Project Summary

## Project Overview

This project implemented a hybrid infrastructure lab that combines an on-premises Windows Server environment with Microsoft Azure services.

The infrastructure was designed to demonstrate practical skills across:

```text
Windows Server
Active Directory
DNS and DHCP
pfSense networking
Microsoft Entra ID
Azure networking
Azure Kubernetes Service
Azure Container Registry
Azure Key Vault
Microsoft Entra Workload Identity
Prometheus
Grafana
Azure Backup
Azure Policy
Azure Cost Management
Failure testing
```

The project focused on infrastructure engineering rather than application development.

---

## On-Premises Infrastructure

The on-premises environment was built using VMware Workstation.

Core systems:

```text
TW-DC01
Active Directory Domain Services
DNS
DHCP
IP: 10.0.0.5

TW-DC02
Active Directory Domain Services
DNS
IP: 10.0.0.6

TW-CLIENT
Domain-joined Windows client
IP: 10.0.0.50

TW-FW01
pfSense firewall and gateway
LAN IP: 10.0.0.1
```

Domain:

```text
techwork.local
```

Two Domain Controllers were deployed to reduce the risk of a single point of failure for authentication and DNS.

---

## Azure Infrastructure

The Azure environment was deployed in:

```text
Region: Poland Central
Resource Group: rg-hybrid-infrastructure
```

Primary virtual network:

```text
vnet-hybrid-prod
10.10.0.0/16
```

Subnet design:

```text
snet-workload
10.10.1.0/24

snet-management
10.10.2.0/24

snet-aks
10.10.3.0/24

GatewaySubnet
10.10.10.0/24
```

The primary workload VM was:

```text
tw-app-01
Private IP: 10.10.1.4
```

A NAT Gateway was configured on `snet-workload` to provide explicit outbound connectivity without assigning a public IP directly to the VM.

---

## Hybrid Connectivity

A Site-to-Site IPsec VPN configuration was created between pfSense and Azure.

The configuration included:

```text
IKEv2
AES256
SHA256
DH Group 14
```

IKE communication was observed, but the tunnel did not fully establish because of external student-network restrictions.

The Azure VPN Gateway was later removed to prevent unnecessary cost.

This limitation was documented rather than represented as successful connectivity.

---

## Hybrid Identity

Microsoft Entra Cloud Sync was used to integrate the on-premises Active Directory environment with Microsoft Entra ID.

Configuration included:

```text
Provisioning Agent: TW-DC02
Scoped OU: AzureSync
Password Hash Synchronization
Provision on Demand
```

Provision on Demand successfully created an in-scope Active Directory user in Microsoft Entra ID.

Continuous synchronization later entered quarantine because the external environment blocked required Azure Service Bus connectivity.

---

## Azure Kubernetes Service

The primary application workload was containerized and deployed to Azure Kubernetes Service.

AKS configuration:

```text
Cluster: aks-hybrid-prod
Node count: 2
Networking: Azure CNI Overlay
```

The container image was stored in Azure Container Registry:

```text
acrhybridronak01
```

The final application deployment was managed with Helm:

```text
Release: hybrid-api-helm
Minimum replicas: 2
Maximum replicas: 5
CPU target: 50%
```

The application was exposed through AKS Application Routing.

---

## High Availability

Multiple resilience mechanisms were validated.

### Pod Self-Healing

One application Pod was manually deleted.

Kubernetes immediately created a replacement Pod and restored the desired replica count.

Result:

```text
PASSED
```

### Horizontal Pod Autoscaling

A load generator increased CPU utilization above the configured HPA target.

Observed behavior:

```text
2 replicas
    |
    v
5 replicas
```

After load removal:

```text
5 replicas
    |
    v
2 replicas
```

Result:

```text
PASSED
```

### Application Availability

During Pod deletion, ten consecutive HTTP requests were sent to the public Ingress endpoint.

Result:

```text
Successful requests: 10
Failed requests: 0
Application availability: Maintained
```

### Node Scheduling Failure

One worker node was cordoned to prevent new workload scheduling.

The application Pod on that node was deleted.

Kubernetes recreated the Pod on the second healthy worker node.

Result:

```text
PASSED
```

---

## Monitoring and Alerting

Prometheus and Grafana were deployed using `kube-prometheus-stack`.

Monitoring included:

```text
Node CPU
Node Memory
Pod CPU
Pod Memory
Resource Requests
Resource Limits
Application workload
HPA behavior
```

A Grafana high CPU alert was configured and validated.

Observed state transition:

```text
Normal
    |
    v
Pending
    |
    v
Firing
```

Webhook notification delivery was also successfully tested.

---

## Security

Security controls were implemented across multiple layers.

### Azure Key Vault

```text
Vault: kv-hybrid-ronak01
Authorization: Azure RBAC
Soft Delete: Enabled
```

### Microsoft Entra Workload Identity

The AKS application accessed Azure Key Vault using passwordless workload identity.

Authentication flow:

```text
AKS Pod
    |
    v
Kubernetes ServiceAccount
    |
    v
Federated Identity Credential
    |
    v
Managed Identity
    |
    v
Azure Key Vault
```

A test Pod successfully retrieved a Key Vault secret using federated authentication.

### RBAC

Least-privilege role assignments were validated.

```text
id-hybrid-api
Role: Key Vault Secrets User
Scope: Key Vault

AKS kubelet identity
Role: AcrPull
Scope: Azure Container Registry
```

### Network Security

SSH access to the Azure VM was restricted using an Azure Network Security Group.

The AKS-managed NSG was intentionally not modified directly.

### Key Vault Network Hardening

The Key Vault firewall was configured with:

```text
Default network action: Deny
Allowed management IP: Restricted
Allowed AKS subnet: snet-aks
Microsoft.KeyVault service endpoint: Enabled
Trusted services bypass: Disabled
```

A Cloud Shell request from an unauthorized address was blocked, while secret retrieval from an authorized AKS Pod succeeded.

### Azure Container Registry

The ACR security review confirmed:

```text
Admin user: Disabled
AKS authentication: Managed Identity
AKS role: AcrPull
Public network access: Enabled
SKU: Standard
```

Private networking was documented as a production hardening recommendation.

### Microsoft Defender for Cloud

Microsoft Defender for Cloud recommendations were reviewed.

No paid Defender plans were enabled because the project uses an Azure for Students subscription and unnecessary cost was intentionally avoided.

---

## Backup and Recovery

Azure Backup was configured for `tw-app-01`.

Recovery Services Vault:

```text
rsv-hybrid-prod
```

Backup policy:

```text
Daily backup
30-day retention
2-day Instant Restore retention
```

On-demand backup was executed successfully.

Recovery points were successfully created.

### File Recovery

Linux File Recovery was tested.

Validated stages:

```text
Recovery point generation
Recovery script download
Recovery script transfer
Linux prerequisites
Outbound HTTPS connectivity
Azure Backup TCP 3260 connectivity
Secure TCP tunnel
```

The final recovery volume mount did not complete because iSCSI target authentication failed.

This limitation was documented clearly.

---

## Governance

Standard resource tags were applied:

```text
environment = lab
project = hybrid-infrastructure
owner = ronak
managed-by = manual
cost-center = student-lab
```

Azure Policy was configured to require the `environment` tag.

Compliance result:

```text
100%
16 of 16 resources compliant
0 non-compliant resources
```

---

## Cost Management

A monthly budget was configured:

```text
Budget: 20 EUR
```

Alerts:

```text
Actual cost: 50%
Actual cost: 80%
Actual cost: 100%
Forecasted cost: 100%
```

Cost Analysis was reviewed and major cost drivers were identified.

The unused VPN Gateway was removed to reduce ongoing cost.

Cost-conscious design decisions also included:

```text
Avoiding unnecessary paid Defender plans
Avoiding Azure Bastion Standard
Reusing an existing Standard Public IP for NAT Gateway
Using Standard ACR instead of Premium
Removing temporary test resources where possible
```

---

## Failure Testing

Controlled Kubernetes failure tests were performed to validate workload resilience.

Results:

```text
Pod Self-Healing Test: PASSED
HPA Scale-Up Test: PASSED
HPA Scale-Down Recovery: PASSED
Ingress Availability During Pod Failure: PASSED
Node Scheduling Failure Simulation: PASSED
Pod Rescheduling to Healthy Node: PASSED
Application Availability During Node Test: PASSED
Node Recovery to Ready State: PASSED
```

These tests confirmed that the application can recover from controlled Pod and scheduling failures while maintaining the desired workload state.

---

## Challenges Encountered

The project included several real-world troubleshooting scenarios.

### External Network Restrictions

The student-network environment blocked or interfered with:

```text
Site-to-Site VPN connectivity
Azure Service Bus connectivity
Some outbound service access
```

These restrictions were outside the infrastructure configuration itself.

### Azure Subscription Limitations

The Azure for Students subscription introduced several constraints:

```text
Public IP quota limitations
ACR Tasks operations unavailable
Cost sensitivity
Limited use of paid services
```

These constraints influenced architecture decisions.

### Azure Backup File Recovery

Although network requirements were successfully validated, the final File Recovery mount did not complete because of iSCSI authentication failure.

The test was documented honestly rather than represented as successful.

---

## Lessons Learned

### 1. Network Constraints Matter

Correct cloud and firewall configuration does not guarantee connectivity if the upstream network blocks required protocols.

### 2. Validate Dependencies Early

Services such as:

```text
Microsoft Entra Cloud Sync
Azure Backup File Recovery
Site-to-Site VPN
```

depend on external endpoints and ports.

Testing these dependencies early can prevent unnecessary troubleshooting later.

### 3. Managed Identity Reduces Secret Risk

Microsoft Entra Workload Identity provided a better security model than storing static Azure credentials inside Kubernetes.

### 4. High Availability Requires Testing

Deploying multiple replicas or worker nodes is not enough.

Controlled failure tests are required to verify that workloads actually recover.

### 5. Monitoring Must Include Alerting

Dashboards alone are not operational monitoring.

The project therefore validated:

```text
Metrics
Dashboards
Alert rules
Alert state transitions
Notification delivery
```

### 6. Backup Is Not the Same as Recovery

Successful backup jobs do not automatically prove that recovery works.

Recovery should be tested independently.

### 7. Cost Is an Architecture Constraint

Student and lab environments require active cost management.

The project used:

```text
Budgets
Cost alerts
Resource cleanup
Reuse of existing resources
Avoidance of unnecessary paid services
```

### 8. Documentation Is Part of the Infrastructure

Troubleshooting decisions, limitations, and test results were documented continuously.

This made it possible to distinguish:

```text
Configured
Validated
Partially validated
Environmentally limited
```

---

## Final Validation Status

```text
On-Premises Infrastructure       VALIDATED
Active Directory                 VALIDATED
DNS                              VALIDATED
DHCP                             VALIDATED
Domain Client                    VALIDATED

Azure Virtual Network            VALIDATED
Azure VM                         VALIDATED
NAT Gateway                      VALIDATED

Site-to-Site VPN                 CONFIGURED / LIMITED

Microsoft Entra Cloud Sync       PARTIALLY VALIDATED

Azure Container Registry         VALIDATED
Azure Kubernetes Service         VALIDATED
Helm Deployment                  VALIDATED
Horizontal Pod Autoscaler        VALIDATED

Prometheus                       VALIDATED
Grafana                          VALIDATED
Alerting                         VALIDATED

Azure Key Vault                  VALIDATED
Workload Identity                VALIDATED
Azure RBAC                       VALIDATED
Network Security                 VALIDATED
ACR Security                     VALIDATED
Defender for Cloud Review        VALIDATED

Azure Backup                     VALIDATED
File Recovery                    LIMITED

Resource Tags                    VALIDATED
Azure Policy                     VALIDATED
Cost Budget                      VALIDATED
Cost Alerts                      VALIDATED

Failure Testing                  VALIDATED
```

---

## Final Outcome

The completed project demonstrates a realistic hybrid infrastructure environment that combines:

```text
Windows Server infrastructure
Active Directory
Microsoft Entra ID
Azure networking
Container orchestration
Monitoring and alerting
Secret management
Passwordless identity
RBAC
Backup
Governance
Cost control
Failure recovery
```

The project also demonstrates real-world troubleshooting and documents environmental limitations where full validation was not possible.

The final result is a:

**documented, monitored, secured, governed, cost-aware, and resilience-tested hybrid infrastructure lab.**
