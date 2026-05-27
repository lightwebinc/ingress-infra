# Repository Rename & Remediation Plan

## Current State

### `lightweb-inc` (personal account) — 9 repos, transfer FAILED

| Current name            | New name        |
| ----------------------- | --------------- |
| ingress-infra         | ingress-infra   |
| bitcoin-listener        | listener-infra  |
| bitcoin-multicast       | bsv-multicast   |
| bitcoin-multicast-test  | multicast-test  |
| bitcoin-shard-common    | shard-common    |
| shard-proxy     | shard-proxy     |
| bitcoin-shard-listener  | shard-listener  |
| bitcoin-retry-endpoint  | retry-endpoint  |
| bitcoin-subtx-generator | subtx-generator |

### `lightwebinc` (org) — 11 repos, transfer SUCCEEDED, rename still needed

| Current name                 | New name             |
| ---------------------------- | -------------------- |
| bitcoin-multicast-kube-infra | multicast-kube-infra |
| bitcoin-shard-manifest       | shard-manifest       |
| bitcoin-shard-manifest-helm  | shard-manifest-helm  |
| bitcoin-manifest             | manifest-infra       |
| bitcoin-retry-endpoint-helm  | retry-endpoint-helm  |
| bitcoin-shard-listener-helm  | shard-listener-helm  |
| bitcoin-subtx-generator-helm | subtx-generator-helm |
| shard-proxy-helm     | shard-proxy-helm     |
| bitcoin-retransmission       | retransmission-infra |
| 10gb-direct-testing          | (no change)          |
| pay-per-flow                 | (no change)          |

---

## Phase 0 — `gh` Auth Setup (Prerequisite) — DONE

Account topology:
- `lightweb-inc` — new account; owns the `lightwebinc` organization AND owns the
  9 personal repos that need transfer. Universal admin everywhere we need it.
- `jefflightweb` — old account; team member of `lightwebinc` with admin on at
  least one org repo (verified). Not used by this plan.

Verified state:

```
✓ lightweb-inc   — scopes: admin:public_key, gist, read:org, repo  (admin:true on lightweb-inc/ingress-infra AND on lightwebinc/* as org owner)
✓ jefflightweb   — scopes: admin:public_key, gist, read:org, repo, write:packages
```

Use `gh auth switch --user lightweb-inc` once at the start of Phase 1; it
satisfies all three steps (no mid-flow switch needed).

---

## Phase 1 — GitHub Operations

### Step 1: Rename 9 repos at `lightweb-inc` to final names, then transfer to `lightwebinc`

The `bitcoin-*` names are retired at `lightwebinc` (artifact of the account rename), so
transferring with the old names would fail. Rename to final names at `lightweb-inc` first,
then transfer. `gh repo transfer` does not exist — transfer uses the REST API.

```bash
# Switch to lightweb-inc account owner first
gh auth switch --user lightweb-inc

gh api -X PATCH repos/lightweb-inc/ingress-infra         -f name=ingress-infra
sleep 5
gh api -X POST  repos/lightweb-inc/ingress-infra/transfer  -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-listener        -f name=listener-infra
sleep 5
gh api -X POST  repos/lightweb-inc/listener-infra/transfer -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-multicast       -f name=bsv-multicast
sleep 5
gh api -X POST  repos/lightweb-inc/bsv-multicast/transfer  -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-multicast-test  -f name=multicast-test
sleep 5
gh api -X POST  repos/lightweb-inc/multicast-test/transfer -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-shard-common    -f name=shard-common
sleep 5
gh api -X POST  repos/lightweb-inc/shard-common/transfer   -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/shard-proxy     -f name=shard-proxy
sleep 5
gh api -X POST  repos/lightweb-inc/shard-proxy/transfer    -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-shard-listener  -f name=shard-listener
sleep 5
gh api -X POST  repos/lightweb-inc/shard-listener/transfer -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-retry-endpoint  -f name=retry-endpoint
sleep 5
gh api -X POST  repos/lightweb-inc/retry-endpoint/transfer -f new_owner=lightwebinc
sleep 5

gh api -X PATCH repos/lightweb-inc/bitcoin-subtx-generator -f name=subtx-generator
sleep 5
gh api -X POST  repos/lightweb-inc/subtx-generator/transfer -f new_owner=lightwebinc
sleep 5
```

### Step 2: Rename 9 `bitcoin-*` repos already at `lightwebinc` org

```bash
# Still on lightweb-inc (org owner — has admin on every org repo)

gh repo rename multicast-kube-infra  -R lightwebinc/bitcoin-multicast-kube-infra --yes
sleep 1
gh repo rename shard-manifest        -R lightwebinc/bitcoin-shard-manifest        --yes
sleep 1
gh  repo rename shard-manifest-helm   -R lightwebinc/bitcoin-shard-manifest-helm   --yes
sleep 1
gh repo rename manifest-infra        -R lightwebinc/bitcoin-manifest              --yes
sleep 1
gh repo rename retry-endpoint-helm   -R lightwebinc/bitcoin-retry-endpoint-helm   --yes
sleep 1
gh repo rename shard-listener-helm   -R lightwebinc/bitcoin-shard-listener-helm   --yes
sleep 1
gh repo rename subtx-generator-helm  -R lightwebinc/bitcoin-subtx-generator-helm  --yes
sleep 1
gh repo rename shard-proxy-helm      -R lightwebinc/shard-proxy-helm      --yes
sleep 1
gh repo rename retransmission-infra  -R lightwebinc/bitcoin-retransmission        --yes
```

### Step 3: Update all local git remotes

```bash
git -C ~/repo/ingress-infra           remote set-url origin git@github.com:lightwebinc/ingress-infra.git
git -C ~/repo/bitcoin-listener          remote set-url origin git@github.com:lightwebinc/listener-infra.git
git -C ~/repo/bitcoin-multicast         remote set-url origin git@github.com:lightwebinc/bsv-multicast.git
git -C ~/repo/bitcoin-multicast-test    remote set-url origin git@github.com:lightwebinc/multicast-test.git
git -C ~/repo/bitcoin-shard-common      remote set-url origin git@github.com:lightwebinc/shard-common.git
git -C ~/repo/shard-proxy       remote set-url origin git@github.com:lightwebinc/shard-proxy.git
git -C ~/repo/bitcoin-shard-listener    remote set-url origin git@github.com:lightwebinc/shard-listener.git
git -C ~/repo/bitcoin-retry-endpoint    remote set-url origin git@github.com:lightwebinc/retry-endpoint.git
git -C ~/repo/bitcoin-subtx-generator   remote set-url origin git@github.com:lightwebinc/subtx-generator.git
git -C ~/repo/bitcoin-shard-manifest      remote set-url origin git@github.com:lightwebinc/shard-manifest.git
git -C ~/repo/bitcoin-shard-manifest-helm remote set-url origin git@github.com:lightwebinc/shard-manifest-helm.git
git -C ~/repo/bitcoin-manifest            remote set-url origin git@github.com:lightwebinc/manifest-infra.git
git -C ~/repo/bitcoin-retry-endpoint-helm remote set-url origin git@github.com:lightwebinc/retry-endpoint-helm.git
git -C ~/repo/bitcoin-shard-listener-helm remote set-url origin git@github.com:lightwebinc/shard-listener-helm.git
git -C ~/repo/bitcoin-subtx-generator-helm remote set-url origin git@github.com:lightwebinc/subtx-generator-helm.git
git -C ~/repo/shard-proxy-helm    remote set-url origin git@github.com:lightwebinc/shard-proxy-helm.git
git -C ~/repo/bitcoin-retransmission      remote set-url origin git@github.com:lightwebinc/retransmission-infra.git
git -C ~/repo/bitcoin-multicast-kube-infra remote set-url origin git@github.com:lightwebinc/multicast-kube-infra.git
```

---

## Phase 2 — Code Changes

Execution order: **shard-common first** (5 repos depend on it), then everything
else in parallel. Every section below also requires the per-repo follow-ups in
[2.7 — Per-repo Documentation & Badges](#27--per-repo-documentation--badges) and
[2.8 — Cross-repo Scripts and Docs](#28--cross-repo-scripts-and-docs).

### 2.1 — `bitcoin-shard-common` (blocker)

| File            | Change                                                                                       |
| --------------- | -------------------------------------------------------------------------------------------- |
| `go.mod`        | `module github.com/lightwebinc/bitcoin-shard-common` → `github.com/lightwebinc/shard-common` |
| All `.go` files | replace all `github.com/lightwebinc/bitcoin-shard-common` import paths                       |

Run `go mod tidy`, commit, push, **tag new release** (e.g. `v0.10.0`).

### 2.2 — Five shard-common dependents (parallel, after 2.1 is tagged)

#### `shard-proxy`

| File                       | Change                                                                                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `go.mod`                   | `module` → `github.com/lightwebinc/shard-proxy`; update `require` shard-common to new path + new tag                                                            |
| `ci/go.mod`                | `module` → `github.com/lightwebinc/shard-proxy/ci`                                                                                                              |
| All `.go` files            | replace import paths                                                                                                                                            |
| `Dockerfile`               | ldflags `-X github.com/lightwebinc/shard-proxy/...`; `-o /out/shard-proxy`; `COPY /out/shard-proxy`; `ENTRYPOINT ["/usr/local/bin/shard-proxy"]`                |
| `test/Dockerfile.e2e`      | `-o /shard-proxy` → `-o /shard-proxy`; `COPY` and `/usr/local/bin/` paths                                                                               |
| `test/run-e2e.sh`          | any hardcoded binary names                                                                                                                                      |
| `.github/workflows/ci.yml` | `repository: lightwebinc/shard-common`; `path: shard-common`; `path: shard-proxy`; `go mod edit -replace ...=../shard-common`; `working-directory: shard-proxy` |

#### `bitcoin-shard-listener`

| File                       | Change                                                                                            |
| -------------------------- | ------------------------------------------------------------------------------------------------- |
| `go.mod`                   | `module` → `github.com/lightwebinc/shard-listener`; update require                                |
| `ci/go.mod`                | `module` → `github.com/lightwebinc/shard-listener/ci`                                             |
| All `.go` files            | replace import paths                                                                              |
| `Dockerfile`               | ldflags, binary name → `shard-listener`                                                           |
| `test/Dockerfile.e2e`      | binary name and `dockerfile:` path (`bitcoin-shard-listener/test/...` → `shard-listener/test/...`) |
| `test/docker-compose.yml`  | service name `bsl-e2e`; `dockerfile: bitcoin-shard-listener/...` path                             |
| `test/run-e2e.sh`          | hardcoded binary names                                                                            |
| `.github/workflows/ci.yml` | shard-common and shard-proxy refs, paths, working dirs                                            |

#### `bitcoin-retry-endpoint`

| File                       | Change                                                             |
| -------------------------- | ------------------------------------------------------------------ |
| `go.mod`                   | `module` → `github.com/lightwebinc/retry-endpoint`; update require |
| `ci/go.mod`                | `module` → `github.com/lightwebinc/retry-endpoint/ci`              |
| All `.go` files            | replace import paths                                               |
| `Dockerfile`               | ldflags, binary name → `retry-endpoint`                            |
| `.github/workflows/ci.yml` | shard-common ref, paths                                            |

#### `bitcoin-subtx-generator`

| File                       | Change                                                              |
| -------------------------- | ------------------------------------------------------------------- |
| `go.mod`                   | `module` → `github.com/lightwebinc/subtx-generator`; update require |
| `ci/go.mod`                | `module` → `github.com/lightwebinc/subtx-generator/ci`              |
| All `.go` files            | replace import paths                                                |
| `.github/workflows/ci.yml` | shard-common ref, paths                                             |

#### `bitcoin-shard-manifest`

| File                       | Change                                                                             |
| -------------------------- | ---------------------------------------------------------------------------------- |
| `go.mod`                   | `module` → `github.com/lightwebinc/shard-manifest`; update require                 |
| `ci/go.mod`                | `module` → `github.com/lightwebinc/shard-manifest/ci`                              |
| All `.go` files            | replace import paths                                                               |
| `Dockerfile`               | ldflags for both binaries; daemon binary → `shard-manifest`; `manifest-emit` stays |
| `.github/workflows/ci.yml` | shard-common ref, paths                                                            |

### 2.3 — `bitcoin-multicast-test`

Does NOT have `shard-common` in go.mod `require` — only its self-module path
changes. But `harness/build/build.go` carries the shard-common module path as
a **string constant** and looks up the local checkout by directory name; both
must be updated. Scenario tests, vm-lab ansible inventories, and shell scripts
also carry hardcoded names.

| File                                       | Change                                                                                                  |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `go.mod`                                   | `module` → `github.com/lightwebinc/multicast-test`                                                      |
| `harness/build/build.go`                   | `commonModule = "github.com/lightwebinc/bitcoin-shard-common"` → new path; `"bitcoin-shard-common"` dir-name fallback → new local dir name; error/comment refs |
| `harness/driver/docker/docker.go`          | any `bitcoin-*` image/container names                                                                   |
| `harness/env/env.go`                       | repo path / module refs                                                                                 |
| `harness/scenarios/*_test.go` (35+ files)  | any binary names, image refs, repo path lookups                                                         |
| `vm-lab/ansible/ingress-hosts.yml`         | repo URL comment → `lightwebinc/ingress-infra`; path refs `~/repo/ingress-infra/ansible`; binary `/tmp/shard-proxy` |
| `vm-lab/ansible/listener-hosts.yml`        | repo URL comment → `lightwebinc/listener-infra`; path refs; binary `/tmp/shard-listener`                |
| `vm-lab/ansible/retry-hosts.yml`           | repo URL comment → `lightwebinc/retransmission-infra`; path refs; binary `/tmp/retry-endpoint`          |
| `vm-lab/ansible/{os-update.yml,run-deploy.sh}` | audit                                                                                                |
| `vm-lab/{build,deploy}.sh`                 | hardcoded repo dirs/binaries                                                                            |
| `vm-lab/docs/bitcoin-shard-{proxy,listener}.md` | rename files to `shard-{proxy,listener}.md` + update content                                       |
| `vm-lab/docs/grafana/bitcoin-*.json` (3 files) | rename files (`bitcoin-retry-endpoint.json` → `retry-endpoint.json`, etc.) + dashboard titles      |
| `vm-lab/docs/prometheus/prometheus.yml`    | scrape job names / targets referencing `bitcoin-*`                                                      |
| `vm-lab/lab/*.sh`, `vm-lab/scenarios/**/*.{sh,md}` | hardcoded binary names and repo dirs                                                            |
| `README.md`, `vm-lab/README.md`, `vm-lab/docs/*.md` | repo URLs, sibling repo names                                                                  |

### 2.4 — Ansible-only repos (parallel, no ordering constraint)

Each of these has a role directory named after the binary (e.g.
`ansible/roles/shard-proxy/`). Rename the directory AND update every
`site.yml` and playbook that lists `role: bitcoin-*`. Also audit role
`tasks/main.yml` for comments and task names that mention the binary.

#### `ingress-infra` (→ `ingress-infra`)

| File                                          | Change                                                                |
| --------------------------------------------- | --------------------------------------------------------------------- |
| `ansible/group_vars/all.yml`                  | `proxy_repo` → `https://github.com/lightwebinc/shard-proxy.git`; `proxy_install_dir` → `/opt/shard-proxy` |
| `ansible/roles/shard-proxy/`          | rename directory → `ansible/roles/shard-proxy/`                       |
| `ansible/site.yml`                            | `role: shard-proxy` → `role: shard-proxy`                     |
| `ansible/roles/shard-proxy/tasks/main.yml`    | task name comments referencing `shard-proxy`                  |
| `ansible/roles/shard-proxy/handlers/main.yml` | same                                                                  |
| `terraform/modules/ingress-node/variables.tf` | audit for `bitcoin-*` defaults                                        |
| `terraform/examples/aws-ec2/{main,variables}.tf` | same                                                               |

#### `bitcoin-listener` (→ `listener-infra`)

| File                                              | Change                                                                  |
| ------------------------------------------------- | ----------------------------------------------------------------------- |
| `ansible/group_vars/all.yml`                      | `listener_repo` → `https://github.com/lightwebinc/shard-listener.git`; `listener_install_dir` → `/opt/shard-listener` |
| `ansible/roles/bitcoin-shard-listener/`           | rename directory → `ansible/roles/shard-listener/`                      |
| `ansible/site.yml`                                | `role: bitcoin-shard-listener` → `role: shard-listener`                 |
| `ansible/roles/shard-listener/{tasks,handlers}/main.yml` | comments/task names                                               |
| `terraform/modules/listener-node/variables.tf`    | audit                                                                   |
| `terraform/examples/aws-ec2/{main,variables}.tf`  | same                                                                    |

#### `bitcoin-manifest` (→ `manifest-infra`)

| File                                              | Change                                                                  |
| ------------------------------------------------- | ----------------------------------------------------------------------- |
| `ansible/group_vars/all.yml`                      | `manifest_repo` → `https://github.com/lightwebinc/shard-manifest.git`; `manifest_install_dir` → `/opt/shard-manifest` |
| `ansible/roles/bitcoin-shard-manifest/`           | rename directory → `ansible/roles/shard-manifest/`                      |
| `ansible/site.yml`                                | `role: bitcoin-shard-manifest` → `role: shard-manifest`                 |
| `ansible/roles/shard-manifest/{tasks,handlers}/main.yml` | comments/task names                                               |

#### `bitcoin-retransmission` (→ `retransmission-infra`)

| File                                              | Change                                                                  |
| ------------------------------------------------- | ----------------------------------------------------------------------- |
| `ansible/group_vars/all.yml`                      | `retry_repo` → `https://github.com/lightwebinc/retry-endpoint.git`; `retry_install_dir` → `/opt/retry-endpoint` |
| `ansible/roles/bitcoin-retry-endpoint/`           | rename directory → `ansible/roles/retry-endpoint/`                      |
| `ansible/site.yml`                                | `role: bitcoin-retry-endpoint` → `role: retry-endpoint`                 |
| `ansible/roles/retry-endpoint/{tasks,handlers}/main.yml` | comments/task names                                               |
| `ansible/inventory/hosts.example.yml`             | audit for `bitcoin-*` paths                                             |
| `terraform/modules/retry-endpoint-node/variables.tf` | audit                                                                |
| `terraform/examples/aws-ec2/{main,variables}.tf`  | same                                                                    |

### 2.5 — Helm repos (5 repos, all already in `lightwebinc` org, parallel)

Same file set for each chart. **Critical:** the chart's `_helpers.tpl` defines
template functions keyed by the chart name (e.g. `{{- define "shard-proxy.fullname" -}}`),
and every other template file calls them via `{{ include "shard-proxy.fullname" . }}`.
Renaming the chart in `Chart.yaml` without renaming these helpers breaks the chart.
Do a chart-wide `bitcoin-<name>` → `<name>` replace across all `templates/*.{tpl,yaml}`.

| File                       | Field / Pattern                                  | Change                                                    |
| -------------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| `Chart.yaml`               | `name`                                           | drop `bitcoin-` prefix                                    |
| `Chart.yaml`               | `home`, `sources`, `icon`                        | `lightwebinc/bitcoin-*` → `lightwebinc/*`                 |
| `Chart.yaml`               | `maintainers[].name`/`url`                       | `lightwebinc` → keep (org slug unchanged)                 |
| `values.yaml`              | `image.repository`                               | `ghcr.io/lightwebinc/bitcoin-*` → `ghcr.io/lightwebinc/*` |
| `values.schema.json`       | any image-default/regex referencing `bitcoin-*`  | update                                                    |
| `cr.yaml`                  | `git-repo`                                       | drop `bitcoin-` prefix                                    |
| `templates/_helpers.tpl`   | `define "bitcoin-<name>.{name,fullname,chart,labels,selectorLabels}"` | drop `bitcoin-` prefix in every define |
| `templates/_helpers.tpl`   | `app.kubernetes.io/part-of: bitcoin-multicast`   | → `app.kubernetes.io/part-of: bsv-multicast`              |
| `templates/*.yaml`         | every `{{ include "bitcoin-<name>.<fn>" . }}`    | drop `bitcoin-` prefix to match new helper names          |
| `templates/NOTES.txt`      | `bitcoin-<name>.fullname` includes + URL refs    | update both                                               |
| `README.md`                | install/upgrade examples, links                  | update chart name and repo URLs                           |
| `.github/workflows/release.yml` | (image push paths if any)                   | usually none here; `oci://ghcr.io/lightwebinc/charts` stays |

Repos: `shard-proxy-helm`, `shard-listener-helm`, `retry-endpoint-helm`,
`subtx-generator-helm`, `shard-manifest-helm`

### 2.6 — Docs/manifests-only repos

#### `bitcoin-multicast` (→ `bsv-multicast`)
Pure documentation but riddled with repo URLs.

| File pattern                          | Change                                              |
| ------------------------------------- | --------------------------------------------------- |
| `README.md`                           | repo URL/title; sibling repo links                  |
| `DESIGN.md`                           | sibling repo links                                  |
| `docs/brc-*.md` (12 files)            | references to `bitcoin-shard-*` repo URLs           |
| `containerization/*.md` (~10 files)   | references to component repos and images           |

#### `bitcoin-multicast-kube-infra` (→ `multicast-kube-infra`)

| File                          | Change                                                    |
| ----------------------------- | --------------------------------------------------------- |
| `README.md`                   | repo URLs + image refs                                    |
| `Makefile`                    | any `ghcr.io/lightwebinc/bitcoin-*` image refs            |
| `scripts/verify.sh`           | any hardcoded `bitcoin-*` names                           |
| `docs/{architecture,networking,quickstart-k0s,troubleshooting}.md` | references |
| `argocd/`, `apps/`, `platform/`, `distributions/` | audit YAML manifests for image repos  |

### 2.7 — Per-repo Documentation & Badges

Every Go service repo has a README with shield.io badges and links pointing at
`github.com/lightwebinc/bitcoin-*`. Update these along with each repo's docs.

For each of `shard-common`, `shard-proxy`, `shard-listener`, `retry-endpoint`,
`subtx-generator`, `shard-manifest`, `multicast-test`:

| File pattern                     | Change                                                            |
| -------------------------------- | ----------------------------------------------------------------- |
| `README.md` (top heading + badges) | repo title; CI/CodeQL/Release/pkg.go.dev/goreportcard URLs        |
| `README.md` body                  | install snippets, `go get`, `git clone` URLs                      |
| `docs/architecture.md`           | self-name references + sibling repo names                         |
| `docs/configuration.md`          | env var names that include the binary name; example commands     |
| `docs/security.md` (where present)| references                                                       |
| `Makefile`                       | `BINARY := bitcoin-<name>` style vars; image tag templates        |

For each Ansible/Terraform repo (`ingress-infra`, `listener-infra`,
`manifest-infra`, `retransmission-infra`):

| File pattern                     | Change                                                            |
| -------------------------------- | ----------------------------------------------------------------- |
| `README.md`                      | repo title + clone URL                                            |
| `docs/{architecture,ansible,bgp,networking,security,terraform}.md` | references to self and component repos |
| `docs/os/{freebsd-14,ubuntu-24.04}.md` | install paths, git clone URLs                                |

For each Helm repo (`shard-proxy-helm`, `shard-listener-helm`,
`retry-endpoint-helm`, `subtx-generator-helm`, `shard-manifest-helm`):

| File           | Change                                                          |
| -------------- | --------------------------------------------------------------- |
| `README.md`    | `helm install` examples (chart name, repo URL)                  |

### 2.8 — Cross-repo Scripts and Docs

These live outside any single repo and reference multiple `bitcoin-*` names.

| Path                                              | Change                                                              |
| ------------------------------------------------- | ------------------------------------------------------------------- |
| `~/repo/bump_all_tags.sh`                         | hardcoded list `("bitcoin-retry-endpoint" ...)` → new names         |
| `~/repo/update-shard-common.sh`                   | `MODULE="github.com/lightwebinc/bitcoin-shard-common"`; `COMMON_DIR`; `DEPENDENT_DIRS` array |
| `~/repo/multicast-skills/architecture.md`         | repo list, dep graph, sibling refs                                  |
| `~/repo/multicast-skills/build-deploy.md`         | clone URLs, image refs                                              |
| `~/repo/multicast-skills/conventions.md`          | module path examples                                                |
| `~/repo/multicast-skills/testing.md`              | repo names in test instructions                                     |
| `~/repo/multicast-skills/testing-environments.md` | repo names                                                          |
| `~/repo/multicast-skills/protocol.md`             | repo refs (if any)                                                  |
| `~/repo/10gb-direct-testing/*.sh`                 | hardcoded `shard-proxy` etc. binary names/paths             |
| `~/repo/10gb-direct-testing/README.md`            | references                                                          |
| `~/repo/.windsurf/workflows/{build-and-push,run-scenario}.md` | hardcoded repo dirs and image refs                       |
| `~/repo/pay-per-flow/DESIGN.md`                   | sibling repo references                                             |
| `~/repo/.claude/settings.local.json`              | allowlists may reference `bitcoin-*` paths                          |
| Each repo's `.claude/settings.local.json`         | same                                                                |

---

## Phase 3 — GHCR Image Migration

After Phase 2 CI completes:

- New images publish to `ghcr.io/lightwebinc/shard-proxy`,
  `ghcr.io/lightwebinc/shard-listener`, etc.
- Old images at `ghcr.io/lightwebinc/bitcoin-*` remain until manually deleted.
- Helm charts (Phase 2.5) already reference new paths.
- Running deployments need a `helm upgrade` cycle to pull new image repos.

---

## Deferred / Out of Scope

- **System users** (`bitcoin-proxy`, `bitcoin-listener`, etc.) — OS-level users
  on deployed nodes; rename via a separate Ansible pass or leave as-is
- **K8s namespace** `bitcoin-mcast` — deployed resource; rename requires
  coordinated namespace migration separate from this plan
- **Local directory renames** (`~/repo/bitcoin-*` → `~/repo/<new-name>`) — not
  cosmetic if you keep the scripts in 2.8 (`update-shard-common.sh`,
  `bump_all_tags.sh`, `.windsurf/workflows/*`) since they hardcode dirs.
  Recommendation: rename local dirs in the same pass as 2.8 so the scripts
  point at one canonical layout.

---

## Execution Order Summary

```
Phase 0:  gh auth login: add lightweb-inc owner + lightwebinc org owner
          verify admin:true on sample repos from each

Phase 1:  lightweb-inc: rename 9 repos to final names → transfer to lightwebinc
          lightwebinc: rename 9 existing org repos
          update local git remotes
          └─ no code changes yet; redirects handle old clone URLs

Phase 2:  shard-common: update module path → tag release
          └─ unblocks code/import updates in:
             shard-proxy               ─┐
             shard-listener            ─┤
             retry-endpoint            ─┼─ Go service repos (parallel)
             subtx-generator           ─┤
             shard-manifest            ─┘
             multicast-test            ─┐
             ingress-infra (ansible)   ─┤
             listener-infra (ansible)  ─┤
             manifest-infra (ansible)  ─┼─ infra + doc repos (parallel)
             retransmission-infra      ─┤
             5 helm repos              ─┤
             bsv-multicast (docs)      ─┤
             multicast-kube-infra      ─┘
          Then per-repo follow-ups:
             2.7 READMEs/badges/docs (every repo)
             2.8 cross-repo scripts: bump_all_tags.sh, update-shard-common.sh,
                 multicast-skills/, 10gb-direct-testing/, .windsurf/

Phase 3:  helm upgrade cycle on deployed clusters → pull new GHCR image paths
```
