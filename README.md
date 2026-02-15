# REST-API-RECONCILIATION


# Mermaid Diagrams for Production Async Sync Pattern

## 1. Overall Async Sync Flow

```mermaid
sequenceDiagram
    participant User
    participant Salesforce UI
    participant Apex Controller
    participant Queueable Job
    participant Master API
    participant Database
    participant Platform Event

    User->>Salesforce UI: Update Account
    Salesforce UI->>Apex Controller: Save changes
    Apex Controller->>Database: Update Account (Status: PENDING)
    Apex Controller->>Queueable Job: Enqueue AsyncSyncJob
    Apex Controller-->>User: ✅ Saved (immediate response)
    
    Note over User,Salesforce UI: User can continue working
    
    Queueable Job->>Database: Update Status → PROCESSING
    Queueable Job->>Master API: PUT /accounts/{id}
    
    alt Success
        Master API-->>Queueable Job: 200 OK + Full Object
        Queueable Job->>Queueable Job: Validate Response
        Queueable Job->>Queueable Job: Check Version
        Queueable Job->>Database: Update Account + Status → SYNCED
        Queueable Job->>Platform Event: Publish Success Event
        Platform Event-->>User: 🔔 Sync Complete
    else API Error
        Master API-->>Queueable Job: 500 Server Error
        Queueable Job->>Database: Update Status → ERROR
        Queueable Job->>Queueable Job: Schedule Retry
        Queueable Job->>Platform Event: Publish Error Event
        Platform Event-->>User: ⚠️ Sync Failed (will retry)
    else Validation Error
        Master API-->>Queueable Job: 200 OK + Invalid Data
        Queueable Job->>Queueable Job: Data Validation Failed
        Queueable Job->>Database: Update Status → VALIDATION_FAILED
        Queueable Job->>Platform Event: Publish Validation Error
        Platform Event-->>User: ❌ Data Issue (manual review)
    end
```

## 2. Job State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: User saves record
    
    PENDING --> PROCESSING: Job starts execution
    PENDING --> ERROR: Job fails to start
    
    PROCESSING --> SYNCED: Success
    PROCESSING --> ERROR: API error (retryable)
    PROCESSING --> VALIDATION_FAILED: Bad data
    PROCESSING --> VERSION_CONFLICT: Stale version
    PROCESSING --> SKIPPED: Already synced
    
    ERROR --> RETRY_PENDING: Auto-retry scheduled
    RETRY_PENDING --> PROCESSING: Retry attempt
    
    ERROR --> DEAD_LETTER: Max retries exceeded
    
    VALIDATION_FAILED --> PENDING: Manual correction
    VERSION_CONFLICT --> PENDING: Conflict resolved
    
    SYNCED --> [*]
    DEAD_LETTER --> [*]
    SKIPPED --> [*]
    
    note right of SYNCED
        Final success state
        ExternalVersion updated
    end note
    
    note right of ERROR
        Transient failures
        Network, timeout, 5xx
    end note
    
    note right of VALIDATION_FAILED
        Permanent failures
        Bad data, missing fields
    end note
```

## 3. Error Handling & Retry Logic

```mermaid
flowchart TD
    Start([Queueable Job Starts]) --> UpdateStatus[Update Status: PROCESSING]
    UpdateStatus --> CallAPI[HTTP Callout to Master API]
    
    CallAPI --> CheckHTTP{HTTP Status?}
    
    CheckHTTP -->|200 OK| ParseResponse[Parse JSON Response]
    CheckHTTP -->|4xx Client Error| LogError[Log Permanent Error]
    CheckHTTP -->|5xx Server Error| CheckRetry{Retry Count < 3?}
    CheckHTTP -->|Timeout| CheckRetry
    
    ParseResponse --> ValidateData{Valid Data?}
    
    ValidateData -->|Yes| CheckVersion{Version Check}
    ValidateData -->|No| ValidationFailed[Status: VALIDATION_FAILED]
    
    CheckVersion -->|Newer| ApplyUpdate[Update Salesforce Record]
    CheckVersion -->|Same/Older| SkipUpdate[Status: SKIPPED]
    CheckVersion -->|Conflict| ConflictDetected[Status: VERSION_CONFLICT]
    
    ApplyUpdate --> Success[Status: SYNCED]
    Success --> PublishSuccess[Publish Success Event]
    PublishSuccess --> End([End])
    
    CheckRetry -->|Yes| ScheduleRetry[Schedule Retry<br/>Exponential Backoff]
    CheckRetry -->|No| DeadLetter[Status: DEAD_LETTER]
    
    ScheduleRetry --> RetryStatus[Status: RETRY_PENDING]
    RetryStatus --> End
    
    LogError --> PermError[Status: ERROR]
    PermError --> PublishError[Publish Error Event]
    PublishError --> End
    
    ValidationFailed --> PublishValidation[Publish Validation Event]
    PublishValidation --> End
    
    SkipUpdate --> End
    ConflictDetected --> PublishConflict[Publish Conflict Event]
    PublishConflict --> End
    
    DeadLetter --> Alert[Send Alert to Ops]
    Alert --> End
    
    style Success fill:#90EE90
    style ValidationFailed fill:#FFB6C1
    style DeadLetter fill:#FF6B6B
    style PermError fill:#FFA07A
    style ConflictDetected fill:#FFD700
```

## 4. Complete System Architecture

```mermaid
graph TB
    subgraph "User Interface"
        User([User])
        LWC[Lightning Web Component]
        Dashboard[Sync Monitor Dashboard]
    end
    
    subgraph "Salesforce Platform"
        ApexController[Apex Controller]
        QueueableJob[AsyncSyncJob<br/>Queueable]
        Database[(Salesforce Database)]
        PlatformEvent[Platform Events]
        
        subgraph "Background Jobs"
            AnomalyDetector[Anomaly Detector<br/>Hourly]
            ReconciliationBatch[Reconciliation Batch<br/>Weekly]
            RetryScheduler[Retry Scheduler<br/>Every 5 min]
        end
        
        subgraph "Monitoring"
            MetricsCollector[Metrics Collector]
            AlertManager[Alert Manager]
        end
    end
    
    subgraph "External System"
        MasterAPI[Master REST API]
        MasterDB[(Master Database)]
    end
    
    subgraph "Logging & Alerts"
        SyncLog[(Sync Logs)]
        AlertLog[(Alert Log)]
        Slack[Slack Notifications]
        Email[Email Alerts]
    end
    
    User -->|1. Edit Record| LWC
    LWC -->|2. Save| ApexController
    ApexController -->|3. Update Status: PENDING| Database
    ApexController -->|4. Enqueue| QueueableJob
    ApexController -->|5. Return Success| LWC
    
    QueueableJob -->|6. Callout| MasterAPI
    MasterAPI -->|7. Query/Update| MasterDB
    MasterDB -->|8. Response| MasterAPI
    MasterAPI -->|9. Full Object| QueueableJob
    
    QueueableJob -->|10. Validate & Update| Database
    QueueableJob -->|11. Publish Event| PlatformEvent
    
    PlatformEvent -->|12. Notify| LWC
    PlatformEvent -->|13. Trigger| MetricsCollector
    
    Database -->|Status Data| Dashboard
    MetricsCollector -->|Metrics| Dashboard
    
    AnomalyDetector -->|Check| Database
    AnomalyDetector -->|Alert if threshold| AlertManager
    
    ReconciliationBatch -->|Fetch All| Database
    ReconciliationBatch -->|Compare| MasterAPI
    ReconciliationBatch -->|Log Discrepancies| SyncLog
    
    RetryScheduler -->|Find ERROR records| Database
    RetryScheduler -->|Re-enqueue| QueueableJob
    
    AlertManager -->|Critical Alerts| Slack
    AlertManager -->|Error Reports| Email
    AlertManager -->|Log All| AlertLog
    
    QueueableJob -.->|Log Events| SyncLog
    
    style QueueableJob fill:#4A90E2
    style Database fill:#50C878
    style MasterAPI fill:#FF6B6B
    style Dashboard fill:#FFD700
```

## 5. Data Validation Pipeline

```mermaid
flowchart LR
    subgraph "Response Received"
        APIResponse[API Response<br/>200 OK]
    end
    
    subgraph "Validation Pipeline"
        V1{HTTP Status<br/>Valid?}
        V2{JSON<br/>Parseable?}
        V3{Required<br/>Fields?}
        V4{Data Types<br/>Correct?}
        V5{Business<br/>Rules?}
        V6{Version<br/>Check?}
        V7{Checksum<br/>Valid?}
    end
    
    subgraph "Actions"
        Apply[Apply Update]
        Reject[Reject & Log]
        Skip[Skip Update]
    end
    
    APIResponse --> V1
    V1 -->|✅ 200| V2
    V1 -->|❌ 4xx/5xx| Reject
    
    V2 -->|✅ Valid JSON| V3
    V2 -->|❌ Parse Error| Reject
    
    V3 -->|✅ id, name, etc.| V4
    V3 -->|❌ Missing fields| Reject
    
    V4 -->|✅ String, Number| V5
    V4 -->|❌ Type mismatch| Reject
    
    V5 -->|✅ Status valid| V6
    V5 -->|❌ Invalid status| Reject
    
    V6 -->|✅ Newer version| V7
    V6 -->|❌ Stale/same| Skip
    
    V7 -->|✅ Match| Apply
    V7 -->|❌ Mismatch| Reject
    
    style Apply fill:#90EE90
    style Reject fill:#FFB6C1
    style Skip fill:#FFD700
```

## 6. Reconciliation Process Flow

```mermaid
sequenceDiagram
    participant Scheduler
    participant ReconciliationBatch
    participant Salesforce DB
    participant Master API
    participant Report
    participant Admin

    Note over Scheduler: Sunday 2:00 AM
    Scheduler->>ReconciliationBatch: Execute Weekly Job
    
    ReconciliationBatch->>Salesforce DB: Query all Accounts<br/>with ExternalId
    Salesforce DB-->>ReconciliationBatch: 10,000 records
    
    loop For each batch (200 records)
        ReconciliationBatch->>Master API: GET /accounts/{id}
        Master API-->>ReconciliationBatch: Remote data
        
        ReconciliationBatch->>ReconciliationBatch: Compare:<br/>- Version<br/>- Name<br/>- Email<br/>- Status
        
        alt Data Matches
            ReconciliationBatch->>ReconciliationBatch: ✅ OK - no action
        else Version Mismatch
            ReconciliationBatch->>Report: Log: Version drift
        else Data Mismatch
            ReconciliationBatch->>Report: Log: Data discrepancy
        else Missing in Remote
            ReconciliationBatch->>Report: Log: Orphaned record
        end
    end
    
    ReconciliationBatch->>Report: Generate Summary Report
    Report->>Admin: Email:<br/>- Total checked: 10,000<br/>- Discrepancies: 47<br/>- Actions needed: 12
    
    Admin->>Salesforce DB: Review & Fix Issues
```

## 7. Monitoring Dashboard Data Flow

```mermaid
graph LR
    subgraph "Data Sources"
        DB[(Salesforce DB)]
        SyncLog[(Sync Logs)]
        JobQueue[(AsyncApexJob)]
    end
    
    subgraph "Metrics Calculation"
        Calc1[Success Rate<br/>Calculator]
        Calc2[Status Breakdown<br/>Aggregator]
        Calc3[Avg Duration<br/>Calculator]
        Calc4[Error Pattern<br/>Detector]
    end
    
    subgraph "Dashboard Components"
        Gauge[Success Rate Gauge]
        Pie[Status Breakdown<br/>Pie Chart]
        Line[Sync Duration<br/>Line Chart]
        Table[Recent Errors<br/>Table]
        Alert[Alert Banner]
    end
    
    DB -->|SyncStatus__c| Calc1
    DB -->|SyncStatus__c| Calc2
    SyncLog -->|Duration| Calc3
    SyncLog -->|Error messages| Calc4
    JobQueue -->|Job status| Calc1
    
    Calc1 -->|95.8%| Gauge
    Calc2 -->|SYNCED: 9500<br/>ERROR: 420<br/>PENDING: 80| Pie
    Calc3 -->|avg: 347ms| Line
    Calc4 -->|Top 5 errors| Table
    
    Calc1 -->|< 90%?| Alert
    
    style Gauge fill:#90EE90
    style Alert fill:#FFB6C1
```

## 8. Retry Strategy with Exponential Backoff

```mermaid
graph TD
    Start([Error Occurred]) --> CheckRetry{Retry Count?}
    
    CheckRetry -->|0| Retry1[Retry #1<br/>Wait: 5 seconds]
    CheckRetry -->|1| Retry2[Retry #2<br/>Wait: 25 seconds]
    CheckRetry -->|2| Retry3[Retry #3<br/>Wait: 125 seconds]
    CheckRetry -->|3+| DLQ[Dead Letter Queue<br/>Manual Review]
    
    Retry1 --> Execute1[Execute Sync Job]
    Retry2 --> Execute2[Execute Sync Job]
    Retry3 --> Execute3[Execute Sync Job]
    
    Execute1 --> Check1{Success?}
    Execute2 --> Check2{Success?}
    Execute3 --> Check3{Success?}
    
    Check1 -->|Yes| Success1[✅ SYNCED]
    Check1 -->|No| IncrementCount1[Retry Count = 1]
    IncrementCount1 --> CheckRetry
    
    Check2 -->|Yes| Success2[✅ SYNCED]
    Check2 -->|No| IncrementCount2[Retry Count = 2]
    IncrementCount2 --> CheckRetry
    
    Check3 -->|Yes| Success3[✅ SYNCED]
    Check3 -->|No| IncrementCount3[Retry Count = 3]
    IncrementCount3 --> CheckRetry
    
    DLQ --> Alert[Alert Admins]
    Alert --> End([End - Manual Fix])
    
    Success1 --> End2([End - Success])
    Success2 --> End2
    Success3 --> End2
    
    style Success1 fill:#90EE90
    style Success2 fill:#90EE90
    style Success3 fill:#90EE90
    style DLQ fill:#FF6B6B
    
    Note1[Backoff Formula:<br/>delay = base * 5^retry_count<br/>base = 1 second]
    
    style Note1 fill:#E6E6FA
```

## 9. Version Conflict Detection & Resolution

```mermaid
flowchart TD
    Start([Sync Job Receives Response]) --> ExtractVersion[Extract Version from Response]
    
    ExtractVersion --> QueryLocal[Query Local Record<br/>for Current Version]
    
    QueryLocal --> Compare{Version<br/>Comparison}
    
    Compare -->|Remote > Local| Safe[✅ Safe to Update]
    Compare -->|Remote = Local| Duplicate[Same Version<br/>Skip Update]
    Compare -->|Remote < Local| Conflict[⚠️ Version Conflict]
    
    Safe --> CheckModTime{Local Modified<br/>Recently?}
    
    CheckModTime -->|No| ApplyUpdate[Apply Update<br/>Status: SYNCED]
    CheckModTime -->|Yes, During Sync| DetectConflict[Concurrent Modification<br/>Detected]
    
    DetectConflict --> MarkConflict[Status: VERSION_CONFLICT<br/>Store Both Versions]
    
    MarkConflict --> CreateTask[Create Task for<br/>Manual Review]
    
    CreateTask --> NotifyUser[Notify Record Owner:<br/>'Conflict needs resolution']
    
    NotifyUser --> UIPrompt{User Action}
    
    UIPrompt -->|Use Remote| AcceptRemote[Apply Remote Version<br/>Increment Local Version]
    UIPrompt -->|Use Local| PushLocal[Push Local to Remote<br/>Via UPDATE API]
    UIPrompt -->|Merge| ManualMerge[User Merges Fields<br/>Manually]
    
    Duplicate --> Log[Log: Already Synced]
    Conflict --> LogConflict[Log: Stale Remote Data]
    
    ApplyUpdate --> End([End])
    AcceptRemote --> End
    PushLocal --> End
    ManualMerge --> End
    Log --> End
    LogConflict --> End
    
    style ApplyUpdate fill:#90EE90
    style MarkConflict fill:#FFD700
    style DetectConflict fill:#FFA500
```

## 10. Complete Timeline View

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant UI
    participant Apex
    participant Queue as Queueable Job
    participant API as Master API
    participant DB as Salesforce DB
    participant PE as Platform Event

    Note over User,UI: T+0s: User Action
    User->>UI: Edit record
    UI->>Apex: Save request
    
    Note over Apex,DB: T+0.5s: Initial Save
    Apex->>DB: Update (Status: PENDING)
    Apex->>Queue: Enqueue AsyncSyncJob
    Apex-->>UI: Success response
    UI-->>User: ✅ Saved (continue working)
    
    Note over Queue,API: T+2s: Background Processing
    Queue->>DB: Update (Status: PROCESSING)
    Queue->>API: PUT /accounts/{id}
    
    Note over API: T+2.5s: Master Processing
    API->>API: Query database (300ms)
    API->>API: Update record (100ms)
    API-->>Queue: 200 OK + Full Object (500ms total)
    
    Note over Queue: T+3s: Validation (200ms)
    Queue->>Queue: Parse JSON (50ms)
    Queue->>Queue: Validate fields (50ms)
    Queue->>Queue: Check version (50ms)
    Queue->>Queue: Verify checksum (50ms)
    
    Note over Queue,DB: T+3.2s: Update Salesforce
    Queue->>DB: Update record (100ms)
    Queue->>DB: Update Status: SYNCED
    Queue->>PE: Publish success event (50ms)
    
    Note over PE,User: T+3.4s: Notification
    PE-->>UI: Event received
    UI-->>User: 🔔 Sync complete!
    
    Note over Queue: T+3.2s: Background (parallel)
    Queue->>DB: Log sync event
    Queue->>DB: Update metrics
```

---

These diagrams cover:
1. **Sequence diagram** - User interaction flow
2. **State machine** - Job lifecycle states
3. **Flowchart** - Error handling logic
4. **Architecture** - Complete system overview
5. **Validation pipeline** - Data quality checks
6. **Reconciliation** - Weekly audit process
7. **Monitoring** - Dashboard data flow
8. **Retry logic** - Exponential backoff
9. **Conflict resolution** - Version management
10. **Timeline** - End-to-end timing

You can copy these into any Mermaid-compatible tool (Markdown editors, Mermaid Live Editor, Notion, GitHub, etc.) to visualize the complete async sync architecture! 🎨
