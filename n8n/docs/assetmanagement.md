# Assetmanagement Stack

Dieses Verzeichnis enthält die Kern-Infrastruktur für die Datenhaltung des Personal Portfolio Report Projekts.

## Dienste

### postgres
Der zentrale Datenbank-Dienst, der auf PostgreSQL 16 (Alpine-basiert) läuft.

- **Container-Name**: `portfolio-postgres`
- **Image**: `postgres:16-alpine`
- **Netzwerk**: `portfolio_net`
- **Volumes**:
    - `postgres_data`: Persistente Speicherung der Datenbankinhalte.
    - `./init`: Enthält SQL-Skripte zur Initialisierung (Schema und Seed-Daten).

## Datenbank-Initialisierung

Die Datenbank wird automatisch beim ersten Start über die Skripte im Ordner `init/` konfiguriert:

1. `01_schema.sql`: Erstellt das Tabellenschema.
2. `02_seed.sql`: Stammdaten für Assets und Konten.
3. `03_seed_transactions.sql`: Historische Transaktionsdaten.
4. `04_seed_dividends.sql`: Aufzeichnung von Dividendenzahlungen.
5. `05_seed_age.sql`: Österreich-spezifische Korrekturdaten für die Vorabbesteuerung (AgE).

## Docker Compose Konfiguration

Der Stack ist so konfiguriert, dass er robust und wartbar ist. Folgende Optionen sind in der `compose.yml` definiert:

- **Image**: Nutzt `postgres:16-alpine` für geringen Ressourcenverbrauch.
- **Container Name**: Festgelegt auf `portfolio-postgres` zur eindeutigen Identifizierung.
- **Restart Policy**: `unless-stopped` stellt sicher, dass der Dienst nach Fehlern oder Systemneustarts wieder anläuft.
- **Environment**: Nutzt eine `.env`-Datei (`env_file`) zur Trennung von Konfiguration und Code.
- **Volumes**: 
    - Ein benanntes Volume (`postgres_data`) für Persistenz.
    - Ein lokaler Mount (`./init`) für die automatische Schemagerierung beim Start.
- **Networks**: Integration in das interne `portfolio_net`.
- **Healthcheck**: Ein konfigurierter Check (`pg_isready`), der die Verfügbarkeit der Datenbank überwacht (Intervall: 10s, Timeout: 5s, Retries: 5).

## Architektur-Philosophie und Insights

### Source of Truth
Die Datenbank folgt dem Prinzip, dass der Bestand (`quantity`) und der Einstandspreis (`buy_in`) nicht statisch gespeichert, sondern dynamisch aus der Tabelle `transactions` berechnet werden. Dies verhindert Inkonsistenzen und ermöglicht eine lückenlose Historie.

### Steuerliche Logik (Österreich)
Das Schema ist speziell auf das österreichische Steuerrecht ausgelegt:
- **Einstandspreis-Berechnung**: Gebühren (`fees`) werden steuerlich berücksichtigt und erhöhen den Einstandspreis (`SUM(quantity * price + fees) / SUM(quantity)`).
- **AgE (Ausschüttungsgleiche Erträge)**: Die Tabelle `etf_age_adjustments` speichert Korrekturwerte, die den steuerlichen Einstandspreis jährlich anpassen, um eine Doppelbesteuerung beim Verkauf zu vermeiden.
- **Kapitalmaßnahmen**: Splits oder Bonus-Aktien werden als Transaktionen mit `price_per_unit = 0` abgebildet.

### Gold-Abbildung
Gold-Positionen teilen sich denselben Ticker (`GC=F`), werden aber durch die `quantity_unit` (`g` oder `oz`) unterschieden. Die API übernimmt die Umrechnung, während die DB die physischen Einheiten speichert.

## Datenbankschema

Die Datenbank nutzt mehrere Enums und Tabellen, um das Portfolio abzubilden.

### Enums (Benutzerdefinierte Typen)

- `asset_class_type`: `ETF`, `Aktie`, `Krypto`, `Gold`, `Cash`
- `basket_type`: `Core`, `Diversified Tech ex USA`, `US Big Tech Overweight`, `High Risk Stocks`, `Krypto`
- `account_type`: `Depot`, `Wallet`, `Sparkonto`, `Girokonto`, `Bundesschatz`, `Lager`

### Tabellen

#### accounts
Verwaltet die verschiedenen Depots, Wallets und Konten.
- `id`: Eindeutige Kennung.
- `name`: Name des Kontos (z. B. "Flatex Depot").
- `type`: Art des Kontos (`account_type`).

#### account_assets
Stammdaten der im Portfolio gehaltenen Vermögenswerte. Der aktuelle Bestand wird dynamisch aus den Transaktionen berechnet.
- `id`: Eindeutige Kennung.
- `name`: Bezeichnung des Assets.
- `isin` / `wkn`: Standard-Identifikatoren.
- `yahoo_ticker`: Ticker-Symbol für die Preisabfrage (z. B. `VWCE.DE`).
- `asset_class`: Klassifizierung des Assets.
- `basket`: Strategische Zuordnung.
- `account_id`: Verknüpfung mit einem Konto (`accounts`).
- `currency`: Handelswährung (Standard: `EUR`).
- `quantity_unit`: Einheit der Menge (Standard: `share`, für Gold z. B. `g`).
- `investment_thesis`: Freitext zur Begründung des Investments.

#### watchlist
Beobachtungsliste für potenzielle Investments. Ähnlicher Aufbau wie `account_assets`, aber ohne Kontenzuordnung.

#### transactions
Die "Quelle der Wahrheit" für Bestände und Einstandspreise.
- `asset_id`: Verknüpfung zum Asset.
- `type`: Art der Transaktion (`BUY`, `SELL`, `SPLIT`).
- `quantity`: Menge (Anzahl der Anteile).
- `price_per_unit`: Preis pro Einheit zum Zeitpunkt der Transaktion.
- `fees`: Transaktionsgebühren (erhöhen bei Käufen den steuerlichen Einstandspreis).
- `transaction_date`: Datum der Ausführung.

#### dividends
Dokumentation von Brutto-Dividendenzahlungen.
- `asset_id`: Zugehöriges Asset.
- `amount`: Gesamtbetrag der Ausschüttung.
- `transaction_date`: Datum des Zuflusses.

#### etf_age_adjustments
Spezifisch für Österreich: Erfassung von ausschüttungsgleichen Erträgen (AgE).
- `fiscal_year`: Das Steuerjahr der Meldung.
- `age_per_unit`: Betrag pro Anteil, um den der steuerliche Einstandspreis erhöht werden muss.
- `tax_paid`: Gezahlte KESt auf diese Erträge.

## Umgebungsvariablen

Die Konfiguration erfolgt über die `.env`-Datei:

| Variable | Beschreibung |
|---|---|
| `POSTGRES_DB` | Name der Datenbank (Standard: `portfolio`) |
| `POSTGRES_USER` | Benutzername für den Datenbankzugriff |
| `POSTGRES_PASSWORD` | Passwort für den Datenbankbenutzer |

## Netzwerk

Der Stack nutzt das interne Bridge-Netzwerk `portfolio_net`, um eine sichere Kommunikation zwischen der Datenbank, der API und n8n zu ermöglichen.
