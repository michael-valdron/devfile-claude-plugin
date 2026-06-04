---
name: building-devfile-registry
description: Guides creation, build, deployment, and verification of a custom OCI devfile registry. Covers registry repo structure (stacks/, stack.yaml, extraDevfileEntries.yaml), index image build via build-tools, Helm/Operator deployment, and integration testing. Use when the user asks to build, create, deploy, or customize a devfile registry, devfile stacks, index.json, or devfile-index container images.
---

# Building a Devfile Registry

End-to-end workflow for a custom OCI devfile registry. Official docs: https://devfile.io/docs/2.3.0/building-a-custom-devfile-registry

## Architecture

A devfile registry has two runtime components:

| Component | Role |
|-----------|------|
| **Index server** | Serves REST API, `index.json`, registry viewer |
| **OCI registry** | Stores stack artifacts as OCI images |

The **devfile index image** (built from your registry repo) contains `index.json` + packaged stacks. Deploy it with the [Registry Operator](https://github.com/devfile/registry-operator) or the [Helm chart](https://github.com/devfile/registry-support/blob/main/deploy/chart/devfile-registry/README.md).

Build tools, the Helm chart, and integration tests live in the separate **[registry-support](https://github.com/devfile/registry-support)** repository under `build-tools/`, `index/generator/`, `deploy/chart/`, and `tests/`. Do not assume the current workspace is a clone of registry-support; clone it when a step needs those tools, or use an existing clone path as `$REGISTRY_SUPPORT`.

## Task Progress

Copy and track:

```
- [ ] Step 1: Create or update registry repository
- [ ] Step 2: Validate stack structure
- [ ] Step 3: Build devfile index image
- [ ] Step 4: Push image to container registry
- [ ] Step 5: Deploy to cluster
- [ ] Step 6: Verify registry is healthy
```

---

## Step 1: Create or update registry repository

Fork or create a Git repo modeled on https://github.com/devfile/registry.

**Required layout:**

```
<registry-repo>/
├── stacks/
│   └── <stack-name>/
│       ├── stack.yaml          # optional; required for multi-version stacks
│       ├── devfile.yaml        # single-version stack
│       └── <version>/          # multi-version stack
│           └── devfile.yaml
├── extraDevfileEntries.yaml    # optional external samples/stacks
└── .ci/                        # optional CI build scripts
```

**Multi-version stack** — add `stack.yaml` at the stack root:

```yaml
name: go
description: Go runtime stack
displayName: Go Runtime
icon: https://example.com/icon.svg
versions:
  - version: 1.0.0
    default: true   # exactly one default required
  - version: 2.0.0
```

**External samples** — root-level `extraDevfileEntries.yaml` references Git repos to cache at build time. See https://github.com/devfile/registry-support/blob/main/tests/registry/extraDevfileEntries.yaml for examples.

Each stack directory must contain at least one `devfile.yaml`. Additional files (Dockerfiles, K8s manifests, VSX plugins) are tar-archived automatically during build.

For field-level schema details, see [reference.md](reference.md).

---

## Step 2: Validate stack structure

Before building, confirm:

- [ ] `stacks/` exists at repo root
- [ ] Each stack has at least one `devfile.yaml`
- [ ] Multi-version stacks have `stack.yaml` with exactly one `default: true` version
- [ ] `devfile.yaml` metadata `name`/`version` match directory and `stack.yaml` entries
- [ ] `extraDevfileEntries.yaml` Git remotes are reachable (if present)

The index generator validates structure during build and fails on errors.

---

## Step 3: Build devfile index image

### Prerequisites

- Go 1.24+
- Docker 17.05+ (or Podman with `export USE_PODMAN=true`)
- Git
- [yq](https://github.com/mikefarah/yq) 4.x

### Option A: Build from a registry repo (most common)

Clone [registry-support](https://github.com/devfile/registry-support) (or set `$REGISTRY_SUPPORT` to an existing clone), then run from its root:

```bash
git clone https://github.com/devfile/registry-support.git "${REGISTRY_SUPPORT:-registry-support}"
cd "${REGISTRY_SUPPORT:-registry-support}"
bash build-tools/build_image.sh <path-to-registry-repo>
```

This runs `build-tools/build.sh`, generates `index.json`, and produces a `devfile-index` Docker image.

For offline/air-gapped builds (from the registry-support clone):

```bash
cd "${REGISTRY_SUPPORT:-registry-support}"
bash build-tools/build_image.sh <path-to-registry-repo> 1
```

### Option B: Build from within a registry repo

Add a multi-stage Dockerfile (see https://github.com/devfile/registry `.ci/Dockerfile`) or run:

```bash
bash .ci/build.sh
```

On Apple Silicon targeting amd64 clusters: `export PLATFORM_EV=linux/arm64`.

### Option C: Develop registry-support components locally

From a clone of [registry-support](https://github.com/devfile/registry-support), build all components with mock test data:

```bash
cd "${REGISTRY_SUPPORT:-registry-support}"
bash build_registry.sh              # linux/amd64
bash build_registry.sh linux/arm64  # other arch
```

Uses `tests/registry/` in that repo and produces `devfile-index:latest` via `.ci/Dockerfile`.

---

## Step 4: Push image

```bash
docker tag devfile-index <registry>/<user>/devfile-index:<tag>
docker push <registry>/<user>/devfile-index:<tag>
```

---

## Step 5: Deploy

### Helm (Kubernetes)

From a clone of [registry-support](https://github.com/devfile/registry-support):

```bash
cd "${REGISTRY_SUPPORT:-registry-support}"
helm install devfile-registry deploy/chart/devfile-registry \
  --set global.ingress.domain=<ingress-domain> \
  --set devfileIndex.image=<registry>/<user>/devfile-index \
  --set devfileIndex.tag=<tag>
```

OpenShift: add `--set global.isOpenShift=true` or use `deploy/helm-openshift-install.sh` from the same repo.

Headless (no viewer): `--set global.headless=true`.

Full chart options: [deploy/chart/devfile-registry/README.md](https://github.com/devfile/registry-support/blob/main/deploy/chart/devfile-registry/README.md).

### Operator (recommended for production)

Install https://github.com/devfile/registry-operator, then create a `DevfileRegistry` CR with `spec.devfileIndexImage` set to your pushed image.

---

## Step 6: Verify

### Quick health check

```bash
curl -s https://<registry-host>/v2/index | head
curl -s https://<registry-host>/devfiles/go/1.2.0
```

### Integration tests

From a clone of [registry-support](https://github.com/devfile/registry-support):

```bash
cd "${REGISTRY_SUPPORT:-registry-support}/tests/integration"
./docker-build.sh
docker run --env REGISTRY=https://<registry-host> \
  --env IS_TEST_REGISTRY=true \
  devfile-registry-integration
```

For a custom registry with non-standard stacks, omit `IS_TEST_REGISTRY=true`.

See [tests/integration/README.md](https://github.com/devfile/registry-support/blob/main/tests/integration/README.md).

### registry-library CLI

From a clone of [registry-support](https://github.com/devfile/registry-support):

```bash
cd "${REGISTRY_SUPPORT:-registry-support}/registry-library"
bash build.sh
./registry-library list https://<registry-host>
```

---

## Decision guide

| Goal | Path |
|------|------|
| Custom registry for my team | Fork [devfile/registry](https://github.com/devfile/registry) → Option A build (via [registry-support](https://github.com/devfile/registry-support)) → Helm/Operator deploy |
| Contribute stacks to community registry | PR to https://github.com/devfile/registry |
| Develop index server / build tools | Clone [registry-support](https://github.com/devfile/registry-support) → Option C (`build_registry.sh`) |

---

## Additional resources

- Registry repo layout & index schema: [reference.md](reference.md)
- [registry-support](https://github.com/devfile/registry-support) — build tools, index server, Helm chart, integration tests
- Build tools: [build-tools/README.md](https://github.com/devfile/registry-support/blob/main/build-tools/README.md)
- Index server API: [index/server/README.md](https://github.com/devfile/registry-support/blob/main/index/server/README.md)
- Deploy chart: [deploy/chart/devfile-registry/README.md](https://github.com/devfile/registry-support/blob/main/deploy/chart/devfile-registry/README.md)
- Integration tests: [tests/integration/README.md](https://github.com/devfile/registry-support/blob/main/tests/integration/README.md)
- Official docs: https://devfile.io/docs/2.3.0/building-a-custom-devfile-registry
