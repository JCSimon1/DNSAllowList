# DNS Allowlist Management

Dieses Repository verwaltet die DNS-Allowlist für zwei DNS-Resolver im Homelab:

- **Technitium DNS**
- **AdGuard Home**

Die zentrale Quelldatei ist `Allow-list`. Daraus wird automatisch eine für AdGuard Home kompatible Version erzeugt.

## Architektur

```text
                         GitHub
                           │
                    Allow-list
                     (Quelle)
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
        Technitium DNS           GitHub Action
        direkte Verwendung             │
                                      ▼
                              Allow-list-AdGuard
                                      │
                                      ▼
                               AdGuard Home
```

Beide DNS-Server werden im LAN per DHCP verteilt und verwenden identische Blocklisten. Dadurch bleibt die DNS-Filterung auch beim Ausfall eines der beiden Resolver verfügbar.

## Dateien

```text
DNSAllowList/
├── Allow-list
├── Allow-list-AdGuard
├── scripts/
│   └── convert_allowlist.py
└── .github/
    └── workflows/
        └── convert-allowlist.yml
```

### `Allow-list`

Dies ist die **Single Source of Truth**. Die Datei wird manuell gepflegt und enthält normale Domains, Regex-Regeln, Kommentare und Leerzeilen.

### `Allow-list-AdGuard`

Diese Datei wird **automatisch generiert** und darf nicht manuell bearbeitet werden.

Normale Domains werden z. B. so konvertiert:

```text
github.com
```

zu:

```text
@@||github.com^
```

Regex-Regeln werden in AdGuard-Regex-Exceptions umgewandelt, mit `$important`, damit die Allowlist gegenüber normalen Blockierungsregeln Vorrang erhält.

## Pflege der Allowlist

Das Repository wird lokal mit Visual Studio Code gepflegt.

```powershell
git clone git@github.com:JCSimon1/DNSAllowList.git
cd DNSAllowList
code .
```

Änderungen prüfen:

```powershell
git diff
```

Committen und zu GitHub übertragen:

```powershell
git add Allow-list
git commit -m "Update DNS allowlist"
git push origin main
```

## Automatische Konvertierung

Bei einem Push auf `main`, der `Allow-list`, den Konverter oder den Workflow betrifft, startet automatisch:

```text
.github/workflows/convert-allowlist.yml
```

Die GitHub Action:

1. checkt das Repository aus
2. startet Python
3. führt `scripts/convert_allowlist.py` aus
4. prüft die generierte Datei
5. erkennt Änderungen
6. committed die neue `Allow-list-AdGuard`
7. pusht den Commit zurück nach GitHub

Der Workflow kann außerdem manuell über **GitHub → Actions → Convert Allow-list for AdGuard Home → Run workflow** gestartet werden.

## AdGuard Home

In AdGuard Home wird die generierte Datei als DNS-Allowlist eingebunden:

```text
https://raw.githubusercontent.com/JCSimon1/DNSAllowList/main/Allow-list-AdGuard
```

In AdGuard Home:

**Filters → DNS allowlists → Add allowlist**

Die Raw-URL als Quelle eintragen.

## Technitium DNS

Technitium verwendet weiterhin direkt die ursprüngliche:

```text
Allow-list
```

Damit bleibt die bestehende Technitium-Syntax unverändert.

## Regeltypen

### Normale Domain

Quelle:

```text
example.com
```

AdGuard:

```text
@@||example.com^
```

### Regex

Quelle:

```text
^((alt)[0-9](-))?mtalk\.google\.com$
```

AdGuard:

```text
@@/^((alt)[0-9](-))?mtalk\.google\.com$/$important
```

### Kommentare

Kommentare werden in der generierten Datei erhalten.

```text
amd.com # AMD website
```

wird zu:

```text
@@||amd.com^ # AMD website
```

## Lokaler Test

Der Konverter kann vor einem Commit lokal getestet werden:

```powershell
python .\scripts\convert_allowlist.py
```

Eine erfolgreiche Ausführung meldet:

```text
Conversion successful.
```

Die generierte Datei kann anschließend geprüft werden:

```powershell
Get-Content .\Allow-list-AdGuard | Select-Object -First 30
Get-Content .\Allow-list-AdGuard | Select-Object -Last 30
```

## GitHub SSH

Das Repository verwendet SSH:

```text
git@github.com:JCSimon1/DNSAllowList.git
```

Der verwendete SSH-Key ist:

```text
~/.ssh/github_key
```

Relevanter Eintrag in `~/.ssh/config`:

```text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
```

SSH testen:

```powershell
ssh -T git@github.com
```

Erwartete Antwort:

```text
Hi JCSimon1! You've successfully authenticated, but GitHub does not provide shell access.
```

Git-Remote prüfen:

```powershell
git remote -v
```

Erwartet:

```text
origin  git@github.com:JCSimon1/DNSAllowList.git (fetch)
origin  git@github.com:JCSimon1/DNSAllowList.git (push)
```

## Best Practices

- **Nur `Allow-list` manuell bearbeiten.**
- `Allow-list-AdGuard` nicht manuell bearbeiten.
- Vor Änderungen `git diff` verwenden.
- Den Konverter vor größeren Änderungen lokal testen.
- Nach einem Push den GitHub-Action-Lauf prüfen.
- In AdGuard Home kontrollieren, ob die Allowlist erfolgreich aktualisiert wurde.
- Bei neuen Sonderformaten zuerst den Konverter erweitern und testen.

## Aktueller Stand

Die Allowlist enthält normale Domains und Regex-Regeln. Der Konverter verarbeitet die bestehende Liste erfolgreich und erzeugt daraus eine AdGuard-kompatible Allowlist.

Damit ist die manuelle Pflege auf **eine einzige Datei (`Allow-list`)** reduziert. Änderungen werden über GitHub Actions automatisch für AdGuard Home aufbereitet.
