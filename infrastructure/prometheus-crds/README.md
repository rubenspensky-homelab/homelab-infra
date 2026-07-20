# Prometheus Operator CRDs

This wrapper installs Prometheus Operator CRDs before CloudNativePG and other
components create `PodMonitor`, `ServiceMonitor`, or related resources.

Chart `prometheus-operator-crds` `30.0.0` provides Prometheus Operator `v0.92.0`,
matching the operator version bundled with `kube-prometheus-stack` `87.3.0`.

The main Prometheus stack must keep its bundled CRD installation disabled to
ensure that this application is the single owner of these CRDs.

The CRDs use Argo CD's `Prune=false` sync option. CRD removal also deletes every
custom resource of that kind, so uninstalling them must always be an explicit,
manual operation.
