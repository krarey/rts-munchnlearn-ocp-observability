# Monitoring the HashiStack on OpenShift Container Platform

## OpenShift Configuration

Intro intro intro

### OpenShift Local

If using OpenShift Local, the monitoring operator is not enabled by default. Enable it before running `crc start`.

```shell
# You may want to increase RAM to the OCP VM to support the added workloads:
crc config set memory 24576
# Enable the cluster monitoring operator
crc config set enable-cluster-monitoring true
# Start OpenShift Local
crc start
```

### Enable monitoring for user-defined projects

OpenShift's cluster monitoring stack allows optional deployment of a second Prometheus instance for monitoring of user-managed projects/namespaces. See the [OpenShift documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/monitoring/enabling-monitoring-for-user-defined-projects) for more info.

Update the [cluster-monitoring-config ConfigMap](./resources/ConfigMap-cluster-monitoring-config.yaml) to set the `enableUserWorkload` feature flag:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

After enabling this flag, a dedicated instance of the Prometheus Operator will be deployed to the `openshift-user-workload-monitoring` project. This operator, in turn, will deploy dedicated instances of Prometheus and Thanos, and watch user-managed projects for `ServiceMonitor`, `PodMonitor`, and `PrometheusMonitor` resources.

> [!WARNING] Persistent Metrics Storage
> By default, this will only store metrics from user-defined projects in ephemeral storage. To protect metrics and alerting state from data loss, the administrator responsible for enabling user workload monitoring should also [configure a persistent volume](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/monitoring/configuring-the-monitoring-stack#configuring-persistent-storage_configuring-the-monitoring-stack) or configure the monitoring stack to [push metrics to a remote system](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/monitoring/configuring-the-monitoring-stack#configuring-remote-write-storage_configuring-the-monitoring-stack) for long-term storage.

### Grant user permissions

Grant the `monitoring-edit` ClusterRole to any users you want to be able to configure project-level monitoring. This allows both the deployment of the ServiceMonitor/PodMonitor resources and the PrometheusRule alerting resources.

```shell
oc adm policy add-role-to-user monitoring-edit <user> -n <project>
```

## Prometheus Metrics via Vault

In the following section, we will use a simple Vault deployment to demonstrate the workflow for Prometheus metrics retrieval. Detailed configuration of the Vault Helm chart is not in the scope of this guide.

### Telemetry Configuration

Vault natively emits Prometheus metrics, so requires no special collector configuration. The Vault configuration only needs to be modified to (optionally) enable unauthenticated access to the metrics endpoint, and configure the desired metric retention time and metric label configuration:

```yaml
server:
  ha:
    raft:
      config: |
        # Other config blocks removed for brevity
        listener "tcp" {
          address         = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file   = "/vault/userconfig/vault-tls/tls.crt"
          tls_key_file    = "/vault/userconfig/vault-tls/tls.key"

          # Enable unauthenticated metrics access (necessary for Prometheus Operator)
          telemetry {
            unauthenticated_metrics_access = "true"
          }
        }

        telemetry {
          prometheus_retention_time = "30s"
          disable_hostname          = true
        }
```

> [!NOTE] Partial Example
> This example has been heavily cut down for clarity. Please reference the complete [sample values.yaml overrides](./reference/values-vault.yaml) included with this repository for more detail.

### Service Monitor

The Vault Helm Chart has built-in support for the Prometheus Operator ServiceMonitor CRD, however its configuration only scrapes metrics from the active server pod. Since Vault secondaries and performance secondaries emit metrics that are relevant when diagnosing cluster performance and request errors, I recommend not using the chart's native support and instead applying a customized ServiceMonitor until [hashicorp/vault-helm#1085](https://github.com/hashicorp/vault-helm/issues/1085) is resolved.

See the [ServiceMonitor-vault.yaml](./resources/ServiceMonitor-vault.yaml) file included in this repository for a complete example.

### Alerts

`PrometheusRule` resources deployed alongside Vault will be automatically picked up by the Prometheus operator and used to create new alerts within Alertmanager. These may be deployed either via the Vault Helm chart, or manually.

#### Alert Configuration - Helm Values

```yaml
serverTelemetry:
  serviceMonitor:
    # Deploy the ServiceMonitor manually, to include standby pods
    enabled: false
  prometheusRules:
      enabled: true
      rules:
        - alert: vault-HighResponseTime
          annotations:
            message: The response time of Vault is over 500ms on average over the last 5 minutes.
          expr: vault_core_handle_request{quantile="0.5", namespace="mynamespace"} > 500
          for: 5m
          labels:
            severity: warning
        - alert: vault-HighResponseTime
          annotations:
            message: The response time of Vault is over 1s on average over the last 5 minutes.
          expr: vault_core_handle_request{quantile="0.5", namespace="mynamespace"} > 1000
          for: 5m
          labels:
            severity: critical
```

#### Alert Configuration - Separate Manifest

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vault
  namespace: vault
spec:
  groups:
  - name: vault
    rules:
    - alert: vault-HighResponseTime
      annotations:
        message: The response time of Vault is over 500ms on average over the last
          5 minutes.
      expr: vault_core_handle_request{quantile="0.5", namespace="mynamespace"} > 500
      for: 5m
      labels:
        severity: warning
    - alert: vault-HighResponseTime
      annotations:
        message: The response time of Vault is over 1s on average over the last 5
          minutes.
      expr: vault_core_handle_request{quantile="0.5", namespace="mynamespace"} > 1000
      for: 5m
      labels:
        severity: critical
```

## OpenTelemetry Metrics via HCP Terraform Agent

Unlike the rest of our product suite, the HCP Terraform Agent uses the OpenTelemetry standard to submit metrics to a configured collection endpoint. The Prometheus instance at the core of OpenShift's monitoring stack does not natively understand the OpenTelemetry metrics format, or provide listeners capable of handling the OpenTelemetry Protocol.

To address this limitation, we will deploy an agent called the OpenTelemetry Collector, which will act as an intermediary to collect metrics events and re-publish them in Prometheus-compatible format.

Red Hat publishes its own distribution of the OpenTelemetry Collector agent, the Red Hat build of OpenTelemetry, whose deployment is conveniently abstracted via a Kubernetes operator available in the OpenShift OperatorHub.

### OpenTelemetry Operator Installation

Install the Red Hat build of OpenTelemetry Operator via OperatorHub, using the OpenShift Console or OpenShift CLI. Instructions are [provided in the OpenShift documentation.](https://docs.openshift.com/container-platform/4.17/observability/otel/otel-installing.html)

Once the operator is deployed and running, we need to configure it to deploy instances of the collector itself. There are three primary methods for deploying the OpenTelemetry collector on Kubernetes:

* Deployment - Deploy a set of collector instances to a centralized namespace, fronted by a Kubernetes service. Provides easy deployment and scaling flexibility, but introduces cross-node traffic and introduces network dependencies as an application submitting metrics is not guaranteed to connect to a collector located on the same host. Additionally, metrics viewed in the OpenShift console are filtered by the project/namespace in which the collector is running, not the project in which the workload submitting metrics is running.
* DaemonSet - Deploys one collector instance per worker in a centralized namespace. Requires exposing a HostPort for the agent, which applications may use the [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/) to discover. Provides strong separation of concerns, but many organizations discourage or forbid the use of HostPort definitions in Kubernetes.
* Sidecar - Injects a sidecar container definition to any pods submitted with the `sidecar.opentelemetry.io/inject: "true"` annotation. Provides the strongest separation of concerns, but more complex monitoring configuration as collectors will be deployed to potentially many namespaces (with the advantage that metrics in the OpenShift console will be visible within the same namespace as the origin application). Also incurs the highest resource cost.

### Collector Deployment

For simplicity in this demonstration, we will configure the OpenTelemetry operator to create collectors using the `Deployment` approach detailed above.

```shell
# Create a new project for the collector deployment
oc new-project observability
```

[Create an `OpenTelemetryCollector` resource](./resources/OpenTelemetryCollector-observability.yaml) instructing the operator to create a new deployment of the OpenTelemetry Collector and launch a single instance:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel
  namespace: observability
spec:
  managementState: managed
  mode: deployment
  observability:
    metrics:
      enableMetrics: true
  config:
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
    processors:
      batch: {}
      memory_limiter:
        check_interval: 5s
        limit_percentage: 80
        spike_limit_percentage: 25
    exporters:
      prometheus:
        endpoint: 0.0.0.0:8889
        resource_to_telemetry_conversion:
          enabled: true
    service:
      telemetry:
        metrics:
          address: ":8888"
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

Of particular note, the `spec.observability.metrics.enableMetrics` setting will automatically configure a matching `ServiceMonitor` resource, which the OpenShift monitoring stack will use to scrape our now-Prometheus-format metrics.

### HCP Terraform Operator Deployment

Ensure you have the `helm` binary installed and available on your shell `PATH`. Configure it to use the HashiCorp Helm Repository:

```shell
# Add the HashiCorp repository
helm repo add hashicorp https://helm.releases.hashicorp.com
# Fetch the latest manifests from all configured repositories
helm repo update
```

Deploy a default configuration of the [HCP Terraform Operator](https://github.com/hashicorp/hcp-terraform-operator) into a new project:

```shell
# Create a new OCP project for the operator
oc new-project hcp-terraform-operator
# Install an instance of the hashicorp/hcp-terraform-operator Helm chart into the configured namespace, using default configuration values
helm install hcp-terraform-operator hashicorp/hcp-terraform-operator --namespace hcp-terraform-operator --set replicaCount=1
```

### Terraform Agent Pool Configuration

[Define a new `AgentPool` resource](./resources/AgentPool-monitoring-demo.yaml), extending the agent container spec to set the `TFC_AGENT_OTLP_ADDRESS` environment variable to the service discovery hostname and listener port of our OpenTelemetry collector instance (in this case, otel-collector.observability, port 4317).

Upon launch, the TFC agent will attempt to connect to the OpenTelemetry Collector. If the connection is unsuccessful, the process will emit an error and refuse to start.

### Viewing Metrics

Since the OpenTelemetry operator automatically configured a ServiceMonitor resource, metrics emitted from TFC agents will become visible within the `observability` namespace following the next successful Prometheus scrape.
