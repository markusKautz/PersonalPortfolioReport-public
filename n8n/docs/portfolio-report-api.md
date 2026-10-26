# Portfolio Report API Stack

Dieser Stack stellt eine spezialisierte API zur Verfügung, um Marktdaten abzufragen und die finalen Berichte zu generieren.

## Dienste

### portfolio-report-api
Eine FastAPI-Anwendung, die auf Python basiert.

- **Container-Name**: `portfolio-report-api`
- **Build**: Basierend auf dem lokalen `Dockerfile`.
- **Netzwerk**: `portfolio_net` (extern).
- **Volumes**:
    - `./scripts`: Mapping der Python-Skripte in den Container (`/app/scripts`), read-only.

## Docker Compose Konfiguration

Die API wird lokal gebaut, um Flexibilität bei den Abhängigkeiten zu gewährleisten:

- **Build**: Nutzt das lokale `Dockerfile` im aktuellen Verzeichnis.
- **Container Name**: `portfolio-report-api`.
- **Restart Policy**: `unless-stopped`.
- **Environment**: Nutzt eine `.env`-Datei (`env_file`) für anwendungsspezifische Parameter.
- **Volumes**: Ein lokaler Mount (`./scripts`) bildet die Python-Logik in den Container ab (`ro` - read only für erhöhte Sicherheit).
- **Networks**: Verbindet sich mit dem **externen** Netzwerk `portfolio_net`, um von n8n erreichbar zu sein.

## Architektur-Philosophie und Insights

### EUR-Konsistenzstrategie
Ein Kernprinzip der API ist die **Währungskonsistenz**. Alle Daten werden einheitlich in EUR geliefert, um Komplexität in nachgelagerten Systemen (n8n/KI) zu vermeiden:
- **Bevorzugung von Frankfurt-Tickern**: Assets werden nach Möglichkeit über Frankfurt (`.F` / `.DE`) abgefragt, um direkt EUR-Kurse zu erhalten.
- **In-place FX-Konvertierung**: Werte wie Gold (`GC=F`), die primär in USD vorliegen, werden intern über aktuelle FX-Kurse (`EURUSD=X`) umgerechnet.
- **Vorteil**: Der `Combine Node` in n8n muss keine Währungslogik mehr implementieren; er rechnet immer mit `price` in EUR.

### Technische Design-Entscheidungen
- **Pydantic vs. dict**: Die API nutzt `dict[str, Any]` statt strikter Pydantic-Modelle für den Datenaustausch. Dies wurde gewählt, da Pydantic-Modelle (selbst mit `extra="allow"`) bei der Serialisierung (`model_dump`) oft zusätzliche Felder ignorierten. `dict` bietet hier ein zuverlässigeres Passthrough für dynamische Daten.
- **Reporting**: Statt komplexer PDF-Generierung wird auf HTML mit modernen CSS-Features gesetzt. Dies ermöglicht horizontales Scrolling für Tabellen und eine einfachere Farbdynamik (Trend/Performance) via JS/CSS.

## Funktionalität

Die API bietet zwei Hauptfunktionen:

1. **Marktdaten-Abruf**: Nutzt `yfinance`, um aktuelle Kurse, gleitende Durchschnitte (GD50, GD200) sowie Performance-Metriken (CAGR, Volatilität, Sharpe Ratio) zu berechnen. Alle Preise werden einheitlich in EUR geliefert.
2. **HTML-Generierung**: Nutzt Pandoc und WeasyPrint, um aus strukturierten Daten (JSON) visuell ansprechende HTML-Berichte basierend auf dem `report.html`-Template zu erstellen.

## Umgebungsvariablen

Die Konfiguration erfolgt über die `.env`-Datei:

| Variable | Beschreibung |
|---|---|
| `RISK_FREE_RATE` | Der risikofreie Zinssatz für die Berechnung der Sharpe Ratio (z. B. `0.03` für 3%). |

## Entwicklung und Deployment

- **Dockerfile**: Installiert notwendige System-Abhängigkeiten wie `pandoc`, `weasyprint` und die Python-Libraries aus `requirements.txt`.
- **Scripts**:
    - `api.py`: Definiert die FastAPI-Endpunkte.
    - `fetch_prices.py`: Logik für die Abfrage und FX-Konvertierung der Ticker-Symbole.
    - `templates/report.html`: Das CSS/HTML-Grundgerüst für den Bericht.
