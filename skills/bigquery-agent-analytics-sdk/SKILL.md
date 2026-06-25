---
name: bigquery-agent-analytics-sdk
description: Analyze, evaluate, and curate AI agent traces stored in BigQuery with observability dashboards, LLM-as-Judge evaluation, trajectory matching, and Agent Context Graph decision-trace extraction at scale.
triggers:
  - analyze agent traces in BigQuery
  - evaluate AI agent performance with LLM judges
  - extract agent decision traces with context graph
  - set up agent observability in BigQuery
  - trace agent execution paths and trajectories
  - detect drift in agent behavior over time
  - build agent evaluation pipelines with BigQuery
  - materialize agent context graphs for analysis
---

# BigQuery Agent Analytics SDK

> Skill by [ara.so](https://ara.so) — Data Skills collection

The BigQuery Agent Analytics SDK is an open-source Python toolkit for analyzing AI agent telemetry stored in BigQuery. It provides observability (trace reconstruction, DAG visualization), evaluation (LLM-as-Judge, trajectory matching, multi-trial pass@k), and advanced analytics (Agent Context Graph for decision-trace extraction, drift detection, memory service). Built on top of BigQuery Agent Analytics, it's designed for ML engineers running agents in production who need to measure quality, understand behavior, and detect regressions at scale.

## Installation

```bash
# Core SDK
pip install bigquery-agent-analytics

# With LLM judge support
pip install bigquery-agent-analytics[llm]

# With BigFrames support
pip install bigquery-agent-analytics[bigframes]

# All features
pip install bigquery-agent-analytics[llm,bigframes]
```

**Prerequisites:**
- Python 3.10+
- Google Cloud project with BigQuery enabled
- Agent traces in BigQuery (via [ADK BigQuery Trace Exporter](https://github.com/google/adk-python/tree/main/contributing/extensions/bigquery_trace_exporter))
- `GOOGLE_APPLICATION_CREDENTIALS` or `gcloud auth application-default login` configured

## Core Client API

### Initialize Client

```python
from bigquery_agent_analytics import Client

# Basic initialization
client = Client(
    project_id="my-gcp-project",
    dataset_id="agent_analytics"
)

# With custom location
client = Client(
    project_id="my-project",
    dataset_id="agent_traces",
    location="US"
)
```

### Retrieve and Visualize Traces

```python
# Get a single trace
trace = client.get_trace("trace-abc-123")

# Render as ASCII DAG
trace.render()

# Get trace as dictionary
trace_dict = trace.to_dict()

# Get all traces for a session
traces = client.get_traces_for_session("session-xyz-456")
for trace in traces:
    print(f"Trace: {trace.trace_id}, Events: {len(trace.events)}")

# Query traces by time range
from datetime import datetime, timedelta
end_time = datetime.utcnow()
start_time = end_time - timedelta(hours=24)

recent_traces = client.get_traces(
    start_time=start_time,
    end_time=end_time,
    limit=100
)
```

### Event Semantics

```python
from bigquery_agent_analytics.event_semantics import (
    is_agent_start,
    is_agent_end,
    is_tool_invocation,
    is_llm_call,
    get_input_text,
    get_output_text
)

for event in trace.events:
    if is_agent_start(event):
        print(f"Agent started: {get_input_text(event)}")
    elif is_tool_invocation(event):
        print(f"Tool called: {event.get('tool_name')}")
    elif is_llm_call(event):
        print(f"LLM invoked: {event.get('model_name')}")
    elif is_agent_end(event):
        print(f"Agent response: {get_output_text(event)}")
```

## CLI Commands

The SDK includes `bqaa` (or `bq-agent-sdk`) CLI with 12+ commands:

### Diagnostics

```bash
# Check connectivity and permissions
bqaa diagnose --project-id $PROJECT_ID --dataset-id agent_analytics

# List available traces
bqaa list-traces --project-id $PROJECT_ID --dataset-id agent_analytics --limit 10

# Inspect a specific trace
bqaa inspect-trace --project-id $PROJECT_ID --dataset-id agent_analytics \
    --trace-id "trace-abc-123"

# Seed sample events for testing
bqaa seed-events --project-id $PROJECT_ID --dataset-id agent_analytics \
    --sessions 5
```

### Views Management

```bash
# Create per-event-type BigQuery views
bqaa create-views --project-id $PROJECT_ID --dataset-id agent_analytics

# List created views
bqaa list-views --project-id $PROJECT_ID --dataset-id agent_analytics

# Drop views
bqaa drop-views --project-id $PROJECT_ID --dataset-id agent_analytics
```

### Evaluation

```bash
# Run system evaluation (latency, tokens, cost)
bqaa evaluate --project-id $PROJECT_ID --dataset-id agent_analytics \
    --evaluator system --output-table eval_results

# Run LLM-as-Judge evaluation
bqaa evaluate --project-id $PROJECT_ID --dataset-id agent_analytics \
    --evaluator llm-judge --model-name gemini-2.0-flash-exp \
    --criteria correctness,hallucination --output-table llm_eval_results

# Run trajectory evaluation
bqaa evaluate-trajectory --project-id $PROJECT_ID --dataset-id agent_analytics \
    --golden-path "plan,execute,verify" --match-strategy exact
```

### Agent Context Graph

```bash
# Extract decision traces from agent context graph
bqaa context-graph --project-id $PROJECT_ID --dataset-id agent_analytics \
    --graph agent_decisions_graph --lookback-hours 24 --format json

# Schedule periodic materialization (creates Cloud Scheduler + Cloud Run)
bqaa schedule-context-graph --project-id $PROJECT_ID \
    --dataset-id agent_analytics --graph agent_decisions_graph \
    --cron "0 */6 * * *" --region us-central1
```

## Observability

### Create Event-Type Views

```python
from bigquery_agent_analytics.views import ViewManager

view_manager = ViewManager(client)

# Create all standard views
view_manager.create_all_views()

# List created views
views = view_manager.list_views()
for view in views:
    print(f"{view['view_name']}: {view['event_type']}")

# Query a specific view
agent_starts = client.query("""
    SELECT session_id, trace_id, input_text, timestamp
    FROM `my-project.agent_analytics.agent_start_events`
    WHERE DATE(timestamp) = CURRENT_DATE()
    LIMIT 10
""")

for row in agent_starts:
    print(f"{row.session_id}: {row.input_text}")
```

### Trace Visualization

```python
# Render trace as text
trace.render()

# Export trace to JSON
import json
with open("trace.json", "w") as f:
    json.dump(trace.to_dict(), f, indent=2)

# Analyze trace structure
print(f"Total events: {len(trace.events)}")
print(f"Trace duration: {trace.duration_ms}ms")
print(f"Session: {trace.session_id}")

# Filter events
tool_events = [e for e in trace.events if is_tool_invocation(e)]
print(f"Tool invocations: {len(tool_events)}")
```

## Evaluation

### System Evaluator (Code-Based Metrics)

```python
from bigquery_agent_analytics.evaluators import SystemEvaluator

evaluator = SystemEvaluator(client)

# Run evaluation on recent traces
results = evaluator.evaluate(
    start_time=datetime.utcnow() - timedelta(hours=24),
    end_time=datetime.utcnow()
)

# Results include: latency, token_count, turn_count, error_rate, cost
for result in results:
    print(f"Trace {result['trace_id']}: "
          f"latency={result['latency_ms']}ms, "
          f"tokens={result['token_count']}, "
          f"cost=${result['cost']:.4f}")

# Save to BigQuery
evaluator.save_results(results, output_table="eval_results")
```

### LLM-as-Judge Evaluation

```python
from bigquery_agent_analytics.evaluators import LLMAsJudge

judge = LLMAsJudge(
    client=client,
    model_name="gemini-2.0-flash-exp",
    criteria=["correctness", "hallucination", "helpfulness"]
)

# Evaluate traces
llm_results = judge.evaluate_traces(
    trace_ids=["trace-1", "trace-2", "trace-3"]
)

# Results include scores for each criterion (0.0-1.0)
for result in llm_results:
    print(f"Trace {result['trace_id']}:")
    print(f"  Correctness: {result['correctness_score']:.2f}")
    print(f"  Hallucination: {result['hallucination_score']:.2f}")
    print(f"  Helpfulness: {result['helpfulness_score']:.2f}")

# Save to BigQuery
judge.save_results(llm_results, output_table="llm_eval_results")
```

### Trajectory Matching

```python
from bigquery_agent_analytics.trace_evaluator import TrajectoryEvaluator

trajectory_eval = TrajectoryEvaluator(client)

# Define golden path (sequence of event types or tool names)
golden_path = ["search_documents", "extract_entities", "summarize"]

# Exact matching
exact_matches = trajectory_eval.evaluate(
    trace_ids=["trace-1", "trace-2"],
    golden_path=golden_path,
    match_strategy="exact"
)

# In-order matching (allows extra steps)
in_order_matches = trajectory_eval.evaluate(
    trace_ids=["trace-1", "trace-2"],
    golden_path=golden_path,
    match_strategy="in_order"
)

# Any-order matching (all steps present, any order)
any_order_matches = trajectory_eval.evaluate(
    trace_ids=["trace-1", "trace-2"],
    golden_path=golden_path,
    match_strategy="any_order"
)

for result in exact_matches:
    print(f"Trace {result['trace_id']}: "
          f"matches={result['matches']}, "
          f"similarity={result['similarity']:.2f}")
```

### Multi-Trial Evaluation (pass@k)

```python
from bigquery_agent_analytics.multi_trial import MultiTrialEvaluator

multi_trial = MultiTrialEvaluator(client)

# Run N trials per task and compute pass@k
results = multi_trial.evaluate(
    task_ids=["task-1", "task-2", "task-3"],
    trials_per_task=5,
    k_values=[1, 3, 5],
    evaluation_fn=lambda trace: judge.evaluate_traces([trace.trace_id])[0]
)

# Results include pass@1, pass@3, pass@5 metrics
for task_result in results:
    print(f"Task {task_result['task_id']}:")
    print(f"  pass@1: {task_result['pass_at_1']:.2%}")
    print(f"  pass@3: {task_result['pass_at_3']:.2%}")
    print(f"  pass@5: {task_result['pass_at_5']:.2%}")
```

### Grader Composition

```python
from bigquery_agent_analytics.grader_pipeline import (
    GraderPipeline,
    WeightedAverageStrategy,
    BinaryThresholdStrategy,
    MajorityVoteStrategy
)

# Weighted average of multiple evaluators
pipeline = GraderPipeline(strategy=WeightedAverageStrategy())
pipeline.add_grader(
    evaluator=judge,
    criteria="correctness",
    weight=0.6
)
pipeline.add_grader(
    evaluator=judge,
    criteria="helpfulness",
    weight=0.4
)

composite_score = pipeline.evaluate(trace_id="trace-123")
print(f"Composite score: {composite_score:.2f}")

# Binary threshold (pass/fail)
binary_pipeline = GraderPipeline(strategy=BinaryThresholdStrategy(threshold=0.7))
binary_pipeline.add_grader(evaluator=judge, criteria="correctness")
passed = binary_pipeline.evaluate(trace_id="trace-123")
print(f"Passed: {passed}")

# Majority vote across multiple judges
majority_pipeline = GraderPipeline(strategy=MajorityVoteStrategy())
majority_pipeline.add_grader(evaluator=judge, criteria="correctness")
majority_pipeline.add_grader(evaluator=judge, criteria="hallucination")
majority_pipeline.add_grader(evaluator=judge, criteria="helpfulness")
result = majority_pipeline.evaluate(trace_id="trace-123")
print(f"Majority vote: {result}")
```

### Eval Suite Lifecycle Management

```python
from bigquery_agent_analytics.eval_suite import EvalSuite

suite = EvalSuite(client)

# Define eval suite
suite.add_test(
    test_id="greeting_test",
    input_text="Hello, how are you?",
    expected_tools=["greeting_handler"],
    min_correctness=0.8
)

suite.add_test(
    test_id="search_test",
    input_text="Find documents about AI",
    expected_tools=["search_documents", "extract_entities"],
    min_correctness=0.9
)

# Run suite
suite_results = suite.run()

# Check graduation criteria (all tests pass threshold)
if suite.check_graduation(min_pass_rate=0.95):
    print("✓ Suite ready for production")
    suite.graduate()

# Check saturation (no improvement over N runs)
if suite.check_saturation(lookback_runs=10, improvement_threshold=0.01):
    print("⚠ Suite saturated - consider adding new tests")

# Save suite results
suite.save_results(output_table="eval_suite_results")
```

## Agent Context Graph (Decision-Trace Extraction)

The Agent Context Graph extracts decision traces from your agent's context: what requests it handled, what options it weighed, and what outcomes it committed, materialized as a BigQuery property graph.

### Deploy Context Graph (One-Time Setup)

```bash
# 1. Create tables (nodes: DecisionRequest, DecisionOption, DecisionOutcome)
export PROJECT_ID="my-project"
export DATASET="agent_analytics"

cat > table_ddl.sql <<'EOF'
CREATE TABLE IF NOT EXISTS `${PROJECT_ID}.${DATASET}.decision_requests` (
  request_id STRING NOT NULL,
  request_text STRING,
  timestamp TIMESTAMP,
  session_id STRING
);

CREATE TABLE IF NOT EXISTS `${PROJECT_ID}.${DATASET}.decision_options` (
  option_id STRING NOT NULL,
  request_id STRING NOT NULL,
  option_label STRING,
  option_score FLOAT64
);

CREATE TABLE IF NOT EXISTS `${PROJECT_ID}.${DATASET}.decision_outcomes` (
  outcome_id STRING NOT NULL,
  request_id STRING NOT NULL,
  status STRING,
  rationale STRING
);
EOF

envsubst < table_ddl.sql | bq query --use_legacy_sql=false

# 2. Create property graph
cat > property_graph.sql <<'EOF'
CREATE OR REPLACE PROPERTY GRAPH `${PROJECT_ID}.${DATASET}.agent_decisions_graph`
NODE TABLES (
  decision_requests AS DecisionRequest
    KEY(request_id)
    PROPERTIES(request_text, timestamp, session_id),
  decision_options AS DecisionOption
    KEY(option_id)
    PROPERTIES(option_label, option_score),
  decision_outcomes AS DecisionOutcome
    KEY(outcome_id)
    PROPERTIES(status, rationale)
)
EDGE TABLES (
  decision_options AS evaluatesOption
    SOURCE KEY(request_id) REFERENCES DecisionRequest
    DESTINATION KEY(option_id) REFERENCES DecisionOption
    PROPERTIES(option_score),
  decision_outcomes AS resultedIn
    SOURCE KEY(request_id) REFERENCES DecisionRequest
    DESTINATION KEY(outcome_id) REFERENCES DecisionOutcome
    PROPERTIES(status)
);
EOF

envsubst < property_graph.sql | bq query --use_legacy_sql=false
```

### Extract Decision Traces (Python API)

```python
from bigquery_agent_analytics.context_graph import ContextGraph

graph = ContextGraph(
    client=client,
    graph_name="agent_decisions_graph"
)

# Materialize window (extract traces from agent_events)
graph.materialize_window(
    lookback_hours=24,
    extract_fn=lambda events: {
        "request_id": events[0].get("trace_id"),
        "request_text": events[0].get("input_text"),
        "timestamp": events[0].get("timestamp"),
        "options": [
            {"option_id": f"opt-{i}", "option_label": opt}
            for i, opt in enumerate(events[0].get("options", []))
        ],
        "outcome": {
            "outcome_id": f"out-{events[-1].get('trace_id')}",
            "status": events[-1].get("status"),
            "rationale": events[-1].get("rationale")
        }
    }
)

# Query graph with GQL
decision_traces = client.query("""
SELECT * FROM GRAPH_TABLE(
  agent_analytics.agent_decisions_graph
  MATCH (req:DecisionRequest)-[eo:evaluatesOption]->(opt:DecisionOption),
        (req)-[ri:resultedIn]->(out:DecisionOutcome)
  COLUMNS (
    req.request_text AS question,
    opt.option_label AS considered,
    out.status AS outcome,
    out.rationale AS rationale
  )
)
WHERE req.timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
LIMIT 100
""")

for row in decision_traces:
    print(f"Q: {row.question} | Considered: {row.considered} | "
          f"Outcome: {row.outcome} | Rationale: {row.rationale}")
```

### Extract Decision Traces (CLI)

```bash
# Extract from last 24 hours
bqaa context-graph --project-id $PROJECT_ID --dataset-id agent_analytics \
    --graph agent_decisions_graph --lookback-hours 24 --format json

# Output: {"ok": true, "rows_inserted": 150, "sessions_processed": 5}

# Query extracted traces
bq query --use_legacy_sql=false "
SELECT * FROM GRAPH_TABLE(
  agent_analytics.agent_decisions_graph
  MATCH (req:DecisionRequest)-[eo:evaluatesOption]->(opt:DecisionOption),
        (req)-[ri:resultedIn]->(out:DecisionOutcome)
  COLUMNS (req.request_text AS question, opt.option_label AS considered,
           out.status AS outcome, out.rationale AS rationale)
) LIMIT 10"
```

### Schedule Periodic Materialization

```bash
# Deploy Cloud Scheduler + Cloud Run for automatic extraction
bqaa schedule-context-graph --project-id $PROJECT_ID \
    --dataset-id agent_analytics --graph agent_decisions_graph \
    --cron "0 */6 * * *" --region us-central1

# Manual trigger
bqaa context-graph --project-id $PROJECT_ID --dataset-id agent_analytics \
    --graph agent_decisions_graph --lookback-hours 6
```

See [Agent Context Graph Codelab](docs/codelabs/periodic_materialization.md) for the full 35-minute guided walkthrough.

## AI/ML Integration

### BigQuery AI Functions

```python
from bigquery_agent_analytics.ai_ml_integration import AIMLIntegration

ai_ml = AIMLIntegration(client)

# Generate summaries with AI.GENERATE
summaries = ai_ml.generate_summaries(
    input_table="agent_analytics.agent_end_events",
    prompt_column="output_text",
    model_name="gemini-2.0-flash-exp",
    output_table="trace_summaries"
)

# Classify sentiment with AI.CLASSIFY
classifications = ai_ml.classify_sentiment(
    input_table="agent_analytics.agent_end_events",
    text_column="output_text",
    output_table="sentiment_results"
)

# Generate embeddings with AI.EMBED
embeddings = ai_ml.generate_embeddings(
    input_table="agent_analytics.agent_start_events",
    text_column="input_text",
    embedding_model="textembedding-gecko@003",
    output_table="input_embeddings"
)

# Anomaly detection on latency
anomalies = ai_ml.detect_anomalies(
    metric_table="eval_results",
    metric_column="latency_ms",
    timestamp_column="timestamp",
    output_table="latency_anomalies"
)
```

### BigFrames Evaluation

```python
from bigquery_agent_analytics.bigframes_evaluator import BigFramesEvaluator
import bigframes.pandas as bpd

bf_eval = BigFramesEvaluator(client)

# Load traces as BigFrames DataFrame
df = bpd.read_gbq("agent_analytics.agent_events")

# Compute metrics
metrics_df = bf_eval.compute_metrics(
    df,
    metrics=["latency", "token_count", "error_rate"]
)

# Aggregate by session
session_metrics = metrics_df.groupby("session_id").agg({
    "latency_ms": "mean",
    "token_count": "sum",
    "error_rate": "mean"
})

# Save results
session_metrics.to_gbq("agent_analytics.session_metrics")
```

### Categorical (Hatteras-Style) Evaluation

```python
from bigquery_agent_analytics.categorical_evaluator import CategoricalEvaluator

cat_eval = CategoricalEvaluator(client)

# Define categories
categories = {
    "greeting": ["hello", "hi", "hey"],
    "search": ["find", "search", "lookup"],
    "summarize": ["summarize", "summary", "tldr"]
}

# Classify traces
classifications = cat_eval.classify_traces(
    trace_ids=["trace-1", "trace-2", "trace-3"],
    categories=categories
)

# Compute per-category metrics
category_metrics = cat_eval.compute_category_metrics(
    classifications=classifications,
    evaluator=judge
)

for category, metrics in category_metrics.items():
    print(f"{category}: accuracy={metrics['accuracy']:.2%}, "
          f"precision={metrics['precision']:.2%}")
```

## Analytics

### Drift Detection

```python
from bigquery_agent_analytics.feedback import DriftDetector

drift = DriftDetector(client)

# Compare question distributions (golden vs production)
drift_results = drift.detect_drift(
    golden_table="agent_analytics.golden_questions",
    production_table="agent_analytics.agent_start_events",
    text_column="input_text",
    lookback_days=7
)

if drift_results["drift_detected"]:
    print(f"⚠ Drift detected! KL divergence: {drift_results['kl_divergence']:.4f}")
    print(f"Top drifting terms: {drift_results['top_drifting_terms']}")
else:
    print("✓ No significant drift detected")

# Save drift report
drift.save_report(drift_results, output_table="drift_reports")
```

### Multi-Stage Insights Pipeline

```python
from bigquery_agent_analytics.insights import InsightsPipeline

pipeline = InsightsPipeline(client)

# Stage 1: Extract features
pipeline.add_stage(
    name="feature_extraction",
    query="""
    SELECT trace_id, session_id, latency_ms, token_count, tool_count
    FROM `agent_analytics.eval_results`
    WHERE DATE(timestamp) = CURRENT_DATE()
    """
)

# Stage 2: Aggregate by session
pipeline.add_stage(
    name="session_aggregation",
    query="""
    SELECT session_id,
           AVG(latency_ms) AS avg_latency,
           SUM(token_count) AS total_tokens,
           COUNT(*) AS trace_count
    FROM {feature_extraction}
    GROUP BY session_id
    """
)

# Stage 3: Identify outliers
pipeline.add_stage(
    name="outlier_detection",
    query="""
    SELECT * FROM {session_aggregation}
    WHERE avg_latency > (SELECT APPROX_QUANTILES(avg_latency, 100)[OFFSET(95)]
                         FROM {session_aggregation})
    """
)

# Run pipeline
results = pipeline.run()
outlier_sessions = results["outlier_detection"]

for row in outlier_sessions:
    print(f"Session {row.session_id}: avg_latency={row.avg_latency}ms")
```

### Long-Horizon Memory Service

```python
from bigquery_agent_analytics.memory_service import MemoryService

memory = MemoryService(client)

# Store agent memory
memory.store(
    session_id="session-123",
    memory_key="user_preferences",
    memory_value={"theme": "dark", "language": "en"},
    ttl_hours=720  # 30 days
)

# Retrieve memory
prefs = memory.retrieve(
    session_id="session-123",
    memory_key="user_preferences"
)
print(f"User preferences: {prefs}")

# Cross-session retrieval (find similar sessions)
similar_sessions = memory.find_similar_sessions(
    current_session_id="session-123",
    similarity_threshold=0.8,
    max_results=5
)

for session in similar_sessions:
    print(f"Similar session: {session['session_id']}, "
          f"similarity={session['similarity']:.2f}")
```

## Configuration

### Environment Variables

```bash
# Required
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account-key.json"
export PROJECT_ID="my-gcp-project"
export DATASET_ID="agent_analytics"

# Optional
export BIGQUERY_LOCATION="US"  # Default: US
export EVAL_MODEL="gemini-2.0-flash-exp"  # LLM judge model
export LOG_LEVEL="INFO"  # DEBUG, INFO, WARNING, ERROR
```

### Client Configuration

```python
# Custom BigQuery client
from google.cloud import bigquery

bq_client = bigquery.Client(
    project="my-project",
    location="EU"
)

client = Client(
    project_id="my-project",
    dataset_id="agent_analytics",
    bq_client=bq_client  # Use custom client
)

# Custom job labels
client.default_job_labels = {
    "team": "ml-platform",
    "env": "production"
}
```

## Common Patterns

### Continuous Evaluation Pipeline

```python
from datetime import datetime, timedelta

# Evaluate last hour every hour
def continuous_eval_job():
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)
    
    # System metrics
    system_eval = SystemEvaluator(client)
    sys_results = system_eval.evaluate(start_time, end_time)
    system_eval.save_results(sys_results, "hourly_system_metrics")
    
    # LLM judge
    judge = LLMAsJudge(client, model_name="gemini-2.0-flash-exp")
    llm_results = judge.evaluate_traces(
        trace_ids=[r["trace_id"] for r in sys_results]
    )
    judge.save_results(llm_results, "hourly_llm_scores")
    
    # Alert on anomalies
    ai_ml = AIMLIntegration(client)
    anomalies = ai_ml.detect_anomalies(
        metric_table="hourly_system_metrics",
        metric_column="latency_ms",
        timestamp_column="timestamp"
    )
    
    if len(anomalies) > 0:
        alert_on_call(anomalies)

# Schedule with Cloud Scheduler
# (or run as cron in your environment)
```

### Golden Dataset Validation

```python
# Compare production against golden dataset
golden_traces = client.get_traces_for_session("golden-session-id")
prod_traces = client.get_traces(
    start_time=datetime.utcnow() - timedelta(hours=1),
    end_time=datetime.utcnow()
)

trajectory_eval = TrajectoryEvaluator(client)

for prod_trace in prod_traces[:10]:  # Sample
    for golden_trace in golden_traces:
        result = trajectory_eval.evaluate(
            trace_ids=[prod_trace.trace_id],
            golden_path=extract_golden_path(golden_trace),
            match_strategy="in_order"
        )
        if not result[0]["matches"]:
            print(f"⚠ Trace {prod_trace.trace_id} diverged from golden path")
```

### Cost Attribution by Team

```python
# Query job labels for cost attribution
cost_query = """
SELECT
  labels.value AS team,
  SUM(total_bytes_billed) / POW(10, 12) AS tb_billed,
  SUM(total_slot_ms) / 1000 / 3600 AS slot_hours
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
CROSS JOIN UNNEST(labels) AS labels
WHERE labels.key = 'team'
  AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND state = 'DONE'
GROUP BY team
ORDER BY tb_billed DESC
"""

cost_by_team = client.query(cost_query)
for row in cost_by_team:
    print(f"{row.team}: {row.tb_billed:.2f} TB billed, "
          f"{row.slot_hours:.2f} slot-hours")
```

### Trace Sampling for Manual Review

```python
# Sample traces for human review
import random

all_traces = client.get_traces(
    start_time=datetime.utcnow() - timedelta(days=1),
    end_time=datetime.utcnow()
)

# Stratified sampling: by latency buckets
low_latency = [t for t in all_traces if t.duration_ms < 1000]
mid_latency = [t for t in all_traces if 1000 <= t.duration_ms < 5000]
high_latency = [t for t in all_traces if t.duration_ms >= 5000]

sample = (
    random.sample(low_latency, min(10, len(low_latency))) +
    random.sample(mid_latency, min(10, len(mid_latency))) +
    random.sample(high_latency, min(
