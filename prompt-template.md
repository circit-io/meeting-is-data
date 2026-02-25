# LLM Topic Classification Prompt Template

This is a genericized version of the system prompt used to classify meeting topics. Adapt the categories to your organization.

## System Prompt

```
You are a meeting analyst. Your task is to extract specific, actionable topics
from meeting summaries or transcripts and return them as structured JSON.

RULES:
1. Return a JSON object with a "topics" array containing up to 3 topic objects.
2. Each topic object has: "name" (string), "confidence" (float 0-1), "is_new" (boolean).
3. Prefer existing topics from the EXISTING_TOPICS list below when a semantic match exists.
4. Set "is_new": true only when no existing topic fits. Be conservative — merging
   into an existing topic is almost always better than creating a new one.
5. Be specific but not hyper-specific. "Authentication Token Migration" is good.
   "Tuesday discussion about maybe changing auth" is bad.
6. Use role references, not personal names (e.g., "the team lead" not "John").
7. Do not include compensation, performance, or HR-related content in topics.
8. If the meeting content is too vague to classify meaningfully, return an empty array.

CATEGORIES (adapt these to your org):
- Client/Customer Issues (e.g., "[Client X] Integration Problem")
- Technical Problems (e.g., "API Rate Limiting", "Database Migration")
- Infrastructure (e.g., "Cloud Cost Optimization", "CI/CD Pipeline")
- Integrations (e.g., "Third-Party API v3 Upgrade")
- Projects (e.g., "Q1 Product Launch", "Mobile App Redesign")
- Tools & Systems (e.g., "Monitoring Stack Upgrade")
- Process (e.g., "Sprint Retrospective Action Items")

RESTRICTED TAGS (never use these — too generic):
- Your company name as a standalone tag
- "Daily Standup", "Weekly Sync", "Team Meeting"
- Any recurring meeting series name

EXISTING_TOPICS (injected at runtime — these are the topics already in your taxonomy):
{existing_topics}
```

## User Prompt

```
Classify the following meeting content into up to 3 topics.

Meeting title: {title}
Date: {date}
Attendees: {count} participants

Content:
{summary_or_transcript}
```

## Expected Output

```json
{
  "topics": [
    { "name": "Authentication Token Migration", "confidence": 0.92, "is_new": false },
    { "name": "Cross-Team Architecture Decision", "confidence": 0.85, "is_new": true },
    { "name": "API v3 Compatibility", "confidence": 0.78, "is_new": false }
  ]
}
```

## Enforcing Structured Output

Different providers offer different mechanisms to guarantee valid JSON:

### OpenAI / Azure OpenAI
Use `response_format: { type: "json_schema", json_schema: { ... } }` to enforce your exact schema. This guarantees valid JSON every time — no parsing failures.

### Anthropic Claude
Use the [tool use trick](https://platform.claude.com/cookbook/tool-use-extracting-structured-json): define a tool with your desired JSON schema as the input schema, then force the model to "call" the tool. The tool input is your structured output. Alternatively, use Claude's native structured output support with `tool_choice: { type: "tool", name: "classify_topics" }`.

### General Fallback
If your provider doesn't support structured output natively, instruct the model to return raw JSON and wrap the call in a try/catch with a retry on parse failure. Add `"Output raw JSON only — no markdown fences, no explanation."` to the system prompt.

## Implementation Notes

### Taxonomy Injection

The `{existing_topics}` placeholder is critical. Before each classification call, query your data warehouse for all existing topic names and inject them into the prompt. This pushes the LLM toward reusing existing tags instead of creating new ones.

Without taxonomy injection, expect a **~1:1 ratio of topics to meetings** (we saw 373 topics for 369 meetings). With it, you'll get better clustering, but orphan rates will still be high (~60% in our experience).

**Format the existing topics as a numbered list** — this gives the model clear anchors:
```
1. Authentication Token Migration
2. API v3 Compatibility
3. Cloud Cost Optimization
...
```

### Next-Level: Similarity Gating

The prompt-only approach has limits. For better taxonomy consistency, add a pre-classification step using embeddings:

1. After the LLM returns candidate topics, embed each one using your embedding model
2. Compute cosine similarity against all existing topic embeddings
3. If similarity > threshold (e.g., 0.85), map to the existing topic
4. Only create a new topic if no match exceeds the threshold

This catches cases like "Auth Token Migration" vs "Authentication Token Migration" that the LLM might treat as separate topics.

**Alternative approach (BERTopic-inspired):** Use [BERTopic](https://maartengr.github.io/BERTopic/getting_started/representation/llm.html) as a post-processing step. BERTopic clusters documents first, then uses an LLM to generate human-readable topic labels for each cluster — reversing the typical flow.

### Copilot / AI Summary vs Raw Transcript

If your meeting platform provides AI-generated summaries (e.g., Microsoft Copilot, Otter.ai), use those as input instead of raw transcripts:

- **Better signal-to-noise ratio** — summaries strip filler, crosstalk, off-topic tangents
- **Lower token cost** — summaries are 10-50x shorter than full transcripts
- **Faster processing** — fewer tokens = lower latency per API call

Fall back to raw transcripts only when no AI summary is available.

### Chain-of-Thought for Classification

For simple topic classification, **direct classification outperforms chain-of-thought** in our testing. CoT adds tokens (= cost) without meaningfully improving accuracy for this task. Reserve CoT for complex reasoning — use direct JSON output for classification.

If you do want reasoning (e.g., for audit trails), ask for it in a separate field:
```json
{
  "reasoning": "The meeting focused on...",
  "topics": [...]
}
```

### Model Selection

We use GPT-5 via Azure OpenAI, but any instruction-following LLM works. The classification task is straightforward — GPT-4o, Claude Haiku/Sonnet, or similar models produce comparable results at lower cost. **Use the cheapest model that reliably produces valid JSON.** For most orgs, that's GPT-4o-mini or Claude Haiku.

### Cost Optimization

- **Batch classification:** If your provider supports it, batch multiple meetings into a single API call (e.g., "Classify these 5 meetings" with numbered outputs). Amortizes system prompt tokens across multiple inputs.
- **Cache system prompts:** Use provider-specific prompt caching (Azure OpenAI prompt caching, Anthropic prompt caching) to reduce input token costs on repeated calls with the same system prompt.
- **Minimize taxonomy injection:** If your topic list exceeds ~200 items, inject only the top 50 most recent/frequent topics. The LLM doesn't need the full taxonomy for every call.
