# Pi-hole Konfiguration

Dieser Ordner enthält alle Dateien, die direkt auf den Pi-hole-Host gehören.

## Dateien

### `pihole.toml`

Hauptkonfiguration für Pi-hole v6. Enthält alle geänderten Einstellungen gegenüber dem Default — erkennbar an `### CHANGED, default = ...`.

Wichtigste Anpassungen:

| Setting | Wert | Grund |
|---------|------|-------|
| `dns.upstreams` | `127.0.0.1#5335` | Lokales Unbound statt externer DNS-Provider |
| `dns.interface` | `eth0` | Feste Bindung an LAN-Interface |
| `dns.listeningMode` | `ALL` | Docker-Container in anderen Hosts können Pi als DNS nutzen |
| `dns.hosts` | `*.example.lan` | Lokale Hostnamen-Auflösung |
| `dns.revServers` | `192.168.1.0/24` → Fritz!Box | Conditional Forwarding für DHCP-Hostnamen |
| `dns.dnssec` | `false` | DNSSEC wird von Unbound erledigt, nicht doppelt validieren |

Vor dem Einspielen anpassen: eigene IPs, Hostnamen und Domain eintragen. Passwort-Hash (`pwhash`) bleibt leer und wird nach dem Start per `pihole setpassword` gesetzt.

```bash
sudo systemctl stop pihole-FTL
sudo cp pihole/pihole.toml /etc/pihole/pihole.toml
sudo systemctl start pihole-FTL
```

### `99-block-aaaa.conf` und `99-noipv6.conf`

dnsmasq-Snippets als Failsafe-Layer für den AAAA-Block. Greifen auf dnsmasq-Ebene, unabhängig davon ob Pi-hole-FTL gerade läuft oder die Regex-Regel noch nicht aktiv ist.

Das `99-`-Präfix sorgt dafür, dass diese Dateien **nach** Pi-holes eigenen Configs geladen werden (`01-pihole.conf`, `02-pihole-dhcp.conf` usw.) und damit Vorrang haben.

| Datei | Inhalt | Wirkung |
|-------|--------|---------|
| `99-block-aaaa.conf` | `address=/.example.lan/::0` | AAAA-Anfragen für `*.example.lan` → IPv6-Null-Adresse |
| `99-noipv6.conf` | `address=/example.lan/::` | AAAA-Anfragen für `example.lan` selbst → IPv6-Null-Adresse |

Die eigene Domain statt `example.lan` einsetzen, dann:

```bash
sudo cp pihole/99-*.conf /etc/dnsmasq.d/
sudo pihole reloaddns
```

> **Hinweis:** Die primäre Lösung für den AAAA-Block ist die Pi-hole-Regex `.*;querytype=AAAA;reply=nodata` (liefert sauberes NODATA statt `::` zurück). Diese Snippets sind nur Backup falls die Regex fehlt oder Pi-hole gerade neu startet.
