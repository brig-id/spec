# ADR-011 — Releases: tagged versions everywhere, release-please, Renovate

**Status:** Accepted (2026-11-13)

## Context

- `app` has one tag (`v0.1.0`) and no release process. `server-leaf` builds its image from a
  bare `app` commit written twice in its workflow, bumped by hand before each release.
- `core` is pinned by commit in `server-leaf`'s `Cargo.toml`. The organisation's rule is the
  opposite: pin a tag, never a commit that only a rebased branch reaches.
- A release is three manual edits that must agree (`Cargo.toml`, `CHANGELOG.md`, the tag).
  They have disagreed more than once.
- Dependabot opens the update pull requests. It cannot follow the `app` version a server
  build uses.
- The reference server `helm.brig.id` already deploys every `leaf` tag by itself. That is
  written nowhere in this repository.

## Decision

### 1. Every repository releases by a semver tag, created by CI

`crypto`, `core`, `app` and each server publish versions as tags `vX.Y.Z` on `main`. A tag
is never created by hand or by an agent: it is the result of merging a release pull request.

### 2. release-please prepares and cuts the releases

| | release-please | git-cliff |
| --- | --- | --- |
| What it is | Release automation: keeps a release pull request up to date; merging it creates the tag and the GitHub Release | A changelog generator |
| Version bump | Computed from Conventional Commits; edits `Cargo.toml` + `Cargo.lock` (`rust`) or `package.json` (`node`) | `--bump` computes it; editing the manifests, tagging and publishing are scripts to write |
| Changelog | Fixed layout, grouped by commit type | Fully templated; handles gitmoji well |
| Fit with "everything through a pull request" | The release *is* a pull request a human merges | Needs a workflow of our own to open that pull request |
| Covers Rust and Node | Yes, one tool | Yes for the changelog; for the rest, `release-plz` exists for Rust only |

**release-please**, in manifest mode, in every repository. It removes the three manual edits
and makes the release a reviewed pull request, which is how everything else already reaches
`main`. git-cliff produces nicer changelogs but leaves the release itself to write and
maintain; it remains an option as a changelog renderer if the fixed layout becomes a
problem.

Settings that follow from the organisation's conventions:

- `pull-request-title-pattern` is a literal that passes the commit check of the repository,
  e.g. `chore(leaf): 🔖 release v${version}`.
- `bump-minor-pre-major: true` while versions are `0.x`.
- The workflows that publish (container image, artefacts) run **in the same workflow** as
  release-please, on its `release_created` output: a tag pushed with the default
  `GITHUB_TOKEN` does not trigger other workflows.
- To check on the first repository (`app`): that release-please recognises its release pull
  request after a rebase merge, the only merge allowed here.

### 3. Renovate replaces Dependabot

One bot, in every repository, with a shared preset kept in `brig-id/.github`:

- the usual managers (cargo, npm, GitHub Actions, Dockerfile digests);
- `crypto` and `core` pinned **by tag** in `Cargo.toml` (`tag = "vX.Y.Z"`), updated through
  the cargo manager's git-tag datasources (to confirm on the first pull request; otherwise
  the custom manager below);
- **the `app` version of a server build**: one pinned reference per server repository, tag
  and commit together, updated by a custom (regex) manager on the `github-tags` datasource:

  ```yaml
  # renovate: datasource=github-tags depName=brig-id/app
  ref: 0123456789abcdef0123456789abcdef01234567 # v1.2.0
  ```

  The server keeps building the UI from source at that commit: nothing binary is trusted.

Renovate never merges. A human merges every pull request, as today.

### 4. The chain

```text
crypto vX ──▶ core (Renovate PR) ──▶ core vY ──┐
                                               ├──▶ server (Renovate PR) ──▶ server vZ ──▶ image, binaries
app vW ────────────────────────────────────────┘
```

### 5. The reference server follows the tags (as built)

`helm.brig.id` deploys by pulling, not by being pushed to: a timer on the machine reads its
deployment files from a private repository and starts the highest `vX.Y.Z` image published
for `leaf`. No workflow reaches the machine and GitHub holds no deployment secret.

**Publishing a server tag is therefore deploying to the reference server.** The production
gate is the merge of the release pull request. This is documented in
[`operations.md`](../operations.md) §6; nothing is to be built.

## Consequences

- `app`: release-please, `CHANGELOG.md` kept by it, first release cut through it.
- `crypto`, `core`, `server-leaf`: release-please; the release checklist of the organisation's
  `AGENTS.md` shrinks to "merge the release pull request".
- `.github`: the Renovate preset; Dependabot configurations removed in the same change.
  Installing the Renovate app on the organisation is an owner action.
- `server-leaf`: `core` pinned by tag; the two `app` references of the workflow become one.

## References

- release-please: <https://github.com/googleapis/release-please> (manifest releases,
  `docs/customizing.md`)
- git-cliff: <https://git-cliff.org>; release-plz: <https://release-plz.dev>
- Renovate: <https://docs.renovatebot.com/modules/manager/cargo/>,
  <https://docs.renovatebot.com/modules/manager/regex/>
