Act as a Principal Database Architect and Senior Django Engineer.

I am building "SmartProject AI," a software project health platform featuring a dual-track AI architecture:
1. Tactical Track: Fine-tuned DistilBERT model predicting bug report severity (Low, Medium, High, Critical).
2. Strategic Track: Selective State Space Models (Mamba / SSM) running on daily time-series sequence data to predict project momentum and failure risk over time.

Please write the complete, production-ready database layer using Django ORM and PostgreSQL. Your output must cover the following four sections in detail:

----------------------------------------------------------------------
SECTION 1: DJANGO MODELS (`models.py`)
----------------------------------------------------------------------
Design clean, normalized models with proper data types, indexing, foreign keys, constraints, and string representations for:

1. DeveloperProfile:
   - Identifiers: name, email (unique).
   - Tech skills array (JSONField for skills like Python, Django, PyTorch).
   - Metrics: active_workload (count of assigned open bugs), historical_resolution_rate (avg hours per resolved bug).
   - Indexing: Index on active_workload for fast developer-matching lookups.

2. BugReport:
   - Text fields: title, description, raw_logs/stack_trace (optional).
   - AI Tactical Outputs: severity_predicted (Choices: Low, Medium, High, Critical), severity_confidence (float), priority_level (P0, P1, P2, P3).
   - Status & Lifecycle: status (Choices: Open, In Progress, Resolved), created_at, resolved_at.
   - Relationships: Foreign key to DeveloperProfile (null/blank allowed on submission).
   - Indexing: Composite index on (status, severity_predicted) and indexing on created_at for time-series queries.

3. DailyProjectMetric:
   - Date tracking: date (unique, indexed).
   - Daily counts: bugs_opened, bugs_closed, accumulated_backlog.
   - Aggregate metrics: mean_resolution_time_hrs (float), defect_density (float).
   - Constraints: Ensure count metrics are non-negative.

4. ProjectHealthSnapshot:
   - Timestamps: timestamp (auto_now_add, indexed).
   - SSM Output: health_score (float, 0.0 to 1.0), risk_trajectory (Choices: Stable, Accelerating, Degrading, Critical Risk).
   - Sequence State: state_vector (JSONField to store hidden state vectors for SSM continuity).

----------------------------------------------------------------------
SECTION 2: CUSTOM ORM QUERYSETS & MANAGERS (`managers.py`)
----------------------------------------------------------------------
Write custom Django Managers and QuerySets with optimized database methods:
- `BugReportQuerySet`: Methods for `.unassigned()`, `.high_severity()`, and `.get_resolution_velocity(days=30)`.
- `DeveloperQuerySet`: Method for `.optimal_candidates(required_skill, max_workload=5)` to support the Developer Match algorithm.
- `DailyProjectMetricQuerySet`: Method `.get_lookback_sequence(days=30)` that extracts ordered numerical features ready to be converted directly into a PyTorch tensor for the Mamba SSM model.

----------------------------------------------------------------------
SECTION 3: DATABASE SEED SCRIPT (`seed_db.py`)
----------------------------------------------------------------------
Provide a standalone Django management command or Python script using `factory_boy` / standard Django ORM to generate realistic dummy seed data:
- 5 Developer Profiles with realistic skills and workloads.
- 50 Bug Reports with mixed titles, descriptions, severities, and statuses over the past 60 days.
- 30 consecutive days of `DailyProjectMetric` records simulating a declining project trajectory (increasing backlog and rising mean resolution time) to test the Mamba risk spike.

----------------------------------------------------------------------
SECTION 4: RAW SQL PERFORMANCE & INDEXING ADVICE
----------------------------------------------------------------------
- Identify potential N+1 query pitfalls in this schema and show how to use `select_related` / `prefetch_related` for API endpoints.
- List recommended PostgreSQL specific indexes (e.g., B-Tree, GIN for JSONFields) to ensure high query performance as bug counts scale into hundreds of thousands.

Ensure all code includes clear inline comments, proper import statements, and typing annotations.
