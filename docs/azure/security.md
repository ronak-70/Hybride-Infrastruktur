# Security

## Overview

Phase 11 of the Hybrid Infrastructure project focuses on securing access between Azure Kubernetes Service (AKS) workloads and Azure platform services.

The primary security implementation in this phase uses Azure Key Vault together with Microsoft Entra Workload Identity. The goal is to allow Kubernetes workloads to access Azure resources without storing long-lived passwords, client secrets, or credentials inside container images, Kubernetes manifests, Helm values, or the GitHub repository.

A dedicated user-assigned managed identity was created for the `hybrid-api` workload and federated with a Kubernetes ServiceAccount.

The resulting authentication architecture is:

```text
hybrid-api Pod
      |
      v
Kubernetes ServiceAccount
hybrid-api-sa
      |
      v
Microsoft Entra Workload Identity
      |
      v
Federated Identity Credential
fic-hybrid-api
      |
      v
User Assigned Managed Identity
id-hybrid-api
      |
      v
Azure RBAC
      |
      v
Azure Key Vault
kv-hybrid-ronak01
      |
      v
Application Secrets
```

This architecture provides passwordless authentication between the Kubernetes workload and Azure Key Vault.

---

## Security Objectives

The security implementation was designed with the following objectives:

- Avoid storing Azure credentials inside Kubernetes YAML files.
- Avoid storing application secrets inside container images.
- Avoid committing secrets to GitHub.
- Use Microsoft Entra identities for workload authentication.
- Apply least-privilege access to Azure Key Vault.
- Use Azure RBAC instead of legacy Key Vault access policies.
- Separate the application identity from the AKS cluster identity.
- Restrict Key Vault network access to approved networks.
- Validate access directly from a Kubernetes workload.
- Use short-lived federated tokens instead of static application credentials.

---

## Azure Key Vault

Azure Key Vault was deployed to securely store application secrets.

The Key Vault was created in the existing project resource group:

```text
Key Vault name: kv-hybrid-ronak01
Resource group: rg-hybrid-infrastructure
Region: Poland Central
SKU: Standard
Authorization model: Azure RBAC
Soft delete: Enabled
```

Before the Key Vault could be created, the Azure subscription required registration of the `Microsoft.KeyVault` resource provider.

The provider was registered using:

```bash
az provider register --namespace Microsoft.KeyVault
```

The registration state was verified using:

```bash
az provider show \
  --namespace Microsoft.KeyVault \
  --query registrationState \
  -o tsv
```

The provider successfully returned:

```text
Registered
```

The Key Vault was then created using:

```bash
az keyvault create \
  --name kv-hybrid-ronak01 \
  --resource-group rg-hybrid-infrastructure \
  --location polandcentral \
  --enable-rbac-authorization true
```

The deployment completed successfully with the following provisioning state:

```text
Provisioning state: Succeeded
RBAC authorization: Enabled
```

The Key Vault URI is:

```text
https://kv-hybrid-ronak01.vault.azure.net/
```

---

## Key Vault Secret

A test secret was created to validate secure workload access.

The secret name is:

```text
hybrid-api-message
```

The secret was created using:

```bash
az keyvault secret set \
  --vault-name kv-hybrid-ronak01 \
  --name hybrid-api-message \
  --value "Secret successfully retrieved from Azure Key Vault"
```

The test value contains no production credential and was used only to validate the security architecture.

No Key Vault secret value, password, token, or application credential is stored in the GitHub repository.

---

## AKS Workload Identity

Microsoft Entra Workload Identity was enabled on the existing AKS cluster.

The cluster configuration was updated using:

```bash
az aks update \
  --resource-group rg-hybrid-infrastructure \
  --name aks-hybrid-prod \
  --enable-oidc-issuer \
  --enable-workload-identity
```

The Workload Identity configuration was verified using:

```bash
az aks show \
  --resource-group rg-hybrid-infrastructure \
  --name aks-hybrid-prod \
  --query "[securityProfile.workloadIdentity.enabled,oidcIssuerProfile.enabled]"
```

The result was:

```text
[
  true,
  true
]
```

This confirmed that both Microsoft Entra Workload Identity and the AKS OIDC issuer are enabled.

The OIDC issuer URL was retrieved using:

```bash
az aks show \
  --resource-group rg-hybrid-infrastructure \
  --name aks-hybrid-prod \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv
```

The OIDC issuer establishes the trust source used by Microsoft Entra ID when Kubernetes ServiceAccount tokens are exchanged for Azure access tokens.

---

## User Assigned Managed Identity

A dedicated user-assigned managed identity was created for the application workload.

The identity configuration is:

```text
Managed Identity name: id-hybrid-api
Resource group: rg-hybrid-infrastructure
Region: Poland Central
Purpose: Azure identity for the hybrid-api Kubernetes workload
```

The identity was created using:

```bash
az identity create \
  --name id-hybrid-api \
  --resource-group rg-hybrid-infrastructure \
  --location polandcentral
```

The identity Client ID and Principal ID were retrieved using:

```bash
CLIENT_ID=$(az identity show \
  --name id-hybrid-api \
  --resource-group rg-hybrid-infrastructure \
  --query clientId \
  -o tsv)

PRINCIPAL_ID=$(az identity show \
  --name id-hybrid-api \
  --resource-group rg-hybrid-infrastructure \
  --query principalId \
  -o tsv)
```

The identifiers are intentionally not stored in the public project documentation because they are not required to understand the architecture.

---

## Key Vault RBAC

Azure role-based access control was used to provide access to the Key Vault.

The application managed identity was granted the following role:

```text
Role: Key Vault Secrets User
Identity: id-hybrid-api
Scope: kv-hybrid-ronak01
```

The role assignment was created using:

```bash
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "$KV_ID"
```

This allows the workload identity to read Key Vault secrets while avoiding unnecessary administrative permissions.

The application identity was not granted permission to manage the Key Vault itself.

This follows the principle of least privilege.

---

## Kubernetes ServiceAccount

A dedicated Kubernetes ServiceAccount was created for the application identity.

The ServiceAccount configuration is:

```text
ServiceAccount: hybrid-api-sa
Namespace: default
```

The ServiceAccount was configured with the Client ID of the user-assigned managed identity.

The manifest was created as:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hybrid-api-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: <MANAGED_IDENTITY_CLIENT_ID>
```

The ServiceAccount was deployed using:

```bash
kubectl apply -f serviceaccount.yaml
```

The configuration was verified using:

```bash
kubectl get serviceaccount hybrid-api-sa \
  -n default \
  -o yaml
```

The ServiceAccount was successfully created in the `default` namespace.

---

## Federated Identity Credential

A federated identity credential was created to establish trust between the Kubernetes ServiceAccount and the Azure managed identity.

The configuration is:

```text
Federated credential name: fic-hybrid-api
Managed identity: id-hybrid-api
Kubernetes namespace: default
ServiceAccount: hybrid-api-sa
Audience: api://AzureADTokenExchange
```

The federated credential subject is:

```text
system:serviceaccount:default:hybrid-api-sa
```

The credential was created using:

```bash
az identity federated-credential create \
  --name fic-hybrid-api \
  --identity-name id-hybrid-api \
  --resource-group rg-hybrid-infrastructure \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:default:hybrid-api-sa" \
  --audiences "api://AzureADTokenExchange"
```

The federated identity configuration was verified using:

```bash
az identity federated-credential list \
  --identity-name id-hybrid-api \
  --resource-group rg-hybrid-infrastructure \
  -o table
```

The credential was successfully listed with:

```text
Name: fic-hybrid-api
Subject: system:serviceaccount:default:hybrid-api-sa
```

---

## Workload Identity Test

A temporary Kubernetes pod was created to validate the complete Workload Identity authentication flow.

The test pod used:

```text
Pod name: workload-identity-test
Namespace: default
ServiceAccount: hybrid-api-sa
Workload Identity: Enabled
Container image: Microsoft Azure CLI
```

The test pod manifest was:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: workload-identity-test
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: hybrid-api-sa
  containers:
    - name: azure-cli
      image: mcr.microsoft.com/azure-cli:latest
      command:
        - /bin/sh
        - -c
        - |
          sleep 3600
```

The pod was deployed using:

```bash
kubectl apply -f workload-identity-test.yaml
```

The pod status was verified using:

```bash
kubectl get pod workload-identity-test
```

The pod successfully entered:

```text
READY: 1/1
STATUS: Running
```

An interactive shell was opened inside the test pod:

```bash
kubectl exec -it workload-identity-test -- /bin/sh
```

Inside the pod, authentication to Microsoft Entra ID was performed using the projected federated token:

```bash
az login \
  --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" \
  --service-principal \
  --username "$AZURE_CLIENT_ID" \
  --tenant "$AZURE_TENANT_ID"
```

Authentication completed successfully.

The pod then retrieved the test secret directly from Azure Key Vault:

```bash
az keyvault secret show \
  --vault-name kv-hybrid-ronak01 \
  --name hybrid-api-message \
  --query value \
  -o tsv
```

The command successfully returned:

```text
Secret successfully retrieved from Azure Key Vault
```

After the successful test, the temporary pod was removed:

```bash
kubectl delete pod workload-identity-test
```

---

## Security Benefits

### No Static Azure Credentials

The Kubernetes workload does not require a stored Azure client secret or password. Authentication is performed using short-lived federated tokens.

### Least Privilege Access

The `id-hybrid-api` managed identity receives only the `Key Vault Secrets User` role required to read application secrets.

### Secret Separation

Application secrets are stored in Azure Key Vault instead of Docker images, GitHub, Kubernetes Deployment YAML, Helm values, or source code.

### Identity Separation

The application uses a dedicated workload identity instead of using the AKS cluster identity. This separates infrastructure permissions from application permissions.

### Federated Authentication

The Kubernetes ServiceAccount token is trusted through Microsoft Entra federation. No permanent Microsoft Entra application password is required.

---

## Network Security Group Review

The Azure Network Security Group configuration was reviewed as part of Phase 11 security hardening.

The workload NSG is:

```text
NSG: tw-app-01NSG
Associated subnet: snet-workload
Associated NIC: tw-app-01VMNic
VM: tw-app-01
VM private IP: 10.10.1.4
```

The NSG is associated with both the workload subnet and the virtual machine network interface.

The custom inbound rule is:

```text
Rule name: Allow-SSH-MyIP
Priority: 100
Protocol: TCP
Port: 22
Source: 134.130.119.93
Destination: Any
Action: Allow
```

SSH access is therefore restricted to the approved management public IP instead of being exposed to the entire internet.

The default Azure inbound rules remain active:

```text
AllowVNetInBound
AllowAzureLoadBalancerInBound
DenyAllInBound
```

No custom outbound security rules are currently configured.

The AKS-managed NSG inside the `MC_...` resource group was reviewed but was not manually modified because it is managed as part of the AKS infrastructure.

---

## Key Vault Network Hardening

The Azure Key Vault network configuration was hardened after the initial Workload Identity validation.

Public access from all networks was removed and the Key Vault firewall was configured to allow access only from selected virtual networks and IP addresses.

The following network restrictions were configured:

```text
Default network action: Deny
Allowed management IP: 134.130.119.93/32
Allowed virtual network: vnet-hybrid-prod
Allowed subnet: snet-aks
Trusted services bypass: Disabled
```

The `Microsoft.KeyVault` service endpoint was enabled on the AKS subnet.

The configuration was verified using:

```bash
az network vnet subnet show \
  --resource-group rg-hybrid-infrastructure \
  --vnet-name vnet-hybrid-prod \
  --name snet-aks \
  --query serviceEndpoints \
  -o table
```

The result confirmed:

```text
Subnet: snet-aks
Service endpoint: Microsoft.KeyVault
Provisioning state: Succeeded
```

The Key Vault network rules were verified using:

```bash
az keyvault network-rule list \
  --name kv-hybrid-ronak01
```

The configuration confirmed:

```text
Default action: Deny
Bypass: None
Allowed IP: 134.130.119.93/32
Allowed VNet/Subnet: vnet-hybrid-prod / snet-aks
```

After the firewall restrictions were applied, access from Azure Cloud Shell was intentionally blocked because Cloud Shell was not running from an authorized network.

The Key Vault returned:

```text
ForbiddenByFirewall
Client address is not authorized
```

This confirmed that the Key Vault firewall was actively enforcing the configured network restrictions.

A second validation test was then performed from inside the AKS cluster.

A temporary Workload Identity test pod was created in the `default` namespace using the `hybrid-api-sa` Kubernetes ServiceAccount.

The pod successfully authenticated to Microsoft Entra ID using the federated token.

The authentication type was:

```text
servicePrincipal
```

The pod then retrieved the `hybrid-api-message` secret from Azure Key Vault.

The returned value was:

```text
Secret successfully retrieved from Azure Key Vault
```

This confirmed that the complete hardened access path was operational:

```text
Unauthorized network
        |
        v
Key Vault Firewall
        |
      BLOCKED


AKS snet-aks
        |
        v
Microsoft.KeyVault Service Endpoint
        |
        v
Workload Identity
        |
        v
Azure RBAC
        |
        v
Azure Key Vault
        |
        v
Secret Retrieval
```

The network hardening test successfully validated:

```text
Key Vault firewall: Enabled
Default network action: Deny
Unauthorized Cloud Shell access: Blocked
AKS subnet service endpoint: Configured
AKS subnet access: Allowed
Workload Identity authentication: Successful
Key Vault RBAC authorization: Successful
Secret retrieval from AKS Pod: Successful
```

This implementation ensures that Key Vault access is protected by both identity-based authorization and network-level restrictions.

---

## Azure RBAC Review

Azure role assignments were reviewed to verify that application and AKS identities follow the principle of least privilege.

The `id-hybrid-api` user-assigned managed identity has the following role assignment:

````text
Identity: id-hybrid-api
Role: Key Vault Secrets User
Scope: kv-hybrid-ronak01

The identity does not have broad subscription or resource-group permissions such as:

```text
Owner
Contributor
User Access Administrator

This limits the application identity to reading secrets from the required Key Vault.

The AKS kubelet managed identity was also reviewed.

Its ACR role assignment is:

```text
Role: AcrPull
Scope: acrhybridronak01

This allows AKS nodes to pull container images from the Azure Container Registry without granting registry management permissions.

The subscription administrator account currently has privileged subscription-level access required for managing the lab environment.

For a production environment, privileged subscription-level roles should be minimized and reviewed regularly.

RBAC review result:

```text
Application managed identity: Least privilege confirmed
Key Vault access scope: Resource-level
AKS kubelet ACR access: AcrPull only
Broad application identity roles: None
RBAC hardening status: Validated

````

## Azure Container Registry Security Review

The Azure Container Registry security configuration was reviewed as part of Phase 11.

The registry configuration is:

```text
Registry: acrhybridronak01
SKU: Standard
Admin user: Disabled
Public network access: Enabled

```

The ACR administrator account is disabled.

This prevents the environment from relying on the registry's static administrator username and password for image access.

AKS accesses the registry using the kubelet managed identity with the following Azure RBAC assignment:

```text
Role: AcrPull
Scope: acrhybridronak01
```

This provides the AKS nodes with permission to pull container images without granting unnecessary registry management permissions.

The current authentication path is:

```text
AKS kubelet identity
        |
        | AcrPull
        v
Azure Container Registry
acrhybridronak01
        |
        v
hybrid-api:v1
```

Public network access remains enabled because the registry uses the Standard SKU and the current lab workflow requires external image push access.

For a production environment, additional network isolation should be considered, such as:

```text
Premium ACR
Private Endpoint
Disabled public network access
Private DNS integration
Controlled CI/CD network access
```

ACR security review result:

```text
ACR admin user: Disabled
Static administrator credentials: Not used
AKS authentication: Managed Identity
AKS registry permission: AcrPull only
Public network access: Enabled
Registry SKU: Standard
Production private networking recommendation: Documented

```

## Microsoft Defender for Cloud Review

Microsoft Defender for Cloud recommendations were reviewed as part of the Phase 11 security assessment.

At the time of the review, the Defender for Cloud dashboard showed:

```text
Critical recommendations: 0
High recommendations: 0
Medium recommendations: 0
Low recommendations: 0
Active attack paths: 0
Overdue recommendations: 0

```

Several additional recommendations were displayed with the status Not evaluated.

Relevant recommendations included:

```text
Azure Backup should be enabled for virtual machines
AKS clusters should have Defender profile enabled
AKS clusters should have the Azure Policy add-on enabled
Container registries should not allow unrestricted network access
Container registries should use private link
Diagnostic logs in Key Vault should be enabled
Diagnostic logs in Kubernetes services should be enabled
Firewall should be enabled on Key Vault
Key vaults should have deletion protection enabled
Kubernetes API server should be configured with restricted access
Microsoft Defender for Key Vault should be enabled
Microsoft Defender for Resource Manager should be enabled
Microsoft Defender for Storage should be enabled
Microsoft Defender for servers should be enabled
Subscription ownership should be reviewed
Virtual networks should be protected with additional network security controls
```

The recommendations were reviewed according to the scope, architecture, cost constraints, and purpose of the lab environment.

The Key Vault firewall recommendation has already been addressed by configuring:

```text
Default network action: Deny
Allowed management IP: Restricted
Allowed AKS subnet: snet-aks
Microsoft.KeyVault service endpoint: Enabled
```

The ACR private networking recommendations were documented as production hardening recommendations. The current registry uses the Standard SKU and public network access remains enabled to support the existing development workflow.

The Azure Backup recommendation will be addressed during Phase 12 - Backup and Recovery.

Paid Microsoft Defender plans were not enabled as part of this lab to avoid unnecessary cost on the Azure for Students subscription.

The following production enhancements are documented for future implementation:

```text
Microsoft Defender for Containers
Microsoft Defender for Servers
Microsoft Defender for Key Vault
Private Endpoint for Azure Container Registry
Restricted AKS API server access
Azure Policy integration for AKS
Centralized diagnostic logging
Additional virtual network perimeter protection
```

Defender for Cloud review result:

```text
Security recommendations reviewed: Yes
Critical findings: None displayed
High-risk findings: None displayed
Key Vault firewall recommendation: Addressed
ACR private networking: Documented for production
Azure Backup: Deferred to Phase 12
Paid Defender plans: Not enabled
Security posture review: Completed
```

## Final Security Baseline

A final security baseline was defined for the Hybrid Infrastructure project after reviewing identity, network access, Azure RBAC, Azure Container Registry, Azure Key Vault, AKS Workload Identity, and Microsoft Defender for Cloud recommendations.

The baseline separates controls that were implemented and validated in the lab from additional production-hardening recommendations.

### Identity and Access Management

The environment uses Microsoft Entra ID and Azure RBAC for access control.

The following controls are implemented:

```text
Microsoft Entra authentication: Used
Azure RBAC: Used
Application workload identity: Dedicated
Static Azure credentials in Kubernetes: Not required
Federated authentication: Enabled
Least privilege for application identity: Confirmed
```

The `hybrid-api` workload uses:

```text
Kubernetes ServiceAccount: hybrid-api-sa
Managed Identity: id-hybrid-api
Federated Identity Credential: fic-hybrid-api
```

The managed identity has only:

```text
Role: Key Vault Secrets User
Scope: kv-hybrid-ronak01
```

No broad roles such as Owner or Contributor are assigned to the application identity.

---

### Network Security

Network access is controlled through Azure Network Security Groups and service-specific network restrictions.

The workload NSG configuration includes:

```text
NSG: tw-app-01NSG
Protected subnet: snet-workload
Protected NIC: tw-app-01VMNic
SSH port: TCP/22
Allowed SSH source: 134.130.119.93
Default inbound deny: Active
```

SSH access is therefore restricted to the approved management IP instead of allowing unrestricted internet access.

The AKS-managed NSG was not manually modified because it is managed by the AKS platform.

---

### Key Vault Security

Azure Key Vault is protected by both identity-based and network-based controls.

The implemented controls are:

```text
Authorization model: Azure RBAC
Soft delete: Enabled
Network default action: Deny
Public access from all networks: Disabled
Allowed management IP: Restricted
Allowed AKS subnet: snet-aks
Microsoft.KeyVault service endpoint: Enabled
Trusted services bypass: Disabled
```

The application accesses Key Vault through Microsoft Entra Workload Identity.

The hardened access path is:

```text
AKS Pod
   |
   v
Kubernetes ServiceAccount
   |
   v
Microsoft Entra Workload Identity
   |
   v
Managed Identity
   |
   v
Azure RBAC
   |
   v
Key Vault Firewall
   |
   v
Azure Key Vault
```

Unauthorized Cloud Shell network access was successfully blocked by the Key Vault firewall.

Authorized access from the AKS workload was successfully validated.

---

### Container Registry Security

Azure Container Registry was reviewed for authentication and authorization security.

The current configuration is:

```text
Registry: acrhybridronak01
SKU: Standard
Admin user: Disabled
Public network access: Enabled
```

AKS accesses the registry using its kubelet managed identity.

The assigned role is:

```text
Role: AcrPull
Scope: acrhybridronak01
```

The registry administrator username and password are not used by the AKS workload.

Private registry networking is documented as a production-hardening recommendation.

---

### Kubernetes Security

The AKS security configuration includes:

```text
OIDC issuer: Enabled
Microsoft Entra Workload Identity: Enabled
Application ServiceAccount: Dedicated
Application Azure identity: Dedicated
Application secrets: Stored outside Kubernetes manifests
Container registry authentication: Managed Identity
```

The application does not require static Azure credentials inside:

```text
Docker images
Kubernetes manifests
Helm values
GitHub repository
Application source code
```

---

### Secret Management

Application secrets are stored in Azure Key Vault rather than directly in the application deployment configuration.

The test secret retrieval workflow was successfully validated from inside AKS.

Validated result:

```text
Federated token authentication: Successful
Microsoft Entra authentication: Successful
Azure RBAC authorization: Successful
Key Vault network authorization: Successful
Secret retrieval: Successful
```

This provides multiple layers of protection for application secrets.

---

### Monitoring and Security Posture

Microsoft Defender for Cloud recommendations were reviewed.

At the time of the review:

```text
Critical recommendations displayed: 0
High recommendations displayed: 0
Active attack paths: 0
Overdue recommendations: 0
```

Several recommendations were shown as `Not evaluated`.

No paid Microsoft Defender plans were enabled as part of the student lab environment.

Relevant recommendations were either:

```text
Implemented
Validated
Deferred to a later project phase
Documented as production hardening
```

---

### Implemented Security Controls

The following controls are currently implemented and validated:

```text
Azure RBAC
Least-privilege application identity
AKS Workload Identity
OIDC federation
Dedicated Kubernetes ServiceAccount
Dedicated User Assigned Managed Identity
Azure Key Vault secret storage
Key Vault firewall
Key Vault subnet restriction
Microsoft.KeyVault service endpoint
Restricted SSH source IP
Default NSG inbound deny
ACR admin account disabled
AKS AcrPull managed identity access
Defender for Cloud security review
```

---

### Production Hardening Recommendations

The following controls are recommended for a production deployment but are outside the current lab scope:

```text
Private Endpoint for Azure Key Vault
Private Endpoint for Azure Container Registry
Premium Azure Container Registry
Disable ACR public network access
Restrict AKS API server access
Enable Azure Policy for AKS
Enable appropriate Microsoft Defender paid plans
Enable centralized diagnostic logging
Enable Key Vault diagnostic logs
Enable AKS diagnostic logs
Deploy additional network perimeter controls
Use privileged identity management for administrative roles
Review subscription Owner assignments regularly
Implement centralized security alerting
```

Private connectivity, stronger network isolation, detailed logging, and least-privilege privileged access are consistent with Microsoft's broader cloud security guidance.

---

### Security Baseline Result

The final Phase 11 security baseline is:

```text
Identity security: Implemented
Workload Identity: Implemented and validated
Azure RBAC: Reviewed
Least privilege: Validated
Key Vault security: Hardened
Key Vault network firewall: Validated
NSG security: Reviewed
SSH exposure: Restricted
ACR authentication: Hardened
ACR permissions: Reviewed
Defender for Cloud: Reviewed
Static application credentials: Not required
Production hardening items: Documented
```

The security controls implemented in this phase provide identity separation, least-privilege authorization, network restriction, secure secret storage, and passwordless workload authentication.

## Current Status

The Azure Key Vault, AKS Workload Identity, NSG review, and Key Vault network-hardening implementation are operational and have been successfully validated.

Current security status:

```text
Azure Key Vault: Deployed
Key Vault authorization model: Azure RBAC
Key Vault soft delete: Enabled

AKS OIDC issuer: Enabled
AKS Workload Identity: Enabled
Managed Identity: id-hybrid-api
Kubernetes ServiceAccount: hybrid-api-sa
Federated Identity Credential: fic-hybrid-api
Key Vault Secrets User RBAC: Configured

Federated authentication: Working
Secret retrieval from AKS Pod: Successful
Static Azure credentials: Not required

Workload NSG: Reviewed
SSH source restriction: 134.130.119.93 only
Default deny inbound: Active

Key Vault firewall: Enabled
Default Key Vault network action: Deny
Allowed AKS subnet: snet-aks
Microsoft.KeyVault service endpoint: Enabled
Unauthorized Cloud Shell access: Blocked
Authorized AKS workload access: Successful
```

The remaining Phase 11 security-hardening tasks are:

```text
Review Azure RBAC assignments
Review ACR access permissions
Review Microsoft Defender for Cloud recommendations
Define final security baseline
```

```text
Azure RBAC assignments: Reviewed
Application identity least privilege: Confirmed
AKS kubelet ACR access: AcrPull only
```

```text
ACR security: Reviewed
ACR admin user: Disabled
AKS ACR authentication: Managed Identity
AKS ACR role: AcrPull only
ACR public network access: Enabled
Production private endpoint recommendation: Documented
```

```text
Microsoft Defender for Cloud: Reviewed
Critical recommendations: 0
High recommendations: 0
Key Vault firewall recommendation: Addressed
ACR network hardening: Production recommendation documented
Azure Backup recommendation: Deferred to Phase 12
Paid Defender plans: Not enabled
```

Phase 11 Security Hardening: Completed

Azure RBAC review: Completed
ACR security review: Completed
Microsoft Defender for Cloud review: Completed
Final security baseline: Defined

Key Vault network hardening: Validated
Workload Identity: Validated
Least privilege: Validated
Production recommendations: Documented
