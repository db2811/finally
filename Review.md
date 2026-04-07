# FinAlly — Production-Grade Trading Platform: Review & Build Plan

## Current State Summary

The repository is approximately **20% complete**. The market data subsystem (simulator, Massive API client, price cache, SSE stream factory) is production-ready with unit tests. Everything else — FastAPI app, database, portfolio logic, LLM integration, frontend, Docker orchestration, and E2E tests — remains to be built.

| Component | Status |
|---|---|
| Market Data Simulator (GBM) | ✅ Complete |
| Massive API Client (Polygon.io) | ✅ Complete |
| In-memory Price Cache | ✅ Complete |
| SSE Stream Factory | ✅ Complete |
| Market Data Unit Tests | ✅ Complete |
| FastAPI App Entry Point | ❌ Missing |
| Database Schema & Init | ❌ Missing |
| Portfolio & Trade Logic | ❌ Missing |
| Watchlist API | ❌ Missing |
| LLM Chat Integration | ❌ Missing |
| Frontend (Next.js) | ❌ Missing |
| Dockerfile (multi-stage) | ❌ Missing |
| Start/Stop Scripts | ❌ Missing |
| E2E Tests (Playwright) | ❌ Missing |

---

## Build Plan: Steps to a Production-Grade Platform

### Phase 1 — Backend Foundation

#### Step 1: FastAPI Application Entry Point

Create `backend/app/main.py` as the FastAPI application root:

- Initialize the FastAPI app with metadata (title, version, description)
- Mount all API routers (market, portfolio, watchlist, chat, health)
- Register startup/shutdown lifecycle hooks:
  - On startup: initialize the SQLite database (create tables, seed default data), start the market data background task
  - On shutdown: gracefully stop the market data background task
- Configure CORS (allow all origins in development; restrict in production)
- Serve the Next.js static export at the root path (`/*`) via `StaticFiles`, with a fallback to `index.html` for SPA routing
- Register global exception handlers for consistent JSON error responses

Production considerations:
- Use `lifespan` context manager (preferred over deprecated `on_event`)
- Ensure startup failures (e.g., DB init errors) crash the process cleanly so Docker restarts it
- Add request logging middleware

---

#### Step 2: Database Layer

Create `backend/app/db/` with the following structure:

**`schema.py`** — SQLite table definitions using raw SQL (no ORM, keeps the dependency surface minimal):

```sql
-- users_profile
CREATE TABLE IF NOT EXISTS users_profile (
    id TEXT PRIMARY KEY DEFAULT 'default',
    cash_balance REAL NOT NULL DEFAULT 10000.0,
    created_at TEXT NOT NULL
);

-- watchlist
CREATE TABLE IF NOT EXISTS watchlist (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    added_at TEXT NOT NULL,
    UNIQUE(user_id, ticker)
);

-- positions
CREATE TABLE IF NOT EXISTS positions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    quantity REAL NOT NULL,
    avg_cost REAL NOT NULL,
    updated_at TEXT NOT NULL,
    UNIQUE(user_id, ticker)
);

-- trades
CREATE TABLE IF NOT EXISTS trades (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    ticker TEXT NOT NULL,
    side TEXT NOT NULL CHECK(side IN ('buy', 'sell')),
    quantity REAL NOT NULL,
    price REAL NOT NULL,
    executed_at TEXT NOT NULL
);

-- portfolio_snapshots
CREATE TABLE IF NOT EXISTS portfolio_snapshots (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    total_value REAL NOT NULL,
    recorded_at TEXT NOT NULL
);

-- chat_messages
CREATE TABLE IF NOT EXISTS chat_messages (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL DEFAULT 'default',
    role TEXT NOT NULL CHECK(role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    actions TEXT,
    created_at TEXT NOT NULL
);
```

**`init.py`** — Database initialization logic:
- Check if the SQLite file exists and tables are populated
- Run schema creation SQL if needed
- Seed the default user profile (`id='default'`, `cash_balance=10000.0`)
- Seed the default 10-ticker watchlist (AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX)
- Use `check_same_thread=False` and a connection pool pattern for thread safety

**`connection.py`** — Thin connection manager:
- Expose a `get_db()` dependency for FastAPI route injection
- Use `sqlite3.Row` as the row factory for dict-like access
- Ensure WAL mode is enabled (`PRAGMA journal_mode=WAL`) for better concurrent read performance

Production considerations:
- Enable WAL mode and `PRAGMA synchronous=NORMAL` for a good durability/performance balance
- Index `(user_id, ticker)` columns on trades and positions for query performance
- The schema must be idempotent (`IF NOT EXISTS`) — safe to run on every startup

---

#### Step 3: Portfolio API

Create `backend/app/routes/portfolio.py`:

**`GET /api/portfolio`** — Returns:
```json
{
  "cash_balance": 8500.00,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 5,
      "avg_cost": 190.00,
      "current_price": 195.50,
      "unrealized_pnl": 27.50,
      "pnl_pct": 2.89,
      "market_value": 977.50
    }
  ],
  "total_value": 9477.50,
  "total_pnl": 477.50,
  "total_pnl_pct": 4.78
}
```
- Reads positions from DB, enriches with current prices from the `PriceCache`
- Calculates unrealized P&L: `(current_price - avg_cost) * quantity`

**`POST /api/portfolio/trade`** — Request body: `{ticker, quantity, side}`
- Validate ticker is in the price cache (i.e., a known ticker)
- Validate quantity > 0
- For `buy`: check `cash_balance >= quantity * current_price`, deduct cash, upsert position (update `avg_cost` using weighted average), append to trades log
- For `sell`: check position exists and `quantity <= held_quantity`, add proceeds to cash, reduce/delete position, append to trades log
- After execution: immediately record a `portfolio_snapshot`
- Return the updated portfolio state

Trade execution formula for weighted average cost:
```
new_avg_cost = (existing_qty * existing_avg_cost + new_qty * price) / (existing_qty + new_qty)
```

**`GET /api/portfolio/history`** — Returns portfolio snapshots for the P&L chart:
```json
{
  "snapshots": [
    {"recorded_at": "2024-01-15T10:30:00Z", "total_value": 10250.00},
    ...
  ]
}
```

Production considerations:
- All trade operations must be wrapped in a single SQLite transaction (atomicity)
- Edge cases: selling exactly all shares (delete position row), buying a ticker not yet in watchlist (still valid)
- Portfolio snapshot background task: record every 30 seconds regardless of activity

---

#### Step 4: Watchlist API

Create `backend/app/routes/watchlist.py`:

**`GET /api/watchlist`** — Returns all tickers with live prices:
```json
{
  "tickers": [
    {
      "ticker": "AAPL",
      "price": 191.50,
      "prev_price": 191.20,
      "change_pct": 0.16,
      "direction": "up"
    }
  ]
}
```

**`POST /api/watchlist`** — `{ticker}`: validates ticker format (uppercase letters, 1-5 chars), checks it isn't already in watchlist, inserts, triggers market data system to start tracking it

**`DELETE /api/watchlist/{ticker}`** — Removes ticker; market data continues tracking it if it has an open position (to keep portfolio P&L live)

Production considerations:
- Ticker validation: `re.match(r'^[A-Z]{1,5}$', ticker)` at minimum
- When a new ticker is added, the price cache may not have data for it immediately — return `null` for price fields until the next market data poll

---

#### Step 5: SSE Stream Integration

Wire the existing `stream.py` factory into the FastAPI app:

- Register `GET /api/stream/prices` using the stream router from `backend/app/market/stream.py`
- Ensure the market data background task (simulator or Massive client) is started in the app lifespan and properly injects into the price cache
- The stream should push all tickers in the cache at ~500ms intervals
- Add proper SSE headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no` (prevents Nginx buffering)

---

#### Step 6: Health Check

**`GET /api/health`**:
```json
{
  "status": "ok",
  "market_data": "simulator",
  "db": "connected",
  "uptime_seconds": 3600
}
```

Used by Docker health checks and monitoring.

---

### Phase 2 — LLM Chat Integration

#### Step 7: LLM Chat Endpoint

Create `backend/app/routes/chat.py` and `backend/app/llm/`:

**System prompt** — "FinAlly, an AI trading assistant" persona:
- Concise, data-driven, finance-focused
- Always responds with valid structured JSON
- Can analyze portfolio composition, risk concentration, P&L
- Can execute trades and manage the watchlist when requested

**Context injection** — Before each LLM call, build a context block:
```
Portfolio: $8,500 cash | 5 AAPL @ $190 avg ($977.50 market value, +$27.50 P&L)
Watchlist: AAPL $191.50 (+0.16%), GOOGL $174.20 (-0.08%), ...
Total portfolio value: $9,477.50 (+4.78% unrealized)
```

**Structured output schema**:
```json
{
  "message": "string (required)",
  "trades": [{"ticker": "string", "side": "buy|sell", "quantity": "number"}],
  "watchlist_changes": [{"ticker": "string", "action": "add|remove"}]
}
```

**LLM call** — Use LiteLLM with OpenRouter targeting `openrouter/openai/gpt-oss-120b` with Cerebras inference provider. Use structured outputs (JSON mode) to guarantee valid schema.

**Auto-execution** — After parsing the LLM response:
1. Execute each trade in `trades[]` via the same trade logic as `POST /api/portfolio/trade`
2. Apply each `watchlist_changes[]` action
3. Collect results (successes and any validation errors)
4. Store the full exchange in `chat_messages` (user message + assistant response with actions JSON)
5. Return complete response to frontend

**Mock mode** — When `LLM_MOCK=true`, return a deterministic canned response:
```json
{
  "message": "Your portfolio looks well-diversified. AAPL is your largest position at 10% of portfolio value.",
  "trades": [],
  "watchlist_changes": []
}
```

Production considerations:
- Limit conversation history loaded for context (e.g., last 20 messages) to avoid token explosion
- Set a reasonable timeout on the LiteLLM call (e.g., 30 seconds)
- If structured output parsing fails, return an error message to the user rather than crashing
- Rate limit the chat endpoint (e.g., 10 req/min) to prevent runaway API costs

---

#### Step 8: Portfolio Snapshot Background Task

Create `backend/app/tasks/snapshot.py`:
- Runs every 30 seconds via `asyncio` background task
- Calculates current total portfolio value (cash + all positions at current prices)
- Inserts into `portfolio_snapshots`
- Also called synchronously after every trade execution

---

### Phase 3 — Frontend

#### Step 9: Next.js Project Setup

Initialize `frontend/` as a Next.js 14+ TypeScript project with:
- `output: 'export'` in `next.config.ts` for static export
- Tailwind CSS with a custom dark theme (`#0d1117` background, `#ecad0a` accent yellow, `#209dd7` blue, `#753991` purple)
- `lightweight-charts` (TradingView) for performant canvas-based price charts
- A single-page layout with no routing needed

Key dependencies:
```json
{
  "next": "^14",
  "react": "^18",
  "typescript": "^5",
  "tailwindcss": "^3",
  "lightweight-charts": "^4",
  "recharts": "^2"
}
```

---

#### Step 10: SSE Price Stream Hook

Create `frontend/src/hooks/usePriceStream.ts`:
- Opens `EventSource` to `/api/stream/prices`
- Maintains a map of `ticker → PriceUpdate` in React state
- Tracks previous prices to determine flash direction (green = uptick, red = downtick)
- Accumulates sparkline history per ticker (array of recent prices, capped at e.g. 100 points)
- Handles reconnection automatically (EventSource built-in) + exposes connection status (connected/reconnecting/disconnected)
- Cleans up the EventSource on unmount

---

#### Step 11: Core UI Components

**`Header`** — Portfolio total value (live, updates with each price tick), cash balance, connection status dot (green/yellow/red)

**`WatchlistPanel`** — Grid of tickers:
- Ticker symbol, current price with flash animation (green/red CSS transition ~500ms), daily change %
- Sparkline mini-chart per ticker (canvas, drawn from accumulated SSE history)
- Click a ticker to select it as the main chart focus
- Add ticker input + remove button

**`MainChart`** — Full price history chart for the selected ticker using `lightweight-charts`:
- Candlestick or area chart, dark theme
- Updates in real time as SSE prices arrive
- Shows ticker name and current price in the header

**`PortfolioHeatmap`** — Treemap visualization:
- Each rectangle = one position, sized proportionally to market value
- Color: green gradient for profit, red gradient for loss (neutral gray at breakeven)
- Shows ticker and P&L % on hover
- Can be implemented with `recharts` Treemap component

**`PnLChart`** — Line chart of `portfolio_snapshots.total_value` over time:
- Fetches from `GET /api/portfolio/history` on load, refreshes periodically
- Shows the current total value as the rightmost point

**`PositionsTable`** — Sortable table: ticker, quantity, avg cost, current price, unrealized P&L ($), P&L (%), market value. Color-coded P&L cells.

**`TradeBar`** — Compact input row: ticker text field, quantity field, Buy button (blue `#209dd7`), Sell button (red). Submits to `POST /api/portfolio/trade`. Shows inline error/success feedback.

**`ChatPanel`** — Docked sidebar (collapsible):
- Scrollable message history with role-based styling (user vs assistant)
- Inline action chips showing executed trades and watchlist changes
- Loading spinner while awaiting LLM response
- Text input + Submit button (purple `#753991`)

---

#### Step 12: Price Flash Animation

In Tailwind config, add custom keyframes:
```css
@keyframes flash-green {
  0% { background-color: rgba(34, 197, 94, 0.4); }
  100% { background-color: transparent; }
}
@keyframes flash-red {
  0% { background-color: rgba(239, 68, 68, 0.4); }
  100% { background-color: transparent; }
}
```

When a new price arrives in the SSE stream:
1. Add a `flash-green` or `flash-red` CSS class to the price cell
2. Remove it after 500ms (via `setTimeout` + state update)
3. This creates the Bloomberg-style price tick flash

---

### Phase 4 — Docker & Deployment

#### Step 13: Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build Next.js static export
FROM node:20-slim AS frontend-builder
WORKDIR /app/frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# Stage 2: Python backend
FROM python:3.12-slim AS backend
WORKDIR /app

# Install uv
RUN pip install uv

# Install Python dependencies
COPY backend/pyproject.toml backend/uv.lock ./
RUN uv sync --frozen --no-dev

# Copy backend source
COPY backend/ ./backend/

# Copy frontend static export
COPY --from=frontend-builder /app/frontend/out ./static/

# Create db directory
RUN mkdir -p /app/db

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "backend.app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Production considerations:
- Use `--frozen` to ensure exact lockfile reproducibility
- The `static/` directory is served by FastAPI's `StaticFiles` mount
- `HEALTHCHECK` instruction in Dockerfile calling `GET /api/health`

---

#### Step 14: Docker Compose

`docker-compose.yml` for development convenience:
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - finally-data:/app/db
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  finally-data:
```

---

#### Step 15: Start/Stop Scripts

**`scripts/start_mac.sh`**:
```bash
#!/bin/bash
set -e
IMAGE_NAME="finally"
CONTAINER_NAME="finally-app"

# Stop existing container if running
docker stop $CONTAINER_NAME 2>/dev/null || true
docker rm $CONTAINER_NAME 2>/dev/null || true

# Build image (or use --no-build to skip)
if [[ "$1" != "--no-build" ]]; then
  echo "Building image..."
  docker build -t $IMAGE_NAME .
fi

# Run container
docker run -d \
  --name $CONTAINER_NAME \
  -p 8000:8000 \
  -v finally-data:/app/db \
  --env-file .env \
  $IMAGE_NAME

echo "FinAlly is running at http://localhost:8000"
# Open browser on macOS
open http://localhost:8000 2>/dev/null || true
```

**`scripts/stop_mac.sh`**:
```bash
#!/bin/bash
docker stop finally-app && docker rm finally-app
echo "FinAlly stopped. Data volume preserved."
```

PowerShell equivalents for Windows follow the same logic using `docker` CLI.

---

### Phase 5 — Testing

#### Step 16: Backend Unit Tests (pytest)

Extend `backend/tests/` with:

**`tests/test_portfolio.py`**:
- Buy with sufficient cash → position created, cash deducted, avg cost correct
- Buy more of existing position → avg cost recalculated (weighted average)
- Buy with insufficient cash → `400` error, no state change
- Sell exactly all shares → position row deleted, cash added
- Sell more than held → `400` error
- Sell at a loss → cash increases, position updated correctly

**`tests/test_routes.py`**:
- All API endpoints return correct HTTP status codes
- Response shapes match documented schemas
- Watchlist add/remove operations
- Trade endpoint validation (invalid ticker, negative quantity, wrong side)

**`tests/test_llm.py`**:
- Structured output parsing handles all valid schemas
- Graceful handling of malformed/partial JSON responses
- Mock mode returns deterministic response
- Trade validation within chat flow (LLM-requested trade with insufficient funds)

---

#### Step 17: Frontend Unit Tests

In `frontend/src/__tests__/`:
- `WatchlistPanel.test.tsx` — renders tickers, price flash animation triggers on price change, add/remove works
- `PositionsTable.test.tsx` — correct P&L calculations displayed, zero positions state
- `ChatPanel.test.tsx` — message rendering, loading state, action chips visible on trade execution
- `usePriceStream.test.ts` — hook accumulates sparkline data, handles reconnection state

---

#### Step 18: E2E Tests (Playwright)

Create `test/` directory with:

**`docker-compose.test.yml`**:
```yaml
version: '3.8'
services:
  app:
    build:
      context: ..
      dockerfile: Dockerfile
    environment:
      - LLM_MOCK=true
    ports:
      - "8000:8000"
    volumes:
      - test-data:/app/db
  playwright:
    image: mcr.microsoft.com/playwright:v1.40.0-jammy
    depends_on:
      - app
    volumes:
      - ./:/tests
    working_dir: /tests
    command: npx playwright test

volumes:
  test-data:
```

**Key test scenarios** (`test/e2e/`):

1. **Fresh start**: Page loads, 10 default tickers visible, $10,000 cash shown, prices updating
2. **Watchlist add**: Type a ticker, click Add, it appears with a price
3. **Watchlist remove**: Click remove on a ticker, it disappears
4. **Buy shares**: Enter ticker + quantity, click Buy, cash decreases, position row appears
5. **Sell shares**: Sell existing position, cash increases, position updates
6. **Portfolio heatmap**: After buying, heatmap shows colored rectangle for position
7. **P&L chart**: After trades, chart has data points
8. **AI chat (mocked)**: Send a message, see a response, no spinner remains
9. **SSE resilience**: Disconnect network tab and verify reconnection indicator changes + recovers
10. **Insufficient funds**: Attempt to buy more than cash allows, see error message

---

### Phase 6 — Production Hardening

#### Step 19: Security & Reliability

- **Input validation**: Pydantic models for all request bodies; reject unknown fields
- **SQL injection prevention**: Use parameterized queries throughout (never string-interpolate SQL)
- **Error boundaries**: Frontend React error boundaries prevent full-page crashes on component failures
- **Rate limiting**: Add `slowapi` to FastAPI for rate limiting on `/api/chat` and `/api/portfolio/trade`
- **Secrets management**: Never log `OPENROUTER_API_KEY`; validate its presence at startup

#### Step 20: Observability & UX Polish

- **Structured logging**: Use Python's `logging` with JSON formatter for production; include request IDs
- **Connection quality indicator**: Show latency (time since last SSE event) in the header dot color
- **Trade feedback**: After buy/sell, briefly highlight the affected row in the positions table
- **Error toast notifications**: Non-blocking error messages for failed trades or connection issues
- **Empty states**: Meaningful messages when positions table is empty, when portfolio heatmap has no holdings
- **Keyboard shortcuts**: `B` to focus trade bar buy, `S` for sell, `/` to focus chat input

---

## Summary: Build Order & Dependencies

```
Phase 1 (Backend Foundation)
  Step 1: FastAPI entry point
  Step 2: SQLite database layer        ← depends on Step 1
  Step 3: Portfolio API                ← depends on Steps 1, 2
  Step 4: Watchlist API                ← depends on Steps 1, 2
  Step 5: SSE stream wiring            ← depends on Step 1 (market data already done)
  Step 6: Health check                 ← depends on Step 1

Phase 2 (LLM)
  Step 7: LLM chat endpoint            ← depends on Phase 1
  Step 8: Snapshot background task     ← depends on Steps 2, 3

Phase 3 (Frontend)
  Step 9:  Next.js setup               ← independent
  Step 10: SSE hook                    ← depends on Step 9
  Steps 11–12: UI components           ← depends on Steps 9, 10

Phase 4 (Docker & Deployment)
  Step 13: Dockerfile                  ← depends on Phases 1, 3
  Step 14: Docker Compose              ← depends on Step 13
  Step 15: Start/stop scripts          ← depends on Step 14

Phase 5 (Testing)
  Step 16: Backend unit tests          ← depends on Phase 1
  Step 17: Frontend unit tests         ← depends on Phase 3
  Step 18: E2E tests                   ← depends on Phase 4

Phase 6 (Hardening)
  Steps 19–20: Security & polish       ← depends on all phases
```

---

*This review plan was generated by analyzing the current repository state against the PLAN.md specification. The market data subsystem is production-ready; the remaining 80% of the platform follows naturally from this foundation.*
