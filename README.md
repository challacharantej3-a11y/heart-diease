# Heart Health Guide

A small, multi-page health education website with an integrated screening
tool. It replaces the original single-page Streamlit script with a proper
**frontend / backend split**, plus five educational pages that give the
screening tool context, sources, and clear safety guidance.

```
heart-disease-predictor/
├── backend/                          # Flask REST API + model artifacts
│   ├── app.py                         # API server (loads the .pkl files, exposes /api/predict)
│   ├── requirements.txt
│   ├── knn_heart_model.pkl
│   ├── heart_scaler.pkl
│   └── heart_columns.pkl
├── frontend/                          # Static HTML/CSS/JS site (no build step, no framework)
│   ├── index.html                      # Home: site purpose, disclaimers, topic navigation
│   ├── understanding-heart-disease.html
│   ├── risk-factors.html
│   ├── warning-signs.html              # Emergency guidance + symptom differences by sex
│   ├── prevention.html
│   ├── risk-check.html                 # The screening tool (was the old index.html)
│   ├── resources.html                  # Finding care, emergency guidance, cited sources
│   ├── style.css                       # Shared design system for all pages
│   └── script.js                       # Shared nav logic + screening-tool logic
├── legacy-streamlit-app/              # Original app1.py, kept for reference
│   └── app1.py
└── README.md
```

## About the content

Every page other than the tool itself is educational: it explains heart
disease, risk factors, warning signs, and prevention in plain language, and
is explicit that it does not diagnose anything. Statistics and
recommendations are paraphrased from the CDC, the American Heart
Association, and the World Health Organization — see `resources.html` for
the full source list. A persistent disclaimer bar appears on every page, and
an emergency-guidance box appears on the warning-signs and risk-check pages
specifically, since those are the pages most likely to be visited by someone
with active symptoms or acute concern.

## What was confirmed working

- `backend/app.py` was run locally and tested against:
  - `GET /api/health` → `{"status": "ok"}`
  - `POST /api/predict` with a low-risk profile → `{"prediction": 0, "risk": "Low"}`
  - `POST /api/predict` with a high-risk profile → `{"prediction": 1, "risk": "High"}`
  - Missing-field and invalid-value payloads → `400` with a descriptive `error` message
  - CORS response headers, confirming the API can be called from a different
    origin if you ever split frontend/backend across two hosts
- Every page in `frontend/` (`index.html`, `understanding-heart-disease.html`,
  `risk-factors.html`, `warning-signs.html`, `prevention.html`,
  `risk-check.html`, `resources.html`, plus `style.css` and `script.js`) was
  requested from the running Flask server and confirmed to return `200 OK`,
  confirming Flask serves the whole site correctly on one port.
- Every internal link across all seven pages was checked against the actual
  filenames on disk to confirm there are no broken links.
- The prediction logic (raw input → one-hot fields → fill missing expected
  columns → reorder → scale → predict) is a direct port of the original
  `app1.py` logic — same feature engineering, same model files, unchanged.

**Not yet tested (worth doing before production use):** load testing,
HTTPS/reverse-proxy configuration, and running under a production WSGI
server (gunicorn/uwsgi) rather than Flask's built-in dev server.

## Run it locally (single command, single port)

The backend serves the frontend itself, so you only need one process:

```bash
cd backend
pip install -r requirements.txt
python app.py
```

Open **http://localhost:5000** — the form and the API are both served from
here. No separate frontend server, no build step, no bundler.

By default it runs on port 5000. To use a different port:

```bash
PORT=8000 python app.py
```

## Run it as two separate services (optional)

If you'd rather host the frontend on a static host (Netlify, S3, nginx) and
the backend on its own server:

1. Deploy `backend/` (with its `requirements.txt`) to your Python host and
   run `python app.py`.
2. Deploy the contents of `frontend/` to your static host.
3. In `frontend/script.js`, set `API_BASE` to your backend's full URL, e.g.:
   ```js
   const API_BASE = "https://your-backend.example.com";
   ```
   The backend already sends permissive CORS headers, so this works out of
   the box — no backend changes needed.

## API reference

### `POST /api/predict`

Request body (all fields required):

```json
{
  "age": 52,
  "sex": "M",
  "chest_pain": "ATA",
  "resting_bp": 130,
  "cholesterol": 220,
  "fasting_bs": 0,
  "resting_ecg": "Normal",
  "max_hr": 165,
  "exercise_angina": "N",
  "oldpeak": 1.2,
  "st_slope": "Up"
}
```

Response:

```json
{ "prediction": 0, "risk": "Low" }
```

or, on bad input:

```json
{ "error": "Field 'sex' must be one of ('M', 'F')" }
```

### `GET /api/health`

Returns `{"status": "ok"}`. Useful for uptime checks / load balancer probes.

## Notes and open questions

- The `.pkl` files were trained with an older scikit-learn version than the
  one used to test this package here (1.6.1 vs the environment's 1.8.0).
  Predictions ran correctly despite scikit-learn's version-mismatch warning,
  but if you hit different results on your machine, re-export the model with
  your installed scikit-learn version as a first troubleshooting step.
- The Flask dev server is fine for local use and demos. For a real
  deployment, put it behind gunicorn/uwsgi and a reverse proxy (nginx) —
  this wasn't set up or tested here, since it depends on your hosting
  environment.
- This tool is a screening aid built on a small public dataset, not a
  medical device. The UI copy reflects that; keep it that way if you extend
  the app.
