# Smart Expense & Cashflow Tracker — 3-Week Study & Development Plan (Team of 2–3)

A focused roadmap to build a small but solid **fintech backend** that tracks expenses, auto-categorizes them, forecasts cash flow, and exposes simple insight/report endpoints—aligned with our course stack (**Spring Boot 3 / Java 17, Postgres + MongoDB, JWT/OAuth2, Docker, GitHub Actions, AWS EC2**). Core features mirror the proposal: **expense tracking**, **cash-flow forecasting**, **actionable insights**, **dashboards/reporting**, optional **bank data integration** and **scheduling**.&#x20;

---

## 1) What “Done” Looks Like (beginner-friendly scope)

* Users sign in (JWT).
* Upload a **transactions CSV** (or call a sandbox) → transactions are **stored & auto-categorized**.&#x20;
* System returns **cash-flow forecast** (next month spend/income) + **insights** (e.g., overspend warnings, unused subscriptions idea).&#x20;
* **Dashboards/Reports** endpoints serve totals, trends, and simple exports.&#x20;
* Optional: **Plaid sandbox** import and a daily **scheduled** refresh.&#x20;
* All services run locally with **Docker Compose** and on **AWS EC2**.
* **OpenAPI/Swagger** is available per service.

> Keep it small: pick **CSV upload first**, add Plaid sandbox only if time remains.

---

## 2) Proposed Microservices (6)

1. **API Gateway** (Spring Cloud Gateway)
   Routes `/auth/**`, `/users/**`, `/transactions/**`, `/categorize/**`, `/forecast/**`, `/reports/**`. Central CORS and (optional) light rate-limit.

2. **Auth Service**
   `/auth/login` issues JWT for `USER`/`ADMIN`. Other services run as **resource servers** validating JWT.

3. **Users & Budgets Service** (PostgreSQL)
   Manage user profile, default currency, and simple **monthly budget** per category.

4. **Transactions & Import Service** (PostgreSQL)
   CSV upload endpoint (and optional Plaid sandbox import) that writes `Transaction` rows (date, amount, merchant, description, raw category). Schedules optional daily refresh.&#x20;

5. **Categorization & Rules Service** (PostgreSQL)
   Rule engine for mapping merchants/keywords → normalized categories; exposes `/categorize` to (re)classify transactions and persist results. “Auto-categorization” lives here.&#x20;

6. **Forecasts, Insights & Reports Service** (MongoDB)
   Stores **forecast snapshots** and **insight documents** (overspend alerts, anomalies, subscription candidates) as documents; exposes `/forecast` and `/reports` for totals, trends, and simple CSV/JSON exports. (This also satisfies our **MongoDB** requirement.)&#x20;

**Why this split?**
It cleanly matches idea features (tracking, forecasting, insights, reporting, 3rd-party import, scheduling) while staying manageable for **2–3 people**.&#x20;

---

## 3) Architecture & Data (minimal, clear)

* **PostgreSQL**: `users`, `budgets`, `transactions`, `categories`, `categorization_rules`.
* **MongoDB**: `forecastSnapshots`, `insights`, `reportCaches` (optional pre-computed aggregates).
* **Security**: JWT on all write endpoints; public GETs only where safe (e.g., docs).
* **Resilience**: short HTTP timeouts for external calls; graceful fallbacks (skip Plaid if down; accept CSV).
* **Config** via env vars & Spring profiles (`dev`, `prod`).
* **Docs**: springdoc-openapi at `/swagger-ui.html` per service.

---

## 4) 3-Week Plan (major steps)

### Week 1 — Foundations & First Slice

**Goal:** skeletons, data models, auth, CSV → stored transactions (no forecast yet).

1. **Kickoff & Board (½ day)**

   * Create **GitHub Project (Kanban)**; columns: To-do / In-progress / In-review / Done.
   * Roles (rotate weekly): **Tech Lead**, **DevOps/CI**, **Data/Domain**.

2. **Bootstrap services (Day 1–2)**

   * Create 6 Spring Boot apps, add deps: Web, Security, Validation, springdoc.
   * Data deps: Postgres in Users/Transactions/Categorization; Mongo in Forecasts/Reports.

3. **Security baseline (Day 2)**

   * Auth issues JWT; other services require JWT on POST/PUT/DELETE.
   * Gateway forwards Authorization header.

4. **Data models & CSV ingest (Day 2–3)**

   * Postgres schemas (Flyway): `Transaction(id, userId, date, amount, merchant, description, rawCategory, normalizedCategory)`.
   * Implement `POST /transactions/import/csv` (multipart) → persist rows.
   * Show totals endpoint: `/transactions/summary?month=YYYY-MM`.

5. **Docker & Compose (Day 3–4)**

   * Multi-stage Dockerfiles (Temurin JRE 17).
   * `docker-compose.yml` for gateway, 5 services, Postgres, Mongo.
   * Bring stack up locally and view Swagger UIs.

**Deliverable (end Week 1):**
Login → upload CSV → list transactions + a simple monthly total.

---

### Week 2 — Categorization + Forecasts + Insights; CI/CD

**Goal:** auto-categorize, generate a basic forecast, expose insights, set up CI/CD.

1. **Categorization (Day 1–2)**

   * Rules table + endpoints (`/rules` CRUD).
   * Batch endpoint: `/categorize/run?month=YYYY-MM` updates `normalizedCategory`.
   * Return per-category spend summary (used later by budgets/reports).&#x20;

2. **Forecasts (Day 2–3)**

   * In **Forecasts & Insights** service, compute a **simple forecast**: for each category + net cash flow, use a moving average (last 3 months) or naive seasonal average; store a `forecastSnapshot` doc with timestamp. (Good enough for “predict next month’s spending/income”.)&#x20;

3. **Insights (Day 3)**

   * Generate documents like:

     * **Overspend**: predicted spend > budget.
     * **Anomaly**: category or merchant deviates > N% from average.
     * **Subscription candidate**: repeated monthly merchant.
   * Expose `/insights/latest` returning a concise list.&#x20;

4. **Reports (Day 3–4)**

   * `/reports/summary?month=` → totals, category breakdown, trend data.
   * `/reports/export.csv?month=` returns a CSV (simple string builder).

5. **CI/CD (Day 4–5)**

   * GitHub Actions CI on PR: checkout → setup-java 17 → `mvn -DskipTests package` for all services.
   * On tag `vX.Y.Z`: build & push Docker images (or build on EC2 if registry is too much).

**Deliverable (end Week 2):**
Upload CSV → categorize → forecast & insights available; basic CI/CD working.

---

### Week 3 — Budgets, Scheduling, AWS Deploy, Polish

**Goal:** budgets & scheduled refresh, deploy to EC2, docs & demo.

1. **Budgets (Day 1)**

   * Users set monthly budget per category in **Users & Budgets**.
   * Forecast service uses these to emit **overspend** insights.

2. **Scheduling (Day 1–2)**

   * Add `@Scheduled` jobs: nightly re-categorize last N days; weekly forecast snapshot; (optional) Plaid sandbox import.&#x20;

3. **AWS EC2 Deploy (Day 2–3)**

   * t2.micro Ubuntu, install Docker + Compose plugin.
   * Security Group: expose **only Gateway** (port 80).
   * Copy `.env` + `compose.prod.yml` (prod profiles, secrets).
   * **Option A:** pull images (if pushing from CI) then `docker compose -f compose.prod.yml up -d`.
   * **Option B:** build on EC2 (beginner path).

4. **Docs & Demo (Day 3–4)**

   * Root README: architecture diagram, env vars, local & prod quick start, **demo script**.
   * Per-service README: env vars, ports, endpoints.

5. **Polish (Day 4–5)**

   * Actuator health/info; consistent error JSON `{code,message,timestamp}`.
   * Light input validation; sensible defaults; graceful failures.

**Deliverable (end Week 3):**
System running on **EC2**; demo shows CSV → categorized → forecast/insights → reports, with budgets influencing overspend insight.

---

## 5) Minimal Endpoint Checklist

**Auth**

* `POST /auth/login` → `{jwt}`

**Users & Budgets (Postgres)**

* `GET/PUT /users/me` (profile, currency)
* `GET/POST/PUT/DELETE /budgets` (category, amount, month)

**Transactions & Import (Postgres)**

* `POST /transactions/import/csv` (multipart)
* `GET /transactions?month=`
* `GET /transactions/summary?month=`

**Categorization & Rules (Postgres)**

* `GET/POST/PUT/DELETE /rules`
* `POST /categorize/run?month=`

**Forecasts, Insights & Reports (Mongo)**

* `POST /forecast/run?month=` (also scheduled)
* `GET /forecast/latest`
* `GET /insights/latest`
* `GET /reports/summary?month=`
* `GET /reports/export.csv?month=`

**Gateway**

* Public entrypoint; forwards JWT to services.

---

## 6) Repo, Ports & Env (monorepo example)

```
smart-expense/
  gateway/
  auth-service/
  users-budgets-service/
  transactions-service/
  categorization-service/
  forecasts-insights-reports-service/
  docker-compose.yml
  compose.prod.yml
  README.md
```

**Ports (suggested):** Gateway 8080 • Auth 8081 • Users 8082 • Tx 8083 • Categorization 8084 • Forecasts/Insights/Reports 8085

**.env (example):**

```
JWT_SECRET=super-long-random
POSTGRES_URL=jdbc:postgresql://postgres:5432/finance
POSTGRES_USER=app
POSTGRES_PASSWORD=app
MONGO_URI=mongodb://app:app@mongo:27017
PLAID_CLIENT_ID=optional
PLAID_SECRET=optional
PLAID_ENV=sandbox
PROFILE_ACTIVE=prod|dev
```

---

## 7) Criteria Alignment (quick map)

* **Architecture**: 6 microservices with clear boundaries; Gateway routing; OpenAPI per service.
* **Security**: JWT on writes; roles; secrets via env vars.
* **Data**: Postgres for structured finance; Mongo for forecasts/insights/report caches.
* **Integrations**: CSV first; Plaid sandbox optional; **scheduling** for refresh/forecast.&#x20;
* **Resilience**: timeouts; graceful fallbacks; idempotent import jobs.
* **Build/Packaging**: Maven; multi-stage Docker; compose for local & prod.
* **CI/CD**: GitHub Actions build on PR; tag builds images and/or deploys.
* **Cloud**: EC2 with Docker; Gateway public, services internal.
* **Observability**: Actuator health/info; structured logs.
* **Docs**: Root & per-service READMEs; demo script; env var lists.

> Testing: keep it **light** (manual smoke + Postman collection). Focus on delivering a working system.

---

## 8) First Issues to Open (copy to your board)

* Bootstrap Gateway + route stubs
* Auth: `/auth/login` → JWT
* Users & Budgets: schema + CRUD
* Transactions: CSV import + summary
* Categorization: rules + batch run
* Forecasts/Insights: moving-average forecast + overspend/anomaly/subscription insights; reports summary/export
* Docker: local compose with Postgres & Mongo
* CI: GitHub Actions build; tag workflow
* EC2: compose.prod.yml + runbook
* Docs: Swagger links + demo script

---

### Demo Script (60–90 seconds)

1. Login → obtain JWT.
2. Upload CSV (last 3 months).
3. Run **categorize** → show category totals.
4. Set **budgets** → run **forecast** → open **insights** (overspend warning).
5. Hit **reports** summary + export CSV.

That’s your three-week, 2–3-person plan—tight, achievable, and aligned with the proposal’s goals of **expense tracking**, **cash-flow forecasting**, **insights**, and **reporting**, with optional bank API + scheduling to showcase growth.&#x20;
