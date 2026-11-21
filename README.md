# hello-world

Minimal Go HTTP service used to exercise the `koorikla/platform` stack (Cluster API + Argo CD + OpenChoreo 1.3.3) end to end.

## Endpoints

- `GET /` — returns JSON `{"message", "hostname", "version"}`
- `GET /healthz` — returns `200 OK` with `{"status": "ok"}`
- `GET /version` — returns `200 OK` with `{"version": "<git sha>"}` (the short commit SHA of the build; promotion checks compare it per environment)
- `GET /db` — runs `SELECT 1` on the database: `200 OK` with `{"status": "ok"}`, `503` with `{"status": "error"}` (cause in the log) or `{"status": "not configured"}` when no database is wired. Not part of `/healthz`: a database outage neither restarts nor unroutes the app.

Configuration:

- `PORT` (default `8080`)
- `GREETING` (default `hello`)
- `DATABASE_URL` or libpq's `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD` (set by the platform from the `orders-db` Resource, see below; optional)
- `main.version` (injected at build time via `-ldflags "-X main.version=<git sha>"`, default `dev`)

## Local run & verification

```bash
mise exec -- go test -race -v ./...
mise exec -- go run .
```

Build the container image locally:

```bash
docker build --build-arg VERSION=dev -t hello-world:dev .
```

## Build flow

**Default: OpenChoreo workflow plane.** `deploy/openchoreo/component.yaml` names the platform's ClusterWorkflow `go`.
Every commit to `main` is built automatically (`autoBuild: true`): the platform's auto-build trigger creates one
WorkflowRun `hello-world-<sha[:12]>` per commit (local installation: a poll, about a minute after the merge; cloud: a
signed GitHub App webhook; koorikla/platform docs/build-plane.md "Auto-build"). Backstage → hello-world → **Build**
starts one on demand. The run uses the platform's workflow plane, with the same
[koorikla/platform `ci/`](https://github.com/koorikla/platform/tree/main/ci) scripts as the GitHub path. Run pods admit only digest-pinned images, so
a `checks.image` override must be `image@sha256:<digest>`:

1. checkout → `scripts/go/check.sh` (`go vet`, golangci-lint if present, `go test -race`, govulncheck) and a Trivy
   lockfile scan;
2. rootless BuildKit builds `linux/amd64` + `linux/arm64` from the digest-pinned base images (pulled through the
   Harbor proxy caches, upstream as the fallback) and pushes
   `harbor.openchoreo.localhost:9443/apps/demo/hello-world:<short-sha>` (platform registry) with SLSA provenance;
3. Trivy scan per platform (fails on CRITICAL), syft SBOM, cosign signature with the platform key + vuln and SBOM
   attestations;
4. commits `spec.container.image` (`<repo>:<short-sha>@sha256:…`) and `spec.source.commit` into
   `deploy/openchoreo/workload.yaml` with `[skip ci]`; Argo CD syncs it and `autoDeploy` releases it to development.
   The commit is authored and signed by the platform's GitHub App bot: each run gets an installation token for this
   repository only (no personal access token; koorikla/platform docs/build-plane.md "Git credentials"), so GitHub
   shows it as Verified.

**GitHub Actions** (`.github/workflows/ci.yaml`) runs pull-request checks only: one call to the platform's reusable
[`app-go.yaml`](https://github.com/koorikla/platform/blob/main/.github/workflows/app-go.yaml) with
`build_container: false` (go vet, golangci-lint, `go test -race`, govulncheck, CodeQL, and the OpenChoreo manifests in
`deploy/openchoreo/` against the schemas of the OpenChoreo version the platform runs). This repository contains no CI
code. Building and signing on GitHub Actions instead is a platform option for other apps (koorikla/platform
docs/build-plane.md); this example uses one build path.

## Files

`deploy/openchoreo/` holds the OpenChoreo 1.3.3 developer manifests for project `demo`:

- `component.yaml`: `Component/hello-world` referencing `ClusterComponentType` `deployment/service` with `autoDeploy: true`.
- `workload.yaml`: `Workload/hello-world-workload` defining the image (written by the build), `GREETING` env, `GREETING_SECRET` (from the SecretReference below), and external `HTTP` endpoint on port `8080`.
- `greeting-secret.yaml`: `SecretReference/hello-world-greeting`, a pointer to key `hello-world` / property `greeting` in the project's OpenBao mount `kv-tenant-demo`; the value is never in git. How to write, consume and rotate it: koorikla/platform docs/openchoreo.md "Application secrets (developer view)". CI fails if a `secretKeyRef` names a missing SecretReference or key.
- `release-bindings/development.yaml`: `ReleaseBinding/hello-world-development` (omits `spec.releaseName`; `autoDeploy: true` causes the OpenChoreo controller to cut an immutable `ComponentRelease` named `hello-world-<hash>` and update this binding automatically, while Argo CD ignores `/spec/releaseName`).
- `release-bindings/staging.yaml` & `production.yaml`: `ReleaseBinding` objects for `staging` (on `dev1`) and `production` (on `prod1`), dormant until promoted.
- `orders-db.yaml`: `Resource/orders-db` of the platform's `ClusterResourceType` `postgresql` (koorikla/platform docs/data-services.md "PostgreSQL from OpenChoreo"). `workload.yaml` `dependencies.resources` wires its outputs into the container: `PGHOST`/`PGPORT`/`PGDATABASE` as values, `PGUSER`/`PGPASSWORD`/`DATABASE_URL` as `secretKeyRef`s to the database's Secret `<xr>-app` in the cell namespace (never in git, never on the control plane).
- `release-bindings/orders-db-development.yaml`: `ResourceReleaseBinding/orders-db-development` runs the database in `development` only. It carries the annotation `platform.koorikla.io/resource-release: latest` and no `spec.resourceRelease`: OpenChoreo 1.3.3 never sets that pin itself, so the platform's auto-build pass keeps it on `orders-db`'s latest release on every pass, and a parameter or platform type change rolls into development by itself (koorikla/platform docs/promotion.md "Resources"). PR previews get their own `orders-db` binding from the platform, nothing here. A production binding must instead pin `spec.resourceRelease: orders-db-<hash>` explicitly (copied from `kubectl get resource orders-db -o jsonpath='{.status.latestRelease.name}'`, like a `ComponentRelease` name); the platform never moves production pins. There is no staging or production binding yet (production cells cannot run CloudNativePG before koorikla/platform#327): a `ComponentRelease` with this dependency waits (`ResourceDependenciesPending`) in any environment without an `orders-db-<env>` binding, so add `orders-db-staging.yaml` / `orders-db-production.yaml` (production: `highAvailability: true`, `backup.enabled: true`) before promoting it. CI fails if a dependency names a missing Resource or one without a development binding, if a non-production binding has neither the `latest` annotation nor a `<resource>-<hash>` pin, or if a production binding uses the annotation or lacks an explicit pin.

### Promotion

To promote to `staging` or `production`, open a PR setting `spec.releaseName` in `deploy/openchoreo/release-bindings/staging.yaml` (or `production.yaml`) to the `ComponentRelease` name (`hello-world-<hash>`) currently running in the preceding environment.

Promoting to `production` is automatic up to the PR: once the `staging` rollout succeeds, the platform's promotion gate opens exactly one PR `promote/hello-world-production` (koorikla/platform docs/promotion.md); merge it to deploy.

### Platform-side registration

The platform repository (`koorikla/platform`) registers this app in `installations/local/tenants/demo.yaml`, which creates the `demo` OpenChoreo `Project`, an isolated `AppProject` (`demo`) whitelisting the OpenChoreo developer kinds (`Component`, `Workload`, `ReleaseBinding`, `SecretReference`, ...) in namespace `default`, and an Argo CD `Application` syncing `deploy/openchoreo`.
