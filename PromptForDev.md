Act as a Principal Machine Learning Engineer, Full-Stack Django Architect, and Creative Director. 

I am building "SmartProject AI," a software project health and bug triage platform with a dual-track AI architecture and an anti-gravity-themed interactive user experience.

- Track 1 (Tactical AI): Fine-tuning DistilBERT to classify bug report severity (Low, Medium, High, Critical) using text features.
- Track 2 (Strategic AI): Sequence modeling using Selective State Space Models (Mamba / SSM) on time-series project metrics (open/closed bugs, mean resolution time, defect density) to predict overall project failure/risk trajectory over time.
- Interface & UX Theme: "Anti-Gravity" visual identity featuring dark-mode aesthetics, zero-gravity floating UI physics, dynamic dynamic particle interactions, and futuristic dashboard telemetry.

Please generate the complete codebase, system architecture, and step-by-step implementation guide divided into the following structured execution environments:

----------------------------------------------------------------------
PART 1: GOOGLE COLAB NOTEBOOKS (Model Fine-Tuning & Weights Export)
----------------------------------------------------------------------

1. Tactical AI Pipeline (DistilBERT Multi-Class Severity Classifier):
   - Dependencies & Setup: Install `transformers`, `datasets`, `accelerate`, `scikit-learn`, and PyTorch with GPU check.
   - Preprocessing & Cleaning: 
     * Load bug dataset (e.g., Eclipse JDT / NASA MDP format) containing bug titles, descriptions, and severity labels.
     * Strip code clutter, stack traces, HTML tags, and normalize text into a combined `[Title + Description]` field.
     * Encode categorical severity levels (Low=0, Medium=1, High=2, Critical=3) and save label mapping JSON (`label2id`, `id2label`).
   - Tokenization & Dataset Builder: Use `DistilBertTokenizerFast` with max length padding and truncation. Split into 80/10/10 Train/Val/Test sets.
   - Training Loop: Fine-tune `distilbert-base-uncased` using Hugging Face `Trainer` on a GPU instance. Set hyperparameter configurations (learning rate, batch size, warmup steps, weight decay).
   - Validation Metrics: Compute Macro F1, Weighted F1, Precision, Recall, and output a confusion matrix array.
   - Serialization & Export: Export the model weights, tokenizer configs, and label mapping dictionary using `model.save_pretrained('./smartproject_distilbert')` and zip for deployment.

2. Strategic AI Pipeline (Mamba / Selective State Space Model):
   - Sequence Setup: Implement a PyTorch-compatible Selective State Space Model (SSM / Mamba block architecture) or structured state-space layer designed for sequence regression/classification.
   - Synthetic Sequence Generator: Generate daily time-series sequence tensors representing N days (e.g., 30-day lookback window) with metrics:
     * `bugs_opened_today`
     * `bugs_closed_today`
     * `mean_resolution_time`
     * `accumulated_backlog`
     * `defect_density`
   - Target Labeling: Generate a ground-truth "Project Risk Score" (0.0 to 1.0) or binary "Failure Trajectory Flag" based on backlog accumulation and resolution decay trends.
   - SSM Training: Train the sequence model over continuous time windows to capture project momentum and trajectory decay.
   - Export: Save model weights (`smartproject_mamba.pth`) and normalization parameters for live inference in Django.

----------------------------------------------------------------------
PART 2: DJANGO WEB APP ARCHITECTURE & INTEGRATION
----------------------------------------------------------------------

1. Data Models (`models.py`):
   - `BugReport`: Title, Description, Severity (AI Predicted), Priority, Status (Open/In Progress/Resolved), Created At, Resolved At, Assigned Developer.
   - `DeveloperProfile`: Name, Tech Stack, Historical Resolution Rate, Active Workload.
   - `DailyProjectMetric`: Date, Bugs Opened, Bugs Closed, Mean Resolution Time, Total Backlog, Defect Density.
   - `ProjectHealthSnapshot`: Date, Calculated Health Score, SSM Risk Trajectory, Momentum State (Accelerating, Stable, Degrading).

2. Local Inference Service (`tactical_ai/services.py`):
   - Implement a thread-safe Singleton class `DistilBertPredictor` that loads local fine-tuned model weights and tokenizer at Django startup (zero external API calls).
   - Write function `predict_bug_severity(title, description)` returning predicted severity class, index, and confidence score vector.

3. Daily Aggregation Engine (`strategic_ai/tasks.py`):
   - Write Django ORM aggregation functions to calculate daily metrics and update `DailyProjectMetric`.
   - Write service `calculate_project_momentum()` that constructs a tensor from the past 30 days of metrics, passes it through the local Mamba/SSM model, updates `ProjectHealthSnapshot`, and evaluates failure risk.

4. Views & REST APIs (`api/views.py`):
   - `POST /api/bugs/create/`: Accepts bug report text, passes payload to DistilBERT for real-time severity prediction, calculates priority via business rules matrix, assigns optimal developer, and stores bug.
   - `GET /api/project/health/`: Returns 30-day daily sequence metrics, current Mamba project health/risk trajectory score, and momentum status for front-end rendering.

----------------------------------------------------------------------
PART 3: ANTI-GRAVITY UI/UX BLUEPRINT & DESIGN SYSTEM
----------------------------------------------------------------------

1. Core Visual Concept & Aesthetic:
   - "Anti-Gravity Cyber-Telemetry": Dark-mode palette (Deep Space `#0B0F19`, Neon Cyan `#00F3FF`, Purple Glow `#8A2BE2`, Danger Amber `#FF3366`).
   - Zero-Gravity Mechanics: UI elements float smoothly along Z-indexes with floating card micro-interactions, subtle weightless hover states, and dynamic mouse-following particle vectors.

2. Component Breakdown:
   - Hero Section / Anti-Gravity Canvas: Interactive WebGL or Canvas background where floating node particles react to cursor movements. Nodes change gravity vectors (attract/repel) based on real-time project risk level.
   - Tactical Bug Submission Card: Floating submission modal with glassmorphism styling. Upon submitting text, display real-time gravity impact animations as the AI classifies severity.
   - Strategic Momentum Dashboard: A telemetry console visualizing the Mamba SSM output curve. Includes a "Zero-G Gauge" showing project momentum direction (Stable, Accelerating, or Free-Falling).

3. Step-by-Step Execution Plan & Technical Gotchas:
   - Provide development milestones covering backend API initialization, ML model artifact integration, and animation performance optimization.
   - List technical gotchas regarding multi-threading model loading in Django, PyTorch CPU memory footprints on production servers, and managing high-FPS canvas animation loops without frame drops.

Provide production-ready, highly modular, well-commented Python code, clean Django structures, and exact architectural instructions.
