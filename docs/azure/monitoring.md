# Prometheus and Grafana Monitoring

## Overview

The monitoring phase of the Hybrid Infrastructure project introduces observability for the Azure Kubernetes Service environment using Prometheus and Grafana.

The goal of this phase is to collect and visualize Kubernetes infrastructure and application metrics, including node CPU usage, node memory usage, pod CPU usage, pod memory usage, resource requests, resource limits, and scaling behavior.

The monitoring stack was deployed directly inside the AKS cluster using the `kube-prometheus-stack` Helm chart.

The monitoring architecture is:

Internet / Administrator
|
v
Grafana
|
v
Prometheus
|
v
AKS Metrics
|
+--> Nodes
+--> Pods
+--> Kubernetes Resources
+--> hybrid-api-helm

## Monitoring Stack Deployment

A dedicated Kubernetes namespace was created for monitoring:

````bash
kubectl create namespace monitoring

The Prometheus Community Helm repository was added:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

The monitoring stack was installed using the kube-prometheus-stack Helm chart:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring

The Helm release was verified using:

```bash
helm list -n monitoring

The monitoring components were then checked using:

```bash
kubectl get pods -n monitoring

The deployment includes the main components required for Kubernetes observability, including Prometheus, Grafana, kube-state-metrics, Prometheus Operator, node-exporter, and Alertmanager.

The kube-prometheus-stack Helm chart also provisions a collection of Kubernetes Grafana dashboards automatically. The chart includes Grafana, kube-state-metrics, and prometheus-node-exporter as dependencies and provides curated dashboards for Kubernetes monitoring.

Grafana Access

Grafana was initially exposed securely using Kubernetes port forwarding rather than creating another public LoadBalancer.

The local port forwarding command was:


```powershell
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

Grafana was then accessed locally from the Windows workstation using:

```text
http://localhost:3000


The Grafana administrator username is:


admin

The administrator password was retrieved securely from the Kubernetes Secret and was not stored in the GitHub repository.

This approach avoids exposing the Grafana administrative interface directly to the public internet during the lab configuration.

Kubernetes Dashboard Validation

The Grafana Kubernetes dashboards were successfully loaded after the monitoring stack installation.

The following dashboard was used to validate application and Kubernetes metrics:


Kubernetes / Compute Resources / Namespace (Pods)

The dashboard was filtered to the following Kubernetes namespace:


default

The hybrid-api-helm application pods were successfully detected by Prometheus and displayed in Grafana.

The monitored pods included the running replicas of:


hybrid-api-helm

The dashboard successfully displayed metrics including:


CPU utilisation
CPU requests
CPU limits
Memory utilisation
Memory requests
Memory limits
Pod-level resource consumption

This confirmed that the Prometheus and Grafana monitoring pipeline was operational.

Application Metrics

The hybrid-api-helm application was monitored through the Kubernetes namespace dashboard.

At the time of validation, both application pods were visible in Grafana and Prometheus was collecting CPU and memory metrics for each replica.

The application uses the following Kubernetes resource configuration:


CPU request: 50m
CPU limit: 250m
Memory request: 64Mi
Memory limit: 256Mi

Grafana displayed the actual application resource usage alongside the configured Kubernetes requests and limits.

This provides visibility into whether the application is over-provisioned, under-provisioned, or approaching configured resource limits.

Monitoring Architecture

The resulting monitoring data flow is:


AKS Nodes and Pods
        |
        v
Prometheus Exporters / Kubernetes Metrics
        |
        v
Prometheus
        |
        v
Grafana
        |
        v
Kubernetes Dashboards

For the application workload, the monitoring path is:


hybrid-api-helm Pods
        |
        v
Kubernetes Metrics
        |
        v
Prometheus
        |
        v
Grafana Dashboard

The monitoring stack therefore provides centralized observability for both the AKS infrastructure and the containerized application.

Validation Result

The Prometheus and Grafana monitoring implementation was successfully validated.

The following monitoring functionality is currently operational:


Prometheus deployment: Successful
Grafana deployment: Successful
Kubernetes metrics collection: Working
AKS node CPU monitoring: Working
AKS node memory monitoring: Working
Pod CPU monitoring: Working
Pod memory monitoring: Working
Resource request monitoring: Working
Resource limit monitoring: Working
hybrid-api-helm pods visible in Grafana: Successful
Kubernetes dashboards: Working
Grafana local access: Successful

The current implementation confirms that the AKS environment can be monitored using a dedicated Prometheus and Grafana observability stack.

Current Status

The AKS monitoring environment is operational.

Prometheus is collecting Kubernetes infrastructure and workload metrics, and Grafana is successfully visualizing the collected data using Kubernetes dashboards.

The hybrid-api-helm application is visible in the monitoring system, and CPU and memory usage can be compared directly with the resource requests and limits configured in the Helm chart.

The remaining monitoring tasks are:


Observe HPA scaling behavior in Grafana under load
Configure application-specific dashboards
Configure alerting rules
Document monitoring failure tests



````
