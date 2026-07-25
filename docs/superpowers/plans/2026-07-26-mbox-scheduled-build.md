# mbox Scheduled Build Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an isolated daily build that publishes the latest stable and pre-release versions of `enfein/mbox` for Linux `amd64` and `arm64`.

**Architecture:** A new GitHub Actions workflow mirrors the existing release discovery and source-build flow while using source-prefixed release tags to avoid collisions. The mbox workflow reads build configuration from the checked-out source, adds `with_v2ray_api`, and publishes two pure-Go archives plus checksums.

**Tech Stack:** GitHub Actions, Bash, GitHub CLI, jq, Go

## Global Constraints

- Keep `.github/workflows/build.yml` behaviorally unchanged.
- Run mbox on its own daily schedule, manual trigger, path trigger, and `build-mbox` concurrency group.
- Track both the latest stable release and latest pre-release from `enfein/mbox`.
- Publish upstream `vX.Y.Z` as local `mbox-vX.Y.Z` to prevent release tag collisions.
- Build exactly `linux/amd64` and `linux/arm64` with `CGO_ENABLED=0`.
- Read `release/DEFAULT_BUILD_TAGS_OTHERS` and `release/LDFLAGS` from mbox, then append `with_v2ray_api`.
- Name archives `mbox-<version>-linux-<arch>.tar.gz`; keep the executable inside named `sing-box`.
- Do not commit changes unless the user explicitly requests a commit.

## File Structure

- Create `.github/workflows/build-mbox.yml`: discover mbox releases, build two Linux targets, package them, and publish source-prefixed releases.
- Modify `README.md`: document both sources, source-specific build configuration, release tags, and artifact names.
- Keep `docs/superpowers/specs/2026-07-26-mbox-scheduled-build-design.md` as the approved design record.

---

### Task 1: Add The Isolated mbox Workflow

**Files:**
- Create: `.github/workflows/build-mbox.yml`
- Reference: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: GitHub Releases API for `enfein/mbox`, mbox tags, `release/DEFAULT_BUILD_TAGS_OTHERS`, and `release/LDFLAGS`.
- Produces: releases tagged `mbox-${upstream_tag}` containing two archives and `checksums.sha256`.

- [ ] **Step 1: Confirm the workflow does not exist yet**

Run:

```bash
test ! -e .github/workflows/build-mbox.yml
```

Expected: exit status 0 with no output.

- [ ] **Step 2: Create the complete workflow**

Create `.github/workflows/build-mbox.yml` with this content:

```yaml
name: Build mbox

on:
  schedule:
    - cron: '47 3 * * *'
  workflow_dispatch:
  push:
    branches: [main]
    paths: ['.github/workflows/build-mbox.yml']

permissions:
  contents: write

concurrency:
  group: build-mbox
  cancel-in-progress: false

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      targets: ${{ steps.decide.outputs.targets }}
    steps:
      - name: Find latest release and pre-release
        id: latest
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          stable="$(gh api repos/enfein/mbox/releases/latest --jq .tag_name)"
          pre="$(gh api 'repos/enfein/mbox/releases?per_page=50' --jq '[.[] | select(.prerelease)][0].tag_name // empty')"
          echo "latest stable: $stable, latest pre-release: ${pre:-none}"
          echo "stable=$stable" >> "$GITHUB_OUTPUT"
          echo "pre=$pre" >> "$GITHUB_OUTPUT"

      - name: Skip tags already released here
        id: decide
        env:
          GH_TOKEN: ${{ github.token }}
          STABLE: ${{ steps.latest.outputs.stable }}
          PRE: ${{ steps.latest.outputs.pre }}
        run: |
          set -euo pipefail
          candidates="$(jq -cn --arg s "$STABLE" --arg p "$PRE" \
            '[{tag: $s, prerelease: false}] + (if $p != "" and $p != $s then [{tag: $p, prerelease: true}] else [] end)')"
          targets='[]'
          while read -r item; do
            tag="$(jq -r .tag <<<"$item")"
            release_tag="mbox-$tag"
            if gh release view "$release_tag" --repo "$GITHUB_REPOSITORY" >/dev/null 2>&1; then
              echo "release $release_tag already exists, skipping"
            else
              item="$(jq -c --arg release_tag "$release_tag" '. + {release_tag: $release_tag}' <<<"$item")"
              targets="$(jq -c --argjson i "$item" '. + [$i]' <<<"$targets")"
            fi
          done < <(jq -c '.[]' <<<"$candidates")
          echo "targets to build: $targets"
          echo "targets=$targets" >> "$GITHUB_OUTPUT"

  build:
    needs: check
    if: needs.check.outputs.targets != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.check.outputs.targets) }}
    steps:
      - name: Checkout mbox ${{ matrix.tag }}
        uses: actions/checkout@v4
        with:
          repository: enfein/mbox
          ref: ${{ matrix.tag }}

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: stable
          check-latest: true

      - name: Load build configuration
        run: |
          echo "BUILD_TAGS=$(cat release/DEFAULT_BUILD_TAGS_OTHERS),with_v2ray_api" >> "$GITHUB_ENV"
          echo "LDFLAGS=$(cat release/LDFLAGS)" >> "$GITHUB_ENV"

      - name: Build
        env:
          TAG: ${{ matrix.tag }}
          CGO_ENABLED: '0'
        run: |
          set -euo pipefail
          VERSION="${TAG#v}"
          mkdir -p dist
          for arch in amd64 arm64; do
            echo "::group::build linux/$arch"
            name="mbox-$VERSION-linux-$arch"
            mkdir -p "dist/$name"
            GOOS=linux GOARCH="$arch" go build -v -trimpath \
              -tags "$BUILD_TAGS" \
              -ldflags "-X github.com/sagernet/sing-box/constant.Version=$VERSION $LDFLAGS -s -w -buildid=" \
              -o "dist/$name/sing-box" ./cmd/sing-box
            tar -C dist -czf "dist/$name.tar.gz" "$name"
            rm -r "dist/${name:?}"
            echo "::endgroup::"
          done
          (cd dist && sha256sum ./*.tar.gz > checksums.sha256)
          ls -lh dist

      - name: Create release
        env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ matrix.tag }}
          RELEASE_TAG: ${{ matrix.release_tag }}
          PRERELEASE: ${{ matrix.prerelease }}
        run: |
          set -euo pipefail
          args=()
          if [ "$PRERELEASE" = "true" ]; then
            args+=(--prerelease)
          fi
          gh release create "$RELEASE_TAG" --repo "$GITHUB_REPOSITORY" \
            --title "mbox $TAG" \
            --notes "mbox $TAG built with tags: \`$BUILD_TAGS\`

          Upstream release: https://github.com/enfein/mbox/releases/tag/$TAG" \
            "${args[@]}" dist/*.tar.gz dist/checksums.sha256
```

- [ ] **Step 3: Validate GitHub Actions syntax**

Run:

```bash
go run github.com/rhysd/actionlint/cmd/actionlint@latest .github/workflows/build-mbox.yml
```

Expected: exit status 0 with no diagnostics.

- [ ] **Step 4: Assert isolation, targets, tags, and artifact naming**

Run:

```bash
grep -F "group: build-mbox" .github/workflows/build-mbox.yml && \
grep -F "release_tag=\"mbox-\$tag\"" .github/workflows/build-mbox.yml && \
grep -F "for arch in amd64 arm64" .github/workflows/build-mbox.yml && \
grep -F 'name="mbox-$VERSION-linux-$arch"' .github/workflows/build-mbox.yml && \
grep -F "release/DEFAULT_BUILD_TAGS_OTHERS" .github/workflows/build-mbox.yml && \
grep -F "with_v2ray_api" .github/workflows/build-mbox.yml
```

Expected: six matching lines and exit status 0.

### Task 2: Document Both Scheduled Sources

**Files:**
- Modify: `README.md:1-21`

**Interfaces:**
- Consumes: artifact and release naming defined by both workflows.
- Produces: user-facing instructions that distinguish official sing-box builds from mbox builds.

- [ ] **Step 1: Replace the README with source-specific documentation**

Use this complete content:

```markdown
# singbox-build

定时构建以下项目的最新稳定版与最新预发布版，并在各自的上游构建标签基础上额外启用 `with_v2ray_api`（V2Ray gRPC 统计 API）：

- [sing-box](https://github.com/SagerNet/sing-box)
- [mbox](https://github.com/enfein/mbox)

Scheduled Linux builds of the latest stable and pre-release versions of sing-box and mbox, with `with_v2ray_api` enabled on top of each source's default build tags.

## 构建配置

sing-box 使用本仓库维护的上游默认标签及 `with_v2ray_api`：

```
with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_v2ray_api
```

mbox 使用对应版本的 `release/DEFAULT_BUILD_TAGS_OTHERS` 和 `release/LDFLAGS`，并在构建标签中追加 `with_v2ray_api`。

## 产物

两个独立 workflow 每天检查一次对应上游的最新 release 与最新 pre-release。尚未发布的版本会自动构建并发布到 [Releases](../../releases)，预发布版本会带 pre-release 标记。

sing-box 保留上游 tag，例如 `v1.13.0`：

- `sing-box-<version>-linux-amd64.tar.gz`
- `sing-box-<version>-linux-arm64.tar.gz`

mbox 使用带来源前缀的 tag，例如 `mbox-v1.13.13`：

- `mbox-<version>-linux-amd64.tar.gz`
- `mbox-<version>-linux-arm64.tar.gz`

每个 Release 均包含对应两个归档的 `checksums.sha256`。也可在 Actions 页面分别手动触发 workflow。
```

- [ ] **Step 2: Check that every published name and source is documented**

Run:

```bash
grep -F "SagerNet/sing-box" README.md && \
grep -F "enfein/mbox" README.md && \
grep -F 'mbox-v1.13.13' README.md && \
grep -F 'mbox-<version>-linux-amd64.tar.gz' README.md && \
grep -F 'mbox-<version>-linux-arm64.tar.gz' README.md && \
grep -F 'checksums.sha256' README.md
```

Expected: six matching lines and exit status 0.

### Task 3: Run Final Repository Verification

**Files:**
- Verify: `.github/workflows/build-mbox.yml`
- Verify: `.github/workflows/build.yml`
- Verify: `README.md`

**Interfaces:**
- Consumes: completed workflow and documentation.
- Produces: evidence that new behavior is valid and the old workflow was not modified.

- [ ] **Step 1: Run the workflow linter and whitespace checks**

Run:

```bash
go run github.com/rhysd/actionlint/cmd/actionlint@latest .github/workflows/build.yml .github/workflows/build-mbox.yml
```

Expected: both commands exit 0 with no diagnostics.

- [ ] **Step 2: Confirm the existing workflow has no changes**

Run:

```bash
git diff --exit-code -- .github/workflows/build.yml
```

Expected: exit status 0 with no output.

- [ ] **Step 3: Review the final scoped diff**

Run:

```bash
git diff -- .github/workflows/build-mbox.yml README.md docs/superpowers/specs/2026-07-26-mbox-scheduled-build-design.md docs/superpowers/plans/2026-07-26-mbox-scheduled-build.md
```

Expected: only the new mbox workflow, README update, and design/plan documentation are present; no unrelated files are modified.
