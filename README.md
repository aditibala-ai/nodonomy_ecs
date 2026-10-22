# Nodonomy — full software pipeline

Raw telemetry → Database & MQTT module → Backend (Decision Engine + REST API + auth) → Frontend dashboard.

This repo does NOT include real hardware, LoRa, or Wi-SUN — that's simulated. Everything else
(validation, storage, business logic, alerts, auth, dashboard) is real, working code you run locally.

```
nodonomy/
├── backend/          FastAPI app: DB & MQTT ingestion + Decision Engine + REST API + auth
├── simulator/         Fakes the hardware layer — live random data OR a fixed sample dataset
└── frontend/          Single-file dashboard (open directly in a browser)
```

## 0. Prerequisites

- Python 3.10+
- PostgreSQL running locally (or use the SQLite shortcut below — zero setup)

## 1. Set up the database

**Option A — PostgreSQL (matches the spec):**
```bash
# macOS: brew install postgresql && brew services start postgresql
# Linux:  sudo apt install postgresql && sudo service postgresql start

psql postgres -c "CREATE USER nodonomy WITH PASSWORD 'nodonomy';"
psql postgres -c "CREATE DATABASE nodonomy OWNER nodonomy;"
```

**Option B — SQLite (fastest way to just see it running):**
Skip the above. In step 2, set `DATABASE_URL=sqlite:///./nodonomy.db` in your `.env` instead.

## 2. Run the backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# Edit .env: set DATABASE_URL (Postgres or sqlite), and change JWT_SECRET to a real random value

uvicorn app.main:app --reload --port 8000
```

Leave this running. Visit `http://localhost:8000/docs` — FastAPI's auto-generated
interactive API docs, useful for testing endpoints directly before the frontend is wired up.

Tables are created automatically on first startup.

## 3. Create a login user

In a second terminal:
```bash
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "kruthika", "password": "yourpassword123"}'
```

## 4. Feed it data — two ways

**Fixed sample dataset (recommended for a demo/grading run — predictable, finite):**
```bash
cd simulator
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
python load_sample_data.py
```
This loads 3 nodes with a deliberate intrusion event, a low-battery event, and two low-priority
events — see `sample_dataset.json` for exactly what it sends and why.

**Live random simulator (for an ongoing, ever-changing demo):**
```bash
cd simulator
python simulate_nodes.py --interval 5
```
Runs forever, publishing new fake readings every 5 seconds for 3 nodes, until you Ctrl+C it.

Both scripts talk to the backend over plain HTTP (`POST /telemetry/ingest`) so you don't need a
real MQTT broker to see the pipeline work end to end. `simulate_nodes.py --mode mqtt` is also
included if you install Mosquitto locally and want to test the real MQTT path — see below.

## 5. Open the dashboard

Just open `frontend/index.html` directly in your browser (double-click it, or `open frontend/index.html`).
Log in with the username/password you registered in step 3.

If your browser blocks `fetch` from a `file://` page, serve it instead:
```bash
cd frontend
python -m http.server 5500
# then visit http://localhost:5500
```

## Optional: run the real MQTT path instead of the HTTP shortcut

The simulator posting straight to `/telemetry/ingest` is a stand-in for "gateway publishes to
MQTT broker → Database & MQTT module subscribes and stores it" — same validation, same storage,
same node status logic, just skipping the broker hop. If you want the real MQTT hop too:

```bash
# Install Mosquitto: brew install mosquitto  /  sudo apt install mosquitto
mosquitto -v                                    # terminal 1: run the broker

cd backend && source venv/bin/activate
python -m app.mqtt_ingest                       # terminal 2: run the subscriber

cd simulator && source venv/bin/activate
python simulate_nodes.py --mode mqtt --interval 5   # terminal 3: publish over MQTT
```

Note: packets ingested this way are validated and stored, but the Decision Engine currently
runs from `/telemetry/ingest` in the API layer (see `routers/telemetry.py`). If you use the
pure-MQTT path, call the Decision Engine from `mqtt_ingest.py`'s `_on_message` the same way —
that's a good first "read every line" exercise once we go through the code together.

## Security implemented

- Passwords hashed with bcrypt — never stored or returned in plaintext
- JWT bearer tokens on every protected route (`nodes`, `telemetry`, `alerts`, `system`)
- Pydantic validates every incoming packet's structure and value ranges before it touches the DB
- HMAC-SHA256 checksum per packet — the ingestion module recomputes and rejects mismatches, so a
  tampered or malformed packet never gets stored (see `security.py` and `sample_dataset.json`/
  `simulate_nodes.py` for how the checksum is generated on the sending side)
- Centralized error handling — a bad request or server error never crashes the app or leaks a
  stack trace to the client

## What's simulated vs. real

| Layer | Status |
|---|---|
| Sensors, STM8, LoRa, ESP32, Wi-SUN | Simulated (`simulator/`) |
| MQTT broker | Optional real Mosquitto, or skipped via the HTTP shortcut |
| Database & MQTT module (validation + storage) | **Real** (`backend/app/mqtt_ingest.py`) |
| Backend (Decision Engine, REST API, auth) | **Real** (`backend/app/`) |
| Frontend dashboard | **Real** (`frontend/index.html`) |
