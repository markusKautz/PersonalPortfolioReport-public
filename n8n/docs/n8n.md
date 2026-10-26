# n8n Stack

n8n serves as the central workflow automation for the project. It orchestrates the process from data retrieval to delivery of the report.

## Services

### n8n
The main application for workflow orchestration.

- **Container name**: `n8n`
- **Image**: `docker.n8n.io/n8nio/n8n`
- **Networks**:
    - `portfolio_net`: For access to the PostgreSQL database and the internal API.
    - `caddy_net`: Enables external access via the Caddy reverse proxy (SSL/HTTPS).
- **Volumes**:
    - `n8n_data`: Persistent storage of n8n configuration and database (internal SQLite by default).
    - `./local-files`: Mapping for access to local files (e.g. exports or temporary reports).

## Environment Variables

Configuration is controlled via the `.env` file. Important variables:

| Variable | Description |
|---|---|
| `GENERIC_TIMEZONE` | Time zone for cron jobs (e.g. `Europe/Vienna`). |
| `N8N_ENCRYPTION_KEY` | Key used to encrypt credentials. |
| `N8N_HOST` | The hostname under which n8n is reachable. |
| `N8N_PROTOCOL` | Protocol (usually `https`). |

## Docker Compose Configuration

n8n is configured to run behind a proxy and to interact with local files:

- **Image**: Official `docker.n8n.io/n8nio/n8n`.
- **Container name**: `n8n`.
- **Restart policy**: `unless-stopped`.
- **Environment**:
    - Combines a `.env` file (`env_file`) with explicit environment variables in `compose.yml`.
    - Configures protocol (`https`), hostname, webhook URLs and proxy trust (`N8N_TRUST_PROXY`).
- **Volumes**:
    - Named volume (`n8n_data`) for application data.
    - Local mount (`./local-files`) for file system operations within workflows.
- **Networks**:
    - Connects to two **external** networks: `caddy_net` (for ingress traffic) and `portfolio_net` (for access to DB and API).

## Architecture Philosophy and Insights

### AI Strategy: Opus Direct
Originally a two-stage model was planned (Sonnet for preprocessing, Opus for the final text). In practice, the combined data JSON (50–100 KB) turned out to be small enough for Claude Opus. Dropping the intermediate step reduces latency, cost and complexity.

### Cost Structure
With 15 positions and a weekly report, the workflow incurs running costs of roughly **USD 0.38 per week**:
- **Claude Opus**: Practically the entire cost, for generating the report.
- **Tavily Search**: Free; the Researcher tier (free) covers ~48 queries per week (news/earnings/macro).
- **yfinance**: Free.

More positions or more frequent runs increase the cost accordingly, possibly exceeding the free Tavily quota.

### Workflow Optimization
To avoid race conditions when merging data, holdings and watchlist values are queried in a single PostgreSQL node via `UNION ALL`. A discriminator field (`record_type`) allows them to be told apart later in the workflow.

## Workflow Function

The n8n workflow performs the following main steps:
1. **Trigger**: Every Saturday at 08:00.
2. **Data retrieval**: Reads holdings and watchlist from the PostgreSQL database.
3. **Enrichment**: Fetches prices via the `portfolio-report-api` and searches news via Tavily.
4. **AI processing**: Claude Opus writes the report text based on the collected data.
5. **Report generation**: Generates HTML via the API.
6. **Delivery**: Sends the result via Telegram bot.
