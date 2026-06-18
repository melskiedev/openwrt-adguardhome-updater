# openwrt-adguardhome-updater

Vanilla OpenWrt AdGuard Home updater for manual AdGuardHome binary installs.

Package-managed installs via `apk` or `opkg` are not updated directly. They can be migrated to standalone upstream binaries with `--migrate-from-package`.

---

## Disclaimer

This script is provided for personal use only. Use at your own risk.

The author is not responsible for any damage, data loss, bricked devices, broken networks, or any other issues that may result from using this script.

Always perform a full sysupgrade backup before running this script. Ensure you have physical or LAN access to your router before proceeding with any update or migration.

By using this script, you acknowledge that you understand the risks and accept full responsibility for the outcome.

---

## Overview

This script updates the AdGuardHome binary on vanilla OpenWrt routers where AdGuardHome was installed manually as a standalone binary. It detects the existing install location, checks the current and latest versions, backs up before touching anything, installs the new binary, and validates DNS health after the update. If the health check fails, it automatically rolls back to the previous backup.

The config file (`AdGuardHome.yaml`), blocklists, query logs, and all data are preserved. Only the binary is replaced.

---

## Requirements

| Requirement | Details |
|---|---|
| OS | Vanilla OpenWrt with `sh`, `tar`, and `gzip` |
| Downloader | `curl` recommended; `wget` supported only if HTTPS-capable |
| Checksum tool | `sha256sum`, BusyBox `sha256sum`, or `openssl` (release channel only) |
| Install type | Manual binary install, or package-managed install when using `--migrate-from-package` |
| Free space | At least 60 MB in `/tmp` |
| AdGuardHome | Already installed and running as a service via `/etc/init.d/` |

> Normal update mode refuses to overwrite package-managed AdGuardHome installs. To stay package-managed, update with `apk` or `opkg`. To convert the package-managed install to an upstream standalone binary, use:
>
> ```sh
> openwrt-adguardhome-updater --migrate-from-package --yes-i-have-lan-access
> ```

---

## Supported Architectures

| `uname -m` | AdGuardHome asset |
|---|---|
| `aarch64`, `arm64` | `arm64` |
| `armv8l`, `armv7l` | `armv7` |
| `armv6l` | `armv6` |
| `armv5l` | `armv5` |
| `x86_64`, `amd64` | `amd64` |
| `i386`, `i486`, `i686`, `i786` | `386` |
| `riscv64` | `riscv64` |
| `mips` / `mips64` (big-endian) | `mips_softfloat` / `mips64_softfloat` |
| `mips` / `mips64` (little-endian), `mipsel` | `mipsle_softfloat` / `mips64le_softfloat` |

Unsupported architectures are rejected before any download or install attempt.

---

## Install

On many OpenWrt routers, **wget is already available** and is the easiest way to install. **curl is optional** — if `curl` is not installed or HTTPS fails, use wget instead.

**Using wget (recommended on OpenWrt):**
```sh
wget -q https://raw.githubusercontent.com/melskiedev/openwrt-adguardhome-updater/main/openwrt-adguardhome-updater -O /usr/bin/openwrt-adguardhome-updater && chmod +x /usr/bin/openwrt-adguardhome-updater && /usr/bin/openwrt-adguardhome-updater
```

**Using curl** (requires the `curl` package and CA certificates):
```sh
opkg update && opkg install curl ca-bundle
```
```sh
curl -fsSL https://raw.githubusercontent.com/melskiedev/openwrt-adguardhome-updater/main/openwrt-adguardhome-updater -o /usr/bin/openwrt-adguardhome-updater && chmod +x /usr/bin/openwrt-adguardhome-updater && /usr/bin/openwrt-adguardhome-updater
```

> Run manually. Do not add to cron.

> On some OpenWrt 25.12+ builds, `/usr/bin/wget` may point to `wget-nossl`, which cannot fetch HTTPS URLs. If wget fails with `HTTPS support not compiled in`, install curl: `opkg install curl ca-bundle` (or `apk add curl ca-bundle`).

---

## Usage

Run with no flags in a terminal:

```sh
openwrt-adguardhome-updater
```

### Dry run first (always)

```sh
openwrt-adguardhome-updater --dry-run
```

Expected output:

```
[->] Checking prerequisites

[OK] Install directory: /etc/AdGuardHome
[OK] Binary:            /etc/AdGuardHome/AdGuardHome
[OK] Config:            /etc/AdGuardHome/AdGuardHome.yaml
[OK] Current version:   v0.107.x
[OK] Latest version:    v0.107.x (release)

[->] Dry run only. No changes made.
```

Confirm the detected paths and versions are correct before proceeding.

### Update without prompts

Use `--yes` when you already know you want to proceed:

```sh
openwrt-adguardhome-updater --yes
```

### Update

```sh
openwrt-adguardhome-updater
```

### Force reinstall (already on latest)

```sh
openwrt-adguardhome-updater --force
```

### Non-standard install location

```sh
openwrt-adguardhome-updater --install-dir /opt/AdGuardHome --dry-run
openwrt-adguardhome-updater --install-dir /opt/AdGuardHome
```

---

## All Flags

| Flag | Description |
|---|---|
| *(no flags)* | Open menu (when run in a terminal) |
| `--dry-run` | Check current vs latest version, make no changes |
| `--force` | Reinstall even if already on latest version |
| `--yes` / `-y` | Skip confirmation prompts |
| `--no-persist` | Skip adding paths to `/etc/sysupgrade.conf` |
| `--keep-tmp` | Do not delete `/tmp` download files after install |
| `--install-dir <path>` | Override detected install directory |
| `--channel <name>` | Set release channel: `release`, `beta`, `edge`, `development` |
| `--reset-channel` | Reset saved channel to `release` |
| `--theme <name>` | Set UI theme: `classic`, `green`, `amber`, `ocean`, `mono` |
| `--reset-theme` | Reset saved theme to `classic` |
| `--rollback` | Restore the latest backup |
| `--rollback <file>` | Restore a specific backup file |
| `--list-backups` | List available backups with sizes |
| `--delete-backup <file>` | Delete a specific backup file |
| `--delete-backup all` | Delete all backups (prompts for confirmation) |
| `--migrate-from-package` | Convert package-managed AdGuardHome to standalone binary |
| `--migrate-from-apk` | Alias for `--migrate-from-package` |
| `--migrate-from-opkg` | Alias for `--migrate-from-package` |
| `--yes-i-have-lan-access` | Required confirmation for migration (DNS drops briefly) |
| `--help` | Show usage |

---

## How It Works

### Detection

The script detects the AdGuardHome install in this order:

1. `--install-dir` if provided
2. Running process via `/proc/<pid>/exe` (most reliable)
3. Probe of known locations: `/etc/AdGuardHome`, `/opt/AdGuardHome`, `/usr/local/AdGuardHome`
4. `/usr/bin/AdGuardHome` as a standalone binary (warned, not treated as a directory install)

The init script is detected from `/etc/init.d/adguardhome` or `/etc/init.d/AdGuardHome`. The DNS port is read directly from `AdGuardHome.yaml` using the `dns.port` field, with a fallback to `3053` if the config is unreadable.

### Update flow

1. Preflight: root check, OpenWrt check, install detection, package-manager guard, dependency check, free space check, arch detection, version fetch
2. Backup: full `AGH_DIR` + init script + updater script archived to `/root/adguardhome-updater/backups/`, keeping last 3
3. Download: `AdGuardHome_linux_<arch>.tar.gz` from the selected channel
4. Verify: on the `release` channel, SHA256 is checked against `checksums.txt` from the same GitHub release (`beta`/`edge`/`development` skip verification with a warning)
5. Install: stop service, replace binary, start service
6. Health check: process running + port listening (authoritative). DNS resolution test is best-effort only and will not trigger rollback on WAN/upstream issues.
7. Auto-rollback: if process or port check fails, the latest backup is restored automatically
8. Channel save: selected channel is persisted to `/root/adguardhome-updater/channel.conf`
9. sysupgrade persistence: key paths added to `/etc/sysupgrade.conf`

### Backup contents

Each backup is a `.tar.gz` archive stored at:

```
/root/adguardhome-updater/backups/adguardhome-backup-YYYYMMDD-HHMMSS.tar.gz
```

It contains:
- Full `AGH_DIR` (binary + config + data)
- Init script
- Updater script (if present at `/usr/bin/openwrt-adguardhome-updater`)
- Manifest with paths and timestamp

A maximum of 3 backups are kept. Older ones are pruned automatically.

> Backups contain your full AdGuardHome directory, including config and any credentials stored in `AdGuardHome.yaml`. Treat backup files as sensitive.

---

## Backup Management

```sh
# List available backups
openwrt-adguardhome-updater --list-backups

# Roll back to the latest backup
openwrt-adguardhome-updater --rollback

# Roll back to a specific backup
openwrt-adguardhome-updater --rollback /root/adguardhome-updater/backups/adguardhome-backup-20260522-143200.tar.gz

# Delete a specific backup
openwrt-adguardhome-updater --delete-backup /root/adguardhome-updater/backups/adguardhome-backup-20260522-143200.tar.gz

# Delete all backups (requires typing 'yes' to confirm)
openwrt-adguardhome-updater --delete-backup all
```

---

## Post-Update Validation

After a successful update, migration, or rollback, the script prints the detected AdGuardHome web interface URL when it can determine the router LAN IPv4 address.

After a successful update, verify manually:

```sh
# Check service status
/etc/init.d/adguardhome status

# Confirm binary version
/etc/AdGuardHome/AdGuardHome --version

# Check DNS port is listening and verify bind address
netstat -tulnp | grep -E 'AdGuardHome|:3053|:2997|:3000'

# Test DNS resolution through the local resolver chain (dnsmasq -> AGH)
nslookup cloudflare.com 127.0.0.1
```

> The DNS listener should be bound to `127.0.0.1:<port>` or `[::1]:<port>` for loopback-only access. If it shows `0.0.0.0:<port>` or `:::port`, the service is listening on all interfaces and reachable from LAN clients directly.

---

## sysupgrade Persistence

The script automatically adds the detected paths to `/etc/sysupgrade.conf` on each run. The exact paths depend on where AdGuardHome is installed, but typically include:

```
<detected AGH_DIR>/          e.g. /etc/AdGuardHome/ or /opt/AdGuardHome/
<detected init script>       e.g. /etc/init.d/adguardhome
/usr/bin/openwrt-adguardhome-updater
/root/adguardhome-updater/
```

The full state directory `/root/adguardhome-updater/` is persisted, which covers backups, saved channel (`channel.conf`), and saved theme (`theme.conf`).

This ensures the AdGuardHome install, config, updater script, backups, and preferences survive a firmware upgrade.

> After a sysupgrade, run `--dry-run` again to confirm everything is intact before the next update.

---

## Do Not Cron This Script

Run this manually. The update process stops AdGuardHome briefly, which interrupts DNS for all clients on the network. It also performs a DNS resolution test as part of health checking, which assumes the network is otherwise stable. Scheduling this as a cron job is not recommended.

---

## Fresh Installs

This tool does not install AdGuardHome from scratch.

For a fresh install, use the official AdGuardHome installer or the OpenWrt package first. Once AdGuardHome is installed and running as a service, this tool can update the binary or migrate a package-managed install to an upstream standalone binary. See the Migrating section above.

---

## Migrating from OpenWrt Package to Standalone Binary

Package-managed installs are not updated directly in normal mode. To convert an `apk` or `opkg` AdGuardHome install to a standalone upstream binary:

```sh
openwrt-adguardhome-updater --migrate-from-package --yes-i-have-lan-access
```

Aliases are also supported:

```sh
openwrt-adguardhome-updater --migrate-from-apk --yes-i-have-lan-access
openwrt-adguardhome-updater --migrate-from-opkg --yes-i-have-lan-access
```

Migration briefly stops AdGuardHome, so DNS may drop during the process. Only run this when you have physical or LAN access to the router.

After migration, do not update AdGuardHome with `apk` or `opkg`. Use this updater for all future updates.

---

## Channels

The default channel is `release`. Select a channel from the menu or via `--channel`.

| Channel | Source | Notes |
|---|---|---|
| `release` | GitHub Releases API + GitHub release assets | Stable, recommended |
| `beta` | `static.adtidy.org/adguardhome/beta/` | Pre-release, not for production |
| `edge` | `static.adtidy.org/adguardhome/edge/` | Development builds, not for production |
| `development` | `static.adtidy.org/adguardhome/development/` | Highly experimental |

For `release`, the script reads the latest version from the GitHub API, downloads the matching release asset, and verifies the tarball SHA256 against `checksums.txt` from the same release.

For `beta`, `edge`, and `development`, the script downloads the latest channel tarball directly from AdGuard's static server. Checksum verification is skipped (with a warning). `version.json` is fetched as a best-effort for version display only. If it is unavailable, the updater still proceeds with the download.

```sh
# CLI channel selection
openwrt-adguardhome-updater --channel beta --dry-run
openwrt-adguardhome-updater --channel edge --dry-run
openwrt-adguardhome-updater --channel development --dry-run
```

---

## License

MIT
