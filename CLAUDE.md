# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-powered stock analysis system for A-shares (Chinese market), Hong Kong stocks, and US stocks. Python 3.10+ backend with FastAPI REST API, multi-LLM analysis (Gemini/Claude/OpenAI), 6 data providers with automatic failover, and a React 19 frontend.

## Commands

### Backend

```bash
# Install dependencies
pip install -r requirements.txt

# Run analysis
python main.py                              # Full analysis (stocks from .env STOCK_LIST)
python main.py --stocks 600519,hk00700,AAPL # Specific stocks
python main.py --dry-run --no-notify        # Data only, no AI analysis or notifications
python main.py --market-review              # Market overview only
python main.py --backtest --backtest-code 600519  # Backtesting

# Start API server
python main.py --serve-only --port 8000
# or: uvicorn server:app --reload

# Start with Web UI
python main.py --webui

# Linting & syntax checks
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
python -m py_compile main.py src/*.py data_provider/*.py

# Run tests
python -m pytest                    # All tests
python -m pytest -m "not network"   # Skip network-dependent tests (CI default)
python -m pytest -m unit            # Unit tests only
python -m pytest tests/test_storage.py::test_function_name  # Single test

# CI gate (syntax + flake8 + offline tests)
./scripts/ci_gate.sh

# Integration test scenarios (require .env with API keys)
./test.sh quick       # Single stock quick test
./test.sh dry-run     # Data fetch without AI
./test.sh syntax      # Python syntax check
./test.sh all         # All scenarios
```

### Frontend (`apps/dsa-web/`)

```bash
cd apps/dsa-web
npm install
npm run dev       # Dev server (Vite, port 5173)
npm run build     # Production build (tsc + vite)
npm run lint      # ESLint
```

## Architecture

### Analysis Pipeline

`main.py` → `StockAnalysisPipeline` (src/core/pipeline.py) orchestrates the flow:
1. **Data Fetch** — `DataFetcherManager` (data_provider/base.py) uses Strategy Pattern with 6 providers, auto-failover by priority
2. **Trend Analysis** — `StockTrendAnalyzer` (src/stock_analyzer.py) computes technical indicators (MA, MACD, bias ratio, PE)
3. **News Search** — `SearchService` (src/search_service.py) aggregates from Tavily/SerpAPI/Bocha/Brave with freshness filtering
4. **AI Analysis** — `GeminiAnalyzer` (src/analyzer.py) supports Gemini (primary), Claude, OpenAI-compatible APIs
5. **Notification** — `NotificationService` (src/notification.py) delivers via WeChat/Feishu/Telegram/Email/Discord/DingTalk/webhooks
6. **Storage** — SQLite via SQLAlchemy (src/storage.py)

### Data Provider Priority (Strategy Pattern)

Priority 0: efinance (default) / tushare (when TUSHARE_TOKEN set)
Priority 1: akshare
Priority 2: pytdx, tushare (without token)
Priority 3: baostock
Priority 4: yfinance (fallback)

Each fetcher implements `BaseFetcher`. The manager tries providers in priority order with exponential backoff and rate limiting.

### Stock Code Conventions

- A-shares: 6-digit numeric (`600519`, `000001`, `300750`)
- Hong Kong: `hk` prefix + 5 digits (`hk00700`, `hk09988`)
- US stocks: ticker symbol (`AAPL`, `TSLA`, `BRK.B`)
- ETFs: 6-digit numeric same as A-shares (`563230`, `512400`)
- `normalize_stock_code()` in data_provider/base.py strips exchange prefixes/suffixes

### API Layer

FastAPI app factory in `api/app.py` → routes in `api/v1/endpoints/`. Schemas in `api/v1/schemas/`. Dependency injection via `api/deps.py`. Service layer in `src/services/`, repository layer in `src/repositories/`.

### Bot Framework

Multi-platform bot in `bot/` — `dispatcher.py` handles command routing, `bot/commands/` contains command implementations. Platforms: Telegram, Discord, Feishu, DingTalk, WeChat (each in `bot/platforms/`).

### Frontend

React 19 + Vite + TailwindCSS 4 + Zustand for state. Located in `apps/dsa-web/`. API client in `src/api/`. Backend serves the built frontend from `static/` directory in production.

## Code Style

- **Python**: black (line-length=120), isort (profile=black), flake8 (ignores: E501, W503, E203, E402)
- **Target versions**: Python 3.10/3.11/3.12
- **Comments in code**: English only
- **Commit messages**: English only, conventional commits format (`feat:`, `fix:`, `refactor:`, etc.)
- **No `Co-Authored-By`** in commits (per AGENTS.md)
- Version tags in commit messages: `#patch`, `#minor`, `#major`, `#skip`

## Test Markers

```ini
# setup.cfg
unit        — fast offline unit tests
integration — service-level, no external network
network     — requires external network or third-party services
```

CI runs `python -m pytest -m "not network"` to avoid flaky external dependencies.

## Key Environment Variables

Configured via `.env` (see `.env.example` for full list):

- `STOCK_LIST` — comma-separated stock codes
- `GEMINI_API_KEY` / `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` — at least one AI provider required
- `TUSHARE_TOKEN` — elevates tushare to Priority 0 data source
- `TAVILY_API_KEYS` / `SERPAPI_API_KEYS` / `BOCHA_API_KEYS` — news search providers
- `NEWS_MAX_AGE_DAYS` (default: 3) — news freshness filter
- `BIAS_THRESHOLD` (default: 5.0) — deviation rate threshold for buy signals
- `REPORT_TYPE` (simple/full) — analysis report detail level
- `REPORT_SUMMARY_ONLY` — output summary table only
- `SCHEDULE_ENABLED` / `SCHEDULE_TIME` — scheduled analysis
- `WEBUI_HOST` / `WEBUI_PORT` — server binding

## Docker

```bash
docker-compose -f docker/docker-compose.yml up -d
```

Multi-stage build: Node 20 (frontend) → Python 3.11-slim (backend). Timezone set to Asia/Shanghai.
