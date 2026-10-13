# webapp Helm Chart

Production-grade Helm chart for deploying a containerized web application on Kubernetes.

## Features

- **Deployment** with configurable replicas, container image, and environment
- **Service** (ClusterIP / NodePort / LoadBalancer)
- **Ingress** support (optional) with TLS and custom annotations (e.g. NGINX, cert-manager)
- **Horizontal Pod Autoscaler** (optional) on CPU and/or memory utilization
- **PodDisruptionBudget** (optional) to protect availability during voluntary disruptions
- **ConfigMap** for non-sensitive application configuration, injected as environment variables
- **Liveness and readiness probes** for reliable traffic routing and self-healing
- **Resource requests and limits** on every container
- **Hardened security context**: non-root user, read-only root filesystem, dropped capabilities
- Scheduling controls: `nodeSelector`, `tolerations`, and `affinity` (with commented examples)

## Prerequisites

- Helm 3.x
- `kubectl` configured against a Kubernetes cluster (1.30+ recommended)
- An ingress controller (e.g. ingress-nginx) if you enable `ingress`

## Installing

```bash
# From the chart directory
helm install my-release .

# With custom values
helm install my-release . -f my-values.yaml

# Into a specific namespace (creates it if missing)
helm install my-release . --namespace webapp --create-namespace
```

## Upgrading

```bash
helm upgrade my-release . -f my-values.yaml
```

## Uninstalling

```bash
helm uninstall my-release
```

This removes all Kubernetes resources associated with the release.

## Configuration

The following table lists the most commonly customized values. See `values.yaml` for the full list.

| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Number of pod replicas (ignored when HPA is enabled) | `2` |
| `image.repository` | Container image repository | `ghcr.io/example-org/webapp` |
| `image.tag` | Image tag (defaults to chart `appVersion`) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | Ingress class (e.g. `nginx`) | `""` |
| `ingress.hosts` | Hostname/path routing rules | `chart-example.local` |
| `resources.requests` / `resources.limits` | CPU/memory requests and limits | `100m/128Mi` → `500m/512Mi` |
| `autoscaling.enabled` | Enable Horizontal Pod Autoscaler | `false` |
| `autoscaling.minReplicas` / `maxReplicas` | HPA replica bounds | `2` / `6` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU for scaling | `70` |
| `podDisruptionBudget.enabled` | Enable PodDisruptionBudget | `false` |
| `podDisruptionBudget.minAvailable` | Minimum pods available during disruptions | `1` |
| `configMap.data` | Key/value config injected as env vars | `APP_ENV`, `LOG_LEVEL`, … |
| `env` | Extra container environment variables | `PORT=8080` |
| `livenessProbe` / `readinessProbe` | HTTP health check definitions | `/healthz`, `/ready` |
| `nodeSelector` / `tolerations` / `affinity` | Pod scheduling constraints | `{}` / `[]` / `{}` |

### Example: custom values file

```yaml
# my-values.yaml
replicaCount: 3

image:
  repository: ghcr.io/example-org/webapp
  tag: "1.2.0"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-example-tls
      hosts:
        - app.example.com

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 65

podDisruptionBudget:
  enabled: true
  minAvailable: 2

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

```bash
helm install my-release . -f my-values.yaml
```

## Production hardening included

This chart ships with production-safe defaults and opt-in guards:

- **Probes**: HTTP liveness (`/healthz`) and readiness (`/ready`) probes with sensible delays so unhealthy pods are restarted and unready pods never receive traffic.
- **PodDisruptionBudget**: optional `minAvailable` guard keeps capacity during node drains and cluster upgrades.
- **HPA**: optional CPU/memory-based autoscaling; `replicaCount` is automatically ignored when the HPA manages replicas.
- **Resource limits**: requests and limits are set on the container, preventing noisy-neighbor issues and enabling the scheduler to bin-pack correctly.
- **Non-root execution**: the container runs as UID 1000 with `runAsNonRoot`, a read-only root filesystem, `allowPrivilegeEscalation: false`, and all Linux capabilities dropped.
- **Rolling updates**: the default `RollingUpdate` strategy (with readiness gates) keeps the app serving traffic during deploys.

## CI

Pull requests run `helm lint` and `helm template` (default values plus a full-feature render) via GitHub Actions - see `.github/workflows/lint.yml`.
