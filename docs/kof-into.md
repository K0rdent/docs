# k0rdent Observability and FinOps (KOF)

## Overview

KOF provides enterprise-grade observability and FinOps capabilities for k0rdent-managed Kubernetes clusters. It enables centralized metrics, logging, and cost management through a unified OpenTelemetry-based architecture.

## Architecture Components

![kof-architecture](https://github.com/k0rdent/kof/blob/main/docs/otel.png)

### 1. Management Layer

- Centralized Grafana dashboard
- Promxy for cross-cluster metrics aggregation
- VictoriaMetrics for alerting and record rules
- Service template deployment engine

### 2. Storage Layer

- Regional VictoriaMetrics clusters
- Dedicated Grafana instances
- VMAuth authentication service
- High-performance data retention

### 3. Collection Layer

- OpenTelemetry collectors
- Automated deployment across clusters
- Secure metric and log collection
- Efficient data routing

## Prerequisites

- k0rdent management cluster
- Infrastructure provider credentials
- Domain for service endpoints
- cert-manager for SSL certificates
- ingress-nginx controller

------------------------------------------------------------------------

## 1. Management Layer chart

- central grafana interface
- promxy to forward calls to multiple downstream regional metrics servers
- local victoriametrics storage for alerting record rules
- k0rdent helmchart definitions and service templates to deploy storage and collectors charts into managedclusters

## 2. Storage Layer chart

Deploys [VictoriaMetrics](https://victoriametrics.com/) storages for metrics and logs.

- Grafana - storage-cluster scoped Grafana instance, deployed and configured with grafana-operator
- vmcluster - metrics storage, ingestion, querying
- vmlogs - logs storage
- vmauth - auth frontend for metrics and logs ingestion and query services

## 3. Collection Layer charts

### a. Operators chart

- opentelemetry-operator - [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/)
- prometheus-operator-crds - [Prometheus Operator](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-operator-crds)

This chart pre-installs all required CRDs to create Opentelemetry Collectors for metrics and logs

### b. Collectors chart

- opentelemetry-collectors - [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) configured to monitor logs and metrics and send them to a storage cluster

------------------------------------------------------------------------

## Implementation Guide

### 1. Storage Cluster Deployment

``` yaml
apiVersion: kcm.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: kof-storage-cluster
  namespace: kof
spec:
  template: kof-storage
  config:
    ingress:
      vmauth:
        host: vmauth.storage.example.com
      grafana:
        host: grafana.storage.example.com
    retention:
      metrics: 30d
      logs: 14d
    storage:
      class: standard
      size: 100Gi
```

### 2. Collector Deployment

``` yaml
apiVersion: kcm.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: kof-collector
  namespace: kof
spec:
  template: kof-collectors
  config:
    storage:
      endpoint: vmauth.storage.example.com
    collection:
      logLevel: info
      metrics:
        scrapeInterval: 30s
      logs:
        excludePaths: 
          - /var/log/lastlog
          - /var/log/tallylog
```

### 3. Access Configuration

#### Grafana Setup

1.  Retrieve access credentials:

``` bash
kubectl get secret -n kof grafana-admin-credentials -o jsonpath="{.data.GF_SECURITY_ADMIN_USER}" | base64 -d
kubectl get secret -n kof grafana-admin-credentials -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 -d
```

2.  Configure data sources:

``` yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: metrics
spec:
  name: VictoriaMetrics
  type: prometheus
  access: proxy
  url: http://vmcluster-vmselect.victoria-metrics:8481/select/0/prometheus
```

## Dashboard Organization

- Login to <http://127.0.0.1:3000/dashboards> with user/pass printed above.

- Open a dashboard.

  <figure>
  <img
  src="PR%20for%20KOF%20User%20Documentation-media/f1c073c2c9425975198ac4c3d212d0eb7910145f.png"
  title="wikilink" alt="Pastedimage20250122181247.png" />
  <figcaption
  aria-hidden="true">Pastedimage20250122181247.png</figcaption>
  </figure>

### 1. Cluster Overview

- Health metrics
- Resource utilization
- Performance trends
- Cost analysis

### 2. Logging Interface

- Real-time log streaming
- Full-text search
- Log aggregation
- Alert correlation

### 3. Cost Management

- Resource cost tracking
- Usage analysis
- Budget monitoring
- Optimization recommendations

## Scaling Guidelines

### Regional Expansion

1.  Deploy storage clusters in new regions
2.  Update Promxy configuration
3.  Configure collector routing

### Cluster Addition

1.  Apply collector template
2.  Verify data flow
3.  Configure custom dashboards

## Maintenance

### Backup Requirements

- Grafana configurations
- Alert definitions
- Custom dashboards
- Retention policies

### Health Monitoring

``` bash
# Check collector status
kubectl get pods -n opentelemetry-system

# Verify storage components  
kubectl get pods -n victoria-metrics

# Monitor Grafana
kubectl get pods -n grafana-system
```

## Architecture Diagram

    ┌───────────────────────┐
    │    KOF Mothership    │
    │ ┌─────────┐ ┌──────┐ │
    │ │ Grafana │ │Promxy│ │
    │ └─────────┘ └──────┘ │
    └───────────┬───────────┘
                │
        ┌───────┴───────┐
        │               │
    ┌───┴───┐       ┌───┴───┐
    │Storage│       │Storage│ Regional
    └───┬───┘       └───┬───┘ Clusters
        │               │
    ┌───┴───┐       ┌───┴───┐
    │Collect│       │Collect│ Managed
    └───────┘       └───────┘ Clusters

## Resource Limits

### Storage Clusters

``` yaml
resources:
  vmcluster:
    limits:
      cpu: 4
      memory: 8Gi
    requests:
      cpu: 2
      memory: 4Gi
```

### Collectors

``` yaml
resources:
  collector:
    limits:
      cpu: 1
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 512Mi
```

## Version Compatibility

| Component       | Version  | Notes                         |
|-----------------|----------|-------------------------------|
| k0rdent         | ≥ 1.0.0  | Required for template support |
| Kubernetes      | ≥ 1.24   | Earlier versions untested     |
| OpenTelemetry   | ≥ 0.81.0 | Recommended minimum           |
| VictoriaMetrics | ≥ 1.91.0 | Required for clustering       |
