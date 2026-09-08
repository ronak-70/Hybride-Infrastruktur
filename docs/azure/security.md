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
