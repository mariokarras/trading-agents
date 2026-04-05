# Technology Stack

**Project:** TradingAgents v0.3 — Multi-Agent Trading Research Platform
**Researched:** 2026-04-04
**Overall confidence:** MEDIUM-HIGH (most recommendations verified via PyPI and official docs; API pricing verified via vendor sites)

---

## Existing Stack (Retained)

These are already in the project and remain the correct choices. No changes recommended.

| Technology | Current Version | Purpose | Status |
|------------|----------------|---------|--------|
| Python | 3.13 | Runtime | Keep. Well-supported by all deps |
| LangGraph | >=0.4.8 (pyproject) | Agent orchestration | **Upgrade to >=1.1.3** (latest stable, GA since late 2025) |
| LangChain Core | >=0.3.81 | LLM abstractions | Keep. Aligns with LangGraph 1.x |
| langchain-openai | >=0.3.23 | OpenAI LLM provider | Keep |
| langchain-anthropic | >=0.3.15 | Claude LLM provider | Keep |
| yfinance | >=0.2.63 | Market data, **options chains** | Keep. Already provides options chain data |
| Rich | >=14.0.0 | CLI formatting | **Upgrade to >=14.1.0** (latest, Aug 2025) |
| Typer | >=0.21.0 | CLI framework | Keep. Latest as of Feb 2026 |
| pandas | >=2.3.0 | Data manipulation | Keep |
| Redis | >=6.2.0 | Caching layer | Keep for LLM response caching |

**Confidence:** HIGH -- versions verified against PyPI as of April 2026.

---

## New: Semantic News Search (Exa API)

### Recommended Stack

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `exa-py` | >=2.10.2 | Exa Python SDK | Official SDK, actively maintained (last release Mar 26, 2026). MIT license. |
| `langchain-exa` | >=1.1.0 | LangChain integration | First-party LangChain integration (released Mar 26, 2026). Wraps Exa as a LangChain Tool, direct drop-in for agent tool lists. |

### Exa API Details

| Detail | Value |
|--------|-------|
| **Pricing** | $7 per 1,000 searches. $10 free credits on signup. |
| **Rate limits** | 10 QPS default (600 req/min). Enterprise can negotiate higher. |
| **Max results per search** | 10 (standard), up to 1,000 (enterprise) |
| **Auth** | API key via `EXA_API_KEY` env var |
| **Content retrieval** | Can return full page text or highlights alongside search results |

### Why Exa Over Alternatives

Exa uses **embeddings-based semantic search** -- it understands meaning, not just keywords. This matters for financial news because:
- "Fed signals hawkish stance" should match "interest rate hike expectations" -- keyword search misses this
- Exa returns structured results with text content, ideal for feeding directly into LLM analyst agents
- Native LangChain integration means zero custom wrapper code

### What NOT to Use

| Alternative | Why Not |
|-------------|---------|
| Brave Search API | Keyword-based, not semantic. Weaker at conceptual matching for financial research. |
| Tavily | Decent but Exa's semantic model is stronger for discovery-oriented queries. Tavily is better for factual lookups. |
| Google Custom Search | No semantic capability. Rate limits are restrictive. Overkill setup for this use case. |

**Confidence:** MEDIUM-HIGH. Exa pricing and SDK versions verified via PyPI and exa.ai/pricing. Semantic quality claim based on multiple independent comparisons but not personally benchmarked.

---

## New: Real-Time Web Research (Perplexity API)

### Recommended Stack

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `langchain-perplexity` | latest | LangChain integration | First-party package. Provides ChatPerplexity (LLM) and PerplexitySearchTool (tool). |

**Do NOT use `exa-py` or raw `requests` for this.** The `langchain-perplexity` package gives you both a chat model and a search tool that plug directly into LangGraph nodes.

### Perplexity API Details

| Detail | Value |
|--------|-------|
| **Model: Sonar** | Input: $1/M tokens, Output: $1/M tokens + per-request search fee |
| **Model: Sonar Pro** | Input: $3/M tokens, Output: $15/M tokens + per-request search fee |
| **Model: Sonar Deep Research** | Input: $2/M, Output: $8/M, Citations: $2/M, Search: $5/1K queries, Reasoning: $3/M |
| **Rate limits** | 50 RPM (most models). Search API: 3 RPS. Tiered system (0-5) based on cumulative spend. |
| **Free tier** | None. Pro subscribers get $5/mo API credit. Must add payment method. |
| **Auth** | API key via `PERPLEXITY_API_KEY` env var |
| **2026 update** | Citation tokens no longer billed for standard Sonar and Sonar Pro |

### Recommended Model Selection

| Use Case | Model | Rationale |
|----------|-------|-----------|
| Quick market news lookup | `sonar` | Cheapest. Good for "What happened with AAPL today?" |
| Deep research synthesis | `sonar-pro` | Better reasoning, more thorough citations |
| Complex multi-source analysis | `sonar-deep-research` | When you need the agent to really dig. Use sparingly -- expensive. |

### What NOT to Use

| Alternative | Why Not |
|-------------|---------|
| Raw web scraping for news | Fragile, requires maintenance, no synthesis |
| OpenAI web browsing | Not available via API. ChatGPT-only feature. |
| You.com API | Less mature, smaller index |

**Confidence:** MEDIUM-HIGH. Pricing verified via docs.perplexity.ai and multiple cross-references. Rate limits verified. langchain-perplexity package confirmed on PyPI.

---

## New: SEC Filing Scraping (Firecrawl)

### Recommended Stack

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `firecrawl-py` | >=4.22.0 | Firecrawl Python SDK | Latest release Apr 3, 2026. Handles JS-rendered pages, returns clean markdown. |

### Firecrawl API Details

| Detail | Value |
|--------|-------|
| **Free tier** | 500 credits (500 pages at 1 credit/page) |
| **Hobby plan** | $16/mo |
| **Standard plan** | $83/mo, 200 req/min |
| **Growth plan** | Higher rate limits, 500 req/min |
| **Credit cost** | 1 credit/page base. JSON mode: +4 credits. Enhanced proxy: +4 credits. |
| **Rate limits (free)** | 10 scrape/min, 1 crawl/min, 5 search/min |
| **Data retention** | Crawl results available 24 hours. Screenshot URLs expire 24 hours. |
| **Auth** | API key via `FIRECRAWL_API_KEY` env var |

### SEC Filing Strategy

For SEC filings specifically:
1. **EDGAR direct** for structured data (10-K, 10-Q XBRL) -- free, no API key needed
2. **Firecrawl** for unstructured filings, proxy statements, earnings call transcripts hosted on investor relations pages
3. Use Firecrawl's `scrape` mode (not `crawl`) for individual filing pages to conserve credits

### What NOT to Use

| Alternative | Why Not |
|-------------|---------|
| BeautifulSoup + requests | Works for simple EDGAR pages but fails on JS-rendered investor relations sites. No markdown conversion. |
| Selenium/Playwright | Heavy dependency, complex setup, no LLM-optimized output format |
| Jina Reader API | Less mature, fewer features than Firecrawl for structured extraction |
| SEC-API.io | Paid service with narrower scope. Firecrawl is more general-purpose. |

**Note:** For basic SEC EDGAR access (directly from sec.gov), consider the free `edgartools` or `sec-api` Python packages first. Use Firecrawl only when you need to scrape rendered pages (investor relations sites, earnings transcripts, etc.).

**Confidence:** HIGH for Firecrawl SDK version/pricing (verified PyPI + firecrawl.dev/pricing). MEDIUM for SEC-specific workflow (based on general web scraping patterns, not tested against specific filing types).

---

## New: Options Chain Data & Greeks Calculation

### Recommended Stack

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `yfinance` | >=0.2.63 | Options chain data | **Already in project.** Provides full chains with strikes, bid/ask, volume, OI, IV per contract. Free. No API key. |
| `py_vollib` | >=1.0.3 | Greeks calculation (Black-Scholes, Black-Scholes-Merton) | Best-in-class for analytical Greeks. Uses LetsBeRational for fast IV solving. |
| `scipy` | >=1.14.0 | Numerical methods dependency | Required by py_vollib. Also useful for custom numerical Greeks if needed. |
| `numpy` | >=2.0.0 | Array operations | Fast vectorized calculations across option chains |

### Options Data Pipeline

```
yfinance.Ticker("AAPL").options          -> list of expiry dates
yfinance.Ticker("AAPL").option_chain(dt) -> calls DataFrame, puts DataFrame
                                            Columns: strike, bid, ask, lastPrice,
                                            volume, openInterest, impliedVolatility
```

### Greeks Calculation Strategy

| Greek | Source | Method |
|-------|--------|--------|
| **Delta** | py_vollib | `analytical.delta(flag, S, K, t, r, sigma)` |
| **Gamma** | py_vollib | `analytical.gamma(flag, S, K, t, r, sigma)` |
| **Theta** | py_vollib | `analytical.theta(flag, S, K, t, r, sigma)` |
| **Vega** | py_vollib | `analytical.vega(flag, S, K, t, r, sigma)` |
| **Rho** | py_vollib | `analytical.rho(flag, S, K, t, r, sigma)` |
| **IV Rank** | Custom calculation | Compare current IV to 52-week high/low IV range from historical data |
| **IV Percentile** | Custom calculation | % of trading days in past year with IV below current level |

### Unusual Activity Detection

No free API provides this directly. Build it from yfinance data:
- **Volume/OI ratio** > 3x = unusual volume
- **Volume spike** vs 20-day average > 2 standard deviations
- **Put/Call ratio** deviations from historical norms per ticker
- **Large premium trades** = high volume + far OTM + near expiry

### What NOT to Use

| Alternative | Why Not |
|-------------|---------|
| `mibian` | Last updated 2016 (v0.1.3). Abandoned. Use py_vollib instead. |
| `QuantLib-Python` (v1.40) | Massively over-engineered for this use case. 500+ MB install. Great for exotic derivatives pricing, overkill for vanilla Greeks on equity options. |
| CBOE DataShop API | Paid subscription only ($$$). No free tier. Overkill for personal use. |
| `optlib` | Depends on deprecated TDAmeritrade API. Not viable. |
| `yoptions` | Small package, limited maintenance signal. yfinance covers same ground better. |

**Confidence:** HIGH for yfinance options data (well-documented, widely used). MEDIUM for py_vollib (functional and correct, but last PyPI update not recent -- the math is stable though, Black-Scholes doesn't change). LOW for unusual activity detection (custom heuristics, needs tuning).

---

## New: Portfolio Tracking (SQLite)

### Recommended Stack

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `sqlite3` | (stdlib) | Database engine | Built into Python. Zero dependency. Single .db file. Perfect for personal use. |
| `sqlmodel` | >=0.0.22 | ORM / schema definition | Pydantic + SQLAlchemy. Type-safe models. Same author as FastAPI/Typer (tiangolo). |

### Why sqlmodel Over Raw sqlite3

- **Type safety**: Pydantic models for portfolio positions prevent data errors
- **Migration-friendly**: Schema changes are explicit Python class changes
- **Serialization**: Models serialize to JSON trivially (for CLI output and agent consumption)
- **Ecosystem fit**: Same Pydantic models work with Typer CLI and Rich tables

### Alternative Considered

| Alternative | Why Not |
|-------------|---------|
| Raw `sqlite3` + dicts | Works but error-prone. No schema enforcement. Manual serialization. |
| SQLAlchemy alone | Heavier than needed. sqlmodel wraps it with better DX. |
| Peewee | Lighter ORM but no Pydantic integration. |
| TinyDB | JSON-based, no SQL. Slower for queries across positions. |
| PostgreSQL/Supabase | Over-engineered for personal local use. SQLite is correct here. |

### Suggested Schema

```sql
-- Core tables
positions        (id, ticker, asset_type, quantity, avg_cost, opened_at, closed_at, notes)
transactions     (id, position_id, action, quantity, price, fees, executed_at)
snapshots        (id, position_id, market_price, total_value, unrealized_pnl, snapshot_at)
trade_decisions  (id, ticker, decision, confidence, reasoning, agents_involved, decided_at)

-- asset_type: 'stock' | 'option'
-- For options: ticker format like 'AAPL250418C00170000' (OCC symbology)
-- action: 'buy' | 'sell' | 'sell_short' | 'cover'
```

**Confidence:** HIGH. SQLite is the obvious correct choice for local personal portfolio tracking. sqlmodel recommendation is MEDIUM (strong library but version <1.0).

---

## New: Structured CLI Output

### Recommended Stack (Already Mostly in Project)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| `rich` | >=14.1.0 | Tables, panels, progress, syntax highlighting | Already in project. Latest stable. |
| `typer` | >=0.21.0 | CLI framework with Rich integration | Already in project. Built-in Rich support. |

### Output Strategy

| Output Type | Format | Library |
|-------------|--------|---------|
| Agent analysis results | Rich Panel + Table | `rich.panel.Panel`, `rich.table.Table` |
| Portfolio positions | Rich Table | `rich.table.Table` with conditional coloring |
| Trade decisions | Rich Panel with Markdown | `rich.markdown.Markdown` inside `rich.panel.Panel` |
| Options chains | Rich Table | `rich.table.Table` with IV/Greeks columns |
| JSON export | stdout JSON | `json.dumps()` with `--json` flag on Typer commands |
| Debug/trace | Rich Console | `rich.console.Console` with `--verbose` flag |

### Dual Output Pattern

```python
# Every command supports both human and machine output
@app.command()
def analyze(ticker: str, json_output: bool = typer.Option(False, "--json")):
    result = run_analysis(ticker)
    if json_output:
        print(json.dumps(result.model_dump()))
    else:
        render_rich_analysis(result)
```

**Confidence:** HIGH. Rich and Typer are already in the project and are the correct tools.

---

## Installation Summary

```bash
# Upgrade existing
pip install "langgraph>=1.1.3" "rich>=14.1.0"

# New: Semantic search
pip install "exa-py>=2.10.2" "langchain-exa>=1.1.0"

# New: Web research
pip install "langchain-perplexity"

# New: SEC scraping
pip install "firecrawl-py>=4.22.0"

# New: Options Greeks
pip install "py_vollib>=1.0.3"

# New: Portfolio ORM
pip install "sqlmodel>=0.0.22"

# Already satisfied by existing deps
# scipy, numpy (via stockstats/pandas)
```

### Required Environment Variables

```bash
# .env additions
EXA_API_KEY=         # From https://dashboard.exa.ai
PERPLEXITY_API_KEY=  # From https://www.perplexity.ai/settings/api
FIRECRAWL_API_KEY=   # From https://www.firecrawl.dev/app/api-keys
```

---

## Monthly Cost Estimate (Personal Use)

| Service | Estimated Monthly Usage | Estimated Cost |
|---------|------------------------|----------------|
| Exa API | ~2,000 searches | ~$14/mo |
| Perplexity API (Sonar) | ~500 requests | ~$5-10/mo |
| Firecrawl | Free tier (500 credits) or Hobby ($16) | $0-16/mo |
| yfinance | Unlimited | Free |
| SQLite | Local | Free |
| **Total** | | **~$19-40/mo** |

This assumes moderate personal research usage. Heavy backtesting or automated daily runs would increase Exa and Perplexity costs.

---

## Sources

- [Exa API Pricing](https://exa.ai/pricing)
- [Exa Python SDK (PyPI)](https://pypi.org/project/exa-py/)
- [langchain-exa (PyPI)](https://pypi.org/project/langchain-exa/)
- [Perplexity API Pricing](https://docs.perplexity.ai/docs/getting-started/pricing)
- [Perplexity Rate Limits](https://docs.perplexity.ai/docs/admin/rate-limits-usage-tiers)
- [langchain-perplexity (PyPI)](https://pypi.org/project/langchain-perplexity/)
- [Firecrawl Pricing](https://www.firecrawl.dev/pricing)
- [Firecrawl Rate Limits](https://docs.firecrawl.dev/rate-limits)
- [firecrawl-py (PyPI)](https://pypi.org/project/firecrawl-py/)
- [py_vollib (GitHub)](https://github.com/vollib/py_vollib)
- [vollib.org Documentation](https://vollib.org/)
- [Rich 14.1.0 Documentation](https://rich.readthedocs.io/en/latest/)
- [Typer (PyPI)](https://pypi.org/project/typer/)
- [LangGraph (PyPI)](https://pypi.org/project/langgraph/)
- [yfinance Options Documentation](https://ranaroussi.github.io/yfinance/)
- [LangGraph 1.0 GA Announcement](https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available)
