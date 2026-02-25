# Meeting Is Data — Starter Kit

Resources from the conference talk **"Meeting Is Data: How to Extract Shared Context from Everyday Conversations"** by Jakub Sikora, VP Engineering at Circit.

## What's Here

| File | Description |
|------|-------------|
| [prompt-template.md](prompt-template.md) | LLM system prompt for topic classification with structured JSON output, taxonomy injection, and similarity gating |
| [graph-api-queries.md](graph-api-queries.md) | Microsoft Graph API endpoints for meetings, transcripts, Copilot AI insights, attendance reports, and sensitivity labels |
| [pipeline-skeleton.md](pipeline-skeleton.md) | Data pipeline pattern (ADF/Airflow) — 18-activity flow with error handling and cost optimization |
| [privacy-checklist.md](privacy-checklist.md) | 5-layer privacy framework including EU AI Act considerations and jurisdiction-specific requirements |
| [cost-estimation.md](cost-estimation.md) | Cost model with real EUR numbers, optimization levers, and scaling inflection points |
| [grafana-dashboard.json](grafana-dashboard.json) | Starter dashboard JSON with Topics Overview, Deep Dive LLM panels, Knowledge Graph, and subscription flow |

## The Architecture Pattern

```
Data Sources          Processing           Intelligence         Presentation
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│ MS Teams     │────>│ ADF/Airflow  │────>│ Azure OpenAI │────>│ Synapse/    │
│ (Graph API)  │     │              │     │ (GPT-4o/5)   │     │ BigQuery    │
├─────────────┤     │ Nightly batch│     │              │     ├─────────────┤
│ MS Loop      │────>│              │     │ Topic class. │     │ Grafana     │
│ (OneDrive)   │     └──────────────┘     │ Summarization│     │ Tableau     │
└─────────────┘                           └──────────────┘     │ Power BI    │
                                                               │ Any SQL     │
                                                               └─────────────┘
```

**The key design decision:** Everything lands in one unified SQL view (`MeetingCombinedData`). Every downstream tool reads from that single source. No data silos, no sync issues.

## How to Use This

1. **Start without infrastructure.** Do the "Monday Morning" exercise from the talk first — count meetings, ask one lead what they missed, list 10 topics.
2. **Pick your stack.** The pattern is cloud-agnostic. Swap ADF for Airflow, Synapse for BigQuery, Azure OpenAI for any LLM API.
3. **Start with the prompt.** The classification prompt is the most immediately reusable artifact. Test it against a few meeting transcripts before building pipelines.
4. **Privacy first.** Use the checklist before writing any code. Know what you're NOT capturing before you decide what to capture.

## Talk Reference Numbers

These are real numbers from a 30-engineer, 3-team org running this system since January 2026:

- **369** meetings processed in ~2 months
- **373** topics classified (60% orphans — the hardest unsolved problem)
- **~2 weeks** from idea to first dashboard
- **~EUR 30/mo** LLM cost (Azure OpenAI)
- **~EUR 150/mo** total infrastructure cost
- **$0 Graph API costs** (Microsoft removed metered API charges in August 2025)

## What's Changed Since the Talk

This starter kit incorporates research and updates through February 2026:
- Graph API: AI Insights API is now GA under `/copilot` namespace (was beta)
- Graph API: Metered API charges removed (August 2025)
- Teams: Copilot no longer auto-enables persisted transcripts (November 2025)
- Privacy: EU AI Act obligations phasing in; EDPB right-to-erasure enforcement report published
- Privacy: Colorado AI Act (June 2026) and Illinois HB 3773 (January 2026) adding US requirements
- Grafana: Dashboard schema v2 available; Infinity datasource UQL support for LLM response parsing
- ADF: Fabric Data Factory is the strategic direction; migration tools available

## License

These templates are provided as-is for educational purposes. Adapt them to your own infrastructure, compliance requirements, and organizational context.
