# Security

## Overview

Phase 11 of the Hybrid Infrastructure project focuses on securing access between the Azure Kubernetes Service workload and Azure platform services.

The primary security implementation in this phase uses Azure Key Vault together with Microsoft Entra Workload Identity.

The goal is to allow Kubernetes workloads to access Azure resources without storing long-lived passwords, client secrets, or credentials inside container images, Kubernetes manifests, or the GitHub repository.

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
Public network access: Enabled
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

The secret was successfully stored in Azure Key Vault.

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

This confirmed that both:

```text
Microsoft Entra Workload Identity: Enabled
OIDC Issuer: Enabled
```

are active on the AKS cluster.

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

The resulting authorization model is:

```text
id-hybrid-api
      |
      | Key Vault Secrets User
      v
kv-hybrid-ronak01
      |
      v
Read Secrets
```

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

The resulting trust relationship is:

```text
AKS OIDC Issuer
       |
       v
hybrid-api-sa
       |
       v
fic-hybrid-api
       |
       v
id-hybrid-api
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

The pod successfully entered the following state:

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

The identity inside the pod was recognized as a service principal rather than a user account.

The pod then attempted to retrieve the test secret directly from Azure Key Vault:

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

This confirmed that the pod could authenticate to Microsoft Entra ID using Kubernetes Workload Identity and retrieve a Key Vault secret using Azure RBAC.

After the successful test, the temporary pod was removed:

```bash
kubectl delete pod workload-identity-test
```

The test resource was successfully deleted from the cluster.

---

## Security Validation Result

The complete security authentication path was successfully validated.

The tested flow was:

```text
Kubernetes Pod
      |
      v
hybrid-api-sa
      |
      | OIDC ServiceAccount Token
      v
Microsoft Entra ID
      |
      v
fic-hybrid-api
      |
      v
id-hybrid-api
      |
      | Key Vault Secrets User
      v
Azure Key Vault
      |
      v
hybrid-api-message
```

The following security functionality was successfully validated:

```text
Microsoft.KeyVault provider registration: Successful
Azure Key Vault deployment: Successful
Azure RBAC authorization: Enabled
Test secret creation: Successful
AKS OIDC issuer: Enabled
AKS Workload Identity: Enabled
User Assigned Managed Identity: Created
Kubernetes ServiceAccount: Created
Federated Identity Credential: Created
Federated authentication: Successful
Key Vault RBAC assignment: Successful
Authentication from Kubernetes Pod: Successful
Key Vault secret retrieval from Pod: Successful
Static Azure credentials inside Pod: Not required
Test Pod cleanup: Successful
```

---

## Security Benefits

The implemented architecture provides several security advantages.

### No Static Azure Credentials

The Kubernetes workload does not require a stored Azure client secret or password.

Authentication is performed using short-lived federated tokens.

### Least Privilege Access

The `id-hybrid-api` managed identity receives only the `Key Vault Secrets User` role required to read application secrets.

It does not receive administrative permissions over the Key Vault.

### Secret Separation

Application secrets are stored in Azure Key Vault instead of:

```text
Docker images
GitHub repository
Kubernetes Deployment YAML
Helm values.yaml
Source code
```

### Identity Separation

The application uses a dedicated workload identity instead of using the AKS cluster identity.

This separates infrastructure permissions from application permissions.

### Federated Authentication

The Kubernetes ServiceAccount token is trusted through Microsoft Entra federation.

No permanent Entra application password is required.

---

## Current Status

The Azure Key Vault and AKS Workload Identity security implementation is operational and has been successfully validated.

Current security status:

```text
Azure Key Vault: Deployed
Key Vault region: Poland Central
Key Vault authorization model: Azure RBAC
Key Vault soft delete: Enabled
Application secret: Stored securely
AKS OIDC issuer: Enabled
AKS Workload Identity: Enabled
Managed Identity: id-hybrid-api
Kubernetes ServiceAccount: hybrid-api-sa
Federated Identity Credential: fic-hybrid-api
Key Vault Secrets User RBAC: Configured
Federated authentication: Working
Secret retrieval from AKS Pod: Successful
Static Azure credentials: Not required
```

The Workload Identity and Key Vault implementation is considered successfully validated for the current project scope.

The remaining Phase 11 security-hardening tasks are:

```text
Review Azure NSG configuration
Review AKS public exposure
Review Azure RBAC assignments
Review ACR access permissions
Review Key Vault network access
Review Microsoft Defender for Cloud recommendations
Define final security baseline
```

These items will be reviewed before Phase 11 is considered fully complete.
