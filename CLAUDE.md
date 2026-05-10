# Cloud & Capital — Claude Code FinOps Skills

This file gives Claude Code full context for the Cloud & Capital toolchain.
Copy it to the root of any cloudandcapital repo (or to `~/.claude/CLAUDE.md`)
to load FinOps context into every Claude Code session automatically.

---

## The Stack

Seven tools. One pipeline. Full Cloud + AI + SaaS + Kubernetes coverage.

| Tool | What it does | Repo |
|------|-------------|------|
| **Cloud Cost Guard** | Unified dashboard — Cloud / K8s / AI / SaaS spend, Lumen AI assistant, MCP server | `cloudandcapital/cloud-cost-guard` |
| **FinOps Lite** | Multi-cloud cost pull (AWS/Azure/GCP), FOCUS 2026 export, validator | `cloudandcapital/finops-lite` |
| **FinOps Watchdog** | Baseline-aware cost anomaly detection from any cost CSV | `cloudandcapital/finops-watchdog` |
| **Recovery Economics** | Resilience cost modeling — backup, restore, scenario compare | `cloudandcapital/recovery-economics` |
| **AI Cost Lens** | AI/LLM spend tracking — OpenAI, Anthropic, Bedrock | `cloudandcapital/ai-cost-lens` |
| **SaaS Cost Analyzer** | SaaS license governance — per-seat costs, unused seats, renewals | `cloudandcapital/saas-cost-analyzer` |
| **Tech Spend Command Center** | Executive summary — unified Cloud + AI + SaaS report | `cloudandcapital/tech-spend-command-center` |

---

## Cloud Cost Guard

**Location:** `cloud-cost-guard/`  
**Stack:** FastAPI + React (CRA + shadcn/ui + Recharts) + Vercel  
**Live demo:** https://guard.cloudandcapital.com

### Running locally

```bash
# Backend (FastAPI)
cd cloud-cost-guard
pip install -r backend/requirements.txt
uvicorn app:app --reload --port 8000

# Frontend (separate terminal)
cd frontend && npm install && npm start
```

The frontend proxies `/api` to `localhost:8000` during dev.

### Architecture

- `app.py` — main FastAPI server with synthetic data generation for all clouds + K8s + AI + SaaS
- `backend/connectors/` — live data connectors (aws, azure, gcp, k8s); each has `is_configured()` + `get_cost_data()`
- `backend/mcp_server.py` — MCP server exposing Lumen's FinOps tools to Claude Code / Cursor
- `frontend/src/App.js` — full dashboard; reads from `/api/*` endpoints
- `frontend/src/data/report.json` — static fallback report (aggregated by Cloud Capital Report)

### API endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/summary?window=30d&cloud=aws` | KPIs + top products + findings |
| `GET /api/findings?sort=savings&limit=50` | All findings, sorted |
| `GET /api/clouds` | All three clouds in one response |
| `GET /api/cost-trend?days=30&cloud=aws` | Daily cost series |
| `GET /api/service-breakdown?window=30d&cloud=aws` | Services pie data |
| `GET /api/k8s` | Kubernetes namespace + node pool data |
| `GET /api/top-movers?days=7&cloud=aws` | Biggest cost changes |

### Connector pattern

Every connector follows the same interface:

```python
class XxxConnector:
    def is_configured(self) -> bool: ...     # checks env vars
    def get_cost_data(self, window_days) -> dict: ...  # returns normalized dict
```

When `is_configured()` returns False, `app.py` falls back to synthetic data automatically.

### MCP server (Lumen)

```bash
# Run standalone
python backend/mcp_server.py

# Tools exposed:
#   get_cloud_summary   — AWS/Azure/GCP totals and trends
#   get_findings        — prioritized savings findings
#   get_cost_by_cloud   — per-cloud breakdown
#   get_saas_spend      — SaaS tool and license data
#   get_ai_spend        — AI/LLM model-level costs
#   get_k8s_spend       — Kubernetes namespace + node pool data
#   ask_lumen           — natural language FinOps query
```

**Claude Code config** (`~/.claude/mcp_servers.json`):
```json
{
  "mcpServers": {
    "cloud-cost-guard": {
      "command": "python",
      "args": ["/path/to/cloud-cost-guard/backend/mcp_server.py"]
    }
  }
}
```

**Cursor config** (`.cursor/mcp.json` in project root):
```json
{
  "mcpServers": {
    "lumen": {
      "command": "python",
      "args": ["backend/mcp_server.py"]
    }
  }
}
```

---

## FinOps Lite

**Location:** `finops-lite/`  
**Stack:** Python CLI (Click + Rich)  
**Install:** `pip install .` or `pipx install .`

### Key commands

```bash
# Cost overview (last 30 days)
finops cost overview --days 30

# Monthly report
finops cost monthly 2026-04

# Compare two months
finops cost compare 2026-04 2026-03

# Export FOCUS 1.0 CSV (legacy)
finops export focus --days 30 > focus-1.0.csv

# Export FOCUS 2026 CSV
finops export focus2026 --days 30 --output focus-2026.csv

# Validate any billing CSV against FOCUS 2026
finops validate focus my-billing-export.csv

# Signals / anomaly flags
finops signals detect --threshold 0.15
```

### FOCUS 2026 schema

Key columns added in FOCUS 2026 vs FOCUS 1.0:

| Column | Description |
|--------|-------------|
| `ChargeClass` | `Standard` or `Correction` (retroactive adjustments) |
| `EffectiveCost` | Cost after discounts / commitments applied |
| `ListCost` | On-demand list price (before discounts) |
| `ContractedCost` | Negotiated/contracted price |
| `CommitmentDiscountId/Name/Type/Status` | RI / Savings Plan / CUD tracking |
| `CapacityReservationId/Status` | Capacity reservation tracking |
| `ConsumedUnit/Quantity` | Actual usage (vs pricing unit) |
| `ServiceCategory` | Standardized taxonomy (Compute, Storage, Databases…) |
| `RegionId/Name` | Normalized region identifiers |
| `SkuId/SkuPriceId` | Pricing SKU references |
| `x_focus_schema_version` | `2026.0` — marks file as FOCUS 2026 compliant |

---

## FinOps Watchdog

**Location:** `finops-watchdog/`  
**Stack:** Python CLI

```bash
# Detect anomalies from a cost CSV
finops-watchdog detect --input costs.csv --baseline-days 30 --threshold 0.15

# Generate a markdown anomaly report
finops-watchdog detect --input costs.csv --report

# Output JSON for downstream tools
finops-watchdog detect --input costs.csv --format json
```

Watchdog compares recent spend against a rolling baseline using MAD (median absolute deviation) to minimize false positives from normal billing variance.

---

## Recovery Economics

**Location:** `recovery-economics/`  
**Stack:** Python CLI + scenario modeling

```bash
# Model recovery cost for a workload
recovery-economics model --workload workload.csv

# Compare two recovery scenarios
recovery-economics compare --baseline current.csv --alternative proposed.csv

# Output structured JSON for Cloud Cost Guard
recovery-economics model --workload workload.csv --format json
```

Key metrics:
- `monthly_storage_cost` — backup storage at target tier
- `monthly_restore_cost` — estimated egress + compute on restore
- `total_monthly_resilience_cost` — sum of above
- `rpo_hours`, `rto_hours` — recovery objectives

---

## AI Cost Lens

**Location:** `ai-cost-lens/`  
**Stack:** Python CLI

```bash
# Summarize AI spend across all configured providers
ai-cost-lens summary --days 30

# Per-model breakdown
ai-cost-lens breakdown --provider anthropic

# Export FOCUS 2026 compatible output
ai-cost-lens export --format focus2026
```

Supported providers: `openai`, `anthropic`, `bedrock` (AWS).  
Set credentials via `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or AWS profile.

---

## SaaS Cost Analyzer

**Location:** `saas-cost-analyzer/`  
**Stack:** Python CLI

```bash
# Analyze SaaS spend from CSV input
saas-cost-analyzer analyze --input saas-licenses.csv

# Flag unused seats
saas-cost-analyzer unused --threshold 0 --days 30

# Forecast renewal cost
saas-cost-analyzer forecast --months 12
```

---

## Tech Spend Command Center

**Location:** `tech-spend-command-center/`  
**Stack:** Python CLI

```bash
# Generate unified executive summary
tech-spend-command-center report --format markdown

# JSON output for downstream dashboards
tech-spend-command-center report --format json

# Include Kubernetes data
tech-spend-command-center report --include-k8s
```

---

## FinOps Best Practices (Claude Code context)

### Cost optimization priority order

1. **Orphaned resources** — delete first (zero risk, immediate savings)
   - Unattached EBS volumes, unused Elastic IPs, idle load balancers
2. **Rightsizing** — downsize after validating utilization over 7+ days
   - EC2 < 15% CPU p50, Azure VMs < 20% DTU, GKE nodes < 40% utilization
3. **Commitment discounts** — buy RIs / Savings Plans after rightsizing
   - Only commit to capacity you've already rightsized
4. **Architecture changes** — last; highest risk, highest reward
   - Serverless migration, storage tiering, spot instance adoption

### Kubernetes cost optimization patterns

```
# Identify namespace waste
kubectl top pods --all-namespaces --sort-by=cpu

# Check resource requests vs actual usage (requires metrics-server)
kubectl resource-capacity --pods --sort cpu.util

# Right-size recommendations (VPA dry run)
kubectl apply -f vpa.yaml --dry-run=server

# Node pool autoscaler
gcloud container clusters update CLUSTER --enable-autoscaling \
  --min-nodes=3 --max-nodes=12 --node-pool=general-purpose
```

Key K8s waste signals:
- Namespace CPU request ratio < 40% → right-size resource requests
- Node pool utilization < 50% → reduce minimum node count
- Large memory-to-CPU ratio mismatch → change machine family

### AI / LLM cost optimization patterns

```python
# Prompt caching (Anthropic) — cache system prompts
# Saves ~90% of input token costs on repeated calls
client.messages.create(
    model="claude-sonnet-4-6",
    system=[{"type": "text", "text": long_system_prompt,
              "cache_control": {"type": "ephemeral"}}],
    ...
)

# Model routing — use smaller models for classification/extraction
# claude-haiku-4-5: fast, cheap → routing, classification
# claude-sonnet-4-6: balanced → most tasks
# claude-opus-4-7: most capable → complex reasoning only
```

### SaaS spend governance

```bash
# 90-day no-login audit (Okta / Google Workspace)
saas-cost-analyzer unused --login-threshold-days 90

# Seat recovery workflow:
# 1. Export unused seats report
# 2. Send 14-day warning to user
# 3. Downgrade to free tier or remove on renewal date
# 4. Track savings in Tech Spend Command Center
```

### FOCUS 2026 field reference (for debugging exports)

```python
# Required fields — validation will fail without these:
REQUIRED = [
    "BillingAccountId",       # your AWS account / Azure subscription / GCP project
    "BillingPeriodStart",     # ISO date, e.g. 2026-04-01
    "BillingPeriodEnd",       # ISO date, exclusive
    "ChargePeriodStart",      # usually == BillingPeriodStart for daily granularity
    "ChargePeriodEnd",        # usually == BillingPeriodEnd
    "ChargeCategory",         # Usage | Purchase | Tax | Credit | Adjustment
    "ChargeClass",            # Standard | Correction  ← NEW in FOCUS 2026
    "BilledCost",             # decimal — blended/net cost
    "EffectiveCost",          # decimal — after commitment discounts
    "BillingCurrency",        # 3-letter ISO, e.g. USD
    "ProviderName",           # aws | azure | gcp
    "ServiceName",            # e.g. Amazon EC2
]

# Common mistakes:
# - ChargeClass missing → add "Standard" for normal charges, "Correction" for credits
# - EffectiveCost == BilledCost → acceptable when RI/SP data unavailable
# - BillingPeriodEnd > ChargePeriodEnd → flag as data quality issue
```

---

## Interpreting Lumen's Findings

Lumen (the AI layer in Cloud Cost Guard) produces findings with these fields:

```json
{
  "finding_id": "aws-ebs-unattached",
  "title": "12 unattached EBS volumes accumulating cost",
  "severity": "high",        // critical | high | medium | low
  "confidence": "very_high", // very_high | high | medium | low
  "monthly_savings_usd_est": 280.00,
  "risk_level": "Low",       // Low | Medium | High
  "implementation_time": "2-3 hours",
  "suggested_action": "Snapshot and delete unattached EBS volumes.",
  "commands": ["aws ec2 describe-volumes ..."]
}
```

**Reading confidence levels:**
- `very_high` — resource is definitively idle/orphaned; safe to act immediately
- `high` — strong utilization signal over 7+ days; verify in staging first
- `medium` — moderate signal; check with resource owner before acting
- `low` — limited data; investigate before taking action

**Reading risk levels:**
- `Low` — no service impact expected; can proceed without change window
- `Medium` — minor service impact possible; schedule during off-peak
- `High` — significant impact possible; requires change management

---

## Data Pipeline (full end-to-end)

```
FinOps Lite ──────────┐
                       ├──► Cloud Capital Report ──► report.json ──► Cloud Cost Guard Dashboard
FinOps Watchdog ───────┤
                       │
Recovery Economics ────┘

AI Cost Lens ──────────► ai_spend section of report.json
SaaS Cost Analyzer ────► saas_spend section of report.json
K8s Connector ─────────► /api/k8s endpoint (live or synthetic)
```

To regenerate `report.json`:
```bash
finops cost overview --format json > cost.json
finops-watchdog detect --input costs.csv --format json > anomalies.json
recovery-economics model --workload workloads.csv --format json > resilience.json
# Then aggregate into report.json format (see cloud-cost-guard/docs/report-schema.md)
```

---

## Contributing to the Stack

- All tools follow the same pattern: narrow CLI responsibility, JSON output, FOCUS 2026 compatible
- Connectors always have `is_configured()` + synthetic fallback
- Tests use pytest; run `pytest` in any repo root
- Commit messages: `feat:`, `fix:`, `chore:`, `docs:` prefixes
- PRs: one feature per PR; link to FinOps Foundation spec where relevant

---

*Cloud & Capital — Finance · AI · Data · Cloud*  
*https://cloudandcapital.com*
