# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes Helm chart (`load-chart`) that deploys a PostgreSQL connectivity test Job. The chart demonstrates ArgoCD sync waves by ensuring PostgreSQL (and optionally RabbitMQ) dependencies are healthy before running the test job. The chart follows Helm v2 API conventions.

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

- **Job**: PostgreSQL connectivity test that executes `SELECT 1` and exits
- **ServiceAccount**: Dedicated pod identity for the job

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

All configuration is driven by `values.yaml`. Key sections:

- **job.image**: PostgreSQL client image (default: `postgres:16-alpine`)
- **job.postgres**: PostgreSQL connection details (host, port, database, user, password)
- **job.backoffLimit**: Number of retry attempts before marking job as failed (default: 3)
- **job.ttlSecondsAfterFinished**: Time to keep completed jobs before cleanup (default: 300 seconds)
- **resources**: CPU/memory limits and requests for the job container
- **securityContext**: Pod and container security settings
- **postgresql.enabled**: Enable/disable PostgreSQL dependency (default: true)
- **rabbitmq.enabled**: Enable/disable RabbitMQ dependency (default: true)

### Job Behavior

The Job ([templates/job.yaml](templates/job.yaml)):
1. Waits for PostgreSQL to be ready using `pg_isready` (30 second timeout)
2. Executes `SELECT 1` against the PostgreSQL database
3. Exits with code 0 on success, 1 on failure
4. Automatically cleaned up 5 minutes after completion (configurable via `job.ttlSecondsAfterFinished`)

The job uses the official PostgreSQL Alpine image which includes `psql` and `pg_isready` utilities.

## Important Files

- **Chart.yaml**: Chart metadata and dependency declarations
- **values.yaml**: Default configuration values with extensive inline documentation
- **Chart.lock**: Locked dependency versions (regenerate with `helm dependency update`)
- **.helmignore**: Files excluded from chart packaging

## Testing

The chart includes a Helm test in `templates/tests/test-connection.yaml` for basic connectivity testing. The main PostgreSQL connectivity test is performed by the Job resource itself.

## ArgoCD Integration

The chart is configured with ArgoCD sync waves to control deployment order and ensure dependencies are healthy before the main application starts.

### Sync Wave Strategy

- **Wave 0**: ServiceAccount, PostgreSQL, and RabbitMQ (if enabled) deploy first
- **Wave 1**: PostgreSQL test Job runs after dependencies are healthy

### Configuration

Sync waves are configured via annotations in the templates:

- **ServiceAccount** ([templates/serviceaccount.yaml:9](templates/serviceaccount.yaml#L9)): `sync-wave: "0"`
- **Job** ([templates/job.yaml:6](templates/job.yaml#L6)): `sync-wave: "1"` (customizable via `argocd.job.syncWave` in values.yaml)

Dependencies (PostgreSQL and RabbitMQ) use `commonAnnotations` in [values.yaml](values.yaml#L189-L190) to set `sync-wave: "0"` on all their resources.

### Customizing Sync Waves

To customize the job sync wave in your values file:

```yaml
argocd:
  job:
    syncWave: 2  # Run job in wave 2 instead of wave 1
```

ArgoCD will wait for each wave to become healthy before proceeding to the next wave. This ensures PostgreSQL is fully ready (StatefulSet healthy, pods running, health checks passing) before the test job executes.

### Example ArgoCD Application

The repository includes [argocd-application.yaml](argocd-application.yaml) - an ArgoCD Application manifest configured to deploy this chart from `https://github.com/boskiv/test-argo-helm-chart.git` with PostgreSQL and RabbitMQ dependencies disabled by default.

To deploy:
```bash
kubectl apply -f argocd-application.yaml
```
