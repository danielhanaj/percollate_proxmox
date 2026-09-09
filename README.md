# percollate-proxmox

Single-file installer for [percollate](https://github.com/danburzo/percollate)
with the web UI (this repository). It creates a Debian 13 LXC on Proxmox VE and
installs the app **bare-metal** (no Docker): system Chromium, Node.js, npm, and
a systemd service. The web server and worker run directly on the LXC from
`/opt/percollate` and expose the UI on port 8080.

`percollate.sh` is **fully self-contained** — nothing is downloaded from your
GitHub: the install fragment *and* the entire `web-ui/` (server.js, worker.js,
`public/`) are embedded in the script and written out at install/update time.
The only network dependency for "obtaining a version" is the official
`danburzo/percollate` GitHub API, read at install time.

## Usage

```sh
# on the Proxmox VE host
var_os='debian' bash -c "$(curl -fsSL https://raw.githubusercontent.com/danielhanaj/percollate_proxmox/refs/heads/main/percollate.sh)"
```

or, once hosted anywhere you like (gist, plain web server, repo):

```sh
bash -c "$(wget -qLO - https://example.com/percollate.sh)"
```

Choose Default or Advanced, wait for the build, then open
`http://<lxc-ip>:8080`.

To update later: open the container's console in the Proxmox GUI and run the
same `bash -c "$(wget -qLO - ...)"` command with the **newest** `percollate.sh`
— it shows the **Update** menu, which refreshes the embedded web-ui files,
re-reads the latest percollate release, runs `npm install`, and restarts.

## How it works

- **On the Proxmox host** the script bootstraps the community-scripts engine
  (`community-scripts/core`), materializes the embedded install fragment into a
  temp `COMMUNITY_SCRIPTS_ROOT`, and calls `build_container` — the engine reads
  `install/percollate-install.sh` from disk instead of the network and runs it
  inside the new LXC. The default engine banner only exists for apps whose
  header the core project generated, so `percollate.sh` overrides `header_info()`
  with embedded ASCII art (the banner text lives in `build-percollate.sh`).
- **Inside the LXC** the install fragment: installs deps + Node 24, creates the
  `percollate` user, writes the web-ui files into `/opt/percollate`, queries the
  official GitHub API for the latest percollate release and pins
  `"percollate": "<version>"` in a generated `package.json`, runs `npm install`,
  and installs + starts the systemd unit.
- **Re-run inside the LXC** the same script detects the container and offers
  **Update** (`_percollate_write_ui` + `_percollate_setup_package` +
  `systemctl restart`).
- Helpers used from the engine: `update_os`, `setup_nodejs`, `motd_ssh`,
  `customize`, `cleanup_lxc`, `msg_*`. No Docker anywhere — this aligns with the
  community-scripts policy *"We do NOT use Docker for our installation scripts."*

## Native stack

| Component  | Choice | Why |
| ---------- | ------ | --- |
| OS         | Debian 13 (trixie) | system `chromium` tracks the upstream 15x line via stable/security updates — matches the verified Docker pair |
| percollate | latest release from `danburzo/percollate` GitHub API | single source of truth for the version; pinned into `package.json` at install |
| Node.js    | 24 (via `setup_nodejs`) | ESM server/worker, puppeteer 25 needs Node ≥18 |
| Browser    | system `chromium` apt package | no userland Chrome download (~150 MB saved) |
| puppeteer  | forced to **25.10.0** via npm `overrides` (same trick as the Docker `resolutions`) | percollate pins puppeteer 19.x; 25.x drives the trixie Chromium |
| Service    | `percollate.service` (systemd, user `percollate`) | auto-start, restart-on-failure, no Docker |

The runtime equals the Docker image's `NO_SANDBOX=1` mode (verified working,
including the two-URL EPUB case): unprivileged LXCs block user namespaces, so
`NO_SANDBOX=1` passes `--no-sandbox` to Chromium. Output files land in
`/var/lib/percollate` and are streamed back through the web UI.

## Regenerating the installer

`percollate.sh` is generated — the embedded web-ui files appear twice (install
fragment + update helper) and are emitted from a single source so they cannot
drift. If you change anything under `web-ui/`, regenerate:

```sh
bash proxmox-lxc/build-percollate.sh   # rewrites proxmox-lxc/percollate.sh
```

Don't hand-edit the embedded blocks in `percollate.sh`; edit `web-ui/` and the
fragment/helper scaffolding in `build-percollate.sh`, then re-run.

## Layout

```
proxmox-lxc/
├── LICENSE               # MIT
├── README.md
├── build-percollate.sh   # dev-time: assembles percollate.sh from web-ui/
└── percollate.sh         # the whole installer (host + install fragment +
                          #   embedded web-ui sources) — the only file you run
```

## Notes

- LXC defaults: 2 vCPU, 3072 MB RAM, 12 GB disk (headless Chromium dominates).
- If the GitHub API is unreachable at install time (or returns a non-`x.y.z`
  tag), the version falls back to `4.3.0`.
- If a future Debian release drops Chromium below ~145, bump the `overrides`
  puppeteer version in `build-percollate.sh` to the matching line of
  https://pptr.dev/chromium-versions and regenerate.
- Inside the embedded fragment use `${APPLICATION}` (exported by the engine from
  the host `APP` variable) — plain `$APP` is unbound there and aborts the
  install under `set -u`.
