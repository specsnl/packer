# Multi-Image Release Pipeline — Implementation Plan

## Goal

This repository contains multiple Docker images that must be **released
independently**. Each release is triggered manually, picks one image, and bumps
that image's version (major / minor / patch / prerelease). The computed version
is baked into the image, pushed to GHCR as a multi-arch manifest, and recorded
back in git.

Today the repo contains one image (`packer`); the pipeline is built to scale to
several without restructuring.

## Context and conventions

- **Images live at the repo root**, one directory per image, each with a
  `Dockerfile`: `packer/Dockerfile`, and later `<image>/Dockerfile`. No
  `images/` wrapper directory.
- **GHCR path is `ghcr.io/<owner>/<image>`** — i.e.
  `ghcr.io/specsnl/packer`. This matches what `main.yml` and `pr.yml` already
  publish, so existing pull references keep working. Each new image becomes its
  own top-level GHCR package.
- **`ghcr.io/specsnl/utils:latest`** is a public image containing `svu`, `jq`,
  `git`, and `bash`. No credentials block is needed to pull it.
- **`specsnl/github-actions@2.1.0`** is the pinned version of the reusable
  workflows. Verified against the tag: `build.yml` accepts `runs-on`,
  `platform`, `image-name`, `dockerfile`, `context`, `title`, `description`;
  `merge.yml` accepts `runs-on`, `image-name`, `title`, `description`,
  `raw-tag`, `raw-tags`, `version`. Everything this plan relies on exists.
- **Git tags are the single source of truth for versions.** Nothing queries the
  registry. Tags are namespaced per image: `<image>/v<semver>` (e.g.
  `packer/v1.3.0`, `packer/v2.0.1-rc.2`). Each image has its own independent
  version line within one repo.
- **Build is native per-arch** (no QEMU): `ubuntu-24.04` for `linux/amd64`,
  `ubuntu-24.04-arm` for `linux/arm64`.
- **Five jobs in order:** `version` → `lint` → `build` → `merge` → `tag`. The
  git tag is pushed last, only after the image is successfully in GHCR, so a
  failed build never leaves a phantom version claimed in git.

### Meaning of the mutable tags

| Tag                           | Meaning                                      | Written by                      |
|-------------------------------|----------------------------------------------|---------------------------------|
| `latest`                      | most recent stable release                   | `release.yml` (stable only)     |
| `<channel>` (`rc`, `beta`, …) | most recent release on that prerelease train | `release.yml` (prerelease only) |
| `<semver>`                    | immutable, one specific release              | `release.yml`                   |
| `edge`                        | most recent commit on `main`                 | `main.yml`                      |

This is a change from current behaviour: `main.yml` publishes `latest` today,
which means `latest` tracks `main` rather than releases. It becomes `edge`.

---

## Relationship to the existing workflows

- **[main.yml](../.github/workflows/main.yml)** — keep, but change the `merge`
  job to `raw-tag: edge` so pushes to `main` no longer overwrite the released
  `latest`. Continuous build verification is retained.
- **[pr.yml](../.github/workflows/pr.yml)** — unchanged.
- **[tag.yml](../.github/workflows/tag.yml)** — **delete.** It builds and merges
  on `push: tags: ['*']`, which is a second, overlapping release mechanism.
  It also cannot fire for releases made by this pipeline: tags pushed using
  `GITHUB_TOKEN` do not trigger workflow runs. Leaving it in place is dead
  weight that reads like a working release path.
- **New: `release.yml`** — the workflow specified below.

---

## Version line start

Existing tags are unnamespaced (`0.2.0`–`0.5.0`, `v0.1.0`–`v0.1.2`) and are
left in place as history. The `packer/v*` line **starts fresh at `1.0.0`**.

On a fresh namespace svu fails cleanly with
`Failed to get current tag for repo: no tags match 'packer/v*'` rather than
silently starting from `0.0.0`, so the first release uses the explicit `version`
input:

```
image: packer, version: 1.0.0
```

Subsequent releases use the normal bump path. No seed tag needs to be pushed by
hand — the first release creates `packer/v1.0.0` itself, and only after a
successful build. Same applies to every future image: its first release is an
explicit `version`.

---

## Workflow inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      image:
        type: choice
        options: [packer]        # keep in sync with the top-level image dirs
      release_type:
        type: choice
        options: [patch, minor, major, prerelease]
        default: patch
      prerelease_id:
        description: Set to START a prerelease train (e.g. rc.1, beta.1). Empty for stable.
        type: string
        required: false
      version:
        description: Explicit version, overrides everything else (e.g. 1.4.0 or 1.4.0-rc.1)
        type: string
        required: false
```

### Input → behaviour

| Current state | `release_type` | `prerelease_id` | Result                                    |
|---------------|----------------|-----------------|-------------------------------------------|
| `1.2.3`       | `patch`        | _(empty)_       | `1.2.4`                                   |
| `1.2.3`       | `minor`        | _(empty)_       | `1.3.0`                                   |
| `1.2.3`       | `major`        | _(empty)_       | `2.0.0`                                   |
| `1.2.3`       | `minor`        | `rc.1`          | `1.3.0-rc.1` (starts a train)             |
| `1.3.0-rc.1`  | `prerelease`   | _(empty)_       | `1.3.0-rc.2` (auto-increments)            |
| `1.3.0-rc.2`  | `patch`        | _(empty)_       | `1.3.0` (finalizes; drops the prerelease) |

svu behaviours relied on:

- `svu prerelease` auto-increments the trailing counter when the current tag is
  already a prerelease — no identifier needed to continue a train.
- Starting a train from a stable tag requires a bump command plus an identifier:
  `svu minor --prerelease rc.1`.
- `svu patch` on a prerelease tag clears the prerelease without incrementing —
  this is promotion to stable.

`prerelease_id` is what disambiguates a stable bump from a prerelease bump.
The `version` field is the escape hatch for anything the bump logic doesn't
model; when set it is used verbatim.

### svu tag selection — `--tag.prefix` does not filter

This is the single most important detail in the whole design, and it is not what
the flag names suggest. **`--tag.prefix` only strips and prints; it does not
control which tag svu selects.** svu picks the current tag by its own ordering,
prefix-blind, and only then tries to parse it under the given prefix — so in a
repo with more than one tag namespace it reads the wrong tag and errors out.

Verified against `ghcr.io/specsnl/utils:latest` in a scratch repo containing
`packer/v1.2.3`, `ansible/v3.0.0` and `0.5.0`:

```console
$ svu patch --tag.prefix=ansible/v
  ERROR  Could not get current version from tag: 'packer/v1.2.3': invalid semantic version.

$ svu patch --tag.prefix=ansible/v --tag.pattern='ansible/v*'
  {"version":"ansible/v3.0.1","major":3,"minor":0,"patch":1,"prefix":"ansible/v"}
```

**`--tag.pattern` is mandatory, not optional.** Every svu invocation must pass
both `--tag.prefix="<image>/v"` and `--tag.pattern="<image>/v*"`. Omitting the
pattern produces a pipeline that works by luck while one namespace exists and
breaks the moment a second image is added — and this repo's existing bare
`0.2.0`–`0.5.0` tags are already enough to trigger it.

Also verified: the `--json` shape the reconstruction below depends on is real —
`svu prerelease` on `packer/v1.3.0-rc.1` yields
`{"prerelease":"rc","build":"2"}`, which reconstructs to `1.3.0-rc.2`. Note that
the JSON `version` field carries the prefix (`"packer/v1.2.4"`), which is why
the script rebuilds the semver from `major`/`minor`/`patch` instead of reading
`version` directly.

---

## Per-image metadata

`title` and `description` feed the OCI labels and must not regress from what
`main.yml` passes today. They live in the `version` job as a `case` on the image
name — one place to edit when an image is added, and they travel with the
computed version to both `build` and `merge`.

```bash
case "${IMAGE}" in
  packer)
    TITLE="packer"
    DESCRIPTION="Docker Image with HashiCorp Packer and Ansible installed"
    ;;
  *)
    echo "::error::Unknown image '${IMAGE}' — add it to the metadata case block" >&2
    exit 1
    ;;
esac
```

The catch-all `exit 1` is deliberate: adding an entry to the `image` choice list
without adding metadata should fail loudly at the start of the run, not produce
an unlabelled image.

---

## Version resolution script

Used in the `version` job. svu's `--json` output shape:

```go
type VersionInfo struct {
    Version    string `json:"version"`
    Major      uint64 `json:"major"`
    Minor      uint64 `json:"minor"`
    Patch      uint64 `json:"patch"`
    Prefix     string `json:"prefix,omitempty"`
    Metadata   string `json:"metadata,omitempty"`
    Prerelease string `json:"prerelease,omitempty"`  // "rc", "beta", ...
    Build      string `json:"build,omitempty"`       // "1", "2", ... (counter)
}
```

Semver is reconstructed from the numeric fields (prefix-agnostic and exact):

```bash
set -euo pipefail

PREFIX="${IMAGE}/v"
PATTERN="${PREFIX}*"

if [ -n "${EXPLICIT_VERSION}" ]; then
  SEMVER="${EXPLICIT_VERSION}"
  core="${SEMVER%%+*}"
  case "$core" in
    *-*) IS_PRERELEASE=true; pre="${core#*-}"; CHANNEL="${pre%%.*}" ;;
    *)   IS_PRERELEASE=false; CHANNEL="" ;;
  esac
else
  case "${RELEASE_TYPE}" in
    prerelease) SUBCMD="prerelease" ;;
    *)          SUBCMD="${RELEASE_TYPE}" ;;
  esac

  # --tag.pattern is what confines svu to this image's tag namespace.
  # --tag.prefix alone does NOT filter (see "svu tag selection" below).
  JSON=$(svu "${SUBCMD}" ${PRERELEASE_ID:+--prerelease "${PRERELEASE_ID}"} \
              --tag.prefix="${PREFIX}" --tag.pattern="${PATTERN}" --json)

  SEMVER=$(printf '%s' "$JSON" | jq -r '
    "\(.major).\(.minor).\(.patch)"
    + (if (.prerelease // "") != "" then "-\(.prerelease)"
         + (if (.build // "") != "" then ".\(.build)" else "" end)
       else "" end)
    + (if (.metadata // "") != "" then "+\(.metadata)" else "" end)
  ')
  IS_PRERELEASE=$(printf '%s' "$JSON" | jq -r 'if (.prerelease // "") == "" then "false" else "true" end')
  CHANNEL=$(printf '%s' "$JSON" | jq -r '.prerelease // ""')
fi

GIT_TAG="${PREFIX}${SEMVER}"

# Fail fast: don't spend two native builds on a version that can't be recorded.
# The non-force push in the `tag` job is the real guard; this is the early one.
if git rev-parse -q --verify "refs/tags/${GIT_TAG}" >/dev/null; then
  echo "::error::Tag ${GIT_TAG} already exists — this version was already released" >&2
  exit 1
fi

{
  echo "semver=${SEMVER}"
  echo "git_tag=${GIT_TAG}"
  echo "is_prerelease=${IS_PRERELEASE}"
  echo "channel=${CHANNEL}"
  echo "title=${TITLE}"
  echo "description=${DESCRIPTION}"
} >> "$GITHUB_OUTPUT"

echo "Releasing ${IMAGE} ${SEMVER} (tag ${GIT_TAG})" >> "$GITHUB_STEP_SUMMARY"
```

Notes for the implementer:

- `>> "$GITHUB_OUTPUT"` works normally inside a container job.
- The container default shell is `sh`; `bash` is set explicitly at the job level
  via `defaults.run.shell`.
- `${PRERELEASE_ID:+--prerelease "${PRERELEASE_ID}"}` is POSIX conditional
  expansion: the flag is only included when `PRERELEASE_ID` is non-empty.
- The existence check needs the tag fetched, hence `fetch-depth: 0` +
  `fetch-tags: true` on the checkout.

---

## Tagging rules (applied by `merge.yml` via `create-manifest`)

- **`semver`** — always; passed as `raw-tag`.
- **`latest`** — only when `is_prerelease == 'false'`; passed as a `raw-tags`
  line with `enable=`.
- **`channel`** (`rc`, `beta`, …) — only when `is_prerelease == 'true'`; passed
  as a `raw-tags` line with `enable=`. Lets consumers opt into a prerelease
  track without ever pulling `latest`.
- **OCI `org.opencontainers.image.version`** label and annotation — passed via
  the `version` input.

On a stable release the channel line renders as `type=raw,value=,enable=false`.
That is harmless: `docker/metadata-action` evaluates `enable` and discards the
directive before the empty value is ever used.

---

## Full workflow

`.github/workflows/release.yml`:

```yaml
name: Release image

on:
  workflow_dispatch:
    inputs:
      image:
        type: choice
        options: [packer]        # keep in sync with the top-level image dirs
      release_type:
        type: choice
        options: [patch, minor, major, prerelease]
        default: patch
      prerelease_id:
        description: Set to START a prerelease train (e.g. rc.1, beta.1). Empty for stable.
        type: string
        required: false
      version:
        description: Explicit version, overrides everything else (e.g. 1.4.0 or 1.4.0-rc.1)
        type: string
        required: false

permissions:
  packages: write
  contents: write
  actions: write        # matches main.yml; the reusable build needs it for cache

# Prevent two concurrent releases of the same image from computing the same
# next version and colliding on the git tag push.
concurrency:
  group: release-${{ inputs.image }}
  cancel-in-progress: false

jobs:
  version:
    name: Version
    runs-on: ubuntu-24.04
    container:
      image: ghcr.io/specsnl/utils:latest   # public; no credentials needed
    defaults:
      run:
        shell: bash                          # container default is sh
    outputs:
      semver:        ${{ steps.v.outputs.semver }}
      git_tag:       ${{ steps.v.outputs.git_tag }}
      is_prerelease: ${{ steps.v.outputs.is_prerelease }}
      channel:       ${{ steps.v.outputs.channel }}
      title:         ${{ steps.v.outputs.title }}
      description:   ${{ steps.v.outputs.description }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0     # svu needs full history to walk tags
          fetch-tags: true
      - id: v
        env:
          IMAGE:            ${{ inputs.image }}
          RELEASE_TYPE:     ${{ inputs.release_type }}
          PRERELEASE_ID:    ${{ inputs.prerelease_id }}
          EXPLICIT_VERSION: ${{ inputs.version }}
        run: |
          # <-- per-image metadata case + version resolution script from above -->

  lint:
    name: Linting
    needs: version
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Hadolint
        uses: hadolint/hadolint-action@v3.4.0
        with:
          dockerfile: ${{ inputs.image }}/Dockerfile

  build:
    name: Build
    needs: [version, lint]
    uses: specsnl/github-actions/.github/workflows/build.yml@2.1.0
    strategy:
      fail-fast: false
      matrix:
        runs-on:
          - { os: ubuntu-24.04,     platform: linux/amd64 }
          - { os: ubuntu-24.04-arm, platform: linux/arm64 }
    with:
      runs-on:     ${{ matrix.runs-on.os }}
      platform:    ${{ matrix.runs-on.platform }}
      image-name:  ghcr.io/${{ github.repository_owner }}/${{ inputs.image }}
      dockerfile:  ${{ inputs.image }}/Dockerfile
      title:       ${{ needs.version.outputs.title }}
      description: ${{ needs.version.outputs.description }}

  merge:
    name: Merge
    needs: [version, build]
    uses: specsnl/github-actions/.github/workflows/merge.yml@2.1.0
    with:
      runs-on:     ubuntu-24.04
      image-name:  ghcr.io/${{ github.repository_owner }}/${{ inputs.image }}
      raw-tag:     ${{ needs.version.outputs.semver }}
      raw-tags: |
        type=raw,value=latest,enable=${{ needs.version.outputs.is_prerelease == 'false' }}
        type=raw,value=${{ needs.version.outputs.channel }},enable=${{ needs.version.outputs.is_prerelease == 'true' }}
      version:     ${{ needs.version.outputs.semver }}
      title:       ${{ needs.version.outputs.title }}
      description: ${{ needs.version.outputs.description }}

  # Tag git LAST — only after a successful build and merge. A failed build
  # never leaves a version claimed in git. No force-push: if the tag already
  # exists the push fails loudly, which is the duplicate-release guard.
  tag:
    name: Tag
    needs: [version, merge]
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - name: Push git tag
        env:
          GIT_TAG: ${{ needs.version.outputs.git_tag }}
          SEMVER:  ${{ needs.version.outputs.semver }}
          IMAGE:   ${{ inputs.image }}
          SHA:     ${{ github.sha }}
        run: |
          set -euo pipefail
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git tag -a "${GIT_TAG}" -m "Release ${IMAGE} ${SEMVER}" "${SHA}"
          git push origin "refs/tags/${GIT_TAG}"
          echo "Tagged ${SHA} as ${GIT_TAG}" >> "$GITHUB_STEP_SUMMARY"
```

Notes on this workflow versus the earlier draft:

- `git config user.name` / `user.email` are required — `git tag -a` aborts
  without a committer identity, which would have failed every release.
- The tag is created on `${{ github.sha }}` explicitly rather than on whatever
  `HEAD` the checkout resolved. Same commit in practice for `workflow_dispatch`,
  but it makes "the tag points at the commit that was built" a property of the
  workflow rather than a coincidence.
- `if: success()` was dropped from `tag`: a `needs` job that fails already
  skips its dependents, so the condition was a no-op.
- `actions/checkout@v7` matches the version already used across this repo
  (the earlier draft used `@v6`).

---

## Implementation checklist

- [x] ~~Confirm `ghcr.io/specsnl/utils:latest` exists, is public, and contains
      `svu`, `jq`, `git`, `bash`.~~ Verified — image pulls anonymously and ships
      svu with `--tag.prefix` / `--tag.pattern` / `--json` support.
- [ ] Every svu call passes **both** `--tag.prefix` and `--tag.pattern`. See
      "svu tag selection" above — the pattern is what scopes svu to the image's
      namespace, and omitting it is a latent break, not a style nit.
- [ ] Add `.github/workflows/release.yml` as specified above.
- [ ] Change `main.yml`'s `merge` job to `raw-tag: edge`.
- [ ] Delete `.github/workflows/tag.yml`.
- [ ] Update `README.md` to document the tag meanings (`latest`, `edge`,
      `<channel>`, `<semver>`) and how to cut a release.
- [ ] First release: dispatch with `image: packer`, `version: 1.0.0`. Verify
      `packer/v1.0.0` lands in git and `ghcr.io/specsnl/packer:1.0.0` +
      `:latest` land in GHCR.
- [ ] Then test each release path in order:
      - `patch` (stable bump) → `1.0.1`
      - `minor` (stable bump) → `1.1.0`
      - `major` (stable bump) → `2.0.0`
      - `minor` + `prerelease_id: rc.1` → `2.1.0-rc.1`, tagged `rc`, **not** `latest`
      - `prerelease` with empty `prerelease_id` → `2.1.0-rc.2`
      - `patch` with empty `prerelease_id` → `2.1.0`, tagged `latest`
      - re-dispatch a version that already exists → fails in the `version` job
        before any build runs
- [ ] When adding a second image: create `<image>/Dockerfile`, add it to the
      `image` choice list, add its `case` entry in the metadata block, and add
      its paths-filter entry + lint/build/merge jobs in `pr.yml` and `main.yml`.
