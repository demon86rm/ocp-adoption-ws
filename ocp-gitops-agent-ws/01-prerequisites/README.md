# 01 - Prerequisites

Confirm tools, contexts and the hub Argo CD before changing anything. This step changes nothing on a cluster.

**Prerequisites**

- Two OpenShift clusters: one hub, one spoke. The spoke must reach the hub's Principal route (or load balancer) over HTTPS, and DNS for the hub host must resolve from the spoke.
- Red Hat OpenShift GitOps Operator 1.22 installed on both.
- `cluster-admin` on both clusters.

## 1. Install the tools

**What:** `oc`, `helm` (later than v3.8.0, only needed for the Helm path), and `argocd-agentctl`.

**Why:** `argocd-agentctl` creates the certificates and registers each spoke.

```bash
oc version --client
helm version --short
argocd-agentctl version
```

Expected output: a version for each. Get `argocd-agentctl` from the link in the Additional Resources section of the Red Hat page.

## 2. Name one context per cluster

**What:** Log in to both clusters and give each a distinct context name.

**Why:** Every command in this guide passes `--context` or `--principal-context` and `--agent-context`, so a wrong default cannot touch the wrong cluster.

```bash
oc config get-contexts
export PRINCIPAL_CONTEXT="<hub-context>"
export AGENT_CONTEXT="<spoke-context>"
export PRINCIPAL_NS="openshift-gitops"
export AGENT_NS="openshift-gitops"
export AGENT_NAME="spoke-a"
oc whoami --context "${PRINCIPAL_CONTEXT}"
oc whoami --context "${AGENT_CONTEXT}"
```

Expected output: your user name twice. To rename a context: `oc config rename-context <old> <new>`.

### Verify

```bash
oc auth can-i '*' '*' --context "${PRINCIPAL_CONTEXT}"
oc auth can-i '*' '*' --context "${AGENT_CONTEXT}"
```

Expected output: `yes` twice.

## 3. Check the operator and the hub Argo CD

**What:** Confirm the operator version and that the default Argo CD instance exists on the hub.

**Why:** The Principal is switched on inside an existing Argo CD instance. It must be cluster-scoped and must allow applications in other namespaces ("Apps in any namespace").

```bash
oc get csv -n openshift-gitops-operator --context "${PRINCIPAL_CONTEXT}"
oc get argocd -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

Expected output: a `openshift-gitops-operator` ClusterServiceVersion at 1.22 in `Succeeded`, and an `ArgoCD` named `openshift-gitops`. Repeat both on the spoke with `AGENT_CONTEXT` and `AGENT_NS`.

### Cluster scope and apps in any namespace

The operator makes the default instance cluster-scoped for the namespace in its `ARGOCD_CLUSTER_CONFIG_NAMESPACES` setting. Check it on both clusters:

```bash
oc get subscription openshift-gitops-operator \
  -n openshift-gitops-operator \
  --context "${PRINCIPAL_CONTEXT}" \
  -o jsonpath='{.spec.config.env}{"\n"}'
```

Expected output: an entry `ARGOCD_CLUSTER_CONFIG_NAMESPACES` whose value includes `openshift-gitops`. If it is missing, follow "cluster-scoped Argo CD instance" in the Additional Resources of the Red Hat page. The page lists it as a prerequisite and does not repeat the steps. `sourceNamespaces` (apps in any namespace) is set in step 02.

### Verify

You have two context names, `cluster-admin` on both, a 1.22 operator on both, and a default `ArgoCD` on both.

## Reset

Nothing was created on a cluster. To drop the shell variables:

```bash
unset PRINCIPAL_CONTEXT PRINCIPAL_NS AGENT_CONTEXT AGENT_NS AGENT_NAME
```

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, sections 1.1 and 1.2 (`rh_gitops_1.22_argocd_agent.md`).
