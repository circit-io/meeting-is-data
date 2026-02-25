# Privacy & Compliance Checklist

Use this checklist before building a meeting data pipeline. No single layer is reliable alone — that's why you need multiple.

> **Regulatory context (as of early 2026):** The EU AI Act is in effect with obligations phasing in through 2027. The EDPB published a [coordinated enforcement report on the right to erasure](https://www.edpb.europa.eu/news/news/2026/edpb-identifies-challenges-hindering-full-implementation-right-erasure_en) (Feb 2026) highlighting challenges with backup deletion, anonymization, and manual processing. Colorado's AI Act takes effect June 2026. Illinois HB 3773 (AI in employment) took effect January 2026.

## Layer 1: Pre-Pipeline Gatekeeping (STRONGEST)

Platform-enforced controls. These prevent data from ever entering your pipeline.

- [ ] **Sensitivity labels configured** — MS Purview labels mark Confidential / Highly Confidential meetings. Your pipeline reads `sensitivityLabelAssignment` from the Graph API and filters these out at ingestion.
- [ ] **DLP policies for Copilot** — Configure Purview DLP policies to block Copilot from referencing files with sensitive labels. This prevents sensitive content from appearing in AI-generated meeting summaries.
- [ ] **1-on-1 exclusion** — Meetings with exactly 2 attendees are excluded by default.
- [ ] **Department exclusion** — Meetings organized by HR, Legal, or Finance departments are excluded.
- [ ] **Title-based exclusion** — Meetings matching patterns like "performance review", "salary", "1:1", "disciplinary" are excluded.
- [ ] **Recording/transcription consent** — Only meetings where Teams recording/transcription notification was shown (implicit consent) are processed. Note: as of November 2025, Copilot in Teams no longer auto-enables persisted transcripts.
- [ ] **Channel meeting exclusion** — Consider excluding channel meetings if your compliance requirements demand it (several Graph API endpoints don't support them anyway).

## Layer 2: Post-Meeting Consent (AUTOMATED)

Gives meeting owners control over what gets shared.

- [ ] **Automated consent email** — After each meeting, the organizer receives an email: "Can this meeting's insights be shared with the wider team?"
- [ ] **Default: no sharing** — If no response within N days, the meeting is NOT published. Opt-in, not opt-out.
- [ ] **Exclusion respected** — If the organizer says no, all data for that meeting is purged from the pipeline output.
- [ ] **Consent audit trail** — Log consent decisions (who, when, what meeting) for compliance.

## Layer 3: PII Controls (WEAKEST — needs multiple sub-layers)

Preventing personal information from appearing in LLM output.

- [ ] **LLM system prompt** — Instructs the model: "Use role references, not personal names. Do not include compensation, performance, or personal information."
- [ ] **LLM provider PII filter** — Enable your LLM provider's content filtering (e.g., Azure OpenAI Content Filtering for PII).
- [ ] **Pre-LLM PII detection** (recommended) — Run PII detection on input text before sending to the LLM. Options:
  - [Microsoft Presidio](https://github.com/microsoft/presidio) — open-source, supports spaCy and Huggingface Transformer NER models, context-aware detection, handles redaction/masking/encryption
  - [Azure AI Language PII detection](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview) — managed service, strong for structured PII (emails, phone numbers, addresses)
  - [spaCy NER](https://spacy.io/) with custom models — good for domain-specific entity recognition
- [ ] **Output validation** — Spot-check LLM outputs periodically. The system prompt is the weakest layer — it will occasionally leak names.

## Layer 4: Governance

Data lifecycle and compliance controls.

- [ ] **No audio stored** — Pipeline processes transcripts/summaries only. No audio recordings are retained.
- [ ] **Retention policies** — Raw transcripts and intermediate JSON are deleted after a defined period (e.g., 90 days). Graph API meeting resources expire after 60 days automatically.
- [ ] **Right to erasure** — Maintain a participant-to-meeting mapping so you can delete all data referencing a specific person on request (GDPR Article 17). The EDPB's Feb 2026 report highlights that backup deletion and anonymization are the most common compliance gaps.
- [ ] **Access control** — Dashboard access is scoped. Not everyone sees everything. Use your existing RBAC.
- [ ] **Audit trail** — Log who accessed what data and when. Standard for SOC2 / ISO 27001.
- [ ] **Data minimization** — Only extract and store the fields you actually need. Don't bulk-download full transcripts if summaries suffice.

## Layer 5: EU AI Act Considerations

If you operate in the EU or process data of EU residents:

- [ ] **Risk classification** — Determine if your meeting data pipeline constitutes a "high-risk AI system" under the EU AI Act. Meeting topic classification for internal dashboards is generally **not** high-risk, but check if the output influences employment decisions (which would make it high-risk).
- [ ] **Transparency obligation** — If your system is AI-powered, inform employees that AI is processing meeting data and explain what it does. This is required regardless of risk level.
- [ ] **Human oversight** — Ensure humans can review and override AI-generated topic classifications. Don't use LLM output as the sole input for any consequential decision.

## The Honest Assessment

| Layer | Reliability | Why |
|-------|-------------|-----|
| Sensitivity labels | High | Platform-enforced by Microsoft, not by your code |
| Metadata exclusion | High | Deterministic rules, no AI involved |
| DLP policies | High | Platform-enforced content restrictions |
| Consent email | Medium | Depends on organizer response rate |
| LLM system prompt | Low | LLMs don't reliably follow instructions 100% of the time |
| PII output filter | Medium | Catches common patterns, misses edge cases |
| Governance | High | Standard infra controls, well-understood |

**Bottom line:** Start from what you're NOT allowed to capture. Work backwards from there. The strongest controls are the ones that prevent data from entering the pipeline at all.

## Before You Launch

- [ ] Legal review of data processing scope
- [ ] Works council / employee representative consultation (required in EU — non-negotiable)
- [ ] Privacy Impact Assessment (PIA/DPIA) if processing employee communications
- [ ] Update your data processing register
- [ ] Communicate to employees: what's captured, what's not, how to opt out
- [ ] Check jurisdiction-specific requirements: Colorado AI Act (June 2026), Illinois HB 3773 (January 2026) if you have US employees
- [ ] Document your AI governance and risk-management program (required by Colorado AI Act for "high-risk" systems)
