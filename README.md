# UniFi Backup Reader

A single-file browser tool that decodes UniFi Network (`.unf`) and UniFi OS console (`.unifi`) backups locally and renders them as a clean, searchable, copy-friendly settings page. Built by [AyeTech](https://ayetech.au).

Designed for the moment an engineer needs to recreate a config on a fresh controller, recover a lost Wi-Fi password from a backup, or just inspect what's actually saved in a UniFi backup file. Everything runs in your browser — no data ever leaves the page.

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
- **No backend** — single HTML file, ~210 KB, opens directly in any modern browser.
- **No telemetry** — your backup is parsed entirely in your tab. Nothing is uploaded.

## Usage

1. Download `index.html`.
2. Open it in any modern browser (Chrome, Safari, Firefox, Edge).
3. Drag a `.unf` or `.unifi` file onto the drop zone, or click to pick.
4. Browse the tabs, expand the sections, copy what you need.

Where to grab a backup:

- **Cloud archive**: visit [account.ui.com/backups](https://account.ui.com/backups), open the console you want, download the latest backup (`.unifi`).
- **Controller UI**: open the UniFi Network web UI → *Settings → System → Backups → Download Backup* (`.unf`).

## How it works

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

All decryption, decompression, tar parsing and BSON parsing run client-side. The HTML file loads exactly two JS modules from jsdelivr CDN at runtime — [fflate](https://github.com/101arrowz/fflate) (gzip/zip) and [aes-js](https://github.com/ricmoo/aes-js) (AES) — for which the source is open.

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

## Prior art and credits

This tool stands on the shoulders of others:

- [**zhangyoufu/unifi-backup-decrypt**](https://github.com/zhangyoufu/unifi-backup-decrypt) — the canonical Python decryptor for `.unf`. First to publicly document the static `.unf` key/IV pair.
- [**Darknetzz/UniFi-Backup-Explorer**](https://github.com/Darknetzz/UniFi-Backup-Explorer) — the closest prior browser-based reader and a useful reference for the .unf flow.

## Privacy

- The HTML file makes outbound requests only to load Google Fonts, two open-source JS libraries from jsdelivr, and the AyeTech logo from `ayetech.au` (for branding). No request ever carries your backup data.
- Backups are read using the browser's `FileReader` API. They never touch a network.
- If you want to verify, open DevTools → Network and watch what fires when you drop a file. Spoiler: nothing.

## Limitations

- Floorplan images are routinely missing or broken in `.unf` backups (known Ubiquiti issue) — this tool surfaces what's there but cannot recover what UniFi didn't save.
- The `.unifi` decryption key is static and embedded in UniFi OS firmware. If Ubiquiti rotates to a new format version (`_v3.unifi`), this tool will need a corresponding update.
- The tool is read-only. You cannot edit a backup and re-pack it. (This is intentional — repacking would require regenerating valid CRCs and keeping the archive byte-aligned, which is out of scope.)

## License

[MIT](LICENSE) — do what you want, no warranty. If you build something cool with it, a link back is appreciated but not required.

---

Built with care by [AyeTech](https://ayetech.au). Made for engineers who've ever stared at an old UniFi backup wondering what was actually in it.
