# NetWatch — ML-Powered Network Intrusion Detection System

An ML-powered passive network intrusion detection system (NIDS) designed to surface high-fidelity, actionable alerts to a Security Operations Center (SOC) analyst. This platform leverages a two-pronged machine learning approach to detect both known campaigns and novel zero-day attacks without blocking or dropping live traffic.

Everything is functional—live traffic simulation (replay), real-time WebSocket alert feeds, strict FPR-budgeted ML scoring, concept drift monitoring, and an interactive SOC dashboard.

## Contents
- [Quick start](#quick-start)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Database schema](#database-schema)
- [API overview](#api-overview)
- [Edge cases and how they are handled](#edge-cases-and-how-they-are-handled)
- [Assumptions](#assumptions)
- [Project layout](#project-layout)

## Quick start
Requires Python 3.11+ and Node 20+. The application comes with scripts to start the backend API, the Next.js SOC dashboard, and the network traffic replay engine concurrently.

### Mac/Linux
```bash
git clone [https://github.com/Awesome06/NetWatch.git](https://github.com/Awesome06/NetWatch.git)
cd NetWatch
chmod +x start.sh
./start.sh
```

### Windows
```cmd
git clone [https://github.com/Awesome06/NetWatch.git](https://github.com/Awesome06/NetWatch.git)
cd NetWatch
start.bat
```

## Signing in / Initializing
Authentication is currently mocked to focus on core SOC triage workflows. Upon starting the application, a replay script automatically begins simulating live network traffic using held-out test data. The SQLite database is seeded with initial synthetic benign flows to establish a baseline for the dashboard instantly.

## Tech stack

| Layer | Choice |
| :--- | :--- |
| **Frontend** | Next.js (App Router), TypeScript, Tailwind CSS |
| **Backend / API** | Python, FastAPI, WebSockets |
| **Machine Learning** | LightGBM, scikit-learn (Isolation Forest), Pandas |
| **Database** | SQLite, SQLAlchemy 2.0 |
| **Simulation** | Custom Python Replay Engine |

## Architecture

```text
┌──────────────────────────── Browser ────────────────────────────┐
│                        Next.js SOC UI                           │
│                                                                 │
│   ├── UI Components (Alert Feed, Metrics, Drift Charts)         │
│   └── State Management (WebSocket Client, REST Fetch)           │
└───────────────┬───────────────────────────────▲─────────────────┘
                │ HTTP / REST                   │ WebSockets
┌───────────────▼───────────────────────────────┴─────────────────┐
│                        FastAPI Backend                          │
│                                                                 │
│   ├── api/            REST endpoints & WS connections           │
│   ├── models/         Decision Engine (combine.py)              │
│   └── evaluate/       LightGBM & Isolation Forest Inference     │
└───────────────▲───────────────────────────────┬─────────────────┘
                │ Internal API                  │
┌───────────────┴────────────────┐      ┌───────▼─────────────────┐
│     Replay Engine (Traffic)    │      │         SQLite          │
└────────────────────────────────┘      └─────────────────────────┘
```

Three decisions shape everything else:
1. **Passive Analysis.** Nothing in this repository blocks or drops traffic. The system operates entirely out-of-band, analyzing network flows and presenting triaged alerts to analysts.
2. **FPR Budgeting over Probability.** The alert threshold does not come from an arbitrary 0.5 probability. It is mathematically derived from a strict false-positive budget (e.g., max 0.5% FPR) to ensure analysts are not overwhelmed with noise.
3. **True Zero-Day Testing.** Rare attack families (e.g., Infiltration, Heartbleed) are never trained on. They serve purely as test material to validate the Isolation Forest's anomaly detection.

## Database schema
Three core tables handling the entities and alerts.

```text
network_flows
├─ id PK
├─ timestamp
├─ src_ip
├─ dst_ip
├─ port
├─ protocol
├─ bytes_in
└─ bytes_out

alerts
├─ id PK
├─ flow_id ─► network_flows.id
├─ timestamp
├─ prediction_type 'known_attack' | 'anomaly'
├─ severity 'low' | 'medium' | 'high' | 'critical'
├─ model_confidence
└─ is_acknowledged BOOLEAN

drift_metrics
├─ id PK
├─ timestamp
├─ feature_name
├─ psi_score
├─ ks_stat
└─ is_drifting BOOLEAN
```

## API overview
All REST routes are prefixed with `/api`. Real-time feeds use `/ws`.

| Method | Path | Purpose |
| :--- | :--- | :--- |
| `GET` | `/api/alerts` | List recent unacknowledged alerts |
| `PATCH` | `/api/alerts/{id}` | Update alert status (e.g., mark as acknowledged) |
| `GET` | `/api/stats` | Retrieve aggregated metrics for the SOC dashboard |
| `POST` | `/api/ingest` | Ingest new network flow batch from replay/sensor |
| `WS` | `/ws/alerts` | WebSocket feed for real-time alert broadcasting |

## Edge cases and how they are handled

| Case | Mechanism |
| :--- | :--- |
| **Data Leakage** | No random train/test splits. Flows are grouped into 5-minute blocks. The pipeline fails the build if a block lands in both sets. |
| **Novel / Zero-Day Attacks** | The LightGBM classifier misses these, but the Isolation Forest (trained exclusively on benign traffic) flags them as anomalies. |
| **Imbalanced Datasets** | Bare accuracy is ignored. Evaluated strictly on per-class precision/recall, PR-AUC, and false alerts per 10,000 benign flows. |
| **Concept Drift** | Automated scripts monitor Population Stability Index (PSI) and KS tests per feature, flagging drift to trigger retraining. |

## Assumptions
- **Passive Context:** Assume the system operates on a mirrored port or TAP. It cannot drop packets; it can only alert.
- **Dataset Focus:** Built and evaluated primarily against CICIDS2017 flow data structures.
- **Analyst Workflow:** Assume a default SOC analyst is viewing the dashboard; focus remains on data fidelity and alert triage rather than RBAC (Role-Based Access Control) flows.

## Project layout

```text
.
├── api/                  # FastAPI scoring service, WebSocket feed
├── dashboard/            # Next.js SOC UI
├── docs/                 # Extensive documentation (DATA_SETUP, CODE_TOUR, evaluation)
├── ml/
│   ├── data/             # Leakage-free time-block splits and label mapping
│   ├── drift/            # PSI/KS test monitors for drift
│   ├── evaluate/         # Metrics, FPR budgeting, LOFO tests
│   └── models/           # LightGBM, Isolation Forest, combine.py
├── replay/               # Simulates live traffic for demos
├── scripts/              # Dataset download, synthetic generation
├── tests/                # Pytest suite ensuring pipeline integrity
├── HOW_TO_RUN.md         # Detailed local setup instructions
└── README.md
```
## Documentation
To keep this root README clean, deep dives into the ML methodology are housed in the `docs/` folder:
* **[Data Setup & Ingestion](docs/DATA_SETUP.md):** Downloading and verifying CICIDS2017 flow data.
* **[Code Tour](docs/CODE_TOUR.md):** Architectural deep dive into the dual-model inference pipeline.
* **[Evaluation Methodology](docs/evaluation.md):** Why we use PR-AUC and strict FPR budgeting over bare accuracy.
* **[Drift Strategy](docs/drift_strategy.md):** PSI and KS-test monitoring for the LightGBM classifier.

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contact
**Mrigank Bhatnagar**  
* [LinkedIn](https://linkedin.com/in/mrigank-bhatnagar)
* [GitHub](https://github.com/Awesome06)
