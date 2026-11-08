# SkillProof
### AI-Based Real-time Monitoring of Training Centres for Attendance and Infrastructure Compliance

**Smart India Hackathon 2026 · Problem Statement ID: SIH26245**

SkillProof is a proposed zero-trust monitoring platform for helping authorised programme administrators and inspectors assess training-centre attendance and infrastructure compliance. The system is designed around three components:

1. **Edge Inference Agent** — performs on-premises camera-frame processing, attendance matching and asset counting.
2. **Cloud Backend** — receives signed telemetry, verifies its origin and integrity, records audit events, and calculates centre-level compliance indicators.
3. **Inspector Dashboard** — presents centre status, attendance summaries, infrastructure counts, alerts and audit history.

> **Prototype status:** This repository is an initial project scaffold. It contains the planned folder structure, module boundaries, documentation and configuration templates. The application logic, model integration, database migrations, deployment manifests and user interface are intended to be implemented and validated in subsequent development phases. A file or module name in this scaffold does not mean that the capability is already implemented.

---

## 1. Problem statement

Training programmes need reliable ways to understand whether scheduled batches are operating as expected and whether required infrastructure is available. Manual inspection and periodic reporting can make it difficult to identify attendance irregularities, missing equipment or inconsistent records quickly.

SkillProof proposes a monitoring workflow that combines edge-side computer vision with verifiable telemetry and a central inspection dashboard. It aims to provide:

- Batch-level attendance evidence and consistency summaries.
- Counts of selected infrastructure assets, such as desktops, machinery and training kits.
- Detection of suspicious patterns for human review, including repeated or inconsistent attendance evidence and possible asset-count inflation.
- An auditable record of telemetry receipt, verification outcomes and compliance-score changes.
- Offline operation at the centre, with secure forwarding when connectivity returns.

The system is intended to **support inspection and decision-making**, not to make irreversible decisions about trainees or centres automatically.

## 2. Proposed solution and high-level architecture

```text
                    TRAINING CENTRE
┌───────────────────────────────────────────────────────────┐
│  Camera / approved video source                            │
│             │                                             │
│             ▼                                             │
│  EDGE INFERENCE AGENT                                     │
│  ├─ Capture scheduler (batch hours, configurable interval)│
│  ├─ Face detection / tracking / liveness / matching       │
│  ├─ Asset detection and counting                          │
│  ├─ Privacy guard and local frame disposal                │
│  ├─ Telemetry builder + HMAC-SHA256 signing               │
│  └─ Encrypted offline queue and retry                     │
└─────────────────────────────┬─────────────────────────────┘
                              │ HTTPS + signed telemetry
                              ▼
                         CLOUD PLATFORM
┌───────────────────────────────────────────────────────────┐
│  FastAPI ingestion API                                     │
│       │                                                   │
│       ├─ Device identity and HMAC verification             │
│       ├─ Request validation / replay protection            │
│       └─ Queue accepted event                              │
│                         │                                 │
│                         ▼                                 │
│                 Celery workers + Redis                    │
│                         │                                 │
│             ┌───────────┴────────────┐                    │
│             ▼                        ▼                    │
│       PostgreSQL + pgvector     Compliance engine          │
│       Rosters / vectors         Attendance / asset checks  │
│       Telemetry / audit         A–F index + risk flags      │
└─────────────────────────────┬─────────────────────────────┘
                              │ Authenticated API
                              ▼
┌───────────────────────────────────────────────────────────┐
│  INSPECTOR DASHBOARD — Next.js + Tailwind CSS              │
│  Centre overview · Attendance · Infrastructure · Alerts   │
│  Compliance rankings · Audit trail · Review workflow      │
└───────────────────────────────────────────────────────────┘
```

### Zero-trust design principles

SkillProof follows a zero-trust approach: no edge device, request or user is trusted solely because it is inside a centre network.

- **Device identity:** each edge installation is provisioned with a unique device identity and secret.
- **Message integrity:** telemetry is authenticated using HMAC-SHA256 over a canonical payload.
- **Transport security:** telemetry is sent over HTTPS/TLS. HMAC does not replace TLS.
- **Replay resistance:** packets should include a unique event ID, device ID and timestamp; the backend should reject duplicates and stale events according to a configured policy.
- **Least privilege:** edge devices, API services, workers, database roles and dashboard users receive only the permissions they need.
- **Auditability:** ingestion, verification, administrative changes and score revisions are recorded.
- **Human review:** anomalies and low-confidence matches are flagged for authorised review rather than treated as definitive proof of misconduct.

## 3. Main components

### A. Edge Inference Agent (`edge/`)

The edge agent is intended to run on a computer installed at an authorised training centre.

Planned responsibilities:

1. **Capture scheduling** — capture or sample an approved camera feed at a configurable interval (for example, every 10–15 minutes) during configured batch hours.
2. **Face pipeline** — detect and track faces using an SCRFD-compatible detector, evaluate presentation-attack/liveness signals using a Silent-Face-compatible model, and compare an embedding with an authorised roster using an ArcFace-compatible embedding model.
3. **Multi-snapshot confirmation** — require two or three consecutive positive observations before creating an attendance event. The exact threshold, time window and confidence values must be configurable and validated in a controlled pilot.
4. **Asset pipeline** — use a YOLOv8-compatible detector to estimate counts for configured classes such as desktops, machinery and training kits.
5. **Privacy guard** — process frames locally and discard raw frame buffers as soon as the required inference is complete. Do not upload raw face images or video in normal telemetry.
6. **Telemetry signing** — create a minimal structured payload and authenticate it with HMAC-SHA256.
7. **Offline-first delivery** — persist signed events in a protected local queue when the network is unavailable, then retry delivery with backoff and duplicate-safe event IDs.

**Important:** Face embeddings are biometric data and remain sensitive even when raw photographs are deleted. Local deletion alone does not establish legal compliance. The deployment must have a valid legal basis, clear notices, appropriate consent or other applicable authorisation, access controls, retention limits, a grievance process and a documented assessment under applicable Indian law and programme rules. Provide a non-biometric attendance alternative where required. Do not use this system for covert surveillance.

### B. Cloud Backend (`backend/`)

The backend is planned as a FastAPI service with asynchronous processing.

Planned responsibilities:

- Accept telemetry packets from registered edge devices.
- Validate schema, device status, timestamps and event identifiers.
- Verify HMAC-SHA256 signatures using securely stored device secrets.
- Apply replay protection and idempotency checks.
- Enqueue accepted packets for Celery workers through Redis.
- Store centre, batch, roster, embedding references, telemetry and audit records in PostgreSQL; use pgvector only where biometric matching is lawfully approved and necessary.
- Calculate an explainable A–F Compliance Index using attendance consistency and infrastructure checks.
- Expose authenticated APIs for the inspector dashboard.

#### Proposed compliance model

The initial concept is a weighted score from 0 to 100, mapped to letter bands. The weights and thresholds below are **illustrative starting points only** and must be calibrated with programme stakeholders and real pilot data.

| Dimension | Example contribution | What it measures |
|---|---:|---|
| Attendance consistency | 60% | Verified attendance against scheduled batch strength over time |
| Infrastructure compliance | 40% | Observed asset counts against approved centre inventory |

Possible initial grade bands:

| Grade | Illustrative score |
|---|---:|
| A | 90–100 |
| B | 80–89 |
| C | 70–79 |
| D | 60–69 |
| E | 40–59 |
| F | 0–39 |

An asset-inflation check should compare observed counts with approved inventory, historical patterns and inspection evidence. A mismatch should create a **flag for review**, not an automatic accusation. Scores should show their inputs, data freshness, confidence and missing-data status. A centre should not be penalised merely because an edge device was offline or a camera could not produce a reliable observation.

### C. Inspector Dashboard (`dashboard/`)

The dashboard is planned as a responsive Next.js application styled with Tailwind CSS.

Planned screens and widgets:

- State/administrator overview of active training centres.
- Centre detail page with batch schedule and recent telemetry.
- Verified daily attendance summaries with confidence and data freshness.
- Infrastructure inventory versus observed asset counts.
- High-risk anomaly queue with reason, severity and review status.
- Live audit table showing event time, centre, device, verification result and processing status.
- A–F compliance rankings with score explanations and historical trends.
- Role-based access for inspectors, administrators and auditors.

The dashboard must not expose raw face images or unrestricted biometric vectors.

## 4. Repository structure

```text
SkillProof/
├── README.md
├── .gitignore
├── .env.example
├── LICENSE
├── docs/
│   ├── architecture.md
│   ├── api-contract.md
│   ├── data-dictionary.md
│   ├── privacy-and-security.md
│   └── roadmap.md
├── edge/
│   ├── README.md
│   ├── requirements.txt
│   ├── config/
│   │   └── edge-config.example.yaml
│   ├── src/
│   │   ├── main.py
│   │   ├── capture/
│   │   │   ├── scheduler.py
│   │   │   └── camera_source.py
│   │   ├── vision/
│   │   │   ├── face_detector.py
│   │   │   ├── liveness.py
│   │   │   ├── face_matcher.py
│   │   │   └── asset_detector.py
│   │   ├── privacy/
│   │   │   └── frame_guard.py
│   │   ├── telemetry/
│   │   │   ├── payload.py
│   │   │   ├── signer.py
│   │   │   └── offline_queue.py
│   │   └── utils/
│   │       └── connectivity.py
│   └── tests/
│       └── .gitkeep
├── backend/
│   ├── README.md
│   ├── requirements.txt
│   ├── alembic.ini
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   │   ├── routes/
│   │   │   │   ├── ingest.py
│   │   │   │   ├── centres.py
│   │   │   │   ├── attendance.py
│   │   │   │   └── compliance.py
│   │   │   └── dependencies.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── security.py
│   │   │   └── logging.py
│   │   ├── models/
│   │   │   ├── centre.py
│   │   │   ├── device.py
│   │   │   ├── trainee.py
│   │   │   ├── telemetry.py
│   │   │   └── audit.py
│   │   ├── schemas/
│   │   │   └── telemetry.py
│   │   ├── services/
│   │   │   ├── signature_service.py
│   │   │   ├── attendance_service.py
│   │   │   ├── asset_service.py
│   │   │   └── compliance_engine.py
│   │   ├── workers/
│   │   │   ├── celery_app.py
│   │   │   └── telemetry_tasks.py
│   │   └── db/
│   │       ├── session.py
│   │       └── base.py
│   ├── migrations/
│   │   └── versions/
│   │       └── .gitkeep
│   └── tests/
│       └── .gitkeep
├── dashboard/
│   ├── README.md
│   ├── package.json
│   ├── next.config.ts
│   ├── postcss.config.mjs
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── public/
│   │   └── .gitkeep
│   └── src/
│       ├── app/
│       │   ├── layout.tsx
│       │   ├── page.tsx
│       │   ├── centres/
│       │   ├── attendance/
│       │   ├── alerts/
│       │   └── audit/
│       ├── components/
│       │   ├── layout/
│       │   ├── dashboard/
│       │   ├── charts/
│       │   └── tables/
│       ├── lib/
│       │   ├── api.ts
│       │   └── types.ts
│       └── styles/
│           └── globals.css
├── infra/
│   ├── docker/
│   │   └── .gitkeep
│   ├── postgres/
│   │   └── init.sql
│   └── deployment/
│       └── .gitkeep
└── .github/
    └── workflows/
        └── .gitkeep
```

## 5. Telemetry contract (proposed)

The following is an illustrative event shape. It intentionally contains derived observations rather than raw camera frames.

```json
{
  "event_id": "uuid",
  "device_id": "centre-device-001",
  "centre_id": "centre-001",
  "batch_id": "batch-2026-08-a",
  "captured_at": "2026-08-12T10:15:00Z",
  "schema_version": "1.0",
  "attendance": {
    "observations": [
      {
        "roster_subject_ref": "pseudonymous-reference",
        "match_confidence": 0.97,
        "liveness_passed": true,
        "consecutive_confirmations": 3
      }
    ]
  },
  "assets": {
    "desktop": 24,
    "machinery": 4,
    "training_kit": 18
  },
  "model_versions": {
    "face_detector": "configured-version",
    "liveness": "configured-version",
    "face_embedding": "configured-version",
    "asset_detector": "configured-version"
  }
}
```

The final contract must define canonical JSON serialization for signing, timestamp tolerance, allowed class names, confidence ranges, maximum packet size, schema versioning and error responses. Do not include names, raw images or reusable biometric templates in ordinary telemetry. A pseudonymous reference is still potentially personal data and must be protected.

## 6. Suggested data model

The PostgreSQL schema is expected to include the following logical entities:

| Entity | Purpose |
|---|---|
| `centres` | Centre identity, location reference, status and approved inventory |
| `batches` | Batch schedule, course and centre relationship |
| `trainees` | Minimum required trainee identity and roster status |
| `biometric_templates` | Separately protected embedding records, only if approved |
| `edge_devices` | Device identity, key reference, status and last-seen time |
| `telemetry_events` | Received event metadata, signature status and processing state |
| `attendance_observations` | Pseudonymous attendance evidence and verification metadata |
| `asset_observations` | Detected asset class, count, confidence and timestamp |
| `compliance_scores` | Score, grade, component values and scoring version |
| `anomaly_flags` | Rule, evidence reference, severity and human review status |
| `audit_logs` | Append-oriented record of security and administrative events |

Use PostgreSQL constraints and indexes for integrity and query performance. Enable the pgvector extension only if the approved design requires vector search. Keep biometric templates in a separately permissioned data area, encrypt them at rest, restrict query access, and define deletion and rotation procedures.

## 7. Security, privacy and responsible deployment

SkillProof may process biometric and other personal data. Privacy and security must be design requirements, not items added after implementation.

- **Purpose limitation:** collect only what is necessary for the stated attendance and infrastructure purpose.
- **Notice and lawful basis:** provide clear notice to trainees, trainers and centre staff; establish the applicable lawful basis and required authorisations before any collection.
- **Data minimisation:** process video on the edge. Do not transmit or centrally store raw frames as part of the normal workflow.
- **Biometric protection:** embeddings are sensitive biometric data. Protect them with encryption, strict role-based access, audit logging, retention limits and controlled deletion.
- **Alternative process:** provide a practical non-biometric attendance method for people who cannot or do not participate, where required by law or programme policy.
- **Accuracy and bias testing:** evaluate false accepts, false rejects, liveness failures and performance across relevant lighting, camera and demographic conditions. Do not use face matching as the sole basis for denying attendance or benefits.
- **Human oversight:** provide a correction and appeal process; require authorised review before adverse action.
- **Device security:** secure boot where available, OS hardening, encrypted local storage, key rotation, patching and physical tamper controls.
- **Secret management:** never commit real HMAC keys, database passwords, API tokens or private certificates. Use a secrets manager or protected environment configuration.
- **Retention and deletion:** document retention periods for each data category and verify deletion across edge queues, backups, logs and database replicas.
- **Incident response:** define procedures for lost devices, compromised keys, data exposure and incorrect attendance records.

This README is a technical proposal, not legal advice or a certification of DPDP Act compliance. Before a live deployment, obtain a privacy/legal review against the Digital Personal Data Protection Act, 2023, applicable rules and notifications in force, government programme requirements and institutional policies.

## 8. Technology stack

| Layer | Proposed technologies |
|---|---|
| Edge runtime | Python, OpenCV, SCRFD-compatible face detection, Silent-Face-compatible liveness, ArcFace-compatible embeddings, YOLOv8 |
| Edge security | HMAC-SHA256, HTTPS/TLS, protected local queue |
| API | Python, FastAPI, Pydantic |
| Async processing | Celery, Redis |
| Database | PostgreSQL, pgvector, SQLAlchemy, Alembic |
| Dashboard | Next.js, React, TypeScript, Tailwind CSS |
| Operations | Docker, environment-based configuration, structured logs, automated tests |

Model names indicate intended integration points. Verify model licences, weights, redistribution terms, hardware requirements and suitability before selecting production dependencies.

## 9. Development roadmap

**Phase 1 — Repository and contracts**
- Finalise requirements, threat model, data-flow diagram and telemetry schema.
- Define centre, batch, device and role identifiers.
- Decide privacy/legal basis and non-biometric fallback.

**Phase 2 — Backend foundation**
- Build FastAPI health and ingestion endpoints.
- Implement device registration, canonical payload signing/verification and replay protection.
- Add PostgreSQL models, migrations, Redis and Celery worker flow.

**Phase 3 — Edge prototype**
- Build configurable capture scheduler and simulated camera source.
- Add face and asset inference adapters.
- Implement consecutive-snapshot logic, local privacy guard, signed payloads and offline retry queue.

**Phase 4 — Dashboard**
- Build centre overview, attendance, asset inventory, alerts and audit screens.
- Add authentication, role-based access and API integration.

**Phase 5 — Validation and pilot**
- Test signature tampering, replay, offline recovery and device compromise scenarios.
- Measure model accuracy and false-match rates under realistic conditions.
- Validate asset counts against manual ground truth.
- Conduct privacy review, security testing, accessibility checks and stakeholder feedback.

## 10. Local development (planned)

This repository is currently a scaffold. The commands below are a suggested setup sequence for when the corresponding application files and dependencies are implemented.

### Backend
```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
```

### Edge agent
```bash
cd edge
python -m venv .venv
# Windows: .venv\Scripts\activate
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
```

### Dashboard
```bash
cd dashboard
npm install
npm run dev
```

Do not expect these commands to launch a working application until the scaffold modules, dependencies and environment configuration have been implemented.

## 11. Configuration and secrets

Copy `.env.example` to a local `.env` file and fill in development-only values. Never commit `.env` or real secrets. The edge device should receive its own unique secret through a secure provisioning process; do not use one shared secret across all centres. Production secrets should be stored in an appropriate secrets manager and rotated if exposure is suspected.

## 12. Current scope and limitations

At this stage:

- The repository structure and design documentation are provided.
- The code paths are placeholders for planned modules.
- No trained model weights are included.
- No real camera capture, face recognition, liveness detection, asset detection, API, worker, database migration or dashboard is implemented yet.
- No DPDP compliance certification or production security audit has been performed.

These limitations are intentional for the initial SIH project repository. The next development stage is to implement and test each module against the architecture and requirements documented here.

## 13. Team and contribution

Contributions should follow the module boundaries above. Before merging changes, include tests, update the relevant documentation, avoid committing personal data or secrets, and describe any change to telemetry fields or scoring logic.

---

**SkillProof — Trustworthy training-centre monitoring through edge intelligence, verifiable telemetry and accountable inspection.**
