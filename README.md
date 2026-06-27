# modusops-templates

Central, versioned library of [modusOps](https://github.com/adrian-andersson) **pipeline templates and
repo scaffolding**.

This repo is a **source, not a live dependency**. The modusOps tooling pulls an asset from a GitHub
Release and **writes it as a local file** into the consumer's repo (vendor-at-fetch). Nothing here is
referenced at pipeline compile- or run-time — the privileged asset is reviewed in the consumer's own PR
and pinned (version + hash) in a `.modusops.lock` file.

## Two categories

Every asset has a **`category`** that decides *how it's consumed*:

- **`pipeline`** — a building block you *compose into* a pipeline (register / install / notify). Vendored
  into a consumer-chosen templates dir.
- **`repoScaffold`** — repo *furniture* you *stamp into* a repo at a **fixed dest** under `.github/`
  (CI workflows, PR/issue templates). These are **dogfooded** — the assets shipped are this repo's own
  `.github/` files.

## Layout

```text
templates/
  azd/   pipeline templates  — one file per template (`templates/azd/<name>.yml`)
  gh/    pipeline composite actions — DIRECTORIES (`templates/gh/<name>/action.yml`)
.github/   this repo's OWN CI + furniture, ALSO shipped as repoScaffold assets (dogfooded):
  workflows/prValidation.yml   lints azd + gh templates on PRs (structure / parameter / gh-action / secret → gate)
  workflows/release.yml        cuts the next rolling-integer tag (vN) on merge + attaches the full asset set
                               (+ checksums.txt + manifest.json). The same vN is the gh `...@vN` ref pin.
  PULL_REQUEST_TEMPLATE.md     pr template     |   ISSUE_TEMPLATE/   issue-form set
manifest.json   Authoritative index: per-template category / kind / asset / dest, plus the `sets`
                (archetypes). The tooling resolves via this, not by parsing filenames.
```

## Asset shapes & integrity

Integrity is **always a single file SHA256** — there is no multi-file/tree path. An asset is either a
single file, or a composite-action directory whose anchor is its inner `action.yml`. The consumer
tooling recomputes the anchor at `Add`/`Test` time; `release.yml` writes the same values to
`checksums.txt`.

| Shape | Examples | Release asset | Integrity anchor |
| --- | --- | --- | --- |
| **single file** | azd template, gh workflow, PR template, one issue form | `<...>.yml` / `<...>.md` | **file SHA256** |
| **composite action** (dir with `action.yml`) | gh register / install / notify | `gh.<name>.zip` | `action.yml` **file SHA256** |

> **One file per asset.** Repo furniture that used to ship as a multi-file directory (the issue-template
> set) is now split into **one asset per file** — e.g. `templatesRepoIssue` (the form) and
> `templatesRepoIssueConfig` (`config.yml`). This keeps every asset on the plain file-SHA path: drift is
> a one-file hash compare, and a shared dest dir like `.github/ISSUE_TEMPLATE/` never causes false drift
> from a sibling file. The earlier canonical-tree-hash path has been retired.

Composite-action zips are **Option A** (directory/archive assets) from `azdConcepts/TemplateLibrary.md`
— chosen so gh actions keep their honest directory shape rather than being squeezed into a single file.

## Naming convention

- **Release asset:** `{platform}.{name}.{ext}` for `pipeline` assets;
  `{platform}.{kind}.{name}.{ext}` for `repoScaffold` assets (the `kind` — `workflow` / `prTemplate` /
  `issueTemplate` — disambiguates the furniture). The `name` of a `repoScaffold` asset is **prefixed
  with its repo type** (`templatesRepo*`) so you can tell which kind of repo it furnishes. `platform` is
  `azd`/`gh`; `name` is camelCase with **no dots** (the dot is the field separator); `ext` is `yml`/`md`
  (file) or `zip` (directory). Examples: `azd.registerModusOpsFeeds.yml`,
  `gh.registerModusOpsFeeds.zip`, `gh.workflow.templatesRepoRelease.yml`.
- **Vendored shape:** a single-file asset lands at its dest as-is; a `.zip` expands into a directory.
  `pipeline` assets vendor into the consumer's templates dir (the platform prefix is dropped);
  `repoScaffold` assets vendor to their **fixed `dest`** (`.github/workflows/<name>.yml`, etc.).
- **Channels are versions, not filenames.** Pin a `vN` in the lockfile; never blind "latest". (A
  prerelease channel under rolling-integer versioning is an open question — see
  `azdConcepts/TemplateLibrary.md`.)

## Current assets

### Pipeline templates

| Name | azd | gh | Purpose |
| --- | :-: | :-: | --- |
| `registerModusOpsFeeds` | ✅ | ✅ | Vault + `CredentialInfo` feed registration (the primary credential model). |
| `installModusOpsModules` | ✅ | ✅ | Credential-agnostic module install + `always()` teardown. (gh inlines the GitHub Packages folder-casing fix.) |
| `sendTeamsChannelMessage` | ✅ | ✅¹ | Teams channel webhook notification. |
| `sendDiscordChannelMessage` | ✅ | ✅ | Discord channel webhook notification. |

> ¹ The gh Teams action is ported but **not yet exercised on a live Teams webhook** (no test account).

The gh templates are **composite actions** (not reusable workflows) so the register → install pair shares
one job — the per-run SecretStore vault built by `registerModusOpsFeeds` must survive into
`installModusOpsModules`, which only holds within a single job. The whole chain (register, private install
through the bound `GITHUB_TOKEN`, Discord notify, `@vN` pin) was validated on a hosted ubuntu runner; see
`azdConcepts/TemplateLibrary.md` (§ GitHub Actions portability / PoC result).

### Repo scaffolding (`repoScaffold`, gh)

These are this repo's own `.github/` furniture, shipped as assets and vendored to a fixed dest. The
names carry a **`repoType` prefix** (`templatesRepo`) so it's clear they furnish a *templates* repo —
a future `moduleRepo*` PR-validation would be a distinct, non-colliding asset:

| Name | Kind | Dest | Purpose |
| --- | --- | --- | --- |
| `templatesRepoPRValidation` | workflow | `.github/workflows/prValidation.yml` | PR-gate template linter. |
| `templatesRepoRelease` | workflow | `.github/workflows/release.yml` | Rolling-integer release + asset attach. |
| `templatesRepoPullRequest` | prTemplate | `.github/PULL_REQUEST_TEMPLATE.md` | Simple PR template (N+, no SemVer matrix). |
| `templatesRepoIssue` | issueTemplate | `.github/ISSUE_TEMPLATE/issue.yml` | Issue form (report / request a template). |
| `templatesRepoIssueConfig` | issueTemplate | `.github/ISSUE_TEMPLATE/config.yml` | Issue-chooser config (blank off + docs link). |

> The issue furniture is **two single-file assets**, not one directory set — see *Asset shapes &
> integrity* above. `config.yml` is GitHub's required exact filename (it was `_config.yml`).

### Sets (archetypes)

A **set** is a named bundle applied in one call with `Add-MORepoScaffold`. Two shapes, both declared
in `manifest.json` under `sets`:

| Set | Type | Members |
| --- | --- | --- |
| `pipelineCore` | archetype (curated, azd + gh) | just the load-bearing pair — `registerModusOpsFeeds` + `installModusOpsModules` (no furniture, no provisioning) |
| `templateLibrary` | archetype (curated, gh) | `templatesRepoPRValidation` + `templatesRepoRelease` + `templatesRepoPullRequest` + `templatesRepoIssue` + `templatesRepoIssueConfig` |
| `workflowSet` | selector (derived, gh) | every `repoScaffold` `workflow` (auto-includes new ones) |
| `azdOpsRepo` | archetype (azd, file + provision) | register/install pair, then branch-policy + build-service repo permission over REST |

A **curated** set is an explicit, tested-together list; a **selector** is a query over `category`/`kind`
evaluated against that manifest version (late-bound but deterministic — the version is lock-pinned).

A step is either a **file** (vendored) or, on azd, a **provision** step that runs a modusOps
provisioning cmdlet (branch policy, repo permission) — its args bound from the caller's `-With` +
context. Provision steps are limited to a fixed **allow-list** of cmdlets, so a vendored manifest can
never invoke an arbitrary command.

```powershell
Set-MOPlatform gh                                    # set the default once
Add-MORepoScaffold -Archetype pipelineCore           # just the register + install pair — quickest start
Add-MORepoScaffold -Archetype templateLibrary        # or stamp the whole templates-repo set, lock-pinned

# azd: vendor + provision in one call
Add-MORepoScaffold -Archetype azdOpsRepo -OrganizationUri https://dev.azure.com/contoso `
  -ProjectName modusOps -With @{ repo = 'modusOps'; buildId = 42 }
```

### gh consumer usage (sketch)

```yaml
on: workflow_dispatch
permissions: { contents: read, packages: read }
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: OWNER/modusops-templates/templates/gh/registerModusOpsFeeds@v1   # composite => same-job vault
        with:
          feeds: '[ { "name": "modusops", "uri": "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json" } ]'
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: OWNER/modusops-templates/templates/gh/installModusOpsModules@v1
        with:
          modules: '[ { "name": "Az.Accounts", "repository": "MAR" }, { "name": "ModusOps.Toolkit", "repository": "modusops", "version": "1.0.1-prev001" } ]'
```

> The templates repo's **Actions access policy** must allow the consumer (Settings → Actions → General →
> Access) — the GitHub analog of Azure DevOps resource authorization.

Templates stay **thin** — orchestration only. Real logic lives in versioned PowerShell modules
(e.g. `ModusOps.Toolkit`) pulled at runtime, not in this YAML.

## Versioning & releases

- **Whole-library rolling integer (N+).** Each merge to `main` cuts the next single-integer tag
  (`v1`, `v2`, `v3`, …); the first release is `v1`. The version is the *set-level* compatibility
  coordinate; pin it in the consumer lockfile. The same `vN` tag is also the immutable ref for gh
  consumers who pin cross-repo (`...@vN`) — one tag scheme, not two. (SemVer was retired: the library
  is a curated set, so one monotonic coordinate is simpler than per-component versions.)
- **Full template set is attached to every release** (self-contained, safe to prune old releases).
- **Release notes list only what actually changed** (diffed against the previous tag), so an unchanged
  template never looks like it was modified.
- Integrity anchor is the **SHA256 recorded in the consumer's `.modusops.lock`**, not the tag (GitHub
  releases are mutable).

## License

MIT — see [LICENSE](LICENSE).
