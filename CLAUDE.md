# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes Helm chart (`load-chart`) that demonstrates ArgoCD sync waves with database migrations. The chart deploys PostgreSQL, runs database migrations, and then verifies the migrations with a test job. Each phase uses sync waves to ensure proper ordering. The chart follows Helm v2 API conventions.

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

- **Migrations Job**: Runs SQL migrations to create database schema and sample data
- **Test Job**: Verifies migrations were applied correctly
- **Cleanup Job** (optional): Deletes the ArgoCD Application after successful tests
- **Notification Job** (optional): Sends Slack notifications about test results
- **ServiceAccount**: Dedicated pod identity for jobs
- **Role/RoleBinding**: Permissions to get job status (when cleanup or notification enabled)
- **ClusterRole/ClusterRoleBinding**: Permissions to delete ArgoCD applications (when cleanup enabled)

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

- **migrations.image**: PostgreSQL client image for migrations (default: `postgres:16-alpine`)
- **migrations.postgres**: PostgreSQL connection details for migrations
- **migrations.backoffLimit**: Number of retry attempts before marking job as failed (default: 3)
- **migrations.ttlSecondsAfterFinished**: Time to keep completed jobs before cleanup (default: 300 seconds)
- **job.image**: PostgreSQL client image for tests (default: `postgres:16-alpine`)
- **job.postgres**: PostgreSQL connection details for tests
- **job.backoffLimit**: Number of retry attempts before marking job as failed (default: 3)
- **job.ttlSecondsAfterFinished**: Time to keep completed jobs before cleanup (default: 300 seconds)
- **resources**: CPU/memory limits and requests for job containers
- **securityContext**: Pod and container security settings
- **postgresql.enabled**: Enable/disable PostgreSQL dependency (default: true)
- **rabbitmq.enabled**: Enable/disable RabbitMQ dependency (default: true)

### Job Behavior

#### Migrations Job ([templates/migrations-job.yaml](templates/migrations-job.yaml))
1. Waits for PostgreSQL to be ready using `pg_isready` (60 second timeout)
2. Creates database schema:
   - `users` table (id, username, email, created_at)
   - `products` table (id, name, price, stock, created_at)
   - `orders` table (id, user_id, product_id, quantity, total_price, created_at) with foreign keys
3. Inserts sample data (2 users, 2 products)
4. Exits with code 0 on success, 1 on failure
5. Automatically cleaned up 5 minutes after completion

#### Test Job ([templates/job.yaml](templates/job.yaml))
1. Waits for PostgreSQL to be ready using `pg_isready` (30 second timeout)
2. Runs 6 verification tests:
   - Basic connectivity (SELECT 1)
   - Verifies `users` table exists
   - Verifies `products` table exists
   - Verifies `orders` table exists
   - Verifies sample data (at least 2 users and 2 products)
   - Verifies foreign key constraints (2 foreign keys on orders table)
3. Exits with code 0 if all tests pass, 1 on any failure
4. Automatically cleaned up 5 minutes after completion

#### Cleanup Job ([templates/cleanup-job.yaml](templates/cleanup-job.yaml)) - Optional
1. Waits for test job to complete successfully (300 second timeout by default)
2. Checks test job status using `kubectl get job`
3. If tests passed, deletes the ArgoCD Application using `kubectl delete application`
4. If tests failed, exits without deleting the application
5. Automatically cleaned up 1 minute after completion

**Note**: The cleanup job is disabled by default. Set `cleanup.enabled: true` to enable it.

#### Notification Job ([templates/notification-job.yaml](templates/notification-job.yaml)) - Optional
1. Waits for test job to complete (success or failure, 300 second timeout by default)
2. Checks test job status using `kubectl get job`
3. Sends Slack notification with test results:
   - ✅ Success notification if tests passed (includes verified items)
   - ❌ Failure notification if tests failed (includes error details)
   - ❌ Timeout notification if test job didn't complete within timeout
4. Uses Slack webhook to send formatted messages
5. Automatically cleaned up 1 minute after completion

**Note**: The notification job is disabled by default. Set `notification.enabled: true` and provide `notification.slackWebhookUrl` to enable it.

The migrations and test jobs use the official PostgreSQL Alpine image. The cleanup job uses the Bitnami kubectl image. The notification job uses the curl image.

## Important Files

- **Chart.yaml**: Chart metadata and dependency declarations
- **values.yaml**: Default configuration values with extensive inline documentation
- **Chart.lock**: Locked dependency versions (regenerate with `helm dependency update`)
- **.helmignore**: Files excluded from chart packaging

## Testing

The chart includes a Helm test in `templates/tests/test-connection.yaml` for basic PostgreSQL connectivity testing using `pg_isready`.

## ArgoCD Integration

The chart is configured with ArgoCD sync waves to demonstrate a complete migration workflow with proper ordering.

### Sync Wave Strategy

- **Wave 0**: ServiceAccount, RBAC (Role/RoleBinding/ClusterRole/ClusterRoleBinding), PostgreSQL, and RabbitMQ (if enabled) deploy first and become healthy
- **Wave 1**: Migrations Job runs and creates database schema + sample data
- **Wave 2**: Test Job verifies migrations were applied correctly
- **Wave 3**: Cleanup Job (if enabled) deletes the ArgoCD Application and Notification Job (if enabled) sends Slack notifications after tests complete

This ensures:
1. Database and permissions are fully ready before migrations run
2. Migrations complete successfully before tests run
3. Tests validate the entire migration workflow
4. Cleanup and notifications happen after tests complete (cleanup only if tests pass, notifications for both success and failure)

### Configuration

Sync waves are configured via annotations in the templates:

- **ServiceAccount** ([templates/serviceaccount.yaml:9](templates/serviceaccount.yaml#L9)): `sync-wave: "0"`
- **RBAC Resources** ([templates/role.yaml](templates/role.yaml), [templates/rolebinding.yaml](templates/rolebinding.yaml)): `sync-wave: "0"`
- **Migrations Job** ([templates/migrations-job.yaml:9](templates/migrations-job.yaml#L9)): `sync-wave: "1"` (customizable via `argocd.migrations.syncWave` in values.yaml)
- **Test Job** ([templates/job.yaml:9](templates/job.yaml#L9)): `sync-wave: "2"` (customizable via `argocd.job.syncWave` in values.yaml)
- **Cleanup Job** ([templates/cleanup-job.yaml:9](templates/cleanup-job.yaml#L9)): `sync-wave: "3"` (customizable via `argocd.cleanup.syncWave` in values.yaml, only created if `cleanup.enabled: true`)
- **Notification Job** ([templates/notification-job.yaml:9](templates/notification-job.yaml#L9)): `sync-wave: "3"` (customizable via `argocd.notification.syncWave` in values.yaml, only created if `notification.enabled: true`)

Dependencies (PostgreSQL and RabbitMQ) use `commonAnnotations` in values.yaml to set `sync-wave: "0"` on all their resources.

### Customizing Sync Waves

To customize sync waves in your values file:

```yaml
argocd:
  migrations:
    syncWave: 1  # Run migrations in wave 1 (default)
  job:
    syncWave: 2  # Run tests in wave 2 (default)
  cleanup:
    syncWave: 3  # Run cleanup in wave 3 (default)
  notification:
    syncWave: 3  # Run notifications in wave 3 (default)

cleanup:
  enabled: true  # Enable automatic cleanup after successful tests
  applicationName: "load-chart"  # Name of ArgoCD Application to delete
  applicationNamespace: "argocd"  # Namespace where ArgoCD runs

notification:
  enabled: true  # Enable Slack notifications
  slackWebhookUrl: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"  # Your Slack webhook URL
```

ArgoCD will wait for each wave to become healthy before proceeding to the next wave. This ensures:
- PostgreSQL is fully ready (StatefulSet healthy, pods running, health checks passing) before migrations execute
- Migrations complete successfully (Job status: Completed) before tests run
- Tests validate the complete migration workflow
- Cleanup only runs after tests pass, and deletes the ArgoCD Application which triggers removal of all resources
- Notifications are sent to Slack with test results (both success and failure scenarios)

### Example ArgoCD Application

The repository includes [argocd-application.yaml](argocd-application.yaml) - an ArgoCD Application manifest configured to deploy this chart from `https://github.com/boskiv/test-argo-helm-chart.git` with PostgreSQL and RabbitMQ dependencies disabled by default.

To deploy:
```bash
kubectl apply -f argocd-application.yaml
```
