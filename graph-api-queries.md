# Microsoft Graph API Queries for Meeting Data

These are the Graph API endpoints used to extract meeting data from Microsoft Teams. All require appropriate application permissions in Azure AD.

> **Billing update (August 2025):** Microsoft ceased charging for metered Graph APIs including Teams exports, transcripts, and meeting recordings — removing the cost barrier for apps using these endpoints.

## Prerequisites

- Azure AD App Registration with the following **application** permissions:
  - `OnlineMeetings.Read.All`
  - `Calendars.Read`
  - `OnlineMeetingTranscript.Read.All`
  - `OnlineMeetingArtifact.Read.All` (for attendance reports)
  - `Files.Read.All` (for Loop/OneDrive meeting notes)
  - `InformationProtectionPolicy.Read.All` (for sensitivity labels)
- **Copilot AI Insights:** Delegated permissions only (no application support). Requires `Microsoft 365 Copilot` license on the signed-in user.
- OAuth 2.0 client credentials flow for authentication
- [Application access policy](https://learn.microsoft.com/en-us/graph/cloud-communication-online-meeting-application-access-policy) granted by tenant admin for transcript/recording endpoints
- Secrets stored in a key vault (never hardcode)

## Authentication

```http
POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id={client_id}
&scope=https://graph.microsoft.com/.default
&client_secret={client_secret}
&grant_type=client_credentials
```

## 1. List Calendar Events (Meetings)

Fetch meetings for a specific user within a date range:

```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/calendarView
    ?startDateTime=2026-01-01T00:00:00Z
    &endDateTime=2026-02-01T00:00:00Z
    &$filter=isOnlineMeeting eq true
    &$select=subject,start,end,organizer,attendees,onlineMeeting,sensitivity
    &$top=100
```

**Key fields for filtering:**
- `sensitivity` — use to exclude Confidential/Highly Confidential meetings
- `isOnlineMeeting` — ensures you only process Teams meetings
- `attendees` count — use to exclude 1-on-1s (2 attendees)

## 2. Get Online Meeting Details

```http
GET https://graph.microsoft.com/v1.0/users/{organizer_id}/onlineMeetings
    ?$filter=joinWebUrl eq '{join_url}'
    &$select=id,subject,startDateTime,endDateTime,participants,sensitivityLabelAssignment
```

The `sensitivityLabelAssignment` property (added 2025) returns the Purview sensitivity label applied to the meeting. Use this for compliance filtering.

## 3. Get Meeting Transcripts (GA in v1.0)

**List transcripts for a meeting:**
```http
GET https://graph.microsoft.com/v1.0/users/{organizer_id}/onlineMeetings/{meeting_id}/transcripts
```

**Get transcript content (VTT format):**
```http
GET https://graph.microsoft.com/v1.0/users/{organizer_id}/onlineMeetings/{meeting_id}/transcripts/{transcript_id}/content
    ?$format=text/vtt
```

**Get structured metadata content (JSON within VTT):**
```http
GET https://graph.microsoft.com/v1.0/users/{organizer_id}/onlineMeetings/{meeting_id}/transcripts/{transcript_id}/metadataContent
```

Returns structured JSON per utterance:
```json
{"startDateTime":"2026-02-08T08:22:30Z","endDateTime":"2026-02-08T08:22:31Z","speakerName":"User Name","spokenText":"This is a transcription test.","spokenLanguage":"en-us"}
```

**Bulk retrieval across all meetings (getAllTranscripts):**
```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/onlineMeetings/getAllTranscripts(meetingOrganizerUserId='{user_id}',startDateTime={ISO8601},endDateTime={ISO8601})
```

**Delta sync (incremental updates):**
```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/onlineMeetings/getAllTranscripts/delta
```

> **Note:** Transcripts are only available if recording/transcription was enabled. Content format `text/vtt` is the only supported format (docx was removed May 2023). Meeting resources expire 60 days after start/end time.

## 4. Get Copilot / AI Insights (GA in v1.0 under /copilot)

```http
GET https://graph.microsoft.com/v1.0/copilot/users/{user_id}/onlineMeetings/{meeting_id}/aiInsights
```

Returns structured AI-generated content:

| Field | Description |
|-------|-------------|
| `meetingNotes` | AI-generated summary with titled sections and subpoints |
| `actionItems` | Extracted follow-ups with `ownerDisplayName` |
| `viewpoint.mentionEvents` | When specific participants were mentioned, with timestamps |

**Use this as your preferred input** for topic classification — it's cleaner and cheaper than raw transcripts.

**Limitations:**
- **Delegated permissions only** — application permissions are not supported
- Requires **Microsoft 365 Copilot license** on the signed-in user
- Insights take up to **4 hours** to become available after the meeting ends
- Does not support channel meetings
- Transcription or recording must be turned on during the meeting

## 5. Get Meeting Attendance Reports (GA in v1.0)

```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/onlineMeetings/{meeting_id}/attendanceReports/{report_id}
    ?$expand=attendanceRecords
```

Returns per-attendee data: email, role, total attendance in seconds, and join/leave intervals. Useful for understanding meeting engagement patterns.

## 6. Get Loop Meeting Notes (via OneDrive)

Loop stores meeting notes as `.loop` files in OneDrive for Business. **There is no dedicated Loop API yet** — Microsoft plans initial Graph APIs for Loop workspace management in H1 CY2026.

**Search for .loop files:**
```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/drive/root/search(q='.loop')
    ?$select=name,lastModifiedDateTime,webUrl,size
    &$top=100
```

**Download file content:**
```http
GET https://graph.microsoft.com/v1.0/users/{user_id}/drive/items/{item_id}/content
```

> **Note:** `.loop` files are in a proprietary format and not easily human-readable without Loop's rendering layer. For programmatic content extraction, prefer the **Transcripts API** or **AI Insights API**.

## 7. Sensitivity Labels (via Microsoft Purview)

**List all sensitivity labels in the tenant:**
```http
GET https://graph.microsoft.com/v1.0/security/dataSecurityAndGovernance/sensitivityLabels
```

**Filter labels applicable to Teams meetings:**
```http
GET https://graph.microsoft.com/v1.0/security/dataSecurityAndGovernance/sensitivityLabels
    ?$filter=applicableTo has 'teamwork'
```

> **Limitation:** You cannot filter meetings by sensitivity label server-side. Retrieve meetings first, include `sensitivityLabelAssignment` via `$select`, then filter client-side.

## 8. List Groups / Teams (for batch processing)

```http
GET https://graph.microsoft.com/v1.0/groups
    ?$filter=resourceProvisioningOptions/any(x:x eq 'Team')
    &$select=id,displayName,mail
    &$top=100
```

## Compliance Filtering Logic

Apply these filters **before** sending any data to the LLM:

```
EXCLUDE IF:
  - sensitivityLabelAssignment matches Confidential/Highly Confidential
  - attendees.count == 2                    # 1-on-1s
  - organizer.department IN ('HR', 'Legal') # sensitive departments
  - subject MATCHES ('1:1', '1-on-1', 'performance review', 'salary', 'compensation')
  - isRecordingEnabled == false             # no consent signal
```

## Pagination & Batch Requests

### Pagination
Graph API uses server-driven pagination with `@odata.nextLink`. Follow the full URL — do not extract or manipulate the `$skiptoken` value. Continue requesting until no `@odata.nextLink` is returned.

### Batch Requests
Group up to 20 requests per batch:

```http
POST https://graph.microsoft.com/v1.0/$batch
Content-Type: application/json

{
  "requests": [
    { "id": "1", "method": "GET", "url": "/users/{userId}/onlineMeetings/{meeting1}/transcripts" },
    { "id": "2", "method": "GET", "url": "/users/{userId}/onlineMeetings/{meeting2}/transcripts" },
    { "id": "3", "method": "GET", "url": "/users/{userId}/onlineMeetings/{meeting3}/attendanceReports" }
  ]
}
```

Use `dependsOn` for sequential requests within a batch.

## Rate Limiting

| Scope | Limit |
|-------|-------|
| Meeting information | 2,000 meetings/user/month |
| Calls (per app per tenant) | 50,000 requests/15 seconds |
| Call records (per app per tenant) | 1,500 requests/20 seconds |
| Global (per app, all tenants) | 130,000 requests/10 seconds |

> **September 2025 change:** Per-app/per-user/per-tenant throttling limit reduced to half of the total per-tenant limit.

**Best practices:**
- Implement exponential backoff on `429` responses (use `Retry-After` header)
- Use `$select` to request only needed properties
- Use delta queries for incremental sync
- Cache transcript content locally after first retrieval
- Process during off-peak hours (we run nightly at 02:00)

## New Endpoints Added in 2025-2026

| Feature | Status | Version |
|---------|--------|---------|
| Meeting AI Insights (Copilot) | GA | v1.0 `/copilot` |
| Sensitivity label on meetings | GA | v1.0 |
| Ad hoc call transcripts | GA | v1.0 |
| Transcript metadataContent | GA | v1.0 |
| Meeting expiry property | GA | v1.0 |
| contentCorrelationId filtering | GA | v1.0 |
| Town hall attendance reports | GA | v1.0 |
| Q&A / Engagement APIs | Preview | beta |

## References

- [Online Meetings API](https://learn.microsoft.com/en-us/graph/api/resources/onlinemeeting)
- [Meeting Transcripts API](https://learn.microsoft.com/en-us/graph/api/resources/calltranscript)
- [AI Insights API](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/meeting-transcripts/meeting-insights)
- [Attendance Reports API](https://learn.microsoft.com/en-us/graph/api/meetingattendancereport-get)
- [Sensitivity Labels API](https://learn.microsoft.com/en-us/graph/api/resources/security-sensitivitylabel)
- [Graph API Throttling Limits](https://learn.microsoft.com/en-us/graph/throttling-limits)
- [JSON Batching](https://learn.microsoft.com/en-us/graph/json-batching)
