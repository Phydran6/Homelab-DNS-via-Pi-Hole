# Homelab-DNS-via-Pi-Hole

Pi-hole + Unbound DNS-Setup für `example.lan` auf einem Raspberry Pi 5 (16 GB) mit Raspberry Pi OS Lite (64-bit).

Dieses Repo enthält die laufende DNS-Konfiguration meines Heimnetzes inklusive lokaler Hostnamen-Auflösung für `*.example.lan`, netzweitem Block aller AAAA-Anfragen (kein IPv6 im Netz) und Conditional Forwarding zur Fritz!Box.

## Inhalt

- [Repo-Struktur](#repo-struktur)
- [Architektur-Überblick](#architektur-überblick)
- [Komplett-Setup von 0 auf 100%](#komplett-setup-von-0-auf-100)
  - [0. Hardware & Voraussetzungen](#0-hardware--voraussetzungen)
  - [1. Pi OS Lite installieren](#1-pi-os-lite-installieren)
  - [2. Erstes SSH und Updates](#2-erstes-ssh-und-updates)
  - [3. Statische IP sicherstellen](#3-statische-ip-sicherstellen)
  - [4. Unbound installieren](#4-unbound-installieren-rekursiver-resolver)
  - [5. Pi-hole installieren](#5-pi-hole-installieren)
  - [6. Pi-hole-Konfiguration übernehmen](#6-pi-hole-konfiguration-aus-diesem-repo-übernehmen)
  - [7. Lokale Hostnamen](#7-lokale-hostnamen-für-examplelan)
  - [8. Conditional Forwarding](#8-conditional-forwarding-zur-fritzbox)
  - [9. AAAA-Block aktivieren](#9-aaaa-block-aktivieren-kernpunkt-dieses-repos)
  - [10. dnsmasq-Failsafe-Snippets](#10-optional-dnsmasq-failsafe-snippets)
  - [11. Router umstellen](#11-router-umstellen)
  - [12. Verifikation](#12-verifikation)
- [Wartung](#wartung)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)
- [Lizenz](#lizenz)

## Repo-Struktur

```
homelab-dns/
├── README.md              ← diese Datei (Komplett-Setup)
├── pihole/
│   ├── README.md          ← Pi-hole-Konfig erklärt
│   └── pihole.toml        ← Hauptkonfiguration (anonymisiert)
└── dnsmasq.d/
    ├── README.md          ← dnsmasq-Snippets erklärt
    ├── 99-block-aaaa.conf ← Failsafe AAAA-Block für *.example.lan
    └── 99-noipv6.conf     ← Failsafe AAAA-Block für example.lan
```

## Architektur-Überblick

```
Clients ─► Pi-hole (Pi5, 192.168.1.10) ─► Unbound (127.0.0.1:5335) ─► Root-Server
                │
                ├─ Lokale Hostnamen für *.example.lan
                ├─ Conditional Forwarding für fritz.box → 192.168.1.1
                └─ Regex-Block: alle AAAA-Queries → NODATA
```

- **Pi-hole** als zentrale DNS- und Filterinstanz im LAN.
- **Unbound** als rekursiver Resolver lokal auf dem Pi (Port 5335), keine Upstream-DNS-Provider.
- **AAAA-Block netzweit** weil das Netz kein IPv6 nutzt — verhindert dass Clients in IPv6-Timeouts laufen.

---

## Komplett-Setup von 0 auf 100%

### 0. Hardware & Voraussetzungen

> Ein Raspberry Pi 5 als reiner DNS-Server ist wie ein Ferrari als Einkaufswagen — technisch maßlos überdimensioniert. Du kaufst trotzdem die 16-GB-Variante. Wir alle wissen es. Kein Urteil hier.

- [Raspberry Pi 5](https://www.amazon.de/s?k=Raspberry+Pi+5) — 4 GB reichen, 8/16 GB Luxus
- [microSD-Karte](https://www.amazon.de/s?k=microSD+A2+32GB) — mindestens 16 GB, Klasse A1 oder besser
- [Netzteil 27 W](https://www.amazon.de/s?k=Raspberry+Pi+5+Netzteil+27W) — offizielles Pi 5 Netzteil empfohlen
- [Ethernet-Kabel](https://www.amazon.de/s?k=Ethernet+Kabel+Cat7) — zur Fritz!Box
- Ein Rechner mit dem Raspberry Pi Imager
- Statische IP-Reservierung für den Pi auf der Fritz!Box (hier `192.168.1.10`)

### 1. Pi OS Lite installieren

1. **Raspberry Pi Imager** herunterladen: https://www.raspberrypi.com/software/
2. SD-Karte einlegen, Imager starten.
3. Auswahl:
   - **Device:** Raspberry Pi 5
   - **OS:** Raspberry Pi OS (other) → **Raspberry Pi OS Lite (64-bit)**
   - **Storage:** die SD-Karte
4. Auf das Zahnrad-Symbol klicken (OS-Anpassungen):
   - Hostname: `pihole`
   - SSH aktivieren (Passwort-basiert oder Public-Key)
   - Benutzername: `pihole` + sicheres Passwort
   - WLAN: nicht nötig (LAN bevorzugt für DNS-Server)
   - Locale: `Europe/Berlin`, Tastatur `de`
5. Schreiben, Karte in den Pi, Pi an LAN und Strom.

### 2. Erstes SSH und Updates

Vom eigenen Rechner aus:

```bash
ssh pihole@192.168.1.10
```

Im Pi:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl ca-certificates dnsutils
sudo reboot
```

### 3. Statische IP sicherstellen

Empfohlen: Statisches DHCP-Lease in der Fritz!Box setzen statt IP auf dem Pi zu hardcoden. So ist der Pi immer unter `192.168.1.10` erreichbar.

Fritz!Box-UI → **Heimnetz → Netzwerk → Netzwerkverbindungen** → Pi auswählen → "Diesem Netzwerkgerät immer die gleiche IPv4-Adresse zuweisen" anhaken.

### 4. Unbound installieren (rekursiver Resolver)

```bash
sudo apt install -y unbound
```

Konfigurationsdatei anlegen:

```bash
sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf > /dev/null <<'EOF'
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # Kein IPv6 im Netz
    do-ip6: no
    prefer-ip6: no

    # Sicherheit
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no

    # Performance
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m

    # Private-Adressbereich-Schutz
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
EOF

sudo systemctl restart unbound
sudo systemctl enable unbound
```

Test:

```bash
dig @127.0.0.1 -p 5335 example.com
```

Sollte eine Antwort liefern.

### 5. Pi-hole installieren

```bash
curl -sSL https://install.pi-hole.net | sudo bash
```

Im Installer:

- **Upstream DNS Provider:** Custom → `127.0.0.1#5335` (lokales Unbound)
- **Block lists:** Default reicht erstmal, später erweitern
- **Web Interface:** Yes
- **Web Server (lighttpd):** Yes
- **Log queries:** Yes
- **Privacy Mode:** Show everything

Am Ende notiert der Installer das Web-Admin-Passwort. Aufschreiben oder gleich neu setzen:

```bash
pihole setpassword
```

Web-UI: `http://192.168.1.10/admin`

### 6. Pi-hole-Konfiguration aus diesem Repo übernehmen

Der einfachste Weg: relevante Settings über die Web-UI nachziehen (siehe [pihole/README.md](pihole/README.md) für die Details), oder die `pihole.toml` als Vorlage nehmen:

```bash
# Repo klonen
git clone https://github.com/Phydran6/Homelab-DNS-via-Pi-Hole.git
cd Homelab-DNS-via-Pi-Hole

# Pihole stoppen
sudo systemctl stop pihole-FTL

# Konfig sichern
sudo cp /etc/pihole/pihole.toml /etc/pihole/pihole.toml.bak

# Eigene Konfig einspielen (vorher pihole.toml an dein Netz anpassen!)
sudo cp pihole/pihole.toml /etc/pihole/pihole.toml

# Pihole starten
sudo systemctl start pihole-FTL
```

Wichtig: Die `pihole.toml` enthält netzwerkspezifische Werte (IPs, Hostnamen, Web-Passwort-Hash). Vor dem Übernehmen anpassen — Details im [pihole/README.md](pihole/README.md).

### 7. Lokale Hostnamen für *.example.lan

Diese sind in der `pihole.toml` unter `dns.hosts` hinterlegt. Beispiel:

```toml
hosts = [
  "192.168.1.20 heimdall.example.lan",
  "192.168.1.20 nas2.example.lan",
  ...
]
```

Über die Web-UI: **Settings → Local DNS Records** (in v6: über **All Settings → dns.hosts**).

### 8. Conditional Forwarding zur Fritz!Box

Damit Geräte mit Fritz!Box-Hostname (`*.fritz.box`) aufgelöst werden:

Web-UI → **Settings → DNS → Conditional Forwarding** (in v6: **All Settings → dns.revServers**):

```
true,192.168.1.0/24,192.168.1.1,fritz.box
```

### 9. AAAA-Block aktivieren (Kernpunkt dieses Repos)

Da im Netz kein IPv6 läuft, sollen AAAA-Anfragen sofort mit NODATA beantwortet werden statt Clients in Timeouts laufen zu lassen.

Über die Web-UI: **Domains → RegEx filter** anlegen:

| Feld | Wert |
|------|------|
| Regular Expression | `.*;querytype=AAAA;reply=nodata` |
| Comment | `Block all AAAA queries (network has no IPv6)` |
| Group | Default |
| Aktion | **Add to denied domains** |

Test:

```bash
dig heimdall.example.lan A    @192.168.1.10   # → 192.168.1.20
dig heimdall.example.lan AAAA @192.168.1.10   # → NODATA, ANSWER: 0
```

Mehr Details inklusive Hintergrund: [pihole/README.md](pihole/README.md).

### 10. Optional: dnsmasq-Failsafe-Snippets

Die `dnsmasq.d/`-Snippets sind ein zusätzlicher Schutz, falls Pi-hole nicht läuft oder die Regex-Regel verloren geht. Beschreibung: [dnsmasq.d/README.md](dnsmasq.d/README.md).

```bash
sudo cp dnsmasq.d/*.conf /etc/dnsmasq.d/
sudo pihole reloaddns
```

### 11. Router umstellen

Damit alle Geräte den Pi als DNS nutzen:

Fritz!Box-UI → **Heimnetz → Netzwerk → Netzwerkeinstellungen → IPv4-Konfiguration → Lokaler DNS-Server** → `192.168.1.10`.

Ab jetzt fließt aller DNS-Verkehr im Netz über den Pi.

### 12. Verifikation

Auf einem beliebigen Client:

```bash
nslookup heimdall.example.lan
# → 192.168.1.20 (über 192.168.1.10)

nslookup -type=AAAA heimdall.example.lan
# → keine Adresse / NODATA
```

Pi-hole Web-UI → **Query Log** zeigt alle eingehenden Anfragen.

---

## Wartung

```bash
pihole -up           # Pihole-Update
sudo apt update && sudo apt full-upgrade -y   # System-Update
pihole -g            # Adlists aktualisieren (gravity)
pihole reloaddns     # DNS-Cache flushen
pihole status        # Status checken
```

Backup der wichtigsten Dateien:

```bash
sudo tar czf pihole-backup-$(date +%F).tar.gz \
  /etc/pihole/ \
  /etc/dnsmasq.d/ \
  /etc/unbound/unbound.conf.d/
```

## Troubleshooting

**AAAA wird trotz Regex weiter beantwortet?** → Cache flushen: `pihole reloaddns`. Falls es eine alte Regex ohne `reply=nodata` gibt: in der DB prüfen.

```bash
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db \
  "SELECT id, type, domain, enabled FROM domainlist WHERE type=3;"
```

**Container/VM kann lokale Hostnamen nicht auflösen?** → Sicherstellen dass der Container den Pi als DNS nutzt (Docker erbt normalerweise die Host-Resolver). Prüfung im Container: `cat /etc/resolv.conf`.

**`pihole reloaddns` zeigt readonly-variable Warning?** → Bekannter, harmloser Bug in v6.6.x. Cache wird trotzdem geflusht.

## Changelog

Siehe [CHANGELOG.md](CHANGELOG.md).

## Lizenz

MIT — siehe LICENSE (falls vorhanden).
