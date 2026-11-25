# 02 - User-workload monitoring

Turn on monitoring for user-defined projects, then open the built-in pod dashboard for `app-test`.

The three alerts in step 03 run on platform Prometheus. User-workload monitoring is still required: it is how the project shows up for console users, and the go-live check is that `enableUserWorkload` is true on the test cluster and on production.

**Prerequisites**

- [01-setup](../01-setup/README.md) finished in this shell (`WORKSHOP_CONTEXT` and `APP_NAMESPACE` are set).
- `cluster-admin` on that context.

Allow about 2 minutes after the ConfigMap save for the user-workload pods to become Ready.

## 1. See whether user-workload monitoring is already on

**What:** Read `cluster-monitoring-config` before changing it.

**Why:** ROSA clusters often already have this ConfigMap. You add one key. You keep every other key.

```bash
oc get configmap cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  -o jsonpath='{.data.config\.yaml}{"\n"}'
```

Expected output is one of:

- A YAML document that already contains `enableUserWorkload: true`. Go to section 3.
- A YAML document without that key. Continue at section 2 and edit in place.
- `Error from server (NotFound)`. Continue at section 2 and create the ConfigMap from the snippet.

Save a local copy before you change anything. Leave that file on your workstation. Do not commit it.

```bash
oc get configmap cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  -o yaml > "${HOME}/cluster-monitoring-config.backup.yaml"
```

If the ConfigMap was missing, that command fails with NotFound. That is the create path in section 2. There is nothing to back up.

### Verify

```bash
test -f "${HOME}/cluster-monitoring-config.backup.yaml" && echo "backup present" || echo "no configmap to back up"
```

Expected output: `backup present`, or `no configmap to back up` only when section 1 returned NotFound.

## 2. Set enableUserWorkload

**What:** Make `data.config.yaml` include `enableUserWorkload: true`, and leave every existing key in place.

**Why:** The Cluster Monitoring Operator starts the user-workload stack from that key. Replacing the whole ConfigMap with only this key removes ROSA monitoring settings and can degrade the `monitoring` cluster operator.

[enable-user-workload.yaml](enable-user-workload.yaml) shows the key. It is a snippet.

**ConfigMap already exists.** Edit the live object. Under `data.config.yaml`, add this line at the same indentation as the other top-level keys. Do not remove lines that are already there.

```bash
oc edit configmap cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}"
```

```yaml
enableUserWorkload: true
```

Save and exit. Example shape after the edit (your other keys will differ; keep them):

```yaml
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      retention: 15d
```

**ConfigMap does not exist.** Creating it from the snippet is safe, because there are no other keys to keep.

```bash
oc apply -f enable-user-workload.yaml --context "${WORKSHOP_CONTEXT}"
```

Run that `oc apply` only on the NotFound path.

### Verify

```bash
oc get configmap cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  -o jsonpath='{.data.config\.yaml}{"\n"}'
```

Expected output: the document includes `enableUserWorkload: true`. If you edited an existing ConfigMap, the keys from your backup are still present.

## 3. Wait for the user-workload stack

**What:** Wait until the user-workload pods are Running and the monitoring operator is Available.

**Why:** The console and any later user-project rules read this stack. A ConfigMap save redeploys it.

```bash
oc get pods -n openshift-user-workload-monitoring --context "${WORKSHOP_CONTEXT}"
oc get co monitoring --context "${WORKSHOP_CONTEXT}"
```

Expected output: `prometheus-user-workload-0` and `thanos-ruler-user-workload-0` are `1/1 Running`. `monitoring` is `Available=True`, `Progressing=False`, `Degraded=False`. A user-workload Alertmanager pod is optional and often absent. You do not need it for this workshop.

If pods are still `ContainerCreating` or `Init`, wait and run the same commands again. Two minutes is typical.

### Verify

```bash
oc get pods -n openshift-user-workload-monitoring \
  -l app.kubernetes.io/name=prometheus \
  --context "${WORKSHOP_CONTEXT}"
oc get pods -n openshift-user-workload-monitoring \
  -l app.kubernetes.io/name=thanos-ruler \
  --context "${WORKSHOP_CONTEXT}"
```

Expected output: each listed pod is `Running`. Two replicas are normal on some clusters. A user-workload Alertmanager is not required in this list.

## 4. Open the built-in pod dashboard

**What:** In the web console, open the namespace pod dashboard and set the namespace to the value of `APP_NAMESPACE` (`app-test` unless you changed it).

**Why:** This is the pod view for go-live. This workshop does not add a dashboard pack.

1. Switch to the **Administrator** perspective.
2. Open **Observe** → **Dashboards**.
3. In the dashboard list, select **Kubernetes / Compute Resources / Namespace (Pods)**.
4. Set the namespace variable to `app-test`.

You should see CPU, memory, and pod series for that namespace when pods are running. An empty namespace shows the dashboard with little or no series. That still confirms you are on the right view.

If your build uses a different label for that dashboard, select the pod dashboard in the same **Observe** → **Dashboards** list and write the label on [REPEAT.md](../REPEAT.md).

### Verify

The dashboard title is the pod view, and the namespace selector shows `app-test` (or your `APP_NAMESPACE`).

From the CLI, the same series the dashboard reads for memory:

```bash
echo "${APP_NAMESPACE}"
```

In **Observe** → **Metrics**, run:

```promql
sum by (pod) (
  container_memory_working_set_bytes{namespace="app-test", container!="", container!="POD"}
)
```

Replace `app-test` if your `APP_NAMESPACE` is different. Expected result: one series per running pod, or an empty graph when the namespace has no pods. An error such as `unauthorized` means the console user cannot query metrics. The `oc` login for this workshop is `cluster-admin`, and that user can query.

## Reset

Leave user-workload monitoring on. The test cluster and production both need it.

If this workshop was what turned it on, and you must undo that on a lab cluster, restore the backup from section 1. If the backup does not exist because the ConfigMap was created here, delete only that ConfigMap:

```bash
oc delete configmap cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}" \
  --ignore-not-found
```

Run that delete only when section 1 returned NotFound and you created the ConfigMap in section 2. Deleting a ConfigMap that already had retention or storage keys puts the monitoring operator back on defaults and can disrupt the platform stack.

If user-workload monitoring was already enabled when you started, do nothing in this reset.

## Documentation

- [Configuring user workload monitoring (OpenShift 4.18)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring/configuring-user-workload-monitoring)
- [ROSA monitoring](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/monitoring/index)
