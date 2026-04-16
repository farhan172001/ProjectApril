# 11 - CDC Event Extraction for Process Mining

> Onboarding documentation for external contractors.
> Last updated: 2026-04-07

---

## Table of Contents

1. [Overview](#1-overview)
2. [Tables to Monitor -- Tier 1 (Critical for Process Mining)](#2-tables-to-monitor----tier-1-critical-for-process-mining)
3. [Tables to Monitor -- Tier 2 (Rich Process Context)](#3-tables-to-monitor----tier-2-rich-process-context)
4. [Tables to Monitor -- Tier 3 (Domain-Specific Events)](#4-tables-to-monitor----tier-3-domain-specific-events)
5. [Process Mining Event Mapping](#5-process-mining-event-mapping)
6. [CDC Implementation Options](#6-cdc-implementation-options)
7. [Schema Considerations](#7-schema-considerations)
8. [Volume Estimates](#8-volume-estimates)
9. [Data Quality Considerations](#9-data-quality-considerations)

---

## 1. Overview

### Why CDC?

CMWS (Claims Management Workflow System) uses a **100% stored procedure** database access pattern. Every user action -- reserve a document, move it between queues, assign it to a caseworker -- ultimately results in DML statements executed by Oracle stored procedures. The Java application has no ORM, no inline SQL, and no application-level event bus. The database **is** the event log.

Change Data Capture (CDC) lets us intercept every `INSERT`, `UPDATE`, and `DELETE` at the database level, without modifying a single line of Java code. This is critical for CMWS because:

- The codebase is legacy (Struts 1.x, J2EE, deployed on WebSphere) and any code change requires a full EAR rebuild and deployment cycle
- The stored procedures already write comprehensive audit data (especially to `WORKFLOW_HISTORY`)
- 21 schemas share the same table structures, so one CDC configuration covers all workflows
- The system processes documents across ~20 business units -- CDC captures them all uniformly

### Two CDC Approaches for Oracle

| Approach | License | Latency | Complexity | Best For |
|----------|---------|---------|------------|----------|
| **Oracle GoldenGate** | Enterprise license ($$) | Sub-second | High (dedicated infrastructure) | Production with strict SLAs |
| **Oracle LogMiner** | Built into Oracle (free) | Seconds to minutes | Medium (polling-based) | Proof of concept, cost-sensitive |
| **Debezium (Oracle connector)** | Open source (free) | Seconds | Medium (Kafka required) | Kafka-native architectures |

### Target Architecture

```
Oracle DB (21 schemas)
    |
    | CDC (GoldenGate / LogMiner / Debezium)
    |
    v
Kafka Topics (one per table or per schema)
    |
    | NiFi / Kafka Connect
    |
    v
Elasticsearch (event index)
    |
    | API / Direct query
    |
    v
UiPath Process Mining
```

The goal is to transform raw database changes into a standard process mining event log with four required fields: **case_id**, **activity**, **timestamp**, and **resource** (who performed the action).

---

## 2. Tables to Monitor -- Tier 1 (Critical for Process Mining)

These tables directly map to the process mining event format and provide 80%+ of the data needed.

### WORKFLOW_HISTORY -- The Gold Mine

This is an **append-only audit log**. Every workflow action inserts a row here. The stored procedure `ufn_WF_AddWFHistory` is called from `WorkflowHelper.addWorkflowHistory()` every time a document is enqueued, moved, reserved, assigned, dequeued, or transferred.

**Known columns** (from `USP_WF_GETWFHISTORY` SELECT clause):

| Column | Type | Process Mining Role | Description |
|--------|------|-------------------|-------------|
| `document_id` | NUMBER | **Case ID** | The document/folder being tracked |
| `workflow_event_cd` | NUMBER | **Activity** (coded) | Event type code -- joined to `WORKFLOW_EVENT_TYPE` |
| `user_ions_id` | CHAR | **Resource** | The user who performed the action (IONS ID) |
| `event_data1` | VARCHAR2 | **Activity detail** | Queue name, status, or action description |
| `event_data2` | VARCHAR2 | **Additional context** | Secondary data (varies by event type) |
| `created_date` | DATE/TIMESTAMP | **Timestamp** | When the event occurred |

**Workflow Event Codes** (confirmed from stored procedure source and Java constants):

| Code | Description | Meaning for Process Mining |
|------|-------------|---------------------------|
| 1 | Moved | Document moved to a new queue (event_data1 = target queue name) |
| 2 | Dequeued | Document removed from workflow |
| 3 | Moved (variant) | Another move variant, used interchangeably with code 1 |
| 4 | Assigned | Document assigned to a specific user |
| 8 | Split | Document was split (e.g., TPA split) |
| 9 | Reserved | Document reserved by a user |
| 14 | Enqueued | Document initially entered the workflow |
| 1104 | COB Task Changed | COB work type task status changed |
| 1105 | COB Checklist Changed | COB checklist item completed/updated |
| 1106 | COB/Lockbox Email Notification | Email notification sent |
| 1402 | EPR Task Changed | EPR work type task status changed |
| 1403 | EPR Email Notification | EPR email notification sent |
| 1405 | EPR Task Status Changed | EPR task status transition |

**Event data1 values** (confirmed from stored procedure WHERE clauses):

- Queue movement targets: `LPM Pending`, `Pending`, `LPM Quality Review`, `Quality Review`, `LPM Paid-Up Archive`
- Completion actions: `Completed`, `LPM Completed`, `LRK Completed`
- Deletion actions: `Deleted`, `LPM Deleted`, `LRK Deleted`, `Shared Services Delete`
- Transfer actions: `Manually Transferred`, `LPM Manually Transferred`, `LRK Manually Transferred`

**CDC events to capture:** `INSERT` only (this is an append-only table -- no updates or deletes)

**Why this table alone gives you 80%:** Every reserve, assign, move, dequeue, and transfer is recorded with who did it, when, and what queue was involved. This directly maps to process mining's case_id + activity + timestamp + resource format.

### DOCUMENT_QUEUE -- Real-Time Queue State

This table holds the **current** position of every active document in the workflow. Unlike `WORKFLOW_HISTORY` (which is the log), this is the live state.

**Known columns** (from `USP_WF_ADDQUEUEITEM` INSERT and `USP_WF_UPDATEQUEUEITEM` SELECT):

| Column | Type | Process Mining Role | Description |
|--------|------|-------------------|-------------|
| `document_id` | NUMBER | **Case ID** | The document being tracked |
| `queue_id` | NUMBER | **State** | Current queue (joined to `QUEUE` for name) |
| `queue_type_cd` | NUMBER(7) | **State category** | Denormalized queue type |
| `reserved_by` | CHAR | **Resource** | User who reserved the item |
| `assigned_to` | CHAR | **Resource** | User assigned to work the item |
| `prior_user` | CHAR | Context | Previous user (before reassignment) |
| `prior_queue_id` | NUMBER | Context | Previous queue (before move) |
| `created_date` | DATE | Timestamp | When item entered the queue |
| `created_by` | CHAR | Resource | Who enqueued the item |
| `modified_date` | DATE | **Timestamp** | Last modification time |
| `modified_by` | CHAR | **Resource** | Who last modified the item |

**CDC events to capture:**

| DML | Process Mining Event |
|-----|---------------------|
| `INSERT` | Document enqueued (new item enters workflow) |
| `UPDATE` (queue_id changed) | Document moved to different queue |
| `UPDATE` (assigned_to changed) | Document reassigned |
| `UPDATE` (reserved_by changed) | Document reserved/unreserved |
| `DELETE` | Document dequeued (removed from workflow) |

**Complementary to WORKFLOW_HISTORY:** DOCUMENT_QUEUE captures the before/after state of each change (via CDC's old-value capture), while WORKFLOW_HISTORY provides the semantic description. Use both for a complete picture.

### DOCUMENT -- Document Lifecycle

**Known columns** (from multiple stored procedures):

| Column | Type | Process Mining Role | Description |
|--------|------|-------------------|-------------|
| `document_id` | NUMBER | **Case ID** | Primary key, the universal identifier |
| `document_type_cd` | NUMBER | Context | Document type (joined to `DOCUMENT_TYPE`) |
| `modified_ts` | VARCHAR2 | **Optimistic lock** | Timestamp used for concurrency control |
| `modified_date` | DATE/TIMESTAMP | **Timestamp** | Last modification |
| `modified_by` | CHAR | **Resource** | Who last modified |

**CDC events to capture:**

| DML | Process Mining Event |
|-----|---------------------|
| `INSERT` | New document created (start of case lifecycle) |
| `UPDATE` (modified_date changed) | Document touched (every queue update triggers this) |

**Note:** The `DOCUMENT` table is updated on every queue operation (`USP_WF_UPDATEQUEUEITEM` and `USP_WF_REMOVEQUEUEITEM` both UPDATE DOCUMENT to bump the timestamp). This means CDC on this table will generate high volumes of change events that largely duplicate WORKFLOW_HISTORY. Consider filtering to `INSERT` only unless you need document metadata changes.

---

## 3. Tables to Monitor -- Tier 2 (Rich Process Context)

These tables add human-readable context, subprocess milestones, and quality review data.

### CASE_DIARY -- User Notes and Context

**Known columns** (from `USP_CASEDIARY_ADDENTRY`):

| Column | Type | Process Mining Role |
|--------|------|-------------------|
| `document_id` | NUMBER | Case ID (links to folder document) |
| `diary_text` | VARCHAR2 | Activity detail (what the user noted) |
| `created_date` | DATE | Timestamp |
| `created_by` | CHAR | Resource |
| `modified_date` | DATE | Timestamp (for edits) |
| `modified_by` | CHAR | Resource (for edits) |

**CDC events:** `INSERT` = user added a diary entry. `UPDATE` = user edited a previous entry. Diary entries provide human context for decisions that pure queue movements cannot explain.

### Case Listing Tables -- Case Status Changes

Multiple domain-specific case listing tables exist. Each follows a similar pattern:

| Table | Workflow Domain | Key Procedure |
|-------|----------------|---------------|
| `LRK_CASE_LISTING` | Record Keeping Services | `USP_CASELIST_ADDLRKCASE` |
| `LCNV_CASE_LISTING` | Life Conversions | `USP_CASELIST_ADDLCNVCASE` |
| `GULGVUL_CASE_LISTING` | GUL/GVUL Management | `USP_CASELIST_ADDGULGVULCASE` |
| `NJEA_CASE_LISTING` | NJEA Billing | `USP_CASELIST_ADDNJEACASE` |
| `CASE_LISTING` | ESC / General | `USP_CASELIST_ADDESCCASE` |
| `BILLING_CASE_LISTING` | Lockbox | `USP_CASELIST_ADDCASE` |

**CDC events:** `INSERT` = new case created. `UPDATE` = case status change, admin assignment change, or metadata update. These tables carry case-level metadata (case name, control number, status, case admin) that enriches the document-level events from WORKFLOW_HISTORY.

### COB_WT_TASK / COB_WT_CHECKLIST_ITEM -- COB Subprocess Tracking

These tables track individual tasks and checklist items within Client On-Boarding work types.

**COB_WT_TASK** -- updated via `USP_COB_UPDATEWORKTYPETASK` and `USP_COB_UPDWORKTYPETASKDYNAMIC`:

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `UPDATE` (completion date set) | Task completed -- a subprocess milestone |
| `UPDATE` (owner changed) | Task reassigned to different user |

**COB_WT_CHECKLIST_ITEM** -- updated via `USP_COB_UPDATEWORKTYPECHECKLST` and `USP_COB_UPDWORKTYPECHKLDYNAMIC`:

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `UPDATE` (completed flag set) | Checklist item completed |

These events appear in WORKFLOW_HISTORY as event codes 1104 (task changed) and 1105 (checklist changed), but the actual table changes carry richer data (which task, what changed, who completed it).

### EPR_WT_TASK -- EPR Subprocess Tracking

Parallel to COB, but for Enrollment Campaign Management (EPR). Updated via `USP_EPR_UPDATEWORKTYPETASK` and `USP_EPR_UPDWORKTYPETASKDYNAMIC`.

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `UPDATE` (completion date set) | EPR task completed |
| `UPDATE` (owner changed) | EPR task reassigned |

### COB_MILESTONE -- Milestone Completions

Updated via `USP_COB_UPDATEWORKTYPEMILESTN`.

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `UPDATE` | Milestone status changed -- major phase completion |

### QR_SELECTED_DOCS -- Quality Review Selections

Managed by `USP_QR_ADDDOCUMENTITEM` (insert) and `USP_QR_UPDATEDOCUMENTSTATUS` (update).

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `INSERT` | Document selected for quality review |
| `UPDATE` (status_cd changed) | QR result recorded (pass/fail/FIO) |

### RECORD_HOLD_ORDER / RECORD_DESTRUCTION -- Records Management

Managed by `USP_RADMIN_ADDHOLDORDER`, `USP_RADMIN_UPDATEHOLDORDER`, and `USP_RADMIN_UPDATERD`.

| CDC Event | Process Mining Meaning |
|-----------|----------------------|
| `RECORD_HOLD_ORDER INSERT` | Legal hold applied to a set of documents |
| `RECORD_HOLD_ORDER UPDATE` | Hold order modified or released |
| `RECORD_DESTRUCTION INSERT/UPDATE` | Destruction scheduled, executed, or cancelled |

### ADMIN_HISTORY -- User Management Audit Trail

Written by every `USP_ADMIN_*` procedure that modifies users (add, update, FBU changes).

**CDC events:** `INSERT` only (append-only audit log). Tracks when users are added, deactivated, or moved between functional business units. Useful for organizational context in process mining (e.g., correlating process changes with staffing changes).

---

## 4. Tables to Monitor -- Tier 3 (Domain-Specific Events)

These tables carry domain-specific data for particular workflows. Monitor them only after Tier 1 and Tier 2 are stable.

| Table | Domain | CDC Event | Process Mining Value |
|-------|--------|-----------|---------------------|
| `PRU_LCMS_BENE` | LCMS Claims | INSERT/UPDATE | Beneficiary added/modified on a death claim |
| `PRU_DEATH_CLAIM_COVERAGE` | LCMS Claims | INSERT/UPDATE | Coverage amount updated on a claim |
| `LCMS_ERROR_LOG` | LCMS Claims | INSERT | Error occurred during claim processing |
| `COB_WORK_TYPE` | Client On-Boarding | UPDATE | Work type status, dates, or configuration changed |
| `EPR_WORK_TYPE` | Enrollment Campaigns | UPDATE | EPR work type status changed |
| `COB_WT_DISTRIBUTION_LIST` | Client On-Boarding | INSERT/DELETE | Email distribution list modified |
| `EPR_WT_DISTRIBUTION_LIST` | Enrollment Campaigns | INSERT/DELETE | EPR distribution list modified |
| `DOCUMENT_QTS` | QTS Integration | INSERT/UPDATE | QTS feed event linked to a document |
| `SET_PRIORITY` / `SET_PRIORITY_DETAILS` | All workflows | INSERT/UPDATE | Queue priority rules changed |
| `MU_SLA` | Medical Underwriting | INSERT/UPDATE | SLA created or modified |
| `TARGET_DATE` | All workflows | INSERT/UPDATE | Due date calculated or recalculated |
| `CASE_PROCESS_TRACKING` | LPM workflows | INSERT/UPDATE | Case activity requirement completed |
| `ACTIVITY_PROCESS_HISTORY` | IPR (Productivity) | INSERT | Individual processing rate event recorded |
| `HASTE_REQUEST_TRACKING` | All workflows | INSERT/UPDATE | Expedited request created or updated |
| `ELIGIBILITY_ANNOTATION` | OSGLI | INSERT/UPDATE | Eligibility annotation created or QR status changed |
| `RESTRICTED_CASE_USER_HISTORY` | All workflows | INSERT | Restricted case access grant/revoke audit |

---

## 5. Process Mining Event Mapping

The core requirement for process mining is a flat event log with four columns. Here is how each CDC source maps:

### Primary Mapping (Tier 1)

| CDC Source | Case ID | Activity Name | Timestamp | Resource |
|-----------|---------|--------------|-----------|----------|
| `WORKFLOW_HISTORY` INSERT | `document_id` | Decode `workflow_event_cd` via `WORKFLOW_EVENT_TYPE.workflow_event_desc` (e.g., "Moved", "Dequeued", "Assigned") + `event_data1` for detail | `created_date` | `user_ions_id` |
| `DOCUMENT_QUEUE` INSERT | `document_id` | "Enqueued to {queue_name}" (join `QUEUE` on `queue_id`) | `created_date` | `created_by` |
| `DOCUMENT_QUEUE` UPDATE (queue_id changed) | `document_id` | "Moved to {new_queue_name}" | `modified_date` | `modified_by` |
| `DOCUMENT_QUEUE` UPDATE (assigned_to changed) | `document_id` | "Assigned to {assigned_to}" | `modified_date` | `modified_by` |
| `DOCUMENT_QUEUE` UPDATE (reserved_by changed) | `document_id` | "Reserved by {reserved_by}" or "Unreserved" | `modified_date` | `modified_by` |
| `DOCUMENT_QUEUE` DELETE | `document_id` | "Dequeued" | capture time (CDC metadata) | last `modified_by` |
| `DOCUMENT` INSERT | `document_id` | "Document Created" | `modified_date` | `modified_by` |

### Secondary Mapping (Tier 2)

| CDC Source | Case ID | Activity Name | Timestamp | Resource |
|-----------|---------|--------------|-----------|----------|
| `CASE_DIARY` INSERT | `document_id` | "Diary Entry Added" | `created_date` | `created_by` |
| `COB_WT_TASK` UPDATE (completion_date set) | `work_type_id` -> join `COB_WORK_TYPE` -> `document_id` | "Task Completed: {task_name}" | `completion_date` or `modified_date` | `modified_by` |
| `COB_WT_CHECKLIST_ITEM` UPDATE (completed) | via work_type_id chain | "Checklist Item Completed: {item_name}" | `modified_date` | `modified_by` |
| `EPR_WT_TASK` UPDATE (completion_date set) | via work_type_id chain | "EPR Task Completed: {task_name}" | `completion_date` or `modified_date` | `modified_by` |
| `QR_SELECTED_DOCS` INSERT | `document_id` | "Selected for Quality Review" | `created_date` | system |
| `QR_SELECTED_DOCS` UPDATE (status_cd changed) | `document_id` | "QR {Pass/Fail/FIO}" | `modified_date` | reviewer |
| `RECORD_HOLD_ORDER` INSERT | SSN -> join to `document_id` via `RECORD_HO_DETAILS` | "Legal Hold Applied" | `created_date` | `created_by` |
| `RECORD_DESTRUCTION` UPDATE | `document_id` via `COMMON_METADATA` | "Destruction {Scheduled/Executed/Cancelled}" | `modified_date` | `modified_by` |
| `LRK_CASE_LISTING` INSERT | `document_id` or `control_number` | "Case Created" | `created_date` | `created_by` |
| `LRK_CASE_LISTING` UPDATE | `document_id` or `control_number` | "Case Status Changed" | `modified_date` | `modified_by` |

### Recommended Event Enrichment SQL

For the primary WORKFLOW_HISTORY extraction, use this query pattern to produce process mining-ready rows:

```sql
SELECT
    wh.document_id                          AS case_id,
    wet.workflow_event_desc
        || CASE WHEN wh.event_data1 IS NOT NULL
                THEN ': ' || wh.event_data1
                ELSE '' END                 AS activity,
    wh.created_date                         AS timestamp,
    wh.user_ions_id                         AS resource_id,
    NVL(wu.first_name, '') || ' '
        || NVL(wu.last_name, '')            AS resource_name,
    wh.event_data2                          AS additional_info
FROM WORKFLOW_HISTORY wh
    JOIN WORKFLOW_EVENT_TYPE wet
      ON wh.workflow_event_cd = wet.workflow_event_cd
    LEFT JOIN WORKFLOW_USER wu
      ON wh.user_ions_id = wu.ions_id
ORDER BY wh.document_id, wh.created_date;
```

This is adapted directly from `USP_WF_GETWFHISTORY` in the LRKPROD schema source. Run it against any of the 21 schemas to extract historical events without CDC.

---

## 6. CDC Implementation Options

### Option A: Oracle GoldenGate

**How it works:** GoldenGate reads Oracle redo logs in real-time via its Extract process, writes trail files, and a Replicat or Pump process delivers changes to downstream targets (Kafka, flat files, another database).

**Setup steps:**

1. Enable supplemental logging on the Oracle database:
   ```sql
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ```

2. Enable supplemental logging per table (Tier 1 tables):
   ```sql
   ALTER TABLE LRKPROD.WORKFLOW_HISTORY ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ALTER TABLE LRKPROD.DOCUMENT_QUEUE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ALTER TABLE LRKPROD.DOCUMENT ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ```

3. Configure GoldenGate Extract to capture changes from monitored schemas
4. Configure GoldenGate Pump to deliver to Kafka topics
5. Use Kafka Connect or NiFi to transform and load into Elasticsearch

**Pros:**
- Sub-second latency
- Minimal impact on source database performance
- Supports DDL replication if schema changes
- Enterprise support from Oracle

**Cons:**
- Requires Oracle GoldenGate license (significant cost)
- Complex infrastructure (Extract, Trail, Pump/Replicat processes)
- DBA involvement required for setup and monitoring
- Separate GoldenGate server infrastructure needed

**Best for:** Production deployment with strict latency SLAs and existing Oracle enterprise license agreements.

### Option B: Oracle LogMiner

**How it works:** LogMiner is a built-in Oracle utility that reads archived redo logs and presents DML changes as SQL statements. A polling process (NiFi, custom script, or Debezium) periodically queries LogMiner to extract recent changes.

**Setup steps:**

1. Ensure archivelog mode is enabled:
   ```sql
   SELECT LOG_MODE FROM V$DATABASE;
   -- Should return 'ARCHIVELOG'
   ```

2. Enable supplemental logging:
   ```sql
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
   ```

3. Grant required privileges to the CDC user:
   ```sql
   GRANT SELECT ON V_$LOGMNR_CONTENTS TO cdc_user;
   GRANT SELECT ON V_$LOG TO cdc_user;
   GRANT SELECT ON V_$ARCHIVED_LOG TO cdc_user;
   GRANT EXECUTE ON DBMS_LOGMNR TO cdc_user;
   GRANT EXECUTE ON DBMS_LOGMNR_D TO cdc_user;
   GRANT SELECT ANY TABLE TO cdc_user;
   ```

4. Use NiFi's `CaptureChangeMySQL` (Oracle variant) or custom polling:
   ```sql
   -- Start LogMiner session
   BEGIN
     DBMS_LOGMNR.START_LOGMNR(
       STARTTIME => SYSDATE - 1/24,  -- last hour
       ENDTIME   => SYSDATE,
       OPTIONS   => DBMS_LOGMNR.DICT_FROM_ONLINE_CATALOG
                   + DBMS_LOGMNR.COMMITTED_DATA_ONLY
     );
   END;
   /

   -- Query changes for Tier 1 tables
   SELECT SCN, TIMESTAMP, OPERATION, SEG_OWNER, TABLE_NAME,
          SQL_REDO, SQL_UNDO, ROW_ID
   FROM V$LOGMNR_CONTENTS
   WHERE SEG_OWNER = 'LRKPROD'
     AND TABLE_NAME IN ('WORKFLOW_HISTORY', 'DOCUMENT_QUEUE', 'DOCUMENT')
   ORDER BY SCN;
   ```

**Pros:**
- No additional license cost (built into Oracle)
- No additional infrastructure beyond a polling process
- Works with any Oracle edition
- DBA-friendly -- standard Oracle tooling

**Cons:**
- Higher latency (polling interval, typically 30 seconds to 5 minutes)
- More resource-intensive on the database during log reads
- Must manage SCN (System Change Number) bookmarks to avoid reprocessing
- Archived log retention must be sufficient for the polling interval

**Best for:** Development, proof of concept, and cost-sensitive environments.

### Option C: Debezium (Oracle Connector)

**How it works:** Debezium is an open-source CDC platform that runs as a Kafka Connect source connector. The Oracle connector uses LogMiner under the hood but provides a managed, fault-tolerant streaming experience with automatic offset tracking.

**Setup steps:**

1. Deploy Kafka + Kafka Connect (or use a managed service)
2. Enable supplemental logging (same as LogMiner)
3. Create a Debezium Oracle connector configuration:

   ```json
   {
     "name": "cmws-oracle-cdc",
     "config": {
       "connector.class": "io.debezium.connector.oracle.OracleConnector",
       "database.hostname": "cmwspd01",
       "database.port": "1521",
       "database.user": "cdc_user",
       "database.password": "${CDC_PASSWORD}",
       "database.dbname": "CMWSDB",
       "database.server.name": "cmws",
       "schema.include.list": "LRKPROD,COBPROD,EPRPROD",
       "table.include.list": "LRKPROD.WORKFLOW_HISTORY,LRKPROD.DOCUMENT_QUEUE,LRKPROD.DOCUMENT,COBPROD.WORKFLOW_HISTORY,COBPROD.DOCUMENT_QUEUE,EPRPROD.WORKFLOW_HISTORY,EPRPROD.DOCUMENT_QUEUE",
       "database.history.kafka.bootstrap.servers": "kafka:9092",
       "database.history.kafka.topic": "cmws-schema-changes",
       "snapshot.mode": "schema_only",
       "log.mining.strategy": "online_catalog"
     }
   }
   ```

4. Each table gets a Kafka topic: `cmws.LRKPROD.WORKFLOW_HISTORY`, etc.
5. Consume topics with NiFi, Kafka Streams, or Kafka Connect sink to Elasticsearch

**Debezium event format (example for WORKFLOW_HISTORY INSERT):**

```json
{
  "schema": { ... },
  "payload": {
    "before": null,
    "after": {
      "DOCUMENT_ID": 123456,
      "WORKFLOW_EVENT_CD": 1,
      "USER_IONS_ID": "X280182",
      "EVENT_DATA1": "Completed",
      "EVENT_DATA2": null,
      "CREATED_DATE": 1712505600000
    },
    "source": {
      "version": "2.5.0",
      "connector": "oracle",
      "name": "cmws",
      "ts_ms": 1712505600123,
      "schema": "LRKPROD",
      "table": "WORKFLOW_HISTORY"
    },
    "op": "c",
    "ts_ms": 1712505600456
  }
}
```

**Pros:**
- Open source, no license cost
- Automatic offset management (no lost or duplicate events)
- Kafka-native (integrates with existing streaming infrastructure)
- Supports snapshotting for initial historical load
- Well-documented Oracle connector

**Cons:**
- Requires Kafka infrastructure
- LogMiner-based (same latency characteristics as Option B)
- Oracle connector is less mature than MySQL/PostgreSQL connectors
- Memory-intensive for high-volume schemas

**Best for:** Organizations already running Kafka or planning a streaming architecture.

---

## 7. Schema Considerations

### 21 Schemas, Same Structure

CMWS runs 21 Oracle schemas (one per workflow). The core tables (`WORKFLOW_HISTORY`, `DOCUMENT_QUEUE`, `DOCUMENT`, `QUEUE`, `WORKFLOW_USER`, `WORKFLOW_EVENT_TYPE`) exist in **every** schema with identical structures. This is confirmed by the `ALTER SESSION SET CURRENT_SCHEMA` pattern in `OracleDataAccess.java`.

| Schema | Workflow | DB Server Pair (DEV) | Proc Count |
|--------|----------|---------------------|------------|
| `LRKPROD` | Record Keeping Services | CMWSPD01/CMWSPD03 | 128 |
| `MLLCMPROD` | COSC/MLBO Lockbox | CMWSPD01/CMWSPD03 | 144 |
| `OSGLIPROD` | OSGLI Administration | CMWSPD02/CMWSPD04 | 278 |
| `CONTRACTSPROD` | Contracts Archive | CMWSPD01/CMWSPD03 | 46 |
| `UWPROD` | Underwriting Archive | CMWSPD01/CMWSPD03 | 49 |
| `PROPOSALPROD` | Proposal Unit Archive | CMWSPD01/CMWSPD03 | 47 |
| `WALMARTPROD` | Walmart | CMWSPD01/CMWSPD03 | 116 |
| `LCNVPROD` | Life Conversions | CMWSPD01/CMWSPD03 | 317 |
| `GVULPROD` | GUL/GVUL Management | CMWSPD01/CMWSPD03 | 135 |
| `ESCPROD` | Enrollment Support Center | CMWSPD01/CMWSPD03 | 116 |
| `COBPROD` | Client On-Boarding | CMWSPD01/CMWSPD03 | 140 |
| `COB2PROD` | Small Market COB | CMWSPD01/CMWSPD03 | 129 |
| `VERIZONPROD` | Verizon | CMWSPD01/CMWSPD03 | 94 |
| `EPRPROD` | Enrollment Campaign Mgmt | CMWSPD01/CMWSPD03 | 146 |
| `OSGLIARCHIVE` | OSGLI Archive | CMWSPD02/CMWSPD04 | 274 |
| `MUCMWSPROD` | Medical Underwriting | CMWSPD02/CMWSPD04 | 81 |
| `LCMSGLCDPROD` | LCMS-GLCD | CMWSPD02/CMWSPD04 | 80 |
| `LCMSOSGLIPROD` | LCMS-OSGLI | CMWSPD02/CMWSPD04 | 65 |
| `LCMSPEBPROD` | LCMS-PEB | CMWSPD02/CMWSPD04 | 70 |
| `LCMSILIPROD` | LCMS-ILI | CMWSPD02/CMWSPD04 | 80 |
| `DCMSCMWSPROD` | DCMS | CMWSPD02/CMWSPD04 | 18 |

### Recommended Rollout Strategy

**Phase 1 -- Proof of Concept:** Monitor `LRKPROD` only (Record Keeping Services). This is the most representative workflow with 128 stored procedures and covers all core tables.

**Phase 2 -- High-Value Workflows:** Add `COBPROD` (Client On-Boarding) and `EPRPROD` (Enrollment Campaigns). These have the richest subprocess tracking (milestones, tasks, checklists) and the most complex process flows.

**Phase 3 -- LCMS Claims:** Add `LCMSGLCDPROD`, `LCMSOSGLIPROD`, `LCMSPEBPROD`, `LCMSILIPROD`. These carry domain-specific tables (`PRU_LCMS_BENE`, `PRU_DEATH_CLAIM_COVERAGE`) that add claims context.

**Phase 4 -- All Remaining Schemas:** Add remaining 13 schemas. By this point, the CDC pipeline is proven and scaling is mechanical.

### Cross-Schema Correlation

Documents can move between workflows (e.g., a document might be transferred from RKS to Lockbox). The correlation key is `document_id`, which is globally unique across schemas. When building the process mining event log, include a `schema_name` column to identify which workflow context generated each event:

```sql
-- In the CDC transformation layer, add schema context
SELECT
    'LRKPROD' AS schema_name,
    wh.document_id AS case_id,
    wet.workflow_event_desc || ': ' || wh.event_data1 AS activity,
    wh.created_date AS timestamp,
    wh.user_ions_id AS resource_id
FROM LRKPROD.WORKFLOW_HISTORY wh
    JOIN LRKPROD.WORKFLOW_EVENT_TYPE wet
      ON wh.workflow_event_cd = wet.workflow_event_cd
UNION ALL
SELECT
    'COBPROD' AS schema_name,
    wh.document_id AS case_id,
    ...
FROM COBPROD.WORKFLOW_HISTORY wh
    ...
```

---

## 8. Volume Estimates

### Estimation Methodology

CMWS processes documents for approximately 20 business units across Group Insurance. Volume estimates are based on the density of stored procedure calls and the typical insurance workflow patterns.

### Estimated Daily Event Volumes

| Source Table | Events/Day (est.) | Rationale |
|-------------|-------------------|-----------|
| `WORKFLOW_HISTORY` (all schemas) | 5,000 -- 20,000 | Every enqueue, move, assign, reserve, dequeue generates one row. A document typically generates 5-15 history rows over its lifecycle. |
| `DOCUMENT_QUEUE` changes | 3,000 -- 15,000 | Roughly 1:1 with WORKFLOW_HISTORY minus enqueue events. Each movement is an UPDATE. |
| `DOCUMENT` changes | 3,000 -- 15,000 | Mirror of DOCUMENT_QUEUE changes (timestamp bump on every queue operation). |
| `CASE_DIARY` inserts | 500 -- 2,000 | Caseworkers add diary entries during processing. Not every document gets diary notes. |
| `COB/EPR task/checklist` changes | 200 -- 1,000 | Only COB and EPR workflows have subprocess tracking. |
| `QR_SELECTED_DOCS` changes | 100 -- 500 | Quality review is sampled -- not every document is reviewed. |
| **Total (all tables, all schemas)** | **12,000 -- 55,000** | |

### Storage Sizing for Elasticsearch

| Metric | Estimate |
|--------|----------|
| Average event size (JSON) | ~500 bytes |
| Daily ingest (high estimate) | 55,000 events x 500 bytes = ~27 MB/day |
| Monthly ingest | ~825 MB/month |
| 1-year retention | ~10 GB |
| With 1 replica | ~20 GB |
| Recommended initial allocation | 50 GB (headroom for enrichment, metadata, index overhead) |

These are modest volumes. Elasticsearch on a single node can easily handle this workload. The bottleneck will be the CDC extraction and transformation, not the storage.

### Peak Load Considerations

- **Month-end processing:** Insurance workflows spike at month-end (billing cycles, enrollment deadlines). Expect 2-3x daily averages.
- **Open enrollment periods:** EPR (Enrollment Campaign Management) can see 5-10x normal volumes during annual enrollment.
- **Batch processing:** GIAL batch document loads (via SOAP) can inject hundreds of documents in minutes, generating bursts of WORKFLOW_HISTORY inserts.

---

## 9. Data Quality Considerations

### Timestamp Precision

CMWS uses a mix of Oracle `DATE` and `TIMESTAMP` types:

- `WORKFLOW_HISTORY.created_date` -- likely `DATE` (precision to seconds) based on Sybase migration heritage
- `DOCUMENT.modified_date` -- uses `SYSTIMESTAMP` in `USP_WF_UPDATEQUEUEITEM` (fractional seconds)
- `DOCUMENT.modified_ts` -- `VARCHAR2` used as a string-formatted timestamp for optimistic locking (not a native date type)
- `DOCUMENT_QUEUE.created_date` / `modified_date` -- uses `SYSDATE` (precision to seconds)
- `CASE_DIARY.created_date` -- uses `SYSDATE` (precision to seconds)

**Recommendation:** Normalize all timestamps to millisecond precision in the CDC transformation layer. For `DATE` columns (second precision), append `.000` for consistency. Store all timestamps in UTC in Elasticsearch, converting from the database server's timezone.

### Timezone Handling

- All Oracle timestamps are in the **database server's local timezone** (expected to be US Eastern Time based on Prudential's Newark, NJ headquarters)
- There are no `TIMESTAMP WITH TIME ZONE` columns in the known schema
- **Recommendation:** Confirm the database timezone with DBAs (`SELECT DBTIMEZONE FROM DUAL`), then apply a consistent UTC offset during CDC transformation. Be aware of daylight saving time transitions (ET shifts between UTC-5 and UTC-4).

### Null Handling

Several columns can be null in valid business scenarios:

| Column | When Null | Impact |
|--------|-----------|--------|
| `DOCUMENT_QUEUE.reserved_by` | Item is not reserved (available for pickup) | Activity becomes "Unreserved" or skip event |
| `DOCUMENT_QUEUE.assigned_to` | Item is not assigned to a specific user | Resource field is empty in process mining |
| `DOCUMENT_QUEUE.prior_user` | First time in queue (no prior user) | No before-state for comparison |
| `DOCUMENT_QUEUE.prior_queue_id` | First queue assignment | No source queue for "moved from" |
| `WORKFLOW_HISTORY.event_data1` | Some events have no detail data | Activity name is just the event type desc |
| `WORKFLOW_HISTORY.event_data2` | Most events only use data1 | Skip field |

**Recommendation:** In the CDC transformation, always use `NVL()` or `COALESCE()` to provide default values. Process mining tools typically require non-null case_id and timestamp at minimum.

### Historical Backfill

**WORKFLOW_HISTORY is already an event log.** You do not need CDC to extract historical data. Run the enrichment query from Section 5 directly against each schema to produce a historical event log:

```sql
-- Historical backfill for LRKPROD (adapt for each schema)
SELECT
    'LRKPROD'                               AS schema_name,
    wh.document_id                          AS case_id,
    wet.workflow_event_desc
        || CASE WHEN wh.event_data1 IS NOT NULL
                THEN ': ' || wh.event_data1
                ELSE '' END                 AS activity,
    wh.created_date                         AS event_timestamp,
    wh.user_ions_id                         AS resource_id,
    NVL(wu.first_name, '') || ' '
        || NVL(wu.last_name, '')            AS resource_name,
    wh.event_data2                          AS additional_info
FROM LRKPROD.WORKFLOW_HISTORY wh
    JOIN LRKPROD.WORKFLOW_EVENT_TYPE wet
      ON wh.workflow_event_cd = wet.workflow_event_cd
    LEFT JOIN LRKPROD.WORKFLOW_USER wu
      ON wh.user_ions_id = wu.ions_id
WHERE wh.created_date >= DATE '2024-01-01'   -- adjust date range
ORDER BY wh.document_id, wh.created_date;
```

Run this for each of the 21 schemas and union the results. This gives you a complete historical event log for process mining **today**, while CDC captures ongoing events going forward.

### Deduplication Between WORKFLOW_HISTORY and DOCUMENT_QUEUE CDC

If you monitor both tables, you will get duplicate events for the same action (e.g., a queue move generates both a WORKFLOW_HISTORY INSERT and a DOCUMENT_QUEUE UPDATE). Strategy options:

1. **Use WORKFLOW_HISTORY only** (recommended for simplicity) -- this table is purpose-built as an audit log and carries the semantic event description. It is sufficient for process mining.
2. **Use DOCUMENT_QUEUE CDC only** -- provides before/after state but requires you to infer the activity name from the field changes.
3. **Use both with deduplication** -- join on `document_id` + approximate timestamp match (within 1 second). Use WORKFLOW_HISTORY for the activity name and DOCUMENT_QUEUE for the before/after state enrichment.

**Recommendation for Phase 1:** Start with WORKFLOW_HISTORY only. It is the simplest, most complete, and already in event-log format. Add DOCUMENT_QUEUE CDC in Phase 2 only if you need real-time state tracking or before/after change details.

---

## Appendix: Quick Reference

### Minimum Viable CDC Configuration

To get process mining running with the least effort:

1. **One table:** `WORKFLOW_HISTORY` (INSERT only)
2. **One schema:** `LRKPROD` (proof of concept)
3. **One query:** The enrichment SQL from Section 5
4. **One output:** Flat CSV or Elasticsearch index with columns: `schema_name`, `case_id`, `activity`, `event_timestamp`, `resource_id`, `resource_name`

This can be implemented as a scheduled SQL query (no CDC infrastructure required) for the proof of concept, then upgraded to real-time CDC once the process mining model is validated.

### Related Documentation

- [10 - Database Layer](./10-database-layer.md) -- Stored procedure catalog, table inventory, connection architecture
- [03 - Workflow Engine and GIAL](./03-workflow-and-gial.md) -- Workflow engine deep-dive, queue management, GIAL batch ingestion
- [00 - Functional Overview](./00-functional-overview.md) -- Business context, workflow domains, user roles
