# Postman + Newman + GitHub Actions (Store API)

Automated integration test suite for a mock REST **Store API** (`/products`, `/orders`, `/users`), written in Postman and executed on every push via a Newman-powered GitHub Actions pipeline. Test reports are published to GitHub Pages.

**Live test report:** https://andriygvozd.github.io/Postman-newman-ghActions/

## Tech stack

| Layer            | Tool                                                                 |
|-------------------|-----------------------------------------------------------------------|
| Mock API server   | [yaml-server](https://www.npmjs.com/package/yaml-server) — REST API backed by a YAML "database" (`mockApi/db_stage.yaml`) |
| API tests         | [Postman Collection v2.1](store.collection.json) (`pm.test` / Chai assertions) |
| Test runner       | [Newman](https://github.com/postmanlabs/newman) (Postman's CLI collection runner) |
| Test report       | [newman-reporter-htmlextra](https://github.com/DannyDainton/newman-reporter-htmlextra) |
| CI                | GitHub Actions ([.github/workflows/newman.yml](.github/workflows/newman.yml)) |
| Report hosting    | GitHub Pages, deployed via [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) |
| Runtime           | Node.js 20 |

## Running it locally

```bash
npm install
npm run tern-on-api
```

This resets `mockApi/db_stage.yaml` from `mockApi/db_back_up.yaml` and starts the mock API on `http://localhost:3000`.

Then, in a second terminal, either:

- **Postman app** — import [store.collection.json](store.collection.json) and run it with the Collection Runner, or
- **Newman CLI**:

```bash
npm install -g newman newman-reporter-htmlextra
newman run store.collection.json --reporters cli,htmlextra --reporter-htmlextra-export newman/index.html
```

The `baseUrl` collection variable defaults to `http://localhost:3000`.

## API coverage

Same CRUD + query surface for all three resources — `products`, `orders`, `users`:

| Verb   | Route            | Input      | Output             |
|--------|------------------|------------|---------------------|
| GET    | `/{resource}`              | *none*                     | array               |
| GET    | `/{resource}?page&pageSize`| pagination query params    | array (page slice)  |
| GET    | `/{resource}?sortOrder&sortKey` | sorting query params   | array (sorted)       |
| GET    | `/{resource}/:id`          | id                          | object               |
| POST   | `/{resource}`               | object                      | created object       |
| PUT    | `/{resource}`                | object (with `id`)          | updated object        |
| DELETE | `/{resource}/:id`          | id                           | deleted object        |

## What's new in this collection

The base template only shipped assertions on `List products`. This version adds full, symmetric test coverage across **Products**, **Orders** and **Users**:

- Status-code assertions on every request (`200`/`201`, plus `404` on post-delete checks)
- Response-time assertions (`< 200–300ms`) on every request
- JSON Schema validation on `Get {resource} by ID` (required fields + types, via `pm.response.to.have.jsonSchema`)
- Request/response "echo" assertions on `Create` and `Update` — verifies the response body reflects the fields that were sent
- `Verify {resource} deleted` requests — a `GET` issued right after `DELETE`, asserting `404`, to prove the delete actually took effect
- Pagination tests (`?page=1&pageSize=2`) asserting the returned array length
- Sorting tests (`?sortOrder=ASC&sortKey=...`) asserting the array is actually sorted ascending
- No hardcoded record ids — `Create {resource}` captures the server-generated `id` into a collection variable (`productId` / `orderId` / `userId`) in its own `pm.test`, and `Get` / `Update` / `Remove` / `Verify deleted` all reuse `{{productId}}` etc. instead of a fixed number. Each resource's `Create → Get → Update → Delete → Verify` chain now runs against the record it just created, so the suite no longer depends on seed data or execution order.

## Testing patterns used

- **AAA (Arrange–Act–Assert)** — request body/query params are the *arrange*, the HTTP call is the *act*, `pm.test` blocks are the *assert*.
- **Status-code assertions** — `pm.response.to.have.status(code)`.
- **Response-time assertions** — `pm.expect(pm.response.responseTime).to.be.below(ms)`.
- **JSON Schema validation** — a schema object with `required` + `properties` checked via `pm.response.to.have.jsonSchema(schema)` (Ajv under the hood).
- **Round-trip / echo assertions** — parse `pm.request.body.raw` and compare fields against `pm.response.json()` to confirm the server persisted what was sent.
- **Negative-state verification** — chained `DELETE` → `GET` requests asserting `404`, proving a side effect rather than just a 200 response.
- **Query-parameter behavior tests** — pagination (array length) and sorting (`array.slice().sort()` comparison) checks.
- **State propagation via collection variables** — `pm.collectionVariables.set(...)` in `Create` captures the generated id; downstream requests read it back with `{{productId}}` (URL/body) or `pm.collectionVariables.get(...)` (assertions), keeping each resource's lifecycle self-contained.

## Code review — does this suite cover the basics of API testing?

**Covered:**
- ✅ Happy-path CRUD for all three resources (Create / Read / Update / Delete)
- ✅ Status codes (`200`, `201`, `404`)
- ✅ Response time / performance budget
- ✅ Response schema/contract validation (types + required fields)
- ✅ Data integrity (response echoes request payload correctly)
- ✅ Side-effect verification (delete actually removes the record)
- ✅ Query-parameter behavior (pagination, sorting)
- ✅ `AAA` structure, consistent naming, one concern per `pm.test`
- ✅ No hardcoded record ids — each resource's `Create` request captures the generated id into a collection variable, reused by every subsequent step in that resource's lifecycle, so the suite doesn't depend on seed data or fixed run order

**Gaps / recommendations:**
- ⚠️ **No negative/invalid-input tests** — e.g. `POST` with a missing required field or wrong type should be asserted to return `400`/`422`; currently only valid payloads are exercised.
- ⚠️ **No 404 test for `GET`/`PUT`/`DELETE` on a non-existent ID** as an isolated case — the only 404 coverage is the delete-then-verify chain, on the record the test itself just created.
- ⚠️ **Schema validation only on single-item `GET`** — list endpoints (`List products`, `List orders`, `List users`) check `status` only, not that each array item matches the resource schema.
- ⚠️ **`Content-Type` header check only exists on `List products`** — not applied consistently to the other requests.
- ⚠️ **No edge cases for pagination/sorting** — e.g. `page` beyond the last page, invalid `sortKey`, `pageSize=0`.
- ⚠️ **Inconsistent response-time budgets** (`200ms` for `List products`, `300ms` everywhere else) with no stated rationale — worth standardizing or documenting.
- ⚠️ **No dedicated test/staging environment file** — only a single `baseUrl` collection variable; a proper Postman Environment (`local`, `ci`) would make the CI target explicit and swappable.

## CI/CD pipeline

The pipeline (see [.github/workflows/newman.yml](.github/workflows/newman.yml)) runs automatically on every push/PR to `main`, **and can also be triggered manually** at any time:

1. Install dependencies (`npm ci`) and Newman + `newman-reporter-htmlextra`.
2. Start the mock API (`npm run tern-on-api`) and wait until it responds.
3. Run `newman run store.collection.json` with `cli` + `htmlextra` reporters.
4. Upload the HTML report as a workflow artifact.
5. Publish the report to the `gh-pages` branch, served at https://andriygvozd.github.io/Postman-newman-ghActions/.

### Running the tests manually (no code change needed)

Want to check the API right now, without pushing anything? Trigger the pipeline by hand from the GitHub Actions tab:

1. Open the repository on GitHub: https://github.com/AndriyGvozd/Postman-newman-ghActions
2. Click the **Actions** tab (top navigation bar, between *Pull requests* and *Projects*).
3. In the left sidebar, click **Newman API Tests** (the only workflow listed).
4. On the right, click the **Run workflow** dropdown button.
5. Leave the branch as **main** and click the green **Run workflow** button inside the dropdown.
6. A new run appears at the top of the list within a few seconds (a yellow/orange dot means it's in progress). Click it to watch the live logs.
7. When it finishes with a green check, open **Deploy report to GitHub Pages** in the run's job list, or just go straight to the published report at https://andriygvozd.github.io/Postman-newman-ghActions/ — it's overwritten with the fresh results (usually live within a minute of the run finishing).

No local setup, no Postman app required — this runs the exact same collection (`store.collection.json`) against a freshly started mock API on GitHub's runner.
