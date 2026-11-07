# Install

oUpdate is a Bun and TypeScript tool. It needs [Bun](https://bun.sh); there is no build step.

Install the latest release tag from the [release page](https://github.com/akasecurity/oUpdate/releases), not the tip of `main`, and check that `git describe --tags` prints that tag:

```
git clone --branch <latest release tag> https://github.com/akasecurity/oUpdate.git
cd oupdate
git describe --tags
bun install
bun src/cli.ts review npm ms 2.1.2 2.1.3
```

`bin` in `package.json` maps `oupdate` to `src/cli.ts`, so `bun link` gives you an `oupdate` command.

### What it runs

oUpdate has no runtime dependencies; `bun.lock` holds only the dev tools (TypeScript and Bun's types). `review` and `gate` download release archives from the registry and read them with `bsdtar`; they never install a package or run its code, and the optional GuardDog layer runs in the container you give it. The network calls go to the package registries, OSV, your reviewer endpoint and, for `gate`, your forge. `image` reads two images' manifests and config blobs from their container registry and never pulls a layer or runs the image. Nothing runs as root unless you schedule `host --apply` as a LaunchDaemon yourself; `host --apply --dry-run` runs every check and changes nothing on disk.

### Set up with your agent

Paste this into your coding agent:

```
Set up oUpdate (https://github.com/akasecurity/oUpdate) on this machine for me. Clone it, read its AGENTS.md and follow it step by step. Ask me before anything that changes my system.
```

[AGENTS.md](https://github.com/akasecurity/oUpdate/blob/main/AGENTS.md) gives the agent the order to work in:

- what you want oUpdate for;
- prerequisites, then the install;
- a local or hosted reviewer model, checked on a known-clean update;
- a dry run before any live run;
- scheduling only with your approval.
