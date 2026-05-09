# Changelog

## [Unreleased]

## [1.2.0] - 2026-05-09

### Added
- `backup/README.md` — automatisches Pi-hole-Backup via n8n → SharePoint mit Telegram-Benachrichtigung
- `.gitignore` um SSH-Keys, n8n-Credentials und Editor-Dateien erweitert
- Link zum Backup-Guide im Hauptinhaltsverzeichnis und in der Repo-Struktur

## [1.1.0] - 2026-05-09

### Changed
- Anonymized all personal data in `mnt/user-data/outputs/` — replaced real domain and IP addresses with generic placeholders (`yourdomain.tld`, `10.0.x.x`)
- Removed `.claude/` tooling config from repository (added to `.gitignore`)

## [1.0.0] - 2026-04-28

### Added
- Initial Pi-hole v6 configuration (`pihole.toml`) with Unbound as recursive upstream resolver
- dnsmasq Failsafe-Snippets (`99-block-aaaa.conf`, `99-noipv6.conf`) for local domain AAAA blocking
- Full setup guide in `README.md` covering Pi OS installation through DNS verification
- Anonymized example configs for two deployment scenarios (`homelab-dns`, second-instance)
- Detailed documentation for dnsmasq snippets and Pi-hole config in `mnt/user-data/outputs/`
