# Redpanda Connect Kubernetes Deployment

GitOps configurations for deploying Redpanda Connect on Kubernetes using ArgoCD.

## Repository Structure

```
.
├── standalone/                              # Standalone deployment mode
│   ├── argocd-rpcn-standalone.yaml         # ArgoCD Application manifest
│   └── standalone-mode.yaml                # Helm values for standalone mode
├── streams/                                 # Streams deployment mode
│   ├── argocd-rpcn-streams.yaml            # ArgoCD Application manifest
│   ├── kustomization.yaml                  # Kustomize configuration
│   ├── streams-mode.yaml                   # Helm values for streams mode
│   └── config/                             # Pipeline configurations
│       ├── first-names.yaml                # Pipeline to extract first names
│       └── last-names.yaml                 # Pipeline to extract last names
└── observability/                           # Monitoring stack
    ├── argocd-observability.yaml           # ArgoCD Application manifest
    ├── kustomization.yaml                  # Kustomize configuration for Prometheus/Grafana
    ├── servicemonitor.yaml                 # ServiceMonitor for Redpanda Connect metrics
    ├── redpanda-connect-dashboard.json     # Grafana dashboard for Redpanda Connect
    └── values.yaml                         # Helm values for kube-prometheus-stack
```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- kubectl configured with cluster access
- ArgoCD CLI (optional)
- Redpanda (for pipeline input/output. Get started [here](https://docs.redpanda.com/current/get-started/quick-start/) or try [Serverless](https://www.redpanda.com/product/serverless) free for 14 days)
- For streams mode: Kubernetes secret with Redpanda credentials (see [Secret Management](#secret-management))

## Deployment Modes

This repository supports two deployment modes for Redpanda Connect:

### Standalone Mode

Deploys Redpanda Connect with a single embedded pipeline configuration using Helm.

**Features:**
- Generates fake names every second
- Processes names to uppercase
- Outputs to stdout and a Redpanda topic
- Kubernetes secret integration for credential management

**Deploy:**
```bash
# First, create the namespace and secret with your Redpanda credentials
kubectl create namespace redpanda-connect

kubectl create secret generic redpanda-password \
  --from-literal=RP_PASSWORD=<your-password> \
  --namespace redpanda-connect

kubectl apply -f standalone/argocd-rpcn-standalone.yaml
```

**Configuration:** `standalone/standalone-mode.yaml`

### Streams Mode

Deploys Redpanda Connect with multiple pipeline configurations managed via ConfigMaps.

**Features:**
- Kustomize-based configuration management
- Multiple independent pipeline configurations (first-names and last-names extraction)
- Kubernetes secret integration for credential management
- Automatic ConfigMap hash updates for rolling deployments
- Separate pipeline files for easier management
- Prometheus metrics enabled

**Deploy:**
```bash
kubectl apply -f streams/argocd-rpcn-streams.yaml
```

**Configuration:** `streams/streams-mode.yaml`

### Observability

Deploys a complete monitoring stack using kube-prometheus-stack (Prometheus + Grafana) with pre-configured dashboards and metric scraping for Redpanda Connect.

**Features:**
- Prometheus for metrics collection and alerting
- Grafana for visualization and dashboards
- Pre-configured Redpanda Connect dashboard
- ServiceMonitor for automatic metrics discovery
- Deployed in dedicated observability namespace

**Deploy:**
```bash
kubectl create namespace observability
kubectl apply -f observability/argocd-observability.yaml
```

**Access Grafana:**
```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:80
# Default credentials: admin / password
```

**Configuration:** `observability/values.yaml`

## Pipeline Configurations

### first-names.yaml
Consumes messages from a Redpanda topic and extracts first names from full name data.

**Features:**
- Connects to Redpanda Cloud with TLS and SASL authentication
- Consumes from `rpcn-standalone-topic` using consumer group `rpcn-gitops-first-names`
- Extracts first name by splitting on space and taking the first element
- Outputs to stdout

**Configuration:**
```yaml
input:
  redpanda:
    seed_brokers: ["seed-xxx.byoc.prd.cloud.redpanda.com:9092"]
    tls:
      enabled: true
    sasl:
      - mechanism: SCRAM-SHA-256
        username: rpcn-gitops
        password: ${RP_PASSWORD}
    topics: [rpcn-standalone-topic]
    consumer_group: rpcn-gitops-first-names
pipeline:
  processors:
    - mapping: |
        root.first_name = this.name.split(" ").index(0)
output:
  stdout:
    codec: lines
```

### last-names.yaml
Consumes messages from a Redpanda topic and extracts last names from full name data.

**Features:**
- Same connection configuration as first-names.yaml
- Consumes from `rpcn-standalone-topic` using consumer group `rpcn-gitops-last-names`
- Extracts last name by splitting on space and taking the second element
- Outputs to stdout

## ArgoCD Integration

Both deployment modes use ArgoCD for continuous delivery with the following settings:

- **Source Repository:** https://github.com/bfbarkhouse/redpanda-connect-gitops
- **Target Branch:** main
- **Target Namespace:** redpanda-connect (auto-created)
- **Auto-sync:** Enabled with prune and self-heal

### Standalone Mode Details

Uses ArgoCD's multi-source feature:
1. Helm chart from https://charts.redpanda.com (version 3.1.0)
2. Values files from the GitOps repository

### Streams Mode Details

Uses Kustomize to:
1. Generate ConfigMaps from pipeline files with hash suffixes
2. Deploy Helm chart with custom values
3. Automatically trigger rolling updates when pipelines change

### Observability Mode Details

Uses Kustomize with server-side apply enabled:
- Deploys kube-prometheus-stack via Helm
- Generates ConfigMap for Grafana dashboard
- Uses `ServerSideApply=true` sync option for better handling of large CRDs

### Monitoring ArgoCD Applications

```bash
# Port forward Argo CD and login
kubectl port-forward -n argocd svc/argocd-server 8080:80
argocd login

# List applications
argocd app list

# Get application status
argocd app get redpanda-connect-standalone
argocd app get redpanda-connect-streams
argocd app get observability

# Sync manually if needed
argocd app sync redpanda-connect-standalone
argocd app sync redpanda-connect-streams
argocd app sync observability
```

## Kustomization

### Adding New Pipelines (Streams Mode)

1. Create a new YAML file in `streams/config/` with your pipeline configuration:
   ```yaml
   input:
     # your input configuration
   pipeline:
     processors:
       # your processing logic
   output:
     # your output configuration
   ```

2. Update `streams/kustomization.yaml` to include the new file:
   ```yaml
   configMapGenerator:
     - name: connect-streams
       files:
         - config/first-names.yaml
         - config/last-names.yaml
         - config/your-new-pipeline.yaml
   ```

3. Commit and push changes
4. ArgoCD will automatically sync and deploy with the new pipeline

### Secret Management

The pipelines use Kubernetes secrets for sensitive credentials like passwords. To set up the secret:

```bash
kubectl create secret generic redpanda-password \
  -n redpanda-connect \
  --from-literal=RP_PASSWORD='your-password-here'
```

The secret is referenced in `streams-mode.yaml`:
```yaml
envFrom:
  - secretRef:
      name: redpanda-password
```

This makes the `RP_PASSWORD` environment variable available to all pipelines, which can be referenced using `${RP_PASSWORD}` in pipeline configurations.

### Modifying Helm Values

Edit the respective values files:
- `standalone/standalone-mode.yaml` for standalone mode
- `streams/streams-mode.yaml` for streams mode

Common modifications:
```yaml
deployment:
  replicaCount: 1              # Number of replicas
logger:
  level: INFO                  # Log level (INFO, DEBUG, etc.)
  static_fields:
    '@service': my-service     # Static log fields
metrics:
  prometheus: {}               # Prometheus metrics configuration
envFrom:                       # Environment variables from secrets
  - secretRef:
      name: redpanda-password  # Reference to Kubernetes secret
```

### Updating Helm Chart Version

For standalone mode, edit `standalone/argocd-rpcn-standalone.yaml`:
```yaml
sources:
  - repoURL: https://charts.redpanda.com
    chart: connect
    targetRevision: 3.1.0      # Update this version
```

For streams mode, edit `streams/kustomization.yaml`:
```yaml
helmCharts:
  - name: connect
    version: 3.1.0             # Update this version
```

## Monitoring and Observability

The deployments include:
- Prometheus metrics endpoint enabled
- Configurable log levels (default: INFO)
- Static log fields for better traceability

### Observability Stack

Deploy the complete monitoring stack (Prometheus + Grafana) to visualize metrics:

```bash
kubectl apply -f observability/argocd-observability.yaml
```

The observability stack includes:
- **Prometheus**: Automatically scrapes metrics from Redpanda Connect via ServiceMonitor
- **Grafana**: Pre-configured with Redpanda Connect dashboard
- **AlertManager**: For alerting and notifications

Access Grafana dashboard:
```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:80
# Open http://localhost:3000
# Default credentials: admin / password
```

Access Prometheus:
```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090
```

### View Logs
```bash
kubectl logs -n redpanda-connect -l app.kubernetes.io/instance=redpanda-connect-standalone -f
kubectl logs -n redpanda-connect -l app.kubernetes.io/instance=redpanda-connect-streams -f
```

### Check Metrics Directly
```bash
kubectl port-forward -n redpanda-connect svc/redpanda-connect-streams 8081:80
curl http://localhost:8081/metrics
```

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n redpanda-connect
kubectl describe pod -n redpanda-connect <pod-name>
kubectl logs -n redpanda-connect <pod-name>
```

### Check ArgoCD Application Health
```bash
kubectl get application -n argocd
kubectl describe application -n argocd redpanda-connect-standalone
kubectl describe application -n argocd redpanda-connect-streams
```

### View ConfigMaps (Streams Mode)
```bash
kubectl get configmap -n redpanda-connect
kubectl describe configmap -n redpanda-connect connect-streams-<hash>
kubectl get configmap -n redpanda-connect connect-streams-<hash> -o yaml
```

### Application Won't Sync
1. Check ArgoCD application status:
   ```bash
   argocd app get redpanda-connect-streams
   argocd app get observability
   ```
2. Verify repository access and credentials
3. Check for syntax errors in YAML files
4. Review ArgoCD logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
   ```

### Grafana Dashboard Not Showing Data
1. Verify ServiceMonitor is created:
   ```bash
   kubectl get servicemonitor -n observability
   ```
2. Check Prometheus targets:
   ```bash
   kubectl port-forward -n observability svc/kube-prometheus-stack-prometheus 9090:9090
   # Open http://localhost:9090/targets and verify redpanda-connect endpoints are UP
   ```
3. Verify metrics are being scraped:
   ```bash
   # Query Prometheus for Redpanda Connect metrics
   curl 'http://localhost:9090/api/v1/query?query=up{job="redpanda-connect-streams"}'
   ```

### Pods Crashing
1. Check pod events:
   ```bash
   kubectl describe pod -n redpanda-connect <pod-name>
   ```
2. Check logs for errors:
   ```bash
   kubectl logs -n redpanda-connect <pod-name> --previous
   ```
3. Verify pipeline configuration syntax in ConfigMaps

## Cleanup

### Remove Standalone Deployment
```bash
kubectl delete -f standalone/argocd-rpcn-standalone.yaml
kubectl delete namespace redpanda-connect
```

### Remove Streams Deployment
```bash
kubectl delete -f streams/argocd-rpcn-streams.yaml
kubectl delete namespace redpanda-connect
```

### Remove Observability Stack
```bash
kubectl delete -f observability/argocd-observability.yaml
kubectl delete namespace observability
```

## References

- [Redpanda Connect Documentation](https://docs.redpanda.com/redpanda-connect/)
- [Redpanda Connect Helm Chart](https://github.com/redpanda-data/helm-charts)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

## License

See LICENSE file for details.
