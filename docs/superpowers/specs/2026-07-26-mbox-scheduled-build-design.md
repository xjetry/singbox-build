# mbox Scheduled Build Design

## Goal

Add `enfein/mbox` as a second scheduled source without changing the existing
`SagerNet/sing-box` build. Publish only Linux `amd64` and `arm64` archives built
from source with the V2Ray API enabled.

## Workflow Isolation

Create `.github/workflows/build-mbox.yml` with its own daily schedule,
`workflow_dispatch` trigger, push path filter, and `build-mbox` concurrency
group. Its schedule will be offset from the existing `03:17 UTC` sing-box job
so the two builds do not normally contend for runners at the same time.

The workflow will query `enfein/mbox` for the latest stable release and latest
pre-release. It will preserve the upstream tag for checkout, but use
`mbox-<upstream-tag>` for this repository's release tag. For example, upstream
`v1.13.13` is published here as `mbox-v1.13.13`. This prevents collisions with
official sing-box releases that use the same version.

Before building, the check job will skip each candidate whose prefixed release
already exists. Stable and pre-release candidates remain independent, and a
pre-release is omitted when it is absent or identical to the stable release.

## Build And Packaging

For every selected upstream tag, the workflow will check out `enfein/mbox` and
use the current stable Go toolchain. It will read mbox's
`release/DEFAULT_BUILD_TAGS_OTHERS` and `release/LDFLAGS`, append
`with_v2ray_api` to the upstream build tags, and build `./cmd/sing-box` with
`CGO_ENABLED=0` for exactly these targets:

- `linux/amd64`
- `linux/arm64`

The executable inside each archive remains `sing-box`, matching the upstream
command. Archives identify their source explicitly:

- `mbox-<version>-linux-amd64.tar.gz`
- `mbox-<version>-linux-arm64.tar.gz`

Each release also contains `checksums.sha256` covering both archives. The
release title, notes, and upstream link identify mbox, list the effective build
tags, and retain the upstream pre-release status.

The release is created only after both architectures and the checksum file are
successfully produced. A checkout, compilation, or packaging failure therefore
leaves no incomplete release.

## Documentation And Verification

Update `README.md` to describe both scheduled sources, their release tag
conventions, source-specific build tags, supported architectures, archive
names, and manual workflow dispatch.

Verify the workflow YAML parses, inspect embedded shell scripts with a shell
syntax checker where practical, and review the resulting diff to ensure the
existing sing-box workflow remains behaviorally unchanged.
