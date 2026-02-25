# Data Pipeline Skeleton

This describes the pipeline pattern for ingesting, classifying, and storing meeting data. Adapt to your orchestrator (ADF, Airflow, Prefect, Dagster, etc.).

> **ADF vs Fabric Data Factory (2026):** Microsoft has shifted focus to Fabric Data Factory (FDF). ADF still works and receives maintenance updates, but significant new features ship in FDF. If starting fresh, evaluate FDF. If already on ADF, there's a [migration assessment tool](https://learn.microsoft.com/en-us/azure/data-factory/how-to-assess-your-azure-data-factory-to-fabric-data-factory-migration) and a PowerShell conversion utility. No rush to migrate — ADF is not being deprecated.

## Pipeline 1: Meeting Insights (18 activities)

```
┌─────────────────────────────────────────────────────────────┐
│ STAGE 1: Authentication                                     │
│                                                             │
│  GetClientId ──> GetClientSecret ──> GetTenantId            │
│       │              │                    │                  │
│       └──────────────┴────────────────────┘                  │
│                      │                                       │
│              GetOAuthToken (client_credentials)              │
│                      │                                       │
│              SetTokenVariable                                │
│                      │                                       │
│       ┌──────────────┴──────────────┐                        │
│       │  RefreshTokenLoop (Until)   │  ← keeps token alive   │
│       │  during long-running batch  │    for duration of run │
│       └─────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 2: Collection                                         │
│                                                             │
│  ForEachUser ──────> Fetch calendar events (Graph API)      │
│  ForEachGroup ─────> Fetch group meetings (Graph API)       │
│       │                                                     │
│  LookupMeetings ───> Combine and deduplicate                │
│       │                                                     │
│  FilterOnlineMeetings ──> Apply compliance exclusion rules  │
│       │                                                     │
│  SkipAlreadyProcessed ──> Check against processed IDs       │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 3: Taxonomy Prep                                      │
│                                                             │
│  ExtractExistingTopics ──> Dataflow: query existing tags    │
│  LoadAggregatedTopics ──> Lookup: get topic list            │
│  SetExistingTopicsPrompt ──> Inject into LLM system prompt  │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 4: Processing (ForEach meeting)                       │
│                                                             │
│  TryGetAiInsights ──> Attempt Copilot/Facilitator summary   │
│       │                                                     │
│       ├── YES ──> CallLLM_FromAiInsights                    │
│       │                  │                                   │
│       │           WriteEnrichedJson ──> blob storage         │
│       │                                                     │
│       └── NO ──> GetTranscripts                             │
│                      │                                       │
│               CallLLM_FromTranscripts                        │
│                      │                                       │
│               WriteEnrichedJson ──> blob storage             │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│ STAGE 5: Storage                                            │
│                                                             │
│  TransformToParquet ──> Dataflow: JSON ──> Parquet          │
│       │                                                     │
│  Load to warehouse ──> Synapse / BigQuery / Snowflake       │
│       │                                                     │
│  Unified SQL view: MeetingCombinedData                      │
└─────────────────────────────────────────────────────────────┘
```

### Stage-by-Stage Notes

**Stage 1 — Authentication:**
OAuth tokens expire in ~60 minutes. For pipelines processing hundreds of meetings, use an Until loop that refreshes the token every 50 minutes. In ADF, store secrets in Azure Key Vault and reference them via linked services — never hardcode credentials.

**Stage 2 — Collection:**
The idempotency check (`SkipAlreadyProcessed`) is critical. Track processed meeting IDs in your warehouse. On re-runs, skip meetings that already have enriched output. This prevents duplicate processing and keeps LLM costs predictable.

**Stage 3 — Taxonomy Prep:**
Query your warehouse for all existing topic names before the processing loop. Inject this list into the LLM system prompt. This single step is the difference between ~1:1 topic-to-meeting ratio and meaningful topic clustering.

**Stage 4 — Processing:**
The AI Insights (Copilot summary) → Transcript fallback pattern saves significant token cost. Copilot summaries are ~2,000 tokens vs ~20,000 for raw transcripts. Wrap each meeting in a try/catch — one bad transcript shouldn't block the rest.

**Stage 5 — Storage:**
Parquet is the recommended format for analytical workloads. If you need ACID transactions, schema enforcement, or time travel (e.g., for audit requirements), consider [Delta Lake](https://delta.io/) — it wraps Parquet with a transaction log.

## Pipeline 2: Loop Meeting Notes (14 activities)

Same auth pattern, then:

```
CopyFilesList (OneDrive API) ──> LookupFiles ──> FilterByDate
    │
    ▼
ForEachFile:
    ReadFileContent (HTML) ──> ParseHTML ──> CallLLM ──> WriteEnrichedJson
    │
    ▼
TransformToParquet ──> Load to warehouse
```

> **Note:** Loop files (.loop) are in a proprietary format. If you previously accessed them as .fluid files, those have been migrated. Parse the HTML content to extract text before sending to the LLM.

## Enriched JSON Schema

Each processed meeting produces a JSON blob with this structure:

```json
{
  "meetingId": "abc-123",
  "title": "Sprint Planning — Team Alpha",
  "date": "2026-02-20T10:00:00Z",
  "duration_minutes": 45,
  "attendee_count": 8,
  "source": "teams_copilot",
  "topics": [
    { "name": "Sprint Planning Process", "confidence": 0.92, "is_new": false },
    { "name": "API v3 Migration", "confidence": 0.87, "is_new": false },
    { "name": "Cloud Cost Review", "confidence": 0.78, "is_new": true }
  ],
  "summary": "Team discussed sprint priorities...",
  "action_items": [
    "Follow up with vendor on API access",
    "Schedule architecture review"
  ],
  "blockers": [
    "Test environment access delayed"
  ],
  "sentiment": "productive",
  "processed_at": "2026-02-21T02:15:00Z",
  "model": "gpt-5",
  "tokens_used": 1847
}
```

## Scheduling

- **Frequency:** Nightly (e.g., 02:00 local time)
- **Why nightly, not real-time:** Meetings need to end before transcripts are available. Copilot AI Insights take up to 4 hours to appear. Batch processing is simpler and cheaper.
- **Idempotency:** Track processed meeting IDs. Skip already-processed meetings on re-runs.
- **Trigger:** ADF schedule trigger or Airflow `@daily` DAG schedule. Consider event-based triggers only if you need sub-day latency.

## Error Handling

- Wrap the ForEach processing loop in try/catch
- Log failures per meeting but don't stop the pipeline — one bad transcript shouldn't block 50 good ones
- Alert on **pipeline failure** (not per-meeting failure)
- Set a timeout on LLM calls (30s is usually sufficient for classification)
- Implement exponential backoff for Graph API 429 responses
- For ADF specifically: set `isSequential: false` and `batchCount: 10` on ForEach to parallelize meeting processing while respecting API rate limits

## Metadata-Driven Pattern

For production pipelines processing multiple data sources, consider the [metadata-driven approach](https://learn.microsoft.com/en-us/azure/data-factory/copy-data-tool-metadata-driven): store source configurations in a control table and drive a single parameterized pipeline. This prevents pipeline sprawl as you add new meeting data sources (e.g., Zoom, Google Meet).

## Cost Drivers

| Component | Cost Driver | Optimization |
|-----------|-------------|-------------|
| LLM API | Tokens per meeting | Use Copilot summaries (10-50x fewer tokens than transcripts) |
| Data warehouse | Storage + query volume | Parquet format, partition by month |
| Orchestrator | Pipeline runs | Nightly batch, not per-meeting triggers |
| Blob storage | Raw JSON + Parquet files | Lifecycle policies (delete raw after 90 days) |
| Graph API | Requests per second | Batch requests (20/batch), delta sync for incremental updates |
