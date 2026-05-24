# openwrt-adguardhome-updater

Vanilla OpenWrt AdGuard Home updater for **manual binary installs**.

Not intended for package-managed AdGuardHome installs via `apk` or `opkg`.

---

## Overview

This script updates the AdGuardHome binary on vanilla OpenWrt routers where AdGuardHome was installed manually as a standalone binary. It detects the existing install location, checks the current and latest versions, backs up before touching anything, installs the new binary, and validates DNS health after the update. If the health check fails, it automatically rolls back to the previous backup.

The config file (`AdGuardHome.yaml`), blocklists, query logs, and all data are preserved. Only the binary is replaced.

---

## Requirements

| Requirement | Details |
|---|---|
| OS | Vanilla OpenWrt (any version with `sh`, `wget`, `tar`, `gzip`) |
| Install type | Manual binary install only |
| Free space | At least 60 MB in `/tmp` |
| AdGuardHome | Already installed and running as a service via `/etc/init.d/` |

> This script will **refuse to run** if AdGuardHome is detected as package-managed by `apk` or `opkg`. Use your package manager to update instead:
> ```sh
> # OpenWrt 25.12+
> apk update && apk upgrade adguardhome
>
> # OpenWrt 24.10 and older
> opkg update && opkg upgrade adguardhome
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

```sh
wget -q https://raw.githubusercontent.com/melskiedev/openwrt-adguardhome-updater/main/openwrt-adguardhome-updater -O /usr/bin/openwrt-adguardhome-updater && chmod +x /usr/bin/openwrt-adguardhome-updater && /usr/bin/openwrt-adguardhome-updater --dry-run
```

> Run manually. Do not add to cron.

---

## Usage

### Interactive menu

Run with no flags in a terminal to open the interactive menu:

```sh
openwrt-adguardhome-updater
```

The menu shows a dashboard with router model, firmware, LAN IP, AdGuardHome version, web UI URL, current channel, and time. From the menu you can update, switch channels, rollback, manage backups, pick a theme, and view about info.

### Dry run first (always)

```sh
openwrt-adguardhome-updater --dry-run
```

Expected output:

```
[openwrt-adguardhome-updater] Install directory: /etc/AdGuardHome
[openwrt-adguardhome-updater] Binary path:       /etc/AdGuardHome/AdGuardHome
[openwrt-adguardhome-updater] Config path:       /etc/AdGuardHome/AdGuardHome.yaml
[openwrt-adguardhome-updater] DNS port:          <detected from config>
[openwrt-adguardhome-updater] Current version:   v0.107.x
[openwrt-adguardhome-updater] Latest version:    v0.107.x
```

Confirm the detected paths and versions are correct before proceeding.

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
| *(no flags)* | Open interactive menu (when run in a terminal) |
| `--dry-run` | Check current vs latest version, make no changes |
| `--force` | Reinstall even if already on latest version |
| `--yes` / `-y` | Assume yes to all prompts |
| `--non-interactive` | Skip all prompts, use defaults |
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
4. Install: stop service, replace binary, start service
5. Health check: process running + port listening (authoritative). DNS resolution test is best-effort only and will not trigger rollback on WAN/upstream issues.
6. Auto-rollback: if process or port check fails, the latest backup is restored automatically
7. Channel save: selected channel is persisted to `/root/adguardhome-updater/channel.conf`
8. sysupgrade persistence: key paths added to `/etc/sysupgrade.conf`

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

# Check DNS port is listening (replace port with your configured DNS port)
netstat -tulnp | grep <dns_port>

# Test DNS resolution through AdGuardHome directly
nslookup cloudflare.com 127.0.0.1:<dns_port>
```

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

For a fresh install, use the official AdGuardHome installer or the OpenWrt package first. Once AdGuardHome is installed and running as a service, this tool can update the binary or migrate a package-managed install to an upstream standalone binary.

```sh
# After a fresh install via package manager, migrate to upstream binaries:
openwrt-adguardhome-updater --migrate-from-apk --yes-i-have-lan-access
# or for older OpenWrt:
openwrt-adguardhome-updater --migrate-from-opkg --yes-i-have-lan-access

# Then for all future updates:
openwrt-adguardhome-updater
```

---

## Channels

The default channel is `release`. Select a channel from the interactive menu or via `--channel`.

| Channel | Source | Notes |
|---|---|---|
| `release` | GitHub Releases API + GitHub release assets | Stable, recommended |
| `beta` | `static.adtidy.org/adguardhome/beta/` | Pre-release, not for production |
| `edge` | `static.adtidy.org/adguardhome/edge/` | Development builds, not for production |
| `development` | `static.adtidy.org/adguardhome/development/` | Highly experimental |

For `release`, the script reads the latest version from the GitHub API and downloads the matching release asset.

For `beta`, `edge`, and `development`, the script downloads the latest channel tarball directly from AdGuard's static server. `version.json` is fetched as a best-effort for version display only. If it is unavailable, the updater still proceeds with the download.

```sh
# CLI channel selection
openwrt-adguardhome-updater --channel beta --dry-run
openwrt-adguardhome-updater --channel edge --dry-run
openwrt-adguardhome-updater --channel development --dry-run
```

---

## Disclaimer

This script is provided as-is without any warranty. It stops a running DNS service, replaces a binary, and restarts it. Always run `--dry-run` first. Always verify DNS is working after an update. Keep at least one backup before deleting any.

---

## License

MIT
