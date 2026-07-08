# ace-infra

The IBM App Connect Enterprise (ACE) build-and-delivery **factory**: a reusable Tekton task library and pipeline that turns integration source into a running integration on Cloud Pak for Integration — the App Connect sibling of [`mq-infra`](https://github.com/ibmclientengineering/mq-infra).

Part of the **IBM Client Engineering Cloud Pak for Integration production-deployment demo**: `ace-infra` is the App Connect *factory* whose only output is a GitOps commit — Argo CD, watching the `-apps` repo, does the actual deploying.

## What this is

One pipeline, `ace-createcustomer-dev`, wires four reusable tasks into a supply chain: **clone → build a BAR → publish it → deliver an `IntegrationRuntime` CR as a git commit**. Change the params, not the tasks, and the same factory serves every integration. It builds the **2026 App Connect model** — `ibmint` on the 13.x server image producing a BAR, delivered as an **`IntegrationRuntime`** (barURL + `Configuration`s) — *not* a Toolkit-exported BAR and *not* the deprecated `IntegrationServer` / `Dashboard` / `Designer` CRs the 12.0.x-era guides emitted.

## What's inside

```
tekton/
├── tasks/        # the reusable factory — same tasks, any integration
│   ├── ibm-git-clone.yaml                    # shallow-clone the source into the workspace (generic)
│   ├── ibm-ace-bar-build.yaml                # ibmint package on the 13.x image → a BAR
│   ├── ibm-ace-bar-publish.yaml              # push the BAR to the in-cluster BAR host (nginx) that serves the barURL
│   └── ibm-ace-gitops-integrationruntime.yaml# render an IntegrationRuntime CR into the -apps overlay and commit it
├── pipelines/
│   └── ace-createcustomer-dev.yaml           # wires the four tasks; exits with a commit, never a deploy
└── kustomization.yaml                        # oc apply -k tekton  installs the whole library
```

`tekton/README.md` documents each task's params and the hard-won live-build notes (see below).

## Why it's built this way

- **Factory, not a script.** The tasks are integration-agnostic; the pipeline supplies the params. One library builds every integration, so adding a customer is a new pipeline run, not new YAML to maintain.
- **Separation of duties.** Build (ibmint), publish (BAR host), and delivery (git commit) are distinct, individually reusable steps. Swap the BAR host or the target repo without touching the build.
- **GitOps as the only exit.** The pipeline's sole output is a commit on a delivery branch — nothing is `oc apply`'d to a live namespace by the build. Argo CD reconciles the desired state from git, so every deployment is auditable, reviewable, and reversible: **promotion is a pull request**, and rollback is `git revert`.
- **The current CR model.** Delivering an `IntegrationRuntime` (barURL + `Configuration`s) instead of the retired `IntegrationServer` means the demo reflects how App Connect is actually shipped in 2026 — licenses, operand version, and image are re-derived from the live operator, not copied from a stale blog.
- **Learned in the cluster, not from docs.** The 13.x `ace-server-prod` image bakes `mqsiprofile` into its env (re-sourcing it errors *"repetition disallowed"*); `ibmint` compiles Java **in place**, so the cloned tree is copied to a dir the ACE user owns before packaging; `oc cp` refuses to overwrite, so publish deletes the old BAR first. Those notes live in the code so the next builder doesn't relearn them.

## How it fits the bigger picture

`ace-infra` is one of two build **factories** that feed the same GitOps rails:

- **[`mq-infra`](https://github.com/ibmclientengineering/mq-infra)** — the sibling factory. Same pattern, a `QueueManager` instead of an `IntegrationRuntime`; `mq-qm-dev` is the twin of `ace-createcustomer-dev`.
- **[`multi-tenancy-gitops-apps`](https://github.com/ibmclientengineering/multi-tenancy-gitops-apps)** — where this factory's commits land: the `IntegrationRuntime` is rendered into `ace/environments/<env>/<name>/`, and Argo CD deploys it. This is the `-apps` layer of the four-repo `multi-tenancy-gitops` family (root / infra / services / apps) that composes the whole cluster.

Together: `ace-infra` and `mq-infra` **build**, the `multi-tenancy-gitops` repos **describe the cluster**, and Argo CD **deploys** — the full production-deployment story for the demo.

Maintained by IBM Client Engineering.
