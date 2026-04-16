# 14 - Process Mining with Zero Code Changes

> Onboarding documentation for external contractors.
> Last updated: 2026-04-07

---

## Table of Contents

1. [Goal](#1-goal)
2. [Architecture](#2-architecture)
3. [Data Source 1: Oracle CDC](#3-data-source-1-oracle-cdc)
4. [Data Source 2: WebSphere Access Logs](#4-data-source-2-websphere-access-logs)
5. [What the Combined View Gives You](#5-what-the-combined-view-gives-you)
6. [What Process Mining Reveals](#6-what-process-mining-reveals)
7. [Correlation Strategy](#7-correlation-strategy)
8. [NiFi Flow Design](#8-nifi-flow-design)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [What This Does NOT Capture](#10-what-this-does-not-capture)

---

## 1. Goal

CMWS is being replaced. Before we build the replacement, we need to understand what the current system actually does -- not what the design documents say it does, but what real people do with it every day.

**Objectives:**

- **Understand the actual business processes executed in CMWS.** Not the intended design, but the reality -- including workarounds, shortcuts, and undocumented workflows that have evolved over 20+ years.
- **Capture every user action and system event.** Complete coverage of who did what, when, and to which document.
- **Feed into UiPath Process Mining** to discover real process flows, variants, bottlenecks, and automation candidates.
- **Zero code modifications to the legacy system.** No JARs. No rebuilds. No EAR redeployments. No Zipkin. No Brave. No instrumentation libraries. Nothing touches the running application.
- **Inform the workflow consolidation project.** The data we collect here directly shapes which workflows get consolidated, which steps get automated, and how the replacement system is designed.

We do not care about performance overhead. We care about understanding.

---

## 2. Architecture

The architecture is deliberately simple. Two data sources, one normalization layer, one analytics tool:

```
Oracle DB -----> CDC (Debezium/LogMiner) -----> NiFi -----> Elasticsearch
                                                                |
WebSphere -----> Access Logs -----> Filebeat -----> Elasticsearch
                                                        |
                                                        v
                                                 UiPath Process Mining
```

That is the entire stack. No application-level tracing. No code instrumentation. No shared trace IDs injected into headers. The two data sources correlate naturally through document IDs, user IDs, and timestamps.

**Why this works for CMWS specifically:**

- CMWS uses a 100% stored procedure database access pattern. Every meaningful action writes to the database. CDC catches all of it.
- CMWS runs on WebSphere, which already has built-in NCSA access logging. Every HTTP request is loggable without configuration changes to the application itself.
- The combination of "what the database recorded" and "what HTTP requests the user made" gives us complete coverage of both system events and user behavior.

---

## 3. Data Source 1: Oracle CDC

See [11 - CDC Event Extraction](11-cdc-event-extraction.md) for full implementation details. This section summarizes what matters for process mining.

### What CDC Captures

CDC reads the Oracle redo logs. Every INSERT, UPDATE, and DELETE executed by the stored procedures shows up as a change event. This captures everything the system does to documents -- without touching a single line of Java code.

### Key Tables

| Table | What It Tells You | Process Mining Value |
|-------|-------------------|---------------------|
| **WORKFLOW_HISTORY** | Every queue event: document reserved, moved, completed, returned. Includes document_id, event_type, user_id, timestamp. | **The gold mine.** This is the primary event log for process mining. Every document lifecycle event is here. |
| **DOCUMENT_QUEUE** | Real-time queue state. Which queue a document is in right now, who owns it. | Queue transitions in real time. Detects when items sit idle. |
| **CASE_DIARY** | User notes and decisions. Free-text entries recorded by examiners. | Decision points. Why an examiner took a particular action. |
| **COB_WT_TASK** | COB-specific work tasks and their completion status. | Subprocess milestones within COB workflows. |
| **EPR_WT_TASK** | EPR-specific work tasks. | Subprocess milestones within EPR workflows. |
| **QR_SELECTED_DOCS** | Quality Review document selections. | QR subprocess tracking. |
| Domain tables (per schema) | Schema-specific processing steps. | Detailed subprocess events for each of the 21 business units. |

### What This Means

Every time a stored procedure writes to WORKFLOW_HISTORY -- "document 45678 moved from ACTIVE to PENDING by JSMITH at 14:34:01" -- CDC captures that event and sends it downstream. Multiply this across all 21 schemas and every table listed above, and you have a complete record of everything the system does.

---

## 4. Data Source 2: WebSphere Access Logs

This is the key insight that makes zero-code process mining possible. WebSphere already logs every HTTP request. We just need to turn that logging on and ship the logs to Elasticsearch.

### What Access Logs Contain

Every time a user clicks something in CMWS, the browser sends an HTTP request to WebSphere. The access log records:

| Field | What It Tells You |
|-------|-------------------|
| **Timestamp** | When the action happened |
| **HTTP method + URL path** | What action was taken |
| **User identity** | Who did it (from the WebSEAL `iv-user` header) |
| **Response code** | Whether it succeeded or failed |
| **Response time** | How long the system took to respond |
| **Query parameters** | Document IDs, queue names, search terms -- embedded in the URL |

### URL Patterns That Map to User Actions

CMWS uses descriptive URL paths. Each path maps directly to a user action:

| URL Pattern | User Action |
|-------------|-------------|
| `/workflow/queueview.go` | User viewed a queue |
| `/workflow/inboxview.go` | User checked their inbox |
| `/workflow/search.go` | User searched for a claim |
| `/workflow/getnext.go` | User clicked "Get Next" to pick up work |
| `/workflow/move.go` | User moved an item to another queue |
| `/workflow/assign.go` | User assigned an item |
| `/workflow/approve.go` | User approved an item |
| `/workflow/deny.go` | User denied an item |
| `/Workflow/GetDocument` | User viewed a document (servlet) |
| `/Workflow/PutDocument` | User uploaded a document |
| `/Workflow/PutLetter` | User generated a letter |
| `/services/WorkflowServiceFacade` | SOAP call (from GIAL or ImageViewer) |
| `/services/CaseManagementFacade` | Case management SOAP call |
| `/services/LCMSFacade` | LCMS-specific SOAP call |
| (20+ additional facades) | System-to-system integration calls |

The `.go` suffix is a Struts 1.x convention -- these are Struts action mappings. The `/Workflow/*` paths are direct servlet mappings. The `/services/*` paths are SOAP web service endpoints. Together, they cover every way a user or external system interacts with CMWS.

### How to Configure WebSphere Access Logging

This is a WebSphere Admin Console configuration change. No application code is modified.

**Steps:**

1. Open the WebSphere Admin Console
2. Navigate to: **Servers > Web Servers > CMWS > NCSA Access Logging**
3. Enable extended log format (W3C Extended Log File Format)
4. Configure the following log fields:

```
date time c-ip cs-method cs-uri-stem cs-uri-query sc-status time-taken cs(iv-user)
```

| Field | Description |
|-------|-------------|
| `date` | Date of the request |
| `time` | Time of the request |
| `c-ip` | Client IP address |
| `cs-method` | HTTP method (GET, POST) |
| `cs-uri-stem` | URL path (e.g., `/workflow/move.go`) |
| `cs-uri-query` | Query string (contains document IDs, queue names) |
| `sc-status` | HTTP response status code |
| `time-taken` | Server processing time in milliseconds |
| `cs(iv-user)` | User identity from WebSEAL iv-user header |

5. Set log location: `/var/log/websphere/cmws_access.log`
6. Save and synchronize the configuration

### Shipping Logs to Elasticsearch

Deploy Filebeat on the WebSphere server with the NCSA access log module:

```yaml
# filebeat.yml (relevant excerpt)
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/websphere/cmws_access.log
    fields:
      source_type: websphere_access
    fields_under_root: true

processors:
  - dissect:
      tokenizer: '%{date} %{time} %{client_ip} %{method} %{uri_stem} %{uri_query} %{status} %{time_taken} %{iv_user}'
      field: "message"
  - script:
      lang: javascript
      source: |
        function process(event) {
          var uri = event.Get("uri_stem");
          var query = event.Get("uri_query");
          // Extract action type from URL
          if (uri.includes("getnext.go")) event.Put("action", "Get Next Item");
          else if (uri.includes("move.go")) event.Put("action", "Move Item");
          else if (uri.includes("approve.go")) event.Put("action", "Approve Item");
          else if (uri.includes("deny.go")) event.Put("action", "Deny Item");
          else if (uri.includes("assign.go")) event.Put("action", "Assign Item");
          else if (uri.includes("search.go")) event.Put("action", "Search");
          else if (uri.includes("inboxview.go")) event.Put("action", "View Inbox");
          else if (uri.includes("queueview.go")) event.Put("action", "View Queue");
          else if (uri.includes("GetDocument")) event.Put("action", "View Document");
          else if (uri.includes("PutDocument")) event.Put("action", "Upload Document");
          else if (uri.includes("PutLetter")) event.Put("action", "Generate Letter");
          else if (uri.includes("/services/")) event.Put("action", "SOAP Call");
          // Extract document ID from query params
          if (query) {
            var docMatch = query.match(/docId=(\d+)/);
            if (docMatch) event.Put("document_id", docMatch[1]);
          }
        }

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "cmws-access-logs-%{+yyyy.MM.dd}"
```

This parses URL paths into human-readable action types and extracts document IDs from query parameters -- all at the Filebeat level, before the data reaches Elasticsearch.

---

## 5. What the Combined View Gives You

When you merge CDC events (what the database recorded) with access log events (what HTTP requests the user made), you get a complete picture of every user session.

### User Journey Reconstruction

Here is what a single examiner's work on one document looks like when both sources are combined:

```
14:32:01 - User JSMITH viewed inbox (inboxview.go)               [access log]
14:32:15 - User JSMITH clicked Get Next (getnext.go) -> got document 45678  [access log]
14:32:16 - [CDC] Document 45678 reserved by JSMITH                [CDC]
14:32:20 - User JSMITH viewed document (GetDocument?docId=45678)   [access log]
14:32:45 - User JSMITH viewed document page 2 (GetDocument?docId=45678&page=2)  [access log]
14:33:10 - User JSMITH searched metadata (search.go?ssn=****)     [access log]
14:34:00 - User JSMITH moved item to Pending queue (move.go)      [access log]
14:34:01 - [CDC] Document 45678 moved to PENDING_QUEUE by JSMITH  [CDC]
14:34:01 - [CDC] WORKFLOW_HISTORY: event_type=MOVED, user=JSMITH, queue=Pending  [CDC]
```

The access logs tell you what the user did in the UI. The CDC events confirm what the system recorded in the database. Together, they reconstruct the complete user journey.

### Process Mining Event Format

UiPath Process Mining expects events in a standard format. Here is how the journey above maps to it:

| Case ID | Activity | Timestamp | Resource | Source |
|---------|----------|-----------|----------|--------|
| 45678 | View Inbox | 14:32:01 | JSMITH | access log |
| 45678 | Get Next Item | 14:32:15 | JSMITH | access log |
| 45678 | Reserved | 14:32:16 | JSMITH | CDC |
| 45678 | View Document | 14:32:20 | JSMITH | access log |
| 45678 | Search Metadata | 14:33:10 | JSMITH | access log |
| 45678 | Move to Pending | 14:34:00 | JSMITH | access log |
| 45678 | Queue: Pending | 14:34:01 | JSMITH | CDC |

The **Case ID** is the document_id. Every event related to that document -- regardless of source -- is grouped under the same case. This is what allows process mining to reconstruct the end-to-end flow for each document.

---

## 6. What Process Mining Reveals

With the combined data feeding into UiPath Process Mining, the tool can automatically discover and analyze:

### 1. The Actual Process Map

Not what we designed, but what people really do. Process mining builds a visual flow from the event data. Every path, every loop, every shortcut that examiners have invented over 20 years becomes visible.

### 2. Process Variants

"60% of examiners view the document before approving, 40% approve without viewing." Is that a problem? Is it efficiency or negligence? The data shows us.

### 3. Human Processing Time

The time between `GetDocument` and the next action = time the examiner spent reviewing the document. The time between `View Inbox` and `Get Next Item` = time deciding which item to pick up. These intervals are invisible without access log data.

### 4. Unnecessary Steps

"Users search 3 times on average before finding the right claim." Is the search broken? Is the UI confusing? "Examiners view the queue listing 5 times between processing items." Are they losing their place?

### 5. Automation Candidates

"90% of Walmart items get transferred to a specific queue within 30 seconds of being viewed." If the classification is that obvious, automate it. "100% of items from source X go through the same 4 steps in the same order." That is a rule, not a decision.

### 6. Rework

"15% of items moved to Completed get moved back to Active within 24 hours." That is rework. Why? Which examiners? Which document types? Which queues?

### 7. Workflow Comparison

"COB takes 3x longer than LRK but has the same number of steps -- COB users wait more." Is that a system issue (slow queries) or a process issue (waiting for external information)?

### 8. Training Needs

"New hires take 4x longer on LCMS claims and have 2x the rework rate." This identifies exactly where training investment pays off.

---

## 7. Correlation Strategy

The two data sources do not share a trace ID. They do not need one. They correlate naturally through three fields that both sources contain.

### Correlation Fields

| Field | CDC Source | Access Log Source |
|-------|-----------|-------------------|
| **document_id** | Column value in table rows (e.g., `WORKFLOW_HISTORY.document_id`) | URL query parameter (e.g., `?docId=12345`) |
| **user_id** | Column value (e.g., `WORKFLOW_HISTORY.user_id`, `DOCUMENT_QUEUE.modified_by`) | WebSEAL `iv-user` header value |
| **timestamp** | Database operation timestamp from the redo log | HTTP request timestamp from the access log |

### How Correlation Works

Events from both sources that share the same `document_id` + `user_id` within a narrow time window (~1 second) are the same logical action seen from two perspectives.

For example, when a user clicks "Move to Pending":
1. The browser sends `POST /workflow/move.go?docId=45678&queue=PENDING` -- captured by the access log
2. The Struts action calls the stored procedure, which writes to WORKFLOW_HISTORY and updates DOCUMENT_QUEUE -- captured by CDC

Both events have `document_id=45678`, `user=JSMITH`, and timestamps within ~1 second of each other. NiFi joins them into a single process event.

**No shared trace ID is needed.** The `document_id + user_id + timestamp` combination is sufficient for process mining correlation. This is why zero-code instrumentation works.

---

## 8. NiFi Flow Design

NiFi normalizes both data sources into a unified process event format for UiPath.

### Flow 1: CDC to Process Events

```
Oracle CDC (Debezium/LogMiner)
    |
    v
ConsumeKafka / QueryDatabaseTable
    |
    v
RouteOnAttribute (by source table name)
    |
    +--> WORKFLOW_HISTORY branch --> ExtractText (document_id, event_type, user_id, timestamp)
    +--> DOCUMENT_QUEUE branch   --> ExtractText (document_id, queue_name, modified_by, modified_date)
    +--> CASE_DIARY branch       --> ExtractText (document_id, entry_type, created_by, created_date)
    +--> Domain table branches   --> ExtractText (relevant fields per table)
    |
    v
UpdateAttribute (map to process event schema: case_id, activity, timestamp, resource, source="CDC")
    |
    v
PutElasticsearchRecord --> cmws-process-events index
```

### Flow 2: Access Logs to Process Events

```
QueryElasticsearchHTTP (from cmws-access-logs index)
    |
    v
EvaluateJsonPath (extract uri_stem, uri_query, iv_user, timestamp, time_taken)
    |
    v
RouteOnAttribute (by uri_stem pattern)
    |
    +--> /workflow/getnext.go   --> activity = "Get Next Item"
    +--> /workflow/move.go      --> activity = "Move Item"
    +--> /workflow/approve.go   --> activity = "Approve Item"
    +--> /workflow/deny.go      --> activity = "Deny Item"
    +--> /Workflow/GetDocument   --> activity = "View Document"
    +--> /Workflow/PutDocument   --> activity = "Upload Document"
    +--> /Workflow/PutLetter     --> activity = "Generate Letter"
    +--> (etc.)
    |
    v
ExtractText (document_id from query params)
    |
    v
UpdateAttribute (map to process event schema: case_id, activity, timestamp, resource, source="access_log")
    |
    v
PutElasticsearchRecord --> cmws-process-events index
```

### Flow 3: Combined Process Events to UiPath

UiPath Process Mining reads directly from the `cmws-process-events` Elasticsearch index. No additional NiFi flow is required for this step -- UiPath has a native Elasticsearch connector.

### Unified Process Event Schema

All events in the `cmws-process-events` index share this structure:

```json
{
  "case_id": "45678",
  "activity": "Move to Pending",
  "timestamp": "2026-04-07T14:34:00.123Z",
  "resource": "JSMITH",
  "source": "access_log",
  "source_detail": "/workflow/move.go",
  "queue": "PENDING",
  "response_time_ms": 245,
  "status_code": 200,
  "schema": "COB"
}
```

---

## 9. Implementation Roadmap

This is a 5-week rollout. Each week builds on the previous and delivers immediate value.

### Week 1: WebSphere Access Logging + Filebeat

**Deliverable:** Every HTTP request to CMWS visible in Elasticsearch.

- Enable NCSA access logging in WebSphere Admin Console (configuration change only)
- Deploy Filebeat on the WebSphere server
- Configure Filebeat to parse access logs and ship to Elasticsearch
- Create Kibana dashboards for raw access log exploration
- **Immediate value:** You can see who is using CMWS, how often, and which URL paths are hit most frequently. This alone reveals usage patterns.

### Week 2: CDC on Tier 1 Tables

**Deliverable:** WORKFLOW_HISTORY and DOCUMENT_QUEUE changes streaming into Elasticsearch.

- Set up CDC (Debezium or LogMiner -- see [doc 11](11-cdc-event-extraction.md) for trade-offs)
- Target WORKFLOW_HISTORY and DOCUMENT_QUEUE across all 21 schemas
- Route CDC events through NiFi into Elasticsearch
- **Immediate value:** Every document lifecycle event is captured. Combined with Week 1 data, you can already correlate user actions with system events.

### Week 3: NiFi Normalization Flows

**Deliverable:** Both data sources normalized into the unified process event format.

- Build NiFi Flow 1 (CDC to process events)
- Build NiFi Flow 2 (access logs to process events)
- Implement URL-to-activity mapping for all known URL patterns
- Implement document_id extraction from query parameters
- Write to the `cmws-process-events` index
- **Immediate value:** A single index contains every event from both sources in a format ready for process mining.

### Week 4: UiPath Process Mining -- First Discovery

**Deliverable:** First process maps generated from real CMWS data.

- Connect UiPath Process Mining to the `cmws-process-events` Elasticsearch index
- Configure case_id, activity, timestamp, and resource mappings
- Run initial process discovery across all 21 schemas
- Generate first variant analysis and bottleneck reports
- **Immediate value:** The first real picture of how CMWS is actually used. Expect surprises.

### Week 5: Expand and Refine

**Deliverable:** Full coverage with refined process models.

- Expand CDC to Tier 2 tables (CASE_DIARY, domain-specific task tables)
- Refine activity naming based on initial process mining results
- Build schema-specific process models (COB vs. EPR vs. LCMS vs. LRK)
- Identify top automation candidates and rework patterns
- **Immediate value:** Detailed, per-workflow process intelligence that directly informs the replacement system design.

---

## 10. What This Does NOT Capture

Zero-code means zero-code. There are things this approach cannot see:

| Gap | Why | Mitigation |
|-----|-----|------------|
| **ImageViewer local operations** | Annotations, page navigation within the Java applet -- these happen client-side with no server request | None needed. These are UI interactions, not business process steps. |
| **Time spent reading a document** | No "user is looking at this page" event | Inferred. The time between `GetDocument` and the user's next action is reading time. Not perfect, but good enough for process mining. |
| **Decisions made outside CMWS** | Phone calls, emails, meetings, paper reviews | Out of scope. Process mining reveals the in-system behavior. Combine with interviews for the full picture. |
| **Oracle stored procedure internals** | CDC sees the DML results (INSERTs, UPDATEs) but not the procedural logic that decided what to write | Sufficient for process mining. We know what happened and when. We do not need to know why the stored procedure made a particular choice. |
| **Failed actions with no DB write** | If a user action fails before the stored procedure runs, CDC will not see it | Access logs capture it. A `500` response code on `/workflow/move.go` means the move failed. The access log records it even though the database does not. |
| **Batch jobs and scheduled tasks** | Jobs that run as system accounts may not have meaningful user context | CDC captures the database changes. The user will be `SYSTEM` or a service account. These show up as system-initiated events in the process map. |

---

## Summary

Two data sources. Zero code changes. Complete process visibility.

- **Oracle CDC** captures everything the system does to documents (database events).
- **WebSphere access logs** capture everything users do in the UI (HTTP requests).
- **NiFi** normalizes both into a unified process event format.
- **Elasticsearch** stores everything.
- **UiPath Process Mining** discovers the real processes, variants, bottlenecks, and automation candidates.

The result: a data-driven understanding of what CMWS actually does -- not what we think it does -- so the replacement system can be designed correctly.
