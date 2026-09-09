# Failure Tests

## Overview

Phase 14 of the Hybrid Infrastructure project validates the resilience and recovery behavior of the Azure Kubernetes Service workload.

The objective of these tests was to verify that the deployed application can recover from controlled failures without causing unnecessary service interruption.

The following failure and recovery scenarios were tested:

```text
Pod deletion and self-healing
Horizontal Pod Autoscaler scale-up and recovery
Application availability during pod failure
Node scheduling failure simulation
Pod rescheduling to a healthy node
```

All tests were performed against the production-style AKS workload deployed through Helm.

Application deployment:

```text
Deployment: hybrid-api-helm
Namespace: default
Minimum replicas: 2
Maximum replicas: 5
Ingress public IP: 20.215.101.192
```

---

## Test 1 - Pod Self-Healing

### Objective

The purpose of this test was to validate Kubernetes self-healing behavior when an application Pod is manually deleted.

Kubernetes Deployments maintain a desired replica count. If one of the managed Pods disappears, Kubernetes creates a replacement Pod to restore the desired state.

### Initial State

The application was running with two replicas:

```text
Deployment: hybrid-api-helm
Desired replicas: 2
Available replicas: 2
Pod status: Running
```

The running Pods were verified using:

```bash
kubectl get deployments
kubectl get pods -o wide
```

### Failure Injection

One application Pod was manually deleted:

```bash
kubectl delete pod <POD_NAME>
```

Example:

```text
pod "hybrid-api-helm-6cbc78698c-c95zb" deleted
```

### Recovery Observation

The Pod state was monitored using:

```bash
kubectl get pods -w
```

Kubernetes immediately created a replacement Pod.

Observed behavior:

```text
Original Pod: Deleted
Replacement Pod: Created
Replacement Pod: Running
Desired replica count: Restored to 2
```

### Result

```text
Pod failure introduced: Successful
Replacement Pod created: Yes
Desired replica count restored: Yes
Application deployment healthy: Yes
Test result: PASSED
```

This test confirmed that the Kubernetes Deployment and ReplicaSet controllers successfully maintained the declared application state.

---

## Test 2 - HPA Load and Recovery

### Objective

The purpose of this test was to validate automatic horizontal scaling when CPU utilization increased and automatic scale-down after the load was removed.

The application Horizontal Pod Autoscaler was configured as:

```text
Minimum replicas: 2
Maximum replicas: 5
Target CPU utilization: 50%
```

The HPA configuration was inspected using:

```bash
kubectl get hpa
```

### Load Generation

A temporary BusyBox Pod was created to continuously send HTTP requests to the application Service:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c \
  'while true; do wget -q -O- http://hybrid-api-helm >/dev/null; done'
```

Application CPU consumption and replica count were monitored using:

```bash
kubectl get hpa
kubectl get pods
kubectl top pods
```

### Scale-Up Observation

During the generated load, CPU utilization exceeded the configured 50% target.

Observed values included:

```text
CPU utilization: 77%
Replicas: 4

CPU utilization: 153%
Replicas: 5
```

The HPA successfully scaled the application from:

```text
2 replicas
    |
    v
4 replicas
    |
    v
5 replicas
```

The configured maximum of five replicas was reached.

### Load Removal

The load generator was removed using:

```bash
kubectl delete pod load-generator
```

The HPA was then monitored again:

```bash
kubectl get hpa -w
```

After the CPU utilization decreased, the replica count returned to the configured minimum:

```text
5 replicas
    |
    v
2 replicas
```

### Result

```text
Initial replicas: 2
Load generation: Successful
CPU target exceeded: Yes
Automatic scale-up: Successful
Maximum replicas reached: 5
Load removed: Successful
Automatic scale-down: Successful
Final replicas: 2
Test result: PASSED
```

This validated both the scale-up and recovery behavior of the Kubernetes Horizontal Pod Autoscaler.

---

## Test 3 - Application Availability During Pod Failure

### Objective

The purpose of this test was to verify that the public application endpoint remains available when one application Pod fails.

The application was exposed through the AKS Application Routing Ingress.

Ingress configuration:

```text
Application: hybrid-api-helm
Ingress class: webapprouting.kubernetes.azure.com
Ingress public IP: 20.215.101.192
Protocol: HTTP
Port: 80
```

Initial connectivity was verified using:

```bash
curl http://20.215.101.192
```

The application returned:

```json
{
  "message": "Hybrid Infrastructure API is running",
  "environment": "Azure Kubernetes Service",
  "status": "healthy"
}
```

### Failure Injection

One of the two application Pods was manually deleted:

```bash
kubectl delete pod <POD_NAME>
```

### Availability Test

Immediately after the Pod deletion, ten consecutive HTTP requests were sent to the Ingress endpoint:

```bash
for i in {1..10}; do
  date '+%H:%M:%S'
  curl -s --max-time 5 http://20.215.101.192
  echo
  sleep 1
done
```

All ten requests successfully returned:

```text
status: healthy
```

No failed HTTP request was observed during the test window.

### Service Endpoint Validation

After Kubernetes recreated the failed Pod, the application Pods were verified:

```bash
kubectl get pods
```

Two healthy Pods were running again.

Service endpoints were also inspected:

```bash
kubectl get endpoints hybrid-api-helm
```

Observed endpoints:

```text
10.244.0.198:3000
10.244.1.99:3000
```

### Result

```text
Pod failure introduced: Successful
Ingress remained reachable: Yes
HTTP requests during failure: 10
Successful HTTP responses: 10
Failed HTTP responses: 0
Replacement Pod created: Yes
Service endpoints healthy: Yes
Application availability maintained: Yes
Test result: PASSED
```

This confirmed that the Kubernetes Service and Ingress continued routing requests to healthy application replicas while the failed Pod was replaced.

---

## Test 4 - Node Scheduling Failure Simulation

### Objective

The purpose of this test was to validate Pod rescheduling when one AKS worker node becomes unavailable for new workload scheduling.

A controlled scheduling failure was used instead of shutting down a real node.

This approach provided the same scheduling validation without unnecessarily disrupting the managed AKS infrastructure.

### Initial Node Distribution

The two application Pods were running on separate AKS worker nodes:

```text
Pod 1
Node: aks-agentpool-34719516-vmss000000

Pod 2
Node: aks-agentpool-34719516-vmss000001
```

Both nodes initially reported:

```text
Status: Ready
```

The placement was verified using:

```bash
kubectl get pods -o wide | grep hybrid-api-helm
kubectl get nodes
```

### Disable Scheduling on Node

The first worker node was marked as unschedulable:

```bash
kubectl cordon aks-agentpool-34719516-vmss000000
```

The node status changed to:

```text
Ready,SchedulingDisabled
```

The second node remained:

```text
Ready
```

### Pod Failure Injection

The application Pod running on the cordoned node was deleted:

```bash
kubectl delete pod hybrid-api-helm-6cbc78698c-5ngwn
```

Because the original node was unschedulable, the replacement Pod could not be placed back onto that node.

### Pod Rescheduling

The replacement Pod was observed using:

```bash
kubectl get pods -o wide -w
```

Kubernetes created a new Pod:

```text
hybrid-api-helm-6cbc78698c-f8w4k
```

The replacement was scheduled on:

```text
aks-agentpool-34719516-vmss000001
```

The remaining original Pod was also running on the same healthy node during the test.

This demonstrated that the Kubernetes scheduler successfully selected another available node.

### Application Availability During Node Scheduling Failure

While the node remained cordoned and the replacement Pod was being created, the public application endpoint was tested repeatedly:

```bash
for i in {1..10}; do
  date '+%H:%M:%S'
  curl -s --max-time 5 http://20.215.101.192
  echo
  sleep 1
done
```

The API continued returning:

```text
status: healthy
```

throughout the test.

### Restore Node Scheduling

After the recovery behavior had been validated, the node was returned to normal scheduling:

```bash
kubectl uncordon aks-agentpool-34719516-vmss000000
```

Final node status:

```text
aks-agentpool-34719516-vmss000000   Ready
aks-agentpool-34719516-vmss000001   Ready
```

### Result

```text
Node scheduling disabled: Successful
Pod on affected node deleted: Successful
Replacement Pod created: Successful
Replacement Pod scheduled on healthy node: Successful
Application availability maintained: Yes
Affected node restored to schedulable state: Successful
Final node state: Ready
Test result: PASSED
```

This test validated controlled node-level workload recovery without directly modifying or shutting down the AKS-managed virtual machine scale set.

---

## Failure Test Architecture

The validated recovery behavior can be summarized as:

```text
                    Client
                      |
                      v
                 Public Ingress
                      |
                      v
               Kubernetes Service
                  /         \
                 /           \
                v             v
             Pod A           Pod B
                |             |
                v             v
             Node 1         Node 2


Pod Failure:
Pod A deleted
      |
      v
ReplicaSet creates replacement
      |
      v
Desired replica count restored


High CPU Load:
CPU > HPA target
      |
      v
2 Pods -> 5 Pods
      |
Load removed
      |
      v
5 Pods -> 2 Pods


Node Scheduling Failure:
Node 1 cordoned
      |
Pod on Node 1 deleted
      |
      v
Replacement Pod scheduled on Node 2
      |
      v
Application remains available
```

---

## Validation Summary

Phase 14 produced the following results:

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

The tests demonstrated that the AKS application can tolerate controlled Pod and scheduling failures while maintaining the declared application state.

---

## Resilience Capabilities Validated

The following Kubernetes resilience capabilities were validated:

```text
Deployment desired-state reconciliation
ReplicaSet Pod replacement
Horizontal Pod Autoscaling
Automatic scale-down after load reduction
Service routing to healthy Pods
Ingress availability during Pod failure
Multi-node workload placement
Pod rescheduling to an available worker node
Controlled node cordon and uncordon
```

---

## Test Scope and Safety

The failure tests were intentionally designed to avoid destructive changes to the AKS-managed infrastructure.

The following operations were not performed:

```text
Manual deletion of AKS-managed virtual machines
Direct modification of the AKS managed resource group
Destructive modification of AKS-managed NSGs
Full worker node shutdown
Destructive Ingress configuration changes
Deletion of the production Helm release
```

A controlled `kubectl cordon` test was used to simulate node scheduling unavailability instead.

This allowed the workload recovery behavior to be tested while maintaining the integrity of the managed AKS environment.

---

## Current Status

Phase 14 failure testing is complete for the lab scope.

Final status:

```text
Application replicas: Healthy
HPA: Operational
Ingress: Operational
Service endpoints: Healthy
AKS worker nodes: Ready
Pod self-healing: Validated
Automatic scaling: Validated
Application availability: Validated
Pod rescheduling: Validated
Controlled node recovery: Validated
```

All planned controlled Kubernetes failure tests completed successfully.

The environment was returned to its normal operational state after testing.