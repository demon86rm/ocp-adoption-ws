# Multi-cluster GitOps with the Argo CD Agent

RHACM-style application delivery for a handful of clusters (about 3 to 6) from one hub, without an ACM subscription. Built as an Antora courseware component. Draft: not run on a cluster yet.

Read it as a site: from the repository root run `npm install && npm run dev` and open http://localhost:3000.

| Path | What it is |
| --- | --- |
| `documentation/` | Antora component: pages `00-setup` to `09-rhacm-comparison` (more to come) |
| `00-setup/` | Manifest: GitOps operator subscription |
| `01-hub-principal/` | Manifest: Principal patch for the hub `ArgoCD` |
| `02-register-spokes/` | Manifest: Agent patch for each spoke `ArgoCD` |
| `03-first-application/` | Manifest: first managed `Application` |
| `04-fleet-tagging/` | Manifests: `ApplicationSet`s for Kustomize and Helm, chosen by the `env` tag |
| `05-staged-rollout/` | Manifests: progressive sync flag and the staged `ApplicationSet` |
| `06-teams-projects/` | Manifest: one `AppProject` per environment |
| `07-secrets-bootstrap/` | Manifests: External Secrets operator, Argo CD permissions, platform and demo `ApplicationSet`s |
| `08-production-certificates/` | Manifests: cert-manager operator, issuer, server and agent certificates |
| `REPEAT.md` | Troubleshooting table and teardown order from the test runs |
| `rh_gitops_1.22_argocd_agent.md` | Source: Red Hat *Argo CD Agent installation*, OpenShift GitOps 1.22 |

The sample application is in the public repository `demon86rm/ocp-gitops-fleet-sample`.
