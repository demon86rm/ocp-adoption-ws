# Repeat sheet

Use this page on the test cluster and again on production. Details live in the step folders. This page is only the clicks, the health check, and the production edit.

**Alert owner:** ____________________

That person opens **Observe** → **Alerting** and reviews firing alerts through go-live. Alertmanager routing stays on the cluster default. This round does not add email, chat, or paging.

## Where to click

Administrator perspective.

| You want | Path |
| --- | --- |
| Pod health for `app-test` | **Observe** → **Dashboards** → **Kubernetes / Compute Resources / Namespace (Pods)** → namespace `app-test` |
| The three rules | **Observe** → **Alerting** → **Alerting rules** |
| A query | **Observe** → **Metrics** |

Dashboard label on this cluster, if it differs from the name above: ____________________

## Healthy rule

`application-go-live-alerts` is listed:

```bash
oc get alertingrule application-go-live-alerts -n openshift-monitoring
```

In **Alerting rules**, these three names are present and show no evaluation error:

- `ApplicationPodNotReady` (15 minutes)
- `ApplicationContainerMemoryHigh` (10 minutes, above 90% of the memory limit)
- `ApplicationRouterHTTP5xxSpike` (5xx rate over 5 minutes, above 0.1 per second, for 2 minutes)

Healthy means the rule is listed and evaluating. A rule with no firing alert is still healthy. Do not wait 15 minutes in the workshop for a pod alert to fire.

## If the rules are missing

Check these in order.

1. **Alerting rules** has none of the three names. The `AlertingRule` was not applied, or this login is not `cluster-admin`. From `03-application-alerts/`: `oc apply -f application-alerting-rule.yaml`. `oc auth can-i create alertingrule.monitoring.openshift.io -n openshift-monitoring` must print `yes`.
2. A `PrometheusRule` in the application namespace has an empty rule group, or **Observe** → **Targets** shows no user-workload targets. User-workload monitoring is off. `cluster-monitoring-config` in `openshift-monitoring` needs `enableUserWorkload: true`, merged in place so other keys stay. See `02-user-workload-monitoring/`. The three alerts in this workshop do not use that stack. An empty user-project rule group is still the sign that user-workload monitoring is off.

## Production

Edit the namespace regex in `03-application-alerts/application-alerting-rule.yaml`, then apply the same file on the production cluster. Test value:

```text
namespace=~"app-test"
exported_namespace=~"app-test"
```

Production value, after the real namespace is known (`app-prod` is only a placeholder):

```text
namespace=~"app-test|<production-namespace>"
exported_namespace=~"app-test|<production-namespace>"
```

Repeat step 02 on production so `enableUserWorkload: true` is set there too. Keep the existing ConfigMap keys. Match the router label (`exported_namespace`, or whatever step 03 discovered) and do not select `namespace="openshift-ingress"`.

## Tune one number

`0.1` on `ApplicationRouterHTTP5xxSpike` is about 30 HTTP 5xx responses in five minutes. Change that number in the manifest if it is too sensitive for this traffic. The pod and memory windows stay at 15 minutes and 10 minutes.
