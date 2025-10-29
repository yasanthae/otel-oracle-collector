# OpenTelemetry Oracle Collector

A Kubernetes-ready OpenTelemetry Collector configuration for monitoring Oracle Database metrics. This solution exports Oracle DB metrics to any OTLP-compatible backend using the OpenTelemetry Collector Contrib distribution.

## Overview

This project provides a complete Kubernetes deployment configuration for collecting and exporting Oracle Database metrics using OpenTelemetry. It's designed to work with Oracle Autonomous Databases (using wallet authentication) and traditional Oracle databases.

## Features

- **Oracle Database Monitoring**: Collects comprehensive metrics from Oracle databases using the `oracledb` receiver
- **Kubernetes Native**: Ready-to-deploy Kubernetes manifests with proper resource management
- **Secure Wallet Support**: Built-in support for Oracle Wallet authentication (SSL/TLS)
- **OTLP Export**: Sends metrics to any OpenTelemetry-compatible backend
- **Health Checks**: Includes liveness and readiness probes for reliability
- **Resource Management**: Pre-configured CPU and memory limits
- **Customizable**: Easily configurable collection intervals and resource attributes

## Prerequisites

- Kubernetes cluster (1.19+)
- `kubectl` configured to access your cluster
- Oracle Database (autonomous or traditional)
- Oracle Wallet files (for SSL connections): `cwallet.sso`, `sqlnet.ora`, `tnsnames.ora`
- OTLP-compatible backend endpoint (e.g., Prometheus, Grafana, Jaeger, or custom collector)

## Architecture

```
┌─────────────────┐
│ Oracle Database │
└────────┬────────┘
         │ SSL/Wallet Auth
         │
┌────────▼─────────────────┐
│ OTEL Collector (K8s Pod) │
│  - oracledb receiver     │
│  - batch processor       │
│  - resource processor    │
└────────┬─────────────────┘
         │ OTLP gRPC
         │
┌────────▼───────────┐
│ Your OTLP Backend  │
│ (Observability)    │
└────────────────────┘
```

## Quick Start

### 1. Prepare Oracle Wallet Files

If you're using Oracle Autonomous Database, download the wallet files from Oracle Cloud Console.

Encode your wallet files to base64:

```bash
base64 -i cwallet.sso
base64 -i sqlnet.ora
```

### 2. Configure the Deployment

Edit `otel-oracle-collector.yaml` and replace the following placeholders:

#### Secret: oracle-wallet (lines 14-16)
```yaml
data:
  cwallet.sso: <BASE64_ENCODED_WALLET_FILE>
  sqlnet.ora: <BASE64_ENCODED_SQLNET_ORA_FILE>
```

#### ConfigMap: oracle-wallet-tns (line 26)
```yaml
data:
  tnsnames.ora: |
    <YOUR_TNS_ALIAS> = (description= (retry_count=20)(retry_delay=3)(address=(protocol=tcps)(port=1522)(host=<YOUR_ORACLE_HOST>))(connect_data=(service_name=<YOUR_SERVICE_NAME>))(security=(ssl_server_dn_match=yes)))
```

#### Secret: otel-oracle-secrets (lines 36-37)
```yaml
stringData:
  oracle-user: "<YOUR_ORACLE_USERNAME>"
  oracle-password: "<YOUR_ORACLE_PASSWORD>"
```

#### ConfigMap: otel-collector-config (lines 53, 70, 75)
```yaml
receivers:
  oracledb:
    datasource: "oracle://<USERNAME>:<PASSWORD>@<HOST>:<PORT>/<SERVICE_NAME>?ssl=true&ssl%20verify=false"

processors:
  resource:
    attributes:
      - key: oracle.instance
        value: <YOUR_INSTANCE_NAME>

exporters:
  otlp:
    endpoint: "<YOUR_OTLP_ENDPOINT>:4317"
```

### 3. Deploy to Kubernetes

```bash
kubectl apply -f otel-oracle-collector.yaml
```

This creates:
- Namespace: `monitoring`
- Secrets for Oracle authentication
- ConfigMaps for wallet and collector configuration
- Deployment with OpenTelemetry Collector
- Service to expose the collector

### 4. Verify Deployment

Check the pod status:
```bash
kubectl get pods -n monitoring
```

View logs:
```bash
kubectl logs -n monitoring -l app=otel-collector-oracle -f
```

Check health:
```bash
kubectl port-forward -n monitoring svc/otel-collector-oracle 13133:13133
curl http://localhost:13133
```

## Configuration Details

### Collection Interval
Metrics are collected every 60 seconds by default. To change this, modify:
```yaml
collection_interval: 60s
```

### Resource Attributes
Customize the metadata attached to your metrics:
```yaml
resource:
  attributes:
    - key: service.name
      value: oracle-metrics-collector
    - key: deployment.environment
      value: production
    - key: oracle.instance
      value: <YOUR_INSTANCE_NAME>
```

### Resource Limits
Adjust CPU and memory as needed:
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 200m
    memory: 256Mi
```

## Metrics Collected

The `oracledb` receiver collects various Oracle Database metrics including:
- Database sessions
- CPU usage
- Memory utilization
- Tablespace usage
- Transaction rates
- Wait events
- And more...

For a complete list, refer to the [OpenTelemetry OracleDB Receiver documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/oracledbreceiver).

## Troubleshooting

### Connection Issues
- Verify wallet files are correctly base64 encoded
- Check Oracle credentials in secrets
- Ensure TNS configuration matches your database
- Verify network connectivity from Kubernetes to Oracle

### Pod Not Starting
```bash
kubectl describe pod -n monitoring -l app=otel-collector-oracle
kubectl logs -n monitoring -l app=otel-collector-oracle
```

### Metrics Not Appearing
- Check debug exporter logs for metric output
- Verify OTLP endpoint is accessible
- Ensure Oracle user has appropriate permissions:
  ```sql
  GRANT SELECT ON V_$SESSION TO your_user;
  GRANT SELECT ON V_$SYSSTAT TO your_user;
  GRANT SELECT ON DBA_TABLESPACES TO your_user;
  -- Additional grants as needed
  ```

## Security Considerations

- Secrets are used for sensitive data (credentials, wallet)
- Wallet files are mounted read-only
- Consider using Kubernetes secrets management solutions (e.g., Sealed Secrets, External Secrets Operator)
- Review and adjust network policies as needed
- Use RBAC to restrict access to the monitoring namespace

## Exporter Configuration

The collector can export metrics to various backends. Configure the exporter in the `otel-collector-config` ConfigMap (line 73-85).

### Available Exporters

#### 1. OTLP (OpenTelemetry Protocol)
Export to any OTLP-compatible backend (default configuration):

```yaml
exporters:
  otlp:
    endpoint: "your-backend.example.com:4317"  # gRPC endpoint
    tls:
      insecure: true                            # Set to false for production with TLS
      cert_file: /path/to/cert.pem              # Optional: client certificate
      key_file: /path/to/key.pem                # Optional: client key
    headers:
      api-key: "your-api-key"                   # Optional: authentication headers
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
```

**Compatible with:**
- OpenTelemetry Collector
- Grafana Cloud
- Datadog
- New Relic
- Honeycomb
- Any OTLP-compatible platform

#### 2. OTLP HTTP
Use HTTP instead of gRPC:

```yaml
exporters:
  otlphttp:
    endpoint: "https://your-backend.example.com:4318"
    compression: gzip
    headers:
      authorization: "Bearer <token>"
```

#### 3. Prometheus Remote Write
Export directly to Prometheus:

```yaml
exporters:
  prometheusremotewrite:
    endpoint: "http://prometheus-server:9090/api/v1/write"
    tls:
      insecure: false
    headers:
      X-Scope-OrgID: "tenant1"
```

#### 4. Prometheus Exporter
Expose metrics endpoint for Prometheus to scrape:

```yaml
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: oracle_db
    const_labels:
      environment: production
```

Then add to your Prometheus scrape config:
```yaml
scrape_configs:
  - job_name: 'otel-oracle'
    static_configs:
      - targets: ['otel-collector-oracle.monitoring.svc.cluster.local:8889']
```

#### 5. Logging Exporter
Export to stdout/files for debugging:

```yaml
exporters:
  logging:
    loglevel: info
    sampling_initial: 5
    sampling_thereafter: 200
```

#### 6. File Exporter
Write metrics to files:

```yaml
exporters:
  file:
    path: /var/log/otel/metrics.json
    rotation:
      max_megabytes: 100
      max_days: 7
      max_backups: 3
```

### Multiple Exporters

Send metrics to multiple destinations simultaneously:

```yaml
exporters:
  otlp:
    endpoint: "primary-backend:4317"
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"
  logging:
    loglevel: debug

service:
  pipelines:
    metrics:
      receivers: [oracledb]
      processors: [batch, resource]
      exporters: [otlp, prometheusremotewrite, logging]  # Send to all three
```

### Popular Backend Configurations

#### Grafana Cloud
```yaml
exporters:
  otlp:
    endpoint: "otlp-gateway-prod-us-central-0.grafana.net:443"
    headers:
      authorization: "Basic <base64-encoded-credentials>"
```

#### Datadog
```yaml
exporters:
  otlp:
    endpoint: "api.datadoghq.com:4317"
    headers:
      dd-api-key: "<your-datadog-api-key>"
```

#### New Relic
```yaml
exporters:
  otlp:
    endpoint: "otlp.nr-data.net:4317"
    headers:
      api-key: "<your-new-relic-license-key>"
```

#### Elasticsearch
```yaml
exporters:
  elasticsearch:
    endpoints: ["http://elasticsearch:9200"]
    index: oracle-metrics
    pipeline: otel-pipeline
```

## Customization

### Using Without Wallet (Non-SSL Connection)
Remove wallet volume mounts and update the datasource connection string:
```yaml
datasource: "oracle://<USERNAME>:<PASSWORD>@<HOST>:<PORT>/<SERVICE_NAME>"
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Built with [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
- Uses the [OracleDB Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/oracledbreceiver)

## Support

For issues and questions:
- Open an issue in this repository
- Check [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- Visit [Oracle Database documentation](https://docs.oracle.com/)

## Roadmap

- [ ] Helm chart for easier deployment
- [ ] Additional processors (filtering, sampling)
- [ ] Pre-built Grafana dashboards
- [ ] Multi-database support in single deployment
- [ ] Automated wallet rotation support
