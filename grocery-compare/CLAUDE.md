# Grocery Price Comparison App

## Current State
- Branch: `claude/grocery-price-comparison-5DM6d`
- Full implementation plan at `grocery-compare/PLAN.md`
- No code written yet — ready to start Phase 1

## What to Build (POC)
Compare grocery prices across **Shufersal**, **Rami Levy**, and **Victory**, then **fill a cart** on the chosen vendor's website and give the user a **confirmation link** to complete the order.

## Full Vision (beyond POC)
- Persist order history with barcodes → personal product DB that improves matching over time
- Accept grocery lists via **photo of handwritten Hebrew list** (Claude vision API)
- Show **delivery windows** per vendor
- Show **deals/promotions** and suggest cheaper alternatives
- **Shared lists** — spouse can add items with real-time notifications

## Key Technical Findings from Research
- **Government price data**: Israel's transparency law requires all chains to publish XML price data. `il-supermarket-scraper` + `il-supermarket-parser` Python packages handle download/parsing. No credentials needed.
- **Rami Levy**: Has a reverse-engineered REST API (documented at github.com/shilomagen/rami-levy-mcp). Search: `POST /api/catalog`, Cart: `POST /api/cart`. Auth: Bearer token from `localStorage.ramilevy`.
- **Shufersal**: No API — requires Playwright browser automation. Reference project: github.com/matipojo/shufersal-mcp
- **Victory**: No API — runs on stor.ai white-label platform. Playwright automation needed.
- **Product matching**: Barcode (EAN-13) exact match is gold standard. Fallback: fuzzy Hebrew name matching via `rapidfuzz` with `token_sort_ratio`.
- **Token management**: Auto-refresh on 401 — re-authenticate via Playwright headless.

## Implementation Order
1. Phase 1: Gov data pipeline → CLI price comparison (zero risk, no credentials)
2. Phase 2: Live vendor clients (Rami Levy HTTP API, Shufersal + Victory Playwright)
3. Phase 3: Cart filling automation + confirmation link generation
4. Phase 4: FastAPI web UI

## User Context
- Deployed from home in Israel (no geo-blocking issues with gov data)
- Has personal accounts at all 3 vendors
- Wants auto-refresh for expired tokens (not manual)
- All code goes in `grocery-compare/` directory

## Dependencies
fastapi, uvicorn, jinja2, httpx, playwright, il-supermarket-scraper, il-supermarket-parser, pandas, rapidfuzz, pydantic-settings, python-dotenv, loguru
