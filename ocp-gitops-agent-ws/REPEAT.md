# Repeat sheet

Use this page to add another spoke, to troubleshoot, and before you go to production. Details live in the step folders.

## Add another spoke

Set a new `AGENT_NAME`, `AGENT_CONTEXT` and `AGENT_NS`, then:

1. Hub: `oc create namespace "${AGENT_NAME}"`, add it to `spec.sourceNamespaces` of the hub `ArgoCD` and to your `AppProject` `sourceNamespaces`. Restart the Argo CD pods after an `AppProject` change.
2. Hub: create the `Certificate` `${AGENT_NAME}-agent`, then `argocd-agentctl agent create "${AGENT_NAME}" ...` ([step 03](03-spoke-agent/README.md) section 1).
3. Copy the CA certificate and the client certificate to the spoke with the two `jq` pipes (section 2).
4. Spoke: enable `spec.argoCDAgent.agent` in the spoke `ArgoCD` with the same `principalServerAddress` (section 3).
5. `argocd-agentctl agent list` shows the new agent connected.

The Principal, its certificates and the JWT key are shared. Nothing in step 02 repeats except the namespace and `sourceNamespaces` entry.

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Principal pod stays `0/1` | One of the four secrets is missing | `oc get secrets -n "${PRINCIPAL_NS}" \| grep agent`, then restart the Principal |
| Agent cannot connect | Wrong `principalServerAddress` or route host, hub not reachable or not resolvable from the spoke, missing secrets on the spoke | Compare with `oc get route openshift-gitops-agent-principal`; test from the spoke network |
| `x509: certificate is valid for …, not <route host>` in the agent log | Principal certificate was issued for a different DNS name, or `insecureGenerate: true` | Add the route host to `dnsNames` of the `argocd-agent-principal-tls` `Certificate` and let cert-manager reissue, and set `insecureGenerate: false`. `tlsClientInSecure=true` on the agent also hides it. Lab only. |
| Principal accepts nothing for an agent | Agent name, hub namespace and `sourceNamespaces` entry differ | They must match exactly |
| Agent log: `redis … i/o timeout` or `NOAUTH Authentication required` | Wrong `redis.serverAddress`, or a network policy blocks the agent from Redis | Match `<ArgoCD name>-redis:6379`. On the Helm path the default operator policy blocks it, add the Redis network policy below. |
| Hub UI: `Resource not found in cluster` and agent log `… is forbidden` | Agent service account lacks access to the target namespace | Grant the agent's service account the rights it needs there |

## Helm install path (alternative to the custom resource)

The Red Hat page also installs the agent from the `redhat-argocd-agent` chart (`helm repo add openshift-helm-charts https://charts.openshift.io/`). Set `agentMode`, `server`, `namespaceOverride`, `argoCdRedisSecretName=<ArgoCD name>-redis-initial-password`, `argoCdRedisPasswordKey=admin.password` and `redisAddress`, and pass `--kube-context`. Managed mode with destination-based mapping adds `destinationBasedMapping` and `createNamespace`. Create this policy first on the spoke, in the agent namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: custom-agent-redis-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: <ArgoCD name>-redis
  ingress:
    - ports:
        - protocol: TCP
          port: 6379
      from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redhat-argocd-agent
  policyTypes:
    - Ingress
```

Whether the custom resource path needs the same policy is not stated on the Red Hat page. Check on a real cluster.

## Certificate lifetime and rotation

| Object | Where it lives | Renewal |
| --- | --- | --- |
| CA (`argocd-agent-ca`) | Hub (key and certificate), spokes (certificate only) | Plan it well before expiry. All issued certificates must end before the CA does. A new CA means updating the CA secret on every spoke. |
| Principal and resource-proxy certificates | Hub | cert-manager renews. The Principal reloads the secret without a restart. |
| Agent client certificate | Created on the hub, copied to the spoke | cert-manager renews on the hub. **Copying to the spoke is manual** until you automate it. Keep the common name. |
| JWT key | Hub | Separate from certificates. Rotate with `jwt create-key --upsert`. |

Suggested values in this guide: `duration: 4320h` (180 days), `renewBefore: 1440h` (60 days). Alert on `Certificate` objects that are not `Ready` and on any agent certificate with less than 30 days left on a spoke.

## Before production

- Use a CA from your organisation's PKI, ideally a dedicated intermediate. The key stays on the hub.
- Keep `insecureGenerate: false` on the Principal and `insecure: false` on every agent.
- Automate the agent certificate copy to each spoke, or put it on a calendar before the end date.
- Narrow the autonomous `AppProject` (`destinations`, `sourceRepos`, `clusterResourceWhitelist`).
- One hub namespace per agent. Do not reuse names.
- Decide who may create applications in agent namespaces on the hub, and restrict that by RBAC.
- Back up the hub `ArgoCD`, the CA secret, the JWT secret and the `cluster-<agent>` secrets, or recreate them from your PKI.
- To retire an agent, use the blocklist and manual cleanup in [03-spoke-agent](03-spoke-agent/README.md) Reset.
- Run the upstream configuration check on your `argocd-agentctl` version (`argocd-agentctl --help`, look for `validate` or `check-config`) after every certificate change.
- Test on the target platform (for example ROSA) before promising it.
