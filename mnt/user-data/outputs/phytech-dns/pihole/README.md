# Pi-hole-Konfiguration

Diese `pihole.toml` ist die Hauptkonfiguration der Pi-hole-Instanz auf `10.0.0.30`. Pi-hole v6 verwaltet alle Settings (außer Domain-/Adlists und Clients, die in `gravity.db` liegen) in einer einzigen TOML-Datei.

## Wichtige eigene Anpassungen gegenüber dem Default

Pi-hole markiert eigene Anpassungen mit `### CHANGED, default = ...`. Hier die relevanten:

### `[dns]` — Upstream

```toml
upstreams = ["127.0.0.1#5335"]
```

Lokales Unbound auf Port 5335 als einziger Upstream. Keine externen DNS-Provider.

### `[dns]` — Interface & Listening

```toml
interface = "eth0"
listeningMode = "ALL"
```

`ALL` ist nötig, weil Docker-Container in anderen VMs/Hosts ebenfalls den Pi als DNS-Server nutzen. **Wichtig: Pi muss hinter dem Heimnetz-Router liegen, sonst offener Resolver.**

### `[dns]` — DNSSEC

```toml
dnssec = false
```

Aus, weil DNSSEC-Validierung von Unbound erledigt wird. Doppelte Validierung in Pi-hole nicht nötig.

### `[dns]` — Bogus Private

```toml
bogusPriv = true
```

Reverse-Lookups für private IP-Ranges, die nicht in `/etc/hosts` oder DHCP-Leases stehen, werden mit "no such domain" beantwortet statt upstream durchgereicht.

### `[dns]` — Lokale Hostnamen

Unter `dns.hosts` sind die `*.yourdomain.tld`-Einträge gepflegt:

```toml
hosts = [
  "10.0.0.20 heimdall.yourdomain.tld",
  "10.0.0.20 nas2.yourdomain.tld",
  "10.0.0.20 omv.yourdomain.tld",
  ...
]
```

Format: `<IP> <FQDN>`. Subdomains lokal aufzulösen ist sauberer als sie über Public-DNS via Reverse-Proxy zu routen.

### `[dns.revServers]` — Conditional Forwarding

```toml
revServers = ["true,10.0.0.0/25,10.0.0.1,fritz.box"]
```

Anfragen für `*.fritz.box` und Reverse-Lookups im `10.0.0.0/25`-Bereich werden an die Fritz!Box geforwardet, damit DHCP-Hostnamen wie `mein-laptop.fritz.box` aufgelöst werden.

### `[dns]` — ESNI blockieren

```toml
blockESNI = true
```

`_esni.`-Subdomains von gesperrten Domains werden ebenfalls blockiert.

### `[webserver.api]` — Web-Passwort

```toml
pwhash = ""
```

In dieser anonymisierten Version leer. Beim ersten Start nach dem Einspielen mit `pihole setpassword` setzen.

## Domain-Listen (gravity.db)

Die `gravity.db` (nicht hier im Repo aus Privacy-Gründen) enthält:

### Adlists

- `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` — Standard-Block-Liste

### Regex-Deny

- `.*;querytype=AAAA;reply=nodata` — **netzweiter AAAA-Block** (Kernpunkt)
- Diverse Adult-Domains (Regex)

### Regex-Allow

Aktuell deaktiviert. War früher `(\.|^)yourdomain\.tld$` zur Sicherstellung, dass die eigene Domain nicht durch Adlists geblockt wird.

### Groups

- `Default` — alle Clients
- `noblock` — Sondergruppe mit deaktivierten Filtern für bestimmte MAC-Adressen

## Hintergrund: Warum der AAAA-Block?

Das LAN nutzt kein IPv6. Trotzdem hat der Provider `yourdomain.tld` öffentliche AAAA-Records, und viele Anwendungen bevorzugen IPv6 (Linux glibc, Docker). Konsequenz ohne diesen Block:

1. Container/Tool fragt `heimdall.yourdomain.tld`
2. Resolver liefert AAAA `2001:db8::1`
3. Anwendung versucht IPv6 → Timeout (kein IPv6-Stack im Container)
4. Manche Anwendungen fallen auf A zurück, viele nicht

Die Regex-Regel `.*;querytype=AAAA;reply=nodata` schickt für **alle** AAAA-Anfragen ein leeres NODATA zurück. Resolver fallen sofort auf A zurück, kein Timeout, sauberer Fallback.

Quelle der Regex-Erweiterungen: https://docs.pi-hole.net/regex/pi-hole/

## Anpassen für eigenes Setup

Vor dem Übernehmen der `pihole.toml` mindestens prüfen/anpassen:

| Setting | Hier | Anpassen auf |
|---------|------|--------------|
| `dns.upstreams` | `127.0.0.1#5335` | nur ändern wenn kein Unbound |
| `dns.interface` | `eth0` | ggf. `wlan0` |
| `dns.hosts` | yourdomain.tld-Hostnamen | eigene Hostnamen |
| `dns.revServers` | `10.0.0.0/25` + Fritz!Box | eigenes Subnetz/Router |
| `webserver.api.pwhash` | leer | per `pihole setpassword` setzen |

## Konfig live anwenden

```bash
sudo systemctl stop pihole-FTL
sudo cp pihole.toml /etc/pihole/pihole.toml
sudo systemctl start pihole-FTL
```

Pi-hole validiert die Datei beim Start. Bei Fehler im Log nachschauen:

```bash
sudo journalctl -u pihole-FTL -n 50
```
