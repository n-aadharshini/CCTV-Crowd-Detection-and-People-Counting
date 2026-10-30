
# StoreCount

StoreCount analyzes a store entrance video, tracks detected people, counts qualified crossings through a configurable door line, and exports an annotated video, events, visits, and a one-page report. The web interface supports uploads, progress updates, seeking and event review, ground-truth evaluation, and downloadable artifacts.

**Live demo:** not deployed yet. **Screenshots:** add captures from a deployed instance here. No store footage is bundled. Use only clips you have permission to process and redistribute.

## Features

- Full-frame person detection with Ultralytics YOLO11s, ByteTrack IDs, configurable confidence and inference size, and labeled rectangles. The first ten seconds use a lower-resolution preview pass before analysis continues at configured quality.
- Per-frame visible crowd count and peak crowd size, kept separate from estimated inside occupancy and doorway IN/OUT crossings.
- CrowdVision-style annotated video HUD with tracked movement trails, general-floor and queue zones, threshold alerts, and live processing FPS.
- Bottom-center foot point, line buffer zones, debounce, and hysteresis. Tracks first seen inside establish a baseline instead of being counted as an entry.
- Nonnegative headcount with low-confidence reasons for short tracks and clamped exits; visit dwell and still-inside rows.
- Two calibrated billing-station regions identify the opening staff tracks and build appearance references; likely tracker ID switches are matched back to those workers. Staff boxes and their separate count remain visible, while confirmed staff are excluded from customer occupancy and IN/OUT events.
- Assignment-format `events.csv` has exactly four columns; detailed event analytics and visits are separate files, with automatic format validation.
- Staff identity is seeded by sustained presence in the two configured billing stations, then uses conservative appearance matching to follow track ID changes. It does not label every person who briefly enters a billing region as staff. Adjust the staff polygons and `staff_appearance_threshold` to the camera view.
- Optional color-histogram gallery for re-entry matching near the door. No OSNet model is required.
- Auto line suggestion from the first up to 120 seconds: DBSCAN track endpoint clustering, average motion around the candidate region, and time spent on either side. Always review or edit the suggested line.
- Per-direction precision/recall/F1 against a supplied `timestamp,event` CSV. No ground truth means no accuracy claims.
- MP4 annotation, synced count timeline, events/visits CSV, JSON, metrics, Markdown/PDF note, and zip download.
- Single-worker job queue, SSE progress, live counters, cancellation, upload/duration validation, byte-range video responses, and one-hour cleanup.

## Run locally (Windows PowerShell)

Run commands from the project root folder (the folder containing this README). StoreCount needs Python 3.11, Node.js 20 or later, and npm. FFmpeg is optional: it enables H.264 output when installed; otherwise the app uses its configured OpenCV video codec.

### 1. Install dependencies

```powershell
cd "C:\path\to\MiBOT_AI"
if (!(Test-Path .venv)) { py -3.11 -m venv .venv }
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
cd frontend
npm ci
cd ..
```

If Python 3.11 is not available through `py -3.11`, install Python 3.11 and enable the Python Launcher, or create `.venv` with your installed Python version. Do not run these commands from inside `backend` or `frontend` except for the `npm ci` step.

### 2. Start the backend

Open a PowerShell terminal at the project root and run:

```powershell
.\.venv\Scripts\python.exe -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
```

Keep this terminal open. The backend API is at `http://127.0.0.1:8000`.

### 3. Start the web interface

Open a second PowerShell terminal at the project root:

```powershell
cd frontend
npm run dev
```

Open the local Vite URL printed in that terminal (normally `http://localhost:5173`). Vite forwards API and WebSocket requests to the backend on port 8000. Use one backend process; if the terminal reports that port 8000 is already in use, the backend is already running or another process owns that port. Reuse the running server, stop it with **Ctrl+C** in its terminal, or choose another port and set `VITE_API_URL` before starting Vite, for example:

```powershell
# Backend terminal
.\.venv\Scripts\python.exe -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8001

# Frontend terminal
$env:VITE_API_URL = "http://127.0.0.1:8001"
cd frontend
npm run dev
```

### Optional: serve the built UI and API from one local URL

```powershell
cd frontend
npm run build
cd ..
.\.venv\Scripts\python.exe -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
```

Then open `http://127.0.0.1:8000`. Rebuild the frontend after changing its source files.

### Models and video analysis

The project uses Ultralytics YOLO11 with ByteTrack. `config.yaml` sets `model: auto`: the app selects YOLO11s when CUDA is available and YOLO11n on CPU. The repository includes `yolo11n.pt`, `yolo11s.pt`, and `yolov8n.pt` weights, so the default model does not need to be downloaded on first run. The detector warms up in the background when the backend starts; CPU inference and the first model initialization can take time. To choose a specific model, change `model` in `config.yaml` to `yolo11n.pt`, `yolo11s.pt`, or another supported local model path.

Upload a permitted video from the web interface. The server enforces the file-size and duration limits in `config.yaml`; processed job files are temporary and expire after the configured retention period. The frame-by-frame pipeline can also be run without the web UI:

```powershell
# From the project root
.\.venv\Scripts\python.exe -m backend.pipeline.cli path\to\store.mp4 --output output --config config.yaml
.\.venv\Scripts\python.exe -m backend.pipeline.cli path\to\store.mp4 --output output --ground-truth truth.csv --tolerance 2
```

The pipeline writes `annotated.mp4`, strict four-column `events.csv`, `events_detailed.csv`, `visits.csv`, `crowd_counts.csv`, and `note.md`/`note.pdf`. For the calibrated shop clip, the finite counting gate follows the entrance edge of the lower-right door mat; entry is from the mat into the shop and exit is back toward the mat. The gate ignores crossings outside its short doorway segment and the video shows a boxed EXIT marker. Initial occupancy is estimated from people detected in `initial_occupancy_zone` on frame zero, excluding the two cashier stations; set `auto_initial_headcount: false` and `initial_headcount` to a known count to override this estimate. `visits.csv` includes visits that start with an observed IN crossing; people first seen inside the frame are not assigned fabricated visit times. Crowd count is the number of detected person boxes visible in each sampled frame; it is not a head-only detector and can miss people hidden by overlap. The submission CSV header is exactly `timestamp,person_id,event,headcount`; timestamps use video time in `HH:MM:SS.mmm` by default. The detailed sidecar carries role, confidence, dwell, flags, and notes.

## Tests

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s backend/tests -v
```

Tests cover zone transitions, jitter rejection, groups, first-seen-inside behavior, re-entry visits, headcount clamping, CSV schema, evaluation tolerance, and a mocked-detector 10-second pipeline smoke run. The mocked smoke test validates artifact creation and schema without representing detector accuracy.

## Configuration

All defaults are in [`config.yaml`](config.yaml). In particular:

- `model`, `conf`, `nms_iou`, `tracker`, `track_buffer`, and `frame_stride` configure YOLO11s/ByteTrack detection and tracking.
- `line.points` are normalized endpoints; `outside_side`, `buffer_zone_width`, and `debounce_frames` configure crossing logic.
- `initial_headcount`, `auto_initial_headcount`, `initial_occupancy_zone`, `crowd_zones`, `staff_zones`, `staff_rules`, and `reid` configure the starting count, zone outlines/alert thresholds, and optional role/re-entry heuristics. Adjust the staff polygons to match your camera view.
- `upload.max_size_mb`, `upload.max_duration_seconds`, `retention_minutes`, and `output_codec` configure limits, retention, and rendering.

The UI can override the line, inside direction, staff polygon, detector confidence, stride, and starting headcount per job. The server enforces upload and duration limits from the base YAML; client overrides cannot raise them.

## Demos

No example clip is bundled because there is no redistribution license attached to any footage. Add a properly licensed video and precompute its outputs:

```powershell
python scripts/precompute_demos.py licensed-clip.mp4 --title "Small retail entrance" --description "Front door"
```

This creates `demos/<id>/demo.json`, thumbnail, annotated video, CSV/JSON/metrics, note, and zip. The home page lists those manifests, and results open without processing. Do not publish customer footage without permission.

## API

- `GET /api/health` and `GET /api/demos`
- `POST /api/jobs` multipart: `video`, optional JSON string `config_json`
- `POST /api/calibrate` multipart video; returns suggested normalized line, inside/outside side, and preview
- `GET /api/jobs/{id}` and `GET /api/jobs/{id}/stream` (SSE)
- `POST /api/jobs/{id}/cancel`
- `GET /api/jobs/{id}/result`
- `GET /api/jobs/{id}/files/{name}` (MP4 supports byte ranges)
- `POST /api/jobs/{id}/ground-truth` multipart `file`, optional `tolerance_seconds`
- `GET /api/demos/{id}/result`, `GET /api/demos/{id}/files/{name}`

The in-memory job store and queue are per process; run one Uvicorn worker per instance. Jobs and uploaded videos are local to the container and expire after the configured retention period.

## Docker and deployment

```sh
docker compose up --build
```

Open `http://localhost:8000`. The Dockerfile builds the Vite app, installs FFmpeg and Python dependencies, and serves API plus static assets on port 7860. For Hugging Face Spaces, select Docker SDK and expose port 7860. For Render, create a Docker web service and keep its generated `PORT` environment variable. Attach persistent storage only if retaining temporary outputs for the one-hour session is necessary. This repository has not been deployed and has no configured public URL or GitHub remote.

To publish to Hugging Face after choosing your Space ID, authenticate locally and run:

```powershell
python -m pip install -r requirements-deploy.txt
hf auth login
python scripts/deploy_hf.py YOUR_USERNAME/storecount
```

The deploy script creates a public Docker Space if needed and uploads the app without local caches, job outputs, or demo source videos. It never accepts a token on the command line.

## Metrics and limitations

No demo metrics are claimed until you add and process licensed clips. Evaluation metrics appear only after a ground-truth CSV is supplied. Staff and re-entry classifications are heuristics; a staff zone improves uniform-color reference collection. ReID uses color histograms, which are weaker than a learned appearance embedding. Occlusion recovery and ID-switch detection are not implemented, so the report says so. Groups are counted per tracker box; merged detections can undercount. Camera angle, darkness, dense crowds, and people entering/exiting outside the field of view can reduce accuracy. Detector confidence values are not calibrated probabilities.

## License

MIT. Model weights and video footage may have separate terms; check each before redistribution.
