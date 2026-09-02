# FloodWatch — Beginner-Friendly Blueprint & 14-Day Execution Guide

*A pragmatic MVP architecture for a junior dev team building localized flood risk estimation for Metro Manila.*

**Golden rule for this whole project: if you can't explain it to your teammate in one sentence, it's too complicated for a 2-week MVP.** Every section below is written with that filter applied.

---

## 1. Step-by-Step Data Flow (Explained Simply)

Think of FloodWatch as a **sandwich**: the frontend on top, the backend in the middle doing all the thinking, and two outside data sources feeding it ingredients.

```
┌─────────────────┐
│  React Native    │  1. Gets user's lat/lon (GPS or map pin drop)
│  (Vercel)        │  2. Calls YOUR backend: GET /risk?lat=..&lon=..
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  FastAPI Backend (Render)                     │
│                                                │
│  Step A: Look up static hazard data           │
│     → Query PostGIS: "which flood hazard      │
│       zone contains this point?"              │
│                                                │
│  Step B: Look up live weather data            │
│     → Call Open-Meteo API with the same       │
│       lat/lon, get rainfall forecast          │
│                                                │
│  Step C: Combine both with simple rules       │
│     → hazard_level + rainfall_mm → risk_level │
│                                                │
│  Step D: Return one clean JSON response       │
└────────┬───────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  React Native     │  3. Shows a colored pin/badge:
│                   │     🟢 Low  🟡 Medium  🟠 High  🔴 Severe
└─────────────────┘
```

### Why this design is beginner-friendly
- **One request in, one response out.** The frontend never talks to Open-Meteo or the database directly — it only ever talks to your FastAPI backend. This means only ONE thing can be wrong when debugging: "is my backend endpoint broken?"
- **Two data sources, but they never touch each other directly.** FastAPI is the only place they meet. This is called a "backend for frontend" pattern — a fancy name for "the backend does the annoying coordination work so the frontend can stay dumb and simple."
- **Static data (hazard zones) vs. dynamic data (weather) are handled differently on purpose.** Hazard zones almost never change — you load them into PostGIS *once* before the hackathon and query them. Weather changes every hour — you call Open-Meteo live, every request. Don't try to cache/store weather data unless you have spare time in week 2 (see Day 6 buffer).

### The actual request lifecycle, step by step
1. User opens the app. React Native asks for location permission and gets `{ lat: 14.65, lon: 121.03 }` (or the user taps a spot on the map).
2. React Native calls: `GET https://your-app.onrender.com/risk?lat=14.65&lon=121.03`
3. FastAPI receives it. It runs a PostGIS query: "does any polygon in `flood_hazard_zones` contain this point?" → returns `hazard_level = 2` (Medium) and a barangay name.
4. FastAPI calls Open-Meteo: `GET https://api.open-meteo.com/v1/forecast?latitude=14.65&longitude=121.03&hourly=precipitation&forecast_days=1` → returns rainfall numbers in mm.
5. FastAPI runs the rule-matrix (Section 3) on `hazard_level` + rainfall → `risk_level = "High"`.
6. FastAPI returns:
```json
{
  "lat": 14.65,
  "lon": 121.03,
  "barangay": "Barangay X",
  "hazard_level": 2,
  "rainfall_mm_next_3h": 22.4,
  "risk_level": "High",
  "risk_color": "#FF9800"
}
```
7. React Native renders a marker on the map with that color and a card with the details.

**Debugging tip for the team:** because each step returns clean, independent data, you can test steps A, B, and C with `curl` or Postman *before* the frontend is even ready. Never wait on the frontend team to test your backend logic.

---

## 2. Simple Database Schema & Setup

You only need **two tables** for the MVP. Resist the urge to add more — every extra table is extra debugging surface area during a 2-week sprint.

### 2.1 Enable PostGIS (run once per database)
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

### 2.2 Table 1 — `flood_hazard_zones` (the core table)
This holds the static Project NOAH hazard polygons. Each row is one geographic area with a hazard rating.

```sql
CREATE TABLE flood_hazard_zones (
    id             SERIAL PRIMARY KEY,
    barangay_name  VARCHAR(100),
    city_name      VARCHAR(100),
    hazard_level   INTEGER NOT NULL CHECK (hazard_level IN (1, 2, 3)),
    -- 1 = Low, 2 = Medium, 3 = High (matches Project NOAH's own scale)
    geom           GEOMETRY(POLYGON, 4326) NOT NULL
    -- 4326 = standard GPS coordinate system (lat/lon in degrees). Always use this
    -- unless you have a specific reason not to — it's what GPS and Open-Meteo use.
);

-- This index makes "is point X inside any polygon" queries fast.
-- Without it, PostGIS checks every row one by one — fine for a demo,
-- painfully slow once you load a full city's worth of polygons.
CREATE INDEX idx_flood_hazard_geom ON flood_hazard_zones USING GIST (geom);
```

**How do I actually get data into this table?**
Project NOAH publishes flood hazard shapefiles (`.shp`). The beginner-friendly path:
1. Download the shapefile for your target city/cities from Project NOAH's data portal.
2. Open it in **QGIS** (free) to inspect it and simplify if the polygons are huge/complex (`Vector → Geometry Tools → Simplify`). Simpler polygons = faster queries and smaller free-tier database usage.
3. Use `ogr2ogr` (comes with GDAL, or install via QGIS) to import it straight into Postgres:
```bash
ogr2ogr -f "PostgreSQL" PG:"host=YOUR_RENDER_HOST user=... dbname=... password=..." \
  hazard_data.shp -nln flood_hazard_zones -append \
  -nlt PROMOTE_TO_MULTI
```
   If `ogr2ogr` gives you MultiPolygon vs Polygon errors, it's easier to just change the column type to `GEOMETRY(MULTIPOLYGON, 4326)` than to fight the conversion — this is a very common beginner snag, don't lose a day on it.
4. **Fallback if shapefile wrangling eats too much time:** manually insert 15-20 known flood-prone barangays as rough rectangles using `ST_MakeEnvelope`. It's not scientifically precise, but it's enough for a working MVP demo, and you can always swap in real data later without changing any other code.
```sql
INSERT INTO flood_hazard_zones (barangay_name, city_name, hazard_level, geom)
VALUES (
  'Example Barangay', 'Marikina', 3,
  ST_SetSRID(ST_MakeEnvelope(121.09, 14.63, 121.11, 14.65), 4326)
);
```

### 2.3 Table 2 — `risk_check_logs` (optional, but strongly recommended)
This isn't for the algorithm — it's for **your own sanity while debugging**. Every time someone calls `/risk`, log what happened.

```sql
CREATE TABLE risk_check_logs (
    id              SERIAL PRIMARY KEY,
    lat             DOUBLE PRECISION,
    lon             DOUBLE PRECISION,
    hazard_level    INTEGER,
    rainfall_mm     DOUBLE PRECISION,
    risk_level      VARCHAR(20),
    created_at      TIMESTAMP DEFAULT NOW()
);
```
This one table turns "the app is showing weird results and we don't know why" into "let's look at the last 10 rows in `risk_check_logs` and see exactly what values went into the calculation." That's worth the 5 minutes it takes to set up.

### 2.4 The only spatial query you actually need
```sql
SELECT hazard_level, barangay_name
FROM flood_hazard_zones
WHERE ST_Contains(geom, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326))
LIMIT 1;
```
Plain English: *"Find the one polygon that contains this exact point, and give me its hazard level."*
- `ST_MakePoint(lon, lat)` — **note the order: longitude first, then latitude.** This trips up almost every beginner at least once. GPS coordinates are usually spoken as "lat, lon" but PostGIS functions want `(lon, lat)`.
- If nothing is found (point falls outside all mapped zones), treat it in code as `hazard_level = 1` (Low/Unknown) rather than crashing — Metro Manila coverage from Project NOAH won't be 100% complete everywhere.

---

## 3. Practical Risk Algorithm (Rule-Based, No ML Needed)

For a 2-week MVP, **do not use scikit-learn.** You don't have labeled historical flood incident data to train on, and a model trained on too little data will look impressive in a slide deck and behave randomly in a live demo. A transparent rule matrix is not a lesser choice here — it's the *correct engineering choice* for your constraints, and it's trivial to debug because you can explain any output in one sentence.

### 3.1 Step 1 — Bucket the rainfall
Use PAGASA's own public rainfall warning thresholds so your bands aren't arbitrary:

| Rainfall (mm in next 3 hours) | Rainfall Band |
|---|---|
| < 7.5 | Light |
| 7.5 – 15 | Moderate |
| 15 – 30 | Heavy |
| > 30 | Intense |

### 3.2 Step 2 — Combine hazard zone × rainfall band into final risk
This is just a lookup table (a "risk matrix") — the kind of thing disaster-risk agencies actually use in real life, simplified:

| Hazard Level ↓ / Rainfall → | Light | Moderate | Heavy | Intense |
|---|---|---|---|---|
| **1 – Low**    | Low    | Low      | Medium | High    |
| **2 – Medium** | Low    | Medium   | High   | Severe  |
| **3 – High**   | Medium | High     | Severe | Severe  |

### 3.3 Code it as a plain Python function
```python
def calculate_risk(hazard_level: int, rainfall_mm: float) -> str:
    """
    Turns static hazard data + live rainfall into a single risk word.
    hazard_level: 1 (Low), 2 (Medium), 3 (High) — from PostGIS lookup
    rainfall_mm: forecast rainfall in mm for next 3 hours — from Open-Meteo
    """
    # Step 1: bucket the rainfall
    if rainfall_mm < 7.5:
        band = "light"
    elif rainfall_mm < 15:
        band = "moderate"
    elif rainfall_mm < 30:
        band = "heavy"
    else:
        band = "intense"

    # Step 2: look up the matrix
    matrix = {
        1: {"light": "Low",    "moderate": "Low",    "heavy": "Medium", "intense": "High"},
        2: {"light": "Low",    "moderate": "Medium", "heavy": "High",   "intense": "Severe"},
        3: {"light": "Medium", "moderate": "High",    "heavy": "Severe", "intense": "Severe"},
    }
    return matrix.get(hazard_level, matrix[1])[band]
```
That's it. No training data, no model files, no `pickle` versioning headaches, and any teammate can read this function top to bottom in 30 seconds and know exactly why the app said "Severe."

### 3.4 Optional stretch goal (only if you finish early)
If you have spare time in Days 12-13 and want an ML flourish for the demo, the *safe* way to add it without risking the core deliverable: keep the rule-based function as the real logic, and separately train a tiny `scikit-learn` `LogisticRegression` on a handful of manually-labeled historical flood news reports (rainfall on that day → "did it flood, yes/no") purely as a bonus "confidence score" displayed alongside the rule-based risk — never let it replace the rule-based output as the source of truth.

---

## 4. Core FastAPI Code Skeleton

Project layout (keep it flat — no need for a complex folder structure at this scale):
```
backend/
├── main.py          # FastAPI app + routes
├── database.py       # DB connection setup
├── risk_logic.py      # the calculate_risk() function from Section 3
├── requirements.txt
└── .env               # DATABASE_URL, not committed to git
```

### `database.py`
```python
import os
import psycopg2
from psycopg2.extras import RealDictCursor

# Render gives you a DATABASE_URL connection string automatically —
# copy it from your Render Postgres dashboard into your .env file.
DATABASE_URL = os.environ["DATABASE_URL"]

def get_connection():
    """Opens a fresh connection. Simple and easy to reason about —
    a connection pool is a nice-to-have, not needed at MVP traffic levels."""
    return psycopg2.connect(DATABASE_URL, cursor_factory=RealDictCursor)
```

### `main.py`
```python
import httpx
from fastapi import FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

from database import get_connection
from risk_logic import calculate_risk

app = FastAPI(title="FloodWatch API")

# CORS: without this, your React Native web build (Vercel) or Expo web
# preview will be BLOCKED by the browser from calling this API.
# For MVP, "*" is fine — lock this down later if you have time.
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Pydantic model = defines exactly what shape of JSON we promise to return.
# This also gives you free auto-generated docs at /docs — use them to test!
class RiskResponse(BaseModel):
    lat: float
    lon: float
    barangay: str | None
    hazard_level: int
    rainfall_mm_next_3h: float
    risk_level: str


@app.get("/health")
def health_check():
    """Hit this first whenever something seems broken — if this fails,
    the problem is deployment/hosting, not your logic."""
    return {"status": "ok"}


@app.get("/risk", response_model=RiskResponse)
def get_flood_risk(
    lat: float = Query(..., ge=4.5, le=21.0, description="Latitude, must be within PH"),
    lon: float = Query(..., ge=116.0, le=127.0, description="Longitude, must be within PH"),
):
    """
    The one endpoint the whole app depends on.
    Steps: (A) look up hazard zone -> (B) get rainfall -> (C) combine -> (D) log -> return
    """

    # --- STEP A: look up hazard zone from PostGIS ---
    hazard_level = 1          # default to Low if no zone matches
    barangay_name = None
    try:
        conn = get_connection()
        with conn.cursor() as cur:
            cur.execute(
                """
                SELECT hazard_level, barangay_name
                FROM flood_hazard_zones
                WHERE ST_Contains(geom, ST_SetSRID(ST_MakePoint(%s, %s), 4326))
                LIMIT 1;
                """,
                (lon, lat),  # NOTE: lon first, then lat — matches ST_MakePoint order
            )
            row = cur.fetchone()
            if row:
                hazard_level = row["hazard_level"]
                barangay_name = row["barangay_name"]
        conn.close()
    except Exception as e:
        # Don't crash the whole request just because the DB hiccuped —
        # log it and fall back to a safe default so the demo doesn't break.
        print(f"[DB ERROR] {e}")

    # --- STEP B: get live rainfall from Open-Meteo ---
    rainfall_mm = 0.0
    try:
        with httpx.Client(timeout=5.0) as client:
            resp = client.get(
                "https://api.open-meteo.com/v1/forecast",
                params={
                    "latitude": lat,
                    "longitude": lon,
                    "hourly": "precipitation",
                    "forecast_days": 1,
                },
            )
            resp.raise_for_status()
            data = resp.json()
            # Sum the next 3 hourly precipitation values as a simple proxy
            # for "how much rain is coming soon"
            hourly_precip = data["hourly"]["precipitation"][:3]
            rainfall_mm = sum(hourly_precip)
    except Exception as e:
        print(f"[WEATHER API ERROR] {e}")
        # rainfall_mm stays 0.0 — worst case the app under-estimates risk,
        # which is safer than crashing entirely during a demo.

    # --- STEP C: combine into final risk ---
    risk_level = calculate_risk(hazard_level, rainfall_mm)

    # --- STEP D: log it for debugging (fire-and-forget, don't block the response on failure) ---
    try:
        conn = get_connection()
        with conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO risk_check_logs (lat, lon, hazard_level, rainfall_mm, risk_level)
                VALUES (%s, %s, %s, %s, %s);
                """,
                (lat, lon, hazard_level, rainfall_mm, risk_level),
            )
        conn.commit()
        conn.close()
    except Exception as e:
        print(f"[LOG ERROR] {e}")

    return RiskResponse(
        lat=lat,
        lon=lon,
        barangay=barangay_name,
        hazard_level=hazard_level,
        rainfall_mm_next_3h=round(rainfall_mm, 1),
        risk_level=risk_level,
    )
```

### `requirements.txt`
```
fastapi
uvicorn[standard]
psycopg2-binary
httpx
pydantic
```

**How to run it locally:** `uvicorn main:app --reload` then open `http://localhost:8000/docs` — FastAPI auto-generates an interactive test page. This is the single most useful thing for a junior team: you can test `/risk` with real coordinates before React Native even exists.

---

## 5. 14-Day Learning & Building Roadmap

Assumes a small team (~3-4 people) who can roughly split into "backend/data" and "frontend" but should **pair up and rotate** so everyone understands the whole system by the end — that's better for both learning and for not being blocked when one person is stuck.

Each day is a half-day of focused work + buffer — don't plan 8 solid hours of new material every day, juniors burn out fast when every day is 100% new concepts.

| Day | Focus | Concrete Milestone (you should be able to demo/show this) |
|---|---|---|
| **1** | Setup & accounts | Render account + free Postgres instance created, Vercel account created, GitHub repo with `backend/` and `frontend/` folders, React Native project scaffolded (Expo recommended for beginners — much less native config pain) |
| **2** | PostGIS + data loading | `CREATE EXTENSION postgis;` run successfully, `flood_hazard_zones` table created, at least 10-15 real or manually-drawn hazard polygons loaded, can run the `ST_Contains` query manually in a SQL client and get a sensible result |
| **3** | FastAPI skeleton | `/health` and `/risk` endpoints exist locally, `/risk` successfully queries the DB and returns hazard_level for a hardcoded lat/lon, backend deployed to Render (even if incomplete) so the team gets comfortable with the deploy pipeline early |
| **4** | Weather integration | `/risk` endpoint successfully calls Open-Meteo and returns real rainfall numbers for a test coordinate — test this in isolation with `/docs` before touching anything else |
| **5** | Risk algorithm | `calculate_risk()` function written and unit-tested with at least 6 manual test cases (one per matrix cell edge case), wired into `/risk`, full end-to-end response verified via `/docs` |
| **6** | **Buffer / catch-up day** | No new features — finish anything from Days 1-5 that slipped. If you're on schedule, use this day to write 3-5 automated tests with `pytest` and improve error handling |
| **7** | RN scaffold + maps | React Native app requests location permission and shows a map (Google Maps / react-native-maps) centered on the user's location — nothing else yet, just get the map on screen |
| **8** | Connect frontend to backend | App calls your deployed `/risk` endpoint with the user's real coordinates and can log/console.print the JSON response — this is the "wow it's actually one system now" milestone |
| **9** | Risk UI | Map shows a colored pin (green/yellow/orange/red) at the user's location based on `risk_level`, plus a simple card/bottom-sheet showing barangay name, rainfall, and risk level in plain text |
| **10** | Error & loading states | App shows a spinner while waiting on `/risk`, shows a friendly message if the request fails or times out, and handles "location outside Metro Manila" gracefully instead of crashing |
| **11** | Full integration pass | Deployed frontend (Vercel) talking to deployed backend (Render) talking to deployed DB — test the *entire* live chain, not localhost, at least 5 different real Metro Manila coordinates |
| **12** | Bug bash | Whole team tests the live app together, files bugs as GitHub issues, fixes the top 5 most visible/embarrassing ones first (visual bugs and crashes beat edge-case logic bugs in priority right now) |
| **13** | Polish + docs | README with setup instructions, `risk_check_logs` checked to confirm the app has been generating sensible data, basic loading UX polish, remove/hide obvious placeholder text |
| **14** | Demo prep + final buffer | Rehearse the demo flow (2-3 real coordinates you *know* produce interesting, different risk levels), prepare a 1-slide "what's out of scope / what's next" summary, freeze the code — no new changes after midday |

### A few coaching notes for the team
- **Don't let PostGIS data-loading (Day 2) eat more than one day.** It's the task most likely to blow the schedule for a junior team because shapefile tooling is fiddly. If it's not working by end of Day 2, switch to the manual-rectangle fallback in Section 2.2 and move on — you can always swap in real Project NOAH polygons later without touching any other code, since your API contract (`hazard_level`, `barangay_name`) stays identical either way.
- **Deploy early and often**, starting Day 3, not Day 13. A backend that's "done" locally but has never been deployed is a common source of last-minute panic. Free-tier Render services also "sleep" after inactivity — deploying early lets you discover and get used to that cold-start delay well before demo day.
- **Git workflow:** keep it simple — `main` branch always deployable, everyone works on short-lived feature branches, one teammate reviews before merging. Don't set up anything fancier (no need for `develop`/`staging` branches at this scale).
- **When something breaks, check in this order:** (1) `/health` — is the backend even alive? (2) `/docs` — does `/risk` work when called directly, bypassing the frontend entirely? (3) `risk_check_logs` table — what values actually went into the last few calculations? Only after ruling those out should you suspect the frontend code.
