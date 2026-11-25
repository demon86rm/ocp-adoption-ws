# Argo CD Agent on OpenShift: one hub, many clusters

Run all GitOps from one hub cluster and let every workload cluster (spoke) pull its work through a small agent. The hub holds the Argo CD UI and API. Each spoke runs its own application controller and its own agent. Spokes open an outbound connection to the hub, so the hub never needs credentials for a spoke's API server.

Written for Red Hat OpenShift GitOps 1.22 (source: *Argo CD Agent installation*, last updated 2026-10-01, kept in this folder as `rh_gitops_1.22_argocd_agent.md`). Not executed on a cluster yet.

## How it fits together

```text
            hub cluster                                spoke cluster
  ┌──────────────────────────────┐            ┌───────────────────────────┐
  │ Argo CD (UI, API, repo-server)│            │ Argo CD (app controller,  │
  │ Principal  ◄──────────────────┼── mTLS ────┤ repo-server, redis)       │
  │ resource proxy                │  gRPC :443 │ Agent                     │
  │ one namespace per agent       │            │ workloads                 │
  └──────────────────────────────┘            └───────────────────────────┘
```

| Part | Where | Job |
| --- | --- | --- |
| Principal | hub | Accepts agent connections, mirrors applications, proxies live-resource requests |
| Agent | each spoke | Connects out to the Principal, applies and reports applications |
| `argocd-agentctl` | your workstation | Creates the CA, certificates, JWT key and agent registration |

Two modes. Choose one per spoke.

| Mode | Source of truth | Create the `Application` on | Pick it when |
| --- | --- | --- | --- |
| Managed | Hub | Hub, in the namespace named after the agent | The hub team governs what runs on every cluster |
| Autonomous | Spoke | Spoke, in `openshift-gitops` | The spoke team owns its apps and the hub only watches |

For "govern everything from one hub", use **managed**.

## What you will have when you finish

- A Principal running on the hub with certificates from your own CA.
- One spoke registered and connected, with `argocd-agentctl agent list` showing it.
- A sample application created on the hub and synced onto the spoke.
- A repeat sheet for adding the next spoke.

## Repo map

| Path | What it is |
| --- | --- |
| [01-prerequisites/](01-prerequisites/README.md) | Tools, contexts, hub Argo CD checks |
| [02-hub-principal/](02-hub-principal/README.md) | Enable the Principal, create agent namespaces, create secrets |
| [03-spoke-agent/](03-spoke-agent/README.md) | Register a spoke, copy certificates, enable the Agent |
| [04-deploy-application/](04-deploy-application/README.md) | Deploy an application in managed and autonomous mode |
| [REPEAT.md](REPEAT.md) | Add another spoke, troubleshooting, production checklist |

## Values used in every step

Set these once per shell. Names are placeholders.

| Variable | Meaning | Example |
| --- | --- | --- |
| `PRINCIPAL_CONTEXT` | `oc` context of the hub | `hub-cluster` |
| `PRINCIPAL_NS` | Namespace of the hub Argo CD | `openshift-gitops` |
| `AGENT_CONTEXT` | `oc` context of the spoke | `spoke-a` |
| `AGENT_NS` | Namespace of the spoke Argo CD | `openshift-gitops` |
| `AGENT_NAME` | Agent name. Also the hub namespace for that agent | `spoke-a` |

## Left out

Custom Helm values beyond the install flags, destination-based mapping (mentioned where it changes a step), high availability of the Principal, and spoke-to-spoke traffic. The Helm install path is summarised in [REPEAT.md](REPEAT.md) because on OpenShift the Argo CD custom resource path does the same job with fewer objects.

## Read this first: where the source doc and this guide differ

- The Red Hat page spells one key `argocdAgent` (section 1.3, step e). Every other place, and the operator API, uses `argoCDAgent`. This guide uses `argoCDAgent`.
- The page's verification output for the spoke (section 1.4, step 4) lists the Principal's four secrets. A spoke has `argocd-agent-ca` and `argocd-agent-client-tls`. This guide uses those two.
- The CLI-generated CA is for development and testing only, per the page. [02-hub-principal](02-hub-principal/README.md) says where to swap in your own PKI.
- ROSA: no step here is confirmed on ROSA HCP. Installing operators, creating routes, and the managed-service limits on `openshift-*` namespaces need a test on a real cluster first.
