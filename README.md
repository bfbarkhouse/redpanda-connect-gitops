# Redpanda Connect Kubernetes Deployment

GitOps configurations for deploying Redpanda Connect on Kubernetes using ArgoCD.

## Repository Structure

```
.
├── standalone/                              # Standalone deployment mode
│   ├── argocd-rpcn-standalone.yaml         # ArgoCD Application manifest
│   └── standalone-mode.yaml                # Helm values for standalone mode
└── streams/                                 # Streams deployment mode
    ├── argocd-rpcn-streams.yaml            # ArgoCD Application manifest
    ├── kustomization.yaml                  # Kustomize configuration
    ├── streams-mode.yaml                   # Helm values for streams mode
    └── config/                             # Pipeline configurations
        ├── meow.yaml                       # Sample pipeline 1
        └── woof.yaml                       # Sample pipeline 2
```

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- kubectl configured with cluster access
- ArgoCD CLI (optional)

## Deployment Modes

This repository supports two deployment modes for Redpanda Connect:

### Standalone Mode

Deploys Redpanda Connect with a single embedded pipeline configuration using Helm.

**Features:**
- 1 replica
- Generates fake user data every second
- Processes names to uppercase
- Outputs to stdout

**Deploy:**
```bash
kubectl apply -f standalone/argocd-rpcn-standalone.yaml
```

**Configuration:** `standalone/standalone-mode.yaml`

### Streams Mode

Deploys Redpanda Connect with multiple pipeline configurations managed via ConfigMaps.

**Features:**
- Kustomize-based configuration management
- Multiple independent pipeline configurations
- Automatic ConfigMap hash updates for rolling deployments
- Separate pipeline files for easier management
- 3 replicas
- Prometheus metrics enabled

**Deploy:**
```bash
kubectl apply -f streams/argocd-rpcn-streams.yaml
```

**Configuration:** `streams/streams-mode.yaml`

## Pipeline Configurations

### meow.yaml
Simple pipeline that generates "meow" messages every 2 seconds and outputs to stdout.

```yaml
input:
  generate:
    mapping: root = "meow"
    interval: 2s
    count: 0
output:
  stdout:
    codec: lines
```

### woof.yaml
Additional pipeline for testing multiple concurrent streams (configuration similar to meow.yaml).

## ArgoCD Integration

Both deployment modes use ArgoCD for continuous delivery with the following settings:

- **Source Repository:** https://github.com/bfbarkhouse/redpanda-connect-gitops
- **Target Branch:** main
- **Target Namespace:** redpanda-connect (auto-created)
- **Auto-sync:** Enabled with prune and self-heal
- **Sync Options:** CreateNamespace=true

### Standalone Mode Details

Uses ArgoCD's multi-source feature:
1. Helm chart from https://charts.redpanda.com (version 3.1.0)
2. Values files from the GitOps repository

### Streams Mode Details

Uses Kustomize to:
1. Generate ConfigMaps from pipeline files with hash suffixes
2. Deploy Helm chart with custom values
3. Automatically trigger rolling updates when pipelines change

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

# Sync manually if needed
argocd app sync redpanda-connect-standalone
argocd app sync redpanda-connect-streams
```

## Customization

### Adding New Pipelines (Streams Mode)

1. Create a new YAML file in `streams/config/` with your pipeline configuration:
   ```yaml
   input:
     # your input configuration
   output:
     # your output configuration
   ```

2. Update `streams/kustomization.yaml` to include the new file:
   ```yaml
   configMapGenerator:
     - name: connect-streams
       files:
         - config/woof.yaml
         - config/meow.yaml
         - config/your-new-pipeline.yaml
   ```

3. Commit and push changes
4. ArgoCD will automatically sync and deploy with the new pipeline

### Modifying Helm Values

Edit the respective values files:
- `standalone/standalone-mode.yaml` for standalone mode
- `streams/streams-mode.yaml` for streams mode

Common modifications:
```yaml
deployment:
  replicaCount: 3              # Number of replicas
logger:
  level: INFO                  # Log level (INFO, DEBUG, etc.)
  static_fields:
    '@service': my-service     # Static log fields
metrics:
  prometheus: {}               # Prometheus metrics configuration
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

### View Logs
```bash
kubectl logs -n redpanda-connect -l app.kubernetes.io/name=connect -f
```

### Check Metrics
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
   ```
2. Verify repository access and credentials
3. Check for syntax errors in YAML files
4. Review ArgoCD logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
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

## References

- [Redpanda Connect Documentation](https://docs.redpanda.com/redpanda-connect/)
- [Redpanda Connect Helm Chart](https://github.com/redpanda-data/helm-charts)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)

## License

See LICENSE file for details.
