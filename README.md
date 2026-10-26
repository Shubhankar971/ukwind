# UK Wind Forecast Decision Dashboard

A Dockerized React + FastAPI sample application for reviewing GB wind generation forecasts and evaluating them against actual generation from Elexon BMRS.

## Features
- Dashboard of 30-minute wind generation intervals.
- Latest forecast snapshot available before a configurable decision cutoff.
- Forecast-vs-actual comparison and error metrics (MAE, RMSE, bias, WAPE).
- CSV export.
- Elexon API integration with a demo fallback so the UI runs without API access.
- Explicit forecast publication cutoff to avoid look-ahead bias.

## Data and assumptions
- Actuals: Elexon FUELHH dataset, filtered to `fuelType=WIND`.
- Forecasts: Elexon WINDFOR dataset.
- A forecast is eligible only if `publishTime <= decision_cutoff`; for each target `startTime`, the newest eligible publication is selected.
- Forecast evaluation joins on the target `startTime` (30-minute settlement period).
- Accuracy metrics are calculated only where both forecast and actual values exist.
- `WAPE = sum(abs(error)) / sum(actual) * 100`; omitted when the sum of actual generation is zero.
- Wind generation is not total demand. This sample does not calculate procurement volume because consumer demand, other generation, interconnector flows, balancing actions, and market positions are out of scope.
- Elexon API schemas and access behaviour can change. Verify endpoint parameters against the current official documentation before production use.

## Run locally with Docker
```bash
docker compose up --build
```
Open:
- Frontend: http://localhost:3000
- API docs: http://localhost:8000/docs
- Health: http://localhost:8000/api/health

## Run without Docker
Backend:
```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```
Frontend:
```bash
cd frontend
npm install
npm run dev
```
Set `VITE_API_BASE_URL=http://localhost:8000` in `frontend/.env.local` if needed.

## Configure Elexon
The backend attempts to query the official stream endpoints:
- `https://data.elexon.co.uk/bmrs/api/v1/datasets/FUELHH/stream`
- `https://data.elexon.co.uk/bmrs/api/v1/datasets/WINDFOR/stream`

Set:
- `ELEXON_BASE_URL` (optional; defaults to `https://data.elexon.co.uk/bmrs/api/v1`)
- `ELEXON_TIMEOUT_SECONDS` (optional; defaults to `20`)
- `DEMO_MODE=true` to always use illustrative data.

The implementation sends date bounds as `from` and `to` query parameters and filters the actuals to `fuelType=WIND`. Check the current Elexon endpoint docs for the supported parameter names and required date format. If the API rejects the request or returns no records, the app shows a clear API error / demo data fallback rather than silently presenting demo records as live data.

## Deploy
### Frontend on Vercel
1. Push this project to a GitHub repository.
2. Import the repository in Vercel.
3. Set **Root Directory** to `frontend`.
4. Add environment variable `VITE_API_BASE_URL=https://YOUR-BACKEND-HOST`.
5. Deploy.

A `frontend/vercel.json` is included for SPA routing.

### Backend on Render / Heroku-compatible container hosting
The backend is a Docker container and listens on `0.0.0.0:$PORT` (defaults to 8000). Deploy the repository's root `Dockerfile` to a container host, or deploy `backend/` with Python buildpacks and start command:
```bash
uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}
```
Set `CORS_ORIGINS` to the deployed frontend origin, e.g. `https://your-app.vercel.app`.
Heroku may require a paid plan and account/billing setup. I cannot claim this sample is deployed: deployment requires access to your Vercel/hosting account and a public repository. The code is deployment-ready, but the live URL must be created in your account.

## AI tools disclosure
**AI assistance used:** OpenAI ChatGPT was used to assist with the initial application scaffold, UI/API implementation, documentation, and Docker configuration. Review, test, and validate the code and live API behaviour before using it for operational or trading decisions.

## Security and production checklist
- Configure restrictive CORS origins.
- Add request validation, logging, retries, caching, and monitoring.
- Confirm Elexon API licensing/terms and current endpoint parameters.
- Add automated tests and integration tests against a known response fixture.
- Consider timezone handling explicitly (API timestamps are expected to be timezone-aware).
- Never use demo/fallback values for real procurement decisions.
- This is an illustrative decision-support prototype, not a trading system.
