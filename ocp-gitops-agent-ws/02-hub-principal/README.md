# 02 - Hub: enable the Principal

Turn the hub's Argo CD into the control plane: enable the Principal, create one namespace per agent, then create the certificates and JWT key the Principal needs.

**Prerequisites**

- [01-prerequisites](../01-prerequisites/README.md) finished in this shell.
- cert-manager Operator for Red Hat OpenShift installed on the hub.

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

**What:** A CA secret, an `Issuer`, a server certificate for the Principal's gRPC endpoint, a server certificate for the resource proxy, and a signing key for agent tokens. cert-manager issues and renews the certificates.

**Why:** All Principal and agent traffic is mutually authenticated TLS, and agents receive short-lived JWTs signed with the key. Certificates expire, so they must renew without hand work.

The `argocd-agentctl pki` commands are for development and testing only (upstream docs). This guide does not use them.

### 3a. The CA secret

The Principal signs and verifies with one CA. Create it as a TLS secret named `argocd-agent-ca` in the Principal namespace on the hub. Its private key stays on the hub and is never copied to a spoke.

**Production:** use a CA certificate and key issued by your organisation's PKI (a dedicated intermediate is best, so you can revoke it alone).

```bash
oc create secret tls argocd-agent-ca \
  --cert=<ca.crt> --key=<ca.key> \
  -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

**Lab only, no CA available:** make a throwaway one, load it, and delete the key file.

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 365 \
  -subj "/CN=argocd-agent-ca" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -out ca.crt
oc create secret tls argocd-agent-ca --cert=ca.crt --key=ca.key \
  -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
shred -u ca.key 2>/dev/null || rm -P ca.key
```

### 3b. Issuer and certificates

The shell expands `${PRINCIPAL_DNS}` and `${PRINCIPAL_NS}` in this heredoc, so run it in the shell where they are set. `duration` and `renewBefore` are suggested values, not values from the Red Hat page. Every certificate must stay inside the CA's own validity.

```bash
oc apply --context "${PRINCIPAL_CONTEXT}" -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: argocd-agent-ca
  namespace: ${PRINCIPAL_NS}
spec:
  ca:
    secretName: argocd-agent-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-agent-principal-tls
  namespace: ${PRINCIPAL_NS}
spec:
  secretName: argocd-agent-principal-tls
  commonName: principal
  dnsNames:
    - ${PRINCIPAL_DNS}
    - openshift-gitops-agent-principal.${PRINCIPAL_NS}.svc.cluster.local
  usages: [server auth, digital signature, key encipherment]
  duration: 4320h
  renewBefore: 1440h
  privateKey: {algorithm: RSA, size: 4096}
  issuerRef: {name: argocd-agent-ca, kind: Issuer}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-agent-resource-proxy-tls
  namespace: ${PRINCIPAL_NS}
spec:
  secretName: argocd-agent-resource-proxy-tls
  commonName: resource-proxy
  dnsNames:
    - openshift-gitops-agent-principal-resource-proxy
    - openshift-gitops-agent-principal-resource-proxy.${PRINCIPAL_NS}.svc.cluster.local
  usages: [server auth, client auth, digital signature, key encipherment]
  duration: 4320h
  renewBefore: 1440h
  privateKey: {algorithm: RSA, size: 4096}
  issuerRef: {name: argocd-agent-ca, kind: Issuer}
EOF
```

- The first DNS name of the Principal certificate must equal the route host (or load-balancer host) that agents dial. A mismatch shows up as a certificate error in the agent log.
- The resource-proxy name is `<ArgoCD name>-agent-principal-resource-proxy`. Argo CD reaches it by that service name inside the hub.
- If clients reach the Principal by IP, add `ipAddresses`.

### 3c. The JWT signing key

cert-manager does not issue this key. It is an RSA key, not a certificate, and rotates separately.

```bash
argocd-agentctl jwt create-key \
  --principal-namespace "${PRINCIPAL_NS}" \
  --principal-context "${PRINCIPAL_CONTEXT}"
```

This creates a 4096-bit PKCS#8 key in the `argocd-agent-jwt` secret. `--upsert` (`-u`) replaces it. For production, create that secret from a key generated and stored under your own key management and keep the same secret name.

### Verify

```bash
oc get certificate -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
oc get secrets -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" | grep agent
oc get pod -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"
```

Expected output: both `Certificate` objects `READY True`. Four secrets (`argocd-agent-ca`, `argocd-agent-principal-tls`, `argocd-agent-resource-proxy-tls`, `argocd-agent-jwt`). The Principal pod becomes `1/1 Running`. If it stays `0/1`, restart it: `oc rollout restart deployment/openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}"`.

Check the log:

```bash
oc logs deployment/openshift-gitops-agent-principal -n "${PRINCIPAL_NS}" --context "${PRINCIPAL_CONTEXT}" --tail 20
```

Expected output: lines such as "Redis proxy started", "Resource proxy started" and "Namespace informer synced and ready".

Renewals: the upstream docs say the Principal reloads its certificate secrets without a restart. A bad certificate makes it exit, so it will crash-loop instead of serving a broken certificate. Watch the `Certificate` objects, not just the pods.

## Reset

Disable the Principal by setting `spec.argoCDAgent.principal.enabled: false` and turn the controller back on with `spec.controller.enabled: true`, or restore `${HOME}/hub-argocd.backup.yaml` (remove `resourceVersion`, `uid` and `status` first). Delete the agent namespace, the two `Certificate` objects, the `Issuer` and the four secrets if you want a clean start.

## Documentation

- Red Hat OpenShift GitOps 1.22, *Argo CD Agent installation*, section 1.3 (`rh_gitops_1.22_argocd_agent.md`).
- Argo CD Agent upstream docs, TLS certificates: https://argocd-agent.readthedocs.io/latest/configuration/tls-certificates/
