# PostgreSQL

PostgreSQL in this cluster is managed with CloudNativePG. The operator provides Kubernetes-native PostgreSQL resources, while this repository keeps a single shared PostgreSQL cluster for homelab services.

The current model is intentionally simple: one PostgreSQL cluster, separate databases per application, separate roles per application, and passwords managed through External Secrets.

## Source Paths

- CloudNativePG application: `applications/cloudnative-pg.yaml`
- CloudNativePG wrapper chart: `infrastructure/cloudnative-pg/`
- PostgreSQL cluster application: `applications/postgres-clusters.yaml`
- Shared PostgreSQL cluster: `infrastructure/postgres-clusters/homelab-postgres.yaml`
- Authentik database resources: `infrastructure/authentik/templates/authentik-database.yaml`, `infrastructure/authentik/templates/authentik-database-role.yaml`
- Grafana database resources: `infrastructure/prometheus-stack/templates/grafana-database.yaml`, `infrastructure/prometheus-stack/templates/grafana-database-role.yaml`
- Umami database resources: `infrastructure/umami/database.yaml`, `infrastructure/umami/database-role.yaml`

## CloudNativePG

CloudNativePG installs the PostgreSQL operator and CRDs used by the rest of the platform. It is deployed by Argo CD from:

```text
applications/cloudnative-pg.yaml
```

The application deploys the wrapper chart in:

```text
infrastructure/cloudnative-pg/
```

The wrapper chart depends on the upstream CloudNativePG chart:

```yaml
dependencies:
  - name: cloudnative-pg
    version: 0.29.0
    repository: https://cloudnative-pg.github.io/charts
```

It runs in the `cnpg-system` namespace with sync wave `-8`. This early sync wave matters because the CloudNativePG CRDs must exist before Argo CD can apply resources such as `Cluster`, `Database`, and `DatabaseRole`.

Current operator settings enable monitoring and Grafana dashboard creation:

```yaml
cloudnative-pg:
  monitoring:
    podMonitorEnabled: true
    grafanaDashboard:
      create: true
      namespace: monitoring
      labels:
        grafana_dashboard: "1"
      annotations:
        grafana_folder: Databases
```

The operator has small resource requests and limits because this homelab does not need a large database control plane.

## Why CloudNativePG Is Needed

CloudNativePG avoids managing PostgreSQL manually from inside a database shell. Instead of manually creating users, databases, credentials, and ownership each time an application needs PostgreSQL, those objects are declared in Git and reconciled by Kubernetes.

CloudNativePG provides these Kubernetes resources:

- `Cluster`: manages the PostgreSQL instance itself.
- `Database`: manages a database inside a CloudNativePG cluster.
- `DatabaseRole`: manages a PostgreSQL role/user inside a CloudNativePG cluster.

This is useful for GitOps because:

- Argo CD can reconcile PostgreSQL infrastructure from manifests.
- Database setup is reproducible after rebuilds or restores.
- Application database ownership is visible in Git.
- Passwords can come from External Secrets instead of being typed manually.
- New applications can follow the same database pattern without ad-hoc SQL commands.
- Deleting or changing application manifests does not require manual cleanup steps first.

The goal is not to make PostgreSQL complex. The goal is to make the small amount of PostgreSQL this homelab needs declarative and repeatable.

## Shared PostgreSQL Cluster

The actual PostgreSQL cluster is deployed separately from the operator by:

```text
applications/postgres-clusters.yaml
```

It applies:

```text
infrastructure/postgres-clusters/homelab-postgres.yaml
```

Current cluster:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: homelab-postgres
  namespace: databases
spec:
  description: Shared PostgreSQL cluster for homelab platform services
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:17.10
  storage:
    size: 20Gi
  monitoring:
    enablePodMonitor: true
```

Applications connect through the read-write service:

```text
homelab-postgres-rw.databases.svc.cluster.local
```

The cluster currently uses one PostgreSQL instance. This is enough for the expected homelab traffic and keeps operational overhead low.

## Why One Cluster Is Enough

This homelab does not expect high database traffic. Strong physical isolation between every application's database is also not a priority right now.

Creating one PostgreSQL cluster per application would add boilerplate without a clear benefit for the current workload size. Every additional PostgreSQL cluster would mean more:

- Pods.
- PVCs.
- Resource requests.
- Backup and restore surface area.
- Upgrade work.
- Monitoring noise.
- Operational decisions around sizing and maintenance.

The current isolation level is handled inside the shared cluster:

- Each application gets its own database.
- Each application gets its own PostgreSQL role/user.
- Each application gets its own password secret.
- Each database is owned by the matching application role.

For this environment, that is enough. It keeps the setup practical while still avoiding a single shared superuser or one shared application database.

Another PostgreSQL cluster should only be added when there is a concrete need, such as high traffic, different PostgreSQL versions, strict security isolation, separate backup policies, independent maintenance windows, or workload-specific resource tuning.

## Current Databases

Current application databases in the shared cluster:

| Database | Role | Consumer |
| --- | --- | --- |
| `authentik` | `authentik` | Authentik |
| `grafana` | `grafana` | Grafana from kube-prometheus-stack |
| `umami` | `umami` | Umami analytics |

All of them use `homelab-postgres` as the CloudNativePG cluster and connect through the same read-write service, but they do not share database ownership or credentials.

## Database And Role Pattern

Each application that needs PostgreSQL should declare three things:

- An `ExternalSecret` that creates the database password secret.
- A `DatabaseRole` that creates the application login role.
- A `Database` that creates the application database owned by that role.

The role pattern is:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: DatabaseRole
metadata:
  name: app-name
  namespace: databases
spec:
  cluster:
    name: homelab-postgres
  name: app-name
  login: true
  databaseRoleReclaimPolicy: retain
  passwordSecret:
    name: app-name-db-auth
```

The database pattern is:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: homelab-postgres-app-name
  namespace: databases
spec:
  cluster:
    name: homelab-postgres
  name: app-name
  owner: app-name
  databaseReclaimPolicy: retain
```

This keeps each application database small, explicit, and owned by its own role without creating an entire PostgreSQL cluster per application.

## Reclaim Policy

Application databases and roles use retain policies:

```yaml
databaseReclaimPolicy: retain
databaseRoleReclaimPolicy: retain
```

This is intentional. In a GitOps repository, accidental deletion or pruning of a manifest should not immediately drop a database or role that may contain application data.

The retain policy protects data first. If a database or role truly needs to be removed, that should be an explicit operational action.

## When To Add Another PostgreSQL Cluster

The default choice is to reuse `homelab-postgres`. Add another CloudNativePG cluster only when there is a clear reason.

Good reasons include:

- A workload needs significantly different CPU, memory, or storage sizing.
- A workload needs a different PostgreSQL major version.
- A workload needs strict security or failure-domain isolation.
- A workload needs independent backup and restore policies.
- A workload needs independent upgrade or maintenance windows.
- Shared-cluster traffic becomes a real bottleneck.

Until one of those needs exists, more PostgreSQL clusters would mostly be boilerplate.

## Validation

Check the CloudNativePG operator:

```bash
kubectl -n cnpg-system get pods
```

Check the PostgreSQL cluster:

```bash
kubectl -n databases get cluster
kubectl -n databases get pods
kubectl -n databases get svc homelab-postgres-rw
```

Check managed databases and roles:

```bash
kubectl -n databases get database
kubectl -n databases get databaserole
```

Check monitoring resources:

```bash
kubectl -n databases get podmonitor
kubectl -n monitoring get configmap -l grafana_dashboard=1
```
