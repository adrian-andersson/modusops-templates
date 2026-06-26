# modusops-templates

Central, versioned library of [modusOps](https://github.com/adrian-andersson) pipeline templates.

This repo is a **source, not a live dependency**. The modusOps tooling pulls a template's YAML from a
GitHub Release and **writes it as a local file** into the consumer's repo (vendor-at-fetch). Nothing
here is referenced at pipeline compile- or run-time — the privileged template is reviewed in the
consumer's own PR and pinned (version + SHA256) in a `.modusops.lock` file.

## Layout

```text
templates/
  azd/   Azure DevOps templates  — one file per template (`templates/azd/<name>.yml`)
  gh/    GitHub templates        — composite-action DIRECTORIES (`templates/gh/<name>/action.yml`)
manifest.json   Authoritative name -> asset map (the tooling resolves via this, not by parsing filenames)
.github/workflows/
  prValidation.yml   Lints azd + gh templates on PRs (structure / parameter / gh-action / secret errors gate merge)
  release.yml        SemVer tag on merge + attach the template assets (+ checksums.txt) to a GitHub Release
  tag.yml            Cuts single-int `vN` ref tags so gh consumers can pin `...@vN`
```

## Asset shape (per platform)

A GH composite action is a **directory** (`action.yml` + optionally scripts), not a single file, so the
two platforms ship differently:

| Platform | Source | Release asset | Integrity anchor |
| --- | --- | --- | --- |
| `azd` | `templates/azd/<name>.yml` | `azd.<name>.yml` (single file) | file SHA256 |
| `gh` | `templates/gh/<name>/action.yml` (+ any sidecars) | `gh.<name>.zip` (zipped directory) | **canonical tree hash** |

> **Why a tree hash, not the zip's SHA?** Zip archives are not byte-reproducible (timestamps, entry
> ordering), so the zip's own SHA can't be recomputed by a consumer from the expanded files. The drift
> anchor is therefore a **canonical tree hash**: for every file in the action dir, take its repo-relative
> POSIX path + its SHA256, sort by path, concatenate as `"<relpath>\n<sha>\n"` and SHA256 the result.
> `release.yml` writes this value to `checksums.txt`; the consumer tooling recomputes it at `Add`/`Test`
> time. (Single-file `azd` assets just use the file SHA256.)

This is **Option A** (directory/archive assets) from `azdConcepts/TemplateLibrary.md` — chosen so gh
templates keep their honest composite-action shape rather than being squeezed into a single file.

## Naming convention

- **Release asset:** `{platform}.{name}.{ext}` — `platform` is `azd` or `gh`; `name` is camelCase with
  **no dots** (the dot is reserved as the field separator); `ext` is `yml` (azd) or `zip` (gh).
  Examples: `azd.registerModusOpsFeeds.yml`, `gh.registerModusOpsFeeds.zip`.
- **Vendored local shape:** `azd` → `<name>.yml`; `gh` → `<name>/` (the zip expands into a directory).
  The platform prefix is dropped once it's in the consumer repo.
- **Channels are versions, not filenames.** A preview/prerelease template is the same asset published
  at a prerelease library tag (`vX.Y.Z-preview.N`), recorded in the lockfile. There is no `.preview` asset.

## Current templates

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

- **Whole-library SemVer.** Each merge to `main` cuts a patch release (minor/major via a manual run).
  The version is the *set-level* compatibility coordinate; pin it in the consumer lockfile.
- **Full template set is attached to every release** (self-contained, safe to prune old releases).
- **Release notes list only what actually changed** (diffed against the previous tag), so an unchanged
  template never looks like it was modified.
- Integrity anchor is the **SHA256 recorded in the consumer's `.modusops.lock`**, not the tag (GitHub
  releases are mutable).

## License

MIT — see [LICENSE](LICENSE).
