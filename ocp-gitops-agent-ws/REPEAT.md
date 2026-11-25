# Repeat sheet

Use this page to add another spoke, to troubleshoot, and before you go to production. Details live in the step folders.

## Add another spoke

Set a new `AGENT_NAME`, `AGENT_CONTEXT` and `AGENT_NS`, then:

1. Hub: `oc create namespace "${AGENT_NAME}"`, add it to `spec.sourceNamespaces` of the hub `ArgoCD` and to your `AppProject` `sourceNamespaces`. Restart the Argo CD pods after an `AppProject` change.
2. Hub: `argocd-agentctl agent create "${AGENT_NAME}" ...` ([step 03](03-spoke-agent/README.md) section 1).
3. `argocd-agentctl pki propagate ...` then `argocd-agentctl pki issue agent "${AGENT_NAME}" ...` (section 2).
4. Spoke: enable `spec.argoCDAgent.agent` in the spoke `ArgoCD` with the same `principalServerAddress` (section 3).
5. `argocd-agentctl agent list` shows the new agent connected.

The Principal, its certificates and the JWT key are shared. Nothing in step 02 repeats except the namespace and `sourceNamespaces` entry.

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Principal pod stays `0/1` | One of the four secrets is missing | `oc get secrets -n "${PRINCIPAL_NS}" \| grep agent`, then restart the Principal |
| Agent cannot connect | Wrong `principalServerAddress` or route host, hub not reachable or not resolvable from the spoke, missing secrets on the spoke | Compare with `oc get route openshift-gitops-agent-principal`; test from the spoke network |
| `x509: certificate is valid for …, not <route host>` in the agent log | Principal certificate was issued for a different DNS name, or `insecureGenerate: true` | Re-issue with `pki issue principal --dns <route host> --upsert`, and set `insecureGenerate: false`. `tlsClientInSecure=true` on the agent also hides it. Lab only. |
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

## Before production

- Replace the CLI-generated CA and certificates with your PKI. Same secret names. Plan certificate renewal, because an expired certificate disconnects the agent.
- Keep `insecureGenerate: false` on the Principal and `insecure: false` on every agent.
- Narrow the autonomous `AppProject` (`destinations`, `sourceRepos`, `clusterResourceWhitelist`).
- One hub namespace per agent. Do not reuse names.
- Decide who may create applications in agent namespaces on the hub, and restrict that by RBAC.
- Back up the hub `ArgoCD`, the four Principal secrets and the `cluster-<agent>` secrets, or recreate them from your PKI and `argocd-agentctl`.
- Test on the target platform (for example ROSA HCP) before promising it.
