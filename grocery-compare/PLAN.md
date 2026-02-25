# Grocery Price Comparison App - Implementation Plan

## Full Product Vision

The end-state app is a **weekly grocery ordering assistant** that:
1. **Persists order history** — tracks all past purchases per vendor with barcodes, building a personal product database over time
2. **Accepts grocery lists via image** — user sends a photo of a handwritten Hebrew grocery list, vision/OCR maps items to known products in the DB
3. **Compares prices** across Shufersal, Rami Levy, and Victory
4. **Shows delivery windows** — available delivery times per vendor
5. **Fills the cart automatically** — uses browser automation to add items to the chosen vendor's cart, select delivery date
6. **Sends a confirmation link** — user just clicks to confirm the order (no manual item-by-item adding)
7. **Suggests deals & alternatives** — shows current promotions and cheaper substitute products
8. **Shared lists** — multiple users (e.g., spouse) can add items to the same list with real-time notifications

## POC Scope (First Implementation)

**Prove that we can**: compare prices across 3 vendors AND fill a cart on a vendor's website AND give the user a link to confirm manually.

Everything else is architected with placeholders but NOT implemented yet.

---

## Project Structure

```
grocery-compare/
├── pyproject.toml
├── .env.example
├── .env                              # gitignored
├── .gitignore
├── src/
│   ├── config.py                     # pydantic-settings, loads .env
│   ├── models.py                     # Core data models (see below)
│   │
│   ├── vendors/
│   │   ├── base.py                   # Abstract VendorClient interface
│   │   ├── rami_levy.py              # Direct HTTP API (httpx)
│   │   ├── shufersal.py              # Playwright browser automation
│   │   └── victory.py                # Playwright browser automation
│   │
│   ├── gov_data/
│   │   ├── scraper.py                # Wrapper: il-supermarket-scraper
│   │   ├── parser.py                 # Wrapper: il-supermarket-parser
│   │   └── price_db.py              # In-memory price DB from parsed XML
│   │
│   ├── matching.py                   # Cross-vendor product matching (barcode + fuzzy Hebrew)
│   │
│   ├── cart/                         # [POC] Cart filling automation
│   │   ├── base.py                   # Abstract CartManager interface
│   │   ├── rami_levy_cart.py         # Add to cart via API
│   │   ├── shufersal_cart.py         # Add to cart via Playwright
│   │   └── victory_cart.py           # Add to cart via Playwright
│   │
│   ├── db/                           # [PLACEHOLDER] Persistence layer
│   │   ├── models.py                 # SQLModel/SQLAlchemy ORM models
│   │   ├── repository.py            # CRUD operations
│   │   └── migrations/              # Alembic migrations
│   │
│   ├── vision/                       # [PLACEHOLDER] Image grocery list parsing
│   │   └── list_parser.py           # OCR/vision API → structured product list
│   │
│   ├── delivery/                     # [PLACEHOLDER] Delivery window scraping
│   │   └── slots.py                 # Fetch available delivery times per vendor
│   │
│   ├── deals/                        # [PLACEHOLDER] Deals & alternatives
│   │   └── promotions.py            # Scrape/parse current sales per vendor
│   │
│   ├── sharing/                      # [PLACEHOLDER] Shared lists + notifications
│   │   ├── list_manager.py          # Shared grocery list state
│   │   └── notifications.py         # Push/webhook notifications
│   │
│   ├── api/
│   │   ├── app.py                    # FastAPI app factory
│   │   ├── routes.py                 # API endpoints
│   │   └── schemas.py               # Request/response models
│   │
│   ├── web/
│   │   ├── templates/
│   │   │   └── index.html
│   │   └── static/
│   │       └── style.css
│   │
│   └── cli.py                        # CLI for quick testing
│
├── scripts/
│   ├── download_gov_data.py
│   └── extract_rami_tokens.py
│
├── tests/
│   ├── test_matching.py
│   ├── test_cart.py
│   ├── test_rami_levy.py
│   ├── test_shufersal.py
│   ├── test_victory.py
│   └── conftest.py
│
└── data/                             # gitignored
    ├── dumps/
    ├── parsed/
    └── browser_sessions/
```

---

## Core Data Models (`src/models.py`)

Designed for the full vision — persistence, order history, shared lists:

```python
class Vendor(str, Enum):
    SHUFERSAL = "shufersal"
    RAMI_LEVY = "rami_levy"
    VICTORY = "victory"

class Product(BaseModel):
    vendor: Vendor
    product_id: str              # vendor-specific ID
    barcode: str | None          # EAN-13 — the cross-vendor key
    name: str                    # Hebrew product name
    price: float                 # current price in NIS
    unit_price: float | None     # price per kg/liter
    image_url: str | None
    in_stock: bool = True

class PriceComparison(BaseModel):
    query: str
    results: dict[Vendor, Product | None]
    cheapest: Vendor | None

class GroceryListItem(BaseModel):
    name: str                    # user-entered name or image-parsed name
    quantity: int = 1
    matched_barcode: str | None  # resolved barcode from matching engine

class GroceryList(BaseModel):
    items: list[GroceryListItem]
    # [FUTURE] owner_id, shared_with, created_at, etc.

class CartSession(BaseModel):
    """Result of filling a vendor's cart — the confirmation link."""
    vendor: Vendor
    cart_url: str                # URL user opens to confirm order
    items_added: int
    items_failed: list[str]      # items that couldn't be added
    total_estimate: float
    # [FUTURE] delivery_slot: DeliverySlot | None

# --- PLACEHOLDER MODELS (future phases) ---
# class OrderHistory — tracks past confirmed orders with barcodes + prices
# class DeliverySlot — available delivery windows per vendor
# class Deal — current promotions per vendor
# class UserProfile — for shared lists, preferences, household
```

---

## Vendor Integration Details

### Rami Levy — Direct HTTP API (highest confidence)

A reverse-engineered REST API is documented (via `rami-levy-mcp` project):

- **Search**: `POST https://www.rami-levy.co.il/api/catalog` with `{"q": "term", "store": "331"}`
- **Cart add**: `POST /api/cart` with product IDs and quantities
- **Auth**: Bearer token + ecomtoken + cookie extracted from browser localStorage
- **Response fields**: `id`, `name`, `barcode`, `price`, `images`, `available_in[]`

Token extraction helper (`scripts/extract_rami_tokens.py`): opens Playwright browser, user logs in, script extracts token from `JSON.parse(localStorage.ramilevy).authuser.user.token`.

**Token auto-refresh strategy**: Detect 401 responses → automatically launch headless Playwright to re-authenticate using saved credentials (or prompt user if credentials changed). Store refresh logic in the client itself.

### Shufersal — Playwright Browser Automation

No public API exists (confirmed by `shufersal-mcp` project which also uses Puppeteer).

- **Session**: Persistent Playwright browser context (saves cookies/session)
- **First run**: Browser opens, user logs in manually, session saved to `data/browser_sessions/shufersal/`
- **Search strategy**: Navigate to search URL → intercept XHR/fetch responses (more robust than DOM scraping) → fall back to DOM scraping if needed
- **Cart filling**: Search for each product, click "Add to Cart" button, set quantity

### Victory — Playwright Browser Automation

Runs on **stor.ai** white-label e-commerce platform. No public API docs.

- Same persistent browser context pattern as Shufersal
- **Discovery step**: Run `playwright codegen https://www.victoryonline.co.il` to explore DOM structure and XHR endpoints
- **Search strategy**: XHR interception (stor.ai likely has internal API calls) with DOM scraping fallback
- **Cart filling**: Same Playwright automation pattern

---

## Product Matching Strategy (`src/matching.py`)

This is the core algorithmic challenge.

### Tier 1: Exact Barcode Match (EAN-13)
- Gold standard — same barcode = same physical product
- Available from: government XML data, Rami Levy API response, possibly Shufersal/Victory DOM attributes

### Tier 2: Fuzzy Hebrew Name Match
- Uses `rapidfuzz.fuzz.token_sort_ratio` (order-invariant: "תנובה חלב 3%" ≈ "חלב 3% תנובה")
- Threshold ≥ 65% for a match
- Among multiple matches above threshold, prefer the one with matching brand/manufacturer

### Tier 3: Personal History Match (future)
- Once order history is persisted, use past purchases to improve matching
- "User bought product X from Shufersal last week" → use its barcode to find it at Rami Levy

### Known Challenges
- Store brands (no cross-chain equivalent — Shufersal Green vs Rami Levy private label)
- Weight vs unit products (500g cheese vs cheese per kg)
- Pack sizes (6-pack milk vs single)
- Hebrew morphology (prefix letters ב, ל, מ, ה)

---

## Cart Filling Architecture (`src/cart/`)

```python
class CartManager(ABC):
    vendor: Vendor
    async def initialize(self) -> None: ...
    async def clear_cart(self) -> None: ...
    async def add_item(self, product: Product, quantity: int = 1) -> bool: ...
    async def get_cart_url(self) -> str: ...          # THE KEY OUTPUT
    async def get_cart_summary(self) -> CartSession: ...
    async def close(self) -> None: ...
```

**Flow**:
1. User submits grocery list → price comparison runs
2. User picks a vendor (or app picks cheapest)
3. CartManager clears existing cart, adds each item
4. CartManager returns `CartSession` with `cart_url`
5. User opens `cart_url` in their browser → sees pre-filled cart → confirms order

For Rami Levy this is API calls. For Shufersal/Victory this is Playwright clicking through the UI.

---

## API Endpoints

| Method | Path | Description | POC? |
|--------|------|-------------|------|
| `GET` | `/` | Web UI — grocery list input | Yes |
| `POST` | `/api/compare` | Compare prices for a list of items | Yes |
| `POST` | `/api/fill-cart` | Fill cart at chosen vendor, return confirmation link | Yes |
| `POST` | `/api/upload-list` | Upload image of handwritten grocery list | Placeholder |
| `GET` | `/api/delivery-slots/{vendor}` | Get available delivery windows | Placeholder |
| `GET` | `/api/deals/{vendor}` | Get current deals/promotions | Placeholder |
| `GET` | `/api/history` | Past orders and frequently bought items | Placeholder |
| `POST` | `/api/lists/share` | Share grocery list with another user | Placeholder |

---

## Implementation Order

### Phase 1: Government Price Data (no credentials needed)
1. Project skeleton + `pyproject.toml` + `.gitignore`
2. `src/gov_data/scraper.py` — download XML for 3 chains
3. `src/gov_data/parser.py` — XML → DataFrames
4. `src/gov_data/price_db.py` — barcode index + fuzzy search
5. `src/cli.py` — CLI comparison tool

### Phase 2: Live Store Integration
6. `src/models.py` + `src/vendors/base.py`
7. `src/vendors/rami_levy.py` — HTTP API client
8. `scripts/extract_rami_tokens.py` — token extraction helper
9. `src/vendors/shufersal.py` — Playwright client
10. `src/vendors/victory.py` — Playwright client
11. `src/matching.py` — barcode + fuzzy matching

### Phase 3: Cart Filling (core POC differentiator)
12. `src/cart/base.py` — abstract CartManager
13. `src/cart/rami_levy_cart.py` — API-based cart filling
14. `src/cart/shufersal_cart.py` — Playwright cart filling
15. `src/cart/victory_cart.py` — Playwright cart filling

### Phase 4: Web UI
16. FastAPI app + routes (`/api/compare`, `/api/fill-cart`)
17. HTML template — list input, comparison table, "Fill Cart" button → confirmation link
18. Integration tests + smoke test

### Future Phases (placeholder architecture)
- **Phase 5**: SQLite persistence — order history, personal product DB
- **Phase 6**: Image list parsing — Claude vision API for Hebrew handwriting
- **Phase 7**: Delivery windows — scrape/display available slots
- **Phase 8**: Deals & alternatives — parse promotions, suggest cheaper options
- **Phase 9**: Shared lists — multi-user with notifications

---

## Verification Plan

1. `python scripts/download_gov_data.py` → XML files in `data/dumps/`
2. `python -m src.cli "חלב תנובה 3%"` → comparison table with prices from all 3 chains
3. `python scripts/extract_rami_tokens.py` → tokens extracted → `.env`
4. `pytest tests/test_rami_levy.py -m integration` → search returns products
5. `pytest tests/test_shufersal.py -m integration` → Playwright search works
6. `pytest tests/test_victory.py -m integration` → Playwright search works
7. **Cart test**: fill cart with 3 items at Rami Levy → get cart URL → open in browser → items are there
8. `uvicorn src.api.app:app --reload` → web UI: enter list → compare → pick vendor → fill cart → confirmation link

---

## Key Risks

| Risk | Mitigation |
|------|------------|
| Rami Levy tokens expire | Auto-refresh: detect 401 → re-authenticate via Playwright headless |
| Shufersal DOM unknown | Prefer XHR interception over DOM scraping; `playwright codegen` for exploration |
| Victory/stor.ai undocumented | `playwright codegen`; stor.ai patterns reusable across stores |
| Hebrew fuzzy matching accuracy | Show confidence scores; user confirms; improves with order history |
| Cart filling breaks on site changes | Modular cart managers; easy to update selectors per vendor |
| Anti-bot detection | Persistent sessions with real login; human-like delays; headful browser |

---

## Reference Projects & Resources

- [il-supermarket-scraper](https://github.com/OpenIsraeliSupermarkets/israeli-supermarket-scarpers) — government price XML downloader
- [il-supermarket-parser](https://github.com/OpenIsraeliSupermarkets/israeli-supermarket-parsers) — XML parser
- [rami-levy-mcp](https://github.com/shilomagen/rami-levy-mcp) — documented Rami Levy API
- [shufersal-mcp](https://github.com/matipojo/shufersal-mcp) — Shufersal Puppeteer automation reference
- [groceries-mcp](https://github.com/o-b-one/groceries-mcp) — multi-chain grocery MCP
- [israeli-supermarket-data](https://github.com/AKorets/israeli-supermarket-data) — additional data tools
