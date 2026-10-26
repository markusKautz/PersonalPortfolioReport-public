# Personal Portfolio Report

Automatisierter wöchentlicher Börsenbrief für ein privates Core-Satellite-Portfolio. Jeden Samstag um 08:00 sammelt ein n8n-Workflow Bestände, Kurse, News und die Anlagestrategie, lässt Claude Opus den Fließtext schreiben und schickt den fertigen Bericht als HTML-Datei per Telegram.

Kernidee: **Das LLM berührt keine Zahl.** Alle Kennzahlen und Tabellen werden deterministisch berechnet und gesetzt. Opus schreibt nur Prosa und Platzhalter.

## Workflow

```mermaid
flowchart LR
    cron["Cron<br/>Sa 08:00"] --> pg[("PostgreSQL<br/>Bestand + Watchlist")]

    pg --> api["portfolio-report-api<br/>/fetch-prices<br/>Kurse, GD50/200, CAGR,<br/>Volatilität, Sharpe (EUR)"]
    pg --> tav["Tavily Search<br/>News, Earnings, Makro<br/>(7-Tage-Fenster)"]
    drive["Google Drive<br/>Strategiepapier"]

    api --> combine
    tav --> combine
    drive --> combine

    combine["Combine (Code)<br/>Werte, Performance,<br/>Gewichte, Baskets"] --> opus["Claude Opus<br/>nur Fließtext +<br/>Platzhalter + THESIS-Block"]
    opus --> assembly["Assembly (Code)<br/>Tabellen deterministisch<br/>einsetzen"]
    assembly --> html["portfolio-report-api<br/>/generate-html<br/>Pandoc + CSS-Template"]
    html --> tg["Telegram<br/>HTML-Datei"]
```

| Schritt | Aufgabe |
|---|---|
| PostgreSQL | Lädt Bestand und Watchlist in einer Abfrage (`UNION ALL`). Menge und Einstandspreis werden aus `transactions` berechnet |
| portfolio-report-api | FastAPI + yfinance. Liefert alle Preise und Metriken in EUR (Frankfurt-Ticker, USD-Assets über `EURUSD=X` umgerechnet) und klassifiziert den Trend (bullish, correction, weakening, ...) |
| Tavily | News je Position plus Makro- und Zinsthemen |
| Combine | Berechnet Positionswert, Performance, Gewichtung, Wochen-Performance und Basket-Aggregate |
| Claude Opus | Schreibt den Bericht als Markdown mit Platzhaltern wie `[[TABELLE:EINZELWERTE]]` und `[[GESAMTWERT]]` sowie einem unsichtbaren Block mit Thesis-Verdikten (intakt / angeschlagen / gebrochen) |
| Assembly | Baut die Tabellen im deutschen Zahlenformat und ersetzt die Platzhalter |
| /generate-html | Pandoc + CSS/JS-Template, farbcodierte Trend-, Thesis- und Performance-Werte |

## Datenmodell

```mermaid
erDiagram
    accounts ||--o{ account_assets : "hält"
    account_assets ||--o{ transactions : "hat"
    account_assets ||--o{ dividends : "zahlt"
    account_assets ||--o{ etf_age_adjustments : "hat"

    accounts {
        int id PK
        string name
        enum type "Depot, Wallet, Lager, ..."
    }
    account_assets {
        int id PK
        string name
        string isin
        string yahoo_ticker
        enum asset_class
        enum basket
        int account_id FK
        string quantity_unit "share, g, oz"
        text investment_thesis
    }
    transactions {
        int id PK
        int asset_id FK
        enum type "BUY, SELL, SPLIT"
        decimal quantity
        decimal price_per_unit
        decimal fees
        date transaction_date
    }
    dividends {
        int id PK
        int asset_id FK
        decimal amount
        date transaction_date
    }
    etf_age_adjustments {
        int id PK
        int asset_id FK
        int fiscal_year
        decimal age_per_unit
        decimal tax_paid
    }
    watchlist {
        int id PK
        string name
        string yahoo_ticker
        enum basket
        text investment_thesis
    }
```

- **Bestand und Einstandspreis** werden nicht gespeichert, sondern aus `transactions` abgeleitet. Gebühren erhöhen den Einstandspreis (österreichische Steuerlogik), Splits sind Transaktionen mit `price_per_unit = 0`.
- **AgE** (ausschüttungsgleiche Erträge thesaurierender ETFs) erhöhen den steuerlichen Einstandspreis und verhindern so eine Doppelbesteuerung beim Verkauf.
- `watchlist` ist ähnlich aufgebaut wie `account_assets`, hat aber kein Konto.

## Infrastruktur

Alle Dienste laufen als Docker-Stacks auf einem VPS und sprechen über das interne Netzwerk `portfolio_net` miteinander. Nach außen ist nur n8n über Caddy (`caddy_net`) erreichbar.

```mermaid
flowchart LR
    caddy["Caddy<br/>Reverse Proxy"] -- caddy_net --> n8n
    subgraph portfolio_net
        n8n["n8n"]
        api["portfolio-report-api<br/>FastAPI, yfinance, Pandoc"]
        db[("portfolio-postgres<br/>PostgreSQL 16")]
    end
    n8n --> api
    n8n --> db
```

## Repository-Inhalt

| Pfad | Inhalt |
|---|---|
| `n8n/Portfolio_Report_Weekly_2.json` | Exportierter n8n-Workflow |
| `n8n/docs/n8n.md` | n8n-Stack, Kosten, Designentscheidungen |
| `n8n/docs/portfolio-report-api.md` | API-Stack, EUR-Konsistenz, HTML-Rendering |
| `n8n/docs/assetmanagement.md` | PostgreSQL-Stack und Datenbankschema |

API-Code, Docker-Compose-Dateien und das SQL-Schema liegen in einem separaten Infrastruktur-Repository.

## Kosten

Bei 15 Positionen und wöchentlichem Report ca. **0,38 USD pro Woche**. Tavily kommt mit dem kostenlosen Researcher-Tier aus, yfinance ist ebenfalls kostenlos, die Kosten entfallen damit im Wesentlichen auf Claude Opus.

Mehr Positionen oder häufigere Läufe erhöhen die Kosten entsprechend, sowohl beim Opus-Aufruf als auch bei den Tavily-Suchanfragen (dann ggf. über das Freikontingent hinaus).
