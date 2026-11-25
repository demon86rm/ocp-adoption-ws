# 03 - Application alerts

Load one `AlertingRule` with three conditions, scoped to the application namespace, and confirm the rules are healthy. You do not wait for an alert to fire. Healthy means the rules are listed and evaluating, with no evaluation error.

Alertmanager stays on the cluster default. This step does not create an `AlertmanagerConfig` or a new receiver. Someone looks at **Observe** → **Alerting** in the console.

**Prerequisites**

- [02-user-workload-monitoring](../02-user-workload-monitoring/README.md) finished. `enableUserWorkload: true` is set, and `WORKSHOP_CONTEXT` is still exported.
- `cluster-admin` on that context.

## How these alerts are evaluated

Pod readiness comes from kube-state-metrics. Container memory comes from the kubelet cAdvisor series and from kube-state-metrics limits. Router 5xx responses come from the HAProxy router. Platform Prometheus scrapes all three. User-workload Prometheus scrapes `ServiceMonitor` and `PodMonitor` objects in user projects, so it does not store these series.

`AlertingRule` is the supported object for alerts on platform series. OpenShift copies it to a `PrometheusRule` in `openshift-monitoring` and platform Prometheus evaluates it. The namespace selector inside the expressions limits the alerts to `app-test`.

## 1. Confirm the three series exist

**What:** Run three queries in **Observe** → **Metrics** before you apply the rule.

**Why:** A rule can be healthy and still never fire if a label name on this cluster differs from the manifest. You fix the label first.

Use `app-test`, or your `APP_NAMESPACE` if you changed it in step 01.

Pod not Ready (any current value, including zero):

```promql
kube_pod_status_ready{condition="false", namespace="app-test"}
```

Memory working set and memory limit (both should return series for containers that set a limit):

```promql
container_memory_working_set_bytes{namespace="app-test", container!="", container!="POD"}
```

```promql
kube_pod_container_resource_limits{namespace="app-test", resource="memory"}
```

Router 5xx. Inspect the labels on the raw series before trusting `exported_namespace`:

```promql
haproxy_server_http_responses_total{code="5xx"}
```

Then the scoped rate:

```promql
sum by (exported_namespace, route) (
  rate(haproxy_server_http_responses_total{code="5xx", exported_namespace="app-test"}[5m])
)
```

Expected result: pod and memory queries return series when the namespace has pods. The router rate may be empty when the namespace has no routes or no 5xx traffic. An empty rate is a valid quiet period. The raw `haproxy_server_http_responses_total{code="5xx"}` query should still return series for the cluster's other routes.

If the raw router series has no `exported_namespace` label, find the label that carries the route's project (often still `exported_namespace`, sometimes another name). Update that label in [application-alerting-rule.yaml](application-alerting-rule.yaml) before you apply it. Leave the `namespace` label out of the selector when its value is `openshift-ingress`. That value is the router's project, not the application project.

### Verify

The pod query and the memory queries return data for `app-test`, or you have recorded that the namespace has no pods yet. The router metric name resolves (the query is not a syntax error). You have either confirmed `exported_namespace` or edited the manifest to the label this cluster uses.

## 2. Apply the AlertingRule

**What:** Apply [application-alerting-rule.yaml](application-alerting-rule.yaml).

**Why:** One object carries all three conditions: pod not Ready for 15 minutes, container memory above 90% of its limit for 10 minutes, and router HTTP 5xx above 0.1 responses per second (rate over 5 minutes) for 2 minutes.

On the test cluster the regex is `app-test`. Read the header comment in the file before you apply it anywhere else.

From this directory:

```bash
oc apply -f application-alerting-rule.yaml --context "${WORKSHOP_CONTEXT}"
```

Expected output:

```text
alertingrule.monitoring.openshift.io/application-go-live-alerts created
```

A second apply prints `configured` instead of `created`.

### Verify

```bash
oc get alertingrule application-go-live-alerts \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}"
```

Expected output: the name `application-go-live-alerts` is listed. Then confirm the generated platform rule exists:

```bash
oc get prometheusrule -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  -o name | grep application-go-live-alerts
```

Expected output: one `prometheusrule.monitoring.coreos.com/` line whose name contains `application-go-live-alerts`.

## 3. Confirm the rules are healthy

**What:** Open the three alert names in the console and check that none shows an evaluation error.

**Why:** Go-live is "the rules are healthy", which means they evaluate. It does not mean an alert is firing. Pod not Ready waits 15 minutes, memory waits 10 minutes, and a quiet route produces no 5xx series.

1. **Administrator** perspective.
2. **Observe** → **Alerting** → **Alerting rules**.
3. Filter or search for `ApplicationPodNotReady`, `ApplicationContainerMemoryHigh`, and `ApplicationRouterHTTP5xxSpike`.

Each rule is listed. The state is inactive (or pending or firing if the condition is already true). There is no evaluation-error marker on the rule.

Optional current values, in **Observe** → **Metrics**. These omit the `for` duration, so they show the expression now:

```promql
(
  kube_pod_status_ready{condition="false", namespace=~"app-test"} == 1
)
unless on (namespace, pod)
(
  kube_pod_status_phase{phase="Succeeded", namespace=~"app-test"} == 1
)
```

```promql
(
  sum by (namespace, pod, container) (
    container_memory_working_set_bytes{namespace=~"app-test", container!="", container!="POD"}
  )
  /
  sum by (namespace, pod, container) (
    kube_pod_container_resource_limits{namespace=~"app-test", resource="memory"}
  )
) > 0.9
```

```promql
sum by (exported_namespace, route) (
  rate(haproxy_server_http_responses_total{code="5xx", exported_namespace=~"app-test"}[5m])
) > 0.1
```

A graph with no series means the condition is false right now. That is a healthy quiet rule.

### Verify

All three names appear under **Alerting rules** and none reports an evaluation error.

## 4. Repeat on production

**What:** Copy this file to the production cluster after the test cluster checks pass.

**Why:** Production uses the same alerts. The only edit is the namespace selector.

1. Replace every `app-test` regex in [application-alerting-rule.yaml](application-alerting-rule.yaml) with `app-test|<production-namespace>`. If the production namespace is not chosen yet, the comment in the file uses `app-prod` as the placeholder. Write the real name before you apply it.
2. Point `WORKSHOP_CONTEXT` at the production cluster and repeat steps 02 and 03.
3. Leave Alertmanager routing as you found it.

### Verify

On production, `oc get alertingrule application-go-live-alerts -n openshift-monitoring` lists the rule, and **Observe** → **Alerting rules** shows the same three names with no evaluation error.

## Alternatives considered

| Approach | What it covers | Why it is not this lab |
| --- | --- | --- |
| `PrometheusRule` in the application namespace | User-project series scraped by user-workload monitoring | kube-state-metrics, cAdvisor, and router series are not in that Prometheus. The rule loads and the expressions return no data. |
| `AlertingRule` in `openshift-monitoring` | Platform series, with a namespace selector in the expression | This lab. OpenShift generates the platform `PrometheusRule`. |
| Custom Grafana | Extra dashboards on top of Thanos Querier | Deferred. The built-in **Namespace (Pods)** dashboard is the pod view for this round. |

## Reset

Delete the alert object only. This does not turn off user-workload monitoring and does not change Alertmanager.

```bash
oc delete alertingrule application-go-live-alerts \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  --ignore-not-found
```

The generated `PrometheusRule` is owned by the `AlertingRule` and is removed with it.

### Verify

```bash
oc get alertingrule application-go-live-alerts \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}"
```

Expected output: `Error from server (NotFound)`.

## Documentation

- [`AlertingRule` (OpenShift 4.18 monitoring APIs)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring_apis/alertingrule-monitoring-openshift-io-v1)
- [Configuring user workload monitoring (OpenShift 4.18)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring/configuring-user-workload-monitoring)
- [ROSA monitoring](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/monitoring/index)
