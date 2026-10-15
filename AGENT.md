# AGENT.md

Context for AI agents working in this repository: what it is, how a RasBeacon
plugin is made, every rule RasBeacon checks, and how to build, test and
publish. Read it all before changing anything. People should read
[README.md](README.md) and [docs/guide.md](docs/guide.md) instead.

## What this repository is

A template for publishing plugins for **RasBeacon**, an open-source Raspberry
Pi travel router (<https://github.com/aazam476/RasBeacon>). A plugin is a
container that RasBeacon runs with Podman on the Pi. This repository holds the
plugins' sources (`plugins/<id>/`), builds each into a `.rba` file, and
publishes them as a GitHub release, which RasBeacon's catalog reads.

The authoritative reference is RasBeacon's
[docs/plugins.md](https://github.com/aazam476/RasBeacon/blob/main/docs/plugins.md),
and the code that checks plugins is
[internal/plugin/](https://github.com/aazam476/RasBeacon/tree/main/internal/plugin)
(`manifest.go`, `archive.go`). If this file and that code disagree, the code
wins; `tools/build.sh` always ends by checking with it.

Real plugins built the same way:
<https://github.com/aazam476/RasBeacon-plugins> (AdGuard Home: registry
image + config template + metrics; Samba: own image + USB drive + password;
Argon ONE V2: Go program + I²C + GPIO + power socket).

## Layout

```
plugins/<id>/manifest.json   required: the manifest (below)
plugins/<id>/icon.png        required: PNG, square, 256×256, at most 512 KB
plugins/<id>/README.md       optional, at most 64 KB: shown on its page
plugins/<id>/Containerfile   when the image is built here (then no "image" in the manifest)
plugins/<id>/templates/*     optional: files named in "templates"
plugins/<id>/...             anything the Containerfile copies (sources, scripts)
icons/<id>.svg               the icon's source
tools/build.sh               build every plugin → dist/<id>.rba, dist/<id>.png, dist/rasbeacon-catalog.json
tools/catalog.py             writes dist/rasbeacon-catalog.json from the manifests
tools/rbsign.sh              builds and runs RasBeacon's rbsign (keygen, sign, check)
.github/workflows/ci.yml     on push/PR: build + check every plugin (arm64 runner)
.github/workflows/release.yml  on tag v* or manual: build, sign, publish a release
docs/guide.md                the human guide
```

The folder name doesn't matter to RasBeacon; the manifest's `id` does. Keep
them the same.

## Commands

```sh
tools/build.sh                              # every plugin into dist/ (not signed); ends with plugin-check
tools/build.sh /path/to/rbsign              # same, with a given rbsign
tools/rbsign.sh plugin-check dist/*.rba     # check .rba files exactly as RasBeacon does before installing
tools/rbsign.sh plugin-keygen               # make a signing key (the owner does this, once)
```

Needs Linux, Docker with buildx, Go ≥ 1.24, git, zip, Python 3. Images must be
`linux/arm64`; on amd64 enable emulation once with
`docker run --privileged --rm tonistiigi/binfmt --install arm64`.

There is no way to run a plugin without a Raspberry Pi running RasBeacon.
`plugin-check` passing means RasBeacon will accept the file; whether the
program works needs a Pi (**Plugins → Install from a file**).

## The .rba file

A zip archive (renamed `.rba`), at most 2 GB, holding only these entries:

| Entry | Rules |
|---|---|
| `manifest.json` | required; at most 64 KB |
| `icon.png` | required; must be a PNG; at most 512 KB |
| `README.md` | optional; at most 64 KB |
| `image.tar` | the container image (`docker save` output, arm64) — exactly when the manifest has no `image` |
| `templates/<file>` | at most 20 files, 256 KB each; names `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`; every template the manifest lists must be there |
| `signature` | optional; written by `rbsign plugin-sign` |

Any other entry (or folder other than `templates/`) is refused.
`tools/build.sh` makes it: it copies `manifest.json`, `icon.png`,
`README.md` and `templates/`, builds the `Containerfile` (if any) with
`docker buildx build --platform linux/arm64` and `docker save`s it as
`image.tar`, zips with fixed timestamps, signs if
`RASBEACON_PLUGIN_SIGNING_KEY` is set, and runs `plugin-check`.

## manifest.json

JSON; unknown fields are refused. All fields:

| Field | Type | Rules |
|---|---|---|
| `id` | string | required; `^[a-z0-9][a-z0-9-]{0,30}[a-z0-9]$` (2–32); never change it after release |
| `name` | string | required; 1–40 characters |
| `version` | string | required; 1–40 characters, no spaces or `/`; RasBeacon offers an update when it is newer (numbers compared in order: `1.10.0` > `1.9.2`) |
| `description` | string | at most 300 characters |
| `author` | string | shown to the owner |
| `homepage` | string | must start `https://` |
| `boards` | string[] | `pi5`, `pi4`, `pi3`, `zero2w`; empty = all |
| `image` | object | `{ "ref": "registry/owner/name:tag", "digest": "sha256:<64 hex>" }`; no `@` in ref; only when there is no `image.tar`/Containerfile |
| `permissions` | string[] | from: `internet`, `lan-access`, `usb-drive`, `i2c`, `gpio`, `power-button`, `temperature` |
| `ports` | object[] | `name` (required), `container` (1–65535), `protocol` (`tcp` default, or `udp`), `default` (suggested port on RasBeacon; default = container), `open` (Open button for a web page), `https` (open with https://); no repeats |
| `memoryMb` | int | limit; default 256; at most 8192 |
| `cpus` | number | limit in cores; default 0.5; at most 4 |
| `minMemoryMb`, `minSwapMb` | int | refuse install on a Pi with less memory / swap |
| `storageMb` | int | size of its data file, shared by all volumes; default 256; at most 65536 |
| `volumes` | object[] | `name` (`^[a-z0-9][a-z0-9-]{0,30}$`, unique), `path` (absolute, not `/`, no `..`, `:`, `,`) |
| `usbDrive` | object | `path` (absolute), `optional` (bool); required together with the `usb-drive` permission |
| `env` | object | fixed environment variables; names `^[A-Z_][A-Z0-9_]{0,63}$` |
| `command` | string[] | replaces the image's command |
| `settings` | object[] | see below |
| `templates` | object[] | `file` (in `templates/`), `path` (in the container, inside a volume), `once` (write only if missing) |
| `afterInstall` | string | at most 1000 characters; placeholders allowed |
| `metrics` | object[] | at most 4; see below |
| `health` | object | `url` starting `http://{address}`; healthy when it answers below 500 |

### settings

| Field | Rules |
|---|---|
| `key` | required; `^[A-Za-z][A-Za-z0-9_]{0,39}$`, unique |
| `label` | required |
| `type` | `text`, `password`, `number`, `switch`, `select` |
| `help`, `unit` | shown with the field |
| `default` | must itself pass the checks below |
| `required` | refuse an empty value |
| `options` | for `select`: `[{ "value": "…", "label": "…" }]`, required |
| `min`, `max` | for `number` |
| `env` | environment variable that gets the value; `^[A-Z_][A-Z0-9_]{0,63}$` |

Values are one line, at most 1000 characters. `number` must parse as a number
within `min`/`max`; `switch` is `"true"` or `"false"`; `select` must be one of
the options. `password` values are kept in RasBeacon's secrets, never shown
again; an empty field on a later save keeps the saved one.

### templates

Go `text/template`, rendered before every start (unless `once` and the file
exists), written into the volume holding `path`. Data and functions:

- `.Settings.<key>`: every setting's value, passwords included
- `.Address`: the plugin's address; `.Host`: RasBeacon's address on the hotspot
- `port 3000`: the port on RasBeacon chosen for container port 3000
- `bcrypt <value>`: a bcrypt hash; `json <value>`: a JSON-quoted string (safe as a YAML/JSON scalar); `html <value>`: HTML-escaped (Go built-in)

A missing key renders empty.

### metrics

`label` and `path` required; `url` must start `http://{address}`. `path` is
dotted into the JSON answer (`a.b.0`). `divideBy` is another path (the metric
becomes path / divideBy). `format`: `number` (default, rounded with
thousands separators), `percent` (0–1 → `12.3%`), `bytes`, `ms`, `seconds`,
`text`. `scale` multiplies first. `username` / `password` add Basic
authentication. Read every 30 s, only while running; a failure shows `—`.

### Placeholders

In `afterInstall`, metric and health URLs, and metric `username`/`password`:
`{address}`, `{host}`, `{port:<container port>}`, `{setting:<key>}`
(passwords only in metric credentials, never shown). Unknown placeholders stay
as written.

## What a running plugin gets

- An IPv4 address on `10.89.0.0/24` (fixed while installed); gateway and DNS
  `10.89.0.1` (DNS only with `internet`).
- Its ports: devices on the hotspot reach `RasBeacon's address:<chosen port>`;
  guests, the internet and the uplink network never can. Listen on `0.0.0.0`.
- Network: nothing out unless `internet` (then like devices: through the VPN
  when it is on; nothing while the kill switch blocks). Devices only with
  `lan-access`. RasBeacon itself only for DNS.
- Its volumes; `/` is the image and is reset on every start.
- No privileges, `no-new-privileges`, at most 512 processes, the memory and
  CPU limits. `/sys/class/thermal` and `/sys/class/hwmon` hidden without
  `temperature`. `/dev/i2c-*` with `i2c`, `/dev/gpiochip*` with `gpio`.
- With `power-button`: `RASBEACON_SOCKET` (a Unix socket); `POST /power`
  with `{"action":"shutdown"}` or `{"action":"reboot"}` → 204.
- Output (stdout/stderr) is its log on its page (last 500 lines).
- RasBeacon restarts it when settings are saved. If it exits by itself it is
  started again; after 5 crashes in a row it is stopped until the owner
  starts it.

## Making a new plugin, step by step

1. `cp -r plugins/hello-world plugins/<id>`; set `id`, `name`, `version`
   (`1.0.0`), `description`, `author`, `homepage`.
2. Choose the image:
   - Your own program: write the `Containerfile` (multi-stage builds are fine;
     pin base images to a version; `FROM scratch` for static binaries).
   - A published image: delete the `Containerfile`; add `image` with the tag
     and its digest (`docker buildx imagetools inspect <ref>` → `Digest:`, the
     index digest; it must have a `linux/arm64` variant).
3. Ports: one entry per port the program listens on that devices should
   reach; `open: true` for its web interface. Don't list ports only
   RasBeacon reads (metrics, health): those are reached at `{address}`.
   RasBeacon's own ports 22, 53, 67, 80, 443 can't be chosen on RasBeacon, so
   pick a different `default` for them.
4. Data: list `volumes` for every folder that must survive restarts; set
   `storageMb` to what it needs.
5. Settings: one per thing the owner must choose; `env` or a template to pass
   them; `password` for secrets.
6. Permissions: the fewest that work.
7. `afterInstall` if the owner must do something (RasBeacon never changes its
   own settings for a plugin).
8. Metrics/health if the program has an HTTP API.
9. Icon: `icons/<id>.svg`, exported to `plugins/<id>/icon.png` at 256×256.
10. `README.md` for the owner: what it does, how to use it.
11. `tools/build.sh` must pass. Then ask the person to install
    `dist/<id>.rba` on a Pi.

## Rules for changes

- Bump `version` in a plugin's manifest whenever anything in that plugin
  changes, or RasBeacon won't offer the update.
- Never change a released plugin's `id`.
- Never create, print, store or commit a private signing key. The owner makes
  it with `tools/rbsign.sh plugin-keygen` and stores it as the repository
  secret `RASBEACON_PLUGIN_SIGNING_KEY`; only the Release workflow uses it.
- Don't add files to a plugin's folder that the image doesn't need; the .rba
  only takes the entries listed above.
- Keep `tools/` and `.github/workflows/` working for every plugin; CI builds
  and checks all of them on each push.
- Releases: Actions → Release → Run workflow with a tag like `v1.2.0`, or push
  that tag. The release must be signed (the workflow fails without the
  secret).
