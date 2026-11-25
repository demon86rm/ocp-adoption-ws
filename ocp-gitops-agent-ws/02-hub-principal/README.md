# 02 - Hub: enable the Principal

Turn the hub's Argo CD into the control plane: enable the Principal, create one namespace per agent, then create the certificates and JWT key the Principal needs.

**Prerequisites**

- [01-prerequisites](../01-prerequisites/README.md) finished in this shell.

Set the agent name used for the first spoke if you have not (`AGENT_NAME`). The hub namespace for an agent must have exactly the agent's name.

## 1. Enable the Principal

**What:** Add `spec.argoCDAgent.principal` to the hub's `ArgoCD`, switch the hub application controller off, and list the agent namespaces in `sourceNamespaces`.

**Why:** With the agent model each spoke runs the controller. The hub keeps only the UI, API, repo-server, Redis and the Principal. The Principal maps one agent to one namespace, so `sourceNamespaces` must name every agent namespace exactly.

Review the existing instance first, then merge these fields into it. Do not replace the whole object: the default instance carries operator settings you want to keep.

```bash
oc get argocd openshift-gitops -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" -o yaml > "${HOME}/hub-argocd.backup.yaml"
```

Fields to set (managed and autonomous agents on the first two spokes shown):

```yaml
spec:
  controller:
    enabled: false
  argoCDAgent:
    principal:
      enabled: true
      auth: "mtls:CN=([^,]+)"
      logLevel: "info"
      namespace:
        allowedNamespaces:
          - "*"
      tls:
        insecureGenerate: false
  sourceNamespaces:
    - "spoke-a"
```

Apply with a merge patch, or edit the object:

```bash
oc edit argocd openshift-gitops -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

Notes on the fields:

- `auth: "mtls:CN=([^,]+)"` takes the agent name from the certificate common name. The agent certificate you create in step 03 has the agent name as its CN.
- `insecureGenerate: false` means you supply the certificates (section 3). If you set `true` the Principal generates throwaway ones and agents must disable verification. Do not use that outside a lab.
- `sourceNamespaces` is a list. Add one entry per agent. If an entry does not match an agent name the Principal cannot place or watch that agent's applications.
- Destination-based mapping is an alternative for managed agents only: add `destinationBasedMapping: true` under `principal`. Applications may then live in any hub namespace and pick the agent with `spec.destination.name`. This guide uses the default namespace-based mapping.

### Verify

```bash
oc get pod -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
oc get svc -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
oc get route -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

Expected output: pods for the Principal, Redis, repo-server and server. The Principal pod shows `0/1 Running` until the secrets exist. That is normal here. Services include `openshift-gitops-agent-principal` (443), `-healthz`, `-metrics`, `-redisproxy` and `-resource-proxy` (9090). A route `openshift-gitops-agent-principal` exists.

Record the route host. It is the `PRINCIPAL_DNS` value for later steps:

```bash
export PRINCIPAL_DNS="$(oc get route openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" -o jsonpath='{.spec.host}')"
echo "${PRINCIPAL_DNS}"
```

To use a load balancer instead of the route, set `spec.argoCDAgent.principal.server.service.type: LoadBalancer` and `server.route.enabled: false`. Then use the service's `.status.loadBalancer.ingress[0].hostname` as `PRINCIPAL_DNS`.

## 2. Create the agent namespace on the hub

**What:** One namespace per agent, named exactly like the agent.

**Why:** In managed mode the hub sends only applications found in the agent's namespace. In autonomous mode the Principal creates mirrored applications there.

```bash
oc create namespace "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"
```

Also add the namespace to the `AppProject` that your hub applications use, under `spec.sourceNamespaces`. If you change an `AppProject`, restart the Argo CD pods.

### Verify

```bash
oc get ns "${AGENT_NAME}" --context "${PRINCIPAL_CONTEXT}"
oc get argocd openshift-gitops -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" -o jsonpath='{.spec.sourceNamespaces}{"\n"}'
```

Expected output: namespace `Active`; the agent name appears in the list.

## 3. Create the certificates and the JWT key

**What:** A CA, a server certificate for the Principal's gRPC endpoint, a server certificate for the resource proxy, and a signing key for agent tokens.

**Why:** All Principal and agent traffic is mutually authenticated TLS, and agents receive short-lived JWTs signed with the key.

> The CLI-generated PKI below is for development and testing only. For production, issue the CA and the two server certificates from your organisation's PKI and create the same secrets from them. Keep the secret names the same.

```bash
argocd-agentctl pki init \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"

argocd-agentctl pki issue principal \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --dns "${PRINCIPAL_DNS}"

argocd-agentctl pki issue resource-proxy \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}" \
  --dns "openshift-gitops-agent-principal-resource-proxy"

argocd-agentctl jwt create-key \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"
```

What each does:

- `pki init` creates a self-signed CA and the `argocd-agent-ca` secret.
- `pki issue principal` creates `argocd-agent-principal-tls`. `--dns` must equal the route host (or load-balancer host) that agents dial. A mismatch shows up later as a certificate error in the agent log. Add `--ip` for a comma-separated list of IPs if clients use IPs.
- `pki issue resource-proxy` creates `argocd-agent-resource-proxy-tls`. The DNS name is the in-cluster service name, `<ArgoCD name>-agent-principal-resource-proxy`.
- `jwt create-key` creates a 4096-bit RSA key in `argocd-agent-jwt`.
- Each command takes `--upsert` (`-u`) to replace an existing object.

### Verify

```bash
oc get secrets -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" | grep agent
oc get pod -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

Expected output: four secrets (`argocd-agent-ca`, `argocd-agent-principal-tls`, `argocd-agent-resource-proxy-tls`, `argocd-agent-jwt`). The Principal pod becomes `1/1 Running`. If it stays `0/1`, restart it: `oc rollout restart deployment/openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"`.

Check the log:

```bash
oc logs deployment/openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" --tail 20
```

Expected output: lines such as "Redis proxy started", "Resource proxy started" and "Namespace informer synced and ready".

## Reset

Disable the Principal by setting `spec.argoCDAgent.principal.enabled: false` and turn the controller back on with `spec.controller.enabled: true`, or restore `${HOME}/hub-argocd.backup.yaml` (remove `resourceVersion`, `uid` and `status` first). Delete the agent namespace and the four secrets if you want a clean start.

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, section 1.3 (`rh_gitops_1.22_argocd_agent.md`).
