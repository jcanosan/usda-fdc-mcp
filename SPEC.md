# usda-fdc-mcp — MCP server for USDA FoodData Central

Sub-project of Guzzlers-n-Dragons: exposes USDA FDC data as MCP tools so the
recipe pipeline (and any MCP client) can query real-world nutrition data.

## Data source

- Base URL: `https://api.nal.usda.gov/fdc/v1`
- Auth: `api_key` query param. Get free key at api.data.gov/signup
  (data.gov umbrella key). DEMO_KEY only for exploration (30 req/h).
- Rate limits: 1,000 req/h per IP default → respect `X-RateLimit-Remaining`;
  server must handle 429 with backoff, never hammer.
- OpenAPI spec available at `/v1/json-spec` (handy for tests/fixtures).

## Tools

All nutrition amounts: per 100 g for analytical foods (Foundation, SR Legacy,
Survey) unless the record says otherwise; energy in kcal (watch the kJ twin
entries — filter by nutrient unitName).

### 1. `search_foods`
- **USDA endpoint:** `GET /foods/search`
- **Params:** `query` (required, supports USDA search operators), 
  `data_type[]` (Branded | Foundation | Survey (FNDDS) | SR Legacy), 
  `page_size` (1–20, cap above API max to keep context small), 
  `page_number`, `brand_owner` (Branded only)
- **Returns:** compact list: `fdcId`, `description`, `dataType`,
  `brandOwner` if present, `gtinUpc` if present. Truncates description,
  omits nutrient detail (Nutrient panel is the detail tool's job).
- **Notes:** default `data_type = ["Foundation", "SR Legacy"]` — unfiltered
  searches drown in Branded results.

### 2. `get_food`
- **USDA endpoint:** `GET /food/{fdcId}`
- **Params:** `fdc_id`
- **Returns:** normalized food record: description, dataType, portion info
  (`householdServingFullText`, labelNutrients when present), ingredient list
  (Branded), selected nutrient rows (name, amount, unit).
- **Notes:** pass full towering JSON on — trim to relevant nutrient fields
  (energy, macros, fiber, sugars+added, sodium, potassium, iron, calcium,
  vitD first; full list on request via `include_all_nutrients=true`).

### 3. `get_foods`
- **USDA endpoint:** `POST /foods` (batch by IDs)
- **Params:** `fdc_ids` (1–20, enforced client-side)
- **Returns:** one compact entry per ID (same shape as search results).
- **Notes:** exists to amortize rate limit when hydrating recipe ingredient
  lists — the Guzzlers use case.

### 4. `get_nutrients_dv`
- **USDA endpoint:** `GET /food/{fdcId}` (client-side enrichment)
- **Params:** `fdc_id`
- **Returns:** the same nutrient sheet as `get_food` but extended with
  %-Daily-Value per nutrient, computed against the standard FDA DV table
  (2000 kcal reference: energy 2000 kcal, protein 50 g, carbs 275 g,
  fat 78 g, sat-fat 20 g, fiber 28 g, added sugars 50 g + 10% kcal,
  sodium 2300 mg, potassium 4700 mg, calcium 1300 mg, iron 18 mg,
  vit D 20 mcg, cholesterol 300 mg).
- **Notes:** pure calculation tool — keeps logic out of the LLM.

## Non-goals v1
- No write paths (FDC API has none).
- No barcode/UPC-only tool (part of `search_foods` via `gtinUpc` in query).
- No caching layer beyond per-process TTL LRU (FDC data moves slowly);
  cache keyed by fdc_id, TTL 24 h — saves rate limit on repeated lookups.
- DB fallback for arbitrary fictional ingredients stays in the parent
  project; this server is USDA-only.

## Repo scaffold

```
usda-fdc-mcp/
├── SPEC.md                  # this file
├── graph.dot                # Fabro workflow graph
├── pyproject.toml           # uv-managed, py3.13+
├── src/usda_mcp/
│   ├── __init__.py
│   ├── server.py            # FastMCP entrypoint (`mcp` pkg, stdio transport)
│   ├── fdc_client.py        # aiohttp wrapper: auth, rate-limit tracking, retry/backoff
│   ├── normalize.py         # raw FDC → compact models (pydantic)
│   ├── dv.py                # Daily-Value table + %DV math
│   └── config.py            # FDC_API_KEY from env, base URL, timeouts
├── tests/
│   ├── unit/                # normalize, dv math (no network)
│   ├── files/               # recorded FDC response fixtures (golden)
│   └── conformance/         # MCP inspector / client harness smoke tests
├── examples/
│   └── fixture_search.json  # one real search response for dev w/o key
└── .github/workflows/ci.yml # ruff + mypy + pytest (gates mirror graph.dot)
```

## Verification gates (used by graph.dot and CI identically)
1. `ruff check && ruff format --check`
2. `mypy src/`
3. `pytest tests/unit tests/files` (fixtures, no live USDA calls)
4. `pytest tests/conformance` — spawns server over stdio, calls every tool
   once with fixture-backed responses, asserts schema conformance.
5. Human gate: review a real end-to-end call `search_foods("lembas")` →
   `get_nutrients_dv(fdc_id)` against live API (DEMO_KY or real key).

## Error handling contract (all tools)
- USDA 429 → MCP error with human-readable "rate limit; retry after X"
- USDA 400/404 → MCP error, message includes offending params
- Network/timeout (10 s) → retry 2× exponential, then MCP error
- Malformed USDA payload → MCP error, raw failure snippet for debugging

## Fabro notes
- `FDC_API_KEY` must be in Fabro vault (`fabro secret set FDC_API_KEY ...`)
- Goal file: `graph.dot` at repo root; runs in Fabro sandbox with uv
- Frontier models only on `.critical` / `#conformance` selectors; Go models
  fine for scaffolding, docs, fixtures
