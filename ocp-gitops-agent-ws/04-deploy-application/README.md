# 04 - Deploy an application

Create one `Application` and watch it reach the spoke. Use the section for the mode you chose in step 03.

**Prerequisites**

- [03-spoke-agent](../03-spoke-agent/README.md) finished: the agent shows as connected.
- Contexts and variables from earlier steps are still set.

The sample repository below is the Red Hat getting-started repo. Swap in your own repository and path when you have one.

## A. Managed mode (hub is the source of truth)

**What:** Create the `Application` on the hub, in the namespace named after the agent.

**Why:** The Principal sends every application found in that namespace to that agent. `destination.name` is the agent name. Anything changed by hand on the spoke is reverted.

Save as `application-managed.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-demo-app
  namespace: spoke-a          # the agent name
spec:
  project: default
  source:
    repoURL: https://github.com/redhat-developer/openshift-gitops-getting-started
    targetRevision: HEAD
    path: app
  destination:
    name: spoke-a             # the agent name
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    managedNamespaceMetadata:
      labels:
        argocd.argoproj.io/managed-by: openshift-gitops
    syncOptions:
    - CreateNamespace=true
```

```bash
oc apply -f application-managed.yaml -n "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"
```

With destination-based mapping on both sides, the application may live in any hub namespace.

### Verify

```bash
oc get application.argoproj.io/my-demo-app -n "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"
oc get application.argoproj.io/my-demo-app -n "${AGENT_NS}" --context "${AGENT_CONTEXT}"
oc get all -n my-app --context "${AGENT_CONTEXT}"
```

Expected output: the hub application reaches `Synced` and `Healthy`; a copy exists on the spoke; the sample workload runs in `my-app` on the spoke. The hub UI shows the application and its live resources through the resource proxy.

Check the revert: delete a deployment in `my-app` on the spoke. The agent recreates it.

## B. Autonomous mode (spoke is the source of truth)

**What:** Create the `Application` on the spoke, in the Argo CD namespace.

**Why:** The spoke syncs from Git on its own. The agent reports state to the hub, which mirrors it in the agent's namespace. Changes made on the hub copy are not accepted.

Save as `application-autonomous.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-demo-app
  namespace: openshift-gitops   # the spoke's Argo CD namespace
spec:
  project: default
  source:
    repoURL: https://github.com/redhat-developer/openshift-gitops-getting-started
    targetRevision: HEAD
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    managedNamespaceMetadata:
      labels:
        argocd.argoproj.io/managed-by: openshift-gitops
    syncOptions:
    - CreateNamespace=true
```

```bash
oc apply -f application-autonomous.yaml -n openshift-gitops --context "${AGENT_CONTEXT}"
```

The application must be in the Argo CD installation namespace, otherwise the agent cannot report its status.

### Verify

```bash
oc get application.argoproj.io/my-demo-app -n openshift-gitops --context "${AGENT_CONTEXT}"
oc get application.argoproj.io/my-demo-app -n "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"
```

Expected output: `Synced` and `Healthy` on the spoke; the mirrored copy on the hub shows the same status.

## Reset

```bash
oc delete application.argoproj.io my-demo-app -n "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"   # managed
oc delete application.argoproj.io my-demo-app -n openshift-gitops --context "${AGENT_CONTEXT}"     # autonomous
oc delete namespace my-app --context "${AGENT_CONTEXT}" --ignore-not-found
```

`prune: true` removes the workload with the application in managed mode. Delete `my-app` yourself if it is left over.

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, section 1.8 (`rh_gitops_1.22_argocd_agent.md`).
