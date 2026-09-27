# Poetry App — Runnable Starter Project

A minimal but functional poetry app: write, save, and browse poems, with a
2–3 line preview card on profiles. Backend is FastAPI + PostgreSQL, frontend
is Flutter.

```
poetry_app/
├── docker-compose.yml     # Postgres [5432] + backend, one command to run the API
├── backend/               # FastAPI app (Python)
└── frontend/              # Flutter app (Dart)
```

---

## 1. Run the backend (fastest way: Docker)

Requires Docker + Docker Compose installed.

```bash
cd poetry_app
docker compose up --build
```

This starts:
- Postgres on `localhost:5432` (user/pass: `postgres`/`postgres`, db: `poetry_db`)
- FastAPI on `http://localhost:8000`

Tables are auto-created on startup (see `backend/app/database.py`). Once
running, open the interactive API docs at:

**http://localhost:8000/docs**

You can register a user and create poems directly from that page to sanity
check the API before touching Flutter at all.

### Running the backend without Docker

```bash
cd backend
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Make sure Postgres is running locally and create the DB:
#   createdb poetry_db
cp .env.example .env       # edit DATABASE_URL/SECRET_KEY if needed
export $(cat .env | xargs) # or use a tool like `direnv`/`python-dotenv`

uvicorn app.main:app --reload
```

---

## 2. Run the Flutter frontend

Requires the Flutter SDK installed (`flutter doctor` should pass).

```bash
cd frontend
flutter pub get
flutter run
```

Before running, check `frontend/lib/services/api_service.dart` — the
`baseUrl` needs to match how your emulator/device reaches the backend
(details in `frontend/README.md`). The default (`http://10.0.2.2:8000`) is
correct for the Android emulator.

---

## 3. Try it out

1. Register a new account in the app (or via `/docs`).
2. Log in.
3. Tap **+** on the profile screen and write a poem.
4. Go back to the feed — if the poem is public, it shows up there too, as a
   card with just the first 2–3 lines.
5. Tap a card to read the full poem.

---

## Where to look first when reviewing

| Concern | File |
|---|---|
| DB schema | `backend/app/models.py` |
| Preview truncation logic | `backend/app/preview.py` |
| API routes | `backend/app/routers/*.py` |
| Auth (JWT) | `backend/app/auth.py` |
| Preview card UI | `frontend/lib/widgets/poem_card.dart` |
| API client | `frontend/lib/services/api_service.dart` |
| App state | `frontend/lib/providers/*.dart` |

## Known limitations (intentional, for a first pass)

- No image/scan upload yet (mentioned as a future enhancement in the design doc).
- No refresh tokens — access token just expires after an hour (config in `.env`).
- No pagination UI in Flutter yet (backend already supports `page`/`page_size`).
- Passwords/JWT secret in `.env.example` are dev-only — replace before deploying.
