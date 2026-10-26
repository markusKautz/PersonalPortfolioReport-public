# Portfolio Report API Stack

This stack provides a specialized API for retrieving market data and generating the final reports.

## Services

### portfolio-report-api
A Python-based FastAPI application.

- **Container name**: `portfolio-report-api`
- **Build**: Based on the local `Dockerfile`.
- **Network**: `portfolio_net` (external).
- **Volumes**:
    - `./scripts`: Maps the Python scripts into the container (`/app/scripts`), read-only.

## Docker Compose Configuration

The API is built locally to allow flexibility with dependencies:

- **Build**: Uses the local `Dockerfile` in the current directory.
- **Container name**: `portfolio-report-api`.
- **Restart policy**: `unless-stopped`.
- **Environment**: Uses a `.env` file (`env_file`) for application-specific parameters.
- **Volumes**: A local mount (`./scripts`) maps the Python logic into the container (`ro` - read-only for improved security).
- **Networks**: Connects to the **external** network `portfolio_net` so that it is reachable from n8n.

## Architecture Philosophy and Insights

### EUR Consistency Strategy
A core principle of the API is **currency consistency**. All data is delivered uniformly in EUR to avoid complexity in downstream systems (n8n/AI):
- **Preference for Frankfurt tickers**: Where possible, assets are queried via Frankfurt (`.F` / `.DE`) to get EUR prices directly.
- **In-place FX conversion**: Assets such as gold (`GC=F`) that are primarily quoted in USD are converted internally using current FX rates (`EURUSD=X`).
- **Benefit**: The `Combine` node in n8n no longer needs any currency logic; it always works with `price` in EUR.

### Technical Design Decisions
- **Pydantic vs. dict**: The API uses `dict[str, Any]` instead of strict Pydantic models for data exchange. This was chosen because Pydantic models (even with `extra="allow"`) often dropped additional fields during serialization (`model_dump`). `dict` provides a more reliable passthrough for dynamic data here.
- **Reporting**: Instead of complex PDF generation, the API relies on HTML with modern CSS features. This enables horizontal scrolling for tables and simpler dynamic coloring (trend/performance) via JS/CSS.

## Functionality

The API offers two main functions:

1. **Market data retrieval**: Uses `yfinance` to compute current prices, moving averages (SMA50, SMA200) and performance metrics (CAGR, volatility, Sharpe ratio). All prices are delivered uniformly in EUR.
2. **HTML generation**: Uses Pandoc and WeasyPrint to create visually appealing HTML reports from structured data (JSON), based on the `report.html` template.

## Environment Variables

Configuration is done via the `.env` file:

| Variable | Description |
|---|---|
| `RISK_FREE_RATE` | The risk-free rate used to compute the Sharpe ratio (e.g. `0.03` for 3%). |

## Development and Deployment

- **Dockerfile**: Installs required system dependencies such as `pandoc`, `weasyprint` and the Python libraries from `requirements.txt`.
- **Scripts**:
    - `api.py`: Defines the FastAPI endpoints.
    - `fetch_prices.py`: Logic for querying ticker symbols and FX conversion.
    - `templates/report.html`: The CSS/HTML skeleton for the report.
