# percollate-proxmox

<img width="733" height="710" alt="image" src="https://github.com/user-attachments/assets/87854f4b-f2f2-40cf-9113-fa08ff530ace" />

[Percollate](https://github.com/danburzo/percollate) is a command-line tool
that grabs web pages from (one or more) URLs and bundles them into a
well-formatted **EPUB, PDF, HTML, or Markdown** document. This repository adds a
web UI on top of it, so the same conversions can be triggered from the browser
(`http://<host>:8080`) instead of typing CLI commands.

Single-file installer for percollate with the web UI (this repository). It
creates a Debian 13 LXC on Proxmox VE and installs the app **bare-metal** (no
Docker): system Chromium, Node.js, npm, and a systemd service. The web server
and worker run directly on the LXC from `/opt/percollate` and expose the UI on
port 8080.

`percollate.sh` is **fully self-contained** — nothing is downloaded from your
GitHub: the install fragment *and* the entire `web-ui/` (server.js, worker.js,
`public/`) are embedded in the script and written out at install/update time.
The only network dependency for "obtaining a version" is the official
`danburzo/percollate` GitHub API, read at install time.

This build also adds three worker-level improvements:

- **Readable output names** — generated files are renamed after the article
  title (spaces become `-`) instead of keeping the internal job id.
- **Medium friend links** — URLs containing a Medium `?sk=` share key are
  fetched through Medium's article API, so non-members get the full story
  instead of the paywalled preview.
- **Kindle-friendly EPUB images** — the **Optimized for Kindle** checkbox (on
  by default) shrinks raster images to fit a Paperwhite screen, converts them
  to grayscale, and re-encodes them as JPEG, which cuts EPUB size dramatically
  for e-ink reading.

## Usage

```sh
# on the Proxmox VE host
var_os='debian' bash -c "$(curl -fsSL https://raw.githubusercontent.com/danielhanaj/percollate_proxmox/refs/heads/main/percollate.sh)"
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

## Output names

After percollate produces a file, the worker renames it after the article title
instead of leaving the job id in the name:

- accents are folded and quotes removed
- runs of spaces/punctuation become a single `-`
- names are truncated to 80 characters
- `-2`, `-3`, … are appended when the name already exists in
  `/var/lib/percollate`

For a multi-URL job without a user-supplied title, the first article's title is
used and `-and-N-more` is appended. With `--individual`, each file uses its own
article title.

## Medium friend links

A Medium friend link carries a share key in the query string:

```
https://blog.example.com/some-story-4974cb0cf661?sk=b6a...
```

Plain HTTP clients (including percollate's `node-fetch`) receive Medium's
paywalled preview for these URLs. The worker recognizes the `?sk=` parameter,
extracts the 12-character post id, and requests the unlocked article from
Medium's JSON API:

```
https://medium.com/_/api/posts/<postId>?sk=<shareKey>
```

It converts the returned body model (paragraphs, headings, images, links,
quotes, code blocks, and lists) into a minimal HTML document, then lets
percollate format that document. The original URL is used as the base URL so
relative links still resolve.

This does not bypass Medium's paywall: the `sk` value is the author-issued
friend key, and Medium returns the full body only while that key is valid.

If the share key is expired or revoked, or the API cannot be reached, the
worker logs the reason and falls back to percollate's normal direct fetch.

## Kindle / e-ink image optimization

The web UI has an **Optimized for Kindle** checkbox (enabled by default). When
it is ticked for an `epub` job, the worker rewrites every raster image inside
the finished EPUB: a shrink-only resize to 1200×1600 (a Kindle Paperwhite 11th
gen screen is 1236×1648), transparency flattened onto white, 8-bit grayscale,
and a baseline JPEG re-encode at quality 82. KOReader shows a tapped image in
its own full-screen view, so anything above the screen resolution is wasted
bytes. SVGs stay vector, and any image that would grow after re-encoding is
left untouched.

Untick it to keep the original image bytes. The checkbox is disabled when the
format is not EPUB, and API callers can pass `"kindle": false` in the options
object to get the same behaviour.

Tune the limits in the systemd unit
(`/etc/systemd/system/percollate.service`) and restart:

| Variable | Default | Meaning |
| --- | --- | --- |
| `EPUB_IMAGE_MAX_WIDTH` | `1200` | Max width in pixels (never upscales) |
| `EPUB_IMAGE_MAX_HEIGHT` | `1600` | Max height in pixels |
| `EPUB_IMAGE_QUALITY` | `82` | JPEG quality (1–100) |

```sh
systemctl daemon-reload && systemctl restart percollate
```

The feature relies on the `sharp` and `jszip` npm packages, which the installer
adds to `package.json`. PDF, HTML, and Markdown output are not affected.

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
`/var/lib/percollate` under their article-derived names and are streamed back
through the web UI.

## Regenerating the installer

`percollate.sh` is generated — the embedded web-ui files appear twice (install
fragment + update helper) and are emitted from a single source so they cannot
drift. If you change anything under `web-ui/`, regenerate:

```sh
bash proxmox-lxc/build-percollate.sh   # rewrites proxmox-lxc/percollate.sh
```

Don't hand-edit the embedded blocks in `percollate.sh`; edit `web-ui/` and the
fragment/helper scaffolding in `build-percollate.sh`, then re-run.

The title-based filenames and Medium friend-link handling live in the embedded
`worker.js` block. If you regenerate from `web-ui/`, port those changes to
`web-ui/worker.js` first; otherwise the regenerated installer will overwrite
them.

## Layout

```
proxmox-lxc/
├── LICENSE               # MIT
├── README.md
├── build-percollate.sh   # dev-time: assembles percollate.sh from web-ui/
└── percollate.sh         # the whole installer (host + install fragment +
                          #   embedded web-ui sources) — the only file you run
```

The runtime only needs `percollate.sh`; `build-percollate.sh` and `web-ui/` are
development-time files used to regenerate it.

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
- Medium friend links need a valid, unexpired `sk` key. Expired links still
  convert, but only the public preview is available.
- Friend-link API requests are tried with Node's built-in `fetch`; if
  Cloudflare rejects that request, the worker retries inside the installed
  Chromium. Both paths use the same browser user agent as normal conversions.
