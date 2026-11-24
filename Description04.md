
==================================================
14. BACKEND API AND INTEGRATIONS
==================================================

Use Django REST Framework for API development.

Organize endpoints by functional modules.

SUGGESTED API GROUPS:

AUTHENTICATION:
- User authentication.
- User registration.
- Project membership.
- Role-based permissions.

PROJECTS:
- Create project.
- Update project.
- Retrieve project.
- List projects.
- Project analytics.

REPOSITORIES:
- Connect GitHub.
- List repositories.
- Synchronize repository.
- Retrieve repository information.
- Trigger code indexing.
- Retrieve indexing status.

BUG MANAGEMENT:
- Create bug.
- List bugs.
- Update bug.
- Retrieve bug details.
- Change severity.
- Assign developer.
- Update status.

AI DEBUGGING:
- Start investigation.
- Retrieve investigation status.
- Retrieve root cause analysis.
- Generate patch.
- Approve patch.
- Reject patch.
- Trigger testing.
- Retrieve testing results.
- Request pull request creation.

CODE INTELLIGENCE:
- Search code.
- Retrieve relevant files.
- Retrieve dependency information.
- Retrieve module analysis.

PROJECT HEALTH:
- Retrieve current health.
- Retrieve historical metrics.
- Trigger health analysis.
- Retrieve risk warnings.

CODE RISK:
- Retrieve risky files.
- Retrieve file-level analysis.
- Retrieve risk history.

AGENT MANAGEMENT:
- List agents.
- Start agent workflow.
- Retrieve agent runs.
- Cancel eligible workflows.
- Retrieve execution logs.

PROJECT MANAGEMENT:
- Retrieve sprint analytics.
- Retrieve workload information.
- Retrieve recommendations.
- Generate project reports.

Use asynchronous background processing
for long-running agent workflows.

Use Celery and Redis where appropriate.

Use WebSockets or Server-Sent Events
for real-time progress updates.

Use webhooks to receive GitHub events.

All APIs should have:

- Authentication.
- Authorization.
- Input validation.
- Error handling.
- Rate limiting where appropriate.
- Pagination.
- Logging.
- API documentation.

==================================================
15. RECOMMENDED TECHNOLOGY STACK
==================================================

Do not replace existing technologies
without a clear technical reason.

BACKEND:
- Django 4.2+
- Django REST Framework
- PostgreSQL
- Django ORM

AGENTIC AI:
- LangGraph
- LLM provider abstraction
- Tool calling
- Structured outputs

LLM PROVIDERS:
Support configurable providers such as:
- OpenAI
- Anthropic
- Google Gemini

Avoid tightly coupling business logic
to a single LLM provider.

CODEBASE INTELLIGENCE:
- Tree-sitter or language-specific parsers
- Embedding models
- pgvector or Qdrant
- Hybrid retrieval

BACKGROUND TASKS:
- Celery
- Redis
- Celery Beat

CODE EXECUTION:
- Docker sandbox
- Pytest
- Static analysis tools
- GitHub Actions

GITHUB INTEGRATION:
- GitHub REST API
- GitHub Webhooks
- GitHub App authentication

FRONTEND:

The existing frontend is vanilla HTML,
CSS, and JavaScript.

Evaluate whether to migrate to React
with TypeScript.

If migrating, do it incrementally and
preserve existing functionality.

VISUALIZATION:
- Chart.js or ECharts
- Interactive dependency graphs
- Real-time agent activity

DEPLOYMENT:
- Docker
- Docker Compose
- Environment-based configuration

==================================================
16. SYSTEM RELIABILITY AND AI QUALITY
==================================================

The platform must not prioritize AI complexity
over correctness.

Implement evaluation mechanisms for each
AI component.

DISTILBERT:

Evaluate:
- Macro F1.
- Weighted F1.
- Per-class precision and recall.
- Confusion matrix.
- Confidence calibration.

Use real bug reports for meaningful
performance evaluation.

Synthetic fallback data must be clearly
identified in model documentation.

MAMBA/SSM:

Evaluate:
- MAE.
- RMSE.
- R-squared.
- Temporal generalization.
- Baseline comparisons.

Do not treat synthetic-data performance
as evidence of real-world forecasting ability.

AGENT EVALUATION:

Evaluate:

- Root cause identification accuracy.
- Correctness of generated patches.
- Test pass rate.
- Regression introduction rate.
- Human acceptance rate.
- Unnecessary modification rate.
- Average resolution time.
- Cost per investigation.

RAG EVALUATION:

Evaluate:

- Retrieval precision.
- Retrieval recall.
- Context relevance.
- Source citation correctness.
- Repository synchronization accuracy.

OBSERVABILITY:

Monitor:

- API response times.
- Agent execution time.
- LLM token consumption.
- Model inference latency.
- Background task failures.
- Database performance.
- Test execution failures.

Maintain detailed error logs.

==================================================
17. IMPLEMENTATION ROADMAP
==================================================

Do not attempt to implement every feature
simultaneously.

Develop the platform incrementally.

PHASE 1: EXISTING SYSTEM AUDIT

- Inspect the entire repository.
- Understand the existing architecture.
- Verify existing API endpoints.
- Verify DistilBERT integration.
- Verify Mamba/SSM integration.
- Inspect database models.
- Test Celery and Redis.
- Inspect the frontend.
- Identify broken or incomplete features.
- Document technical debt.

Deliverable:
A complete implementation and migration plan.

PHASE 2: FOUNDATION AND INTEGRATION

- Improve project structure.
- Implement project and repository management.
- Implement GitHub authentication.
- Connect repositories.
- Implement webhook handling.
- Introduce agent execution tracking.
- Implement audit logging.

Deliverable:
A working platform with GitHub integration.

PHASE 3: CODEBASE INTELLIGENCE

- Implement source code parsing.
- Implement code chunking.
- Generate embeddings.
- Configure vector storage.
- Implement hybrid retrieval.
- Implement repository synchronization.
- Build code search functionality.

Deliverable:
A searchable repository intelligence system.

PHASE 4: AUTONOMOUS DEBUGGING

- Implement bug investigation agent.
- Implement root cause analysis.
- Implement code analysis agent.
- Implement fix generation.
- Implement patch visualization.
- Implement sandbox testing.
- Implement independent code review.
- Implement human approval.
- Implement GitHub pull request creation.

Deliverable:
A working end-to-end autonomous debugging
workflow.

PHASE 5: PREDICTIVE INTELLIGENCE

- Improve existing Mamba model evaluation.
- Integrate additional project metrics.
- Implement code risk analysis.
- Build risk visualization.
- Implement early warning mechanisms.

Deliverable:
A predictive project and code intelligence system.

PHASE 6: PROJECT MANAGEMENT AGENT

- Implement sprint analysis.
- Implement bottleneck detection.
- Improve developer recommendations.
- Implement conversational project analytics.
- Implement automated reporting.

Deliverable:
An intelligent project management assistant.

PHASE 7: UI REDESIGN

- Redesign the overview dashboard.
- Build debugging workspace.
- Build repository intelligence pages.
- Build code risk visualization.
- Build project health analytics.
- Build agent activity monitoring.
- Implement responsive layouts.

Deliverable:
A complete engineering intelligence dashboard.

PHASE 8: TESTING AND DEPLOYMENT

- Write backend tests.
- Write frontend tests.
- Test agent workflows.
- Test GitHub integration.
- Test execution isolation.
- Perform security testing.
- Optimize performance.
- Improve Docker deployment.
- Prepare documentation.

Deliverable:
A stable, documented, deployable product.

==================================================
18. DEVELOPMENT RULES FOR THE AI CODING AGENT
==================================================

Follow these rules throughout development.

1. FIRST UNDERSTAND THE EXISTING PROJECT.

Do not immediately start rewriting files.

Inspect the existing repository and understand
its architecture, dependencies, and workflows.

2. PRESERVE WORKING FUNCTIONALITY.

Do not remove existing ML models,
database functionality, or API endpoints
without a justified migration plan.

3. BUILD INCREMENTALLY.

Implement one module at a time.

Ensure that each module works before
moving to the next phase.

4. DO NOT FAKE FUNCTIONALITY.

Never create buttons that do nothing.

Never display fabricated AI results.

Never simulate successful repository operations.

If a feature requires an API key or external
integration, clearly indicate its configuration
and connection status.

5. USE REAL DATA.

Do not hardcode dashboard metrics.

Use actual database records and real API data.

Clearly label sample or synthetic data.

6. KEEP AGENTS GROUNDED.

Agents should use retrieved repository
context and actual project information.

They must provide evidence for technical
conclusions whenever possible.

7. HANDLE FAILURES.

Implement retries where appropriate.

Handle unavailable LLM providers,
GitHub API failures, timeouts, and
background task failures.

Never mark an unsuccessful task as completed.

8. PROTECT SOURCE CODE.

Never expose repository secrets.

Never execute untrusted code directly
on the host machine.

Use isolated execution environments.

9. REQUIRE HUMAN APPROVAL.

Code modifications, commits, pull requests,
merges, and deployments must follow
the permission system.

10. DOCUMENT EVERYTHING.

Maintain documentation for:

- Architecture.
- Database models.
- API endpoints.
- Agent workflows.
- ML models.
- Environment variables.
- Deployment.
- Testing.

==================================================
19. FINAL PRODUCT EXPECTATION
==================================================

The finished SmartProject AI 2.0 should
provide a unified platform where software
engineering teams can:

1. Connect their GitHub repositories.
2. Monitor software project health.
3. Automatically classify incoming bugs.
4. Investigate bugs using repository context.
5. Identify likely root causes.
6. Generate proposed code fixes.
7. Execute automated tests.
8. Review generated modifications.
9. Create approved pull requests.
10. Identify risky code modules.
11. Predict project health problems.
12. Monitor developer workloads.
13. Detect sprint bottlenecks.
14. Receive intelligent recommendations.
15. Interact with an AI project manager.
16. Monitor all AI agent operations.

The ultimate objective is to create a system
that doesn't simply tell developers what
is wrong.

It helps them understand why something
is wrong, what should be done, and how
to verify that the solution works.

FINAL PRODUCT POSITIONING:

SmartProject AI is an Agentic Software
Engineering Intelligence Platform that
combines predictive machine learning,
repository-level code intelligence, and
autonomous AI agents to improve software
development, debugging, and project execution.

Build a reliable engineering product,
not a collection of disconnected AI features.
