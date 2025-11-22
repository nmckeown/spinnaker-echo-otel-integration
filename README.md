# Spinnaker Echo OpenTelemetry Integration

A comprehensive observability solution for Spinnaker that integrates Spinnaker Echo webhook events with OpenTelemetry (OTel) to provide real-time metrics, dashboards, and monitoring capabilities for continuous delivery pipelines.

## Overview

This project provides a production-ready implementation for collecting, processing, and visualizing Spinnaker pipeline execution events using modern observability tools. It transforms Spinnaker's event-driven architecture into actionable metrics for monitoring pipeline performance, failure rates, and DORA metrics.

### Key Features

- **Real-time Pipeline Metrics**: Track pipeline executions, completions, failures, and durations
- **DORA Metrics Support**: Calculate deployment frequency, lead time, MTTR, and change failure rate
- **OpenTelemetry Integration**: Modern, vendor-neutral observability framework
- **Automated Event Processing**: Transform Spinnaker Echo webhook events into structured metrics
- **Grafana Dashboards**: Pre-built dashboards for executive and operational visibility
- **Prometheus Integration**: Industry-standard metrics collection and alerting
- **Stage-Level Tracking**: Monitor individual pipeline stages for granular insights

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
- **Configuration**: `k8s/monitoring/prometheus-minikube.yaml`
- **Scrape Targets**:
  - OpenTelemetry Collector metrics (port 9464)
  - Collector self-monitoring (port 8888)

### Grafana
- **Purpose**: Visualization and dashboards
- **Configuration**: `k8s/monitoring/grafana-minikube.yaml`
- **Dashboard**: `dashboard/grafana-spinnaker-dashboard.yaml`
- **Features**:
  - Executive high-level overview
  - Pipeline performance metrics
  - Failure rate trends
  - Stage-level analysis
  - DORA metrics visualization

### Alertmanager
- **Purpose**: Alert routing and management
- **Configuration**: `k8s/monitoring/alertmanager-minikube.yaml`

## Prerequisites

- Kubernetes cluster (tested with Minikube)
- Spinnaker deployment with Echo service configured
- kubectl CLI tool
- Basic understanding of Spinnaker pipelines

## Installation

### 1. Deploy Collection Components

```bash
# Create collection namespace and deploy Fluent Bit
kubectl apply -f k8s/collection/fluentbit-echo-webhook.yaml

# Deploy OpenTelemetry Collector
kubectl apply -f k8s/collection/otel-collector.yaml
```

### 2. Deploy Monitoring Stack

```bash
# Deploy Prometheus
kubectl apply -f k8s/monitoring/prometheus-minikube.yaml

# Deploy Grafana
kubectl apply -f k8s/monitoring/grafana-minikube.yaml

# Deploy Alertmanager (optional)
kubectl apply -f k8s/monitoring/alertmanager-minikube.yaml
```

### 3. Configure Spinnaker Echo

Configure Spinnaker Echo to send webhook events to Fluent Bit:

```yaml
# In your Spinnaker Echo configuration
rest:
  enabled: true
  endpoints:
    - wrap: false
      url: http://fluentbit-http.collection.svc.cluster.local:8080
```

### 4. Import Grafana Dashboard

1. Access Grafana UI (default: http://localhost:3000)
2. Navigate to Dashboards → Import
3. Upload `dashboard/grafana-spinnaker-dashboard.yaml`
4. Configure Prometheus datasource

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
│   │   ├── alertmanager-minikube.yaml
│   │   ├── grafana-minikube.yaml
│   │   └── prometheus-minikube.yaml
│   └── spinnaker/             # Spinnaker-specific configs
└── screen_captures/           # Screenshots and documentation
```

## Troubleshooting

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

## Contributing

This is a research project for the TU Dublin DevOps MSc program. For questions or contributions, please refer to the project documentation.

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
