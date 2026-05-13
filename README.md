# UniFi Backup Reader

A single-file browser tool that decodes UniFi Network (`.unf`) and UniFi OS console (`.unifi`) backups locally and renders them as a clean, searchable, copy-friendly settings page. Built by [AyeTech](https://ayetech.au).

Designed for the moment an engineer needs to recreate a config on a fresh controller, recover a lost Wi-Fi password from a backup, or just inspect what's actually saved in a UniFi backup file. Everything runs in your browser — no data ever leaves the page.

**Try it now:** [ayetech.au/tools/unifibackupreader/](https://ayetech.au/tools/unifibackupreader/) — current build, hosted by AyeTech. Same single-file tool; your backup is still parsed entirely client-side. If you'd rather not trust a hosted copy, download `index.html` from this repo and open it directly.

## Features

- **Reads both UniFi backup formats** — encrypted `.unf` (UniFi Network application) and `.unifi` (UniFi OS console wrapper). Drop a file, get a page.
- **Wi-Fi credentials matrix** — every SSID, password, security mode and VLAN binding. Click to reveal, click again to copy. Bulk TSV export.
- **Networks-at-a-glance matrix** — every VLAN with subnet, gateway, DHCP range and DNS in one scannable table.
- **DHCP reservations** — every fixed-IP client with MAC, IP, network and local-DNS record, one-click copy.
- **Per-device Port Manager** — unified table merging `port_table`, `port_overrides` and `ethernet_table`. Expand any row to see every per-port setting (PoE mode/class/watts, VLAN, storm control, 802.1X, STP, LLDP, SFP info, counters).
- **Full firewall coverage** — zone-based policies (9.x+), legacy rules, NAT, port forwards, traffic rules, traffic routes, QoS, firewall groups, predefined system rules.
- **Services tab** — RADIUS profiles + local users (with revealable secrets), schedule tasks, dynamic DNS, DNS-over-HTTPS, DHCP option profiles, WireGuard VPN users, hotspot operators/packages/vouchers, content filtering, SSL inspection.
- **Console metadata for `.unifi` files** — hostname, platform, firmware, paired UI.com devices, installed apps, app versions.
- **Universal copy ergonomics** — click any field value to copy, every card has a "copy all" button, every tab has bulk-export buttons (TSV/JSON).
- **Reveal-all toggle** — flip every masked secret in the current tab in one click.
- **Full JSON export** — entire backup as pretty-printed JSON, plus per-collection JSON copy/download.
- **Fully offline, zero outbound requests** — single HTML file (~490 KB) with fflate, aes-js, and all webfonts bundled in. Open it on a plane, on an air-gapped jump host, wherever. Nothing is fetched at runtime.
- **No telemetry** — your backup is parsed entirely in your tab. Nothing is uploaded.

## Usage

Two ways to run it:

- **Hosted by AyeTech** — open [ayetech.au/tools/unifibackupreader/](https://ayetech.au/tools/unifibackupreader/) and skip step 1.
- **Local file** — download `index.html` from this repo, double-click to open in your browser. No install, no server, no internet needed after the download.

Either way:

1. Open the page.
2. Drag a `.unf` or `.unifi` file onto the drop zone, or click to pick.
3. Browse the tabs, expand the sections, copy what you need.

The tool never asks for a server, network connection, login, or permission. The first thing the browser does after loading the HTML is sit and wait for a file. The second thing it does — when you drop one — is parse it locally. There is no third thing.

## Browser support

Tested on current Chrome, Edge, Firefox, and Safari. The bar is roughly:

| Browser | Minimum |
|---|---|
| Chrome / Edge | 89+ |
| Firefox | 89+ |
| Safari | 15+ |

The script is loaded as `<script type="module">`, which rules out IE and very old mobile browsers. Everything else the tool uses — `FileReader`, `TextEncoder`, `TextDecoder`, typed arrays, async/await, Clipboard API — is available everywhere a module script can run.

Performance is fine for typical backups (`.unif` files in the 1–20 MB range parse in under a second on a laptop). Very large backups — a long-lived controller with thousands of clients and a deep stats history — can push the BSON parser into the multi-second range. Everything happens on the main thread; no Web Worker, no WASM. If you hit a slow file, just wait it out.

## Deploy your own copy

Want it hosted on your own domain instead of opening a local file? One click puts a private copy on Cloudflare's edge, free, with no backend:

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ShaunLeslie/unifi-backup-reader)

The repo ships a minimal `wrangler.jsonc` that serves `index.html` as a static asset — no build step, no secrets, no telemetry added. To deploy from your own machine instead:

```bash
npm i -g wrangler
wrangler login
wrangler deploy
```

Where to grab a backup:

- **Cloud archive**: visit [account.ui.com/backups](https://account.ui.com/backups), open the console you want, download the latest backup (`.unifi`).
- **Controller UI**: open the UniFi Network web UI → *Settings → System → Backups → Download Backup* (`.unf`).

## How it works

The pipeline from "binary file you dropped" to "settings page you can read" runs entirely client-side in three stages — decrypt, decompress, parse — repeated across nested layers because UniFi backups are multi-layer containers.

```
.unifi  →  AES-256-CBC decrypt (IV = first 16 bytes, key = embedded)
        →  gunzip
        →  tar extraction
        →  picks `backup/network/db.gz` (the UniFi Network app's data)

.unf    →  AES-128-CBC decrypt (hardcoded key + IV)
        →  unzip
        →  picks `db.gz`

db.gz   →  gunzip
        →  BSON stream parsed in two formats:
             • old mongodump archive (pre-10.x, magic 0x8199e26d)
             • new `__cmd: "select"` stream (UniFi Network 10.x+)
        →  collections grouped, rendered
```

### Stage 1: Decrypt

Both formats are encrypted at rest using AES-CBC. The keys are static and have been publicly known for years, which makes the encryption a deterrent against casual snooping rather than a real secret — anyone with a backup file and this tool (or the equivalent Python utility) can decrypt it. UniFi's design assumes the backup is treated as a sensitive artifact by *you*, not protected by the file format.

- **.unf** uses AES-128-CBC with a 16-byte key (`bcyangkmluohmars`) and a 16-byte IV (`ubntenterpriseap`), both hardcoded. The same key/IV pair has shipped from UniFi Network 5.x through 10.x.
- **.unifi** uses AES-256-CBC with a per-file random IV (the first 16 bytes of the file) and a 32-byte key embedded in UniFi OS firmware. Slightly stronger than `.unf` because the IV varies per backup, but the key is still static and trivially recoverable from any UniFi OS image.

The AES routines come from the bundled [aes-js](https://github.com/ricmoo/aes-js). NoPadding is applied — both backups are produced by UniFi's exporters with block-aligned payloads.

### Stage 2: Decompress and unwrap

Decrypted output is a container, not data. Opening it differs by format:

- **.unf** decrypts to a standard ZIP archive. The file we want is `db.gz`. Other entries (`db_stat.gz`, `version`, `format`, `timestamp`, `system.properties`) feed the file-info sidebar shown after a successful load.
- **.unifi** decrypts to a gzipped TAR file (Unix tarball, gzipped). After ungzipping, the TAR is walked entry by entry. `backup/network/db.gz` is what becomes the Network tabs; the sibling folders (`backup/ucore/`, `backup/uos/`, `backup/users/`, `backup/innerspace/`) plus a few top-level JSON files describe the console hardware, paired UI.com devices, and installed apps — those populate the *Console* tab.

Gzip and ZIP handling come from the bundled [fflate](https://github.com/101arrowz/fflate). TAR extraction is implemented directly in `index.html` (TAR is a simple format — 512-byte headers with octal-encoded sizes, no compression of its own).

### Stage 3: Parse BSON, group collections

The inner `db.gz` ungzips to a MongoDB dump, but the wire format depends on when UniFi Network last touched the backup:

- **Pre-Network 10.x** dumps are *mongodump archive* streams: a magic number (`0x8199e26d`), per-collection metadata, then a sequence of BSON documents. The reader walks the stream and groups documents by their parent collection as it goes.
- **Network 10.x+** dumps use a newer stream where collection routing is inline — `__cmd: "select"` documents are inserted between batches to switch context. The reader interprets each command, then accumulates documents into whatever collection was most recently selected.

The BSON parser is hand-written for this project (no `bson` dependency). It walks the binary, decodes the primitive types UniFi actually uses (string, int32, int64, double, boolean, datetime, ObjectId, embedded document, array, binary), and skips anything else gracefully.

After parsing, collections are mapped to the tabs you navigate: `wlanconf` and `setting` feed Wi-Fi, `networkconf` feeds Networks, `user` feeds DHCP reservations, `device` feeds Port Manager, `firewallrule`/`firewallgroup`/`portforward`/`routing`/`firewallzonepolicy`/etc. feed Firewall, `dpiapp`/`radiusprofile`/`schedule_task`/`dynamicdns`/`hotspotop`/etc. feed Services. Anything without a custom view still ships out via the full JSON export.

## Format details

### `.unf` (UniFi Network application backup)
- AES-128-CBC, NoPadding
- Static key `bcyangkmluohmars` / IV `ubntenterpriseap` (publicly documented since 5.x, identical through 10.x)
- Decrypted content is a standard ZIP containing `db.gz`, `db_stat.gz`, `version`, `format`, `timestamp`, `system.properties`

### `.unifi` (UniFi OS console backup)
- AES-256-CBC, NoPadding
- Per-file random IV stored as the first 16 bytes of the file
- Static 32-byte key embedded in UniFi OS firmware
- Decrypted content is a gzipped tar containing `backup/network/` (a `.unf`-equivalent payload), `backup/ucore/`, `backup/uos/`, `backup/users/`, `backup/innerspace/` and metadata JSON files

## Auditing this build

The whole tool ships as one HTML file with everything bundled. That keeps it portable, but also means you should be able to verify the bundle without trust.

**Verify the page makes no outbound requests.** Open `index.html` in a browser, open DevTools → Network with "Preserve log" on, reload, then drop a backup. The Network panel should stay completely empty after the initial document load. If a request fires that you didn't expect, treat it as a regression and please open an issue.

**Verify the bundled libraries match upstream.** Each third-party library lives in its own `<script>` block between the `<!-- ─── Bundled libraries ─── -->` marker and the main `<script type="module">`. The bodies are pasted in verbatim from the published npm tarballs. To check a build:

```bash
# Fetch upstream sources and hash them
curl -sSL https://cdn.jsdelivr.net/npm/fflate@0.8.2/umd/index.js | shasum -a 256
curl -sSL https://cdn.jsdelivr.net/npm/aes-js@3.1.2/index.js | shasum -a 256

# Extract the same bytes from index.html and hash them.
# fflate UMD lives on a single line between the first <script>...</script>
# pair after the marker. aes-js lives in the second pair.
awk '/^<!-- ─── Bundled libraries/,/^<script type="module">/' index.html
```

The fflate match is byte-exact except for a leading `/*! fflate ... */` attribution comment that's prepended on the same line as the opening `<script>` tag (stripped trivially with `sed 's|/\*!.*\*/||'`). The aes-js file already carries its own MIT license header upstream, so nothing was added — the block between `<script>` and `</script>` is the upstream file unmodified.

**Verify the inlined webfonts.** The three `@font-face` rules near the top of the `<head>` reference data URLs containing base64-encoded woff2 binaries. Decode any one (`base64 -d`) and compare to the same file pulled from Google Fonts' CDN — they should byte-match. The font URLs are listed in the `index.html` git history at the commit that introduced bundling.

**Reproduce the build yourself.** There is no toolchain — the build is "concatenate, no minify." If you'd rather not trust the published HTML, you can produce an equivalent file by:

1. Checking out the last commit before bundling and grabbing that `index.html`.
2. Splicing the upstream `fflate` UMD and `aes-js` sources into a `<script>` pair just before the main `<script type="module">`.
3. Removing the original `import ... from 'https://cdn.jsdelivr.net/...'` lines from the module script.
4. Replacing the three `<link>` lines in `<head>` (Google Fonts preconnects + stylesheet) with a `<style>` block of three `@font-face` rules pointing at base64-encoded woff2 data URLs.
5. Diffing against the published `index.html`.

Any divergence is either a bug or a tampering signal — both worth knowing about.

## Prior art and credits

This tool stands on the shoulders of others.

**Bundled libraries** (inlined into `index.html`, MIT-licensed):

- [**fflate**](https://github.com/101arrowz/fflate) v0.8.2 by [Arjun Barrett](https://github.com/101arrowz) — pure-JS gzip/zip/inflate. Used to gunzip the inner `db.gz`, unzip `.unf` payloads, and gunzip the outer `.unifi` tar. Without it, none of the layers come apart.
- [**aes-js**](https://github.com/ricmoo/aes-js) v3.1.2 by [Richard Moore](https://github.com/ricmoo) — pure-JS AES implementation. Used for both AES-128-CBC (`.unf`) and AES-256-CBC (`.unifi`) decryption.

**Prior art**:

- [**zhangyoufu/unifi-backup-decrypt**](https://github.com/zhangyoufu/unifi-backup-decrypt) — the canonical Python decryptor for `.unf`. First to publicly document the static `.unf` key/IV pair.
- [**Darknetzz/UniFi-Backup-Explorer**](https://github.com/Darknetzz/UniFi-Backup-Explorer) — the closest prior browser-based reader and a useful reference for the .unf flow.

## Privacy

- The page makes **zero outbound requests** at runtime. JS libraries (fflate, aes-js) and webfonts (Outfit, Plus Jakarta Sans, JetBrains Mono) are all bundled directly into `index.html`. It works air-gapped.
- Backups are read using the browser's `FileReader` API. They never touch a network.
- If you want to verify, open DevTools → Network with the file open, then drop a backup. The Network panel will stay empty.

## Limitations

- Floorplan images are routinely missing or broken in `.unf` backups (known Ubiquiti issue) — this tool surfaces what's there but cannot recover what UniFi didn't save.
- The `.unifi` decryption key is static and embedded in UniFi OS firmware. If Ubiquiti rotates to a new format version (`_v3.unifi`), this tool will need a corresponding update.
- The tool is read-only. You cannot edit a backup and re-pack it. (This is intentional — repacking would require regenerating valid CRCs and keeping the archive byte-aligned, which is out of scope.)

## License

[MIT](LICENSE) — do what you want, no warranty. If you build something cool with it, a link back is appreciated but not required.

---

Built with care by [AyeTech](https://ayetech.au). Made for engineers who've ever stared at an old UniFi backup wondering what was actually in it.
