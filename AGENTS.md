# AGENTS.md

## Project

This repository is a polyglot monorepo orchestrated with `just`.

The filesystem is the source of truth. Avoid introducing registries, manifests, tags, or duplicated metadata when the same information can be derived from repository structure.

## Targets

A **target** is anything publishable or deployable.

A target is identified by a directory containing a `Justfile`.

The target ID is the directory's repository-relative path:

```text
js/services/primary/Justfile
→ js/services/primary

docker/guis/web/Justfile
→ docker/guis/web

tofu/aws/environments/production/Justfile
→ tofu/aws/environments/production
```

Target IDs must not be separately declared.

## Target Interface

Targets expose only lifecycle recipes they meaningfully implement.

The lifecycle vocabulary is:

```text
init
check
test
build
publish
deploy
clean
```

Absence means unsupported. Do not add empty placeholder recipes merely to satisfy the interface.

Broad semantics:

* `init` — prepare the target's local execution environment and dependencies.
* `check` — perform static validation.
* `test` — execute tests.
* `build` — produce or plan the target's artifact.
* `publish` — publish an artifact to an external artifact registry.
* `deploy` — reconcile/apply the target to its runtime environment.
* `clean` — remove local generated artifacts only.

`clean` must not destroy remote infrastructure, uninstall releases, or remove shared external state.

## Justfiles

Target `Justfile`s currently require:

```just
set lists
set minimum-version := '1.58.0'
set quiet
set unstable
```

Prefer native `just` features over shell machinery when they improve the model:

* recipe dependencies
* parameterized recipe dependencies
* named `[arg]` parameters
* `[cache]`
* `[private]`
* lists
* built-in functions
* imports/modules where appropriate

Recipes should use strict shell execution:

```just
recipe:
  #!/usr/bin/env bash
  set -euo pipefail
```

Do not add abstractions merely to remove repetition. Repetition across target `Justfile`s is acceptable when it keeps targets independently understandable and executable.

## Dependencies

Cross-target dependencies are declared using `deps`:

```just
deps := [
  'js/services/primary',
  'tofu/aws/environments/ops',
]
```

Dependencies are target IDs.

`deps` describes the cross-target DAG. Do not encode cross-target orchestration using direct shell calls to another target's `Justfile`.

Local relationships within one target should use native Just recipe dependencies:

```just
test: check

deploy: build

publish registry repo tag: (build registry repo tag)
```

Keep these two concepts separate:

```text
deps                 → relationships between targets
recipe dependencies  → relationships within a target
```

## Caching

Caching belongs to the target that understands its own inputs and outputs.

Example:

```just
[cache(
  inputs = [
    'src/**',
    'package.json',
    'tsconfig.json',
  ],
  outputs = ['dist/**'],
)]
build:
  #!/usr/bin/env bash
  set -euo pipefail
  pnpm run build
```

Do not cache external side effects such as deployments.

Do not cache OpenTofu plans based only on filesystem inputs: plans depend on remote infrastructure/provider state.

Docker builds should generally rely on Docker/BuildKit caching rather than Just caching.

## JavaScript

JavaScript targets use pnpm.

GUI targets use Qwik City SSR. Do not introduce CSR/S3/CloudFront GUI deployment architecture unless explicitly requested.

JS source targets produce application artifacts. They do not own container publishing or Kubernetes deployment.

Typical chain:

```text
js/guis/landing
→ docker/guis/landing
→ k8s/guis/landing

js/services/primary
→ docker/services/primary
→ k8s/services/primary
```

## Docker

Docker targets package artifacts produced by JS or other source targets.

Docker targets may `build` and `publish`.

Image identity should be passed explicitly:

```text
registry
repo
tag
```

Do not hard-code environment-specific ECR registry values into Docker targets.

ECR authentication may be implemented as a `[private]` recipe used by `publish`.

Do not implement `clean` by deleting Docker daemon images merely to satisfy the lifecycle interface.

## Kubernetes

Kubernetes targets use Helm.

Helm is intentionally used as the ontology/template layer rather than Kustomize.

Kubernetes targets generally represent Helm releases and may:

```text
init   → helm dependency build
check  → helm lint
build  → helm template
deploy → helm upgrade --install
clean  → remove local rendered manifest
```

Rendered manifests may use:

```text
.manifest.yaml
```

as their local artifact.

`clean` must not run `helm uninstall`.

Workloads should remain simple until requirements justify additional Kubernetes concepts. In particular, do not introduce StatefulSets, operators, GitOps systems, or additional orchestration layers preemptively.

## OpenTofu

OpenTofu targets describe infrastructure.

Use `.tfvars` files for variable sets and `.tfplan` as the build artifact.

Typical semantics:

```text
init   → tofu init
check  → tofu fmt -check + tofu validate
build  → tofu plan -var-file=... -out=.tfplan
deploy → tofu apply .tfplan
```

Do not treat `destroy` as `clean`.

Do not cache `tofu plan`; unchanged files do not imply unchanged remote infrastructure.

## Repository Structure

Current high-level ontology:

```text
docker/
  guis/
  services/

js/
  guis/
  services/
  tools/

just/

k8s/
  caches/
  databases/
  guis/
  ingresses/
  services/

py/

tofu/
  aws/
  cloudflare/
```

Preserve existing naming choices unless explicitly asked to change them.

## `just/`

`just/` is reserved for repository-level orchestration.

Do not move target-specific implementation into `just/`.

The intended orchestration model is:

```text
discover targets
→ read deps
→ validate DAG
→ determine execution plan
→ invoke target lifecycle recipes
```

The orchestrator should understand targets, dependencies, recipes, and execution ordering.

It should **not** contain domain knowledge about JavaScript, Docker, Kubernetes, Helm, AWS, Cloudflare, or OpenTofu.

Target implementations own that knowledge.

## Design Principles

Prefer:

```text
filesystem conventions > duplicated metadata
native Just features   > custom wrappers
explicit dependencies  > implicit ordering
small target contracts > universal abstractions
composition            > special cases
absence                 > meaningless placeholder recipes
```

Keep targets independently executable with `just -f <target>/Justfile ...`.

Before introducing a new abstraction, demonstrate the concrete repetition or problem it removes.