# Gate update PRs on a forge

Run after your update bot (Renovate or Dependabot): for each open PR the bot opened on a `renovate/` branch of the repo, oUpdate diffs the lockfiles between base and head and reviews every package version the PR brings in. It then posts a commit status `oupdate/review` and one comment carrying the per-layer verdicts.

- **Branches:** `--branch-prefix` (repeatable) replaces the `renovate/` default, for example `--branch-prefix dependabot/ --branch-prefix renovate/`.
- **Authors:** only a PR opened by the update bot from a branch of the repo itself is gated; a fork PR, or one anyone else opened, is not, whatever its branch name. `--bot-login` (repeatable) replaces the default logins `renovate[bot]` and `dependabot[bot]` (the GitHub apps), for example `--bot-login renovate` for a self-hosted Renovate on Gitea. Each PR not gated is logged with the reason (`PR #7 skipped: author renovate is not a bot login ...`), so a login that does not match shows up in the run's output.
- **Gate identity:** the gate trusts a success or failure on a head, and edits a comment carrying its marker, only when its own token posted it. It asks the forge who the token is (`GET /user`); `--gate-login` names it instead, and is needed for the Actions `GITHUB_TOKEN`, which cannot read `/user`: `--gate-login "github-actions[bot]"`.
- **Forge:** `--forge gitea` (the default) needs `--endpoint`. `--forge github` talks to `https://api.github.com`, or to `$GITHUB_API_URL` or `--endpoint` on GitHub Enterprise Server; see [On GitHub](#on-github).
- **Token:** `--token-file FILE`, or `--token-env NAME` to read it from an environment variable.
- **Lockfiles:** for npm, `bun.lock`, `package-lock.json` and `npm-shrinkwrap.json` (lockfileVersion 2 and 3), and `pnpm-lock.yaml` (v5, v6 and v9); for PyPI, `uv.lock` and `poetry.lock`. Without `--lock`, the gate reads each of these names that the PR's head has at the repo root. `--lock PATH` (repeatable) names them instead, for a lockfile in a subdirectory, say. All lockfiles go into one status; a version bumped in two of them is reviewed once when both pin it from the same place (registry, source, integrity); a copy from git, a URL, a file, another registry or with other bytes is handled on its own.
- **Requirements files:** a pinned `requirements*.txt` or `*.lock.txt` (pip-compile output, for example) is read only when `--lock` names it, since the name alone does not say a file is pinned. Only exact `name==version` lines count as registry pins, and editables and `name @ url` lines are direct dependencies (below). Any other line (a `-r` or `-c` include, a range, a wildcard, `===`, a bare URL or path, an option oupdate does not know) holds the PR at pending, as a file that cannot be read. An `--extra-index-url` or `--find-links` (`-f`) marks every pin as from a private registry, since pip may take any of them from there. Python package names are compared in their PEP 503 form, so `Typing_Extensions` and `typing-extensions` are one package.
- **Git, url and file dependencies:** they have no registry release to review, so a new or changed one (a git repo, a tarball URL, a local file or folder, a requirements `name @ url` line or editable) fails the PR, its source named in the comment. In `package-lock.json` and `npm-shrinkwrap.json`, only bundled copies (`inBundle` inside a registry package's own `node_modules`, shipped in its reviewed tarball) are skipped; the root's or a workspace's bundled dependencies are fetched from the registry and reviewed like any other, and an entry with a version but no `resolved` is a default-registry package, reviewed like any other; links and local folders (workspace packages, `../x`) are file dependencies too: an unchanged one is no change, a new or changed one fails. Workspace packages in `bun.lock` and `pnpm-lock.yaml`, and uv's editable and virtual entries (the project's own packages), are skipped. In `pnpm-lock.yaml`, an importer dependency resolved to `link:` or `file:` is a file dependency too, unless the link is to another importer (a workspace package); a manifest's registry spec that its importer records as a local link or file fails as not recorded. A `pnpm-lock.yaml` registry entry with neither a tarball nor an integrity holds the PR at pending, as a lockfile that cannot be read.
- **Integrity:** when the lockfile records an integrity, the download is checked against it, so the review reads the exact bytes the install fetches. A mismatch holds the package. For `package-lock.json`, `npm-shrinkwrap.json`, `bun.lock` and `pnpm-lock.yaml`, every hash the lockfile records is checked (sha1, sha256, sha384 and sha512); an integrity of another algorithm or a malformed hash holds the PR at pending as a lockfile that cannot be read. A `resolved` URL, pnpm `tarball` or `bun.lock` tarball that is not the registry's own tarball for that name and version (`<name>/-/<name>-<version>.tgz`, with no query; another package's or another version's tarball) is a URL dependency and fails the PR, since a review of the version on the registry would not read it. For PyPI, the review reads every wheel and the sdist of both releases, since an installer picks the file for its platform, and checks each download against the sha256 PyPI lists for it; a file with no digest, a mismatch or a failed download holds the package. Files with the same bytes at the same path are read once, except a symlink or a `package.json`, which each artifact keeps so it resolves in that artifact. In `uv.lock`, every file locked for a new version (the sdist and each wheel) must be one the review reads: from PyPI, a file URL off `files.pythonhosted.org` is a URL dependency and fails the PR, and a sha256 that PyPI does not list for that name and version fails it too; a file with no URL or sha256, or a listing that cannot be fetched, holds the package. The lockfile's own Python sha256 hashes are compared for swaps (below). In `poetry.lock` and a PyPI requirements file, the same check runs on each file's sha256: a hash PyPI does not list for that name and version fails the PR, and a poetry file with no sha256 holds the package; a requirements pin with no `--hash` is not checked.
- **Swapped source or hash:** a version is compared with its source (resolved or tarball URL, registry or index) and every hash the lockfile records for it, not by name and version alone. The same version with another source or hash fails the PR, since a review of that version on the registry would not see the swap.
- **Private registry:** a version from a registry other than the public npm registry, or an index other than PyPI (a uv `registry` source, a poetry `legacy` source, or a requirements file's `--index-url`, `--extra-index-url` or `--find-links`), is not reviewed: the public registry and OSV know nothing about it. By default it fails the PR for a person to decide (`--private-registry hold`). `--private-registry list` is the opt-in to accept such versions unreviewed: they are listed in the comment as "not reviewed (private registry)" and do not change the status. `--private-registry review` reviews them against the public registry instead, which is a dependency-confusion blind spot.
- **What the gate cannot see fails the PR:** a PR that deletes a lockfile, or changes any file the gate did not read, gets failure with the files named. Only a lockfile the gate read, or a manifest it handles (below), counts as seen; everything else fails, whatever its name: other ecosystems' lockfiles and manifests (`yarn.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock` and the like), package manager and index config (`.npmrc`, `.yarnrc.yml`, `bunfig.toml`, `uv.toml`), a lockfile in a subdirectory that `--lock` does not name, Dockerfiles, compose, Helm and Terraform files, `.pre-commit-config.yaml`, `.nvmrc`, workflow files and any other file. A `package.json`, `pnpm-workspace.yaml` or `pyproject.toml` change counts as seen only when a read lockfile of that ecosystem covers its folder (the lockfile's own folder or one of its workspace members) and only dependency versions changed: the dependency maps, `overrides`, `resolutions` and `engines` in `package.json`; `catalog`, `catalogs` and `overrides` in `pnpm-workspace.yaml`; the project, optional, group and Poetry dependency lists in `pyproject.toml`. Each added or changed dependency must be a registry version, range or tag (a git, URL, file, link, workspace, catalog or user/repo spec, a path or tarball name such as `.`, `..`, `.x` or `x.tgz`, a PEP 508 `name @ url` or a Poetry git, path or url table fails; an `npm:<name>@<range>` alias counts when its range does and the lockfile diff brings in that package at a matching version, a Poetry `source` only when it is `pypi` or a source the `pyproject.toml` declared before the change, and an empty npm spec means any version), and every lockfile covering the folder must change too and, where it keeps a record of the project's own dependencies (`package-lock.json`, `bun.lock`, `pnpm-lock.yaml`, `uv.lock`), record the new spec. Where a lockfile keeps no such record (requirements files, `poetry.lock`, `overrides` and `resolutions`, pnpm's own `peerDependencies`), each such lockfile's own diff must bring in that dependency, at a version the spec allows where the gate can check it (an npm semver range, a PEP 440 specifier set such as `>=1,<3`, `~=2.2` or `==2.*`, or a Poetry constraint such as `^1.2`, `~1.2`, `1.2.*` or `>=1 || ^3`, each branch of a Poetry multiple-constraint list on its own; a prerelease, dev or local locked version, or a clause the gate cannot read, cannot be checked); a constraint-only change whose lockfile diff leaves that package alone passes only when a version the lockfile already locks meets the new spec as checked, so a spec the gate cannot check fails; `engines` is not checked against a lockfile. A change to anything else (lifecycle `scripts`, `main` or `bin`, `packageManager`, `[build-system]`, `[tool.uv.sources]`), an added, deleted or unparseable manifest, or one in a folder no read lockfile covers fails. Only a PR that changes no file passes with "no dependency files changed".
- **A lockfile that cannot be read** (an npm 6 `package-lock.json`, lockfileVersion 1, a `package-lock.json` entry with neither a `resolved` nor a registry version, or a file that does not parse) holds the PR at pending with the reason, never success.

```
oupdate gate --endpoint https://forge.example/api/v1 --repo owner/name \
  --token-file ~/.config/oupdate/forge-token \
  --reviewer '{"type":"openai","name":"local","baseUrls":["http://127.0.0.1:8080/v1"],"model":"your-model"}'
```

- **success:** every change lands.
- **failure:** a layer other than cooldown stopped one.
- **pending:** only cooldown or layer errors held it, or a forge error hit that PR (the other PRs are still gated); the next run reviews it again.

The advisory layer splits OSV entries in two:
- **Malicious** (an OpenSSF `MAL-` id or alias) always stops the update. The comment opens with a MALICIOUS line, because the useful next step is checking where that version is already installed.
- **Vulnerabilities** stop only the ones this update introduces. If the old version already had one, keeping the old version would not help, so it is listed as "already in" and does not stop the update. Vulnerabilities the update removes are listed as "fixes".

Heads that already have a success or failure this gate's token posted, and whose comment from the same token names that head and that verdict, are skipped. A status with the `oupdate/review` context posted by anyone else, or one from the gate's login without that comment (another workflow sharing the Actions `GITHUB_TOKEN` can post as `github-actions[bot]`), is not trusted, and the PR is reviewed again. A PR whose changes all read clean also gets the comment, so its verdict can be kept. GitHub, Gitea and API-compatible forges (Forgejo) are supported.

### On GitHub

The gate reports with a commit status, not a check run, so it works with the Actions `GITHUB_TOKEN` and with a personal access token alike, and branch protection can require `oupdate/review` like any other check. Lockfiles are read raw, so a `package-lock.json` over 1 MB is read in full.

Run it on a schedule from the repo it gates, with `.github/workflows/oupdate-gate.yml`:

```yaml
name: oupdate gate
on:
  schedule: [{ cron: "17 * * * *" }]
  workflow_dispatch: {}
permissions:
  contents: read
  statuses: write
  pull-requests: write
jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: oven-sh/setup-bun@<commit-sha>    # pin actions to a full commit sha
      - uses: actions/checkout@<commit-sha>
        with: { repository: akasecurity/oUpdate, ref: <release tag>, path: .oupdate }
      - run: >
          bun .oupdate/src/cli.ts gate --forge github --repo "$GITHUB_REPOSITORY"
          --token-env GITHUB_TOKEN --gate-login "github-actions[bot]"
          --branch-prefix dependabot/ --branch-prefix renovate/
          --reviewer "$OUPDATE_REVIEWER"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          OUPDATE_REVIEWER: ${{ vars.OUPDATE_REVIEWER }}
          REVIEWER_API_KEY: ${{ secrets.REVIEWER_API_KEY }}
```

`OUPDATE_REVIEWER` is a repository variable holding the reviewer spec, for example `{"type":"openai","name":"cloud","baseUrls":["https://api.example/v1"],"model":"your-model","apiKeyEnv":"REVIEWER_API_KEY"}`; `apiKeyEnv` names the secret the step passes in. A hosted runner cannot reach a model on your own machine; for a local model, use a self-hosted runner.

Why a schedule and not `pull_request`:
- A run that a Dependabot PR triggers gets a read-only token and no secrets, so it could not post the status or call the reviewer.
- The gate never checks out the PR's code; it reads the lockfiles through the API. A scheduled run on the default branch is safe.
- Cooldown holds need runs after the PR opened anyway: a pending PR is reviewed again on each run.
