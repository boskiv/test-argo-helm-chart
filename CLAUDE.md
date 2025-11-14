# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes Helm chart (`load-chart`) for deploying applications with PostgreSQL and RabbitMQ backend services. The chart follows Helm v2 API conventions and supports modern Kubernetes features including Gateway API.

## Common Commands

### Development Workflow

```bash
# Lint the chart for errors
helm lint .

# Update dependencies (PostgreSQL 18.1.8, RabbitMQ 16.0.14)
helm dependency update .

# Render templates locally (dry-run to see generated YAML)
helm template my-release .

# Render with custom values
helm template my-release . -f custom-values.yaml

# Package the chart
helm package .
```

### Installation and Testing

```bash
# Install the chart
helm install my-release .

# Install with custom values
helm install my-release . -f custom-values.yaml

# Run connection test (tests/test-connection.yaml)
helm test my-release

# Upgrade an existing release
helm upgrade my-release .

# Uninstall
helm uninstall my-release
```

## Architecture

### Key Resources

The chart deploys these Kubernetes resources:

- **Deployment**: Main workload with configurable replicas
- **Service**: ClusterIP service (default port 80)
- **ServiceAccount**: Dedicated pod identity
- **Ingress** (conditional): Traditional ingress controller routing
- **HTTPRoute** (conditional): Gateway API routing (mutually exclusive with Ingress)
- **HorizontalPodAutoscaler** (conditional): CPU/memory-based autoscaling

### Template Structure

- **templates/_helpers.tpl**: Contains reusable template functions for names, labels, and selectors. All resource names and labels should use these helpers for consistency.
- **templates/*.yaml**: Each file represents a Kubernetes resource with conditional rendering based on `values.yaml`
- **templates/tests/**: Helm test pods using the `helm.sh/hook: test` annotation

### Dependencies

Declared in `Chart.yaml` with locked versions in `Chart.lock`:

- **postgresql** (Bitnami 18.1.8): Database service, configurable via `values.yaml` under `postgresql.*`
- **rabbitmq** (Bitnami 16.0.14): Message broker using legacy image, configurable via `values.yaml` under `rabbitmq.*`

Both dependencies can be disabled by setting `postgresql.enabled: false` or `rabbitmq.enabled: false` in values.

### Configuration Pattern

All configuration is driven by `values.yaml` (189 lines). Key sections:

- **image**: Container image, tag, pull policy, and secrets
- **replicaCount**: Manual replica setting (ignored if HPA enabled)
- **autoscaling**: HPA configuration
- **service**: Service type and port configuration
- **ingress/httproute**: Routing configuration (use one or the other, not both)
- **resources**: CPU/memory limits and requests
- **securityContext**: Pod and container security settings
- **livenessProbe/readinessProbe**: Health check configuration

When modifying templates, always check if the feature should be conditional based on values (use `{{- if .Values.featureName.enabled }}`).

### Ingress vs HTTPRoute

The chart supports two routing methods (mutually exclusive):

- **Ingress** (`ingress.enabled: true`): Traditional Kubernetes ingress
- **HTTPRoute** (`httproute.enabled: true`): Gateway API for advanced routing

Enable only one at a time. HTTPRoute requires Gateway API CRDs installed in the cluster.

## Important Files

- **Chart.yaml**: Chart metadata and dependency declarations
- **values.yaml**: Default configuration values with extensive inline documentation
- **Chart.lock**: Locked dependency versions (regenerate with `helm dependency update`)
- **.helmignore**: Files excluded from chart packaging

## Testing

The chart includes a Helm test in `templates/tests/test-connection.yaml` that validates service connectivity using wget. The test runs as a post-install hook and can be executed with `helm test <release-name>`.

## ArgoCD Integration

The chart is configured with ArgoCD sync waves to control deployment order and ensure dependencies are healthy before the main application starts.

### Sync Wave Strategy

- **Wave 0**: ServiceAccount, PostgreSQL, and RabbitMQ deploy first
- **Wave 1**: Main Deployment and Service deploy after dependencies are healthy
- **Wave 2**: Ingress, HTTPRoute, and HPA deploy last (if enabled)

### Configuration

Sync waves are configured via annotations in the templates:

- **ServiceAccount** ([templates/serviceaccount.yaml:9](templates/serviceaccount.yaml#L9)): `sync-wave: "0"`
- **Deployment** ([templates/deployment.yaml:8](templates/deployment.yaml#L8)): `sync-wave: "1"` (customizable via `argocd.deployment.syncWave` in values.yaml)
- **Service** ([templates/service.yaml:8](templates/service.yaml#L8)): `sync-wave: "1"`
- **Ingress** ([templates/ingress.yaml:9](templates/ingress.yaml#L9)): `sync-wave: "2"`
- **HTTPRoute** ([templates/httproute.yaml:11](templates/httproute.yaml#L11)): `sync-wave: "2"`
- **HPA** ([templates/hpa.yaml:9](templates/hpa.yaml#L9)): `sync-wave: "2"`

Dependencies (PostgreSQL and RabbitMQ) use `commonAnnotations` in [values.yaml](values.yaml#L177-L178) to set `sync-wave: "0"` on all their resources.

### Customizing Sync Waves

To customize the deployment sync wave in your values file:

```yaml
argocd:
  deployment:
    syncWave: 2  # Deploy application in wave 2 instead of wave 1
```

ArgoCD will wait for each wave to become healthy (pods running, health checks passing) before proceeding to the next wave. This ensures PostgreSQL and RabbitMQ are fully ready before the application deployment starts.
