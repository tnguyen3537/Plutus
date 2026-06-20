## Goal & scope
This is project is personal **economic analytics platform** that tracks weekly macroeconomic indicators and major market indices and presents them on an interactive web page. Three feature areas:
 
1. **Descriptive analytics** — visualize interest rates, inflation, unemployment on a international level and stock indices over time.
2. **Predictive modeling** — forecasts on the weekly series.
3. **Market sentiment** — sentiment scored from news/social text, aligned to the series.
It's a personal project that also serves as a full stack learning iniative from data storage to web rendering.
 
**In scope:** weekly data ingestion + storage, descriptive charts, then forecasting
and sentiment, deployed as a web app.

## Architecture
Two independently deployable halves:
 
- **`pipeline/`** (Python) — *produces* data: ingest → store → analyze →
  forecast → score sentiment → export.
- **`frontend/`** (React + Vite) — *consumes* data and does all filtering/charting
  client-side.
**Data flow:** ingest from APIs → store in Postgres → analyze / forecast / sentiment →
export static JSON (default) *or* serve directly via `supabase-js` + Row Level Security →
frontend renders. No custom backend (FastAPI) until on-demand computation is needed.
 
The only contact point between the two halves is the data handoff (exported JSON, or
Supabase reads), which keeps them independently deployable.
 
## Tech stack (please revise if needed)
- **Storage:** PostgreSQL — Supabase (managed, free tier).
- **Data sources:** FRED (`fredapi`) for macro; Twelve Data (`twelvedata`) for indices;
  Alpha Vantage NEWS_SENTIMENT / NewsAPI / Reddit / GDELT for sentiment text.
- **Analytics / ML:** pandas, NumPy; Prophet or scikit-learn (forecasting);
  FinBERT (`ProsusAI/finbert`) or VADER (sentiment).
- **Frontend:** React + Vite, TailwindCSS, Recharts or Plotly.js, Axios; `supabase-js` if
  reading the DB directly.
- **Deploy:** Vercel (frontend); GitHub Actions (weekly ETL); Cloud Run / Railway only if a
  backend is added later.

## Key decisions & conventions
- **Weekly cadence** for all data — deliberate, to keep storage and payloads small. Don't
  switch to daily/intraday without a real reason.
- **Secrets are server-side only** — API keys and the Postgres `DATABASE_URL` live in
  `.env` / GitHub Actions secrets, never in frontend code. Only the Supabase anon key
  (guarded by RLS) may ship to the browser.
- **Frontend data default is static JSON** (CDN-cached, free); direct `supabase-js` reads
  are an option when live queries are wanted.