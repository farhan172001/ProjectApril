# CMWS System Diagrams

Mermaid diagram collection for the CMWS legacy codebase. Each diagram is self-contained and renderable in any Mermaid-compatible viewer (GitHub, VS Code plugin, Mermaid Live Editor).

---

## a. C4 System Context Diagram

Shows CMWS and all external systems, actors, and integration points at the highest level.

```mermaid
C4Context
    title CMWS System Context Diagram

    Person(caseWorker, "Case Worker", "GI business unit staff processing claims, conversions, enrollments")
    Person(admin, "Administrator", "Manages users, queues, records retention")
    Person(scanner, "Scan Operator", "Scans and indexes physical documents")

    System(cmws, "CMWS", "Claims Management Workflow System - Struts 1.x monolith on WebSphere. 20+ workflow instances across 14 business domains.")

    System_Ext(edm, "EDM", "Enterprise Document Mgmt - REST API middleware (replaced Helix)")
    System_Ext(filenet, "Filenet", "Document repository - stores scanned images and generated letters (replaced Documentum)")
    System_Ext(gial, "GIAL", "Batch ingestion pipeline - accepts XML batches from scanners, transforms, then calls CMWSWeb via HTTP POST + SOAP")
    System_Ext(tam, "Tivoli Access Manager", "Enterprise SSO and role-based access via WEBSEAL reverse proxy")
    System_Ext(oracleDb, "Oracle Database", "Workflow state, metadata, queue items, case management, reporting data")
    System_Ext(cmwsReports, "CMWSReports", "347 Crystal Reports on separate BI server")
    System_Ext(cmwsEar, "CMWSEAR", "Enterprise Archive Retrieval - read-only document access")
    System_Ext(imageViewer, "ImageViewer", "Java Swing applet (253 files) that calls CMWSWeb facades via SOAP")
    System_Ext(externalSystems, "External Systems", "MediConnect, Alliance Vendor, LINX - consume SOAP facades")

    Rel(caseWorker, cmws, "Uses", "HTTPS via WEBSEAL")
    Rel(admin, cmws, "Administers", "HTTPS via WEBSEAL")
    Rel(scanner, gial, "Sends XML batches", "SOAP/MTOM")
    Rel(gial, cmws, "Calls PutDocument, MetadataService, WorkflowService", "HTTP POST + SOAP")
    Rel(cmws, edm, "Store/retrieve docs", "REST API via legacy wrappers")
    Rel(edm, filenet, "Persists content", "Internal")
    Rel(cmws, oracleDb, "Read/write", "JDBC")
    Rel(tam, cmws, "Authenticates", "WEBSEAL headers")
    Rel(cmws, cmwsReports, "Launches reports", "HTTP redirect")
    Rel(imageViewer, cmws, "Calls facades", "SOAP")
    Rel(externalSystems, cmws, "Invoke services", "SOAP")
```

---

## b. C4 Container Diagram

Internal components of the CMWS monolith showing the layered architecture within the EAR deployment containing two WARs.

```mermaid
C4Container
    title CMWS Container Diagram (cmws.ear on WebSphere)

    Person(user, "User", "Case worker or admin")

    Container_Boundary(websphere, "WebSphere Application Server") {
        Container_Boundary(ear, "cmws.ear") {
            Container_Boundary(cmwsWar, "CMWS.war") {
                Container(webFilter, "WEBSEALFilter", "Servlet Filter", "Extracts TAM user identity from WEBSEAL reverse proxy headers")
                Container(strutsServlet, "ActionServlet", "Struts 1.1", "Routes *.go requests to Action classes via struts-config.xml")
                Container(actions, "Action Classes", "Struts Actions", "40+ actions: QueueViewAction, InboxViewAction, SearchAction, ApproveAction, MoveAction, etc.")
                Container(jsps, "JSP Views", "JSP 2.0", "cmwsmain.jsp, queue views, search forms, case listings, admin pages")
                Container(facades, "SOAP Facades", "JAX-RPC ServiceLifecycle", "WorkflowServiceFacade, MetadataServiceFacade, DocumentStorageFacade, domain facades (LCMS, OSGLI, LPM, etc.)")
                Container(domainFacades, "Domain Facades", "Java", "LCMSFacade, OSGLIFacade, LCNVFacade, COBFacade, ESCFacade, etc. - extend FacadeBase")
                Container(processLayer, "Process Layer", "Java", "LCMSProcess, IProcessMetadata, IProcessWorkflow, IProcessStorage, IProcessCaseManagement, factory classes")
                Container(queueEngine, "Queue Engine", "Java", "QueueManager, WorkflowQueue, WorkItem, WorkItemCache, UserInbox, TransferHelper, AutoAssignHelper")
                Container(configUtil, "Configuration", "Spring XML", "ConfigurationUtility loads workflows.xml, engines.xml, queueviews.xml at startup")
                Container(dataAccess, "Data Access", "Java/JDBC", "OracleDataAccess - stored procedure calls via JDBC to Oracle")
                Container(imageViewerJars, "ImageViewer JARs", "Java Swing Applet", "Packed JARs inside CMWS.war - 253-file Java Swing applet served to browser")
            }
            Container_Boundary(gialWar, "gial.war") {
                Container(gialSvcs, "GIAL Services", "JAX-WS", "Batch ingestion pipeline - ImageLoaderService, BatchControl, PDF-to-TIFF conversion")
            }
        }
    }

    ContainerDb(oracleDb, "Oracle DB", "Oracle 12c+", "WORKFLOW_INSTANCE, queue items, metadata, case management, reporting")
    Container_Ext(edmFilenet, "EDM / Filenet", "REST API", "Enterprise Document Management and content repository")
    Container_Ext(biServer, "BI Server", "Crystal Reports", "CMWSReports - 347 Crystal Reports on separate server")

    Rel(user, webFilter, "HTTPS")
    Rel(webFilter, strutsServlet, "Forwards")
    Rel(strutsServlet, actions, "Dispatches")
    Rel(actions, jsps, "Forwards to")
    Rel(actions, facades, "Calls directly")
    Rel(facades, domainFacades, "Delegates to")
    Rel(domainFacades, processLayer, "Uses")
    Rel(processLayer, dataAccess, "Queries")
    Rel(dataAccess, oracleDb, "JDBC")
    Rel(gialSvcs, facades, "HTTP POST + SOAP", "GIAL calls CMWSWeb")
    Rel(gialSvcs, edmFilenet, "REST API")
    Rel(actions, queueEngine, "Manages work items")
    Rel(queueEngine, dataAccess, "Stored procs")
    Rel(configUtil, oracleDb, "Reads config")
```

---

## c. Component Diagram - Domain Module Relationships

Shows how the 14+ business domains relate to the shared workflow framework and to each other.

```mermaid
graph TB
    subgraph "Shared Workflow Framework"
        FacadeBase["FacadeBase<br/><i>ServiceLifecycle</i>"]
        WFS["WorkflowServiceFacade"]
        MSF["MetadataServiceFacade"]
        DSF["DocumentStorageFacade"]
        CMF["CaseManagementFacade"]
        LTR["LetterServiceFacade"]
        QRF["QualityReviewFacade"]
        QTFS["QTSFacade"]
        WMF["WorkflowManagementFacade"]
        RPT["WorkflowReportingFacade"]
        NOT["NotificationEventFacade"]
    end

    subgraph "Domain Modules"
        LRK["lrk<br/>Record Keeping<br/>WF ID: 1"]
        LOCK["lockbox<br/>COSC/MLBO Lockbox<br/>WF ID: 2"]
        OSGLI["osgli<br/>OSGLI Admin<br/>WF ID: 3"]
        CON["contracts<br/>Contracts Archive<br/>WF ID: 4"]
        UW["uw<br/>Underwriting Archive<br/>WF ID: 5"]
        PROP["proposals<br/>Proposal Unit<br/>WF ID: 6"]
        WAL["walmart<br/>Wal Mart<br/>WF ID: 7"]
        LCNV["lcnv<br/>Life Conversions<br/>WF ID: 8"]
        GUL["gulgvul<br/>GUL/GVUL Mgmt<br/>WF ID: 9"]
        ESC["esc<br/>Enrollment Support<br/>WF ID: 10"]
        COB["cob<br/>Client On-Boarding<br/>WF ID: 11"]
        SMCOB["smcob<br/>SM Client On-Boarding<br/>WF ID: 12"]
        VER["verizon<br/>Verizon<br/>WF ID: 13"]
        EPR["epr<br/>Enrollment Campaign<br/>WF ID: 14"]
        MU["mu<br/>Medical Underwriting<br/>WF ID: 101"]
        LCMS["lcms<br/>LCMS-GLCD<br/>WF ID: 102"]
    end

    subgraph "Queue System"
        QM["QueueManager"]
        WQ["WorkflowQueue"]
        WI["WorkItem"]
        WIC["WorkItemCache"]
        UIB["UserInbox"]
        TH["TransferHelper"]
        AAH["AutoAssignHelper"]
    end

    subgraph "Process Interfaces"
        IPW["IProcessWorkflow"]
        IPM["IProcessMetadata"]
        IPS["IProcessStorage"]
        IPC["IProcessCaseManagement"]
        IPL["IProcessLetter"]
        IPR["IProcessReports"]
        IPQR["IProcessQualityReview"]
    end

    LRK --> FacadeBase
    LOCK --> FacadeBase
    OSGLI --> FacadeBase
    LCNV --> FacadeBase
    LCMS --> FacadeBase
    WAL --> FacadeBase
    ESC --> FacadeBase
    COB --> FacadeBase
    EPR --> FacadeBase
    MU --> FacadeBase

    FacadeBase --> WFS
    FacadeBase --> MSF
    FacadeBase --> DSF

    WFS --> IPW
    MSF --> IPM
    DSF --> IPS
    CMF --> IPC

    QM --> WQ
    WQ --> WI
    WIC --> WI
    TH --> WFS
    TH --> MSF
    TH --> DSF
    AAH --> WIC

    OSGLI -.->|"cross-WF transfer"| GUL
    LCNV -.->|"cross-WF transfer"| LRK

    style FacadeBase fill:#4a90d9,color:#fff
    style QM fill:#e67e22,color:#fff
    style TH fill:#e74c3c,color:#fff
```

---

## d. Sequence: Claim Creation (LCMS Domain)

Traces how a new claim is created in the LCMS domain, from user action through the process layer to Oracle.

```mermaid
sequenceDiagram
    autonumber
    actor User as Case Worker
    participant JSP as cmwsmain.jsp
    participant Action as SearchAndAssignClaimAction
    participant Facade as LCMSFacade
    participant CU as ConfigurationUtility
    participant WI as WorkflowInstance
    participant LP as LCMSProcess
    participant MF as MetadataProcessFactory
    participant MP as IProcessMetadata
    participant WPF as WorkflowProcessFactory
    participant WP as IProcessWorkflow
    participant DAL as OracleDataAccess
    participant DB as Oracle DB

    User->>JSP: Submit create claim form
    JSP->>Action: POST /workflow/searchAndAssignClaim.go
    Action->>Facade: createClaim(context, documentId)
    Facade->>Facade: setContextUser(context) [from WEBSEAL ThreadLocal]
    Facade->>CU: getWorkflow(context.getWorkflowId())
    CU-->>Facade: WorkflowInstance (WF ID=102)
    Facade->>CU: getServiceEngine(workflow.getMetadataEngineId())
    CU-->>Facade: ServiceEngine (LCMS_META)
    Facade->>MF: newMetadataProcess(serviceEngine)
    MF-->>Facade: IProcessMetadata impl
    Facade->>LP: new LCMSProcess(metaEngine)
    Facade->>LP: createClaim(context, metaProcess, documentId)
    LP->>DAL: executeQuery("call usp_LCMS_CreateClaim(...)")
    DAL->>DB: JDBC stored procedure call
    DB-->>DAL: ResultSet (claim_id, control_number)
    DAL-->>LP: ResultDataRow[]
    LP->>MP: updateItemMetadata(context, docId, timestamp, metadata)
    MP->>DAL: executeQuery("call usp_META_UpdateMetadata(...)")
    DAL->>DB: Update metadata
    DB-->>DAL: OK
    LP-->>Facade: ResultDataRow[] (new claim info)
    Facade-->>Action: ResultDataRow[]
    Action->>Action: Set request attributes
    Action-->>JSP: forward to success view
    JSP-->>User: Display claim confirmation
```

---

## d2. Sequence: GIAL Batch Ingestion

How scanned documents flow from scanners through the GIAL batch ingestion pipeline into CMWSWeb. GIAL is the caller -- it accepts XML batches from scanners, transforms content, then calls CMWSWeb services.

```mermaid
sequenceDiagram
    autonumber
    participant Scanner as Scanner System
    participant ILS as GIAL ImageLoaderService
    participant BCDb as BatchControl DB
    participant GIAL as GIAL Internal (PDF-to-TIFF)
    participant PutDoc as CMWSWeb PutDocument (HTTP)
    participant MSF as CMWSWeb MetadataServiceFacade
    participant WSF as CMWSWeb WorkflowServiceFacade

    Scanner->>ILS: Submit XML batch (scanned images + metadata)
    ILS->>BCDb: Validate batch (check duplicates, format)
    BCDb-->>ILS: Batch validation result

    alt PDF content needs conversion
        ILS->>GIAL: Convert PDF to TIFF
        GIAL-->>ILS: TIFF content
    end

    ILS->>PutDoc: HTTP POST document content (MTOM)
    PutDoc-->>ILS: documentId + timestamp

    ILS->>MSF: SOAP: updateItemMetadata(context, docId, metadata)
    Note over ILS,MSF: SSN, batch_number, control_number, scan_uid, etc.
    MSF-->>ILS: OK

    ILS->>WSF: SOAP: enqueueItem(context, docId, queueId)
    Note over ILS,WSF: Enqueues document into initial workflow queue
    WSF-->>ILS: OK

    ILS->>BCDb: Update batch status (COMPLETED/FAILED)
    BCDb-->>ILS: OK

    ILS-->>Scanner: ProcessStatus response (success/failure per item)
```

---

## d3. Sequence: ImageViewer Communication

How the ImageViewer Java Swing applet (253 files, packed as JARs inside CMWS.war) communicates with CMWSWeb facades via SOAP and bridges back to the browser via JavaScript.

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Browser
    participant IV as ImageViewerApp (JApplet)
    participant WFF as CMWSWeb WorkflowFacade
    participant CMF as CMWSWeb CaseManagementFacade
    participant DF as CMWSWeb Domain Facades
    participant JSBridge as Browser JavaScript Bridge

    Browser->>IV: Load applet (packed JARs from CMWS.war)
    IV->>JSBridge: Initialize session bridge (cookie/token handoff)
    JSBridge-->>IV: Session context

    IV->>WFF: SOAP: getWorkflowMetadata(workflowId, docId)
    WFF-->>IV: Workflow metadata + permissions

    IV->>CMF: SOAP: getCaseData(controlNumber)
    CMF-->>IV: Case folder, documents, history

    IV->>DF: SOAP: getDomainSpecificData(context)
    Note over IV,DF: Domain-specific facade varies by workflow type
    DF-->>IV: Domain data (claims, policies, etc.)

    IV->>IV: Render document with metadata overlay

    IV->>JSBridge: Session keepalive / navigation events
    JSBridge->>Browser: Update browser state (URL hash, hidden fields)
```

---

## e. Sequence: Document Retrieval

How a document image is fetched from Filenet (via EDM) through the legacy wrapper chain and streamed to the browser. Note: legacy class names (DocumentumRepositoryProcess, FilennetDataAccess) are kept as wrappers even though the underlying storage migrated from Documentum to EDM/Filenet.

```mermaid
sequenceDiagram
    autonumber
    actor User as Case Worker
    participant Browser as Browser/ImageViewer
    participant Servlet as GetDocument Servlet
    participant DSF as DocumentStorageFacade
    participant CSP as CMWSStorageProcess
    participant DRP as DocumentumRepositoryProcess
    participant FDA as FilennetDataAccess
    participant ETS as EdmTransformationService
    participant ESI as EdmServiceImpl
    participant EDMApi as EDM REST API

    User->>Browser: Click document link
    Browser->>Servlet: GET /Workflow/GetDocument?docId=12345&wfId=102
    Servlet->>DSF: getLatestVersion(context, documentId)
    DSF->>CSP: getLatestVersion(context, documentId)
    CSP->>DRP: getDocumentAsFile(context, objectId)
    Note over DRP: Legacy class name kept as wrapper
    DRP->>FDA: retrieveDocument(objectId)
    Note over FDA: Legacy class name (note double-n); now routes to EDM
    FDA->>ETS: transformAndRetrieve(objectId)
    ETS->>ESI: getDocument(objectId)
    ESI->>EDMApi: GET /documents/{objectId}/content
    EDMApi-->>ESI: File content + MIME type
    ESI-->>ETS: DocumentContent
    ETS-->>FDA: TransformedDocument
    FDA-->>DRP: DocumentWrapper (content, mimeType, version)
    DRP-->>CSP: DocumentWrapper
    CSP-->>DSF: DocumentWrapper
    DSF-->>Servlet: DocumentWrapper
    Servlet->>Servlet: Set Content-Type, Content-Disposition headers
    Servlet-->>Browser: Stream binary content (TIFF/PDF)
    Browser-->>User: Render document image
```

---

## f. Sequence: Queue Transfer (Cross-Workflow)

How a work item is transferred between queues across different workflow instances.

```mermaid
sequenceDiagram
    autonumber
    actor User as Case Worker
    participant Action as MoveAction
    participant TH as TransferHelper
    participant WH as WorkflowHelper
    participant DSF as DocumentStorageFacade
    participant MSF as MetadataServiceFacade
    participant WFS as WorkflowServiceFacade
    participant DAL as OracleDataAccess
    participant DB as Oracle DB

    User->>Action: Select item, choose target queue/workflow
    Action->>TH: transferCrossWorkflow(item, targetWfId, targetQueueId, targetDocType, metaDataMap)

    Note over TH: Step 1 - Save current ThreadLocal context
    TH->>WH: saveThreadLocals() [DAL, ServiceEngine, WorkflowId, MetadataProcess, ServiceContext]

    Note over TH: Step 2 - Get document content from source workflow
    TH->>DSF: getLatestVersionInternal(sourceContext, documentId)
    DSF-->>TH: DocumentWrapper (binary content)

    Note over TH: Step 3 - Create document in target workflow
    TH->>TH: Create targetContext (workflowId=target, user=TRANSFR)
    TH->>DSF: createDocumentInternal(targetContext, null, targetDocType, wrapper)
    DSF-->>TH: new document ID + timestamp

    Note over TH: Step 4 - Copy metadata to target
    TH->>MSF: updateItemMetadata(targetContext, newDocId, tsHolder, metadata[])
    Note over TH: Copies: scan_uid, batch_number, control_number, ssn, notes, etc.
    Note over TH: Batch number gets 2-digit workflow ID prefix on first transfer

    Note over TH: Step 5 - Enqueue in target workflow
    TH->>WFS: enqueueTransferItem(targetContext, newDocId, targetQueueId, null, null, additionalInfo)

    Note over TH: Step 6 - Restore original ThreadLocal context
    TH->>WH: restoreThreadLocals()

    Note over TH: Step 7 - Remove from source workflow
    TH->>DAL: executeCommand("call usp_WF_RemoveQueueItem(docId, userId, timestamp)")
    DAL->>DB: Execute stored proc
    TH->>TH: Expire queue cache (setLastLoaded(null))
    TH->>TH: Expire user inbox cache if assigned

    Note over TH: Step 8 - Record history
    TH->>WH: addWorkflowHistory(docId, DEQUEUE, userId, queueName)
    TH->>WH: addWorkflowHistory(docId, TRANSFERRED, userId, targetQueueName)

    Note over TH: Step 9 - Records management (OSGLI WF=3, GULGVUL WF=9)
    alt workflowId == 9
        TH->>TH: GulGvulRecordsAdminHelper.processForRecordsManagement(item)
    else workflowId == 3
        TH->>TH: OSGLIRecordsAdminHelper.processForRecordsManagement(item)
    end

    TH-->>Action: Transfer complete
    Action-->>User: Redirect to queue/inbox view
```

---

## g. Sequence: SOAP Web Service Call

How an external system invokes a CMWS facade via SOAP.

```mermaid
sequenceDiagram
    autonumber
    participant Ext as External System (MediConnect/GIAL)
    participant WS as WebSphere WebServicesServlet
    participant CF as CharsetFilter
    participant WF as WEBSEALFilter
    participant FB as FacadeBase.init()
    participant Facade as LCMSFacade
    participant CU as ConfigurationUtility
    participant Process as LCMSProcess
    participant DAL as OracleDataAccess
    participant DB as Oracle DB

    Ext->>WS: SOAP Request to /services/LCMSFacade
    Note over Ext,WS: SOAP envelope with ServiceContext (workflowId, userName)
    WS->>CF: CharsetFilter.doFilter()
    CF->>WF: WEBSEALFilter.doFilter()
    Note over WF: Extracts iv-user header, sets ThreadLocal
    WF->>WS: Continue filter chain
    WS->>FB: FacadeBase.init(ServletEndpointContext)
    Note over FB: Stores ServletContext, records start time
    WS->>Facade: assignOrCreateClaim(context, documentId, timestamp)
    Facade->>Facade: setContextUser(context)
    Note over Facade: Gets WEBSEAL user from ThreadLocal or dev_user fallback
    Facade->>CU: getWorkflow(context.getWorkflowId())
    CU-->>Facade: WorkflowInstance
    Facade->>Process: assignOrCreateClaim(context, metaProcess, wfProcess, docId, ts)
    Process->>DAL: executeQuery("call usp_LCMS_...()")
    DAL->>DB: Stored procedure
    DB-->>DAL: Results
    DAL-->>Process: ResultDataRow[]
    Process-->>Facade: void
    Facade-->>WS: Return
    WS->>FB: FacadeBase.destroy()
    Note over FB: Logs elapsed time
    WS-->>Ext: SOAP Response
```

---

## h. State Diagram: Workflow Lifecycle

States a claim/document goes through from initial scan to final completion or archival.

```mermaid
stateDiagram-v2
    [*] --> Scanned: Document scanned & indexed

    Scanned --> Ingested: GIAL batch ingestion creates document<br/>(usp_STOR_CreateDocument)
    Ingested --> Queued: Enqueued to initial queue<br/>(usp_WF_AddQueueItem)

    Queued --> Reserved: User clicks Get Next<br/>or manual reserve
    Queued --> AutoAssigned: AutoAssignHelper<br/>matches to existing case

    Reserved --> InProgress: User opens work item
    AutoAssigned --> InProgress: Foldered into case

    InProgress --> PendingApproval: Submit for approval
    InProgress --> Queued: Unassign / return to queue
    InProgress --> Transferred: Cross-workflow transfer<br/>(TransferHelper)
    InProgress --> Moved: Move to different queue<br/>within same workflow

    Moved --> Queued: Re-queued in target

    PendingApproval --> Approved: Supervisor approves<br/>(ApproveAction)
    PendingApproval --> Denied: Supervisor denies<br/>(DenyAction)
    PendingApproval --> Withdrawn: User withdraws<br/>(WithdrawAction)

    Denied --> InProgress: Rework required
    Withdrawn --> InProgress: Continue processing

    Approved --> Completed: Processing finished

    Transferred --> [*]: Item lives in target workflow now

    Completed --> RecordsRetention: Records management<br/>sets destruction date
    RecordsRetention --> Archived: Past retention period
    RecordsRetention --> HoldOrder: Legal hold applied
    HoldOrder --> RecordsRetention: Hold released
    Archived --> Destroyed: Destruction executed<br/>(RMDestroyAction)
    Destroyed --> [*]
    Completed --> [*]
```

---

## i. State Diagram: Queue Item

Lifecycle states of a WorkItem within the queue system.

```mermaid
stateDiagram-v2
    [*] --> Unassigned: usp_WF_AddQueueItem<br/>reservedBy=null, assignedTo=null

    Unassigned --> Reserved: ReserveAction<br/>reservedBy=userId
    Unassigned --> Assigned: AssignAction<br/>assignedTo=userId

    Reserved --> Assigned: AssignAction<br/>assignedTo set
    Reserved --> Unassigned: UnAssignAction<br/>reservedBy cleared

    Assigned --> InUserInbox: Appears in UserInbox
    InUserInbox --> Processing: User opens item

    Processing --> Commented: BulkCommentsAction<br/>adds notes
    Commented --> Processing: Continue work

    Processing --> MovedToQueue: MoveAction<br/>different queue, same WF
    MovedToQueue --> Unassigned: Re-queued

    Processing --> TransferredOut: TransferHelper<br/>cross-workflow transfer
    TransferredOut --> [*]: Removed from source WF<br/>usp_WF_RemoveQueueItem

    Processing --> Completed: Work finished
    Completed --> Dequeued: usp_WF_RemoveQueueItem
    Dequeued --> [*]

    Assigned --> Unassigned: UnAssignAction<br/>both fields cleared

    note right of Reserved
        reservedBy = userId
        assignedTo = null
        Item visible in queue but locked
    end note

    note right of InUserInbox
        WorkItem cached in UserInboxCache
        Cache expires on transfer/move
    end note
```

---

## j. ER Diagram: Domain Model

Key entities and relationships in the CMWS Oracle database schema.

```mermaid
erDiagram
    WORKFLOW_INSTANCE {
        int workflow_id PK
        string workflow_name
        string tam_key
        string workflow_engine_id
        string metadata_engine_id
        string storage_engine_id
        string reporting_engine_id
        string case_mgmt_engine_id
        string wf_mgmt_engine_id
        string letter_engine_id
        string qr_engine_id
        string ipr_engine_id
    }

    QUEUE {
        bigdecimal queue_id PK
        int workflow_id FK
        string queue_name
        int queue_type_cd
        string description
        boolean active_ind
    }

    QUEUE_ITEM {
        bigdecimal document_id PK
        bigdecimal queue_id FK
        string reserved_by
        string assigned_to
        string prior_user
        bigdecimal prior_queue_id
        timestamp modified_ts
        string modified_by
    }

    DOCUMENT {
        bigdecimal document_id PK
        bigdecimal document_type_cd FK
        string object_id
        int repository_id FK
        timestamp modified_ts
        string modified_by
    }

    DOCUMENT_METADATA {
        bigdecimal document_id FK
        string column_name
        string value
        timestamp modified_ts
    }

    CASE_FOLDER {
        bigdecimal folder_document_id PK
        int control_number
        string ssn
        int case_status_cd
        int case_type_cd
        string lead_case_admin
        string backup_case_admin
        boolean inactive_ind
    }

    CASE_DOCUMENT {
        bigdecimal folder_document_id FK
        bigdecimal document_id FK
        timestamp added_ts
    }

    WORKFLOW_HISTORY {
        bigdecimal document_id FK
        int event_type
        string user_id
        string queue_name
        string additional_info
        timestamp event_ts
    }

    USER_ACCESS {
        string user_id PK
        int workflow_id FK
        string full_name
        boolean active_ind
    }

    USER_FBU {
        string user_id FK
        int workflow_id FK
        string fbu_code
    }

    RECORD_SERIES {
        int series_id PK
        string series_name
        int retention_days
    }

    RECORDS_MAP {
        bigdecimal document_type_cd FK
        int series_id FK
        int workflow_id FK
    }

    HOLD_ORDER {
        int hold_id PK
        string ssn
        string description
        boolean active_ind
    }

    RESTRICTED_CASE {
        int control_number FK
        string user_id FK
    }

    WORKFLOW_INSTANCE ||--o{ QUEUE : "has many"
    QUEUE ||--o{ QUEUE_ITEM : "contains"
    QUEUE_ITEM ||--|| DOCUMENT : "references"
    DOCUMENT ||--o{ DOCUMENT_METADATA : "has"
    DOCUMENT }o--|| DOCUMENT : "document_type"
    CASE_FOLDER ||--o{ CASE_DOCUMENT : "contains"
    CASE_DOCUMENT }o--|| DOCUMENT : "references"
    DOCUMENT ||--o{ WORKFLOW_HISTORY : "has"
    WORKFLOW_INSTANCE ||--o{ USER_ACCESS : "authorizes"
    USER_ACCESS ||--o{ USER_FBU : "belongs to"
    RECORDS_MAP }o--|| RECORD_SERIES : "maps to"
    CASE_FOLDER ||--o{ RESTRICTED_CASE : "restricts access"
```

---

## k. Flowchart: Request Processing (Struts)

How an HTTP request flows through the Struts framework from browser to rendered JSP.

```mermaid
flowchart TD
    A[Browser Request<br/>e.g. /workflow/queueview.go] --> B[WebSphere HTTP Transport]
    B --> C[WEBSEALFilter]
    C --> C1{iv-user header present?}
    C1 -->|Yes| C2[Set ThreadLocal: FacadeBase.setWEBSEALUser]
    C1 -->|No| C3[Use dev_user from web.xml context-param]
    C2 --> D[ActionServlet<br/>org.apache.struts.action.ActionServlet]
    C3 --> D

    D --> E[Lookup action mapping in struts-config.xml<br/>path=/workflow/queueview]
    E --> F{Action type?}

    F -->|"type=class"| G[Instantiate QueueViewAction]
    F -->|"forward=path"| H[Direct JSP forward<br/>e.g. /Workflow/search/search.jsp]

    G --> I[AbstractCMWSAction.execute]
    I --> I1[Build ServiceContext<br/>from session: workflowId, userName]
    I1 --> I2[executeCMWSAction<br/>domain-specific logic]

    I2 --> J[Create Facade instances<br/>WorkflowServiceFacade wfFacade = new...]
    J --> K[wfFacade.init<br/>servlet.getServletContext]
    K --> L[Call facade methods<br/>getQueueItems, getWorkflowQueues, etc.]
    L --> M[Set request attributes<br/>queryResults, columnInfo, queueName]
    M --> N[Return ActionForward<br/>mapping.findForward - success]

    N --> O[Struts forwards to JSP<br/>/Workflow/cmwsmain.jsp]
    H --> O
    O --> P[JSP renders HTML<br/>using request attributes]
    P --> Q[HTML Response to Browser]

    style C fill:#e74c3c,color:#fff
    style D fill:#3498db,color:#fff
    style G fill:#2ecc71,color:#fff
    style O fill:#f39c12,color:#fff
```

---

## l. Flowchart: Auto-Assign Logic

How the AutoAssignHelper determines where to route incoming documents to cases and queues.

```mermaid
flowchart TD
    A[New Document Arrives<br/>via GIAL batch ingestion / Scan] --> B[WorkItem created<br/>in WorkItemCache]
    B --> C[Extract SSN from WorkItem]

    C --> D{SSN present?}
    D -->|No| E[Route to manual queue<br/>enqueueNewFolder with<br/>reservedBy=null]

    D -->|Yes| F[findExistingFolder<br/>usp_CASE_FindFolder by SSN]
    F --> G{Existing folder found?}

    G -->|No - New Case| H[createFolder<br/>usp_STOR_CreateDocument<br/>with FOLDER doc type]
    H --> I[Set folder metadata:<br/>control_number, ssn, name,<br/>business_receipt_date, term_date]
    I --> J[Set case_status_cd = NEW_CASE<br/>Set case_type_cd]
    J --> K[folderDocument:<br/>Copy metadata to folder,<br/>addCaseDocument to link doc]

    G -->|Yes - Existing Case| L{Check case_status_cd}
    L -->|Active| M[Set case_status_cd =<br/>NEW_DOCUMENT_RECEIVED]
    L -->|Inactive| E

    M --> N[folderDocument:<br/>Link document to<br/>existing folder]

    K --> O[Check CaseListing for<br/>lead_case_admin]
    N --> O

    O --> P{Case admin assigned?}
    P -->|Yes| Q[enqueueNewFolder<br/>assignedTo = lead_case_admin<br/>or backup_case_admin]
    P -->|No| R[enqueueNewFolder<br/>assignedTo = null<br/>Goes to general queue]

    Q --> S[Calculate ERD<br/>Expected Return Date]
    R --> S

    S --> T{Document type?}
    T -->|Check| U[Copy check_num, check_amt,<br/>check_date to folder metadata]
    T -->|Kit Request| V[Copy elig_amt<br/>to folder metadata]
    T -->|Other| W[Standard metadata only]

    U --> X[updateBusinessReceiptDate<br/>if not already set]
    V --> X
    W --> X

    X --> Y[addWorkflowHistory<br/>CASE_ACTIVITY event]
    Y --> Z[reserveDocument if needed]
    Z --> AA[Document auto-assigned<br/>and foldered]

    style A fill:#3498db,color:#fff
    style H fill:#2ecc71,color:#fff
    style AA fill:#27ae60,color:#fff
    style E fill:#e67e22,color:#fff
```

---

## m. Deployment: Environment Topology

DEV/QA/STAGE/PROD environment chain with all deployed components. cmws.ear contains CMWS.war + gial.war. ImageViewer JARs are packed inside CMWS.war (served as applet). CMWSReports runs on a separate BI server. CMWSEAR orchestrates the Ant build.

```mermaid
graph TB
    subgraph "Build System"
        CMWSEAR_BUILD["CMWSEAR<br/>Orchestrates Ant build<br/>Produces cmws.ear"]
        CMWSEAR_BUILD -->|"packages"| EAR_ARTIFACT["cmws.ear<br/>(CMWS.war + gial.war)"]
    end

    subgraph "DEV Environment"
        DEV_WAS["WebSphere App Server<br/>DEV"]
        subgraph "DEV cmws.ear"
            DEV_CMWS["CMWS.war<br/>(includes ImageViewer JARs)"]
            DEV_GIAL["gial.war<br/>(batch ingestion pipeline)"]
        end
        DEV_DT["CMWSDeploymentTest"]
        DEV_DB[("Oracle DB<br/>DEV Schema")]
        DEV_EDM["EDM / Filenet<br/>DEV"]
        DEV_BI["BI Server<br/>CMWSReports (347 Crystal Reports)"]

        DEV_WAS --- DEV_CMWS
        DEV_WAS --- DEV_GIAL
        DEV_CMWS --> DEV_DB
        DEV_GIAL -->|"HTTP POST + SOAP"| DEV_CMWS
        DEV_CMWS --> DEV_EDM
    end

    subgraph "QA Environment"
        QA_WAS["WebSphere App Server<br/>QA"]
        subgraph "QA cmws.ear"
            QA_CMWS["CMWS.war<br/>(includes ImageViewer JARs)"]
            QA_GIAL["gial.war<br/>(batch ingestion pipeline)"]
        end
        QA_DB[("Oracle DB<br/>QA Schema")]
        QA_EDM["EDM / Filenet<br/>QA"]
        QA_BI["BI Server<br/>CMWSReports"]

        QA_WAS --- QA_CMWS
        QA_WAS --- QA_GIAL
        QA_CMWS --> QA_DB
        QA_GIAL -->|"HTTP POST + SOAP"| QA_CMWS
        QA_CMWS --> QA_EDM
    end

    subgraph "STAGE Environment"
        STG_WAS["WebSphere App Server<br/>STAGE"]
        STG_TAM["Tivoli Access Manager<br/>WEBSEAL"]
        subgraph "STG cmws.ear"
            STG_CMWS["CMWS.war<br/>(includes ImageViewer JARs)"]
            STG_GIAL["gial.war<br/>(batch ingestion pipeline)"]
        end
        STG_DB[("Oracle DB<br/>STAGE Schema")]
        STG_EDM["EDM / Filenet<br/>STAGE"]
        STG_BI["BI Server<br/>CMWSReports"]

        STG_TAM --> STG_WAS
        STG_WAS --- STG_CMWS
        STG_WAS --- STG_GIAL
        STG_CMWS --> STG_DB
        STG_GIAL -->|"HTTP POST + SOAP"| STG_CMWS
        STG_CMWS --> STG_EDM
    end

    subgraph "PROD Environment"
        PROD_LB["Load Balancer"]
        PROD_TAM["Tivoli Access Manager<br/>WEBSEAL Cluster"]
        PROD_WAS1["WebSphere Node 1"]
        PROD_WAS2["WebSphere Node 2"]
        subgraph "PROD Node 1 cmws.ear"
            PROD_CMWS1["CMWS.war<br/>(includes ImageViewer JARs)"]
            PROD_GIAL1["gial.war<br/>(batch ingestion pipeline)"]
        end
        subgraph "PROD Node 2 cmws.ear"
            PROD_CMWS2["CMWS.war<br/>(includes ImageViewer JARs)"]
            PROD_GIAL2["gial.war<br/>(batch ingestion pipeline)"]
        end
        PROD_DB[("Oracle RAC<br/>PROD")]
        PROD_EDM["EDM / Filenet<br/>PROD"]
        PROD_BI["BI Server<br/>CMWSReports (347 Crystal Reports)"]

        PROD_LB --> PROD_TAM
        PROD_TAM --> PROD_WAS1
        PROD_TAM --> PROD_WAS2
        PROD_WAS1 --- PROD_CMWS1
        PROD_WAS1 --- PROD_GIAL1
        PROD_WAS2 --- PROD_CMWS2
        PROD_WAS2 --- PROD_GIAL2
        PROD_CMWS1 --> PROD_DB
        PROD_CMWS2 --> PROD_DB
        PROD_GIAL1 -->|"HTTP POST + SOAP"| PROD_CMWS1
        PROD_GIAL2 -->|"HTTP POST + SOAP"| PROD_CMWS2
        PROD_CMWS1 --> PROD_EDM
        PROD_CMWS2 --> PROD_EDM
    end

    EAR_ARTIFACT ==>|"Deploy"| DEV_WAS
    DEV_WAS ==>|"Promote"| QA_WAS
    QA_WAS ==>|"Promote"| STG_WAS
    STG_WAS ==>|"Promote"| PROD_LB

    subgraph "JNDI Data Sources"
        DS1["jdbc/wf_common_oracle_database1"]
        DS2["jdbc/wf_common_oracle_database2"]
        DS3["jdbc/meta_common_oracle_database1"]
        DS4["jdbc/meta_common_oracle_database2"]
        DS5["jdbc/conn2lcms"]
        DS6["jdbc/meta2dcms"]
        DS7["jdbc/conn2linx"]
    end

    style PROD_LB fill:#e74c3c,color:#fff
    style PROD_TAM fill:#9b59b6,color:#fff
    style PROD_DB fill:#2c3e50,color:#fff
    style CMWSEAR_BUILD fill:#27ae60,color:#fff
```

---

## n. Package Diagram - Java Package Hierarchy

Shows the package structure and dependencies within CMWSWeb and GIAL.

```mermaid
graph TB
    subgraph "CMWSWeb - com.pru.gi"
        subgraph "workflow (shared framework)"
            wf_config["workflow.config<br/><i>ConfigurationUtility</i><br/><i>WorkflowInstance</i><br/><i>ServiceEngine</i>"]
            wf_common["workflow.common<br/><i>ServiceContext</i><br/><i>ResultDataRow</i><br/><i>MetadataValue</i><br/><i>WorkflowException</i><br/><i>CMWSUtil</i><br/><i>DocumentWrapper</i>"]
            wf_facade["workflow.facade<br/><i>FacadeBase</i><br/><i>WorkflowServiceFacade</i><br/><i>MetadataServiceFacade</i><br/><i>DocumentStorageFacade</i><br/><i>CaseManagementFacade</i><br/><i>LetterServiceFacade</i><br/><i>QualityReviewFacade</i><br/><i>QTSFacade</i><br/><i>NotificationFacades</i>"]
            wf_process["workflow.process<br/><i>IProcessWorkflow</i><br/><i>IProcessMetadata</i><br/><i>IProcessStorage</i><br/><i>IProcessCaseManagement</i><br/><i>IProcessLetter</i><br/><i>IProcessReports</i><br/><i>Factory classes</i>"]
            wf_queues["workflow.queues<br/><i>QueueManager</i><br/><i>WorkflowQueue</i><br/><i>WorkItem/Cache</i><br/><i>UserInbox/Cache</i><br/><i>TransferHelper</i><br/><i>WorkflowHelper</i>"]
            wf_data["workflow.data<br/><i>OracleDataAccess</i>"]
            wf_metadata["workflow.metadata<br/><i>MetadataHelper</i>"]
            wf_logging["workflow.logging<br/><i>LoggingUtility</i>"]
            wf_servlets["workflow.servlets<br/><i>GetDocument</i><br/><i>PutDocument</i><br/><i>GetTemplate</i><br/><i>PutTemplate</i><br/><i>PutLetter</i><br/><i>WEBSEALFilter</i><br/><i>ValidateAccess</i><br/><i>GetReport</i>"]
            wf_utility["workflow.utility<br/><i>RandomGUID</i>"]
        end

        subgraph "workflow.web (Struts layer)"
            wf_actions["workflow.web.actions<br/><i>AbstractCMWSAction</i><br/><i>QueueViewAction</i><br/><i>InboxViewAction</i><br/><i>SearchAction</i><br/><i>ApproveAction</i><br/><i>DenyAction</i><br/><i>MoveAction</i><br/><i>AssignAction</i><br/><i>ReserveAction</i><br/><i>GetNextAction</i>"]
            wf_actionforms["workflow.web.actionforms<br/><i>WorkTypeRequestForm</i><br/><i>FileUploadForm</i><br/><i>EASRequestForm</i>"]
            wf_sub_actions["workflow.web.actions.*<br/><i>.admin</i><br/><i>.caselisting</i><br/><i>.reports</i><br/><i>.recordsadmin</i><br/><i>.recordview</i><br/><i>.customerfile</i><br/><i>.ipr</i><br/><i>.qrsampling</i><br/><i>.controladmin</i><br/><i>.lcms</i><br/><i>.osgli</i><br/><i>.epr</i>"]
        end

        subgraph "Domain Packages"
            pkg_lcms["lcms<br/><i>LCMSFacade</i><br/><i>LCMSProcess</i><br/><i>PayeeGIAL</i>"]
            pkg_osgli["osgli<br/><i>OSGLIFacade</i><br/><i>OSGLIRecordsAdminHelper</i>"]
            pkg_lrk["lrk<br/><i>LPMFacade</i>"]
            pkg_lockbox["lockbox<br/><i>LockboxFacade</i>"]
            pkg_lcnv["lcnv<br/><i>LCNVFacade</i><br/><i>lcnv.queue.AutoAssignHelper</i>"]
            pkg_walmart["walmart<br/><i>WalmartFacade</i>"]
            pkg_gulgvul["gulgvul<br/><i>GULGVULFacade</i><br/><i>GulGvulRecordsAdminHelper</i>"]
            pkg_esc["esc<br/><i>ESCFacade</i>"]
            pkg_cob["cob<br/><i>COBWorkTypeItemFacade</i>"]
            pkg_epr["epr<br/><i>EPRWorkTypeItemFacade</i>"]
            pkg_mu["mu<br/><i>MUFacade</i>"]
            pkg_smcob["smcob<br/>(uses cob structures)"]
            pkg_verizon["verizon<br/>(uses shared framework)"]
        end
    end

    subgraph "GIAL - com.pru.gi.gial (batch ingestion pipeline)"
        gial_svcs["gial.svcs<br/><i>ImageLoaderService</i><br/><i>DocumentRepository</i><br/><i>ContentManagement</i><br/><i>TiffUtility</i><br/><i>BatchControl</i>"]
        gial_process["gial.process<br/><i>DocumentumRepositoryProcess</i><br/><i>(legacy name; routes via EDM to Filenet)</i><br/><i>CMWSContentManagementProcess</i><br/><i>LCMSContentManagementProcess</i><br/><i>Factory classes</i>"]
        gial_config["gial.config<br/><i>ConfigurationUtility</i><br/><i>Repository</i><br/><i>LoaderConfig</i><br/><i>EdmAuthConfig</i>"]
        gial_data["gial.data<br/><i>FilennetDataAccess</i><br/><i>(Filenet access; note double n)</i>"]
        gial_common["gial.svcs.common<br/><i>ServiceContext</i><br/><i>DocumentWrapper</i><br/><i>GIALException</i>"]
    end

    wf_actions --> wf_facade
    wf_actions --> wf_queues
    wf_facade --> wf_process
    wf_facade --> wf_config
    wf_facade --> wf_common
    wf_process --> wf_data
    wf_queues --> wf_data
    wf_queues --> wf_common
    wf_config --> wf_common

    pkg_lcms --> wf_facade
    pkg_lcms --> wf_process
    pkg_lcms --> wf_config
    pkg_osgli --> wf_facade
    pkg_lcnv --> wf_queues
    pkg_lcnv --> wf_facade

    gial_svcs --> gial_process
    gial_svcs --> gial_config
    gial_process --> gial_data
    gial_process --> gial_common

    gial_svcs -.->|"HTTP POST + SOAP"| wf_facade

    style wf_config fill:#3498db,color:#fff
    style wf_facade fill:#2ecc71,color:#fff
    style wf_queues fill:#e67e22,color:#fff
    style gial_svcs fill:#9b59b6,color:#fff
```

---

## o. Gantt-style: Domain Complexity

Relative size and complexity of each business domain based on number of engines configured, facade methods, and feature richness. Longer bars indicate more complex domains with more sub-systems (letters, QR, case management, IPR).

```mermaid
gantt
    title Domain Complexity (relative feature count / engine coverage)
    dateFormat X
    axisFormat %s

    section Active Workflows
    LRK (WF 1) - WF+META+STOR+RPT+CASE+WFMGMT+QR          :0, 7
    Lockbox (WF 2) - WF+META+STOR+RPT+WFMGMT+QR            :0, 6
    OSGLI (WF 3) - WF+META+STOR+RPT+CASE+WFMGMT+QR         :0, 7
    Contracts (WF 4) - WF+META+STOR+WFMGMT                  :0, 4
    Underwriting (WF 5) - WF+META+STOR+WFMGMT               :0, 4
    Proposals (WF 6) - WF+META+STOR+WFMGMT                  :0, 4
    Walmart (WF 7) - WF+META+STOR+RPT+CASE+WFMGMT+LTR+QR   :0, 8
    LCNV (WF 8) - WF+META+STOR+RPT+CASE+WFMGMT+LTR+QR      :0, 8
    GUL/GVUL (WF 9) - WF+META+STOR+RPT+CASE+WFMGMT+QR      :0, 7
    ESC (WF 10) - WF+META+STOR+RPT+CASE+WFMGMT+LTR+QR      :0, 8
    COB (WF 11) - WF+META+STOR+RPT+CASE+WFMGMT              :0, 6
    SMCOB (WF 12) - WF+META+STOR+RPT+CASE+WFMGMT+QR        :0, 7
    Verizon (WF 13) - WF+META+STOR+RPT+WFMGMT+QR+IPR       :0, 7
    EPR (WF 14) - WF+META+STOR+RPT+CASE+WFMGMT+QR          :0, 7

    section LCMS Sub-Workflows
    MU (WF 101) - WF+META+STOR+RPT+WFMGMT                   :0, 5
    LCMS-GLCD (WF 102) - WF+META+STOR+RPT+WFMGMT            :0, 5
    DCMS (WF 103) - WF+META+STOR+WFMGMT                     :0, 4
    LCMS-OSGLI (WF 104) - WF+META+STOR+RPT+WFMGMT           :0, 5
    LCMS-PEB (WF 105) - WF+META+STOR+RPT+WFMGMT             :0, 5
    LCMS-ILI (WF 107) - WF+META+STOR+RPT+WFMGMT             :0, 5
    OSGLI Archive (WF 15) - WF+META+STOR+RPT+CASE+WFMGMT+QR :0, 7

    section Most Complex (Letters + QR + Case)
    Walmart - full suite                                      :crit, 0, 8
    LCNV - full suite + auto-assign                          :crit, 0, 8
    ESC - full suite                                         :crit, 0, 8
```
