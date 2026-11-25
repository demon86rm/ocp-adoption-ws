# 03 - Spoke: register and enable the Agent

Register one spoke on the hub, give it a certificate, and switch on the Agent in the spoke's Argo CD.

**Prerequisites**

- [02-hub-principal](../02-hub-principal/README.md) finished: the Principal pod is `1/1 Running`, `PRINCIPAL_DNS` is set, the agent namespace exists on the hub.
- `AGENT_CONTEXT`, `AGENT_NS` and `AGENT_NAME` are set.

Pick the mode now and keep it for this spoke: `managed` (hub is the source of truth) or `autonomous`.

```bash
export AGENT_MODE="managed"
```

## 1. Issue the agent's client certificate and register it

**What:** A client certificate for this agent, signed by the hub CA, and the agent record with its Argo CD cluster secret.

**Why:** The Principal reads the agent name from the certificate common name (`auth: "mtls:CN=([^,]+)"`). On the hub, Argo CD does not talk to the spoke's API server. It talks to the Principal's resource proxy, which relays to the agent over the open connection.

```bash
oc apply --context "${PRINCIPAL_CONTEXT}" -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ${AGENT_NAME}-agent
  namespace: ${PRINCIPAL_NS}
spec:
  secretName: ${AGENT_NAME}-agent
  commonName: ${AGENT_NAME}
  usages: [client auth, digital signature, key encipherment]
  duration: 4320h
  renewBefore: 1440h
  privateKey: {algorithm: RSA, size: 4096}
  issuerRef: {name: argocd-agent-ca, kind: Issuer}
EOF

argocd-agentctl agent create "${AGENT_NAME}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --principal-namespace "${PRINCIPAL_NS}" \
  --resource-proxy-server "openshift-gitops-agent-principal-resource-proxy:9090"
```

`agent create` registers the agent and creates the cluster secret `cluster-<AGENT_NAME>`. It signs the resource-proxy credential with the CA in `argocd-agent-ca`. `agent create --help` lists flags such as `--days` and the options to reuse existing secrets. Check them on your `argocd-agentctl` version. Add `--label key=value` to tag the agent.

The common name must stay the same on every renewal. A different name is a different agent.

### Verify

```bash
oc get certificate "${AGENT_NAME}-agent" -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
oc get secret "cluster-${AGENT_NAME}" -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
argocd-agentctl agent list \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"
```

Expected output: the `Certificate` is `READY True`, the secret exists, and the agent is listed (not connected yet).

## 2. Copy the CA certificate and the client certificate to the spoke

**What:** Two secrets in the spoke's Argo CD namespace: `argocd-agent-ca` (public certificate only) and `argocd-agent-client-tls`.

**Why:** The agent verifies the Principal with the CA and proves itself with the client certificate. The CA private key must not leave the hub.

The pipes below move the secrets without writing the private key to disk. Requires `jq`.

```bash
oc get secret "${AGENT_NAME}-agent" -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" -o json \
  | jq '{apiVersion, kind, type: "kubernetes.io/tls",
         metadata: {name: "argocd-agent-client-tls", namespace: env.AGENT_NS},
         data: {"tls.crt": .data["tls.crt"], "tls.key": .data["tls.key"]}}' \
  | oc apply --context "${AGENT_CONTEXT}" -f -

oc get secret argocd-agent-ca -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" -o json \
  | jq '{apiVersion, kind, type: "Opaque",
         metadata: {name: "argocd-agent-ca", namespace: env.AGENT_NS},
         data: {"tls.crt": .data["tls.crt"], "ca.crt": .data["tls.crt"]}}' \
  | oc apply --context "${AGENT_CONTEXT}" -f -
```

The CA secret carries the same certificate under both `tls.crt` and `ca.crt`, because the key name the agent reads for its root CA is not stated on the Red Hat page and upstream docs use both. If the agent log says it cannot load the CA, check this first.

### Verify

```bash
oc get secrets -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" | grep agent
oc get secret argocd-agent-client-tls -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" \
  -o jsonpath='{.data.tls\.crt}' | base64 --decode | openssl x509 -noout -subject -enddate
oc get secret argocd-agent-ca -n "${AGENT_NS}" --context "${AGENT_CONTEXT}" \
  -o jsonpath='{.data.tls\.key}{"\n"}'
```

Expected output: `argocd-agent-ca` and `argocd-agent-client-tls` exist. The subject shows `CN=<AGENT_NAME>` and an end date. The last command prints an empty line, which proves the CA private key is not on the spoke.

### Renewal

cert-manager renews `${AGENT_NAME}-agent` on the hub, but nothing copies the new certificate to the spoke. Repeat the first command of this section before the end date, or run it from a scheduled job or your secret-sync tool. The agent picks up a changed secret on its next TLS handshake. If the new certificate is bad it exits instead of keeping the old one.

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

On the spoke set `spec.argoCDAgent.agent.enabled: false` (or restore `${HOME}/spoke-argocd.backup.yaml` after removing `resourceVersion`, `uid` and `status`) and delete `argocd-agent-ca` and `argocd-agent-client-tls`.

On the hub, upstream documents no command that removes an agent. Do this by hand:

1. Block the certificate so it can never reconnect: `argocd-agentctl blocklist add <fingerprint>` (the fingerprint comes from the client certificate, `openssl x509 -noout -fingerprint -sha256`).
2. Delete the `Certificate` `${AGENT_NAME}-agent` and its secret, the secret `cluster-${AGENT_NAME}`, and the namespace `${AGENT_NAME}`.
3. Remove `${AGENT_NAME}` from `spec.sourceNamespaces` of the hub `ArgoCD` and of your `AppProject`.

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, sections 1.4, 1.6 and 1.7 (`rh_gitops_1.22_argocd_agent.md`).
