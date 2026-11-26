# OpenShift adoption workshops

Hands-on, repeatable labs for adopting Red Hat OpenShift. Each topic is one folder with numbered steps, a verify block after each step, a reset, and a one-page `REPEAT.md` for running it again on another cluster.

| Folder | Topic | Platform |
| --- | --- | --- |
| [rosa-monitoring-ws/](rosa-monitoring-ws/README.md) | Application monitoring and alerts with the built-in Prometheus stack | ROSA |
| [ocp-gitops-agent-ws/](ocp-gitops-agent-ws/README.md) | One hub governing many clusters with the Argo CD Agent | OpenShift (any) |

## Status

Drafts. Nothing here has been run on a cluster yet. Check each workshop's own README for the platform versions it assumes and for known open points before using it.

## Conventions

- One folder per topic, named `<platform>-<topic>-ws`: `ocp-` when it works on any OpenShift cluster, `rosa-` when it depends on ROSA.
- Each folder has a `README.md` (overview and variables), numbered step folders, and `REPEAT.md`.
- Commands pass `--context` explicitly so a second cluster in your kubeconfig is never touched.
- No customer names, people or organisations in workshop content. Namespaces and cluster names are placeholders.

## Adding a workshop

1. Create `<platform>-<topic>-ws/` with `README.md`, step folders and `REPEAT.md`.
2. Add a row to the table above.
3. Keep the lab under about an hour and runnable from the folder alone.

## Read the workshops as a site

```bash
npm install
npm run dev          # live reload at http://localhost:3000
npx antora site.yml  # one-off build into gh-pages/
```

Antora 2.3 hangs silently on Node 26 (no output, no `gh-pages/`). Build in the pinned container instead:

```bash
podman run --rm -v "$PWD":/antora:Z docker.io/antora/antora@sha256:e2c2a3e2ae01f6dcb26ca4c6fe6bc335a06e05d23c976b5730567100d8475ed4 site.yml
```

Each workshop folder is one Antora component, listed in `site.yml`. `AGENTS.md` holds the writing conventions.
