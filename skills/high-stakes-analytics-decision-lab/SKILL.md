---
name: high-stakes-analytics-decision-lab
description: Platform-neutral analytical skill that profiles messy data, selects case-adaptive methods, and produces source-backed visual reports for high-stakes decisions
triggers:
  - analyze this dataset and build an evidence-based report
  - run a high-stakes decision analysis with data quality gates
  - create an evidence intelligence report from this data
  - perform adaptive analytics with diagnostic predictive or prescriptive routing
  - validate this data and choose the right analytical method
  - generate a decision intelligence brief with uncertainty bounds
  - profile data quality and route to appropriate analysis method
  - build a reproducible evidence report with data lineage
---

# High-Stakes Analytics & Decision Lab

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A platform-neutral, evidence-constrained analytical system that transforms ambiguous questions into reproducible evidence products. It profiles data quality, selects case-adaptive analytical methods (descriptive, diagnostic, predictive, prescriptive), and produces source-backed reports with explicit uncertainty and claim boundaries.

## What It Does

Instead of forcing every dataset through fixed pipelines, this system:

- **Gates data quality** before analysis (detects missing, duplicates, leakage, grain mismatches)
- **Routes adaptively** to descriptive, diagnostic, predictive, or prescriptive methods based on question + data
- **Produces two-layer outputs**: Evidence Intelligence Report (always) + Decision Intelligence Brief (conditional)
- **Preserves lineage** with hashed sources, reproducible transforms, and claim boundaries
- **Handles shared uncertainty** across alternatives (common market, time, operational shocks)

## Installation

### Quick Install (NPX)

```bash
npx skills add limingrui679-design/high-stakes-analytics-decision-lab -g
```

### Manual Python Install

```bash
git clone https://github.com/limingrui679-design/high-stakes-analytics-decision-lab.git
cd high-stakes-analytics-decision-lab
pip install -r requirements.txt
```

### Docker

```bash
docker build -t high-stakes-lab .
docker run -v $(pwd)/data:/data -v $(pwd)/outputs:/outputs high-stakes-lab
```

## Core Architecture

The system follows a fixed evidence spine with adaptive routing:

```
Question → Data Contract → Quality Gate → Adaptive Route → Evidence Report → Decision Brief (conditional)
```

### Four Quality Gate Outcomes

1. **`ready`** — No material issues, continue
2. **`ready_with_documented_limitations`** — Localized issues, visible limits
3. **`needs_user_confirmation`** — Requires explicit approval for cleaning actions
4. **`blocked`** — Critical failure, stop and request corrected data

### Four Analytical Routes

1. **Descriptive** — What is happening? (baseline, trends, distributions)
2. **Diagnostic** — Why? (drivers, decomposition, competing explanations)
3. **Predictive** — What next? (forecasts, validation, calibration, drift)
4. **Prescriptive** — What action? (alternatives, constraints, tail risk, sensitivity)

## Project Structure

```
high-stakes-analytics-decision-lab/
├── src/
│   ├── data_quality/          # Quality profiling & gates
│   ├── routing/               # Adaptive method selection
│   ├── methods/               # Analytical modules (descriptive, diagnostic, etc.)
│   ├── reporting/             # Evidence & decision report generation
│   └── orchestration/         # End-to-end workflow
├── examples/
│   └── real-data-cases/       # 10 complete projects with data + outputs
├── references/
│   ├── data-quality-gate.md   # Quality gate contract
│   ├── method-routing.md      # Route selection rules
│   └── method-modules.md      # Executable boundaries
└── requirements.txt
```

## Configuration

Create a `config.yaml` for your analysis:

```yaml
project:
  name: "customer-churn-analysis"
  question: "Which customers are at risk of churning in next 90 days?"
  decision_owner: "Head of Retention"
  
evidence_contract:
  source: "data/customer_events.csv"
  grain: "customer_id"
  time_field: "event_date"
  target_field: "churned"
  horizon_days: 90
  
data_quality:
  missing_threshold: 0.15
  duplicate_check: true
  leakage_detection: true
  privacy_scan: true
  
routing:
  force_descriptive: true
  enable_diagnostic: true
  enable_predictive: true
  enable_prescriptive: false
  
outputs:
  evidence_report: "outputs/evidence_report.md"
  decision_brief: "outputs/decision_brief.md"
  figures_dir: "outputs/figures/"
  reproducibility_package: "outputs/reproducibility.zip"
```

## Usage Examples

### 1. Basic Evidence Analysis

```python
from src.orchestration import AnalyticsWorkflow
from src.config import load_config

# Load configuration
config = load_config("config.yaml")

# Initialize workflow
workflow = AnalyticsWorkflow(config)

# Run complete evidence pipeline
results = workflow.run()

# Check quality gate outcome
print(f"Quality gate: {results.quality_gate.status}")
print(f"Adaptive route: {results.selected_route}")
print(f"Evidence report: {results.evidence_report_path}")
print(f"Decision brief: {results.decision_brief_path}")
```

### 2. Data Quality Profiling Only

```python
from src.data_quality import DataQualityGate
import pandas as pd

# Load data
df = pd.read_csv("data/messy_data.csv")

# Define data contract
contract = {
    "grain": "transaction_id",
    "time_field": "timestamp",
    "target_field": "outcome",
    "expected_schema": {
        "transaction_id": "string",
        "timestamp": "datetime",
        "amount": "numeric",
        "outcome": "binary"
    }
}

# Profile quality
gate = DataQualityGate(df, contract)
quality_report = gate.profile()

print(f"Status: {quality_report.status}")
print(f"Missing rate: {quality_report.missing_rate}")
print(f"Duplicates: {quality_report.duplicate_count}")
print(f"Leakage detected: {quality_report.has_leakage}")
print(f"Privacy issues: {quality_report.privacy_warnings}")

# Get recommended actions
if quality_report.status == "needs_user_confirmation":
    for action in quality_report.required_approvals:
        print(f"Approve: {action.id} - {action.description}")
```

### 3. Adaptive Route Selection

```python
from src.routing import RouteSelector

# Define question and data characteristics
question_spec = {
    "type": "predictive",
    "estimand": "probability of outcome",
    "population": "active customers",
    "horizon": "90 days"
}

data_characteristics = {
    "n_rows": 15000,
    "n_features": 42,
    "target_prevalence": 0.08,
    "has_time_series": True,
    "has_identifiable_pii": False
}

# Select route
selector = RouteSelector()
route = selector.select(question_spec, data_characteristics)

print(f"Primary route: {route.primary}")
print(f"Additional modules: {route.additional}")
print(f"Methods: {route.selected_methods}")
print(f"Validation strategy: {route.validation}")
```

### 4. Predictive Route with Validation

```python
from src.methods.predictive import PredictiveModule
from src.reporting import EvidenceReportGenerator

# Initialize predictive module
predictor = PredictiveModule(
    target="churned",
    horizon_days=90,
    validation_strategy="temporal_holdout",
    calibration_check=True,
    subgroup_analysis=True
)

# Fit model
predictor.fit(df_train, timestamp_field="signup_date")

# Validate on holdout
validation_results = predictor.validate(df_test)

print(f"AUC: {validation_results.auc:.3f}")
print(f"Calibration slope: {validation_results.calibration_slope:.3f}")
print(f"Brier score: {validation_results.brier:.3f}")
print(f"Worst subgroup AUC: {validation_results.min_subgroup_auc:.3f}")

# Check deployment gate
if validation_results.deployment_status == "do_not_deploy":
    print(f"BLOCKED: {validation_results.blocking_reason}")
else:
    print(f"Validated for deployment with boundaries: {validation_results.boundaries}")

# Generate evidence report
report_gen = EvidenceReportGenerator()
evidence_report = report_gen.generate(
    data_quality=quality_report,
    route=route,
    validation=validation_results,
    output_path="outputs/evidence_report.md"
)
```

### 5. Prescriptive Route with Shared Shocks

```python
from src.methods.prescriptive import PrescriptiveModule

# Define decision problem
decision_spec = {
    "owner": "VP Operations",
    "alternatives": [
        {"id": "status_quo", "cost": 0, "capacity": 100},
        {"id": "expand_10pct", "cost": 50000, "capacity": 110},
        {"id": "expand_25pct", "cost": 120000, "capacity": 125}
    ],
    "criteria": ["expected_revenue", "capacity_utilization", "downside_risk"],
    "constraints": {"max_cost": 100000, "min_capacity": 105}
}

# Model shared uncertainty (all alternatives face same demand shock)
shared_shocks = {
    "market_demand": {"distribution": "normal", "mean": 1.0, "std": 0.15},
    "operational_efficiency": {"distribution": "lognormal", "mean": 1.0, "std": 0.08}
}

# Initialize prescriptive module
prescriptive = PrescriptiveModule(decision_spec, shared_shocks)

# Simulate outcomes
simulation_results = prescriptive.simulate(n_scenarios=10000)

print(f"Recommended alternative: {simulation_results.recommended}")
print(f"Expected value: ${simulation_results.expected_value:,.0f}")
print(f"5th percentile (tail risk): ${simulation_results.percentile_05:,.0f}")
print(f"Reversal conditions: {simulation_results.reversal_conditions}")

# Generate decision brief (only if evidence supports it)
if simulation_results.decision_ready:
    decision_brief = prescriptive.generate_brief(
        evidence_report_path="outputs/evidence_report.md",
        output_path="outputs/decision_brief.md"
    )
else:
    print(f"No decision-ready recommendation: {simulation_results.blocking_reason}")
```

### 6. Complete End-to-End Workflow

```python
from src.orchestration import AnalyticsWorkflow
from src.config import ProjectConfig

# Define complete configuration
config = ProjectConfig(
    question="Should we launch the new pricing tier?",
    data_source="data/user_behavior.parquet",
    evidence_contract={
        "grain": "user_id",
        "time_field": "activity_date",
        "population": "active_monthly_users",
        "estimand": "incremental_revenue",
        "horizon_days": 180
    },
    quality_gates={
        "max_missing": 0.10,
        "detect_leakage": True,
        "privacy_level": "high"
    },
    routing={
        "always_descriptive": True,
        "enable_diagnostic": True,
        "enable_predictive": True,
        "enable_prescriptive": True
    },
    outputs={
        "base_dir": "outputs/pricing_decision",
        "generate_reproducibility_package": True
    }
)

# Run full workflow
workflow = AnalyticsWorkflow(config)
results = workflow.execute()

# Inspect results
print(f"Quality gate: {results.quality_gate.status}")
print(f"Route selected: {results.route.primary} + {results.route.additional}")
print(f"Evidence report: {results.evidence_report_path}")
print(f"Decision status: {results.decision_status}")
print(f"Reproducibility package: {results.reproducibility_package_path}")

# Review figures
for fig_id, fig_path in results.figure_map.items():
    print(f"{fig_id}: {fig_path}")
```

## CLI Usage

### Profile Data Quality

```bash
python -m src.cli profile \
  --data data/messy_data.csv \
  --grain customer_id \
  --time-field signup_date \
  --output outputs/quality_report.json
```

### Run Complete Analysis

```bash
python -m src.cli analyze \
  --config config.yaml \
  --output-dir outputs/
```

### Generate Evidence Report Only

```bash
python -m src.cli evidence \
  --data data/clean_data.parquet \
  --config config.yaml \
  --route descriptive,predictive \
  --output outputs/evidence_report.md
```

### Add Decision Layer

```bash
python -m src.cli decision \
  --evidence-report outputs/evidence_report.md \
  --decision-config decision.yaml \
  --output outputs/decision_brief.md
```

## Real Examples

The repository includes 10 complete real-data projects in `examples/real-data-cases/projects/`:

1. **population-health-survival** — Heart failure risk (299 patients, descriptive → predictive → prescriptive)
2. **behavioral-reading-experiment** — Pseudoword reading (57 paired participants, descriptive → inferential)
3. **census-income-ai** — Income model validation (48,842 records, descriptive → predictive)
4. **bike-demand-operations** — Demand forecasting + allocation (17,379 system-hours)

Each includes:
- Raw data snapshot (hashed)
- Data quality report
- Configuration
- Runnable code
- Machine-readable results (JSON/CSV)
- Evidence Intelligence Report (Markdown)
- All figures (SVG/PNG)
- Decision Intelligence Brief (Markdown)

### Run a Real Example

```bash
cd examples/real-data-cases/projects/census-income-ai
python run.py --config config.yaml
```

Outputs will be in `outputs/`:
- `report.md` — Evidence Intelligence Report
- `decision/report/decision-report.md` — Decision Intelligence Brief
- `figures/` — All analytical figures
- `chart-map.json` — Figure index
- `reproducibility/` — Code + hashes

## Common Patterns

### Pattern 1: Data Quality Gate → Evidence Request

```python
# Use when data quality blocks analysis
gate = DataQualityGate(df, contract)
report = gate.profile()

if report.status == "blocked":
    evidence_request = {
        "status": "evidence_request",
        "reason": report.blocking_reason,
        "required_corrections": report.required_corrections,
        "resubmit_with": report.corrected_contract
    }
    # Stop here, do not proceed to analysis
    return evidence_request
```

### Pattern 2: Negative Validation → Do Not Deploy

```python
# Use when predictive model fails validation
predictor.fit(df_train)
validation = predictor.validate(df_test)

if validation.deployment_status == "do_not_deploy":
    decision_brief = {
        "status": "negative_validation",
        "evidence": validation.evidence_report_link,
        "blocking_issue": validation.blocking_reason,
        "alternatives": ["collect_more_data", "revise_estimand", "stop"]
    }
    # Do not deploy, document negative result
    return decision_brief
```

### Pattern 3: Evidence Sufficient → No Decision Layer Needed

```python
# Use when question is purely evidential
if question_type == "evidence_request":
    # Generate evidence report only
    evidence = generate_evidence_report(results)
    # Do NOT force a decision brief
    return {"evidence_report": evidence, "decision_brief": None}
```

### Pattern 4: Adaptive Route Composition

```python
# Use when multiple routes are justified
if data_characteristics.supports_multiple_routes():
    route = {
        "primary": "descriptive",  # Always first
        "additional": ["diagnostic", "predictive"],  # Add if justified
        "excluded": ["prescriptive"],  # Not enough for action
        "reason": "Insufficient alternatives and constraint data"
    }
```

## Troubleshooting

### Data Quality Gate Blocks Analysis

**Problem**: `status: "blocked"` with `reason: "grain_violation"`

**Solution**: Ensure your data contract matches actual data structure

```python
# Check grain uniqueness
print(f"Unique grain values: {df[grain_field].nunique()}")
print(f"Total rows: {len(df)}")

# If not unique, identify duplicates
dupes = df[df.duplicated(subset=[grain_field], keep=False)]
print(dupes)

# Fix contract or deduplicate explicitly
```

### Missing Field Errors

**Problem**: `KeyError: 'target_field'`

**Solution**: Verify all contract fields exist

```python
contract_fields = [contract["grain"], contract["time_field"], contract["target_field"]]
missing = [f for f in contract_fields if f not in df.columns]
if missing:
    print(f"Missing fields: {missing}")
    print(f"Available columns: {df.columns.tolist()}")
```

### Route Selection Returns "descriptive_only"

**Problem**: Expected predictive route but got descriptive only

**Solution**: Check data volume and target prevalence

```python
print(f"Rows: {len(df)}")
print(f"Target prevalence: {df[target].mean():.3f}")
print(f"Positive cases: {df[target].sum()}")

# Predictive requires minimum sample size and events
# Typically: n > 500 AND positive_cases > 50
```

### Calibration Failure in Predictive Route

**Problem**: `calibration_slope < 0.8` triggers validation failure

**Solution**: Recalibrate or document limitation

```python
from sklearn.calibration import CalibratedClassifierCV

# Recalibrate model
calibrated = CalibratedClassifierCV(model, method='isotonic', cv=5)
calibrated.fit(X_train, y_train)

# Or document as limitation
limitation = {
    "issue": "poor_calibration",
    "metric": f"slope={calibration_slope:.2f}",
    "boundary": "Use for ranking only, not absolute probabilities"
}
```

### Decision Brief Generation Fails

**Problem**: `decision_status: "no_decision_ready"`

**Solution**: This is often correct — not every analysis should produce a decision

```python
# Check if decision layer is actually justified
if not (
    feasible_alternatives_exist and
    constraints_defined and
    decision_owner_identified and
    reversal_conditions_specifiable
):
    # Correctly stop at evidence layer
    print("Evidence report is terminal product")
```

## Environment Variables

```bash
# Optional: Configure output paths
export ANALYTICS_LAB_OUTPUT_DIR=/path/to/outputs
export ANALYTICS_LAB_CACHE_DIR=/path/to/cache

# Optional: Set quality thresholds
export ANALYTICS_LAB_MAX_MISSING=0.15
export ANALYTICS_LAB_MIN_SAMPLE_SIZE=500

# Optional: Enable/disable modules
export ANALYTICS_LAB_ENABLE_PRESCRIPTIVE=false
export ANALYTICS_LAB_GENERATE_REPRODUCIBILITY=true
```

## References

- **Data Quality Gate**: `references/data-quality-gate.md` — Complete quality contract
- **Method Routing**: `references/method-routing.md` — Route selection rules
- **Method Modules**: `references/method-modules.md` — Executable boundaries
- **Evidence Contract**: See `examples/real-data-cases/` for complete project structures

## Key Principles

1. **Evidence before decision** — Always produce Evidence Intelligence Report; Decision Brief is conditional
2. **Gate before analysis** — Data quality must pass explicit thresholds
3. **Route adaptively** — Select methods based on question + data, not templates
4. **Preserve lineage** — Hash sources, version transforms, link claims to figures
5. **Bound claims** — Every prediction/recommendation has explicit limitations and reversal conditions
6. **Stop correctly** — Evidence request, negative validation, and `do_not_deploy` are valid terminal states
