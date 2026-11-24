# SmartProject AI — Comprehensive Technical Description

> **Purpose:** This document is written for use in a presentation and lab report.  
> It covers what the system does, how it is architected, how each AI model works, and how all the parts connect.

---

## 1. Project Overview

**SmartProject AI** is an intelligent software project health and bug triage platform. Its core purpose is to take two historically manual, error-prone processes in software engineering — **deciding how serious a bug is** and **predicting whether a project is heading toward failure** — and automate them with machine learning.

The system uses a **dual-track AI architecture**:

| Track | Name | Model | Task |
|---|---|---|---|
| Track 1 | **Tactical AI** | Fine-tuned DistilBERT (NLP) | Classify the *severity* of a bug report (Low / Medium / High / Critical) in real-time |
| Track 2 | **Strategic AI** | Custom Mamba/SSM (time-series) | Predict the *health trajectory* of the entire project over a 30-day window |

The backend is a **Django 4.2 REST API**, the database is **PostgreSQL**, and the frontend is a single-page dashboard styled with an *"Anti-Gravity Cyber-Telemetry"* dark-mode aesthetic. Asynchronous background tasks are handled by **Celery + Redis**.

---

## 2. Problem Statement

In large software projects:
- Bug reports are typically triaged manually by engineers, which is slow and inconsistent.
- Project managers have no reliable early-warning system to detect when a project is "free-falling" into failure before it's too late.

SmartProject AI addresses both problems by combining **Natural Language Processing** (for bug text understanding) with **Sequence Modeling** (for project trajectory analysis) into a single, deployable web application.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client (Browser / API)                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP
┌──────────────────────────▼──────────────────────────────────────┐
│                  Django REST API (Port 8000)                      │
│  POST /api/bugs/create/      GET /api/project/health/            │
│  GET  /api/bugs/             GET /api/developers/                │
│  POST /api/project/trigger-snapshot/                             │
└────────┬──────────────────────────────────────┬─────────────────┘
         │                                      │
┌────────▼──────────┐                ┌──────────▼──────────────────┐
│  Tactical AI App  │                │    Strategic AI App          │
│  tactical_ai/     │                │    strategic_ai/             │
│                   │                │                              │
│  DistilBertPred-  │                │  MambaPredictor (singleton)  │
│  ictor (singleton)│                │  + Celery Task (daily cron)  │
│                   │                │                              │
│  Model: ~260 MB   │                │  Model: ~few MB             │
│  DistilBERT +     │                │  Pure-PyTorch SSM            │
│  tokenizer        │                │  (Mamba-style blocks)        │
└────────┬──────────┘                └──────────┬──────────────────┘
         │                                      │
┌────────▼──────────────────────────────────────▼─────────────────┐
│              PostgreSQL Database (via Django ORM)                 │
│  BugReport  │  DeveloperProfile  │  DailyProjectMetric           │
│  ProjectHealthSnapshot                                            │
└─────────────────────────────────────────────────────────────────┘
         │
┌────────▼──────────┐
│  Redis + Celery   │
│  Beat Scheduler   │
│  (Daily 00:05 UTC)│
└───────────────────┘
```

---

## 4. Dual-Track AI Pipeline — How It Works

### 4.1 Track 1: Tactical AI — DistilBERT Bug Severity Classifier

#### What it does
When a developer submits a bug report (title + description), the system instantly classifies it into one of four severity levels:
- **Low** → Priority P3 (Backlog)
- **Medium** → Priority P2 (Next Sprint)
- **High** → Priority P1 (Fix This Sprint)
- **Critical** → Priority P0 (Fix Immediately)

#### The Model: DistilBERT
**DistilBERT** (Distilled BERT) is a compact Transformer-based language model published by Hugging Face. It retains ~97% of BERT's performance at 60% of its size (~66M parameters), making it practical for on-server CPU inference.

The base `distilbert-base-uncased` model was **fine-tuned** (transfer learning) for the specific task of 4-class bug severity classification.

#### Training Pipeline (`notebooks/01_tactical_distilbert_training.py`)

**Step 1 — Dataset**
The training data consists of bug reports. The system supports three formats:
- **Eclipse JDT** (real bug reports from the Eclipse IDE bugzilla tracker, with P1–P5 priorities mapped to severity labels)
- **NASA MDP** (NASA defect dataset, pre-labeled)
- **Synthetic fallback** (2,000 auto-generated examples when no real dataset is available)

**Step 2 — Text Preprocessing**
Each bug report goes through a `clean_bug_text()` function that:
1. Strips HTML tags (Bugzilla descriptions contain HTML)
2. Removes Java/Python stack traces (too technical, low semantic signal)
3. Removes URLs, replaces with `[URL]` token
4. Normalizes whitespace
5. Truncates to 1,000 characters

The cleaned title and description are then concatenated:
```
"{title} [SEP] {description}"
```

**Step 3 — Tokenization**
`DistilBertTokenizerFast` converts the text into:
- `input_ids`: Numerical token IDs (max 256 tokens, padded/truncated)
- `attention_mask`: 1/0 mask distinguishing real tokens from padding

**Step 4 — Dataset Split**
The dataset is split with **stratified sampling** (ensures equal class distribution in each split):
- 80% Training
- 10% Validation
- 10% Test

**Step 5 — Fine-Tuning**
The model is trained using Hugging Face's `Trainer` API with these key hyperparameters:

| Hyperparameter | Value |
|---|---|
| Base model | `distilbert-base-uncased` |
| Max sequence length | 256 tokens |
| Learning rate | 2×10⁻⁵ |
| Epochs | 5 |
| Batch size (train) | 16 |
| Warmup steps | 100 |
| Weight decay | 0.01 |
| Evaluation metric | Macro F1 |

The best checkpoint (highest Macro F1 on validation) is saved automatically.

**Step 6 — Evaluation**
On the test set, the model reports:
- **Classification Report** (precision, recall, F1 per class)
- **Confusion Matrix** (4×4: Low/Medium/High/Critical)
- **Macro F1** and **Weighted F1** scores

**Step 7 — Export**
The fine-tuned model is saved using `model.save_pretrained()` which outputs:
```
smartproject_distilbert/
    config.json
    model.safetensors  (or pytorch_model.bin)
    tokenizer_config.json
    vocab.txt
    ...
label_map.json  → {"label2id": {...}, "id2label": {...}}
```

#### Django Integration (`tactical_ai/services.py`)

The `DistilBertPredictor` class implements a **thread-safe singleton pattern**:
- The model is loaded **once** at Django startup (via `AppConfig.ready()`)
- All subsequent requests reuse the same in-memory loaded model
- Uses **double-checked locking** (`threading.Lock`) to prevent race conditions in multi-threaded WSGI environments

**Inference flow for each bug submission:**
```
Input text → Tokenize → torch.no_grad() inference → Softmax → argmax
→ Predicted severity + Confidence score + Priority derivation
```

The severity-to-priority business rules matrix:
```python
SEVERITY_TO_PRIORITY = {
    "Critical": "P0",
    "High":     "P1",
    "Medium":   "P2",
    "Low":      "P3",
}
```

---

### 4.2 Track 2: Strategic AI — Mamba/SSM Project Risk Predictor

#### What it does
Every day at 00:05 UTC, the system collects the **last 30 days of project metrics** and feeds them through a sequence model to compute a **project health score** between 0.0 and 1.0, then classifies the project's **risk trajectory**:

| Health Score Range | Trajectory Label | Meaning |
|---|---|---|
| ≥ 0.75 | 🟢 **Accelerating** | Project momentum improving |
| 0.50 – 0.74 | 🟡 **Stable** | Holding trajectory |
| 0.25 – 0.49 | 🟠 **Degrading** | Project health declining |
| < 0.25 | 🔴 **Critical Risk** | FREE-FALLING — immediate intervention |

#### The Model: Selective State Space Model (Mamba/SSM)

The Strategic AI uses a **Selective State Space Model (SSM)**, specifically a pure-PyTorch implementation of the **Mamba architecture** (Gu & Dao, 2023: *"Mamba: Linear-Time Sequence Modeling with Selective SSMs"*).

Unlike RNNs and Transformers, Mamba is designed for long-range sequence modeling with **linear time complexity**, making it ideal for time-series health monitoring.

> **Why not use the official `mamba-ssm` library?**  
> The `mamba-ssm` library requires CUDA kernel compilation, making it incompatible with Windows development environments and CPU-only Django servers. This project implements a **functionally equivalent pure-PyTorch SSM** that works on any platform.

#### Input Features (5 time-series dimensions)
Each day in the 30-day window produces one row with 5 features:

| Feature | Description |
|---|---|
| `bugs_opened` | New bug reports created on that day |
| `bugs_closed` | Bug reports resolved on that day |
| `mean_resolution_time_hrs` | Average hours to resolve bugs that day |
| `accumulated_backlog` | Total number of open/in-progress bugs |
| `defect_density` | Ratio of bugs opened to bugs resolved (trailing 7 days) |

The input tensor shape is: **(30, 5)** — 30 days × 5 features.

#### Model Architecture (`SelectiveSSMBlock` + `MambaProjectModel`)

```
Input (batch, 30, 5)
    ↓ Linear Projection
(batch, 30, d_model=64)
    ↓ × 3 SelectiveSSMBlock layers
(batch, 30, 64)
    ↓ Global Average Pooling (over time axis)
(batch, 64)
    ↓ Dropout (0.1)
    ↓ Linear(64 → 32) + GELU + Linear(32 → 1) + Sigmoid
Output: health_score in [0.0, 1.0]
```

**Inside each `SelectiveSSMBlock`:**
The core Mamba innovation is *input-dependent* (selective) state transitions:
1. **Input projection** → splits into an inner branch and a gate `z`
2. **Depthwise Conv1D** → captures local temporal context
3. **SiLU activation**
4. **Input-dependent projections** of:
   - **Δ (delta)** — step size (controls how much each time step influences the state)
   - **B** — input matrix
   - **C** — output matrix
5. **SSM scan** — sequential state update: `h_t = Ā·h_{t-1} + B̄·u_t`, `y_t = C·h_t`
6. **Output gating** with SiLU gate from `z`
7. **Residual connection**

#### Training Pipeline (`notebooks/02_strategic_mamba_ssm_training.py`)

**Step 1 — Synthetic Data Generation**
Since real 30-day project metric sequences are rarely publicly available, the model is trained on **3,000 synthetic sequences** generated from four project trajectory archetypes:

| Trajectory | Pattern |
|---|---|
| **Healthy** | Low open rate, high close rate, short resolution time, small backlog |
| **Declining** | Accelerating bug creation, slowing resolution, growing backlog |
| **Recovering** | Fewer new bugs over time, accelerating resolution, shrinking backlog |
| **Volatile** | Weekly sinusoidal cycles in bug rates (sprint rhythm) |

Gaussian noise is added to each sequence to prevent overfitting.

**Step 2 — Health Score Label**
The ground-truth health score is computed as a **weighted composite formula**:
```
health_score = 0.45 × backlog_score + 0.35 × velocity_score + 0.20 × resolution_score
```
- `backlog_score`: 0 backlog = 1.0, 200+ backlog = 0.0
- `velocity_score`: positive when team closes more bugs than opens
- `resolution_score`: < 4 hours = 1.0, > 48 hours = 0.0

**Step 3 — Normalization**
Per-feature mean and standard deviation are computed from the training data and saved as `normalization_params.json` for use during Django inference.

**Step 4 — Training**
- Optimizer: **AdamW** (lr=3×10⁻⁴, weight_decay=1×10⁻⁴)
- Scheduler: **Cosine Annealing LR** (smooth decay to 1×10⁻⁶)
- Loss: **MSE** (Mean Squared Error, regression task)
- Gradient clipping at 1.0 (prevents exploding gradients in SSM scan)
- 80 epochs, batch size 64, 80/10/10 split

**Step 5 — Evaluation**
Reported metrics on the test set:
- MSE, RMSE, MAE
- R² (coefficient of determination)
- Predicted vs. Actual scatter plot

**Step 6 — Export**
```
smartproject_mamba.pth        ← model state_dict (PyTorch weights)
normalization_params.json     ← per-feature mean/std for inference
```

#### Django Integration (`strategic_ai/`)

**`services.py` — MambaPredictor singleton:**
Mirrors the Tactical AI pattern. Loads `smartproject_mamba.pth` once at startup. `predict_risk(sequence_tensor)` normalizes the input tensor and returns:
```json
{
  "health_score": 0.73,
  "risk_trajectory": "Stable",
  "momentum_state": "🟡 STABLE — Holding trajectory"
}
```

**`tasks.py` — Daily Celery Task:**
The Celery Beat scheduler triggers `aggregate_and_snapshot` daily at 00:05 UTC:
1. **`aggregate_daily_metrics()`** — Queries `BugReport` with Django ORM to compute that day's 5-feature row and upserts it into `DailyProjectMetric`.
2. **`calculate_project_momentum()`** — Fetches the past 30 `DailyProjectMetric` records, constructs a `(30, 5)` PyTorch tensor (zero-padded from the left if fewer than 30 records exist), runs Mamba inference, and saves a new `ProjectHealthSnapshot`.

---

## 5. Data Models (Database)

The system has 4 core Django models backed by PostgreSQL:

### `DeveloperProfile`
Stores engineer records:
- `name`, `email` (unique)
- `skills` (JSONField — e.g. `["Python", "Django", "PyTorch"]`)
- `active_workload` (count of assigned open bugs)
- `historical_resolution_rate` (average hours per bug)

### `BugReport`
The central entity:
- `title`, `description`, `raw_logs` (text input)
- `severity_predicted` (AI output: Low/Medium/High/Critical)
- `severity_confidence` (float 0–1)
- `priority_level` (business rule output: P0–P3)
- `status` (Open / In Progress / Resolved)
- `created_at`, `resolved_at` (for resolution time calculation)
- `assigned_developer` (FK to DeveloperProfile)

Composite database indexes on `(status, severity_predicted)` and `(assigned_developer, status)` for fast dashboard queries.

### `DailyProjectMetric`
One row per calendar day — the SSM input stream:
- `bugs_opened`, `bugs_closed`, `accumulated_backlog`
- `mean_resolution_time_hrs`, `defect_density`

### `ProjectHealthSnapshot`
One row per Mamba inference run — the SSM output:
- `health_score` (0.0–1.0)
- `risk_trajectory` (Stable/Accelerating/Degrading/Critical Risk)
- `state_vector` (JSONField — last day's features, for potential warm-start)
- `is_critical` property: True if score < 0.3 or trajectory = Critical Risk

---

## 6. REST API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/bugs/create/` | Submit a bug → DistilBERT classification → Developer assignment → Save |
| `GET` | `/api/bugs/` | Paginated bug list with optional `?severity=` and `?status=` filters |
| `GET` | `/api/project/health/` | 30-day metrics + latest SSM snapshot + resolution velocity |
| `GET` | `/api/developers/` | All developer profiles ordered by workload |
| `POST` | `/api/project/trigger-snapshot/` | Manually trigger Celery aggregation + SSM inference |

### Bug Creation Flow (POST `/api/bugs/create/`)
```
Client sends: { "title": "...", "description": "..." }
    ↓
Validate with BugReportCreateSerializer
    ↓
DistilBertPredictor.predict_bug_severity(title, description)
    → Returns: {severity, confidence, confidence_vector, priority}
    ↓
DeveloperProfile.objects.optimal_candidates(required_skill, max_workload=10)
    → Matches skill to severity, picks developer with lowest active workload
    ↓
BugReport.objects.create(...)  [atomic transaction]
DeveloperProfile.objects.update(active_workload += 1)
    ↓
Returns 201: {bug, ai_prediction, assigned_developer, message}
```

---

## 7. Frontend — Anti-Gravity Dashboard

The frontend (`frontend/templates/index.html`) is a single-page application built with **vanilla HTML/CSS/JavaScript** following the *"Anti-Gravity Cyber-Telemetry"* design system.

**Visual Identity:**
- **Deep Space background:** `#0B0F19`
- **Neon Cyan accents:** `#00F3FF`
- **Purple Glow highlights:** `#8A2BE2`
- **Danger Amber / Critical Red:** `#FF3366`

**Key UI Components:**
1. **Anti-Gravity Particle Canvas** — Interactive WebGL/Canvas background where particle nodes react to mouse cursor movements, changing gravity vectors based on real-time project risk level.
2. **Bug Submission Card** — Glassmorphism-styled floating modal. After text submission, real-time animations reflect the AI's severity classification.
3. **Strategic Momentum Dashboard** — Telemetry console visualizing the Mamba SSM output curve.
4. **Zero-G Gauge** — Animated gauge showing current project momentum direction (Stable / Accelerating / Free-Falling).
5. **Developer Workload Panel** — Live view of all developers and their current bug assignment counts.

---

## 8. Technology Stack Summary

| Layer | Technology |
|---|---|
| **Backend Framework** | Django 4.2 + Django REST Framework |
| **Database** | PostgreSQL 16 (SQLite for local dev) |
| **Task Queue** | Celery 5.4 + Redis 7 + django-celery-beat |
| **ML — Tactical AI** | PyTorch + Hugging Face Transformers (DistilBERT) |
| **ML — Strategic AI** | Pure-PyTorch SSM (Mamba-style) + NumPy + SciPy + einops |
| **Data Processing** | Pandas, scikit-learn |
| **Frontend** | HTML5 + Vanilla CSS + Vanilla JavaScript |
| **Containerization** | Docker + Docker Compose |
| **Dev Tools** | pytest-django, factory-boy, coverage |

---

## 9. Deployment Architecture

Docker Compose orchestrates 5 services:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│  PostgreSQL  │  │    Redis     │  │  Django Web App  │
│  (Port 5432) │  │  (Port 6379) │  │  (Port 8000)     │
└──────────────┘  └──────────────┘  └──────────────────┘
                                     ┌──────────────────┐
                                     │  Celery Worker   │
                                     │  (2 concurrency) │
                                     └──────────────────┘
                                     ┌──────────────────┐
                                     │  Celery Beat     │
                                     │  (Daily cron)    │
                                     └──────────────────┘
```

The `ml_artifacts/` directory is mounted as a shared Docker volume between the web app and Celery worker so both can access the pre-trained model weights.

---

## 10. Key Design Decisions & Technical Highlights

| Decision | Rationale |
|---|---|
| **Singleton pattern for ML models** | Loading a ~260MB DistilBERT model is expensive (~2–5s). Loading once at startup and sharing the in-memory instance across all requests makes inference fast (~15–40ms per request). |
| **Double-checked locking** | Ensures thread safety in multi-threaded WSGI (Gunicorn) environments without the performance cost of locking on every call. |
| **Pure-PyTorch SSM (not mamba-ssm)** | `mamba-ssm` requires Linux + CUDA kernel compilation — incompatible with Windows and CPU-only servers. The custom PyTorch implementation is functionally equivalent for 30-step windows. |
| **Celery for daily aggregation** | The SSM inference + database aggregation runs nightly as a background task to keep the API responsive. |
| **Idempotent metric aggregation** | `update_or_create()` in `aggregate_daily_metrics()` means the Celery task can be safely retried without creating duplicate records. |
| **Zero-padding for short history** | If fewer than 30 days of data exist (new project), the sequence is left-padded with zeros so the SSM model always receives a valid (30, 5) tensor. |
| **Stratified train/val/test splits** | Ensures class balance across all three splits, critical for multi-class classification with potentially imbalanced severity data. |

---

## 11. Summary: What Makes This System Novel

SmartProject AI is not a simple bug tracker. It combines:

1. **NLP-based tactical triage** — a fine-tuned Transformer model reads the *semantics* of a bug description (not just keywords) to understand its real severity.
2. **Temporal strategic forecasting** — a Mamba SSM model reads 30 days of project-wide patterns (not just one bug) to predict whether the project as a whole is healthy or spiraling.
3. **Automatic developer assignment** — skill-matching + workload-balancing assignment algorithm ensures critical bugs go to the right person with capacity.
4. **End-to-end ML deployment** — model training (Colab notebooks), inference server (Django singleton), and scheduled data pipeline (Celery) all work together in a production-ready Docker stack.

The two AI tracks are **complementary**: Tactical AI operates at the *individual bug level* (real-time, NLP), while Strategic AI operates at the *project level* (daily, time-series). Together, they give engineering managers both micro-level and macro-level intelligence in a single platform.
