---
name: data-analytics-skills-claude
description: Use Claude's portable data analytics skills library for quality checks, analysis, documentation, visualization, and stakeholder communication
triggers:
  - "help me run a data quality audit"
  - "I need to analyze this cohort data"
  - "create a semantic model for our metrics"
  - "investigate why this metric dropped"
  - "build a dashboard specification"
  - "document this SQL query in business terms"
  - "plan my data analysis approach"
  - "generate an executive summary of these findings"
---

# Data Analytics Skills for Claude

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A structured library of 31 reusable AI-powered skills that turn Claude into a hands-on analytics partner. Each skill is a self-contained instruction set that works on-demand without setup, asking targeted questions to gather exactly what it needs before executing a complete analytical workflow.

## What This Project Does

This is a **skill collection**, not a library you import. When you describe an analytics task, Claude automatically selects and activates the appropriate skill from six categories:

1. **Data Quality & Validation** (5 skills) - EDA, quality audits, query validation, schema mapping
2. **Documentation & Knowledge** (5 skills) - Semantic models, documentation, SQL translation
3. **Data Analysis & Investigation** (7 skills) - Cohorts, segmentation, funnels, time series, A/B tests
4. **Data Storytelling & Visualization** (5 skills) - Insights, charts, dashboards, narratives
5. **Stakeholder Communication** (5 skills) - Translation, requirements, impact quantification
6. **Workflow Optimization** (4 skills) - Planning, context packaging, peer review

## Installation

### For Individual Use

1. Clone the repository:
```bash
git clone https://github.com/nimrodfisher/data-analytics-skills.git
cd data-analytics-skills
```

2. Reference skills in your Claude conversations:
```
"Use the programmatic-eda skill to analyze this dataset"
```

### For AI Coding Agents

Add to your agent's system prompt or project context:

```python
# In your agent configuration
SKILL_PATH = "./data-analytics-skills"

# Reference specific skills
with open(f"{SKILL_PATH}/01-data-quality-validation/programmatic-eda/SKILL.md") as f:
    skill_context = f.read()
```

### For Teams

Create a central repository with customized skills:

```bash
# Fork and customize
git clone https://github.com/your-org/data-analytics-skills.git
cd data-analytics-skills

# Add company-specific context
mkdir -p 01-data-quality-validation/programmatic-eda/references
echo "# Company Schema" > references/company-schema.md
```

## Key Concepts

### Skill Structure

Every skill follows this pattern:

```
skill-name/
├── SKILL.md              # Complete skill instructions
├── README.md             # Human-readable documentation
└── references/           # Optional: company-specific context
    ├── company-schema.md
    ├── metric-definitions.md
    └── business-rules.md
```

### How Skills Work

1. **Minimal Context Request** - Skill asks only essential information
2. **Structured Execution** - Step-by-step analytical workflow
3. **Assumption Surfacing** - Flags uncertainties explicitly
4. **Templated Output** - Consistent, shareable results

## Common Usage Patterns

### Pattern 1: Exploratory Data Analysis

```python
# You provide data
import pandas as pd

df = pd.read_csv('user_data.csv')
print(df.head())
print(df.info())

# Then say:
# "Run programmatic-eda on this dataset"

# Claude will:
# 1. Ask about business context
# 2. Check data quality systematically
# 3. Identify patterns, outliers, distributions
# 4. Flag potential issues
# 5. Suggest next analysis steps
```

### Pattern 2: Root Cause Investigation

```python
# You have a metric drop
"""
Our daily active users dropped from 10,000 to 8,800 last week.
Activate root-cause-investigation.
"""

# Claude asks:
# - Metric definition
# - Normal baseline/range
# - Recent changes
# - Segmentation dimensions

# Then provides:
# 1. Hypothesis tree
# 2. Data checks to run
# 3. Segmentation analysis
# 4. Timeline correlation
# 5. Likely root causes ranked
```

### Pattern 3: A/B Test Analysis

```python
import pandas as pd
import numpy as np

# Load experiment data
test_data = pd.DataFrame({
    'user_id': range(1000),
    'variant': np.random.choice(['control', 'treatment'], 1000),
    'converted': np.random.binomial(1, 0.15, 1000)
})

# Say: "Analyze this A/B test with ab-test-analysis skill"

# Claude will:
# 1. Validate sample sizes
# 2. Check for bias (SRM test)
# 3. Calculate statistical significance
# 4. Compute confidence intervals
# 5. Assess practical significance
# 6. Flag any issues
```

### Pattern 4: SQL Documentation

```sql
-- Complex query you need to explain
SELECT 
    DATE_TRUNC('week', u.created_at) AS cohort_week,
    COUNT(DISTINCT u.user_id) AS cohort_size,
    COUNT(DISTINCT CASE WHEN o.created_at <= u.created_at + INTERVAL '7 days' 
          THEN o.user_id END) AS week_1_retained
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY 1
ORDER BY 1;
```

Say: "Use sql-to-business-logic to document this query"

Claude produces:
- Plain English explanation
- Business logic breakdown
- Key assumptions
- Edge cases handled
- When to use/not use

### Pattern 5: Dashboard Specification

```
"I need a dashboard for marketing performance. 
Use dashboard-specification skill."
```

Claude asks about:
- Primary audience
- Key questions to answer
- Available data sources
- Refresh frequency needs

Then delivers:
- Metric definitions
- Visual layout mockup
- Filter specifications
- Data requirements
- Success criteria

### Pattern 6: Chaining Multiple Skills

```python
# Full analysis workflow

# 1. Planning phase
"""
Use analysis-planning for investigating signup drop.
Context: B2C SaaS, 15% drop in weekly signups over 2 weeks.
"""

# 2. Data quality check
"""
Run data-quality-audit on signup table.
[Provide schema and sample data]
"""

# 3. Investigation
"""
Execute root-cause-investigation with validated data.
"""

# 4. Documentation
"""
Use analysis-documentation to document findings.
"""

# 5. Stakeholder communication
"""
Generate executive-summary for VP of Growth.
"""
```

## Customization

### Adding Company Context

Create a `references/` folder in any skill:

```bash
cd 02-documentation-knowledge/semantic-model-builder
mkdir references

# Add your definitions
cat > references/metric-definitions.md << 'EOF'
# Standard Metrics

## Activation Rate
- **Formula**: (Users completing 3+ key actions in 7 days) / Total signups
- **Threshold**: >40% is healthy
- **Owner**: Growth team

## Monthly Recurring Revenue (MRR)
- **Formula**: Sum of all active subscription values
- **Excludes**: One-time payments, paused subscriptions
- **Currency**: USD normalized
EOF
```

Claude automatically uses these references when the skill runs.

### Creating Team Conventions

```bash
# Add analysis standards
cat > 06-workflow-optimization/analysis-planning/references/team-conventions.md << 'EOF'
# Analysis Standards

## Before Starting
- [ ] Create JIRA ticket
- [ ] Check if similar analysis exists
- [ ] Schedule 15min scoping with PM

## During Analysis
- [ ] Log all assumptions in analysis-assumptions-log
- [ ] Validate data with data-quality-audit
- [ ] Document queries in SQL comments

## Before Sharing
- [ ] Run analysis-qa-checklist
- [ ] Peer review with another analyst
- [ ] Add to company wiki
EOF
```

## Skill Quick Reference

### When to Use Each Category

| You Need To... | Category | Start With |
|---------------|----------|------------|
| Check new data quality | Data Quality | `programmatic-eda` |
| Review or write SQL | Data Quality | `query-validation`, `schema-mapper` |
| Investigate metric change | Analysis | `root-cause-investigation` |
| Analyze customer behavior | Analysis | `cohort-analysis`, `segmentation-analysis` |
| Measure experiment | Analysis | `ab-test-analysis` |
| Document findings | Documentation | `analysis-documentation` |
| Define metrics once | Documentation | `semantic-model-builder` |
| Build dashboard | Storytelling | `dashboard-specification`, `visualization-builder` |
| Present to executives | Stakeholder | `executive-summary-generator` |
| Gather requirements | Stakeholder | `stakeholder-requirements-gathering` |
| Plan complex analysis | Workflow | `analysis-planning` |

### Most Frequently Used Skills

```python
# Top 10 by typical usage
CORE_SKILLS = [
    "programmatic-eda",              # Start of every analysis
    "root-cause-investigation",      # Metric drops/spikes
    "analysis-planning",             # Before big projects
    "query-validation",              # SQL review
    "semantic-model-builder",        # One-time setup, huge ROI
    "ab-test-analysis",              # Experiments
    "cohort-analysis",               # Retention tracking
    "executive-summary-generator",   # Reporting up
    "visualization-builder",         # Chart decisions
    "analysis-documentation"         # Knowledge capture
]
```

## Troubleshooting

### Issue: Skill asks too many questions

**Solution**: Provide more upfront context

```python
# Instead of:
"Analyze this data"

# Do:
"""
Run programmatic-eda on user signup data.
Context:
- Table: signups
- Rows: 50,000
- Timeframe: Last 90 days
- Business: B2C SaaS
- Main concerns: Conversion rate, signup source quality
"""
```

### Issue: Output doesn't match company standards

**Solution**: Add company references

```bash
cd skill-name/references
# Create markdown files with your standards
# Claude will automatically use them
```

### Issue: Skill produces generic insights

**Solution**: Provide business context explicitly

```python
# Good context example:
"""
Use root-cause-investigation.

Metric: Daily Active Users (DAU)
- Normal range: 8,000-10,000
- Current: 6,500 (35% below normal)
- Started: March 15 (5 days ago)
- Recent changes:
  - New onboarding flow deployed March 14
  - Email provider switched March 10
  - Major competitor launched promo March 12
- Segments to check: Platform (iOS/Android/Web), User cohort, Geography
"""
```

### Issue: Want to use skills programmatically

**Solution**: Reference skills in system prompts

```python
import os

def load_skill(skill_path):
    """Load skill content for programmatic use."""
    with open(f"{skill_path}/SKILL.md", 'r') as f:
        return f.read()

# Example: Build custom analysis pipeline
skills = [
    load_skill("01-data-quality-validation/data-quality-audit"),
    load_skill("03-data-analysis-investigation/cohort-analysis"),
    load_skill("04-data-storytelling-visualization/insight-synthesis")
]

system_prompt = f"""
You are a data analyst. Use these skills in sequence:

{chr(10).join(skills)}

Follow each skill's workflow completely before moving to the next.
"""
```

### Issue: Skills not activating automatically

**Solution**: Skills activate based on natural language. Be explicit:

```python
# Explicit activation:
"Use the cohort-analysis skill to..."
"Activate root-cause-investigation for..."
"Run programmatic-eda on..."

# Or describe the task clearly:
"I need to understand retention by signup cohort"  # → cohort-analysis
"Why did our conversion rate drop?"                # → root-cause-investigation
"Check if this dataset is clean"                   # → data-quality-audit
```

## Real-World Examples

### Example 1: Complete Investigation Workflow

```python
"""
PROJECT: Investigate 20% drop in mobile app engagement

STEP 1: Plan the investigation
→ Use analysis-planning skill
Output: Structured approach with hypotheses

STEP 2: Validate data quality
→ Use data-quality-audit on app_events table
Output: Quality report, issues flagged

STEP 3: Run investigation
→ Use root-cause-investigation
Output: Ranked hypotheses with evidence

STEP 4: Deep dive on top cause
→ Use segmentation-analysis on iOS vs Android
Output: iOS users affected, Android stable

STEP 5: Document findings
→ Use analysis-documentation
Output: Reproducible methodology doc

STEP 6: Present to stakeholders
→ Use executive-summary-generator
Output: 1-page summary for leadership
"""
```

### Example 2: Building a Metrics Framework

```python
"""
PROJECT: Create company-wide metric definitions

STEP 1: Build semantic model
→ Use semantic-model-builder skill

Provide:
- List of key metrics (ARR, MRR, CAC, LTV, Churn, etc.)
- Business definitions
- Owners

Output: Standardized metric layer

STEP 2: Add to each relevant skill
→ Save output to references/metric-definitions.md in:
  - business-metrics-calculator/references/
  - ab-test-analysis/references/
  - executive-summary-generator/references/

STEP 3: Create data catalog
→ Use data-catalog-entry for each data source

Result: Everyone uses same definitions automatically
"""
```

## Configuration

### Environment Variables

No environment variables required. Skills work with data you provide in conversation.

### Optional: Team Configuration

Create a `.data-skills-config.json` in your project root:

```json
{
  "default_references_path": "./company-context",
  "metric_definitions": "./company-context/metrics.md",
  "schema_docs": "./company-context/schema.md",
  "business_rules": "./company-context/rules.md",
  "team_conventions": "./company-context/conventions.md"
}
```

Reference in skills with:
```
See ${project_root}/.data-skills-config.json for company standards
```

## Advanced Patterns

### Pattern: Automated Quality Gates

```python
# Pre-commit hook for analysis files
"""
Before committing analysis:
1. Run analysis-qa-checklist
2. Verify all assumptions logged with analysis-assumptions-log
3. Check documentation exists via analysis-documentation
4. Validate SQL with query-validation
"""
```

### Pattern: Analysis Templates

```python
# Create project template
ANALYSIS_TEMPLATE = """
1. [analysis-planning] → approach.md
2. [data-quality-audit] → quality-report.md
3. [relevant-analysis-skill] → findings.md
4. [analysis-assumptions-log] → assumptions.md
5. [analysis-documentation] → documentation.md
6. [executive-summary-generator] → summary.md
7. [analysis-retrospective] → learnings.md
"""
```

### Pattern: Skill Chaining Syntax

```python
# Define analysis pipeline
"""
PIPELINE: User retention deep-dive

data-quality-audit → cohort-analysis → segmentation-analysis → 
insight-synthesis → visualization-builder → executive-summary-generator

Pass context forward at each step.
Flag any blockers immediately.
"""
```

## Best Practices

1. **Always start with `analysis-planning`** for complex work
2. **Run `data-quality-audit`** before analyzing unfamiliar data
3. **Use `semantic-model-builder`** once, reference forever
4. **Document with `analysis-assumptions-log`** as you go
5. **Chain skills** for complete workflows, don't jump to conclusions
6. **Add company context** to `references/` folders for frequently-used skills
7. **Run `analysis-qa-checklist`** before sharing any findings

## Resources

- **Repository**: https://github.com/nimrodfisher/data-analytics-skills
- **Skill Map**: [Interactive visualization](https://excalidraw.com/#json=wWcmLjEVHAYl4I4fynPSm,d8UC4Lexp2iSy5OPfIyJPQ)
- **Individual Skills**: Browse the repository structure by category
