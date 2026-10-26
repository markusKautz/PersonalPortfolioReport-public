# n8n Stack

n8n dient als zentrale Workflow-Automatisierung für das Projekt. Es steuert den Ablauf von der Datenabfrage bis zum Versand des Berichts.

## Dienste

### n8n
Die Hauptanwendung für die Workflow-Orchestrierung.

- **Container-Name**: `n8n`
- **Image**: `docker.n8n.io/n8nio/n8n`
- **Netzwerke**:
    - `portfolio_net`: Für den Zugriff auf die PostgreSQL-Datenbank und die interne API.
    - `caddy_net`: Ermöglicht den externen Zugriff über den Caddy-Reverse-Proxy (SSL/HTTPS).
- **Volumes**:
    - `n8n_data`: Persistente Speicherung von n8n-Konfigurationen und Datenbank (standardmäßig SQLite intern).
    - `./local-files`: Mapping für den Zugriff auf lokale Dateien (z. B. Exporte oder temporäre Berichte).

## Umgebungsvariablen

Die Konfiguration wird über die `.env`-Datei gesteuert. Wichtige Variablen sind:

| Variable | Beschreibung |
|---|---|
| `GENERIC_TIMEZONE` | Zeitzone für Cron-Jobs (z. B. `Europe/Vienna`). |
| `N8N_ENCRYPTION_KEY` | Schlüssel zur Verschlüsselung von Zugangsdaten (Credentials). |
| `N8N_HOST` | Der Hostname, unter dem n8n erreichbar ist. |
| `N8N_PROTOCOL` | Protokoll (meist `https`). |

## Docker Compose Konfiguration

n8n ist für den Betrieb hinter einem Proxy und die Interaktion mit lokalen Dateien konfiguriert:

- **Image**: Offizielles `docker.n8n.io/n8nio/n8n`.
- **Container Name**: `n8n`.
- **Restart Policy**: `unless-stopped`.
- **Environment**:
    - Kombiniert eine `.env`-Datei (`env_file`) mit expliziten Umgebungsvariablen in der `compose.yml`.
    - Konfiguriert Protokoll (`https`), Hostnamen, Webhook-URLs und Proxy-Vertrauen (`N8N_TRUST_PROXY`).
- **Volumes**:
    - Benanntes Volume (`n8n_data`) für die Anwendungsdaten.
    - Lokaler Mount (`./local-files`) für Dateisystem-Operationen innerhalb von Workflows.
- **Networks**:
    - Verbindet sich mit zwei **externen** Netzwerken: `caddy_net` (für den Ingress-Traffic) und `portfolio_net` (für den Zugriff auf DB und API).

## Architektur-Philosophie und Insights

### KI-Strategie: Opus Direkt
Ursprünglich war ein zweistufiges Modell geplant (Sonnet zur Vorverarbeitung, Opus für den finalen Text). In der Praxis hat sich gezeigt, dass der kombinierte Daten-JSON (50–100 KB) klein genug für Claude Opus ist. Der Verzicht auf den Zwischenschritt reduziert Latenz, Kosten und Komplexität.

### Kostenstruktur
Bei 15 Positionen und wöchentlichem Report verursacht der Workflow laufende Kosten von ca. **0,38 USD pro Woche**:
- **Claude Opus**: Praktisch die gesamten Kosten, für die Generierung des Berichts.
- **Tavily Search**: Kostenlos, das Researcher-Tier (Free) reicht für ~48 Queries pro Woche (News/Earnings/Makro).
- **yfinance**: Kostenlos.

Mehr Positionen oder häufigere Läufe erhöhen die Kosten entsprechend, bei Tavily ggf. über das Freikontingent hinaus.

### Workflow-Optimierung
Um Race Conditions beim Zusammenführen von Daten zu vermeiden, wurde die Abfrage von Bestands- und Watchlist-Werten in einem einzigen PostgreSQL-Node via `UNION ALL` gelöst. Ein Diskriminator-Feld (`record_type`) ermöglicht die spätere Unterscheidung im Workflow.

## Workflow-Funktion

Der n8n-Workflow führt folgende Hauptschritte aus (laut `CLAUDE.md`):
1. **Trigger**: Jeden Samstag um 08:00 Uhr.
2. **Datenabruf**: Liest Bestandsdaten und Watchlist aus der PostgreSQL-Datenbank.
3. **Anreicherung**: Ruft Kurse über die `portfolio-report-api` ab und sucht News via Tavily.
4. **KI-Verarbeitung**: Claude Opus erstellt den Berichtstext basierend auf den gesammelten Daten.
5. **Berichtserstellung**: Generiert HTML über die API.
6. **Versand**: Sendet das Ergebnis via Telegram Bot.
