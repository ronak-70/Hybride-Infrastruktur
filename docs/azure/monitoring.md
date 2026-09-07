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

```text
admin

The administrator password was retrieved securely from the Kubernetes Secret and was not stored in the GitHub repository.

This approach avoids exposing the Grafana administrative interface directly to the public internet during the lab configuration.

Kubernetes Dashboard Validation

The Grafana Kubernetes dashboards were successfully loaded after the monitoring stack installation.

The following dashboard was used to validate application and Kubernetes metrics:

```text
Kubernetes / Compute Resources / Namespace (Pods)

The dashboard was filtered to the following Kubernetes namespace:

```text
default

The hybrid-api-helm application pods were successfully detected by Prometheus and displayed in Grafana.

The monitored pods included the running replicas of:

```text
hybrid-api-helm

The dashboard successfully displayed metrics including:

```text
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

```text
CPU request: 50m
CPU limit: 250m
Memory request: 64Mi
Memory limit: 256Mi

Grafana displayed the actual application resource usage alongside the configured Kubernetes requests and limits.

This provides visibility into whether the application is over-provisioned, under-provisioned, or approaching configured resource limits.

Monitoring Architecture

The resulting monitoring data flow is:

```text
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

```text
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

```text
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

```text
Observe HPA scaling behavior in Grafana under load
Configure application-specific dashboards
Configure alerting rules
Document monitoring failure tests


## Grafana Alerting and Webhook Notification

Grafana Alerting was configured to monitor CPU utilization of the `hybrid-api-helm` application.

A Grafana-managed alert rule named `hybrid-api-high-cpu` was created using Prometheus as the data source.

The alert rule monitors CPU utilization relative to the configured Kubernetes CPU requests for the application pods.

The alert condition was configured as:

```text
Alert rule: hybrid-api-high-cpu
Target application: hybrid-api-helm
Namespace: default
CPU threshold: 80%
Evaluation interval: 1 minute
Pending period: 2 minutes
Severity: warning
Environment: lab

The alert rule uses the following labels:

```
service = hybrid-api
severity = warning
environment = lab

A Webhook contact point named hybrid-api-webhook was created in Grafana.

A notification policy was configured to route alerts containing the following label:

```
service = hybrid-api

to the hybrid-api-webhook contact point.

Webhook.site was used as a temporary external endpoint to validate delivery of Grafana alert notifications.

To test the alert, a temporary Kubernetes load-generator pod was used to generate continuous HTTP traffic against the application.

During the load test, CPU utilization increased above the configured 80% threshold.

The alert state changed through the following sequence:

Normal
   |
   v
Pending
   |
   v
Firing

The alert successfully entered the Firing state after the CPU threshold remained exceeded for the configured pending period.

Grafana then delivered an HTTP POST notification to the configured webhook endpoint.

Webhook.site successfully received the notification requests, confirming that the complete alerting and notification path was operational.

The validated notification flow is:

hybrid-api-helm Pods
        |
        v
Prometheus Metrics
        |
        v
Grafana Alert Rule
        |
        v
Notification Policy
        |
        v
Webhook Contact Point
        |
        v
Webhook.site

The test successfully validated:

High CPU detection: Successful
Alert evaluation: Successful
Pending state: Successful
Firing state: Successful
Notification policy routing: Successful
Webhook contact point: Successful
Webhook POST delivery: Successful

This implementation demonstrates automated monitoring and external notification delivery for application-level resource conditions in the AKS environment.

````
## Current Status

The Prometheus and Grafana monitoring environment is operational and has been successfully validated against the AKS infrastructure and the `hybrid-api-helm` application.

Current monitoring status:

```text
Prometheus deployment: Successful
Grafana deployment: Successful
Kubernetes metrics collection: Working
AKS node monitoring: Working
Pod CPU monitoring: Working
Pod memory monitoring: Working
Resource requests and limits monitoring: Working
hybrid-api-helm monitoring: Successful
HPA monitoring under load: Successful
Grafana alerting: Configured
High CPU alert: Successful
Notification policy: Configured
Webhook contact point: Working
External webhook delivery: Successful

The monitoring and alerting implementation is considered complete for the current project scope.