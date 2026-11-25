# 01 - Setup

Confirm you are `cluster-admin` on one ROSA cluster, name that `oc` context, and set the application namespace this workshop watches. Later steps apply changes only to that context.

**Prerequisites**

- ROSA, OpenShift 4.16 or newer.
- `oc` installed and already logged in with your own credentials.
- Web console access to the same cluster.

This step does not change the cluster.

## 1. Select one context

**What:** Export the `oc` context you will use for every command in this workshop.

**Why:** Later commands pass `--context` explicitly so a second cluster in your kubeconfig is left alone.

```bash
oc config get-contexts
export WORKSHOP_CONTEXT="<name-from-the-current-column>"
oc whoami --context "${WORKSHOP_CONTEXT}"
oc get clusterversion version \
  -o jsonpath='{.status.desired.version}{"\n"}' \
  --context "${WORKSHOP_CONTEXT}"
```

Expected output: your username, then a version of `4.16` or newer (for example `4.18.12`).

Stop if the version is older than 4.16. The alert manifest in step 03 uses `AlertingRule`, which is the supported API for platform alerts from 4.16.

### Verify

```bash
oc config current-context
echo "${WORKSHOP_CONTEXT}"
```

Expected output: both lines are the same context name. If `current-context` differs, later commands still use `WORKSHOP_CONTEXT`. That is intentional.

## 2. Confirm cluster-admin

**What:** Check that this login can edit the monitoring ConfigMap and can create an `AlertingRule`.

**Why:** Both objects live in `openshift-monitoring`. A project admin on the application namespace cannot create them.

```bash
oc auth can-i update configmap/cluster-monitoring-config \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}"
oc auth can-i create alertingrule.monitoring.openshift.io \
  -n openshift-monitoring \
  --context "${WORKSHOP_CONTEXT}"
```

Expected output: `yes` on both lines. If either line is `no`, switch to a `cluster-admin` login before step 02.

### Verify

```bash
oc auth can-i '*' '*' --context "${WORKSHOP_CONTEXT}"
```

Expected output: `yes`.

## 3. Set the namespace variable

**What:** Export the test namespace the pod dashboard and the alert selector use.

**Why:** The rule file ships with `app-test` written in the PromQL. This variable is the reminder you use in the console and in the discovery queries.

```bash
export APP_NAMESPACE="app-test"
oc get ns "${APP_NAMESPACE}" --context "${WORKSHOP_CONTEXT}"
```

Expected output: the namespace exists and is `Active`. If the name on your test cluster differs, export that name instead, and use the same name in the regex in [application-alerting-rule.yaml](../03-application-alerts/application-alerting-rule.yaml) before step 03.

Production is not a second rule. When you repeat this workshop there, add that namespace to the same regex (`app-test|<production-namespace>`). Until the production name is known, the comment in the manifest uses `app-prod` as the placeholder.

### Verify

```bash
echo "${APP_NAMESPACE}"
oc get pods -n "${APP_NAMESPACE}" --context "${WORKSHOP_CONTEXT}"
```

Expected output: the namespace name, then the workload pods (or an empty list if nothing is deployed yet). An empty list still allows the rule to load. The pod dashboard will show series once pods exist.

## ROSA monitoring configuration

`cluster-monitoring-config` in `openshift-monitoring` may already contain retention, storage, or other keys managed for ROSA. Step 02 adds a single key, `enableUserWorkload: true`, by editing the live ConfigMap.

Keep a copy of the other keys. Replacing the ConfigMap with a file that contains only `enableUserWorkload` drops those keys and can make the `monitoring` cluster operator degrade. The file in step 02 is a snippet to read, not a file to apply onto an existing ConfigMap.

## Reset

Nothing was created on the cluster. To drop the shell variables:

```bash
unset WORKSHOP_CONTEXT APP_NAMESPACE
```

## Documentation

- [ROSA monitoring](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/monitoring/index)
