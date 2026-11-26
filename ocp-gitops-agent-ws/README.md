# Multi-cluster GitOps with the Argo CD Agent

RHACM-style application delivery for a handful of clusters (about 3 to 6) from one hub, without an ACM subscription. Built as an Antora courseware component. Draft: not run on a cluster yet.

Read it as a site: from the repository root run `npm install && npm run dev` and open http://localhost:3000.

| Path | What it is |
| --- | --- |
| `documentation/` | Antora component: pages `00-setup` to `03-first-application` (more to come) |
| `00-setup/` | Manifest: GitOps operator subscription |
| `01-hub-principal/` | Manifest: Principal patch for the hub `ArgoCD` |
| `02-register-spokes/` | Manifest: Agent patch for each spoke `ArgoCD` |
| `03-first-application/` | Manifest: first managed `Application` |
| `REPEAT.md` | Older notes (add a spoke, troubleshooting, production checklist). To be folded into the pages |
| `rh_gitops_1.22_argocd_agent.md` | Source: Red Hat *Argo CD Agent installation*, OpenShift GitOps 1.22 |

Planned pages: fleet tagging with `ApplicationSet`, Helm and Kustomize; staged rollout; teams and RBAC; secrets and spoke bootstrap; production certificates with cert-manager; comparison with RHACM.
