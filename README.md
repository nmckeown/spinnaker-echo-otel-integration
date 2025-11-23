# Spinnaker Echo OpenTelemetry Integration

A comprehensive observability solution for Spinnaker that integrates Spinnaker Echo webhook events with OpenTelemetry (OTel) to provide real-time metrics, dashboards, and monitoring capabilities for continuous delivery pipelines.

<img width="2016" height="1057" alt="Screenshot from 2025-11-23 15-39-55" src="https://github.com/user-attachments/assets/0f565084-10be-4546-b033-313dfbf99499" />



## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Key Features](#key-features)
- [Why This Solution?](#why-this-solution)
- [Architecture](#architecture)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Deploy Spinnaker](#1-deploy-spinnaker-if-not-already-installed)
  - [Configure Echo Webhook](#2-configure-spinnaker-echo-webhook)
  - [Deploy Collection Components](#3-deploy-collection-components)
  - [Deploy Monitoring Stack](#4-deploy-monitoring-stack)
  - [Access Spinnaker](#5-access-and-configure-spinnaker)
  - [Import Grafana Dashboard](#6-import-grafana-dashboard)
- [Accessing the Stack](#accessing-the-stack)
- [Validation and Testing](#validation-and-testing)
- [Configuration](#configuration)
- [Metrics and Observability](#metrics-and-observability)
- [Directory Structure](#directory-structure)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations-and-considerations)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)
- [License](#license)
- [References](#references)

## Overview

This project provides a production-ready implementation for collecting, processing, and visualizing Spinnaker pipeline execution events using modern observability tools. It transforms Spinnaker's event-driven architecture into actionable metrics for monitoring pipeline performance, failure rates, and DORA metrics.

## Quick Start

For a complete deployment from scratch:

```bash
# 1. Install Spinnaker
kubectl create namespace opsmx-oss
helm repo add spinnaker https://opsmx.github.io/spinnaker-helm/
helm install oss-spin spinnaker/spinnaker -n opsmx-oss --timeout 600s

# 2. Deploy observability stack
kubectl apply -f k8s/collection/
kubectl apply -f k8s/monitoring/

# 3. Configure Echo webhook (after Spinnaker is running)
# Edit Halyard config to add webhook endpoint

# 4. Access Grafana and import dashboard
kubectl port-forward -n monitoring svc/grafana 3000:3000
# Visit http://localhost:3000 and import dashboard/grafana-spinnaker-dashboard.yaml
```

### Key Features

- **Real-time Pipeline Metrics**: Track pipeline executions, completions, failures, and durations
- **DORA Metrics Support**: Calculate deployment frequency, lead time, MTTR, and change failure rate
- **OpenTelemetry Integration**: Modern, vendor-neutral observability framework with advanced log-to-metrics transformation
- **Automated Event Processing**: Transform Spinnaker Echo webhook events into structured metrics without custom code
- **Grafana Dashboards**: Pre-built dashboards for executive and operational visibility
- **Prometheus Integration**: Industry-standard metrics collection and alerting
- **Stage-Level Tracking**: Monitor individual pipeline stages for granular insights
- **Multi-Cloud Support**: Extract cloud provider, region, and environment from pipeline naming conventions
- **Complete Stack Included**: Spinnaker deployment, collection, processing, storage, and visualization

### Why This Solution?

Traditional Spinnaker deployments lack comprehensive observability for pipeline performance. This solution addresses key challenges:

- **No Built-in Metrics**: Spinnaker doesn't provide out-of-the-box DORA metrics or detailed pipeline analytics
- **Event-Driven Architecture**: Leverages Spinnaker's native Echo webhook events without modifying core services
- **Modern Observability**: Uses OpenTelemetry's powerful transformation capabilities to convert logs to metrics
- **Production-Ready**: Includes all necessary components for a complete observability stack
- **Research-Based**: Developed as part of DevOps MSc research at TU Dublin

## Architecture

The solution consists of four main components:

```
Spinnaker Echo → Fluent Bit → OpenTelemetry Collector → Prometheus → Grafana
```

1. **Spinnaker Echo**: Emits webhook events for pipeline lifecycle events (starting, complete, failed)
2. **Fluent Bit**: Receives webhook events via HTTP and forwards them to OTel Collector
3. **OpenTelemetry Collector**: 
   - Transforms logs into structured metrics
   - Extracts pipeline execution attributes
   - Calculates durations and failure rates
   - Generates MTTR metrics
4. **Prometheus**: Scrapes and stores metrics for querying
5. **Grafana**: Visualizes metrics through customizable dashboards

### Architecture Diagrams

The `arch_drawings/` directory contains visual representations of:
- CD observability problem and solution
- Spinnaker OTel integration architecture
- Lab setup topology

## Components

### Fluent Bit
- **Purpose**: HTTP webhook receiver for Spinnaker Echo events
- **Configuration**: `k8s/collection/fluentbit-echo-webhook.yaml`
- **Features**:
  - HTTP input on port 8080
  - JSON parsing
  - OpenTelemetry output via gRPC
  - Buffer management for high-volume events

### OpenTelemetry Collector
- **Purpose**: Transform Spinnaker events into metrics
- **Configuration**: `k8s/collection/otel-collector.yaml`
- **Key Capabilities**:
  - **Log-to-Metrics Conversion**: Uses connectors to transform event logs into gauge and counter metrics
  - **Attribute Extraction**: Parses execution ID, application, name, status, user, trigger type, timestamps
  - **Pipeline Naming Convention**: Extracts cloud provider, region, environment, domain, and instance from pipeline names
  - **Stage Processing**: Unrolls pipeline stages into individual records for stage-level metrics
  - **Duration Calculation**: Computes pipeline execution duration in seconds
  - **MTTR Metrics**: Tracks last failure and success timestamps for MTTR calculations

### Metrics Generated

| Metric Name | Type | Description |
|-------------|------|-------------|
| `spinnaker.pipeline_executions_total` | Counter | Total pipeline executions |
| `spinnaker.execution_starting_total` | Counter | Pipelines starting |
| `spinnaker.execution_completed_total` | Counter | Successful completions |
| `spinnaker.execution_failed_total` | Counter | Failed executions |
| `spinnaker.execution_completed` | Gauge | Completion event marker (for queries) |
| `spinnaker.execution_incomplete` | Gauge | Failure/cancellation event marker |
| `spinnaker_pipeline_complete_duration_seconds` | Gauge | Duration of completed pipelines |
| `spinnaker_pipeline_incomplete_duration_seconds` | Gauge | Duration of failed pipelines |
| `spinnaker_last_failure_ts_seconds` | Gauge | Timestamp of last failure (for MTTR) |
| `spinnaker_last_success_ts_seconds` | Gauge | Timestamp of last success (for MTTR) |
| `spinnaker.stage_starting_total` | Counter | Stage start events |
| `spinnaker.stage_completed_total` | Counter | Stage completions |
| `spinnaker.stage_failed_total` | Counter | Stage failures |

### Prometheus
- **Purpose**: Metrics storage and querying
- **Configuration**: `k8s/monitoring/prometheus.yaml`
- **Scrape Targets**:
  - OpenTelemetry Collector metrics (port 9464)
  - Collector self-monitoring (port 8888)

### Grafana
- **Purpose**: Visualization and dashboards
- **Configuration**: `k8s/monitoring/grafana.yaml`
- **Dashboard**: `dashboard/grafana-spinnaker-dashboard.yaml`
- **Features**:
  - Executive high-level overview
  - Pipeline performance metrics
  - Failure rate trends
  - Stage-level analysis
  - DORA metrics visualization

### Alertmanager
- **Purpose**: Alert routing and management
- **Configuration**: `k8s/monitoring/alertmanager.yaml`

## Prerequisites

- Kubernetes cluster 1.20 or later with at least 4 cores and 16 GB memory (tested with Minikube)
- Helm 3.10.3 or later
- kubectl CLI tool
- Basic understanding of Spinnaker pipelines

## Installation

This repository includes everything needed to set up a complete Spinnaker observability stack.

### 1. Deploy Spinnaker (if not already installed)

This repository includes the OpsMx Spinnaker Helm chart for easy deployment.

#### Option A: Deploy Spinnaker with Helm (Recommended)

```bash
# Add the Spinnaker Helm repository
helm repo add spinnaker https://opsmx.github.io/spinnaker-helm/
helm repo update

# Create namespace
kubectl create namespace opsmx-oss

# Install Spinnaker
helm install oss-spin spinnaker/spinnaker -n opsmx-oss --timeout 600s

# Wait for pods to be ready
kubectl -n opsmx-oss get pods -w
```

#### Option B: Use Local Chart

```bash
# Create namespace
kubectl create namespace opsmx-oss

# Install from local chart
helm install oss-spin ./k8s/spinnaker/charts/spinnaker-opsmx -n opsmx-oss --timeout 600s
```

#### Option C: GitOps Method (Advanced)

For production environments, you can use the GitOps method to manage Spinnaker configuration in Git:

```bash
# Clone standard GitOps Halyard configuration
git clone https://github.com/OpsMx/standard-gitops-halyard.git

# Copy to your own repository and configure
# Update minimal-values.yaml with your Git repository details

# Install with GitOps
helm install oss-spin spinnaker/spinnaker -f minimal-values.yaml --timeout 600s -n opsmx-oss
```

For detailed Spinnaker installation options (including GitOps method and secrets management), see: `k8s/spinnaker/charts/spinnaker-opsmx/README.md`

### 2. Configure Spinnaker Echo Webhook

Before deploying the collection stack, configure Spinnaker Echo to send webhook events. Edit your Spinnaker configuration:

```bash
# If using Halyard, edit the echo configuration
hal config webhook rest edit --url http://fluentbit-http.collection.svc.cluster.local:8080

# Or add this to your Echo configuration directly
```

Alternatively, update the `.hal/config` file in the Halyard pod:

```yaml
webhook:
  rest:
    enabled: true
    url: http://fluentbit-http.collection.svc.cluster.local:8080
```

Apply the configuration:

```bash
hal deploy apply
```

### 3. Deploy Collection Components

```bash
# Create collection namespace and deploy Fluent Bit
kubectl apply -f k8s/collection/fluentbit-echo-webhook.yaml

# Deploy OpenTelemetry Collector
kubectl apply -f k8s/collection/otel-collector.yaml
```

### 4. Deploy Monitoring Stack

```bash
# Deploy Prometheus
kubectl apply -f k8s/monitoring/prometheus.yaml

# Deploy Grafana
kubectl apply -f k8s/monitoring/grafana.yaml

# Deploy Alertmanager (optional)
kubectl apply -f k8s/monitoring/alertmanager.yaml
```

### 5. Access and Configure Spinnaker

```bash
# Port-forward to access Spinnaker UI
kubectl -n opsmx-oss port-forward svc/spin-deck 9000:9000

# Open browser to http://localhost:9000
```

Create some test pipelines to generate events and validate the observability stack.

### 6. Import Grafana Dashboard

1. Access Grafana UI (default: http://localhost:3000)
2. Navigate to Dashboards → Import
3. Upload `dashboard/grafana-spinnaker-dashboard.yaml`
4. Configure Prometheus datasource

## Accessing the Stack

Once deployed, you can access different components:

### Spinnaker UI
```bash
kubectl port-forward -n opsmx-oss svc/spin-deck 9000:9000
# Access at: http://localhost:9000
```

### Grafana (Dashboards)
```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
# Access at: http://localhost:3000
# Default credentials: admin/admin
```

### Prometheus (Metrics Query)
```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Access at: http://localhost:9090
```

### OpenTelemetry Collector Metrics
```bash
kubectl port-forward -n collection svc/otel-collector 9464:9464
# Metrics endpoint: http://localhost:9464/metrics

kubectl port-forward -n collection svc/otel-collector 8888:8888
# Collector self-metrics: http://localhost:8888/metrics
```

## Validation and Testing

After deployment, validate the observability stack is working:

### 1. Verify All Pods are Running

```bash
# Check collection namespace
kubectl get pods -n collection

# Check monitoring namespace
kubectl get pods -n monitoring

# Check Spinnaker namespace
kubectl get pods -n opsmx-oss
```

### 2. Create a Test Pipeline

1. Access Spinnaker UI: http://localhost:9000
2. Create a new application (e.g., "test-app")
3. Create a simple pipeline with wait stages
4. Execute the pipeline

### 3. Verify Events are Flowing

```bash
# Watch Fluent Bit logs for events
kubectl logs -n collection deployment/fluent-bit-echo -f

# You should see JSON events with "orca:pipeline:starting", "orca:pipeline:complete"
```

### 4. Check Metrics in Prometheus

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Query for Spinnaker metrics at http://localhost:9090
# Try queries like:
#   spinnaker_pipeline_executions_total
#   spinnaker_execution_completed_total
```

### 5. View Dashboards in Grafana

```bash
# Access Grafana at http://localhost:3000
# Navigate to the imported Spinnaker dashboard
# You should see pipeline execution data
```

## Configuration

### Pipeline Naming Convention

The OTel Collector extracts metadata from pipeline names using a specific format:

```
<cloud-provider>-<region>-<environment>-<domain>-<instance>
```

Example: `aws-us-east-1-prod-api-gateway`

This convention enables filtering and grouping by:
- Cloud provider (AWS, GCP, Azure)
- Region
- Environment (dev, staging, prod)
- Domain (api, web, data)
- Instance/service name

### Event Types Processed

- `orca:pipeline:starting` - Pipeline execution started
- `orca:pipeline:complete` - Pipeline completed successfully
- `orca:pipeline:failed` - Pipeline failed or was cancelled
- `orca:stage:starting` - Stage started
- `orca:stage:complete` - Stage completed
- `orca:stage:failed` - Stage failed

## Metrics and Observability

### DORA Metrics

The solution supports calculating the four DORA metrics:

1. **Deployment Frequency**: Query `spinnaker.execution_completed_total` with rate functions
2. **Lead Time for Changes**: Use pipeline duration metrics
3. **Mean Time to Recovery (MTTR)**: Calculate from `spinnaker_last_failure_ts_seconds` and `spinnaker_last_success_ts_seconds`
4. **Change Failure Rate**: Ratio of `spinnaker.execution_failed_total` to `spinnaker.pipeline_executions_total`

### Example PromQL Queries

```promql
# Deployment frequency (per hour)
rate(spinnaker.execution_completed_total[1h]) * 3600

# Average pipeline duration (last 5 minutes)
avg(spinnaker_pipeline_complete_duration_seconds)

# Failure rate (percentage)
(sum(rate(spinnaker.execution_failed_total[5m])) / 
 sum(rate(spinnaker.pipeline_executions_total[5m]))) * 100

# MTTR (time between failure and recovery)
spinnaker_last_success_ts_seconds - spinnaker_last_failure_ts_seconds
```

## Directory Structure

```
.
├── arch_drawings/              # Architecture diagrams
│   ├── cd_otel_observability_problem.png
│   ├── cd_otel_observability_solution.png
│   ├── cd_otel_spinnaker_integration.png
│   └── cd_otel_spinnaker_lab.png
├── dashboard/                  # Grafana dashboards
│   └── grafana-spinnaker-dashboard.yaml
├── events/                     # Sample event data (for testing)
├── k8s/                       # Kubernetes manifests
│   ├── collection/            # Data collection components
│   │   ├── fluentbit-echo-webhook.yaml
│   │   └── otel-collector.yaml
│   ├── monitoring/            # Monitoring stack
│   │   ├── alertmanager.yaml
│   │   ├── grafana.yaml
│   │   └── prometheus.yaml
│   └── spinnaker/             # Spinnaker deployment
│       └── charts/
│           └── spinnaker-opsmx/  # OpsMx Spinnaker Helm chart
└── screen_captures/           # Screenshots and documentation
```

## Troubleshooting

### Check Spinnaker Status
```bash
# Check all Spinnaker pods
kubectl get pods -n opsmx-oss

# Check Echo logs specifically
kubectl logs -n opsmx-oss deployment/spin-echo -f

# Check Halyard configuration
kubectl exec -it -n opsmx-oss deployment/spin-halyard -- hal config
```

### Check Fluent Bit Logs
```bash
kubectl logs -n collection deployment/fluent-bit-echo -f
```

### Check OTel Collector Logs
```bash
kubectl logs -n collection deployment/otel-collector -f
```

### Verify Prometheus Targets
```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Visit http://localhost:9090/targets
```

### Test Webhook Endpoint
```bash
# Port-forward Fluent Bit
kubectl port-forward -n collection svc/fluentbit-http 8080:8080

# Send test event
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"test": "event"}'
```

### Verify Echo Webhook Configuration
```bash
# Check if Echo is sending webhooks
kubectl exec -it -n opsmx-oss deployment/spin-halyard -- cat /home/spinnaker/.hal/config | grep -A 10 webhook

# Check Echo service endpoint
kubectl get svc -n opsmx-oss spin-echo
```

### Debug Pipeline Events
```bash
# Watch Fluent Bit for incoming events
kubectl logs -n collection deployment/fluent-bit-echo -f | grep -i "orca:pipeline"

# Check OTel Collector metrics endpoint
kubectl port-forward -n collection svc/otel-collector 9464:9464
curl http://localhost:9464/metrics | grep spinnaker
```

## Known Limitations and Considerations

- **Event Volume**: High-frequency pipeline executions may require tuning buffer sizes in Fluent Bit configuration
- **Stage Unrolling**: Complex pipelines with many stages will generate multiple metric records
- **Naming Convention**: Cloud metadata extraction requires following the documented pipeline naming convention
- **Storage**: Prometheus retention should be configured based on your metrics volume and retention requirements
- **Single Echo Instance**: Current configuration assumes a single Echo webhook endpoint; scale considerations needed for HA setups

## Future Enhancements

Potential areas for expansion:

- **Distributed Tracing**: Add span correlation for end-to-end pipeline tracing
- **Log Aggregation**: Integrate with Loki for centralized log storage
- **Advanced Alerting**: Pre-configured Prometheus alert rules for common failure scenarios
- **Multi-Cluster Support**: Aggregate metrics from multiple Spinnaker instances
- **Custom Dashboards**: Additional dashboard templates for specific use cases

## Contributing

This is a research project for the TU Dublin DevOps MSc program. Contributions, feedback, and suggestions are welcome:

- **Issues**: Report bugs or suggest features via GitHub issues
- **Pull Requests**: Improvements to configurations, documentation, or dashboards
- **Research Collaboration**: Contact for academic collaboration opportunities

For questions or contributions, please refer to the project documentation or reach out through the repository.

## License

This project is part of academic research at Technological University Dublin (TUD).

## References

- [Spinnaker Documentation](https://spinnaker.io/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Fluent Bit](https://docs.fluentbit.io/)
- [Prometheus](https://prometheus.io/docs/)
- [Grafana](https://grafana.com/docs/)
- [DORA Metrics](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)

## Acknowledgments

Part of the DevOps Research Project at Technological University Dublin (TUD).
