# Gemini Enterprise CLI — FinOps-First Bootstrap & Scale Edition

A field-tested community contribution proposed for the official **Google Gemini Enterprise CLI** repository. This branch equips Gemini CLI with **99 FinOps-governed Google Cloud skills** and documents empirical billing discoveries from 4 months of production testing (August–November).

---

## 1. The 4-Month Billing Benchmark: From $2,158/mo Idle Burn to $296/mo (and $179 Net Credit Draw) at 19h/Day Production

When connecting standard enterprise cloud templates out-of-the-box, always-on services silently drain startup credits even when no code is running. By re-architecting around **Cloud Run + Cloud Storage + Cloud Build**, **Off-Peak Regional Batching**, and **3-Stage Gemini Model Routing**, we achieved an **86% drop in gross usage cost** and a **91% drop in net promotional credit consumption**:

| Billing Period | Workload Activity | Gross Usage Cost | Automatic Discounts (`Spending` + `Subscription`) | Net Promotional Credits Consumed | Key Architectural & FinOps Shift |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **August** | Initial repo connection (Default Templates) | **$1,922.33** | `-4.96%` (-$95.35) | **-$1,826.98** | Always-on defaults: Compute Engine ($593.40), AlloyDB ($477.36), Cloud Composer ($432.58), Cloud Workstations ($208.89), GKE ($76.80), Spanner ($54.31). |
| **September** | Zero computer touch (Weeks of idle burn) | **$2,158.83** | `-4.99%` (-$107.73) | **-$2,051.10** | Idle drain: Compute Engine jumped to $1,344.45 (+127%), Composer ($411.89), Workstations ($145.94), Spanner ($88.11), Networking ($43.23). |
| **October** | Mid-month migration to Serverless & Off-Peak | **$1,076.46** | **`-21.69%`** (-$233.50) | **-$842.95** | Decommissioned AlloyDB (-100% to $0.01), cut Compute (-79%), Composer (-68%), Spanner (-68%), GKE (-67%); moved state to **Cloud Storage ($0.42)** & **Cloud Run ($0.11)**. |
| **November (1–15)** | **19+ hours/day intensive multi-agent production** | **$296.75** | **`-39.67%`** (-$117.71) | **-$179.05** | **6 production Cloud Run services costing $0.18 total** + GCS ($0.42) + Off-Peak Night Batching + 3-Stage Gemini Routing. **Nearly 40% discounted before touching credits!** |

---

## 2. Six Field-Tested FinOps Discoveries Every Startup Should Know

### Discovery #1: Cheap Storage (GCS) + Regional Night-Window Batching Unlocks Up to ~40–50% Automatic Billing Reductions (Before Touching Credits!)
- **What we saw in the data:** In August and September, running standard always-on compute yielded only a `-4.96%` spending discount, forcing **95% of the bill** to eat directly into startup promotional credits.
- **The Breakthrough:** In November, by storing all intermediate state/snapshots in cheap **Cloud Storage (`$0.42/mo`)** and dispatching heavy agent workloads during the **local night window of the target GCP region (`us-central1` Iowa night = Israel daytime)** using **Batch Jobs & Context Caching**, effective automatic billing reductions jumped to **`-39.67%`** (`-32.17%` spending-based discounts + `-7.5%` subscription credits) — cutting net credit draw to just **$179.05** during 19-hour/day heavy work!
- **Why Regional Night Scheduling Works Twice:**
  1. It qualifies batch/cached workloads for massive automatic tier discounts that do not deduct from your startup credits.
  2. Running during `us-central1` off-peak night hours eliminates `429 Resource Exhausted` quota collisions and mid-generation retry storms (which otherwise bill you twice for failed partial context windows).

### Discovery #2: 3-Stage Task-Tiered Gemini Routing (Zero-Thinking Pull -> Heavy Execution -> Independent Audit)
Never use a heavyweight reasoning model with `HIGH` thinking just to read files or fetch context:
1. **Stage 1 — Ingestion & Context Pull (Zero Thinking):** Use `Gemini 3.8 Flash` / `Omni Flash` **with thinking disabled (`thinking_budget=0`)** to pull files from GCS, map schemas, and filter relevant snippets at minimal cost and sub-second latency.
2. **Stage 2 — Core Synthesis & Execution:** Pass **only** the distilled context into heavy models (`4 Carbon High` / `Pro`) for complex code and architectural generation.
3. **Stage 3 — Deterministic Audit Gate:** Run an independent verification pass (`4 Argon High` + `vertex-ai-evaluation-service`) to validate outputs before writing to canonical storage.

### Discovery #3: Zero-Token SHA-256 Delta Gate & Grounding Gatekeeper
- **The Hidden Drain:** Enabling live Google Search Grounding or re-running agents on unchanged upstream data silently multiplies API bills.
- **The Fix:** Compute a deterministic `SHA-256` hash of the input state against the latest GCS snapshot (`snapshots/<agent>_latest.json`). If there is no significant delta, return the verified GCS snapshot for **$0.00 and 0 tokens**. Only invoke live search grounding when explicit telemetry/grounding triggers are present.

### Discovery #4: Replace Always-On Vector DBs ($500+/mo) with Vertex AI RAG Engine (`Serverless Mode`) + GCS
- **The Trap:** Running `AlloyDB` ($477.36/mo in August) + `Cloud Spanner` ($88.11/mo in September) + dedicated `Vertex AI Search` ($30.00/mo idle minimum) costs ~$600/month before a single query runs.
- **The Fix:** Use **Vertex AI RAG Engine in `Serverless mode`** backed by **Cloud Storage buckets ($0.42/mo)** and local/serverless Parquet/DuckDB indexes for early-stage development. AlloyDB cost dropped **-100% (to $0.01)**.

### Discovery #5: Eliminate "Silent Container Taxes" (`Vulnerability Scanning` & `Cloud Workstations`)
- **Cloud Workstations** burned **$208.89/mo in August** and **$145.94/mo in September** while completely untouched due to persistent control-plane and disk billing. Switching to **Cloud Shell + stateless Cloud Build** saved >$200/mo immediately.
- **Container Registry Vulnerability Scanning** automatically charged **$19.50 in October** just scanning intermediate build images. Using **Cloud Run Source Deploys** (`$0.18/mo` across 6 active microservices) and slim packaging reduced `Artifact Registry` billing to **$0.07/mo**.

### Discovery #6: Stage-Aware GCP Product Selection (How to Use Enterprise GCP Tools FinOps-Correctly)
Every Google Cloud product has the right stage and configuration. Do not abandon enterprise tools—configure them with strict FinOps guardrails when your scale justifies them:

| Workload Domain | Stage 1: Day-1 Lean Bootstrap (Zero Idle Cost) | Stage 2: Enterprise Scale (FinOps-Governed Capacity) |
| :--- | :--- | :--- |
| **Compute & Microservices** | **Cloud Run Services & Jobs** (`min-instances=0`, **$0.18/mo for 6 services**) | **GKE Autopilot** (Pod-level billing, eBPF dataplane, zero idle node tax) |
| **Workflow Orchestration** | **Cloud Workflows** + **Cloud Scheduler** + **Eventarc** + **Cloud Build** | **Cloud Composer (Airflow)** with strict auto-scaling bounds |
| **Operational State & DB** | **Cloud Storage (GCS, $0.42/mo)** + **Firestore** + **DuckDB/Parquet** | **AlloyDB** / **Cloud Spanner** / **Bigtable** with right-sized compute & schema guards |
| **Analytics & AI Retrieval** | **BigQuery On-Demand** + **RAG Engine (Serverless Mode)** + **Context Caching** | **BigQuery Slot Reservations** + **Continuous Queries** + **Vertex AI Vector Search** |

---

## 3. Complete Google Cloud FinOps Skill Catalog (99 Skills)

### Stage 1: Zero-Idle Bootstrap & Serverless Foundation (73 Skills)
- [`skills/artifact-registry-containers.SKILL.md`](skills/artifact-registry-containers.SKILL.md)
- [`skills/bigquery-data-transfer-service.SKILL.md`](skills/bigquery-data-transfer-service.SKILL.md)
- [`skills/bigquery-sql.SKILL.md`](skills/bigquery-sql.SKILL.md)
- [`skills/building-data-apps.SKILL.md`](skills/building-data-apps.SKILL.md)
- [`skills/cloud-armor-waf-rules.SKILL.md`](skills/cloud-armor-waf-rules.SKILL.md)
- [`skills/cloud-build-ci-pipelines.SKILL.md`](skills/cloud-build-ci-pipelines.SKILL.md)
- [`skills/cloud-dlp-deidentification.SKILL.md`](skills/cloud-dlp-deidentification.SKILL.md)
- [`skills/cloud-logging-queries.SKILL.md`](skills/cloud-logging-queries.SKILL.md)
- [`skills/cloud-monitoring-alerts.SKILL.md`](skills/cloud-monitoring-alerts.SKILL.md)
- [`skills/cloud-run-jobs-batch.SKILL.md`](skills/cloud-run-jobs-batch.SKILL.md)
- [`skills/cloud-run-services-deploy.SKILL.md`](skills/cloud-run-services-deploy.SKILL.md)
- [`skills/cloud-scheduler-cron.SKILL.md`](skills/cloud-scheduler-cron.SKILL.md)
- [`skills/cloud-trace-opentelemetry.SKILL.md`](skills/cloud-trace-opentelemetry.SKILL.md)
- [`skills/cloud-workflows-orchestration.SKILL.md`](skills/cloud-workflows-orchestration.SKILL.md)
- [`skills/dart-async-concurrency.SKILL.md`](skills/dart-async-concurrency.SKILL.md)
- [`skills/dart-cli-tooling.SKILL.md`](skills/dart-cli-tooling.SKILL.md)
- [`skills/dart-ffi-native-bindings.SKILL.md`](skills/dart-ffi-native-bindings.SKILL.md)
- [`skills/dart-isolates-worker.SKILL.md`](skills/dart-isolates-worker.SKILL.md)
- [`skills/dart-json-serialization.SKILL.md`](skills/dart-json-serialization.SKILL.md)
- [`skills/dart-use-doc-examples.SKILL.md`](skills/dart-use-doc-examples.SKILL.md)
- [`skills/dart-write-documentation.SKILL.md`](skills/dart-write-documentation.SKILL.md)
- [`skills/dataform-bigquery.SKILL.md`](skills/dataform-bigquery.SKILL.md)
- [`skills/dbt-bigquery.SKILL.md`](skills/dbt-bigquery.SKILL.md)
- [`skills/discovering-gcp-data-assets.SKILL.md`](skills/discovering-gcp-data-assets.SKILL.md)
- [`skills/eventarc-gcs-triggers.SKILL.md`](skills/eventarc-gcs-triggers.SKILL.md)
- [`skills/finding-google-skills.SKILL.md`](skills/finding-google-skills.SKILL.md)
- [`skills/firebase-app-check.SKILL.md`](skills/firebase-app-check.SKILL.md)
- [`skills/firebase-auth-basics.SKILL.md`](skills/firebase-auth-basics.SKILL.md)
- [`skills/firebase-cloud-functions.SKILL.md`](skills/firebase-cloud-functions.SKILL.md)
- [`skills/firebase-crashlytics-basics.SKILL.md`](skills/firebase-crashlytics-basics.SKILL.md)
- [`skills/firebase-emulator-suite.SKILL.md`](skills/firebase-emulator-suite.SKILL.md)
- [`skills/firebase-firestore-basics.SKILL.md`](skills/firebase-firestore-basics.SKILL.md)
- [`skills/firebase-hosting-basics.SKILL.md`](skills/firebase-hosting-basics.SKILL.md)
- [`skills/firebase-performance-monitoring.SKILL.md`](skills/firebase-performance-monitoring.SKILL.md)
- [`skills/firebase-remote-config-basics.SKILL.md`](skills/firebase-remote-config-basics.SKILL.md)
- [`skills/firebase-storage-security-rules.SKILL.md`](skills/firebase-storage-security-rules.SKILL.md)
- [`skills/flutter-add-widget-preview.SKILL.md`](skills/flutter-add-widget-preview.SKILL.md)
- [`skills/flutter-add-widget-test.SKILL.md`](skills/flutter-add-widget-test.SKILL.md)
- [`skills/flutter-architecture-clean.SKILL.md`](skills/flutter-architecture-clean.SKILL.md)
- [`skills/flutter-dart-collect-coverage.SKILL.md`](skills/flutter-dart-collect-coverage.SKILL.md)
- [`skills/flutter-dart-resolve-package-conflicts.SKILL.md`](skills/flutter-dart-resolve-package-conflicts.SKILL.md)
- [`skills/flutter-dart-run-static-analysis.SKILL.md`](skills/flutter-dart-run-static-analysis.SKILL.md)
- [`skills/flutter-dart-use-pattern-matching.SKILL.md`](skills/flutter-dart-use-pattern-matching.SKILL.md)
- [`skills/flutter-integration-testing.SKILL.md`](skills/flutter-integration-testing.SKILL.md)
- [`skills/flutter-navigation-routing.SKILL.md`](skills/flutter-navigation-routing.SKILL.md)
- [`skills/flutter-performance-profiling.SKILL.md`](skills/flutter-performance-profiling.SKILL.md)
- [`skills/flutter-platform-channels.SKILL.md`](skills/flutter-platform-channels.SKILL.md)
- [`skills/flutter-state-management.SKILL.md`](skills/flutter-state-management.SKILL.md)
- [`skills/flutter-web-wasm-build.SKILL.md`](skills/flutter-web-wasm-build.SKILL.md)
- [`skills/gcs-security-assessment-audit.SKILL.md`](skills/gcs-security-assessment-audit.SKILL.md)
- [`skills/gcs-security-assessment.SKILL.md`](skills/gcs-security-assessment.SKILL.md)
- [`skills/gcs-security-posture-baseline.SKILL.md`](skills/gcs-security-posture-baseline.SKILL.md)
- [`skills/gemini-omni-flash-api.SKILL.md`](skills/gemini-omni-flash-api.SKILL.md)
- [`skills/google-cloud-auth-verification.SKILL.md`](skills/google-cloud-auth-verification.SKILL.md)
- [`skills/google-cloud-storage-basics.SKILL.md`](skills/google-cloud-storage-basics.SKILL.md)
- [`skills/google-cloud-storage-bucket-architect.SKILL.md`](skills/google-cloud-storage-bucket-architect.SKILL.md)
- [`skills/google-cloud-storage-fuse.SKILL.md`](skills/google-cloud-storage-fuse.SKILL.md)
- [`skills/iam-least-privilege-bindings.SKILL.md`](skills/iam-least-privilege-bindings.SKILL.md)
- [`skills/kms-cmek-encryption.SKILL.md`](skills/kms-cmek-encryption.SKILL.md)
- [`skills/looker-studio-dashboards.SKILL.md`](skills/looker-studio-dashboards.SKILL.md)
- [`skills/pubsub-event-streaming.SKILL.md`](skills/pubsub-event-streaming.SKILL.md)
- [`skills/secret-manager-integration.SKILL.md`](skills/secret-manager-integration.SKILL.md)
- [`skills/terraform-gcp-serverless.SKILL.md`](skills/terraform-gcp-serverless.SKILL.md)
- [`skills/vertex-ai-agent-builder-adk.SKILL.md`](skills/vertex-ai-agent-builder-adk.SKILL.md)
- [`skills/vertex-ai-batch-prediction.SKILL.md`](skills/vertex-ai-batch-prediction.SKILL.md)
- [`skills/vertex-ai-context-caching.SKILL.md`](skills/vertex-ai-context-caching.SKILL.md)
- [`skills/vertex-ai-embeddings-api.SKILL.md`](skills/vertex-ai-embeddings-api.SKILL.md)
- [`skills/vertex-ai-evaluation-service.SKILL.md`](skills/vertex-ai-evaluation-service.SKILL.md)
- [`skills/vertex-ai-function-calling.SKILL.md`](skills/vertex-ai-function-calling.SKILL.md)
- [`skills/vertex-ai-gemini-sdk.SKILL.md`](skills/vertex-ai-gemini-sdk.SKILL.md)
- [`skills/vertex-ai-model-armor.SKILL.md`](skills/vertex-ai-model-armor.SKILL.md)
- [`skills/vpc-service-controls.SKILL.md`](skills/vpc-service-controls.SKILL.md)
- [`skills/workload-identity-federation.SKILL.md`](skills/workload-identity-federation.SKILL.md)

### Stage 2: High-Scale Enterprise GCP Products — FinOps-Governed (26 Skills)
- [`skills/alloydb-ai-vector-query.SKILL.md`](skills/alloydb-ai-vector-query.SKILL.md)
- [`skills/alloydb-postgres-admin.SKILL.md`](skills/alloydb-postgres-admin.SKILL.md)
- [`skills/bigquery-continuous-queries.SKILL.md`](skills/bigquery-continuous-queries.SKILL.md)
- [`skills/bigquery-graph-author.SKILL.md`](skills/bigquery-graph-author.SKILL.md)
- [`skills/bigquery-ml-models.SKILL.md`](skills/bigquery-ml-models.SKILL.md)
- [`skills/bigquery-slot-reservations.SKILL.md`](skills/bigquery-slot-reservations.SKILL.md)
- [`skills/bigquery-vector-search.SKILL.md`](skills/bigquery-vector-search.SKILL.md)
- [`skills/bigtable-timeseries-schema.SKILL.md`](skills/bigtable-timeseries-schema.SKILL.md)
- [`skills/cloud-composer-airflow.SKILL.md`](skills/cloud-composer-airflow.SKILL.md)
- [`skills/cloud-spanner-schema.SKILL.md`](skills/cloud-spanner-schema.SKILL.md)
- [`skills/cloud-sql-postgres-basics.SKILL.md`](skills/cloud-sql-postgres-basics.SKILL.md)
- [`skills/dataflow-streaming-beam.SKILL.md`](skills/dataflow-streaming-beam.SKILL.md)
- [`skills/dataproc-spark-clusters.SKILL.md`](skills/dataproc-spark-clusters.SKILL.md)
- [`skills/gcp-data-pipelines.SKILL.md`](skills/gcp-data-pipelines.SKILL.md)
- [`skills/gcp-dataflow.SKILL.md`](skills/gcp-dataflow.SKILL.md)
- [`skills/gcp-pipeline-orchestration.SKILL.md`](skills/gcp-pipeline-orchestration.SKILL.md)
- [`skills/gcp-pipeline-resource-provisioning.SKILL.md`](skills/gcp-pipeline-resource-provisioning.SKILL.md)
- [`skills/gcp-spark-troubleshooting.SKILL.md`](skills/gcp-spark-troubleshooting.SKILL.md)
- [`skills/gcp-spark.SKILL.md`](skills/gcp-spark.SKILL.md)
- [`skills/gke-autopilot-clusters.SKILL.md`](skills/gke-autopilot-clusters.SKILL.md)
- [`skills/gke-ebpf-dataplane-v2.SKILL.md`](skills/gke-ebpf-dataplane-v2.SKILL.md)
- [`skills/memorystore-redis-caching.SKILL.md`](skills/memorystore-redis-caching.SKILL.md)
- [`skills/vertex-ai-feature-store.SKILL.md`](skills/vertex-ai-feature-store.SKILL.md)
- [`skills/vertex-ai-pipelines-kubeflow.SKILL.md`](skills/vertex-ai-pipelines-kubeflow.SKILL.md)
- [`skills/vertex-ai-vector-search.SKILL.md`](skills/vertex-ai-vector-search.SKILL.md)
- [`skills/vertex-ai-workbench-notebooks.SKILL.md`](skills/vertex-ai-workbench-notebooks.SKILL.md)

---
*Researched, architected, and field-tested in production by **Anat Lerner** (@AnatAdapaAi).*
