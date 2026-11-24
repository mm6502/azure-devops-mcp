---
title: "Fork management: branches, npm release and installation"
module: build-and-deploy
date: 2026-06-10
problem_type: best_practice
component: tooling
severity: medium
applies_when:
  - Maintaining the on-prem fork of microsoft/azure-devops-mcp
  - Releasing a new version of @mm6502/azure-devops-mcp
  - Changing how colleagues install or update the server
tags:
  - fork-management
  - npm
  - trusted-publishing
  - upstream-sync
  - dist-commit
---

# Fork Management: Branches, npm Release and Installation

## Context

`mm6502/azure-devops-mcp` is a fork of `microsoft/azure-devops-mcp` that adds on-prem Azure DevOps Server support (SSPI/PAT). It is published to npm as **`@mm6502/azure-devops-mcp`** with its own version series that starts at 1.0.0 and is independent of upstream versions. Upstream targets the cloud API, so it is never synced automatically: an automatic sync would overwrite the on-prem API 7.1 versions.

## Branches

| Branch     | Purpose                                                              |
| ---------- | -------------------------------------------------------------------- |
| `main`     | Our code and the default branch. Releases are tagged here.           |
| `upstream` | Mirror of `microsoft/azure-devops-mcp` `main`. Updated by hand only. |

The former install target `ditec` was removed on 2026-11-24 (it had been an alias of `main`). `github:...#ditec` links no longer work; use the npm package or `github:mm6502/azure-devops-mcp#v<version>`.

Local remotes: `github` (`mm6502/azure-devops-mcp`) and `microsoft` (`microsoft/azure-devops-mcp`).

**Do not use "Sync fork" on GitHub.** It compares by branch name with `microsoft:main`, and "Discard commits" would delete our work.

Update the mirror by hand (always a fast-forward, `upstream` has no commits of its own):

```bash
git fetch microsoft main
git push github microsoft/main:refs/heads/upstream
```

Then rebase `main` onto `upstream` as described in [rebasing-on-prem-fork-onto-upstream.md](../conventions/rebasing-on-prem-fork-onto-upstream.md).

## Releasing

Workflow `.github/workflows/release.yml` publishes on a pushed tag `v*` with npm trusted publishing (OIDC) and provenance. No npm token is stored in the repository.

1. Set the version without a git tag: `npm version <x.y.z> --no-git-tag-version`. This updates `package.json` and `npm-shrinkwrap.json` together. Set the same version by hand in `server.json` (`npm version` does not touch it; it is not part of the tarball and not published, but keep it in sync).
2. `npm run build` regenerates `src/version.ts` and `dist/`. `dist/` is committed (direct GitHub installs and the CI version check need it).
3. Run `npm run validate-tools`, `npm test` and `npm run format-check`; `npm pack --dry-run` must list `dist/`, `npm-shrinkwrap.json`, `LICENSE.md`, `README.md` and `package.json`.
4. Commit and push `main`.
5. Push **one** tag: `git tag -a v<x.y.z> -m v<x.y.z> && git push github refs/tags/v<x.y.z>`. The tag must equal the version in `package.json`. Never push tags in bulk and never move or delete a pushed tag; a failed run is fixed forward with the next patch version.
6. Check the run of "Publish Package", then `npm view @mm6502/azure-devops-mcp version` and the provenance on the package page. Note the upstream base commit in the release notes.

Things that bit us once:

- `release.yml` pins `npm@11.19.1`. npm 12 refuses `npm ci` when only `npm-shrinkwrap.json` exists, so `npm@latest` makes the release fail before the build.
- Trusted publisher is configured on npmjs.com (user `mm6502`, repository `azure-devops-mcp`, workflow `release.yml`, direct `npm publish` allowed). A new configuration must complete a first release within 2 days.
- A tag ruleset (`protect-release-tags`) restricts creating, changing and deleting `v*` tags to the repository admin. Package publishing access is set to "Require two-factor authentication and disallow bypass 2fa tokens".
- The workflows "Nightly Pre-release" and "Update Package Version" are inherited from upstream and are **disabled** on GitHub (they would publish nightly versions and bump versions). After every rebase or mirror update check `gh workflow list --all` and review any new workflow with a `schedule` or publish trigger before pushing to `main`.
- `.github/dependabot.yml` ignores `win-sso`. Version 1.4.0 moved its native prebuild, which together with `ELECTRON_RUN_AS_NODE` broke startup for fresh installs. The version is pinned to `1.3.3` in `package.json`.

## Dependencies: `npm-shrinkwrap.json` instead of `package-lock.json`

npm ignores `package-lock.json` for git installs and for dependents of a published package. `npm-shrinkwrap.json` is honored, so it replaces the lock file (`npm install --package-lock-only && npm shrinkwrap`) and is listed in `files` of `package.json`. After a rebase regenerate it instead of resolving conflicts by hand. `.github/workflows/build.yml` checks it for drift.

## Installing

The published package contains only the compiled `dist/`, but npm still has to install about 270 production dependencies on the first start (measured on Windows: 35 s to 98 s cold, about 3 s warm; for comparison `enterprise-architect-mcp` with four direct dependencies installs in about 5 s). Bundling the server into one file would shorten this; it is not done yet.

Default for colleagues, updates automatically (one registry request per start, 0.1 to 0.5 s):

```json
{
  "servers": {
    "ado": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@mm6502/azure-devops-mcp@latest", "https://your-server/tfs/YourCollection", "--authentication", "sspi"],
      "env": { "NODE_OPTIONS": "--use-system-ca" }
    }
  }
}
```

Alternatives:

- **Pinned version** (`@mm6502/azure-devops-mcp@1.0.1`): no surprise updates; change the number to update. Use it when a new release breaks something.
- **Global install** (`npm i -g @mm6502/azure-devops-mcp`, command `mcp-server-azuredevops`): fastest start, but updates only when someone runs the install again.
- **Direct from GitHub** (fallback): `github:mm6502/azure-devops-mcp#v<version>`. Pin it to a tag, not to a branch.

`npx` does not work from a directory whose own `package.json` has the same name (this repository): it uses the local project and fails to find the binary. Use `node <clone>\dist\index.js` for development.

## Old installation link

`github:DITEC-Mracka/azure-devops-mcp#ditec` stopped working when the `ditec` branch was removed. The old repository name still redirects to `mm6502/azure-devops-mcp`, so `github:DITEC-Mracka/azure-devops-mcp#v<version>` works. Never create a repository named `azure-devops-mcp` under `DITEC-Mracka`; that would break the redirect.

A published tag can point at a commit that is no longer in the history of `main`: `v1.0.1` points at the commit the package was built from (its provenance names that commit), while `main` was squashed afterwards. Never move such a tag.
