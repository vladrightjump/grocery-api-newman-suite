# Grocery API — Newman automation suite

Functional, negative, contract, auth, and end-to-end workflow tests for the [Simple Grocery Store API](https://simple-grocery-store-api.click), runnable locally and in CI via Newman.

## Quick start

```bash
npm install
npm test                # main collection
npm run test:advanced   # advanced contract / negative / idempotency suite
npm run test:all        # main + 3 data-driven runs + advanced
```

HTML reports land in `results/`. Open `results/report-main.html` (or `report-advanced.html`) to inspect a run. Allure raw results are written under `results/allure-results/<suite>/`.

To target a single folder:

```bash
npx newman run collections/grocery.postman_collection.json \
  -e environments/prod.postman_environment.json \
  --folder "03 Cart"
```

To switch environments (e.g., once a local mock is wired up):

```bash
npm test -- -e environments/local.postman_environment.json
```

## Layout

```
postmannewman/
├── collections/
│   ├── grocery.postman_collection.json        # main suite (folders 00–99)
│   ├── grocery.data.postman_collection.json   # data-driven companion (3 folders)
│   └── grocery.advanced.postman_collection.json # strict contract, pagination, negative bodies, idempotency
├── environments/
│   ├── prod.postman_environment.json          # baseUrl → live API
│   └── local.postman_environment.json         # baseUrl → http://localhost:3000
├── data/
│   ├── categories.csv                         # 7 valid + 2 invalid categories
│   ├── results-bva.csv                        # boundary values for ?results=
│   └── quantities.csv                         # valid + invalid add-item quantities
├── .github/workflows/newman.yml               # push/PR + 6-hourly monitoring run
├── package.json
├── .gitignore
└── README.md
```

## Coverage strategy

Five coverage axes, applied per endpoint where each is relevant:

1. **Functional** — happy-path status, body, headers.
2. **Negative** — missing/invalid params, boundary values, malformed JSON.
3. **Contract** — JSON Schema validation (via `tv4` in the Postman sandbox) of success *and* error response shapes. Schemas live in the collection-level pre-request script so any request can call `tv4.validate(body, schemas.x)`.
4. **Auth** — missing / invalid / malformed bearer token on protected routes.
5. **Workflow / state** — chained lifecycle in `99 E2E Workflow` proving state transitions (cart→order→deletion, PATCH/PUT persistence, DELETE removes).

The collection has ~50 requests, ~90 assertions.

## Coverage matrix

| Endpoint | Positive | Negative / boundary | Contract | Auth |
|---|---|---|---|---|
| `GET /status` | 200, `status==UP` | — | status schema | — |
| `POST /api-clients` | 201, token persisted | 400 missing name / missing email / empty / malformed JSON; 409 duplicate | token + error schema | — |
| `GET /products` | 200, ≤20 default | `category` (data-driven), `results` BVA (data-driven), `available=notabool` → 400 | array + product schema | — |
| `GET /products/:id` | 200 matches; `?product-label` content-type pinned | 404 nonexistent, 400 non-numeric id | product + error schema | — |
| `POST /carts` | 201, `created==true` | — | create-cart schema | — |
| `GET /carts/:id` | 200 | 404 bad id | cart schema | — |
| `GET /carts/:id/items` | 200, empty initially | 404 bad cart | — | — |
| `POST /carts/:id/items` | 201, default qty=1 verified | 400 missing/invalid/out-of-stock productId, malformed JSON; 404 bad cart; qty data-driven | add-item schema | — |
| `PATCH .../items/:itemId` | 204 + GET verifies | 404 bad item | — | — |
| `PUT .../items/:itemId` | 204 + GET verifies + idempotent | — | — | — |
| `DELETE .../items/:itemId` | 204 + GET verifies + 404 on second delete | — | — | — |
| `GET /orders` | 200 array | — | — | 401 no/invalid/malformed |
| `POST /orders` | 201, orderId stored, cart→404 | 400 missing customerName / missing cartId / invalid cartId | create-order schema | 401 |
| `GET /orders/:id` | 200 matches; `?invoice=true` content-type pinned | 404 | order schema | 401 |
| `PATCH /orders/:id` | 204 + GET verifies | — | — | 401 |
| `DELETE /orders/:id` | 204 + GET→404 | 404 bad id (docs say 400; reality is 404 — pinned to reality) | — | 401 |

## Resolution of §8 open questions

The original plan listed two unknowns; both were probed against the live API and pinned to actual behavior:

1. **Out-of-stock add-to-cart** → returns `400` with `{"error": "This product is not in stock and cannot be ordered."}`.
2. **Bad cartId on `/items` endpoint** → returns `404` with `{"error": "No cart with id <X>."}`. (Same for GET and POST.)

A few extra surprises uncovered during probing and pinned in the suite:

- `?invoice=true` on `GET /orders/:id` and `?product-label=true` on `GET /products/:id` both return JSON, not PDF as the docs imply. The content-type assertion accepts either so a future fix to match docs won't break the suite.
- `DELETE /orders/<bad-id>` returns `404`, not `400` as docs suggest.
- `GET /products?results=` is lenient: `0`, `abc`, `10.5` all return `200`. Only `21` (too big) and `-1` (negative) return `400`. The BVA CSV reflects that.
- `POST /carts/:id/items` is also lenient on `quantity`: negative, zero, and fractional values all return `201`. The only `400` comes from a malformed JSON body (e.g. when the row substitutes a non-JSON token). The quantities CSV reflects that.

## CI

`.github/workflows/newman.yml` runs on push and manual dispatch. The matrix runs each suite in parallel (`main`, `categories`, `bva`, `quantities`, `advanced`), uploads htmlextra HTML + JUnit XML + Allure raw results as artifacts, and surfaces JUnit results via `dorny/test-reporter`.

A second job (`allure-report`) downloads every suite's Allure results, merges them, carries history across runs, generates a single dashboard, and publishes it to GitHub Pages (`gh-pages` branch). After the first successful run, enable Pages → Source: `gh-pages` branch in repo settings; the dashboard will be at `https://<owner>.github.io/<repo>/`.

## Advanced contract suite

`grocery.advanced.postman_collection.json` adds tests that the main suite intentionally doesn't cover:

- **Strict schemas** with `additionalProperties:false` — catches any unannounced field the API starts returning.
- **Header contracts** — asserts `Content-Type: application/json` per endpoint.
- **Pagination boundary** — `results=1`, `20`, `21`, `0`, `-5`, `abc`.
- **Schema-violating negatives** — wrong field types, wrong Content-Type, array body where object expected, zero/negative quantity.
- **Repeated-write idempotency** — issues the same PATCH/PUT twice, then GETs to prove state is stable.
- **Read-after-write consistency** — three consecutive GETs to detect eventual-consistency drift.

When running advanced against the live API, several **contract gaps** are pinned in-place (comments marked `CONTRACT GAP:`): the API returns 200 for `results=0` / `results=abc`, and coerces string `productId` / `quantity` to numbers instead of rejecting with 400.

## Roadmap — Phase 5 (deferred)

The plan calls for an OpenAPI 3.1 spec + Portman / `openapi-to-postman` baseline that auto-generates contract assertions from the schema. That layer can be added later without disturbing this collection; the hand-written negative/workflow folders here would then sit on top of the generated baseline.
