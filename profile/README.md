# FranchiseIQ Live

**A Google Cloud–native, multimodal, agentic site intelligence system:** mobile camera, voice, and GPS fused in real time with NYC open data in BigQuery, orchestrated by Gemini-powered agents, persisted in Firestore, and served via Cloud Run—so operators get instant field-to-dashboard decisioning.

---

## The problem

Franchise operators make million-dollar location calls on gut feel and stale spreadsheets. A typical Brooklyn site study means **3–6 months**, **$20K–$50K** for consultants, **weeks** for a PDF—and still a bet. No report captures what the block *feels* like at lunch on a Tuesday, whether foot traffic is real or cut-through, or why three tenants churned in two years.

**Wrong bets cost $500K–$1M+** in failed locations, leases, and buildouts. In the NYC metro alone, tens of thousands of units started with a site decision—usually with incomplete information, too slowly, and at too high a cost.

**The data already exists.** NYC publishes pedestrian counts, business licenses, 311 complaints, sanitation grades, permit activity. The gap is not data—it is **connecting that data to the moment of decision**, when someone is standing on the sidewalk deciding *this block or the next*.

---

## Our solution

**FranchiseIQ Live puts an expert site analyst in your ear while you walk the block.**

You open the mobile app, define your project and objective (concept, budget, priorities). You start a live survey: **camera on**, **location on**, **voice when you choose to ask**. The system:

- Pulls **pedestrian and complaint density** and **business-license context** for the grid around you (from engineered NYC open data in BigQuery).
- Interprets **what the camera sees** (vision signals for crowd cues, queues, vacancy proxies).
- **Scores the site** continuously and surfaces **findings with citations** back to datasets—not generic fluff.
- Supports **spoken Q&A**: you record your question, send it when ready; the backend transcribes, reasons over live context, and can respond with **text + synthesized speech** (`AUDIO_OUT`).
- Lets **HQ follow along** on the **web dashboard**: sessions, sites, scores, findings, transcript—compare blocks before you’re back in the car. **Export** hooks point at GCS for investor-style artifacts.

**What used to take six weeks and tens of thousands of dollars can compress into a live walk with a phone**—with claims grounded in public data and a full audit trail in the dashboard.

---

## What we built (three apps, one platform)

| Repo | Role |
|------|------|
| **`franchiseiq-mobile`** | Expo/React Native field app: projects, sessions, live survey (camera + GPS), manual voice query (start / send), WebSocket client, Firebase Auth token for API. |
| **`franchiseiq-api`** | FastAPI backend: REST (`/api/v1/projects`, `/query`, `/export`, …), WebSocket (`/api/v1/ws/{session_id}`), session orchestration, multi-agent pipeline, Firestore + BigQuery integration. |
| **`franchiseiq-webapp`** | Next.js dashboard: project/session/site views, findings, transcript, comparison and query UX—backed by the same REST API. |

---

## System architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FIELD (MOBILE)                                  │
│  Camera frames + GPS ──► WebSocket: FRAME, AUDIO_IN, COMMAND                │
│  ◄── SCORE_UPDATE, FINDING, TRANSCRIPT_TURN, AUDIO_OUT, SITE_*, ERROR         │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    franchiseiq-api (FastAPI on Cloud Run)                    │
│  • SessionManager: WS lifecycle, audio buffer, site/session commands           │
│  • OrchestratorService: agent fan-in / score / events out                     │
│  • REST: projects, sessions, sites, query, export                             │
└───────┬─────────────────────┬─────────────────────────────┬──────────────────┘
        │                     │                             │
        ▼                     ▼                             ▼
┌───────────────┐   ┌─────────────────┐           ┌──────────────────┐
│ Cloud         │   │ BigQuery         │           │ Vertex AI Gemini │
│ Firestore     │   │ franchiseiq_nyc  │           │ (vision, text,   │
│ projects,     │   │ + raw NYC tables │           │  STT-style,      │
│ sessions,     │   │ spatial features │           │  ADK delegate)   │
│ sites,        │   └─────────────────┘           └──────────────────┘
│ findings,     │             │
│ transcript    │             ▼
└───────────────┘   ┌─────────────────┐
        ▲           │ Optional:       │
        │           │ BQ snapshot →   │
        │           │ Firestore       │
        │           │ neighborhood_   │
        │           │ cache           │
        │           │ (Scheduler)     │
        │           └─────────────────┘
        │
        │  REST (Bearer / Firebase ID token)
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    franchiseiq-webapp (Next.js dashboard)                      │
│  Project detail, sites, findings, transcript, compare, project query          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**End-to-end flow (simplified):**

1. Mobile creates/joins a **project** and **session**, opens a **site survey**.
2. On a timer, **frames + lat/lng** hit the API over WebSocket; the orchestrator runs **parallel agents**, fuses into a **score**, persists, and pushes **FINDING** / **SCORE_UPDATE** (and optional narration).
3. User **starts** voice capture, **sends** one `AUDIO_IN` payload when finished; server transcribes (when enabled), runs **grounded query**, returns **TRANSCRIPT_TURN** + optional **AUDIO_OUT** (TTS).
4. Web app reads the same truth from **Firestore** via REST for review and comparison.

---

## Google Cloud & Firebase (hackathon checklist)

**Runtime & delivery**

- **Cloud Run** — API target deployment.
- **Cloud Build** — container build (`franchiseiq-api/infra/cloudbuild/cloudbuild.yaml`).

**Identity & data**

- **Firebase Authentication** — mobile sign-in; backend verifies ID tokens for REST/WebSocket.
- **Cloud Firestore** — projects, sessions, sites, findings, transcript; optional `neighborhood_cache` from BigQuery refresh.

**Analytics & AI**

- **BigQuery** — curated NYC feature tables + nearest-grid lookups for live scoring.
- **Vertex AI (Gemini)** — vision on frames, text generation for query/narration/predictive labels, audio-in transcription path when `ENABLE_REAL_STT` is on, orchestration-style delegation via `AdkRuntime`.
- **Cloud Text-to-Speech** — spoken AI replies as `AUDIO_OUT` when enabled.

**Supporting**

- **Cloud Scheduler** — hourly hit to `POST /api/v1/internal/cache-refresh` (see `scripts/create_cache_refresh_scheduler.sh`).
- **Cloud Storage** — export job URIs (`gs://franchiseiq-exports/...`) from the export API contract.

---

## Multimodal surface area

| Modality | What happens |
|----------|----------------|
| **Vision** | Base64 frames from the phone → Gemini vision JSON (traffic proxy, queues, anchors, vacancy signals). |
| **Audio in** | User-controlled recording → single `AUDIO_IN` with `is_final` → transcribe + grounded answer. |
| **Audio out** | TTS MP3 payload over WebSocket; mobile plays via Expo AV. |
| **Geospatial** | Every frame carries **lat/lng**; BigQuery joins to **grid-level** features. |
| **Text** | `TRANSCRIPT_TURN` events + `/api/v1/query` for project-level questions. |

---

## Agents and how they talk to each other

Orchestration is **hub-and-spoke** in `OrchestratorService`, not pairwise agent chat:

1. **Parallel fan-out** (per frame): `DataAgent`, `RiskAgent`, `VisionAgent` run concurrently (with timeouts).
2. **Fusion**: `ScorerAgent` consumes all three → **composite score**, component scores, confidence, delta.
3. **Conditional downstream**: `NarratorAgent` when the score move or risk anomaly warrants a short spoken-style line.
4. **Predictive**: `PredictiveAgent` adds a forward-looking finding from score context.
5. **Query path**: `QueryService` (and related Gemini prompts) answers questions **grounded** in the last score + data + risk + vision blob.
6. **ADK-style layer**: `AdkRuntime.delegate` records a structured “accept/rationale/priority” style decision over the scorer invocation (Gemini-backed when real agents are on).

**Transport to clients** is a **typed event bus over WebSocket** (`SCORE_UPDATE`, `FINDING`, `TRANSCRIPT_TURN`, `AUDIO_OUT`, …)—so mobile and web stay in sync with one contract (`franchiseiq-api/docs/contracts.md`).

---

## NYC open data (ingested into BigQuery)

**Raw-style tables (documented in `franchiseiq-api/docs/data-layer.md`):**

- `nyc_311_requests`
- `dca_business_licenses`
- `dot_pedestrian_counts`
- `dob_permits`
- `dohmh_restaurant_grades`
- `dca_license_history`

**Engineered tables** (examples): `complaints_by_grid`, `biz_density_by_grid`, `pedestrian_index`, `permit_activity`, `sanitation_by_zip`, `biz_cohort_outcomes`.

**Live scoring path today** leans on **pedestrian_index**, **complaints_by_grid**, and **biz_density_by_grid**; the rest extend the story for permits, sanitation, and cohort outcomes.

---

## One-liner for judges

**We close the gap between NYC’s open data and the second you’re on the sidewalk—with a real-time, multimodal agent stack on Google Cloud that scores the block, cites the datasets, talks back through TTS, and mirrors everything to a web command center.**

---

## Repo map

- `franchiseiq-mobile/` — Field app (Expo).
- `franchiseiq-api/` — Backend (FastAPI), SQL for BigQuery, Cloud Build config.
- `franchiseiq-webapp/` — Dashboard (Next.js).

For local API run and env vars, see `franchiseiq-api/README.md` and `franchiseiq-api/.env.example`.
