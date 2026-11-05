# oUpdate

oUpdate is an open-source package firewall that reads what each new release changed before it lands. Deterministic checks look for known attack patterns, such as install scripts, Python startup hooks and encoded payloads, and a model you choose (a local model you run, or a cloud one) reads the rest of the diff. On a Mac it handles Homebrew and npm updates; in your repositories it gates dependency-update pull requests with a cooldown, the same review and a commit status your own gate can act on. It is tested against real supply-chain attacks without any threat feeds or advisories.

Source, releases and the evaluation are in the [oUpdate repository](https://github.com/akasecurity/oUpdate).

## What it covers

| Where | What oUpdate reviews | Command |
|---|---|---|
| One release, on demand | an npm or PyPI package update | `oupdate review` |
| One container image, before you repin it | the old and new image's Entrypoint, Cmd and build history, from any registry that allows anonymous pulls | `oupdate image`, see [Check one image](https://github.com/akasecurity/oUpdate#check-one-image) |
| Update PRs in your repos | npm packages in `bun.lock`, `package-lock.json`, `npm-shrinkwrap.json` and `pnpm-lock.yaml`; PyPI packages in `uv.lock`, `poetry.lock` and pinned requirements files; on GitHub, Gitea and API-compatible forges | `oupdate gate` |
| A Mac | Homebrew formulae and casks, global npm packages | `oupdate host --apply` |
| Images, VMs | Docker Compose images, OS packages on Google Cloud VMs, only through a credential broker | `oupdate targets`, see [OS targets](https://github.com/akasecurity/oUpdate#os-targets) |

oUpdate is not a hook on your package manager: it reviews the updates its own commands take.

**Reading a verdict.** Treat a FLAG or a hold as a stop until a person has looked. Treat an APPROVE as one piece of evidence, not clearance: the model reads the release diff and can be wrong, and a release too large for the review budget is held for a person rather than judged.

Early release. The easiest way to install it is to ask your coding agent to set it up; see [Set up with your agent](install.md#set-up-with-your-agent).
