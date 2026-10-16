# <img src=".github/logo.svg" alt="" height="32"> RasBeacon plugin template

Start here to make plugins for [RasBeacon](https://github.com/aazam476/RasBeacon),
the Raspberry Pi travel router. Click **Use this template** to get your own
copy, with a small working plugin and everything to build, sign and publish
plugins.

- **New to RasBeacon plugins?** Follow the [guide](docs/guide.md): your first
  plugin on your Pi in a few minutes, then step by step to a full plugin.
- **Looking up a manifest field or a permission?** The reference is
  [RasBeacon's docs/plugins.md](https://github.com/aazam476/RasBeacon/blob/main/docs/plugins.md).
- **Real plugins to read:** [RasBeacon-plugins](https://github.com/aazam476/RasBeacon-plugins)
  (AdGuard Home, Samba, Argon ONE V2), built the same way as this template.
- **An AI agent working here?** [AGENT.md](AGENT.md) has all the context.

## What's inside

```
plugins/hello-world/   the starting plugin: a web page with one setting
  manifest.json        what the plugin is and needs
  icon.png             its icon (256×256; the source is icons/hello-world.svg)
  README.md            shown on its page in RasBeacon
  Containerfile        its container image
  templates/           files RasBeacon fills in from its settings
icons/                 the icons' SVG sources
tools/build.sh         builds every plugin into dist/<id>.rba, and the catalog
tools/catalog.py       writes dist/rasbeacon-catalog.json
tools/rbsign.sh        RasBeacon's rbsign: signing keys, signing, checking
.github/workflows/     CI (build and check), Release (sign and publish)
docs/guide.md          the guide
AGENT.md               context for AI agents
```

## Quick start

On Linux with Docker (buildx), Go 1.24 or newer, git, zip and Python 3 (an
arm64 machine, like a Raspberry Pi 5 or a cloud arm64 machine, builds
fastest; elsewhere Docker emulates arm64):

```sh
tools/build.sh                            # dist/hello-world.rba, not signed
```

On RasBeacon, open **Plugins → Install from a file**, choose
`dist/hello-world.rba`, and install it. Its page has an **Open** button.

## Publishing

1. **Make a signing key, once:** `tools/rbsign.sh plugin-keygen`. Keep the
   private key safe (a password manager), never commit it, and add it as the
   repository secret `RASBEACON_PLUGIN_SIGNING_KEY` (Settings → Secrets and
   variables → Actions). Keep the same key forever: RasBeacon accepts updates
   only when they are signed with the key of the installed version.
2. **Publish a release:** Actions → Release → Run workflow, with a release
   name such as `v0.1.0`. It builds, signs and publishes every plugin. Name
   releases after the RasBeacon version they're for (`v0.1.0`, `v0.1.1`, …
   for RasBeacon 0.1), as RasBeacon's own plugins do; RasBeacon itself only
   reads the newest release, whatever its name.
3. **Use it:** on RasBeacon, add your repository under **Plugins → Browse
   plugins → Sources** as `owner/repository`.

Bump a plugin's `version` whenever it changes: RasBeacon offers an update
only for a newer version. If a plugin uses something only a newer RasBeacon
has, set `minRasBeacon` in its manifest (like `"0.2.0"`).

## License

Apache-2.0 (see [LICENSE](LICENSE)). Your plugins in your copy can use any
license you like; change LICENSE if you want.
