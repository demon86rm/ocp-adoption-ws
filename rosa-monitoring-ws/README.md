# OpenShift monitoring on ROSA

Prometheus monitoring for application namespaces on Red Hat OpenShift Service on AWS (ROSA): user-workload monitoring enabled, the built-in pod dashboard in use, and one platform `AlertingRule` with three conditions. The same files are what you apply on the test cluster and, after editing the namespace selector, on production.

This workshop takes about 45 minutes. It is written so you can run it again on your own cluster without a live demonstration.

## Environment

- ROSA cluster, OpenShift 4.16 or newer. `AlertingRule` (`monitoring.openshift.io/v1`) is the supported API for alerts on platform metrics from this version.
- `cluster-admin` on that cluster. Enabling user-workload monitoring and creating an `AlertingRule` both require it.
- OpenShift CLI (`oc`) logged in to the cluster you are changing.
- A browser session on the same cluster's web console.

Platform Prometheus already stores pod status, container memory, and router metrics. This workshop does not change retention, persistent storage, or node placement for that stack. On ROSA those settings stay with the managed monitoring configuration.

## What you will have when you finish

- `enableUserWorkload: true` in `cluster-monitoring-config` (`openshift-monitoring`).
- The **Kubernetes / Compute Resources / Namespace (Pods)** dashboard filtered to `app-test`.
- `AlertingRule` `application-go-live-alerts` loaded, with three rules evaluating and no evaluation error.
- A named person who checks **Observe** → **Alerting** through go-live. Alert routing stays on the cluster default.

## Repo map

| Path | What it is |
| --- | --- |
| [01-setup/](01-setup/README.md) | Context, permission check, namespace variables (about 10 minutes) |
| [02-user-workload-monitoring/](02-user-workload-monitoring/README.md) | Enable user-workload monitoring and open the pod dashboard (about 15 minutes) |
| [03-application-alerts/](03-application-alerts/README.md) | Apply the three alerts and confirm they are healthy (about 15 minutes) |
| [REPEAT.md](REPEAT.md) | One-page sheet for the test cluster and for production (about 5 minutes) |

## Quick start

Start at [01-setup](01-setup/README.md). Do the folders in order. Keep [REPEAT.md](REPEAT.md) for the production copy.

## Left out of this round

Custom Grafana dashboards, extra Alertmanager receivers, log forwarding, and OpenTelemetry. Alerts are reviewed in the OpenShift console.
