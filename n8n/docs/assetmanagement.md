# Asset Management Stack

This directory contains the core data storage infrastructure of the Personal Portfolio Report project.

## Services

### postgres
The central database service, running PostgreSQL 16 (Alpine-based).

- **Container name**: `portfolio-postgres`
- **Image**: `postgres:16-alpine`
- **Network**: `portfolio_net`
- **Volumes**:
    - `postgres_data`: Persistent storage of database contents.
    - `./init`: Contains SQL scripts for initialization (schema and seed data).

## Database Initialization

The database is configured automatically on first start via the scripts in the `init/` folder:

1. `01_schema.sql`: Creates the table schema.
2. `02_seed.sql`: Master data for assets and accounts.
3. `03_seed_transactions.sql`: Historical transaction data.
4. `04_seed_dividends.sql`: Record of dividend payments.
5. `05_seed_age.sql`: Austria-specific adjustment data for advance taxation (AgE).

## Docker Compose Configuration

The stack is configured to be robust and maintainable. The following options are defined in `compose.yml`:

- **Image**: Uses `postgres:16-alpine` for low resource usage.
- **Container name**: Fixed to `portfolio-postgres` for unambiguous identification.
- **Restart policy**: `unless-stopped` ensures the service comes back up after failures or system reboots.
- **Environment**: Uses a `.env` file (`env_file`) to separate configuration from code.
- **Volumes**:
    - A named volume (`postgres_data`) for persistence.
    - A local mount (`./init`) for automatic schema generation on startup.
- **Networks**: Integration into the internal `portfolio_net`.
- **Healthcheck**: A configured check (`pg_isready`) that monitors database availability (interval: 10s, timeout: 5s, retries: 5).

## Architecture Philosophy and Insights

### Source of Truth
The database follows the principle that holdings (`quantity`) and cost basis (`buy_in`) are not stored statically but computed dynamically from the `transactions` table. This prevents inconsistencies and provides a complete history.

### Tax Logic (Austria)
The schema is designed specifically for Austrian tax law:
- **Cost basis calculation**: Fees (`fees`) are taken into account for tax purposes and increase the cost basis (`SUM(quantity * price + fees) / SUM(quantity)`).
- **AgE (*ausschüttungsgleiche Erträge*, deemed distributions)**: The `etf_age_adjustments` table stores adjustment values that raise the tax cost basis each year to avoid double taxation on sale.
- **Corporate actions**: Splits or bonus shares are recorded as transactions with `price_per_unit = 0`.

### Gold Representation
Gold positions share the same ticker (`GC=F`) but are distinguished by `quantity_unit` (`g` or `oz`). The API handles the conversion, while the DB stores the physical units.

## Database Schema

The database uses several enums and tables to model the portfolio.

### Enums (Custom Types)

- `asset_class_type`: `ETF`, `Aktie`, `Krypto`, `Gold`, `Cash`
- `basket_type`: `Core`, `Diversified Tech ex USA`, `US Big Tech Overweight`, `High Risk Stocks`, `Krypto`
- `account_type`: `Depot`, `Wallet`, `Sparkonto`, `Girokonto`, `Bundesschatz`, `Lager`

### Tables

#### accounts
Manages the various brokerage accounts, wallets and bank accounts.
- `id`: Unique identifier.
- `name`: Name of the account (e.g. "Flatex Depot").
- `type`: Type of account (`account_type`).

#### account_assets
Master data of the assets held in the portfolio. The current holding is computed dynamically from the transactions.
- `id`: Unique identifier.
- `name`: Name of the asset.
- `isin` / `wkn`: Standard identifiers.
- `yahoo_ticker`: Ticker symbol for price retrieval (e.g. `VWCE.DE`).
- `asset_class`: Classification of the asset.
- `basket`: Strategic allocation.
- `account_id`: Link to an account (`accounts`).
- `currency`: Trading currency (default: `EUR`).
- `quantity_unit`: Unit of quantity (default: `share`, for gold e.g. `g`).
- `investment_thesis`: Free text explaining the rationale for the investment.

#### watchlist
Watchlist for potential investments. Similar structure to `account_assets`, but without an account assignment.

#### transactions
The "source of truth" for holdings and cost basis.
- `asset_id`: Link to the asset.
- `type`: Type of transaction (`BUY`, `SELL`, `SPLIT`).
- `quantity`: Quantity (number of units).
- `price_per_unit`: Price per unit at the time of the transaction.
- `fees`: Transaction fees (increase the tax cost basis on purchases).
- `transaction_date`: Date of execution.

#### dividends
Record of gross dividend payments.
- `asset_id`: Associated asset.
- `amount`: Total amount of the distribution.
- `transaction_date`: Date of receipt.

#### etf_age_adjustments
Austria-specific: records deemed distributions (AgE).
- `fiscal_year`: The tax year of the report.
- `age_per_unit`: Amount per unit by which the tax cost basis must be increased.
- `tax_paid`: Capital gains tax (KESt) paid on these income amounts.

## Environment Variables

Configuration is done via the `.env` file:

| Variable | Description |
|---|---|
| `POSTGRES_DB` | Name of the database (default: `portfolio`) |
| `POSTGRES_USER` | Username for database access |
| `POSTGRES_PASSWORD` | Password for the database user |

## Network

The stack uses the internal bridge network `portfolio_net` to enable secure communication between the database, the API and n8n.
