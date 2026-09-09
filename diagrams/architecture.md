# Final Architecture Diagram

## Hybrid Infrastructure Architecture

```mermaid
flowchart LR

    %% -----------------------------
    %% On-Premises
    %% -----------------------------

    subgraph ONPREM["On-Premises Infrastructure"]
        direction TB

        VMWARE["VMware Workstation"]

        DC01["TW-DC01<br/>AD DS / DNS / DHCP<br/>10.0.0.5"]
        DC02["TW-DC02<br/>AD DS / DNS<br/>10.0.0.6"]
        CLIENT["TW-CLIENT<br/>Domain Joined<br/>10.0.0.50"]
        PFSENSE["TW-FW01 / pfSense<br/>LAN: 10.0.0.1"]

        VMWARE --> DC01
        VMWARE --> DC02
        VMWARE --> CLIENT
        VMWARE --> PFSENSE

        CLIENT --> DC01
        CLIENT --> DC02
    end

    %% -----------------------------
    %% Microsoft Entra ID
    %% -----------------------------

    ENTRA["Microsoft Entra ID"]

    DC02 -->|"Microsoft Entra Cloud Sync<br/>Provision on Demand validated"| ENTRA

    %% -----------------------------
    %% Hybrid VPN
    %% -----------------------------

    VPNCFG["Site-to-Site IPsec VPN<br/>IKEv2 / AES256 / SHA256<br/>Configured, not operational due to<br/>external student-network limitation"]

    PFSENSE -.-> VPNCFG

    %% -----------------------------
    %% Azure VNet
    %% -----------------------------

    subgraph AZURE["Microsoft Azure - Poland Central"]
        direction TB

        subgraph VNET["vnet-hybrid-prod<br/>10.10.0.0/16"]
            direction TB

            WORKLOAD["snet-workload<br/>10.10.1.0/24"]
            MGMT["snet-management<br/>10.10.2.0/24"]
            AKSSUBNET["snet-aks<br/>10.10.3.0/24"]
            GWSUBNET["GatewaySubnet<br/>10.10.10.0/24"]

            VM["tw-app-01<br/>10.10.1.4"]
            NAT["nat-hybrid-workload<br/>Explicit outbound connectivity"]

            WORKLOAD --> VM
            WORKLOAD --> NAT
        end

        %% AKS
        subgraph AKS["Azure Kubernetes Service<br/>aks-hybrid-prod"]
            direction TB

            NODE1["Worker Node 1"]
            NODE2["Worker Node 2"]

            DEPLOY["Helm Release<br/>hybrid-api-helm"]

            POD1["Application Pod"]
            POD2["Application Pod"]

            HPA["HPA<br/>Min 2 / Max 5<br/>CPU Target 50%"]

            SVC["ClusterIP Service"]
            ING["Application Routing Ingress<br/>20.215.101.192"]

            NODE1 --> POD1
            NODE2 --> POD2

            DEPLOY --> POD1
            DEPLOY --> POD2
            HPA --> DEPLOY

            POD1 --> SVC
            POD2 --> SVC
            SVC --> ING
        end

        AKSSUBNET --> AKS

        %% Container Registry
        ACR["Azure Container Registry<br/>acrhybridronak01<br/>Admin User Disabled"]

        ACR -->|"AcrPull via kubelet identity"| AKS

        %% Key Vault / Identity
        KV["Azure Key Vault<br/>kv-hybrid-ronak01<br/>RBAC + Firewall"]

        MI["User Assigned Managed Identity<br/>id-hybrid-api"]

        SA["Kubernetes ServiceAccount<br/>hybrid-api-sa"]

        FIC["Federated Identity Credential<br/>fic-hybrid-api"]

        SA --> FIC
        FIC --> MI
        MI -->|"Key Vault Secrets User"| KV
        AKS --> SA

        %% Monitoring
        subgraph MON["Monitoring and Alerting"]
            PROM["Prometheus"]
            GRAFANA["Grafana"]
            ALERT["Grafana Alerting<br/>High CPU Alert"]
        end

        AKS --> PROM
        PROM --> GRAFANA
        GRAFANA --> ALERT

        %% Backup
        RSV["Recovery Services Vault<br/>rsv-hybrid-prod"]

        VM -->|"Azure Backup"| RSV

        %% Governance
        POLICY["Azure Policy<br/>Require environment tag<br/>100% compliant"]

        BUDGET["Cost Management<br/>Monthly Budget: 20 EUR"]

        POLICY --> VNET
        BUDGET --> AZURE
    end

    %% -----------------------------
    %% External Access
    %% -----------------------------

    USER["External User / Tester"]

    USER -->|"HTTP"| ING

    %% -----------------------------
    %% VPN Relationship
    %% -----------------------------

    VPNCFG -.-> GWSUBNET

    %% -----------------------------
    %% Notes
    %% -----------------------------

    LIMIT["Known Environmental Limitations<br/><br/>• S2S VPN could not fully establish<br/>• Continuous Entra Cloud Sync blocked by Service Bus access<br/>• Azure Backup File Recovery stopped at iSCSI authentication"]

    LIMIT -.-> AZURE
```

---

## Architecture Summary

The final lab architecture combines:

```text
On-Premises Active Directory
Microsoft Entra Cloud Sync
Azure Virtual Network
Azure Kubernetes Service
Azure Container Registry
Azure Key Vault
Microsoft Entra Workload Identity
Prometheus and Grafana
Azure Backup
Azure Policy
Azure Cost Management
NAT Gateway
```

The architecture was intentionally designed to demonstrate:

```text
Hybrid identity
Network segmentation
Container orchestration
Application high availability
Horizontal scaling
Centralized monitoring
Alerting
Passwordless workload authentication
Least-privilege RBAC
Secret management
Backup protection
Governance
Cost control
Failure recovery
```

---

## Validated Resilience

The following resilience scenarios were successfully tested:

```text
Pod self-healing
Pod replacement
HPA scale-up from 2 to 5 replicas
HPA scale-down from 5 to 2 replicas
Application availability during Pod deletion
Pod rescheduling from a cordoned node
Application availability during node scheduling failure
```

---

## Known Limitations

The diagram distinguishes configured functionality from functionality that could not be fully validated due to external environment restrictions.

```text
Site-to-Site VPN:
Configured but not fully established because of the external student-network environment.

Microsoft Entra Cloud Sync:
Provision on Demand validated.
Continuous synchronization later entered quarantine because outbound Azure Service Bus connectivity was blocked.

Azure Backup File Recovery:
Backup and recovery points were validated.
File Recovery progressed through Secure TCP tunnel creation but stopped during iSCSI target authentication.
```

These limitations are documented explicitly rather than represented as successful production functionality.
