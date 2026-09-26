# RehabGrip

RehabGrip is a live hand-rehabilitation monitoring system: an ESP32-S3 glove streams finger-flexion, EMG, and IMU data over WebSocket to a Next.js clinician dashboard that renders a real-time 3D hand, detects reps, scores exercise quality, and generates patient reports.

This repository combines the two previously separate pieces of the project into one codebase:

| Folder | What it is | Originally |
|---|---|---|
| `/` (root) | The Next.js clinician dashboard — 3D hand visualization, session tracking, patient management, EMG view, AI Lab, analytics, and reports | `rehab_t3-main` |
| `/server` | The WebSocket relay server deployed on Render — bridges the ESP32-S3 glove and the dashboard | `rehab_t3_render-main` |

Everything below documents the system as a whole; folder-specific notes remain in `server/README.md`.

---

## How it fits together

```
 ESP32-S3 glove  ──ws──▶  /server (relay)  ──ws──▶  dashboard (this app)
 (flex + EMG +            Node + ws, deployed          Next.js 14, deployed
  IMU sensors)            on Render                    on Vercel
```

- The glove connects to the relay at `ws://<relay>/device` and streams JSON frames (`flex[5]`, `emg`, `imu`, `battery`).
- The relay validates each frame, caches the last good one, and rebroadcasts it to every connected dashboard client at `ws://<relay>/ws`.
- The relay also exposes `GET /health` for uptime checks and pings clients every 20s to keep Render's free tier from sleeping.
- The dashboard connects on demand (not automatically) from the **Session** page, reconstructs orientation from the raw IMU via a complementary filter, drives a Three.js hand model, and detects reps client-side.
- With `NEXT_PUBLIC_WS_URL` unset, the dashboard runs entirely in **SIM mode** — a hidden simulation panel drives the hand with presets/sliders/noise, so the whole UI can be developed and demoed with no hardware or relay running.

---

## Repository layout

```
.
├── app/                      # Next.js App Router pages
│   ├── session/               → live session view (3D hand, connect/disconnect)
│   ├── emg/                    → EMG waveform view
│   ├── patients/[id], /new     → patient list, profile, intake form
│   ├── history/                → past session history
│   ├── lab/                     → "AI Lab": disease simulation + AI assessment
│   ├── analytics/               → aggregate metrics/trends
│   ├── reports/[id]             → generated session/patient reports
│   ├── settings/
│   └── layout.tsx, page.tsx     → root layout (theme init, sidebar) + redirect to /session
│
├── components/
│   ├── SidebarLayout.tsx        # left nav (Session/EMG/Patients/History/AI Lab/Analytics/Reports/Settings)
│   ├── SessionPage.tsx, EMGPage.tsx, PatientsPage.tsx, HistoryPage.tsx,
│   │   AnalyticsPage.tsx, ReportsPage.tsx, SettingsPage.tsx, CalibrationPage.tsx
│   ├── SimulationPanel.tsx      # hidden dev/demo panel — SIM MODE, presets, noise, speed
│   ├── MetricsPanel.tsx         # live rep count / ROM / consistency readouts
│   ├── HealthKeepalive.tsx      # pings the relay's /health endpoint to keep it warm
│   ├── hand/                    # HandScene, HandModel, ConnectionOverlay, OrientationWidget (react-three-fiber)
│   ├── lab/                     # DiseaseEngine, LabHandScene, EMGWaveform, TrajectoryChart,
│   │                             #   MatchScoreWidget, ComparisonMetricsPanel, AIAssessmentPanel
│   ├── patients/                # NewPatientForm, PatientProfilePage
│   ├── reports/                 # ReportDetailPage
│   └── ui/Toast.tsx
│
├── hooks/
│   ├── useWebSocket.ts          # connects to the relay, parses frames, feeds the store (auto-reconnect, backoff 1s→30s)
│   └── useRepDetection.ts       # turns smoothed finger-flexion into rep counts + quality (correct/partial/missed)
│
├── store/handStore.ts           # zustand store — sensorRef (hot path, avoids re-renders), sim state, session/rep stats
├── types/sensor.ts, lab.types.ts
├── lib/                         # formatters, mock data, Indian patient sample data, report data, utils
│
├── server/                      # WebSocket relay server (deploy separately, e.g. on Render)
│   ├── index.js                 # relay: /device (glove) ↔ /ws (dashboards), /health, heartbeat, frame validation
│   ├── package.json
│   ├── render.yaml              # Render Blueprint (rootDir: server)
│   ├── README.md                # relay-specific docs
│   └── docs/useWebSocket.reference.ts   # original integration reference snippet (see note below)
│
├── .env.local.example
├── .gitignore
├── package.json / package-lock.json
├── next.config.js, tailwind.config.js, postcss.config.js, tsconfig.json
```

> **Note on `server/docs/useWebSocket.reference.ts`:** this is the original hand-off snippet written when the relay's message format was being designed. The dashboard's actual, more complete implementation lives at `hooks/useWebSocket.ts` (it already handles reconnect backoff, the complementary filter, and the `R`-to-zero-yaw shortcut). The reference file is kept only for context on the wire format and is not imported anywhere.

---

## Getting started

### 1. Dashboard (this app, root folder)

```bash
npm install
npm run dev
```

Open `http://localhost:3000` — with no `NEXT_PUBLIC_WS_URL` set, it starts in **SIM mode** (auto-cycling hand, no backend needed).

To connect to a real relay/device:

```bash
cp .env.local.example .env.local
# edit .env.local and set NEXT_PUBLIC_WS_URL
npm run dev
```

Then go to `/session` and click **Connect to Device** (it does not auto-connect on load).

### 2. Relay server (`/server`)

```bash
cd server
npm install
npm start
```

Test it locally with `wscat`:

```bash
npx wscat -c ws://localhost:8080/ws
```

See `server/README.md` for the full endpoint list and Render deployment steps.

---

## Deployment

**Relay → Render**
1. Push this repo to GitHub.
2. New Web Service on Render → connect the repo.
3. Root directory: `server` (already set in `server/render.yaml`).
4. Build command: `npm install`, start command: `npm start`.
5. Copy the resulting Render URL.

**Dashboard → Vercel**
1. Import the repo in Vercel (root directory: repo root, i.e. leave default).
2. Under Settings → Environment Variables, set `NEXT_PUBLIC_WS_URL` to `wss://<your-relay>.onrender.com/ws` (`.env.local` is **not** read at build/runtime on Vercel).
3. Deploy.

---

## Data contract (relay ↔ dashboard)

Frames sent by the relay are typed and consumed by `hooks/useWebSocket.ts`:

| Message type | Meaning | Shape |
|---|---|---|
| `sensor_data` | Live frame from the glove | `{ deviceId, timestamp, flex:[index,middle,ring,pinky,thumb], emg:{value}, imu:{ax,ay,az,gx,gy,gz}, battery }` |
| `sensor_data_cached` | Last known frame, replayed immediately on connect | same shape as `sensor_data` |
| `device_status` | Connection state of the glove | `{ status: 'online' \| 'offline' \| 'stale' }` |

- `flex` values are already calibrated angles in degrees (0–90), one per finger.
- IMU gyro is in °/s; the sensor is assumed Y-up. Orientation (roll/pitch/yaw) is derived client-side with a complementary filter. Yaw has no magnetometer correction and will drift — press **R** in the session view to zero it.

---

## Rep detection rules

Implemented in `hooks/useRepDetection.ts`:

- A rep is counted when the mean flexion across all 4 fingers rises above **15°** and then falls back below **10°**.
- Peak quality is classified as:
  - **Correct**: peak between 65°–85°
  - **Partial**: peak 30°–65°, or above 85°
  - **Missed**: peak below 30°

---

## Simulation mode

Click the near-invisible **⌥** control at the bottom-left of the session view to open the simulation panel:

- Toggle **SIM MODE** on/off
- Drag per-finger sliders or pick presets
- **AUTO CYCLE** through a demo sequence
- Add jitter with **NOISE** / **ADD NOISE**
- **SPEED** controls the per-frame smoothing factor (default `0.11` at 60fps)

This lets the entire dashboard — 3D hand, rep detection, EMG view, AI Lab, reports — be exercised and demoed without any physical glove or relay server running.

---

## Tech stack

**Dashboard:** Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS, Zustand, Framer Motion, Three.js via `@react-three/fiber` and `@react-three/drei`.

**Relay server:** Node.js, the `ws` WebSocket library, a plain `http` server (Render requires an HTTP port), no database — state is kept in memory (last good frame, connected clients).

---

## Project background

This is a hand-rehabilitation glove + dashboard project (RehabGrip) built around an ESP32-S3-based device that tracks finger flexion, forearm EMG, and wrist IMU during rehab exercises, streamed live to a clinician-facing web dashboard for session monitoring, patient tracking, and automated reporting.
