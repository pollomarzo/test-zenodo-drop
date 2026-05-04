# Automated Zenodo deposits from git tags

## Goal

After PR review merges to `main`, a maintainer pushes a git tag and a Zenodo
deposit (or new version) is prepared automatically. A human confirms by
clicking Publish on Zenodo. No metadata duplication: `myst.yml` is the single
source of truth.

## Architecture

Logic lives in a single Python script (`zenodo-deposit.py`) so we're not
locked into GitHub Actions. The CI workflows are thin wrappers — checkout,
install, run script, open PR.

The script will live in [`isp-actions-config`](../isp-actions-config/) once
stabilized. During testing it lives in this repo for fast iteration.

### Two distinct flows

**First deposit (one-time per paper):** split into "prepare" and "publish"
so the PDF that lands on Zenodo is built from `main` *after* the DOI is
committed — same build as what the rendered site links to.

```
workflow_dispatch ──▶ Prepare
                        ├─ POST deposition (prereserve_doi=true)
                        ├─ open PR: add doi: to myst.yml
                        └─ comment links Zenodo draft URL

(maintainer reviews draft metadata, merges PR)

git tag v1.0.0  ─────▶ Publish
                        ├─ build PDF from main (DOI now baked in)
                        ├─ bundle source + supplementary
                        ├─ upload files to existing draft
                        └─ leaves draft + posts URL

(maintainer clicks Publish on zenodo.org)
```

**New version (every subsequent tag):** no PR needed, `myst.yml` already has
the concept DOI.

```
git tag v1.1.0  ─────▶ Publish
                        ├─ resolve concept DOI → latest deposition id
                        ├─ POST .../actions/newversion
                        ├─ build, bundle, upload
                        └─ leaves draft + posts URL
```

### Idempotency

The publish workflow looks up existing deposits by
`related_identifiers.identifier == <github_url>`. If a draft already exists
(e.g. previous run failed mid-upload), reuse it instead of creating a parallel
one.

## The script: `zenodo-deposit.py`

Single CLI, no GHA-specific code:

```
zenodo-deposit.py prepare \
  --myst myst.yml \
  --version 1.0.0 \
  --repo impact-scholars/<repo> \
  --site-url https://impact-scholars.github.io/<repo> \
  --token $ZENODO_TOKEN \
  [--sandbox]
  > prepare-result.json     # {concept_doi, draft_url}

zenodo-deposit.py publish \
  --myst myst.yml \
  --bundle-dir ./_bundle/ \
  --tag v1.0.0 \
  --site-url https://impact-scholars.github.io/<repo> \
  --token $ZENODO_TOKEN \
  [--sandbox]
  > publish-result.json     # {version_doi, draft_url}

zenodo-deposit.py status \
  --myst myst.yml \
  --token $ZENODO_TOKEN \
  [--sandbox]
  # prints: concept DOI, open draft (if any), latest published version, validation summary
```

Three subcommands. `prepare` and `publish` are idempotent. `status` is a
read-only dry-run helper for debugging.

### `prepare` writes back to myst.yml

The prepare step canonicalizes `project.github` from `--repo` and writes
`project.doi` (the prereserved concept DOI). If `project.github` is missing
or stale (wrong owner after an org migration, etc.), prepare corrects it.
Both changes land in the same PR.

### `publish` validation gates

Before any upload, `publish` fails fast if:

- The tag (`v1.0.0`) version (`1.0.0`) doesn't match the deposit metadata.
- The tagged commit's `myst.yml` has no `project.doi`. This catches the
  case where a maintainer tags before merging the prepare PR — without it,
  we'd build and upload a PDF that's missing the DOI on its title page.
- Required bundle artifacts are missing (`paper.pdf`, `source.zip`,
  `publication-provenance.json`).
- The concept DOI in `myst.yml` resolves to no Zenodo record (sandbox vs
  production token mismatch, deleted draft, etc.).

## Metadata mapping (`myst.yml` → Zenodo)

| myst.yml | Zenodo |
|---|---|
| `project.title` | `title` |
| `project.authors[].name` | `creators[].name` |
| `project.authors[].affiliations[0]` | `creators[].affiliation` (first only) |
| `project.authors[].orcid` (if present) | `creators[].orcid` |
| `project.keywords` | `keywords` |
| `project.license` (`CC-BY-4.0`) | `license` (`cc-by-4.0`, lowercased) |
| `project.venue` | appended to description |
| `project.funding` | free-text in description |
| `project.github` | `related_identifiers` (`isVersionOf`, scheme: `url`) |
| computed site URL | `related_identifiers` (`isIdenticalTo`, scheme: `url`) |
| `project.doi` (concept) | drives newversion vs prepare branch — never sent |
| `project.date` (or tag date) | `publication_date` |
| — | `upload_type: publication`, `publication_type: article` |
| — | `version: <git tag>` |

CRediT `roles` are not mapped — they don't fit Zenodo's `contributors.type`
enum cleanly, and they're already on the rendered page where readers look.

ORCID is sent if present, omitted otherwise. No schema change to `myst.yml`.

No Zenodo Community for now.

## Bundle contents (per deposit)

Computed deterministically in CI, dropped in `_bundle/`:

- `paper.pdf` — built by `myst build --pdf`. Other outputs (HTML
  archive, JATS/XML) are deferred — needs more thorough testing to
  confirm what MyST emits reliably and what's worth including.
- `source.zip` — `git archive --format=zip HEAD`
- supplementary files at repo root: any `*.csv`, `*.png`, `*.txt`, `*.zip`,
  `*.bib` not under `_build/`, `node_modules/`, `.git/`, `.github/`,
  `thumbnails/`
- `myst.yml` itself (deposit is self-describing)
- `publication-provenance.json` — generated by `publish`, contains:
  - `repo` (`impact-scholars/<repo>`), `commit_sha`, `tag`
  - `site_url` (rendered HTML)
  - `concept_doi`, `version_doi`
  - `review_pr` references if discoverable from commit history
  - `built_at` (ISO timestamp)
  
  Cheap to produce, makes the linkage between the Zenodo record, the git
  commit, and the rendered site auditable from inside the deposit itself.

## Workflows

Two thin files, both calling reusable workflows from `isp-actions-config`
once stable.

### `prepare-zenodo.yml` (workflow_dispatch only)

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        required: true
        description: Release version (e.g. 1.0.0, no leading v)
      sandbox:
        required: false
        type: boolean
        default: false
        description: Use sandbox.zenodo.org
permissions:
  contents: write
  pull-requests: write
jobs:
  prepare:
    # 1. checkout
    # 2. setup python
    # 3. run zenodo-deposit.py prepare --version <version> [--sandbox]
    # 4. peter-evans/create-pull-request@v6 with the myst.yml change
```

### `publish-zenodo.yml` (tag push)

```yaml
on:
  push:
    tags: ['v*']
jobs:
  publish:
    # 1. checkout (the tag)
    # 2. micromamba env (matches deploy-paper.yml)
    # 3. myst build --all
    # 4. assemble _bundle/ (incl. publication-provenance.json)
    # 5. zenodo-deposit.py publish --tag ${{ github.ref_name }}
    #    (validation gates run here — fail fast if tag/DOI/artifacts wrong)
    # 6. comment draft URL on the tag's commit
```

Sandbox vs production token selection:

- **Prepare** takes an explicit `sandbox` workflow input. The first
  deposit has no `project.doi` yet, so there's nothing to infer from —
  and we want explicit control while testing the first deposit anyway.
- **Publish** (tag push) reads `project.doi` prefix — `10.5072/...`
  ⇒ `ZENODO_SANDBOX_TOKEN`, `10.5281/...` ⇒ `ZENODO_TOKEN`. Avoids
  needing a workflow input on tag push.

## Decisions locked in

1. ✅ Tag convention: semver `vMAJOR.MINOR.PATCH`. Maintainer tags after merge.
2. ✅ Draft + human-confirm publish. Publish is irreversible; one click is cheap.
3. ✅ Concept DOI only in `myst.yml`. Version DOIs surface only on Zenodo.
4. ✅ ORCID: send if present. No `myst.yml` schema change.
5. ✅ No Zenodo Community.
6. ✅ Idempotent: lookup by `related_identifiers` GitHub URL.

## Testing on this repo

This repo is the testbed.

1. Use `https://sandbox.zenodo.org` first — same API, fake DOIs (`10.5072/...`),
   periodic resets. Token from sandbox.zenodo.org/account/settings/applications/.
2. Store token as repo secret `ZENODO_SANDBOX_TOKEN`.
3. Run `prepare-zenodo.yml` via `workflow_dispatch`. Verify:
   - Draft appears on sandbox with correct metadata.
   - PR opens with `doi: 10.5072/zenodo.<id>` added to `myst.yml`.
4. Merge PR. Push tag `v0.1.0`.
5. Verify `publish-zenodo.yml` ran: bundle uploaded, draft URL posted.
6. Click Publish on sandbox. Verify version DOI resolves.
7. Push `v0.1.1`. Verify newversion path: no PR, new draft, same concept DOI.
8. Once happy: switch to production token (`ZENODO_TOKEN`), drop `--sandbox`,
   move the script + workflows to `isp-actions-config`, ship.

## Open items

- **Sandbox abandonment:** orphan drafts pile up on sandbox during testing.
  Cleanup is manual via UI; not a blocker.
- **Bundle file rules:** the "everything at root not in build/.git" rule is
  a guess based on existing papers. If a paper has data in subdirectories
  we'd want to revisit. For now: explicit `downloads:` entries in `myst.yml`
  could override the heuristic.
- **`myst.yml` rewrite mechanics:** preserving comments and key order matters
  for diff readability in the PR. Use `ruamel.yaml` (round-trip safe) rather
  than `pyyaml`.
- **Tag-on-fork:** if authors are working from forks, the tag-triggered
  workflow runs on the fork, not the canonical repo. Tagging convention
  needs to be: maintainer tags `impact-scholars/<repo>` directly after merge.
  Document this.

## Possible enhancements

- **Auto-publish flag.** Add `--auto-publish` to `publish` (and a matching
  `workflow_dispatch` input on the tag workflow) so trusted runs can
  skip the manual click on Zenodo. Off by default; only the explicit
  flag path publishes. Useful once the flow is well-trusted; not needed
  for initial rollout.

## Not in scope

- Backfilling the 9 existing 2023 papers (deferred — they already have DOIs).
- Mapping CRediT roles into Zenodo contributors.
- Cross-paper aggregation / journal-level DOIs.
