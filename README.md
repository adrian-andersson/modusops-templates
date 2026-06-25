# modusops-templates

Central, versioned library of [modusOps](https://github.com/adrian-andersson) pipeline templates.

This repo is a **source, not a live dependency**. The modusOps tooling pulls a template's YAML from a
GitHub Release and **writes it as a local file** into the consumer's repo (vendor-at-fetch). Nothing
here is referenced at pipeline compile- or run-time — the privileged template is reviewed in the
consumer's own PR and pinned (version + SHA256) in a `.modusops.lock` file.

## Layout

```text
templates/
  azd/   Azure DevOps templates (compile-time `- template:` includes)
  gh/    GitHub templates (composite actions) — placeholder until GH support lands
manifest.json   Authoritative name -> asset map (the tooling resolves via this, not by parsing filenames)
.github/workflows/
  prValidation.yml   YAML structure/spacing lint on PRs (structure errors gate merge)
  release.yml        SemVer tag on merge + attach the template assets to a GitHub Release
```

## Naming convention

- **Release asset:** `{platform}.{name}.yml` — `platform` is `azd` or `gh`; `name` is camelCase with
  **no dots** (the dot is reserved as the field separator). Example: `azd.registerModusOpsFeeds.yml`.
- **Vendored local name:** `{name}.yml` (the platform prefix is dropped once it's in the consumer repo).
- **Channels are versions, not filenames.** A preview/prerelease template is the same asset published
  at a prerelease library tag (`vX.Y.Z-preview.N`), recorded in the lockfile. There is no `.preview.yml`.

## Current templates (azd)

| Name | Purpose |
| --- | --- |
| `registerModusOpsFeeds` | Vault + `CredentialInfo` feed registration (the primary credential model). |
| `installModusOpsModules` | Credential-agnostic module install + `always()` teardown. |
| `sendTeamsChannelMessage` | Teams channel webhook notification. |
| `sendDiscordChannelMessage` | Discord channel webhook notification. |

> Templates stay **thin** — orchestration only. Real logic lives in versioned PowerShell modules
> (e.g. `ModusOps.Toolkit`) pulled at runtime, not in this YAML.

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
