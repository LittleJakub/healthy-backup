# 🩺 healthy-backup

**Health-gated backup for OpenClaw rigs. Audits first — only backs up if healthy.**

You've got an OpenClaw setup. Config files, secrets manifests, skill directories, workspace state. Stuff you'd really rather not lose. And you have absolutely no idea if it's actually getting backed up in a usable state — or whether the last archive was a encrypted snapshot of a half-broken rig.

healthy-backup fixes that. Before touching a single file, it audits your rig across 8 categories. If anything critical is wrong, it aborts with a clear report. No silent backups of broken configurations. No encrypted archives you'll discover are corrupt when you actually need them.

---

## How it works

```
healthy-backup runs
  → 8-category health audit
      binaries, config, directories, disk, encryption,
      secrets perms, cloud remote, Ollama
  → hard block on any failure
  → stage files for your tier (secrets paths always excluded)
  → compress → AES256 GPG encrypt → save
  → prune old archives (keep last N)
  → optional rclone cloud sync
```

No daemon. No database. No root. Just cron, bash, and a GPG key.

---

## What you get

Three backup tiers depending on how much you want to capture:

| Tier | What's included |
|------|----------------|
| `minimal` | `openclaw.json` (scrubbed) + secrets manifest (key names only) |
| `migratable` | minimal + full `~/.openclaw` + `DEPENDENCIES.md` |
| `full` | migratable + workspace + skills directories |

Every archive contains a `HEALTH_REPORT.txt` — the full audit log from the moment it was created. You'll know the rig was healthy when it was snapshotted.

---

## Secrets policy

**openclaw.json is never copied verbatim.** Before staging, `jq walk()` replaces the value of any field whose name contains `password`, `token`, `secret`, or `key` with `"<redacted>"`. Config structure is preserved. The live file on disk is never touched.

**Certain paths are hard-excluded from rsync at every tier, with no config override:**

```
~/.openclaw/shared/secrets/
~/.openclaw/credentials/
*.key  *.pem  *.env  *.secret  .env
```

**GPG passphrase** is written to a `chmod 600` temp file and passed via `--passphrase-file` — never on the CLI, never visible in `ps`. Deleted via `trap EXIT`.

---

## Requirements

- bash 4+, tar, gpg, jq, rsync — standard on any modern Linux
- [OpenClaw](https://openclaw.ai) with at least one configured agent
- Optional: rclone (only if cloud sync enabled)

No sudo. No root. No system-level writes outside `~/.openclaw/` and your configured backup directory.

---

## Install

```bash
# Download and extract into your OpenClaw skills directory
tar -xzf healthy-backup-1.3.0.tar.gz --strip-components=1 \
  -C ~/.openclaw/skills/healthy-backup/

chmod +x ~/.openclaw/skills/healthy-backup/*.sh
bash ~/.openclaw/skills/healthy-backup/setup.sh
```

The setup wizard walks you through everything: tier, backup location, retention, cloud sync, dependency collectors, encryption key, cron schedule. It writes config to `~/.openclaw/config/healthy-backup/hb-config.json`, runs a dry-run automatically, and optionally installs the cron job — all in one flow.

---

## Usage

```bash
# First run — full audit + show what would be staged, nothing written:
bash healthy-backup.sh --dry-run

# Run a backup:
bash healthy-backup.sh

# Verify all archives in your backup directory:
bash healthy-backup.sh --verify

# Verify a single archive:
bash healthy-backup.sh --verify /path/to/healthy-backup-20260309-050001-migratable.tgz.gpg
```

---

## Cron

```bash
# Daily at 05:00 — adjust to taste
0 5 * * * bash ~/.openclaw/skills/healthy-backup/healthy-backup.sh >> ~/.openclaw/logs/healthy-backup.log 2>&1
```

Or let `setup.sh` install it for you.

---

## Restore

```bash
gpg --batch --passphrase-file ~/.openclaw/credentials/backup.key \
    --decrypt healthy-backup-YYYYMMDD-HHMMSS-migratable.tgz.gpg \
  | tar -xzf - -C /restore/target
```

---

## Health checks

| Check | On failure |
|-------|-----------|
| Required binaries (`tar`, `gpg`, `jq`, `rsync`) | Hard abort |
| `openclaw.json` is valid JSON | Hard abort |
| Key directories exist | Hard abort |
| Free disk ≥ `minDiskMb` | Hard abort |
| Encryption password available | Hard abort |
| `backup.key` permissions = 600 | Hard abort |
| Secrets file permissions = 600 | Hard abort |
| rclone remote reachable (if configured) | Hard abort |
| Skills dir present | Warn only |
| Ollama models loaded | Warn only |

---

## Configuration

Config lives at `~/.openclaw/config/healthy-backup/hb-config.json` (written by `setup.sh`):

```json
{
  "backupTier":      "migratable",
  "backupRoot":      "~/openclaw-backups",
  "uploadMode":      "local-only",
  "maxBackups":      5,
  "minDiskMb":       500,
  "collectOllama":   true,
  "collectNpm":      false,
  "collectCrontab":  false
}
```

Config resolution order: `hb-config.json` → env var → built-in default.

---

## Files

| File | Purpose |
|------|---------|
| `healthy-backup.sh` | Main script — backup, `--dry-run`, `--verify` |
| `setup.sh` | Interactive first-run configurator |

That's it. Two scripts.

---

## Release checksums (v1.3.0)

```
f05e443aa13c27d86210d23d029200c6d93ecab34a6fa0516be2bca98e25a732  healthy-backup.sh
0d3eb4f77c582c028aed633d7c8a5517032937fd4d8e38774d03d9c2c22c89b2  setup.sh
```

```bash
sha256sum healthy-backup.sh setup.sh
```

---

## Part of the hiVe stack

healthy-backup was built as part of [hiVe](https://github.com/LittleJakub) — a personal multi-agent system running on OpenClaw. It's designed to be lightweight, auditable, and safe to run unattended on a rig that has real secrets on it.

If you're running your own OpenClaw setup and want backups you can actually trust, this is for you.

---

## License

MIT. Do whatever you want with it. If it saves your bacon, a ⭐ is always appreciated.
