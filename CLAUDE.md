## Goal & scope
This is project is personal **economic analytics platform** that tracks weekly macroeconomic indicators and major market indices and presents them on an interactive web page. Three feature areas:
 
1. **Descriptive analytics** — visualize interest rates, inflation, unemployment on a international level and stock indices over time.
2. **Predictive modeling** — forecasts on the weekly series.
3. **Market sentiment** — sentiment scored from news/social text, aligned to the series.
It's a personal project that also serves as a full stack learning iniative from data storage to web rendering.
 
**In scope:** weekly data ingestion + storage, descriptive charts, then forecasting
and sentiment, deployed as a web app.

## Phasing

- **Phase 1 — Descriptive (MVP):** FRED macro data → Postgres → export JSON to Supabase Storage → React line charts. One series first (e.g. Fed Funds Rate), then expand.
- **Phase 2 — Expand descriptive:** Add more FRED series + `yfinance` stock indices. Add date filtering, multi-series charts.
- **Phase 3 — Forecasting:** ARIMA/SARIMA via `statsmodels` on one series. Store forecasts in DB. Overlay on chart. Upgrade to Prophet only once ARIMA is understood.
- **Phase 4 — Sentiment:** VADER on NewsAPI first. Wire the pipeline, then upgrade model quality with FinBERT if needed.

## Architecture
Two independently deployable halves:
 
- **`backend/`** (Python) — *produces* data: ingest → store → analyze →
  forecast → score sentiment → export.
- **`frontend/`** (React + Vite) — *consumes* data and does all filtering/charting
  client-side.

**Data flow:** ingest from APIs → store in Postgres → analyze / forecast / sentiment →
export static JSON to Supabase Storage → frontend fetches and renders. No custom backend
(FastAPI) until on-demand computation is needed.
 
The only contact point between the two halves is the JSON data handoff (Supabase Storage
public URL), which keeps them independently deployable.

## Database schema 

```sql
CREATE TABLE series (
  id        SERIAL PRIMARY KEY,
  code      TEXT UNIQUE NOT NULL,  -- e.g. 'FEDFUNDS', 'SP500'
  name      TEXT NOT NULL,
  source    TEXT NOT NULL,         -- 'fred', 'yfinance'
  frequency TEXT NOT NULL,
  unit      TEXT                   -- '%', 'index', 'thousands'
);

CREATE TABLE observations (
  id        BIGSERIAL PRIMARY KEY,
  series_id INT REFERENCES series(id),
  date      DATE NOT NULL,
  value     NUMERIC,
  UNIQUE (series_id, date)
);

CREATE TABLE forecasts (
  id            BIGSERIAL PRIMARY KEY,
  series_id     INT REFERENCES series(id),
  run_date      DATE NOT NULL,
  forecast_date DATE NOT NULL,
  value         NUMERIC,
  model         TEXT
);
```

## Tech stack
- **Storage:** PostgreSQL — Supabase (managed, free tier). Static JSON exports stored in Supabase Storage (free 1GB).
- **Data sources:**
  - Phase 1–2: FRED (`fredapi`) for macro; `yfinance` (no API key, free) for stock indices.
  - Phase 4 only: NewsAPI for sentiment text. Reddit / GDELT as stretch options.
- **Analytics / ML:**
  - Forecasting: `statsmodels` ARIMA/SARIMA and scikit-learn (Phase 3 default); Prophet as a later upgrade once ARIMA is understood.
  - Sentiment: VADER (Phase 4 default); FinBERT (`ProsusAI/finbert`) as a later upgrade.
- **Frontend:** React + Vite, TailwindCSS, Recharts, Axios; `supabase-js` if reading the DB directly.
- **Deploy:** Vercel (frontend); GitHub Actions (weekly ETL); Cloud Run / Railway only if a backend is added later.

## Key decisions & conventions
- **Weekly cadence** for all data — deliberate, to keep storage and payloads small. Don't
  switch to daily/intraday without a real reason.
- **Secrets are server-side only** — API keys and the Postgres `DATABASE_URL` live in
  `.env` / GitHub Actions secrets, never in frontend code. Only the Supabase anon key
  (guarded by RLS) may ship to the browser.
- **Frontend data default is static JSON** served from Supabase Storage (CDN-cached, free).
  Direct `supabase-js` reads are an option when live queries are needed.
- **Schema first** — design and create DB tables before writing any pipeline code.
- **One source at a time** — get FRED fully working before adding `yfinance`. Validate each
  ingestion layer before layering the next.
