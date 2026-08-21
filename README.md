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

## Testing patterns used

- **AAA (Arrange–Act–Assert)** — request body/query params are the *arrange*, the HTTP call is the *act*, `pm.test` blocks are the *assert*.
- **Status-code assertions** — `pm.response.to.have.status(code)`.
- **Response-time assertions** — `pm.expect(pm.response.responseTime).to.be.below(ms)`.
- **JSON Schema validation** — a schema object with `required` + `properties` checked via `pm.response.to.have.jsonSchema(schema)` (Ajv under the hood).
- **Round-trip / echo assertions** — parse `pm.request.body.raw` and compare fields against `pm.response.json()` to confirm the server persisted what was sent.
- **Negative-state verification** — chained `DELETE` → `GET` requests asserting `404`, proving a side effect rather than just a 200 response.
- **Query-parameter behavior tests** — pagination (array length) and sorting (`array.slice().sort()` comparison) checks.

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

**Gaps / recommendations:**
- ⚠️ **No negative/invalid-input tests** — e.g. `POST` with a missing required field or wrong type should be asserted to return `400`/`422`; currently only valid payloads are exercised.
- ⚠️ **No 404 test for `GET`/`PUT`/`DELETE` on a non-existent ID** as an isolated case — the only 404 coverage is the delete-then-verify chain, always on the same record.
- ⚠️ **Hardcoded IDs** (`3`, `4`, `10`) couple requests to the current seed data and to run order — if `db_back_up.yaml` changes or a request runs out of order/in isolation, the collection breaks. Prefer capturing ids from `Create` responses into collection/environment variables (e.g. `pm.collectionVariables.set("productId", jsonData.id)`) and reusing `{{productId}}` downstream.
- ⚠️ **Schema validation only on single-item `GET`** — list endpoints (`List products`, `List orders`, `List users`) check `status` only, not that each array item matches the resource schema.
- ⚠️ **`Content-Type` header check only exists on `List products`** — not applied consistently to the other requests.
- ⚠️ **No edge cases for pagination/sorting** — e.g. `page` beyond the last page, invalid `sortKey`, `pageSize=0`.
- ⚠️ **Inconsistent response-time budgets** (`200ms` for `List products`, `300ms` everywhere else) with no stated rationale — worth standardizing or documenting.
- ⚠️ **No dedicated test/staging environment file** — only a single `baseUrl` collection variable; a proper Postman Environment (`local`, `ci`) would make the CI target explicit and swappable.

## CI/CD pipeline

On every push/PR to `main` (see [.github/workflows/newman.yml](.github/workflows/newman.yml)):

1. Install dependencies (`npm ci`) and Newman + `newman-reporter-htmlextra`.
2. Start the mock API (`npm run tern-on-api`) and wait until it responds.
3. Run `newman run store.collection.json` with `cli` + `htmlextra` reporters.
4. Upload the HTML report as a workflow artifact.
5. Publish the report to the `gh-pages` branch, served at https://andriygvozd.github.io/Postman-newman-ghActions/.
