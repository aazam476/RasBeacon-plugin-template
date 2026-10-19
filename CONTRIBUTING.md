# Contributing to the RasBeacon plugin template

Thanks for helping! Like RasBeacon itself, everything here is written to be
readable by a junior engineer or a second- or third-year computer science
student. If something in the template or the guide is hard to follow, that is
a bug worth reporting.

This repository is the starting point for other people's plugins, so changes
should keep it small: one simple plugin, the tools, and the guide. Plugins
with real features belong in
[RasBeacon-plugins](https://github.com/aazam476/RasBeacon-plugins) (or your
own repository).

## What to change where

- **The guide** (`docs/guide.md`) and the **README**: for people making
  their first plugin. Keep the steps in the order someone needs them.
- **AGENT.md**: everything an AI agent needs to make a plugin. When a
  manifest field or rule changes in RasBeacon, update its tables.
- **The format itself** (manifest fields, permissions, checks) is defined by
  RasBeacon's code and its
  [docs/plugins.md](https://github.com/aazam476/RasBeacon/blob/main/docs/plugins.md):
  change it there first.
- **`tools/` and `.github/workflows/`**: keep them the same as in
  RasBeacon-plugins, so both build and publish plugins the same way.

## Everyday commands

```sh
tools/build.sh                           # every plugin into dist/ (not signed)
tools/rbsign.sh plugin-check dist/*.rba  # check files as RasBeacon does
```

CI (`.github/workflows/ci.yml`) builds and checks every plugin on each push
and pull request.

## Commit messages and pull requests

Start the subject with a type prefix and a colon, then a short,
plain-language description in the imperative, such as
`docs: explain volumes in the guide`. Don't add a scope in parentheses (`docs: …`,
not `docs(guide): …`).

| Prefix | Use for |
|---|---|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `chore:` | Tooling, config, dependencies |
| `refactor:` | Code change that isn't a fix or feature |
| `docs:` | Documentation only |
| `style:` | Formatting, no logic change |
| `test:` | Adding or fixing tests |

One change per pull request. For a change to the hello-world plugin, say
how you tested it on a Pi.

## Signing

Releases are signed by the Release workflow with the
`RASBEACON_PLUGIN_SIGNING_KEY` secret (README.md, *Publishing*, says how to
make it). Never commit a private key, and never put one on a command line.
