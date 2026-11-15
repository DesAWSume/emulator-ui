<p align="center">
  <img src="assets/brand/logo.svg" alt="Emulator UI logo" width="120">
</p>

<p align="center">
  <img src="assets/brand/banner.svg" alt="Emulator UI" width="560">
</p>

<h1 align="center">Emulator UI</h1>

<h3 align="center">A web console for AWS Emulator.</h3>

<p align="center">
  Browse and interact with the resources in a locally running
  <a href="https://github.com/aws/aws-emulator">AWS Emulator</a>, for all 436 services it supports.
</p>

<p align="center">
  <a href="https://github.com/DesAWSume/emulator-ui/actions/workflows/ci.yml"><img src="https://github.com/DesAWSume/emulator-ui/actions/workflows/ci.yml/badge.svg?branch=main" alt="CI"></a>
  <img src="assets/brand/badges/services.svg" alt="services: 436">
  <img src="assets/brand/badges/emulator.svg" alt="tested with: emulator v1.0.322">
  <img src="assets/brand/badges/python.svg" alt="Python: 3.11+">
  <img src="assets/brand/badges/react.svg" alt="React: 18.3">
  <img src="assets/brand/badges/fastapi.svg" alt="FastAPI: 0.115+">
  <img src="assets/brand/badges/license.svg" alt="license: Apache 2.0">
</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ◆
  <a href="#emulator-requirements">Emulator requirements</a> ◆
  <a href="#features">Features</a> ◆
  <a href="#development">Development</a> ◆
  <a href="#contact">Contact</a>
</p>

<p align="center">
  <img src="docs/images/demo.gif" alt="Emulator UI demo: dashboard, search, DynamoDB items, S3, Step Functions history and running an API operation" width="100%">
</p>

## Quick start

Requires Docker and a running AWS Emulator (see [Emulator requirements](#emulator-requirements)).

```bash
git clone https://github.com/DesAWSume/emulator-ui.git && cd emulator-ui
make up         # build the image from source + start -> http://127.0.0.1:8080
make up-image   # or pull the prebuilt image (amd64/arm64) from ghcr.io/desawsume/emulator-ui
make down       # stop the console
```

`make up` needs no registry: it builds `docker/console.Dockerfile` locally and tags it `emulator-ui`. Use this
unless you specifically want a prebuilt image.

`make up-image` pulls `$(IMAGE)`, default `ghcr.io/desawsume/emulator-ui:latest`. That only works once the image
has been published and made public (see [Publishing the image](#publishing-the-image)); until then it fails with
`error from registry: denied`, which is how the registry reports a missing or private package to an anonymous
pull. Point it elsewhere with `make up-image IMAGE=ghcr.io/you/emulator-ui:v0.1.0`.

### Publishing the image

Either push a version tag and let CI do it:

```bash
git tag v0.1.0 && git push origin v0.1.0   # runs .github/workflows/publish-image.yml (amd64 + arm64)
```

or build and push from your machine:

```bash
docker login ghcr.io -u <github-username>   # password = classic PAT with write:packages
make image-push                             # or: make image-push IMAGE=ghcr.io/you/emulator-ui:v0.1.0
```

A new package is **private** by default — it inherits the repository's permissions but not its visibility. For
`make up-image` to work without `docker login`, open the package from your profile's Packages tab once and set
*Package settings → Change visibility → Public*.

The console runs in its own container and never starts or stops emulators. Start them in any order; the console
picks up emulator restarts, new regions and new keys on its own.

## Emulator requirements

| | |
|---|---|
| Container | named `aws-emulator-<port>` (default port `9297`), on the Docker network `aws-emulator`, listening on `0.0.0.0:<port>` inside the container |
| Files | the emulator's `emulator_credentials`, `config` and `ca.pem` in `~/.aws/emulator/<port>/` on the host |
| Host port | published on `127.0.0.1:<port>` (used by `make dev` and `make test`) |
| Regions (optional) | `~/.aws/emulator/regions` with `region=port` lines for regions that have their own emulator |

Example, with an image built from the [official Dockerfile](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html)
and tagged `aws-emulator`:

```bash
docker network create aws-emulator 2>/dev/null || true
mkdir -p ~/.aws/emulator
docker run -d --rm --name aws-emulator-9297 --network aws-emulator \
  --user "$(id -u):$(id -g)" -p 127.0.0.1:9297:9297 \
  -e HOME="$HOME" -v "$HOME/.aws/emulator:$HOME/.aws/emulator" \
  aws-emulator -p 9297 --telemetry off
```

## Features

- **Dashboard:** search all 436 services. Services holding resources are listed first, the rest are grouped by
  category. The sidebar lists only active services, with counts.
- **Any service:** browse its resources as tables, or run *any* API operation from a JSON editor pre-filled from the
  API model.
- **Dedicated pages:**
  - DynamoDB items, S3 browser, Lambda invoke;
  - API Gateway requests;
  - Step Functions executions, EventBridge events, SQS messages;
  - AppSync and IAM.
- **Region switcher:** shared regions use the default emulator; isolated regions (listed in
  `~/.aws/emulator/regions`) get their own.
- **Safe by default:**
  - localhost only;
  - CSRF and DNS-rebinding guards;
  - never calls a non-local endpoint;
  - destructive actions ask for confirmation.

  See [SECURITY.md](SECURITY.md).

Resource counts exclude the items a fresh emulator already has (recorded in `src/emulator_ui/inventory_baseline.json`).

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Emulator unreachable" / "Emulator is off." | Start an emulator that meets the [requirements](#emulator-requirements). |
| No resources | Emulators keep state in memory, so a new run is empty. Create resources, then click **Rescan resources**. |
| Resource missing in another region | Check the region switcher; regions can be isolated. |
| "Not implemented by the emulator yet (501)" | The emulator doesn't support that operation yet. |

## Development

```bash
make dev           # API on :8081 with reload + Vite on http://127.0.0.1:5173
make test          # backend tests (live tests need a running emulator)
make ui-test       # frontend typecheck + tests
make lint          # ruff + shellcheck
make catalog       # regenerate the service catalog and icons (offline)
make inventory-baseline   # re-record a fresh emulator's built-in items
```

Backend: FastAPI in `src/emulator_ui/`. Frontend: React + TypeScript (Vite) in `ui/`. Inside the container, the
console reaches each emulator over the `aws-emulator` network through a loopback relay, so TLS is verified end to end
against the emulator's own certificate. API docs: http://127.0.0.1:8080/api/docs.

## Contact

- Bugs and questions: [GitHub Issues](https://github.com/DesAWSume/emulator-ui/issues)
- Contributing: see [CONTRIBUTING.md](CONTRIBUTING.md)
- Security: see [SECURITY.md](SECURITY.md); don't open public issues

## License

[Apache License 2.0](LICENSE). Copyright 2026 Deeptaroop Mojumdar. See [NOTICE](NOTICE).

This repository is a fork of [D33ptar00p/emulator-ui](https://github.com/D33ptar00p/emulator-ui); it builds and
publishes its own container image but keeps the upstream copyright and attribution the licence requires.
