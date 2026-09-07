# Azure Kubernetes Service (AKS)

## Overview

Phase 8 of the project introduces a containerized application platform in Azure using Azure Kubernetes Service (AKS). The goal of this phase is to deploy and operate a sample application in a managed Kubernetes environment and integrate it with the existing Azure infrastructure. The application is built as a Docker container image, stored in Azure Container Registry (ACR), and deployed to AKS. The implementation also includes public application access, health probes, Kubernetes self-healing, metrics collection, and Horizontal Pod Autoscaling. The AKS environment is deployed inside the existing Hybrid Infrastructure project in the `Poland Central` region.

## Cluster Configuration

The AKS cluster was created with the following configuration:

```text
Cluster name: aks-hybrid-prod
Resource group: rg-hybrid-infrastructure
Region: Poland Central
Subscription: Azure for Students
Pricing tier: Free
Node pool name: agentpool
Node pool mode: System
Node size: Standard_D2s_v3
Node count: 2
Operating system: Ubuntu
```

The AKS deployment completed successfully and the cluster is currently operational.

The cluster status was verified as:

```text
Power state: Running
Cluster operation status: Succeeded
```

Access to the AKS cluster was configured from Azure Cloud Shell using:

```bash
az aks get-credentials \
  --resource-group rg-hybrid-infrastructure \
  --name aks-hybrid-prod \
  --overwrite-existing
```

The Kubernetes context was successfully added to the kubeconfig.

The worker nodes were then verified using:

```bash
kubectl get nodes
```

Both worker nodes were available and ready to host Kubernetes workloads.

## Network Configuration

The AKS cluster was integrated with the existing Azure virtual network used by the Hybrid Infrastructure project.

The virtual network configuration is:

```text
Virtual network: vnet-hybrid-prod
Address space: 10.10.0.0/16
```

A dedicated subnet was created specifically for Azure Kubernetes Service:

```text
Subnet name: snet-aks
Subnet address range: 10.10.3.0/24
```

The cluster uses Azure CNI Overlay networking.

```text
Network configuration: Azure CNI Overlay
```

Azure CNI Overlay allows AKS worker nodes to use IP addresses from the Azure virtual network while Kubernetes pods use a separate overlay address space.

The Kubernetes service network is configured as:

```text
Kubernetes service address range: 10.0.0.0/16
Kubernetes DNS service IP: 10.0.0.10
```

The current subnet structure of `vnet-hybrid-prod` is:

```text
snet-workload     10.10.1.0/24
snet-management   10.10.2.0/24
snet-aks          10.10.3.0/24
GatewaySubnet     10.10.10.0/24
```

Using a dedicated subnet for AKS keeps Kubernetes networking separated from the workload, management, and VPN gateway subnets.

## ACR Integration

Azure Container Registry was deployed to provide a private container image repository for the Kubernetes workloads.

The configured Azure Container Registry is:

```text
Registry name: acrhybridronak01
Login server: acrhybridronak01.azurecr.io
```

The AKS cluster was connected to Azure Container Registry so that Kubernetes worker nodes can pull private container images directly from ACR.

The integration was validated using:

```bash
az aks check-acr \
  --resource-group rg-hybrid-infrastructure \
  --name aks-hybrid-prod \
  --acr acrhybridronak01.azurecr.io
```

The validation completed successfully:

```text
Validating image pull permission: SUCCEEDED
```

This confirmed that the AKS cluster has the required permissions to pull container images from Azure Container Registry.

## Application Deployment

A sample Node.js REST API named `hybrid-api` was created as the application workload for the AKS environment.

The application exposes the following endpoints:

```text
/
```

and:

```text
/health
```

The `/` endpoint returns information about the application and confirms that it is running inside Azure Kubernetes Service.

The `/health` endpoint is used by Kubernetes health probes to verify that the application is running correctly.

The application was containerized using Docker on a Windows workstation.

The Docker image was built using:

```powershell
docker build -t hybrid-api:v1 .
```

The image was tagged for Azure Container Registry:

```powershell
docker tag hybrid-api:v1 acrhybridronak01.azurecr.io/hybrid-api:v1
```

The image was then pushed to ACR:

```powershell
docker push acrhybridronak01.azurecr.io/hybrid-api:v1
```

The final container image is:

```text
acrhybridronak01.azurecr.io/hybrid-api:v1
```

A Kubernetes Deployment was created for the application with two replicas:

```text
Deployment name: hybrid-api
Replicas: 2
Container port: 3000
```

The deployment also includes readiness and liveness probes using the `/health` endpoint.

The running application pods were verified using:

```bash
kubectl get pods
```

Both application pods successfully entered the `Running` state.

A Kubernetes Service of type `LoadBalancer` was created to expose the application externally.

The service configuration is:

```text
Service name: hybrid-api-service
Service type: LoadBalancer
Service port: 80
Target port: 3000
```

The service was verified using:

```bash
kubectl get services
```

During the initial deployment, the external IP address remained in the `Pending` state.

The issue was investigated using:

```bash
kubectl describe service hybrid-api-service
```

The following Azure error was identified:

```text
PublicIPCountLimitReached
Cannot create more than 3 public IP addresses for this subscription in this region.
```

The Azure for Students subscription had reached the Public IP quota for the `Poland Central` region.

An unused Public IP resource was removed from the Azure environment to release quota.

After the Public IP quota became available, the Kubernetes LoadBalancer service successfully received a public IP address:

```text
134.112.83.223
```

The application was then tested using:

```bash
curl http://134.112.83.223
```

The API successfully returned:

```json
{
  "message": "Hybrid Infrastructure API is running",
  "environment": "Azure Kubernetes Service",
  "status": "healthy"
}
```

This confirmed that the complete application deployment path was operational:

```text
Docker Image
     |
     v
Azure Container Registry
     |
     v
Azure Kubernetes Service
     |
     v
Kubernetes Deployment
     |
     v
Application Pods
     |
     v
LoadBalancer Service
     |
     v
Internet
```

## Self-Healing Test

Kubernetes self-healing functionality was tested by manually deleting one of the running `hybrid-api` pods.

The application pods were first verified using:

```bash
kubectl get pods
```

The deployment was running with two healthy replicas.

One of the application pods was then manually deleted:

```bash
kubectl delete pod <pod-name>
```

Kubernetes immediately detected that the actual number of running replicas was lower than the desired replica count configured in the Deployment.

The Kubernetes Deployment controller automatically created a replacement pod.

The replacement pod successfully entered the `Running` state and the deployment returned to two healthy replicas.

The pods were monitored using:

```bash
kubectl get pods -w
```

This test successfully demonstrated Kubernetes self-healing behavior.

The observed process was:

```text
2 Running Pods
      |
      v
One Pod manually deleted
      |
      v
Deployment detects missing replica
      |
      v
Kubernetes creates replacement Pod
      |
      v
2 Running Pods
```

This confirms that an individual pod failure can be automatically recovered without manual intervention.

## Horizontal Pod Autoscaler Test

A Horizontal Pod Autoscaler (HPA) was configured for the `hybrid-api` deployment to automatically increase or decrease the number of application pods based on CPU utilization.

Before configuring the HPA, Kubernetes metrics were verified using:

```bash
kubectl top pods
```

and:

```bash
kubectl top nodes
```

CPU and memory metrics were successfully returned for both the Kubernetes nodes and the application pods.

The HPA was configured with the following parameters:

```text
Minimum replicas: 2
Maximum replicas: 5
Target CPU utilization: 50%
```

The HPA was created using:

```bash
kubectl autoscale deployment hybrid-api \
  --cpu=50% \
  --min=2 \
  --max=5
```

The HPA configuration was verified using:

```bash
kubectl get hpa
```

Under normal workload conditions, the application CPU utilization remained at approximately:

```text
2% - 3%
```

The deployment therefore remained at its minimum configured replica count:

```text
2 replicas
```

To test automatic horizontal scaling, a temporary load-generator pod was created.

The load generator continuously sent HTTP requests to the internal Kubernetes service:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://hybrid-api-service; done"
```

The HPA was monitored in real time using:

```bash
kubectl get hpa -w
```

During the load test, CPU utilization increased significantly above the configured 50% target.

CPU utilization reached values above 100%, including values greater than the configured HPA target.

Kubernetes automatically increased the number of application replicas from:

```text
2 replicas
```

to:

```text
5 replicas
```

The additional application pods were created automatically by Kubernetes.

The scaling behavior was verified using:

```bash
kubectl get pods
```

After successful scale-up validation, the temporary load-generator pod was removed:

```bash
kubectl delete pod load-generator
```

After the artificial load was removed, CPU utilization returned to approximately:

```text
2%
```

After the HPA stabilization period, Kubernetes automatically reduced the number of replicas from five back to the configured minimum of two.

The complete scaling behavior was:

```text
Normal Load
2 Pods
   |
   v
High CPU Load
5 Pods
   |
   v
Load Removed
2 Pods
```

This test successfully validated both HPA scale-up and scale-down functionality.

## Ingress Configuration

The application was initially exposed directly through a Kubernetes Service of type `LoadBalancer`.

To improve the application routing architecture, the AKS Application Routing add-on was enabled and a Kubernetes Ingress resource was introduced.

The application service was changed from `LoadBalancer` to `ClusterIP`:

````text
Service name: hybrid-api-service
Service type: ClusterIP
Service port: 80
Target port: 3000

A Kubernetes Ingress resource named hybrid-api-ingress was created using the AKS Application Routing ingress class:

```text
webapprouting.kubernetes.azure.com

The Ingress routes incoming HTTP requests to the internal hybrid-api-service, which then forwards traffic to the Node.js application pods.

The resulting traffic flow is:

```text
Internet
   |
   v
AKS Ingress Controller
   |
   v
hybrid-api-ingress
   |
   v
hybrid-api-service
   |
   v
hybrid-api Pods

The Ingress received the following public IP address:

```text
20.215.101.192

Application connectivity through the Ingress was successfully validated using:

```text
curl http://20.215.101.192

The application returned:

```json
{
  "message": "Hybrid Infrastructure API is running",
  "environment": "Azure Kubernetes Service",
  "status": "healthy"
}
````

## Helm Deployment

The Kubernetes application resources were migrated from individually managed YAML manifests to a Helm-based deployment.

A Helm chart named `hybrid-api-chart` was created to manage the application as a single deployable package.

The Helm chart contains templates for:

- Kubernetes Deployment
- ClusterIP Service
- Kubernetes Ingress
- Horizontal Pod Autoscaler

Application configuration such as the container image, replica count, resource requests and limits, service ports, ingress configuration, and HPA settings are managed through the `values.yaml` file.

The chart was validated before deployment using:

````bash
helm lint hybrid-api-chart

The rendered Kubernetes manifests were also reviewed using:

```bash
helm template hybrid-api-helm ./hybrid-api-chart

The application was then deployed using:

```bash
helm install hybrid-api-helm ./hybrid-api-chart

The Helm release was verified using:

```bash
helm list

The release successfully entered the deployed state.

The Helm-managed application was validated through the existing Ingress endpoint:

```bash
curl http://20.215.101.192

The API successfully returned a healthy response.

After validating the Helm deployment, the previous manually managed Deployment, Service, Ingress, and HPA resources were removed.

The application is now fully managed through Helm.

Future configuration changes can be deployed using:

```bash
helm upgrade hybrid-api-helm ./hybrid-api-chart

The complete application can also be removed using:

```bash
helm uninstall hybrid-api-helm

This migration successfully centralized Kubernetes application configuration and simplified application deployment and lifecycle management.
````

## Current Status

The Azure Kubernetes Service environment is currently operational and the main Kubernetes capabilities required for the project have been successfully implemented and validated.

The current AKS status is:

```text
AKS cluster: Running
Cluster deployment: Successful
Worker nodes: 2
Node pool: System
Node size: Standard_D2s_v3
Azure CNI Overlay: Configured
Dedicated AKS subnet: Configured
ACR integration: Successful
Container image build: Successful
Container image push: Successful
Application deployment: Successful
Application replicas: 2
Public LoadBalancer: Working
Public application access: Successful
Readiness probe: Configured
Liveness probe: Configured
Self-healing test: Successful
Kubernetes metrics: Working
Horizontal Pod Autoscaler: Configured
HPA scale-up test: Successful
HPA scale-down test: Successful
```

The following Kubernetes-related tasks remain for the next stages of the project:

```text
Ingress configuration
Helm deployment
Prometheus and Grafana monitoring
```

The current AKS implementation successfully demonstrates container image management, Kubernetes deployment, public application access, self-healing, health monitoring, and automatic horizontal scaling inside the Hybrid Infrastructure environment.

Application Routing add-on: Enabled
Ingress: Configured
Ingress public access: Successful
Backend Service: ClusterIP

```text
Helm chart: Configured
Helm release: Deployed
Helm-managed Deployment: Working
Helm-managed Service: Working
Helm-managed Ingress: Working
Helm-managed HPA: Configured
Manual Kubernetes resources: Removed
```
