# Workshop Conventions

## Repository Purpose

A collection of hands-on OpenShift adoption workshops. Each workshop lives in a folder named `<platform>-<topic>-ws` and is published as one Antora component in a single site.

## Structure

- `site.yml` and `dev-site.yml` list one content source per workshop (`start_path: <ws>/documentation`)
- `<ws>/documentation/` holds the AsciiDoc courseware (Antora component)
- `<ws>/documentation/modules/ROOT/pages/` holds the pages in `.adoc`
- YAML manifests live in numbered step folders next to `documentation/`, for example `ocp-gitops-agent-ws/01-hub-principal/`
- Steps are progressive: each builds on the previous one
- `MEMORY.md` and `compass-*.md` are local only and git-ignored. Never commit them or copy names from them

## AsciiDoc Formatting

- Every numbered step has a bold *What* and *Why* pair and a collapsible `.Verify:` block
- Use `[.console-input]` before commands and `[.console-output]` before expected output
- Add `icon:clock[] ~N minutes` above sub-steps that take more than 10 seconds, only from a measured run
- Reference manifests by path in `oc apply -f` commands
- No em dashes. Use regular dashes
- `'''` between sub-steps
- Cross-reference with `xref:page.adoc[Label]`, external links with `link:URL[Label]`
- All doc links point to `docs.redhat.com` or `docs.openshift.com`
- Version numbers come from `_attributes.adoc`
- Every step page ends with a `== Reset` section, reverse order, `--ignore-not-found`

## CLI Conventions

- Every `oc` command carries an explicit `--context`
- No `oc login` in step pages, only in `00-setup.adoc` and `myenv.sh`
- Files with `${VARIABLE}` placeholders are applied with `envsubst < file | oc apply -f -`
- Variables are exported once in `00-setup.adoc`

## Content Rules

- Never include customer names, user names or email addresses
- English only
- Prefer `registry.access.redhat.com` or `registry.redhat.io` images
- Use Red Hat and OpenShift components where they cover the use case
- Never publish screenshots, timings or output that did not come from a real run. Mark unvalidated commands in the page

## YAML Conventions

- Start each file with a comment block: file name and purpose
- Label with `app.kubernetes.io/part-of: <app-name>`
- Deployments have requests, limits and probes
