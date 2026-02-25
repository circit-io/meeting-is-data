# Cost Estimation Template

Real numbers from a 30-engineer, 3-team org running this system since January 2026. Use these as a baseline to estimate your own costs.

## Our Actual Costs (EUR/month)

| Component | Jan 2026 | Feb 2026 | Notes |
|-----------|----------|----------|-------|
| Azure OpenAI | 13.53 | 31.07 | GPT-5, ~22-48M tokens/mo |
| Total infrastructure | 149.87 | 124.74 | All pipeline + storage + compute |
| **OpenAI as % of total** | **9.0%** | **24.9%** | LLM is a fraction of the bill |

**"Less than a team lunch."**

> **Graph API cost update (August 2025):** Microsoft removed charges for metered Graph APIs including Teams exports, transcripts, and recordings. One fewer line item to worry about.

## Cost Breakdown by Component

### LLM API (Azure OpenAI / OpenAI / Anthropic)

```
Cost = (input_tokens + output_tokens) x price_per_token

Our numbers:
- ~370 meetings/month
- Average input: ~2,000 tokens/meeting (Copilot summary)
               ~20,000 tokens/meeting (raw transcript fallback)
- Average output: ~200 tokens/meeting (3 topics + summary)
- Prompt:output ratio: ~90-130:1

Estimate for your org:
  meetings_per_month x avg_tokens_per_meeting x token_price
```

**Optimization levers:**
1. **Use Copilot/AI summaries instead of raw transcripts** — reduces input tokens by 10-50x. This is the single biggest cost lever.
2. **Use the cheapest capable model** — GPT-4o-mini or Claude Haiku for classification. Save GPT-5/Claude Opus for complex summarization.
3. **Prompt caching** — Azure OpenAI and Anthropic both offer prompt caching. Since your system prompt is identical across calls, cache it.
4. **Batch classification** — Group 3-5 meetings into a single API call to amortize system prompt tokens.

### Data Warehouse (Synapse / BigQuery / Snowflake)

- Storage: Parquet files, ~1 KB per meeting → negligible
- Compute: SQL queries on dashboard load → depends on query frequency and complexity
- Our cost: ~EUR 50-80/mo (Synapse serverless, pay-per-query)
- **Tip:** Synapse serverless charges per TB scanned. Partition your Parquet files by month and use `WHERE meetingDate >= ...` filters to minimize scan volume.

### Pipeline Orchestrator (ADF / Airflow / Prefect)

- One nightly run, ~15 min duration
- ADF: ~EUR 10-20/mo for pipeline execution (~1,000 activity runs/day at $0.001/run)
- Self-hosted Airflow: cost of the VM/container
- **Note:** ADF charges per activity run ($0.001 for orchestration activities, $0.25/DIU-hour for data movement). For a pipeline with 18 activities running nightly, expect ~EUR 10-15/mo.

### Blob Storage

- Raw JSON + Parquet files: < 1 GB total
- Cost: < EUR 1/mo

### Grafana (if managed)

- Azure Managed Grafana: ~EUR 30/mo (Essential tier)
- Self-hosted: cost of the VM/container
- Grafana Cloud: free tier may suffice for small teams

## LLM-on-Dashboard-Load Costs

Our deep dive dashboard makes 4 LLM API calls per page load (executive status, blockers, next steps, knowledge graph). These are cached for 5 minutes in Grafana.

```
Cost per page load: ~4 x 3,000 tokens x $0.01/1K = ~$0.12
With 5-min cache, 10 users/day: ~$2.40/day = ~$72/month
```

**Scaling concern:** This model works for small teams. At 100+ daily users, switch to precomputed summaries (run LLM in the nightly pipeline, store results, serve statically).

## Cost Estimation Worksheet

Fill in your numbers:

```
MONTHLY COST ESTIMATE

Meetings per month:          ___
Avg attendees per meeting:   ___
% with Copilot summaries:    ___

LLM Processing:
  Meetings x tokens x price = ___/mo

Dashboard LLM (if live):
  Page loads/day x 4 calls x tokens x price = ___/mo

Data Warehouse:
  Storage + compute = ___/mo

Pipeline:
  Orchestrator runs = ___/mo

Monitoring:
  Grafana/dashboards = ___/mo

────────────────────────────
ESTIMATED TOTAL:            ___/mo
```

## When Does It Stop Being Cheap?

The inflection points:
- **1,000+ meetings/month** — LLM processing cost becomes noticeable. Consider batching, smaller models for classification, or caching.
- **100+ daily dashboard users** — LLM-on-load doesn't scale. Precompute summaries nightly.
- **Multiple LLM providers** — If you add embedding for similarity gating, that's a second API cost. Budget accordingly.
- **Raw transcripts with no Copilot** — If < 20% of your meetings have Copilot summaries, expect 5-10x higher LLM costs. This is the variable that matters most.

At our scale (370 meetings/mo, ~10 dashboard users), the total cost is comparable to a single SaaS tool license. The ROI comes from the first time someone doesn't schedule a meeting because the answer was already on the dashboard.
