# Making RasBeacon plugins

This guide takes you from the hello-world plugin in this repository to a full
plugin, one feature at a time. Each step points to a real plugin in
[RasBeacon-plugins](https://github.com/aazam476/RasBeacon-plugins) that
uses it. For every field and rule, see the reference:
[RasBeacon's docs/plugins.md](https://github.com/aazam476/RasBeacon/blob/main/docs/plugins.md).

1. [What a plugin is](#1-what-a-plugin-is)
2. [Build and install hello-world](#2-build-and-install-hello-world)
3. [How hello-world works](#3-how-hello-world-works)
4. [Make it yours](#4-make-it-yours)
5. [Settings](#5-settings)
6. [Using a published image](#6-using-a-published-image)
7. [Keeping data](#7-keeping-data)
8. [Numbers on its page, and a health check](#8-numbers-on-its-page-and-a-health-check)
9. [Permissions](#9-permissions)
10. [A USB drive](#10-a-usb-drive)
11. [Hardware, and shutting RasBeacon down](#11-hardware-and-shutting-rasbeacon-down)
12. [Telling the owner what to do next](#12-telling-the-owner-what-to-do-next)
13. [Publishing](#13-publishing)
14. [When something goes wrong](#14-when-something-goes-wrong)

## 1. What a plugin is

A plugin is a container (like a Docker container) that RasBeacon runs with
Podman. It comes as a `.rba` file: a zip file holding a `manifest.json` that
says what the plugin is and needs, an icon, and usually the container image
itself.

When the owner installs it, RasBeacon shows what it will be able to do, asks
for its settings and the ports on RasBeacon where the hotspot's devices will
reach it, then runs it. Each plugin gets:

- its own address (on `10.89.0.0/24`), and nothing else on the network unless
  its permissions say so. Devices on the hotspot reach it through RasBeacon's
  address and the ports the owner chose; guests and the internet never can.
- limits on memory and processor, and a fixed amount of storage.
- a log (its output) on its page, and a Running switch.

A plugin can't change RasBeacon's settings or read its passwords.

## 2. Build and install hello-world

You need Linux with Docker (and buildx), Go 1.24 or newer, git, zip and
Python 3. Plugins run on the Raspberry Pi, so images are built for arm64: on
an arm64 machine that is fast; on an ordinary PC, let Docker emulate arm64
once with `docker run --privileged --rm tonistiigi/binfmt --install arm64`.

```sh
tools/build.sh
```

It builds `dist/hello-world.rba` (not signed, which is fine for trying it).
The first run also builds RasBeacon's `rbsign` tool into `.cache/`; the build
ends by checking the file exactly as RasBeacon does.

On RasBeacon, open **Plugins → Install from a file**, choose
`dist/hello-world.rba`, look at the sheet (no permissions, one setting, the
web page's port), and press **Install**. When it says *Running*, press
**Open**.

## 3. How hello-world works

`plugins/hello-world/` holds:

- **`Containerfile`**: the image. BusyBox's web server shows the files in
  `/www` on port 8080.
- **`manifest.json`**: the important parts:
  - `"ports": [{ "name": "Web page", "container": 8080, "open": true }]`:
    the container listens on 8080; the owner picks the port on RasBeacon
    (8080 unless taken); `open` puts an **Open** button on its page.
  - `"volumes": [{ "name": "site", "path": "/www" }]`: a folder of its own
    storage, mounted at `/www`.
  - `"settings"`: one text field, `message`.
  - `"templates": [{ "file": "index.html", "path": "/www/index.html" }]`:
    before every start, RasBeacon fills in `templates/index.html` with the
    settings and writes it into the volume.
  - `memoryMb`, `cpus`, `storageMb`: small limits for a small plugin.
- **`templates/index.html`**: `{{html .Settings.message}}` becomes the
  message (`html` makes it safe in a web page). `{{.Address}}` is the
  plugin's address.

Change the message on its page in RasBeacon and **Save**: the plugin
restarts and the page shows it.

## 4. Make it yours

1. Copy `plugins/hello-world` to `plugins/<your-id>`.
2. In `manifest.json`, set `id` (2 to 32 lowercase letters, digits and
   dashes; it never changes once people install it), `name`, `version`,
   `description`, `author` and `homepage`.
3. Draw a square icon (256×256 PNG; RasBeacon takes 128×128 to 1024×1024)
   as `icon.png`. Keep its source (SVG) in `icons/`.
4. Write `README.md`: what it does and how to use it. RasBeacon shows it on
   the plugin's page, with Markdown's paragraphs, lists, **bold**, *italic*,
   `code` and links.
5. Replace the `Containerfile` with your program's. Images must run on
   `linux/arm64`, and should be small: RasBeacon may run on a Raspberry Pi
   Zero 2 W with 512 MB of memory. Pin base images to a version.
6. `tools/build.sh`, install, try it.

Delete `plugins/hello-world` once you don't need it any more.

## 5. Settings

Settings are drawn as a form by RasBeacon, from the manifest. Types: `text`,
`password`, `number` (with `min`, `max`, `unit`), `switch` (`"true"` or
`"false"`) and `select` (with `options`). Give each a `label`, and a `help`
line if it isn't obvious.

A setting reaches your program two ways:

- **`env`**: an environment variable gets the value. The simplest.
- **A template**: a file in `templates/` filled in with
  `{{.Settings.key}}`. Also available: `{{.Address}}`, `{{.Host}}`
  (RasBeacon's address on the hotspot), `{{port 3000}}` (the port on
  RasBeacon chosen for container port 3000), `{{bcrypt .Settings.password}}`
  (a password hash), `{{json .Settings.key}}` (a quoted string, safe in YAML
  and JSON), and Go's own `html`. Add `"once": true` to write the file only
  the first time, when your program changes it later itself.

`password` settings are kept with RasBeacon's secrets, exactly as typed, and
never shown again; an empty field keeps the saved one.

Templates are written into your volumes before each start, while your
container is stopped. RasBeacon never writes through a symbolic link your
program made there: if a folder on a template's path is a link, the plugin
doesn't start. A template may call `bcrypt` at most 10 times, and a filled-in
file may be at most 4 MB.

- *Samba* uses `env` for its share name, username and password
  ([manifest](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/samba/manifest.json)).
- *Argon ONE V2* uses `number` settings for its fan curve
  ([manifest](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/argon-one/manifest.json)).
- *AdGuard Home* writes its config file once from a template, with the
  password hashed by `bcrypt`
  ([template](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/adguard-home/templates/AdGuardHome.yaml)).

## 6. Using a published image

Instead of building an image, a plugin can use one from a registry, such as
Docker Hub. Remove the `Containerfile` and add to the manifest:

```json
"image": { "ref": "docker.io/owner/name:1.2.3", "digest": "sha256:…" }
```

The digest pins the exact image (find it with
`docker buildx imagetools inspect docker.io/owner/name:1.2.3`, the first
`Digest:`). RasBeacon downloads it when installing, through the VPN, and
shows its size first. The `.rba` is then tiny.

- *AdGuard Home* runs AdGuard's official image this way
  ([manifest](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/adguard-home/manifest.json)).

## 7. Keeping data

Everything in `volumes` is kept across restarts and updates; everything else
in the container is lost when it restarts. All volumes share one file of
`storageMb` (default 256 MB) on the SD card; ask for what you need, not more.
A new version may ask for more; RasBeacon grows the file.

- *AdGuard Home* keeps `work` and `conf`; *Samba* keeps 2 GB for files when no
  USB drive is chosen.

## 8. Numbers on its page, and a health check

Up to four `metrics` show as large numbers on the plugin's page. RasBeacon
reads them every 30 seconds from your program's own HTTP API:

```json
"metrics": [
  { "label": "Queries (24 h)", "url": "http://{address}:3000/control/stats", "path": "num_dns_queries" },
  { "label": "Blocked", "url": "http://{address}:3000/control/stats",
    "path": "num_blocked_filtering", "divideBy": "num_dns_queries", "format": "percent" }
]
```

`path` picks a value from the JSON answer (`a.b.0` for nested values and
lists). Formats: `number`, `percent` (0 to 1), `bytes`, `ms`, `seconds`,
`text`; `scale` multiplies first. `username` and `password` add a login
(placeholders like `{setting:password}` work). A `label` has at most 30
characters.

The URLs reach only your plugin: `http://{address}`, then optionally a port
and a path, nothing else (RasBeacon doesn't follow redirects either).

`health` is a URL that answers below 500 when the plugin works; its page says
*Not answering* otherwise.

- *AdGuard Home* has four metrics with a login, and a health check.
- *Argon ONE V2* serves a tiny `/status` just for its numbers
  ([main.go](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/argon-one/main.go)).

## 9. Permissions

Without permissions a plugin can only answer on its ports and keep its data.
Ask only for what it needs; the owner sees each one in plain words before
installing:

| Permission | Gives |
|---|---|
| `internet` | Connections out, the way devices go (through the VPN when it is on), and RasBeacon's DNS at `10.89.0.1`. |
| `lan-access` | Connections to devices on the hotspot (never guests). |
| `usb-drive` | A USB drive the owner picks (see below). |
| `i2c`, `gpio` | `/dev/i2c-*`, `/dev/gpiochip*`. |
| `power-button` | Asking RasBeacon to shut down or restart. |
| `temperature` | Reading `/sys/class/thermal` (hidden otherwise). |

- *AdGuard Home* asks only for `internet`.

## 10. A USB drive

With the `usb-drive` permission and `"usbDrive": { "path": "/share", "optional": true }`,
the owner picks a drive in the settings and it appears at that path. RasBeacon
mounts it so nothing on it can run, and gives it to one plugin only. With
`optional`, the owner may pick none; a volume at the same path is used then.

- *Samba* shares a USB drive or its own storage
  ([manifest](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/samba/manifest.json),
  [entrypoint.sh](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/samba/entrypoint.sh)).

## 11. Hardware, and shutting RasBeacon down

`i2c` and `gpio` give the devices; use Linux's character devices from your
program (no special libraries needed). Limit the plugin to the boards it
supports with `"boards": ["pi4"]` (`pi5`, `pi4`, `pi3`, `zero2w`).

If your plugin uses something only a newer RasBeacon has, add
`"minRasBeacon": "0.2.0"` (the version that added it). Older RasBeacons then
show it as *Needs RasBeacon 0.2.0 or newer* instead of failing to install it.

With `power-button`, the environment variable `RASBEACON_SOCKET` names a
socket: send `POST /power` with `{"action": "shutdown"}` (or `"reboot"`).
Only root in the container can use it.

- *Argon ONE V2* reads the temperature, drives the case's fan over I²C,
  watches the power button on GPIO 4, and asks RasBeacon to restart or shut
  down ([main.go](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/argon-one/main.go),
  [hardware.go](https://github.com/aazam476/RasBeacon-plugins/blob/main/plugins/argon-one/hardware.go)).

## 12. Telling the owner what to do next

`afterInstall` is shown after installing and stays on the plugin's page.
Placeholders: `{address}` (the plugin's address), `{host}` (RasBeacon's),
`{port:8080}` (the port chosen for container port 8080), `{setting:key}`.
Plugins can't change RasBeacon's settings; say what the owner should change.

- *AdGuard Home* asks the owner to set RasBeacon's DNS servers to its address.

## 13. Publishing

1. Make your signing key once: `tools/rbsign.sh plugin-keygen`. Keep the
   private key safe, add it as the repository secret
   `RASBEACON_PLUGIN_SIGNING_KEY`, and never change it (RasBeacon accepts
   updates only with the same key).
2. Bump `version` in the manifest of every plugin you changed.
3. Actions → Release → Run workflow, with a release name like `v0.1.0`
   (named after the RasBeacon version it's for: `v0.1.0`, `v0.1.1`, … for
   RasBeacon 0.1).
4. On RasBeacon, add your repository as a source (Plugins → Browse plugins →
   Sources); your plugins appear in the catalog.

The release holds each `.rba`, its icon, and `rasbeacon-catalog.json` (names,
versions and descriptions, so the catalog shows them without downloading).

## 14. When something goes wrong

- **`tools/build.sh` stops with a manifest error:** the message names the
  field; the rules are in the
  [reference](https://github.com/aazam476/RasBeacon/blob/main/docs/plugins.md#the-manifest).
  Unknown fields are refused (check the spelling).
- **It installs but says *Not working*:** its page shows why. Look at its
  **Log**.
- **It keeps restarting, then *Stopped after crashing*:** your program exits.
  Look at the log, fix it, build, and install the new file over it (a newer
  `version`).
- **`exec format error` in the log:** the image isn't for arm64.
- **The Open button does nothing:** the port in the manifest must be the one
  your program listens on, and it must listen on all addresses (`0.0.0.0`),
  not `127.0.0.1`.
- **No internet inside the plugin:** add the `internet` permission. With the
  VPN's kill switch on and the VPN down, nothing gets out, by design.
