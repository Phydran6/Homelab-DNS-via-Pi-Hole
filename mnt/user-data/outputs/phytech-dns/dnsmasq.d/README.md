# dnsmasq.d Snippets

Pi-hole nutzt unter der Haube `dnsmasq`. Eigene `.conf`-Snippets in `/etc/dnsmasq.d/` werden beim Start automatisch geladen und überschreiben/ergänzen Pi-holes Verhalten auf einer tieferen Ebene als Regex-Filter in der UI.

## Wozu diese Snippets?

Die Hauptlösung für den AAAA-Block ist die Pi-hole-Regex `.*;querytype=AAAA;reply=nodata` (siehe [../pihole/README.md](../pihole/README.md)). Diese Snippets sind ein **zusätzliches Failsafe-Layer**:

- Greift auch wenn die Regex-Regel aus der `gravity.db` versehentlich gelöscht oder deaktiviert wird.
- Greift, sobald `dnsmasq` läuft — auch wenn Pi-hole-FTL gerade neu lädt.
- Ist enger gefasst (nur `yourdomain.tld`) und damit unkritisch für andere Domains im Netz.

## Dateien

### `99-block-aaaa.conf`

```
address=/.yourdomain.tld/::0
```

Setzt für **alle Subdomains von `yourdomain.tld`** den AAAA-Record auf `::` (IPv6-Null-Adresse). Dnsmasq antwortet bei AAAA-Anfragen direkt damit, ohne Upstream zu fragen.

### `99-noipv6.conf`

```
address=/yourdomain.tld/::
```

Macht dasselbe für die Apex-Domain `yourdomain.tld` selbst (ohne Subdomain-Prefix).

## Hinweis zum Verhalten

Mit `address=/domain/::` antwortet dnsmasq auf AAAA-Queries mit `::` statt mit NODATA. Das ist **nicht ganz so sauber** wie der `reply=nodata` der Pi-hole-Regex, weil manche Resolver `::` als gültige IPv6-Adresse interpretieren und versuchen, dorthin zu verbinden (was natürlich scheitert).

Deshalb: **die Pi-hole-Regex ist die primäre Lösung**, diese Snippets sind nur Backup. Wenn die Regex aktiv ist, greift sie zuerst und liefert das saubere NODATA.

## Installation

```bash
sudo cp 99-*.conf /etc/dnsmasq.d/
sudo pihole reloaddns
```

## Test

```bash
dig yourdomain.tld AAAA           @10.0.0.30   # → :: oder NODATA
dig heimdall.yourdomain.tld AAAA  @10.0.0.30   # → NODATA (durch Pi-hole-Regex)
                                                 #   oder :: (durch dieses Snippet)
dig heimdall.yourdomain.tld A     @10.0.0.30   # → 10.0.0.20 (unverändert)
```

## Anpassung für andere Domains

Statt `yourdomain.tld` einfach die eigene Domain einsetzen:

```
address=/.deinedomain.de/::0
address=/deinedomain.de/::
```

Oder für netzweit AAAA komplett aus (alle Domains):

```
filter-AAAA
```

(letzteres würde aber Pi-holes Regex überflüssig machen — dann lieber nur eines der beiden Verfahren nutzen).

## Entfernen

```bash
sudo rm /etc/dnsmasq.d/99-block-aaaa.conf
sudo rm /etc/dnsmasq.d/99-noipv6.conf
sudo pihole reloaddns
```
