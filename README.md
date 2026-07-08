# ace-infra

Build and pipeline infrastructure for IBM App Connect Enterprise (ACE) on Cloud Pak for Integration.
The App Connect sibling of [`mq-infra`](https://github.com/ibmclientengineering/mq-infra) — same GitOps rails, an integration instead of a queue manager.

Maintained by IBM Client Engineering.

## `tekton/` — the reusable factory

A **reusable Tekton task library** plus a delivery pipeline. One pipeline serves every integration — change the params, not the tasks. This is the 2026 App Connect model: **`ibmint` on the current 13.x image** producing a BAR, delivered as an **`IntegrationRuntime`** CR (barURL + `Configuration`s) — *not* a Toolkit-exported BAR and *not* the deprecated `IntegrationServer` / `Dashboard` / `Designer` CRs the 12.0.x-era guides emitted.

### `tekton/tasks/` — reusable across integrations
| Task | What it does | Key params |
|---|---|---|
| `ibm-git-clone` | Shallow-clone the integration source into the workspace. Generic. | `git-url`, `git-revision`, `subdir` |
| `ibm-ace-bar-build` | `ibmint package` on the 13.x server image → a BAR. Copies source to a writable dir first (ibmint compiles Java in place). | `projects`, `bar-file`, `ace-image` |
| `ibm-ace-bar-publish` | Publish the BAR to the in-cluster BAR host (nginx) that serves the `barURL`. | `bar-host-namespace`, `bar-host-selector` |
| `ibm-ace-gitops-integrationruntime` | Render an **`IntegrationRuntime`** CR into `ace/environments/<env>/<name>/` of gitops-apps and push to a delivery branch. | `environment`, `runtime-name`, `bar-url`, `configurations`, `license`, `version` |

### `tekton/pipelines/`
- `ace-createcustomer-dev` — wires the four tasks: **clone → BAR build → publish → GitOps delivery**. The pipeline's only exit is a commit; Argo CD does the deploying, exactly as MQ's `mq-qm-dev` does for a `QueueManager`.

### Install
```
oc apply -n <pipelines-namespace> -k tekton
```
Then start a run against your integration source; the delivery lands an `IntegrationRuntime` in the gitops-apps `dev` overlay for Argo CD to reconcile.

### Notes from the live build-out
- The 13.x `ace-server-prod` image pre-applies `mqsiprofile` via its baked env — `ibmint` is already on `PATH`; do **not** re-source `mqsiprofile` (it errors *"repetition disallowed"*).
- `ibmint` compiles the Java project **in place**, so the git-cloned tree (owned by the clone step's user) must be copied to a dir the ACE image's user owns before packaging.
- `oc cp` refuses to overwrite an existing file ("File exists"); the publish task removes the old BAR first.
- The publish step needs the pipeline service account to have `edit` on the BAR-host namespace (for the cross-namespace `oc cp`).
