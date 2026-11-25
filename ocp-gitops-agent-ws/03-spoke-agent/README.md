# 03 - Spoke: register and enable the Agent

Register one spoke on the hub, give it a certificate, and switch on the Agent in the spoke's Argo CD.

**Prerequisites**

- [02-hub-principal](../02-hub-principal/README.md) finished: the Principal pod is `1/1 Running`, `PRINCIPAL_DNS` is set, the agent namespace exists on the hub.
- `AGENT_CONTEXT`, `AGENT_NS` and `AGENT_NAME` are set.

Pick the mode now and keep it for this spoke: `managed` (hub is the source of truth) or `autonomous`.

```bash
export AGENT_MODE="managed"
```

## 1. Register the agent on the hub

**What:** Create the agent record, its client identity, and the Argo CD cluster secret the hub uses to reach it.

**Why:** On the hub, Argo CD does not talk to the spoke's API server. It talks to the Principal's resource proxy, which relays to the agent over the open connection.

```bash
argocd-agentctl agent create "${AGENT_NAME}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --principal-namespace "${PRINCIPAL_NS}" \
  --resource-proxy-server "openshift-gitops-agent-principal-resource-proxy:9090"
```

Expected output: the agent is created. The command creates a client certificate signed by the Principal CA and a cluster secret named `cluster-<AGENT_NAME>` in the Principal namespace. Add `--label key=value` to tag the agent.

### Verify

```bash
oc get secret "cluster-${AGENT_NAME}" -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
argocd-agentctl agent list \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"
```

Expected output: the secret exists and the agent is listed (not connected yet).

## 2. Give the spoke the CA and a client certificate

**What:** Copy the CA to the spoke, then issue the agent's client certificate.

**Why:** The agent verifies the Principal with the CA and proves itself with the client certificate. The certificate CN is the agent name, which the Principal's `auth` regex reads.

```bash
argocd-agentctl pki propagate \
  --agent-context "${AGENT_CONTEXT}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --principal-namespace "${PRINCIPAL_NS}" \
  --agent-namespace "${AGENT_NS}"

argocd-agentctl pki issue agent "${AGENT_NAME}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --agent-context "${AGENT_CONTEXT}" \
  --agent-namespace "${AGENT_NS}" \
  --principal-namespace "${PRINCIPAL_NS}"
```

`propagate` creates `argocd-agent-ca` on the spoke (`--force` regenerates). `issue agent` creates `argocd-agent-client-tls` (`--upsert` replaces).

### Verify

```bash
oc get secrets -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" | grep agent
```

Expected output: `argocd-agent-ca` and `argocd-agent-client-tls`.

## 3. Enable the Agent in the spoke's Argo CD

**What:** Add `spec.argoCDAgent.agent` to the spoke's `ArgoCD`, and turn off its web server.

**Why:** The spoke keeps the application controller, repo-server and Redis. Users work in the hub's UI, so the spoke's server is not needed.

Back up and edit the spoke instance. Merge these fields. Do not replace the object.

```bash
oc get argocd openshift-gitops -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" -o yaml > "${HOME}/spoke-argocd.backup.yaml"
oc edit argocd openshift-gitops -n "${AGENT_NS}" --context "${AGENT_CONTEXT}"
```

Managed mode:

```yaml
spec:
  server:
    enabled: false
  argoCDAgent:
    agent:
      enabled: true
      client:
        keepAliveInterval: 50s
        mode: managed
        principalServerAddress: "<PRINCIPAL_DNS>"
        principalServerPort: "443"
      logLevel: "info"
      creds: "mtls:any"
      redis:
        serverAddress: "openshift-gitops-redis:6379"
      tls:
        insecure: false
        rootCASecretName: argocd-agent-ca
        secretName: argocd-agent-client-tls
```

Autonomous mode: same, with `mode: autonomous`.

- Replace `<PRINCIPAL_DNS>` with the value from step 02. It is the route host, without `https://`.
- `openshift-gitops-redis` is `<ArgoCD name>-redis` for the default instance.
- Destination-based mapping on a managed agent: add `destinationBasedMapping: {enabled: true, createNamespace: true}` under `agent`. Only if you set it on the Principal too.
- The Red Hat page says this `ArgoCD` must be cluster-scoped (see [01-prerequisites](../01-prerequisites/README.md)).

### Autonomous mode only: create an AppProject

An autonomous agent needs a project named `default` in its Argo CD namespace:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: openshift-gitops
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - name: '*'
    namespace: '*'
    server: '*'
  sourceRepos:
  - '*'
```

That project allows everything. Narrow `destinations` and `sourceRepos` before production use.

### Verify

```bash
oc get pods -n "${AGENT_NS}" --context "${AGENT_CONTEXT}"
```

Expected output: `openshift-gitops-agent-agent-…`, `openshift-gitops-application-controller-0`, `openshift-gitops-redis-…` and `openshift-gitops-repo-server-…`, all `Running`.

## 4. Confirm the connection from both sides

```bash
oc logs deployment/openshift-gitops-agent-agent -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" --tail 20
oc logs deployment/openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" --tail 10
argocd-agentctl agent list \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"
```

Expected output:

- Agent log: "Authentication successful", "Starting event writer", "Starting to send events to event stream", "Starting to receive events from event stream".
- Principal log: "An agent connected to the subscription stream" and "Updated connection status to 'Successful'" for your agent name.
- `agent list` shows the agent as connected.

If any of that is missing, go to the troubleshooting list in [REPEAT.md](../REPEAT.md).

## Reset

On the spoke set `spec.argoCDAgent.agent.enabled: false` (or restore `${HOME}/spoke-argocd.backup.yaml` after removing `resourceVersion`, `uid` and `status`) and delete `argocd-agent-ca` and `argocd-agent-client-tls`. On the hub run `argocd-agentctl agent --help` for the remove command in your build, then delete `cluster-${AGENT_NAME}` and the agent namespace.

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, sections 1.4, 1.6 and 1.7 (`rh_gitops_1.22_argocd_agent.md`).
