# Repeat sheet: add a spoke, troubleshoot, tear down

Short reference for people who have already done the workshop. The workshop pages are the source of truth. This file lists what went wrong during testing on two ROSA HCP clusters and how it showed up.

## Add another spoke

Follow the bootstrap checklist in `documentation/modules/ROOT/pages/07-secrets-bootstrap.adoc` (section "Bootstrap checklist for a new spoke"). It was assembled from steps that were each run, but not run as a whole on a third cluster.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Principal pod `CrashLoopBackOff` right after step 1 | The secrets do not exist yet | Create the certificates and JWT key. It recovers by itself in about a minute |
| `ERR_BAD_SSL_CLIENT_AUTH_CERT` opening the Principal route in a browser | The Principal requires a client certificate | Expected. Open the `openshift-gitops-server` route for the UI |
| `argocd-agentctl pki issue agent` says principal and agent cannot share a context | The tool refuses it | Add `--same-context` (only on `pki issue agent`) |
| Application stays `Unknown`, "project ... does not exist" | The project was never sent to that spoke, or does not list that cluster as a destination | Add the cluster as a destination in the `AppProject` |
| Sync fails with `forbidden` in the target namespace | `argocd.argoproj.io/managed-by` names the wrong Argo CD instance | Use the spoke's own instance name. Delete the namespace so it is recreated with the right label |
| `ApplicationSet` apps sit in `openshift-gitops` with no status | Namespace-based mapping ignores the template namespace | Use destination-based mapping on the Principal and every agent |
| `clustersecretstores.external-secrets.io is forbidden` on sync | Argo CD does not know API groups added by a later operator | Grant the controller a `ClusterRole` for that group (step 7) |
| Route returns 503 after a Kustomize `namePrefix` | Kustomize does not rewrite the Service name inside a `Route` | Do not prefix names |
| Sample pod crash-loops with `could not bind to address 0.0.0.0:80` | Image binds port 80 under the restricted SCC | Use an image that listens on 8080 |
| Unexpected AWS load balancer appears | A chart set `service.type: LoadBalancer` | Use `ClusterIP` plus a `Route` |
| Spoke agent logs `source UID annotation not found` | Benign noise about GPG keys and the `default` project | Ignore |
| Renewed agent certificate is not used | cert-manager renews on the hub only | Copy it to the spoke again (step 8) |

## Tear down (order matters)

1. Delete the `ApplicationSet`s on the hub (steps 7, 6, 5, 4), then the first application, so nothing recreates workloads.
2. Delete the namespaces they created on each spoke.
3. Remove the extra roles and role bindings (steps 6 and 7).
4. Disable the Agent on each spoke, then the Principal on the hub (steps 2 and 1).
5. Remove the operators you installed (External Secrets, cert-manager, GitOps). Deleting a `Subscription` leaves its `ClusterServiceVersion`.
6. Delete the clusters. They cost money while they run.

Each page ends with a `Reset` section for its own step.
