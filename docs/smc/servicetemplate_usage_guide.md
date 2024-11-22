# ServiceTemplate Usage Guide

MES version21112024

## Overview

ServiceTemplates define reusable service configurations that can be deployed across multiple Kubernetes clusters managed by Project 2A. They support different deployment methods, including Helm charts and raw manifests.

## Usage Patterns

### 1\. Direct Template Usage

```
apiVersion: hmc.mirantis.com/v1alpha1
kind: ServiceTemplate
metadata:
  name: ingress-nginx-v1.11.2
spec:
  helm:
    chartName: mirantis-ha-ingress-nginx
    chartVersion: 1.11.2
```

### 2\. Multi-Cluster Service Deployment

```
apiVersion: hmc.mirantis.com/v1alpha1
kind: MultiClusterService
metadata:
  name: global-nginx
spec:
  services:
    - name: ingress-nginx-v1.11.2
      config:
        controller:
          watchIngressWithoutClass: true
  clusterSelector:
    matchLabels:
      hmc.mirantis.com/managed-by: hmc
  servicesPriority: 100
```

## Template Management

### Version Control

Templates follow semantic versioning (e.g., v1.11.2). Version upgrades can be managed through:

1. Template chains  
2. Direct template updates  
3. Multi-cluster service configurations

### Access Control

```
apiVersion: hmc.mirantis.com/v1alpha1
kind: TemplateManagement
metadata:
  name: template-access
spec:
  allowedNamespaces:
    selector:
      matchLabels:
        environment: production
  serviceTemplateChains:
    - name: ingress-nginx-chain

```

## Priority and Conflict Resolution

* Use `servicesPriority` to manage template precedence  
* Lower numbers have higher priority  
* Set `stopOnConflict: true` to prevent conflicting deployments

## Status Monitoring

Monitor service deployment status through:

```
kubectl get multiclusterservice <name> -o yaml
```

Status conditions include:

* Ready  
* SveltosClusterProfileReady  
* FetchServicesStatusSuccess

## Best Practices

* **TBD**

## Common Operations

### Create New Template

```
kubectl apply -f service-template.yaml
```

### Update Existing Service

```
kubectl patch multiclusterservice <name> --patch '{"spec":{"services":[{"name":"new-template"}]}}' --type=merge
```

### Remove Service

```
kubectl delete multiclusterservice <name>
```

# ServiceTemplate Additional Notes

## Template Lifecycle Management

### ServiceTemplateVersion

```
apiVersion: templates.hmc.mirantis.com/v1alpha1
kind: ServiceTemplateVersion
metadata:
  name: example-service-v1
spec:
  template:
    name: example-service
    version: "1.0.0"
  parentVersion: ""
  configuration:
    helm:
      repository: "https://example.com/charts"
      chart: "example-service"
      version: "1.0.0"
    parameters:
      required:
        - name: replicaCount
          type: integer
          description: "Number of replicas"
          default: 1

```

### ServiceTemplateUpgrade

```
apiVersion: templates.hmc.mirantis.com/v1alpha1
kind: ServiceTemplateUpgrade
metadata:
  name: upgrade-to-v2
spec:
  templateName: example-service
  targetVersion: "2.0.0"
  strategy:
    type: Rolling
    rolling:
      maxUnavailable: 1
      maxSurge: 1
  validation:
    timeout: 5m
  rollback:
    automatic: true
    timeout: 10m
```

## Template Chains

### Chain Definition

```
apiVersion: hmc.mirantis.com/v1alpha1
kind: ServiceTemplateChain
metadata:
  name: service-chain
spec:
  templates:
    - name: service-v1.0.0
    availableUpgrades:
      - name: service-v1.1.0
      - name: service-v1.2.0
```

## Integration with ManagedClusters

### Adding Services to ManagedCluster

```
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: example-cluster
spec:
  template: aws-standalone-cp-v1.1.2
  serviceTemplates:
    - name: ingress-nginx-v1.11.2
      config:
        controller:
          watchIngressWithoutClass: true
```

### 

```

```

## Operational Guidelines (needs further testing)

### Monitoring Template Status

```
# View template status
kubectl get servicetemplate -n hmc-system

# View upgrade status
kubectl get servicetemplateupgrade -n hmc-system

# View deployment status across clusters
kubectl get clustersummary -A
```

### Health Checks

```
healthChecks:
  liveness:
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    initialDelaySeconds: 10
    periodSeconds: 5
```
