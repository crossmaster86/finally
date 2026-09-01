# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Notes — Questions, Clarifications & Simplifications

*Added by a documentation review pass. Items are tagged **[Blocker]** (an agent cannot
proceed without a decision), **[Clarify]** (ambiguous — two agents would build it
differently), or **[Simplify]** (works as written, but there is a cheaper path). Nothing
here changes the spec; these are questions for the author.*

### 13.1 Mismatches with what is already built

The market data component is complete (`planning/MARKET_DATA_SUMMARY.md`). A few places
where §6/§10 no longer describe the shipped code:

1. **[Clarify] SSE event shape.** §6 says "Each SSE event contains ticker, price, previous
   price, timestamp, and change direction" — i.e. one event per ticker. The implementation
   batches: one event per tick containing *all* tickers, keyed by symbol:
   `data: {"AAPL": {ticker, price, previous_price, timestamp, change, change_percent, direction}, ...}`.
   The batched form is better (one parse per tick, no interleaving). Should §6 be updated to
   document the batched payload as the frontend contract, including the leading `retry: 1000`
   directive?

2. **[Blocker] "Daily change %" has no data behind it.** §10 asks the watchlist to show
   *daily* change %, but `PriceUpdate.change_percent` is **tick-to-tick** (vs. ~500ms ago),
   which is a near-zero number and useless as a column. The simulator has no concept of a
   previous close; Massive's snapshot does (`todaysChangePerc`), so the two sources would
   disagree. Options:
   - **(a)** Redefine the column as "change since session open", where session open = the price
     captured when the backend started. Cheap: one extra dict in `PriceCache`, identical
     behaviour for both sources. *Recommended.*
   - **(b)** Have the source supply a `previous_close` field — real for Massive, synthesized for
     the simulator.

   Either way `PriceUpdate` needs a new field, and the plan should say which number the UI renders.

3. **[Clarify] Timestamp units are inconsistent across the API.** SSE sends `timestamp` as a Unix
   float (seconds); every DB column (`executed_at`, `recorded_at`, `created_at`) is an ISO-8601
   string. The frontend will hold both. Can the plan state the convention explicitly — e.g. *"SSE
   uses epoch seconds; all REST/DB timestamps are ISO-8601 UTC with a trailing Z"* — so the P&L
   chart and the price chart don't need two time parsers?

4. **[Clarify] §9 refers to a "cerebras-inference skill"; the skill available in this repo is named
   `cerebras`.** Worth correcting so the LLM agent invokes the right thing.

### 13.2 The watchlist ↔ market data ↔ portfolio seam

This is the largest unspecified area in the document.

5. **[Blocker] Removing a watchlist ticker can break portfolio valuation.**
   `MarketDataSource.remove_ticker()` also *removes the ticker from the PriceCache*. §8 lets the
   user `DELETE /api/watchlist/{ticker}` at any time. If they hold a position in that ticker, its
   live price vanishes and the position can no longer be valued or sold at market. Which rule applies?
   - **(a)** Reject the delete with 409 while a position is open — simple, but the AI may
     legitimately want to tidy the watchlist.
   - **(b)** The tracked set is `watchlist ∪ tickers-with-open-positions`, and `remove_ticker` is
     only called when the ticker is in neither. *Recommended* — matches §6's "all tickers known to
     the system" wording, ~5 lines of code.

   Either way, §7 should note that the `watchlist` table is **not** the authoritative tracked-ticker
   set; the union is.

6. **[Blocker] Who calls `add_ticker` / `remove_ticker`?** §8 documents the watchlist routes but
   never says they must drive the market data source. Please state that `POST /api/watchlist` →
   persist → `await source.add_ticker(t)`, and the delete path does the inverse (subject to #5).
   Otherwise a newly added ticker never streams.

7. **[Blocker] Trading a ticker that has no price.** The trade bar (§10) is a free-text ticker field,
   and the LLM can name any symbol. If the cache has no price, `POST /api/portfolio/trade` cannot fill
   at "current price". Reject with 400 ("no market data for XYZ"), or auto-add to the watchlist and
   wait for the first tick (adds latency and a race)? Recommend rejecting, with the system prompt and
   UI steering users to add the ticker first.

8. **[Clarify] Ticker validation and normalization.** Nothing defines what a valid ticker is — the
   simulator will happily invent a price for `ZZZZ`, `AAPL ` or `aapl`. Please specify: uppercase and
   trim on input, a format rule (`^[A-Z]{1,5}$`?), the response to a duplicate add (409 vs. idempotent
   200), and a maximum watchlist size (the Massive free tier and the SSE payload size both care).
   Also: what does the UI show if the user deletes *every* ticker?

### 13.3 Portfolio & trade semantics

9. **[Blocker] Sell-to-zero behaviour.** §12 hedges — "position updates or disappears". Pick one:
   delete the row at qty 0 (recommended — keeps the positions table and heatmap clean, and `trades`
   is already the permanent audit log), or keep a zero-quantity row.

10. **[Clarify] `avg_cost` on a sell.** State the convention: a sell reduces `quantity` and leaves
    `avg_cost` untouched (realized gain flows into `cash_balance` implicitly). Without this written
    down, two agents will implement two different P&L numbers.

11. **[Clarify] Realized vs unrealized P&L.** §8's `/api/portfolio` returns "unrealized P&L", and
    there is no realized-P&L field or column anywhere. Is that deliberate — realized gains being
    visible only through cash and the total-value line? One sentence would settle it, because the
    positions table and header otherwise look like they are missing a number.

12. **[Clarify] Quantity and money precision.** The schema says `REAL`/fractional but nothing bounds
    it. Please specify: minimum quantity (reject `0` and negatives — with which status code?), rounding
    of quantity and `cash_balance` (2dp, or full float and round only for display?), and whether a buy
    costing $10,000.0000001 fails against a $10,000 balance. Float drift will otherwise produce an
    "insufficient cash" bug on the very first "buy max" attempt.

13. **[Clarify] Pricing a position with a missing or stale price.** If the cache has no entry for a
    held ticker (see #5, or the first seconds after a restart with Massive's 15s poll), does valuation
    fall back to `avg_cost`, to the last snapshot, or return `null` so the UI shows "—"?

### 13.4 Portfolio snapshots & history

14. **[Simplify] `portfolio_snapshots` grows without bound.** Every 30s forever is ~2,900 rows/day and
    ~1M/year, and `/api/portfolio/history` as specified returns all of them into a chart. Cheapest
    fixes, in preference order: add `?since=` / `?limit=` query params (needed regardless — they are
    currently undocumented), and/or prune rows older than N days at startup. Also: should the task keep
    writing identical rows while the app idles with no positions? Skipping a write when the value is
    unchanged costs one comparison and flattens the growth curve entirely.

15. **[Clarify] Gaps in the P&L series.** The volume persists across restarts, so the chart will contain
    multi-hour or multi-day gaps between sessions. Should the line connect across the gap, break, or
    should `/api/portfolio/history` default to "this session only"?

### 13.5 Chat & LLM

16. **[Blocker] There is no way to load chat history.** §7 persists `chat_messages` and §9 reads it back
    for LLM context — but §8 has no `GET /api/chat/history`. On page reload the conversation disappears
    from the UI while the LLM silently still remembers it, which is worse than either extreme. Add the
    endpoint (recommended), or drop the table and keep history in frontend memory only.

17. **[Blocker] `POST /api/chat` request/response shape is undefined.** §9 documents the *LLM's*
    structured output, not the API contract. Suggest specifying explicitly, e.g. request
    `{"message": "..."}`, response `{"message", "trades_executed": [...], "watchlist_changes": [...],
    "errors": [...]}`. Related: after auto-executed trades, does the frontend re-fetch `/api/portfolio`,
    or does the chat response carry the updated portfolio so the header and positions update in one
    round trip?

18. **[Clarify] Conversation history is unbounded.** §9 says "recent conversation history" — how many
    messages? Suggest a hard cap (last 20 messages, or a token budget) so a long session cannot blow the
    context window or the latency budget.

19. **[Clarify] Malformed-LLM-response behaviour.** §12 asks for tests of "graceful handling of malformed
    responses" but §9 never defines what graceful means. Suggest: one retry with a "valid JSON only"
    nudge, then a canned apology message with zero actions — and never a 500. Also worth confirming that
    `openrouter/openai/gpt-oss-120b` on Cerebras honours `response_format: json_schema`; if it only
    honours `json_object`, then prompt-plus-parse *is* the normal path and the plan should say so.

20. **[Clarify] "Buy $2,000 of NVDA" has no representation.** The trade schema carries only `quantity` in
    shares. Either add an optional `notional` field, or state in §9 that the system prompt must instruct
    the LLM to convert dollars to shares using the live prices already in its context (cheaper —
    *recommended*, but it needs writing down or it won't happen).

21. **[Blocker] `LLM_MOCK=true` behaviour is undefined, and the E2E suite depends on it.** §12 requires an
    E2E test where "trade execution appears inline" under mock mode. That only works if the mock returns a
    deterministic, *trade-executing* response. Please specify the mapping — e.g. a user message containing
    "buy" returns a fixed 1-share AAPL buy, anything else returns a fixed analysis string. Without this the
    E2E scenario cannot be written.

22. **[Clarify] Missing `OPENROUTER_API_KEY` with `LLM_MOCK=false`.** §5 calls the key "required", but
    students will run without one. Recommend: the app boots normally and only `/api/chat` returns a
    friendly 503 ("chat unavailable — set OPENROUTER_API_KEY"). Worth stating, so the whole app doesn't
    fail to start.

### 13.6 Runtime, Docker & database

23. **[Blocker] §4 and §11 disagree about the volume.** §4 says the repo's top-level `db/` directory "is
    the runtime volume mount point" and the SQLite file lives there; §11's run command uses a **named**
    volume (`-v finally-data:/app/db`), which means the file is *not* visible in the repo directory. Pick
    one. A bind mount (`-v "$PWD/db:/app/db"`) is friendlier for a teaching project — students can inspect
    and delete the DB — at the cost of file-permission quirks on Linux.

24. **[Clarify] No configurable DB path.** Hardcoding `/app/db/finally.db` means the backend cannot run
    outside Docker for development. Suggest a `DATABASE_PATH` env var defaulting to `db/finally.db`
    relative to the project root.

25. **[Clarify] SQLite concurrency.** Three writers/readers coexist — request handlers, the 30s snapshot
    task, and the chat auto-execution path — alongside long-lived SSE connections on the event loop.
    Without `journal_mode=WAL` and `check_same_thread=False` (or a single serialized connection),
    "database is locked" is close to guaranteed under the E2E suite. One line in §7 would save a day of
    debugging.

26. **[Simplify] "Startup (or first request)" — just pick startup.** §7's lazy-on-first-request path adds a
    check to every request and creates an ordering puzzle, because the market data source must start with
    the ticker set read *from* the database. A single FastAPI `lifespan` handler resolves it in a defined
    order: init+seed DB → read watchlist ∪ position tickers → `create_market_data_source` →
    `source.start(tickers)` → start the snapshot task; reverse on shutdown. Worth spelling out as an
    explicit sequence in §7 or §11.

27. **[Clarify] Static export + API routing order.** FastAPI must mount `/api/*` routes *before* the
    catch-all `StaticFiles(html=True)`, and needs an SPA fallback for unknown paths. Also worth noting the
    Next.js export constraints this implies (no route handlers, no image optimization, and `trailingSlash`
    must match how FastAPI serves directories).

28. **[Clarify] No documented development loop.** Docker is the only described run path, but nobody will
    iterate on the frontend through a multi-stage image rebuild. Suggest documenting the dev setup:
    `uvicorn --reload` on 8000, `next dev` on 3000, and a `rewrites()` entry in `next.config` proxying
    `/api/*` to 8000 — which also preserves the "no CORS" property.

29. **[Clarify] The Massive polling interval isn't configurable via env.** §6 says "configurable interval"
    but §5 lists no variable; the code defaults to 15s. Suggest `MASSIVE_POLL_SECONDS` (default 15).

30. **[Clarify] Real market data looks broken outside market hours.** With `MASSIVE_API_KEY` set on a
    weekend, prices never change: no flashes, flat sparklines, static chart. The plan should warn about
    this and keep the simulator as the demo default (§6 implies this — worth making it explicit).

### 13.7 Frontend

31. **[Blocker] The main chart has no data source.** §10's detail chart shows "price over time", but the
    only price history anywhere is what the frontend has accumulated since page load. So the main chart is
    empty on load and covers exactly the same span as the sparkline beside it. Options:
    - **(a)** State plainly that the main chart is also "since page load". Zero cost, underwhelming.
    - **(b)** Add a bounded ring buffer per ticker in `PriceCache` (e.g. 1,800 points ≈ 15 min at 500ms)
      plus `GET /api/prices/{ticker}/history`. ~30 lines, survives page reloads, and fixes sparklines on
      refresh too. *Recommended.*
    - **(c)** Fetch real history from Massive — only works when a key is set, so it can't be the primary path.

32. **[Simplify] Pick one charting library.** §10 offers "Lightweight Charts or Recharts". They solve
    different problems, and the app needs line charts, sparklines *and* a treemap. Lightweight Charts has no
    treemap, so choosing it means adding a second library anyway. Recommend **Recharts only** (`LineChart`,
    `Treemap`, and a trivial sparkline) — one dependency, one mental model.

33. **[Clarify] Frontend behaviour when tickers appear or disappear mid-stream.** Because the SSE payload is
    a full snapshot keyed by ticker, an added ticker only shows up once it has a price, and a removed one
    silently vanishes from the payload. Should the watchlist render its *rows* from `GET /api/watchlist`
    (source of truth) and use SSE purely for price *values*? That's the sane split and worth stating.

34. **[Clarify] Display formatting isn't specified.** Prices to 2dp, quantities to how many? Currency symbol
    and thousands separators? P&L sign convention (`+$12.34` / `-$12.34`)? Small, but it is the difference
    between "polished terminal" and "obviously assembled by three different agents".

### 13.8 Testing

35. **[Clarify] "SSE resilience: disconnect and verify reconnection" is hard to drive from Playwright.**
    There is no documented way to force a server-side drop. Suggest `context.setOffline(true/false)` or
    route interception, noted in §12 — otherwise this scenario quietly gets skipped.

36. **[Clarify] Missing test dependencies.** `backend/pyproject.toml`'s dev extra has pytest, coverage and
    ruff but no `httpx` — required for FastAPI's `TestClient`/`ASGITransport`, and therefore for every API
    route test §12 asks for. Also not yet present: `litellm`, and `python-dotenv` if the backend is expected
    to read `.env` itself (§5 says it does — but inside Docker the env arrives via `--env-file`, so dotenv is
    only needed for local dev; worth clarifying which).

37. **[Simplify] Set a quality bar, not just a topic list.** §12 lists what to test but sets no threshold. If
    agents are grading their own work, a concrete gate ("backend ≥80% line coverage, `ruff check` clean") is
    more actionable than a list of subjects — and the market data component already hit 84%, so it's proven
    realistic.

### 13.9 Simplification opportunities (summary)

| # | Opportunity | Saving |
|---|---|---|
| S1 | **Three ways to run the app** (4 shell scripts + optional `docker-compose.yml` + `docker-compose.test.yml`) collapse to one. `docker compose up` behaves identically on macOS, Linux and Windows, so the platform-specific scripts become 3-line wrappers — or disappear. | ~4 files, plus the doc drift of keeping 4 scripts in sync |
| S2 | **One charting library** (Recharts) instead of a choice between two (#32). | One dependency, one API |
| S3 | **Lifespan startup instead of lazy init** (#26). | Removes a per-request check and an ordering hazard |
| S4 | **Drop the UUID `id` on `watchlist` and `positions`.** Both already carry `UNIQUE(user_id, ticker)`, which is the natural primary key. `trades` and `chat_messages` genuinely need surrogate IDs; these two don't. | A little code, one less thing to generate correctly |
| S5 | **Snapshot only on change** (#14) rather than unconditionally every 30s. | Bounded table growth, no retention job needed |
| S6 | **Define `user_id = "default"` as one module-level constant**, not a literal repeated across 6 tables and every query. | Makes the future multi-user story a one-file change |
| S7 | **Skip realized-P&L tracking entirely** and say so (#11) — cash already encodes it. | Avoids a column, a calculation, and a class of rounding bugs |

### 13.10 Suggested build order

§4 defines the boundaries but not the sequence, and several components block others. A
dependency-respecting order that keeps a working app at every step:

1. **DB layer** — schema, seed, startup init, `DATABASE_PATH`, WAL. *(Unblocks everything.)*
2. **Portfolio + watchlist API** — `/api/portfolio`, `/api/portfolio/trade`, `/api/watchlist` CRUD, wired to
   the existing `PriceCache` and the tracked-ticker union (#5, #6). Snapshot task.
3. **App shell + Docker** — lifespan wiring, static mount, Dockerfile, scripts. *Now the app runs.*
4. **Frontend core** — header, watchlist with SSE and flashes, positions table, trade bar. *Now it's usable.*
5. **Visualizations** — main chart, heatmap, P&L chart. *(Depends on #31's decision.)*
6. **Chat** — LLM integration, mock mode, chat panel.
7. **E2E suite** — Playwright against the composed stack.

Steps 1–4 are a demonstrable MVP; 5–7 are additive. If time runs short, the honest cut is to trim step 5 to
the P&L chart alone rather than shipping three half-finished visualizations.

### 13.11 Smaller questions

- §2 says "no login, no signup" and §7 hardcodes `user_id="default"` — should the API still accept an
  optional `user_id` param for future-proofing, or is the constant genuinely enough? (Recommend the
  constant; see S6.)
- §4's tree shows `db/.gitkeep` as committed, but the repo has no `db/` directory yet. Should it be created
  now, or does the start script `mkdir` it?
- §11's Dockerfile sketch doesn't mention `uv sync --frozen --no-dev` — worth stating, so the production
  image doesn't ship pytest and ruff.
- Is CI in scope (GitHub Actions running pytest + Playwright), or is local-only sufficient for the course?
- §2 mentions "responsive but desktop-first ... functional on tablet". Is tablet in scope for the E2E suite,
  or aspirational?

---

## 14. Second Review Pass — Code-Grounded Findings

*§13 reviewed this document against itself. This pass reviews it against the repository as it
actually stands (commit `14550e1`, 2026-08-29), by reading the shipped market-data code and
running it. Every finding below is new; where one sharpens, answers or corrects a §13 item it
says so explicitly. Same tags: **[Blocker]** / **[Clarify]** / **[Simplify]**. Numbering continues
from §13.*

### 14.1 Repository reality check

§4, §5, §9 and the root `README.md` all describe the repo in the present tense. Most of it does
not exist yet, which matters because agents are told to treat this document as the contract:

| §4 / §5 / §9 claims | Actually present? |
|---|---|
| `frontend/`, `test/`, `scripts/`, `db/`, `Dockerfile`, `docker-compose.yml` | **No** — none exist |
| `db/.gitkeep` "exists in repo" (§4) | **No** — the `db/` directory has never been created |
| `.env.example` committed (§4); `README.md`'s quick start opens with `cp .env.example .env` | **No** — that command fails on a fresh clone |
| §9: "There is an OPENROUTER_API_KEY in the `.env` file in the project root" | **No `.env` file exists.** An agent that believes this line will write code that reads a key which isn't there, and get a confusing runtime failure rather than a clear config error |
| `backend/` uv project, market data, SSE | **Yes** — 73 tests, 91% line coverage |

61. **[Clarify] Say which tense §4 is written in.** Either mark the not-yet-built entries (a
    "planned" marker, or a status column), or add a one-line preamble: *"§4 describes the target
    layout; only `backend/` and `planning/` exist today."* Cheap, and it stops agents assuming a
    sibling directory is there to import from.

62. **[Blocker] Create `.env.example` and correct §9's claim** before any agent starts step 6 of
    §13.10. §9 should say *"the key is read from `OPENROUTER_API_KEY`; copy `.env.example` to `.env`
    and fill it in"* — a statement that stays true, rather than one about a gitignored file that
    only exists on one machine.

### 14.2 `.gitignore` will silently swallow the frontend — [Blocker]

The committed `.gitignore` is an unmodified 207-line **Python** template. Nothing in it anticipates
Next.js, and three of its rules actively misfire. Verified with `git check-ignore -v`:

```
$ git check-ignore -v frontend/lib/api.ts
.gitignore:17:lib/      frontend/lib/api.ts        # <- IGNORED

$ git check-ignore -v frontend/node_modules/x      # -> not ignored
$ git check-ignore -v frontend/out/index.html      # -> not ignored
$ git check-ignore -v db/finally.db                # -> not ignored
```

63. **`lib/` on line 17 is unanchored, so it matches at any depth.** `frontend/lib/` is one of the
    most common directories in a Next.js project (API client, formatters, hooks). The Frontend
    Engineer agent will write `frontend/lib/api.ts`, `git add -A` will silently skip it, the commit
    will look clean, and the Docker build will fail on a fresh clone with a module-not-found error
    that has no obvious cause. `build/`, `dist/` and `share/` have the same unanchored problem.

64. **`node_modules/`, `.next/` and `out/` are not ignored at all.** The first `npm install` stages
    tens of thousands of files, and `out/` is exactly the Next.js static-export target §11 depends on.

65. **§4 says `finally.db` is gitignored. It isn't.** The template carries Django's `db.sqlite3`,
    not `db/finally.db` or `*.db`. As written, the runtime database — the one file guaranteed to
    differ between every student's checkout — gets committed.

**Recommended fix, before step 1 of §13.10:** anchor the Python rules that need it (`/build/`,
`/dist/`, `/lib/`) and append a frontend + project block:

```gitignore
# Frontend
node_modules/
.next/
out/
frontend/.env*.local

# Runtime database
db/*.db
db/*.db-wal
db/*.db-shm
!db/.gitkeep
```

The `-wal`/`-shm` lines matter once §13 #25's WAL recommendation is adopted — WAL mode creates two
sidecar files next to the database.

### 14.3 Defects in the shipped market-data code that the next steps will hit

These sit in code §13 treated as done. None are visible from the document alone.

66. **[Blocker] `create_stream_router()` mutates a module-level router.** `stream.py` defines
    `router = APIRouter(prefix="/api/stream")` at module scope, and the factory registers
    `@router.get("/prices")` onto *that shared object* on every call. Verified:

    ```python
    r1 = create_stream_router(cache_a); r2 = create_stream_router(cache_b)
    r1 is r2                      # True
    [x.path for x in r2.routes]   # ['/api/stream/prices', '/api/stream/prices']
    ```

    Two registrations, and FastAPI serves the **first** — which closes over the *stale* cache. A
    single production app never notices, but every pytest fixture that builds a fresh app (exactly
    what §12's "API routes" tests require) gets a stream bound to a previous test's cache, and the
    duplicate accumulates across the session. The fix is one line — move `router = APIRouter(...)`
    inside the factory — but it should happen before the app-shell step, because after that the same
    "factory closing over a module global" shape is likely to be copied into the portfolio and
    watchlist routers.

67. **[Blocker] The SSE stream has no heartbeat, so idle connections die silently.**
    `_generate_events` only yields when `price_cache.version` changes. With the simulator that is
    every 500ms, so the problem is invisible in development — but with `MASSIVE_API_KEY` set it is
    **15 seconds of zero bytes**, and outside market hours (§13 #30) it is *forever*. Any proxy in
    front of the app — nginx, App Runner (§11's stretch goal), a corporate proxy — closes an idle
    response stream well inside that window. The client then reconnects on the `retry: 1000`
    directive, gets another silent stream, and loops. Recommend the plan mandate a comment-frame
    keepalive (`yield ": ping\n\n"`) at a fixed cadence — 15s or less — independent of whether
    prices changed. It also gives §13 #35 something deterministic to assert against.

68. **[Clarify] The SSE payload can be a torn snapshot.** `PriceCache.update()` takes the lock once
    *per ticker*, and the simulator's `_run_loop` calls it in a `for` loop, so one tick is ten
    separate critical sections. `version` (read without the lock) can therefore change while a tick
    is half-applied, and `get_all()` can return a dict mixing tick *N* and tick *N-1* prices.
    Harmless for a flashing watchlist; not harmless if the portfolio-snapshot task or trade
    validation reads the cache at that instant and values a portfolio against two different moments.
    Cheapest fix is an `update_many(dict)` that takes the lock once per tick — and the plan should
    state that **portfolio valuation reads one `get_all()` snapshot**, never `get_price()` in a loop.

69. **[Clarify] Removing and re-adding a ticker resets its price to the seed value.**
    `GBMSimulator.remove_ticker` deletes `self._prices[ticker]`, and `_add_ticker_internal` re-seeds
    from `SEED_PRICES`. So AAPL that has drifted to $205 over a long session snaps back to $190 the
    moment the user — or the AI, which §9 encourages to "manage the watchlist proactively" — removes
    and re-adds it. If §13 #5(b) is adopted, the position keeps streaming and its unrealized P&L
    jumps by -7% for no reason. Recommend the simulator retain last-known prices for removed tickers
    (a small `_last_known` dict) and prefer them over `SEED_PRICES` on re-add.

70. **[Clarify] The green/red flash fires far less often for cheap or low-volatility tickers, and
    the plan should set an expectation.** The GBM step is `sigma * sqrt(dt)` with `dt ~ 8.48e-8`, and
    prices are rounded to 2dp, so whether a tick is *visible* depends on price level. Measured from
    the shipped constants:

    | Ticker | Per-tick sigma | Ticks with no visible change |
    |---|---|---|
    | NVDA ($800, σ 0.40) | $0.093 | 4% |
    | MSFT ($420, σ 0.20) | $0.025 | 16% |
    | AAPL ($190, σ 0.22) | $0.012 | 32% |
    | JPM ($195, σ 0.18) | $0.010 | 38% |
    | *user-added ticker seeded near $50* | $0.004 | **83%** |

    The default watchlist is fine. The risk is §13 #8's unknown-ticker path:
    `_add_ticker_internal` falls back to `random.uniform(50.0, 300.0)`, so a ticker the user adds by
    hand can land at $52 and then look **frozen** — the one interaction most likely to be demoed.
    Recommend narrowing the unknown-ticker seed range (say $80–$400), and saying in §6 that the
    simulator's visible tick rate is price-dependent so nobody files it as an SSE bug.

71. **[Clarify] An invalid ticker fails in two completely different ways.** With the simulator,
    `ZZZZ` gets an invented price and trades happily. With Massive, `ZZZZ` is simply absent from the
    snapshot response, no error is raised (`_poll_once` logs at debug and moves on), and the row sits
    blank forever. §13 #8 asks for a *format* rule; this is the stronger requirement: **validation
    must happen at the API layer, before the ticker reaches either source**, so both behave
    identically. It also means "does this ticker exist?" is a question the simulator cannot answer —
    worth stating that the app validates *shape*, not *existence*.

### 14.4 The LLM call will freeze the price stream — [Blocker]

72. The `cerebras` skill in this repo documents the synchronous `litellm.completion(...)`. Called
    directly inside a FastAPI `async def` handler, that **blocks the event loop for the entire round
    trip** — and the SSE generator, the simulator's `_run_loop` and the snapshot task all live on
    that same loop. The user-visible symptom is precise and confusing: every time they send a chat
    message, all prices freeze, sparklines flat-line, and the connection dot may go yellow — for as
    long as the LLM takes. Cerebras is fast, but "fast" is still 300ms–2s of a dead UI, repeated on
    every message.

    §9 should state the requirement explicitly: **wrap the LLM call in `asyncio.to_thread()`** — the
    pattern `massive_client.py` already uses for the synchronous `RESTClient` — or use
    `litellm.acompletion`. This is exactly the kind of cross-cutting constraint that an agent
    building the chat feature in isolation will not think of, because in isolation it works fine.

### 14.5 Contracts the plan still leaves to chance

73. **[Blocker] Nothing says how routes get the `PriceCache`.** The shipped code uses a
    factory-closure (`create_stream_router(cache)`); FastAPI's own idiom is `app.state` plus a
    `Depends`. The portfolio, trade, watchlist and chat routes all need the same cache, and they are
    being built by different agents. Pick one convention and write it into §4's Key Boundaries — the
    lifespan sequence in §13 #26 is the natural place to say where the cache is constructed and how
    it is handed out. Without this, five routers will use three patterns.

74. **[Clarify] Environment variables need one owner.** `os.environ.get` currently lives in
    `factory.py`; §5 plus §13's #24 and #29 add `DATABASE_PATH`, `MASSIVE_POLL_SECONDS`, `LLM_MOCK`
    and `OPENROUTER_API_KEY`. Recommend a single `app/config.py` read once at startup — and, worth
    stating because it is a real testability trap, that values are read **when called, not at import
    time**, which is what `create_market_data_source` already does correctly.

75. **[Blocker] The frontend has no specified empty / loading / error states.** §10 lists components
    and §13 #34 covers number formatting, but nothing says what any panel renders *before the first
    SSE tick*, while a fetch is in flight, or when a call fails. Every agent will invent something
    different and the seams will show. One small table would settle it:

    | Situation | Watchlist | Positions / heatmap | Chat |
    |---|---|---|---|
    | Before first SSE tick | rows from `GET /api/watchlist`, price cell `—` | `—` for value and P&L | normal |
    | No positions held | n/a | "No positions yet" empty state, **not** an empty chart | normal |
    | Watchlist emptied (§13 #8) | "Add a ticker to get started" | unaffected | normal |
    | Fetch fails / SSE disconnected | last known values, dimmed; status dot red | last known values, dimmed | normal |
    | Chat 503 (no API key, §13 #22) | unaffected | unaffected | inline notice, input disabled |

76. **[Clarify] §2's palette is missing the two colours the app uses most.** Accent yellow, blue and
    purple are specified; **up-green, down-red and the flash background tints are not** — and those
    appear in the watchlist, the heatmap, the positions table, the P&L chart and the sparklines.
    Three agents will pick three greens. Specify the pair (plus flash tints at low alpha), and note
    the accessibility point: red/green alone fails for the ~8% of viewers with deuteranopia, so P&L
    should carry an explicit `+`/`-` sign and direction an arrow glyph, not colour alone. §13 #34
    asks for a sign convention for a formatting reason; this is the stronger reason to have one.

77. **[Clarify] `POST /api/chat` is not idempotent and auto-executes trades.** §9 removes the
    confirmation dialog deliberately, which means a double-submit — impatient Enter, a retry after a
    perceived hang (see #72), a React StrictMode double-effect in dev — places the trade twice with
    no way to tell. Recommend: the UI disables the input while a request is in flight, and §9 states
    plainly that the endpoint is not safe to retry. If that feels thin for something spending (fake)
    money, an optional client-generated `request_id` deduplicated server-side is ~10 lines.

78. **[Clarify] Capture the fill price exactly once per trade.** Prices tick every 500ms, so a
    handler that reads `get_price()` for the validation check and again for the `trades` row can
    validate against $190.00 and fill at $190.02 — enough to overdraw cash on a "buy max". State the
    rule: **read the price once; use that one value for validation, the fill, the cash update and the
    stored row.** Distinct from §13 #12, which is about float precision; this is ordering.

### 14.6 Corrections and answers to open §13 items

79. **§13 #4 is half right about the skill name.** The directory and invocation name is `cerebras`;
    the `name:` in the skill's own frontmatter is `cerebras-inference`, which is what §9 currently
    says. So §9 is not wrong so much as ambiguous. Recommend aligning the frontmatter to the
    directory name and having §9 refer to *"the `cerebras` skill"*, so the tool call is unambiguous.

80. **§13 #19's doubt about structured outputs is largely answerable.** The skill documents
    `response_format=<pydantic BaseModel subclass>` together with
    `extra_body={"provider": {"order": ["cerebras"]}}` and `reasoning_effort="low"` as the supported
    pattern. Treat that as the primary path; §13's retry-then-canned-apology fallback stays the
    safety net rather than the normal route. §9 should point at the skill for the call shape instead
    of restating it, so the two cannot drift.

81. **§13.11's CI question, answered:** `.github/workflows/` already contains `claude.yml` and
    `claude-code-review.yml`. Both are Claude review bots — **neither runs pytest, ruff or
    Playwright.** So CI exists in the sense of review automation and does not exist in the sense
    §13 #37 means. A test workflow is ~20 lines and would make the coverage gate real rather than
    advisory.

82. **§13 #37's coverage target, measured:** the backend is at **91%** overall, 73 tests passing in
    5s — comfortably above the proposed 80% gate. But the number hides the gap that matters:

    ```
    app\market\stream.py    36 stmts   24 miss   33%
    ```

    The SSE generator — the single component every frontend feature depends on, and the one holding
    #66 and #68 — is the least-tested module in the codebase. A flat percentage gate passes today and
    keeps passing while `stream.py` stays untested. Recommend either a per-file floor, or naming SSE
    coverage as an explicit deliverable of §13.10's step 3.

83. **§11's docker run command is bash-only, and the start scripts are Windows-inclusive.**
    `-v "$PWD/db:/app/db"` (§13 #23's recommended bind mount) does not expand in PowerShell — there
    it is `${PWD}`. Since §11 explicitly promises `scripts/start_windows.ps1`, the plan should show
    both forms, or adopt §13 S1's `docker compose up` and sidestep the difference entirely. (Docker
    Desktop on Windows handles the bind mount fine; §13 #23's permission caveat is Linux-only.)

### 14.7 Further simplification opportunities

| # | Opportunity | Saving |
|---|---|---|
| S8 | **Import the Massive SDK lazily.** `factory.py` imports `MassiveDataSource` unconditionally, so `massive` is a hard runtime dependency and an import cost for every user — and the default, recommended path never uses it. Move the import inside the `if api_key:` branch and make it an optional extra. | One dependency out of the default install and the Docker image |
| S9 | **Make `README.md` derived, not parallel.** It already restates §5's env table, §4's tree and §11's run command — three places that must change together when §13 #23 (volume), #24 (`DATABASE_PATH`) or #29 (`MASSIVE_POLL_SECONDS`) are resolved, and it already documents a `cp .env.example .env` that fails. Cut it to a quick start plus a link, and name PLAN.md the single source of truth. | Removes three doc-drift surfaces that are already drifting |
| S10 | **`PriceCache.update_many()` instead of a per-ticker lock** (#68). | One critical section per tick; torn snapshots stop being possible |
| S11 | **Drop `change`, `change_percent` and `direction` from the SSE payload.** The client already has `price` and `previous_price`, and §13 #2 is adding a session-open baseline anyway — all three are derivable in one line of TypeScript. They are roughly 40% of the per-ticker payload. | Smaller payload, one less place for the two sources to disagree |
| S12 | **One `Dockerfile` plus `docker compose`, no separate `docker-compose.test.yml`.** §12's test compose file differs from the main one only by `LLM_MOCK=true` and a Playwright service — a compose profile or an override file expresses that without a second full copy. | One file, no duplicated service definition to keep in sync |

### 14.8 The single highest-leverage change

If only one item from §13 or §14 is acted on before building resumes, make it **§14.2 — the
`.gitignore`**. Every other finding here produces a symptom an agent can see and debug. That one
produces a repository that *looks* correct locally, passes review, and is broken for everyone who
clones it — and the longer the frontend is built before it is fixed, the more work silently
disappears.
