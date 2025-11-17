# Beacon File Processor - Visual Architecture Diagrams

This document contains detailed visual diagrams using Mermaid syntax. You can view these diagrams by:
1. Opening this file in GitHub (renders Mermaid automatically)
2. Using a Mermaid preview extension in your IDE
3. Using online tools like https://mermaid.live/

---

## 1. System Overview - Component Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        UI[Web UI / API Client]
    end
    
    subgraph "Upload Layer"
        FU[fb-fileupload<br/>File Upload Servlet]
    end
    
    subgraph "Redis Queue Layer"
        FSQ[FileSplitQ]
        GQ[GroupQ]
        DQ[DltFileQ]
        EQ[ExcludeQ]
        DLQ[DeliveryQ]
    end
    
    subgraph "Processing Layer"
        IS[fb-initialstage<br/>Campaign Poller]
        SS[fb-splitstage<br/>File Splitter]
        GP[fb-groupsprocessor<br/>Group Processor]
        DP[fb-dltfileprocessor<br/>DLT Processor]
        EP[fb-excludeprocessor<br/>Exclude Filter]
        HS[fb-handoverstage<br/>Delivery Handler]
    end
    
    subgraph "Completion Layer"
        CF[fb-campaignfinisher<br/>Completion Tracker]
        DH[fb-downloadhandler<br/>Report Generator]
    end
    
    subgraph "Support Layer"
        CJ[fb-cronjobs<br/>Maintenance]
        IMR[fb-inmemoryrefresh<br/>Cache Refresh]
    end
    
    subgraph "Storage Layer"
        DB[(MariaDB)]
        FS[File System]
        REDIS[(Redis Cache)]
    end
    
    subgraph "External Systems"
        DE[Delivery Engine<br/>Platform]
    end
    
    UI -->|HTTP Upload| FU
    FU -->|Store Files| FS
    FU -->|Create Campaign| DB
    
    IS -->|Poll Campaigns| DB
    IS -->|Push Tasks| FSQ
    IS -->|Push Tasks| GQ
    
    FSQ -->|Consume| SS
    GQ -->|Consume| GP
    DQ -->|Consume| DP
    
    SS -->|Split Files| FS
    SS -->|Push| EQ
    SS -->|Push| DLQ
    
    GP -->|Process| FS
    GP -->|Push to| FSQ
    
    EQ -->|Consume| EP
    EP -->|Filter| FS
    EP -->|Push| DLQ
    
    DLQ -->|Consume| HS
    HS -->|Read Files| FS
    HS -->|Send Messages| DE
    
    CF -->|Poll Status| DB
    CF -->|Update| DB
    CF -->|Clean Queues| REDIS
    
    DH -->|Convert Reports| FS
    
    CJ -->|Cleanup Files| FS
    CJ -->|Update Rates| DB
    
    IMR -->|Refresh| REDIS
    
    style FU fill:#e1f5ff
    style SS fill:#fff4e1
    style EP fill:#ffe1e1
    style HS fill:#e1ffe1
    style CF fill:#f0e1ff
```

---

## 2. Campaign Processing - Sequence Diagram

```mermaid
sequenceDiagram
    participant UI as User Interface
    participant FU as fb-fileupload
    participant DB as Database
    participant IS as fb-initialstage
    participant FSQ as FileSplitQ
    participant SS as fb-splitstage
    participant EQ as ExcludeQ
    participant EP as fb-excludeprocessor
    participant DQ as DeliveryQ
    participant HS as fb-handoverstage
    participant DE as Delivery Engine
    participant CF as fb-campaignfinisher
    
    UI->>FU: Upload File (CSV/XLS/XLSX)
    FU->>FU: Validate & Parse File
    FU->>DB: Create campaign_master<br/>(status=QUEUED)
    FU-->>UI: Return File Stats
    
    loop Every N seconds
        IS->>DB: Poll campaigns<br/>(status=QUEUED)
        DB-->>IS: Campaign Data
        IS->>FSQ: Push Campaign JSON
    end
    
    SS->>FSQ: RPOP Campaign
    SS->>DB: Get Campaign Details
    SS->>SS: Read Master File
    SS->>SS: Split into Chunks
    SS->>DB: Insert campaign_file_splits
    
    alt Has Exclude Groups
        SS->>EQ: LPUSH Split Metadata
        EP->>EQ: RPOPLPUSH to Processing
        EP->>EP: Filter Excluded Numbers
        EP->>EP: Create 2 Files<br/>(Valid & Excluded)
        EP->>DB: Update Exclude Count
        EP->>DQ: LPUSH Valid File
    else No Exclusion
        SS->>DQ: LPUSH Split Metadata
    end
    
    HS->>DQ: RPOPLPUSH to Processing
    HS->>HS: Read Split File
    
    alt Campaign Type
        HS->>HS: ProcessOTM (One-to-Many)
    else
        HS->>HS: ProcessMTM (Many-to-Many)
    else
        HS->>HS: ProcessTEM (Template)
    end
    
    HS->>DE: Send to Platform Queue
    HS->>DB: Update file status=COMPLETED
    HS->>DQ: Remove from Processing
    
    loop Every N seconds
        CF->>DB: Check All Splits Complete
        alt All Complete
            CF->>DB: Update campaign_master<br/>(status=COMPLETED)
            CF->>DQ: Remove Campaign Queue
        end
    end
```

---

## 3. File Upload Flow - Detailed

```mermaid
flowchart TD
    A[User Uploads File] --> B{Valid Request?}
    B -->|No| C[Return Error]
    B -->|Yes| D[Create User Directory]
    
    D --> E{File Type?}
    E -->|ZIP| F[Extract ZIP Contents]
    E -->|Other| G[Save File Directly]
    
    F --> H[Multiple Files Extracted]
    G --> I[Single File Saved]
    H --> J[Create File List]
    I --> J
    
    J --> K[Spawn Concurrent Tasks<br/>FutureTask per File]
    
    K --> L1[Task 1: Parse File]
    K --> L2[Task 2: Parse File]
    K --> L3[Task N: Parse File]
    
    L1 --> M1{Valid Format?}
    L2 --> M2{Valid Format?}
    L3 --> M3{Valid Format?}
    
    M1 -->|Yes| N1[Count Records]
    M1 -->|No| O1[Mark as Failed]
    M2 -->|Yes| N2[Count Records]
    M2 -->|No| O2[Mark as Failed]
    M3 -->|Yes| N3[Count Records]
    M3 -->|No| O3[Mark as Failed]
    
    N1 --> P[Collect Results]
    N2 --> P
    N3 --> P
    O1 --> P
    O2 --> P
    O3 --> P
    
    P --> Q[Push File Paths to<br/>Redis Tracking Queue]
    Q --> R[Build JSON Response]
    R --> S[Return to User]
    
    S --> T{Success Files > 0?}
    T -->|Yes| U[Ready for Campaign Creation]
    T -->|No| V[All Files Failed]
    
    style A fill:#e1f5ff
    style K fill:#fff4e1
    style Q fill:#ffe1e1
    style U fill:#e1ffe1
```

---

## 4. Redis Queue Architecture

```mermaid
graph LR
    subgraph "Work Distribution Queues"
        FSQ[FileSplitQ<br/>List]
        GQ[GroupQ<br/>List]
        DFQ[DltFileQ<br/>List]
        SQ[ScheduleQ<br/>List]
    end
    
    subgraph "Per-Campaign Queues"
        CQ1[campaign_id_1<br/>List]
        CQ2[campaign_id_2<br/>List]
        CQN[campaign_id_N<br/>List]
        
        DQ[DeliveryQ<br/>Master List]
        DQ -.->|Contains IDs| CQ1
        DQ -.->|Contains IDs| CQ2
        DQ -.->|Contains IDs| CQN
    end
    
    subgraph "Processing Tracking"
        PQ1[processing:campaign_id_1<br/>Temporary]
        PQ2[processing:campaign_id_2<br/>Temporary]
        PQN[processing:campaign_id_N<br/>Temporary]
    end
    
    subgraph "Status Queues"
        UQ[UpdateSQLQueue<br/>Batch Updates]
        ENQ[ExcludeNumberQ<br/>Tracking]
        FTQ[FileTrackingQ<br/>Cleanup]
    end
    
    subgraph "Cache Keys"
        CP[config:params<br/>Hash]
        UD[user:details:*<br/>Hash]
        RC[retry:count:*<br/>String]
        HB[heartbeat:*<br/>String with TTL]
    end
    
    FSQ -->|RPOP| P1[Producer]
    P1 -->|LPUSH| CQ1
    CQ1 -->|RPOPLPUSH| PQ1
    PQ1 -->|LREM on success| CQ1
    
    style FSQ fill:#e1f5ff
    style DQ fill:#fff4e1
    style PQ1 fill:#ffe1e1
    style CP fill:#f0e1ff
```

---

## 5. Module Dependency Diagram

```mermaid
graph TD
    subgraph "Shared Modules"
        UTILS[fb-utils<br/>Utilities, DTOs, Singletons]
        LOGGER[fb-logger<br/>Logging Framework]
        PARSER[fb-fileparser<br/>File Parsers]
        BEACON[beaconlib<br/>Platform Library]
    end
    
    subgraph "Service Modules"
        FU[fb-fileupload]
        IS[fb-initialstage]
        SS[fb-splitstage]
        EP[fb-excludeprocessor]
        HS[fb-handoverstage]
        GP[fb-groupsprocessor]
        DP[fb-dltfileprocessor]
        SP[fb-scheduleprocessor]
        CF[fb-campaignfinisher]
        DH[fb-downloadhandler]
        CJ[fb-cronjobs]
        IMR[fb-inmemoryrefresh]
    end
    
    FU --> UTILS
    FU --> LOGGER
    FU --> PARSER
    
    IS --> UTILS
    IS --> LOGGER
    IS --> BEACON
    
    SS --> UTILS
    SS --> LOGGER
    SS --> PARSER
    
    EP --> UTILS
    EP --> LOGGER
    EP --> PARSER
    
    HS --> UTILS
    HS --> LOGGER
    HS --> PARSER
    HS --> BEACON
    
    GP --> UTILS
    GP --> LOGGER
    GP --> PARSER
    
    DP --> UTILS
    DP --> LOGGER
    DP --> PARSER
    
    SP --> UTILS
    SP --> LOGGER
    
    CF --> UTILS
    CF --> LOGGER
    
    DH --> UTILS
    DH --> LOGGER
    
    CJ --> UTILS
    CJ --> LOGGER
    
    IMR --> UTILS
    IMR --> LOGGER
    
    style UTILS fill:#e1f5ff
    style LOGGER fill:#fff4e1
    style PARSER fill:#ffe1e1
```

---

## 6. Campaign State Machine

```mermaid
stateDiagram-v2
    [*] --> QUEUED: Campaign Created
    
    QUEUED --> INITIALIZING: Picked by InitialStage
    INITIALIZING --> SPLITTING: Pushed to FileSplitQ
    
    SPLITTING --> SPLIT_COMPLETE: All chunks created
    SPLITTING --> FAILED: File read error
    
    SPLIT_COMPLETE --> EXCLUDING: Has exclude groups
    SPLIT_COMPLETE --> DELIVERY_QUEUED: No exclusions
    
    EXCLUDING --> DELIVERY_QUEUED: Exclusion complete
    EXCLUDING --> FAILED: Exclusion error
    
    DELIVERY_QUEUED --> DELIVERING: Picked by HandoverStage
    
    DELIVERING --> COMPLETED: All files delivered
    DELIVERING --> PARTIAL: Some files failed
    DELIVERING --> FAILED: All files failed
    
    FAILED --> RETRY: Retry count < max
    RETRY --> SPLITTING: Reprocess
    
    PARTIAL --> COMPLETED: Manual completion
    
    COMPLETED --> [*]
    FAILED --> [*]
    
    note right of QUEUED
        Initial state after
        file upload
    end note
    
    note right of DELIVERING
        Most time is spent
        in this state
    end note
    
    note right of COMPLETED
        Campaign finished,
        ready for cleanup
    end note
```

---

## 7. File Processing Pipeline

```mermaid
graph TD
    A[Master File<br/>campaign.csv] --> B[fb-splitstage]
    
    B --> C1[Split File 1<br/>0-10000 records]
    B --> C2[Split File 2<br/>10001-20000 records]
    B --> C3[Split File 3<br/>20001-30000 records]
    B --> CN[Split File N<br/>N records]
    
    C1 --> D1{Exclude Groups?}
    C2 --> D2{Exclude Groups?}
    C3 --> D3{Exclude Groups?}
    CN --> DN{Exclude Groups?}
    
    D1 -->|Yes| E1[fb-excludeprocessor]
    D1 -->|No| F1[DeliveryQ]
    D2 -->|Yes| E2[fb-excludeprocessor]
    D2 -->|No| F2[DeliveryQ]
    D3 -->|Yes| E3[fb-excludeprocessor]
    D3 -->|No| F3[DeliveryQ]
    DN -->|Yes| EN[fb-excludeprocessor]
    DN -->|No| FN[DeliveryQ]
    
    E1 --> G1[Valid: 8000<br/>Excluded: 2000]
    E2 --> G2[Valid: 7500<br/>Excluded: 2500]
    E3 --> G3[Valid: 9000<br/>Excluded: 1000]
    EN --> GN[Valid: X<br/>Excluded: Y]
    
    G1 --> F1
    G2 --> F2
    G3 --> F3
    GN --> FN
    
    F1 --> H[fb-handoverstage]
    F2 --> H
    F3 --> H
    FN --> H
    
    H --> I{Campaign Type}
    
    I -->|OTM| J1[ProcessOTM<br/>Same message to all]
    I -->|MTM| J2[ProcessMTM<br/>Different messages]
    I -->|TEM| J3[ProcessTEM<br/>Template substitution]
    
    J1 --> K[Delivery Engine Queue]
    J2 --> K
    J3 --> K
    
    K --> L[SMS Gateway]
    
    style A fill:#e1f5ff
    style B fill:#fff4e1
    style E1 fill:#ffe1e1
    style H fill:#e1ffe1
    style K fill:#f0e1ff
```

---

## 8. Group Processing Flow

```mermaid
sequenceDiagram
    participant UI as User Interface
    participant FU as fb-fileupload
    participant DB as Database
    participant IS as fb-initialstage
    participant GQ as GroupQ
    participant GP as fb-groupsprocessor
    participant GFSQ as GroupFileSplitQ
    participant GFSC as GroupFileSplitConsumer
    participant GCQ as GroupCampaignQ
    participant GCC as GroupCampaignConsumer
    participant FSQ as FileSplitQ
    participant SS as fb-splitstage
    
    UI->>FU: Upload Group File
    FU->>DB: Create group_master<br/>(status=QUEUED)
    FU-->>UI: Success
    
    IS->>DB: Poll group_master
    IS->>GQ: Push Group JSON
    
    GP->>GQ: Consume Group
    GP->>GP: Validate Group File
    
    alt Large Group File
        GP->>GFSQ: Push to Split
        GFSC->>GFSQ: Consume
        GFSC->>GFSC: Split Group File
        GFSC->>DB: Update group_files
    else Small Group File
        GP->>DB: Update group_master<br/>(status=PROCESSING)
        GP->>DB: Create group_files
    end
    
    loop Check Completion
        GP->>DB: Check all group_files complete
        alt All Files Complete
            GP->>DB: Update group_master<br/>(status=COMPLETED)
            GP->>GCQ: Push to GroupCampaignQ
        end
    end
    
    GCC->>GCQ: Consume
    GCC->>DB: Create campaign_master<br/>from group
    GCC->>FSQ: Push to FileSplitQ
    
    Note over SS: Continue with<br/>normal campaign flow
    SS->>FSQ: Process Campaign
```

---

## 9. Exclude Processing - Detailed

```mermaid
flowchart TD
    A[ExcludeQ Consumer] --> B[Read Split File Metadata]
    B --> C[Get Campaign Details from DB]
    C --> D[Extract exclude_group_ids]
    
    D --> E{Has Exclude Groups?}
    E -->|No| F[Skip Exclusion]
    E -->|Yes| G[Fetch Exclude Numbers<br/>from Groups]
    
    F --> Z[Push to DeliveryQ]
    
    G --> H[Read Split File<br/>Line by Line]
    H --> I{Number in<br/>Exclude List?}
    
    I -->|Yes| J[Write to<br/>Excluded File]
    I -->|No| K[Write to<br/>Valid File]
    
    J --> L[Increment<br/>Exclude Counter]
    K --> M[Increment<br/>Valid Counter]
    
    L --> N{More Records?}
    M --> N
    
    N -->|Yes| H
    N -->|No| O[Close Files]
    
    O --> P[Update DB:<br/>exclude_count,<br/>valid_count]
    
    P --> Q{Valid Count > 0?}
    Q -->|Yes| R[Update Split File Location<br/>to Valid File]
    Q -->|No| S[Mark File as<br/>COMPLETED<br/>All Excluded]
    
    R --> T[Push Valid File<br/>to DeliveryQ]
    T --> U[Push Excluded File<br/>to ExcludeNumberQ<br/>for tracking]
    
    S --> V[No Delivery Needed]
    
    U --> W[Complete]
    V --> W
    Z --> W
    
    style A fill:#e1f5ff
    style G fill:#fff4e1
    style J fill:#ffe1e1
    style K fill:#e1ffe1
    style T fill:#f0e1ff
```

---

## 10. HandoverStage Processing Types

```mermaid
graph TD
    A[SplitFileConsumer] --> B[Read Campaign Type]
    
    B --> C{Type?}
    
    C -->|OTM| D[ProcessOTM]
    C -->|MTM| E[ProcessMTM]
    C -->|TEM| F[ProcessTEM]
    C -->|GROUP| D
    C -->|QUICK| D
    
    subgraph "ProcessOTM - One Message to Many"
        D --> D1[Read Split File]
        D1 --> D2[Get Single Message<br/>from campaign]
        D2 --> D3[For Each Mobile Number]
        D3 --> D4[Create Payload:<br/>mobile, message, params]
        D4 --> D5[Send to Platform Queue]
        D5 --> D6{More Numbers?}
        D6 -->|Yes| D3
        D6 -->|No| D7[Mark File Complete]
    end
    
    subgraph "ProcessMTM - Many Messages to Many"
        E --> E1[Read Split File]
        E1 --> E2[Each Line Contains:<br/>mobile,message,params]
        E2 --> E3[Parse Line]
        E3 --> E4[Create Payload:<br/>from parsed data]
        E4 --> E5[Send to Platform Queue]
        E5 --> E6{More Lines?}
        E6 -->|Yes| E2
        E6 -->|No| E7[Mark File Complete]
    end
    
    subgraph "ProcessTEM - Template Based"
        F --> F1[Read Split File]
        F1 --> F2[Get Template Details<br/>from campaign]
        F2 --> F3[Each Line Contains:<br/>mobile,var1,var2,...varN]
        F3 --> F4[Parse Line & Variables]
        F4 --> F5[Substitute Variables<br/>in Template]
        F5 --> F6[Create Payload:<br/>mobile, message, params]
        F6 --> F7[Send to Platform Queue]
        F7 --> F8{More Lines?}
        F8 -->|Yes| F3
        F8 -->|No| F9[Mark File Complete]
    end
    
    D7 --> G[Update DB Status]
    E7 --> G
    F9 --> G
    
    G --> H[Remove from<br/>Processing Queue]
    
    style D fill:#e1f5ff
    style E fill:#fff4e1
    style F fill:#ffe1e1
    style G fill:#e1ffe1
```

---

## 11. Error Handling & Retry Flow

```mermaid
flowchart TD
    A[Processing Task] --> B{Exception Occurred?}
    B -->|No| C[Success Path]
    B -->|Yes| D[Catch Exception]
    
    C --> Z[Complete]
    
    D --> E[Log Error]
    E --> F[Get Current Retry Count<br/>from Redis/Metadata]
    
    F --> G{Retry Count <<br/>Max Retry Count?}
    
    G -->|Yes| H[Increment Retry Count]
    G -->|No| I[Max Retries Reached]
    
    H --> J[Update Retry Count<br/>in Metadata]
    J --> K[Calculate Backoff Delay]
    K --> L[Re-push to Queue<br/>with updated metadata]
    
    L --> M[Log Retry Attempt]
    M --> N[Task Will Be Retried]
    
    I --> O[Update DB Status<br/>as FAILED]
    O --> P[Log Final Failure]
    P --> Q{Send Notification?}
    
    Q -->|Yes| R[Send Alert Email/<br/>Notification]
    Q -->|No| S[Skip Notification]
    
    R --> T[Move to DLT Queue]
    S --> T
    
    T --> U[Remove from<br/>Processing Queue]
    U --> V[Manual Intervention<br/>Required]
    
    N --> W[End]
    V --> W
    Z --> W
    
    style D fill:#ffe1e1
    style H fill:#fff4e1
    style I fill:#ff9999
    style T fill:#ffcccc
```

---

## 12. Campaign Completion Tracking

```mermaid
sequenceDiagram
    participant HS as fb-handoverstage
    participant DB as Database
    participant CF as fb-campaignfinisher
    participant RQ as Redis Queues
    participant NS as Notification Service
    
    Note over HS,DB: Multiple HandoverStage instances<br/>process split files
    
    loop For each split file
        HS->>HS: Process Split File
        HS->>DB: Update campaign_file_splits<br/>(status=COMPLETED)
    end
    
    Note over CF: PollerCampaignFilesCompleted<br/>runs every N seconds
    
    loop Polling Loop
        CF->>DB: Query campaign_file_splits<br/>grouped by campaign_id
        DB-->>CF: Split Files Status
        
        CF->>CF: Check if all splits<br/>for a campaign are complete
        
        alt All Splits Complete
            CF->>DB: Update campaign_master<br/>(status=COMPLETED)
            CF->>DB: Aggregate Statistics:<br/>- total_sent<br/>- total_failed<br/>- total_excluded
            CF->>CF: Calculate Campaign<br/>End Time
        end
    end
    
    Note over CF: PollerCampaignMasterCompleted<br/>handles completed campaigns
    
    loop Completion Loop
        CF->>DB: Query completed campaigns
        DB-->>CF: Completed Campaign List
        
        loop For each campaign
            CF->>RQ: Remove Campaign Queue<br/>(campaign_id_XXX)
            CF->>RQ: Remove from DeliveryQ<br/>master list
            CF->>DB: Archive Campaign Data
            CF->>NS: Send Completion<br/>Notification
        end
    end
    
    Note over CF: DQRedisCleaner<br/>cleanup process
    
    loop Cleanup Loop
        CF->>RQ: Scan for Empty<br/>Campaign Queues
        CF->>RQ: Delete Empty Queues
        CF->>DB: Update Cleanup<br/>Timestamp
    end
```

---

## 13. File Cleanup - CronJobs

```mermaid
flowchart TD
    A[CronJob: UnwantedFilesRemoval] --> B[Start Cleanup Process]
    
    B --> C1[Category 1:<br/>Abandoned Upload Files]
    B --> C2[Category 2:<br/>Completed Campaign Files]
    B --> C3[Category 3:<br/>Completed Group Files]
    B --> C4[Category 4:<br/>Completed DLT Files]
    
    subgraph "Abandoned Files"
        C1 --> D1[Query Redis FileTrackingQ]
        D1 --> E1[Get File Paths & Upload Time]
        E1 --> F1{Age > N hours<br/>AND<br/>Not Used in Campaign?}
        F1 -->|Yes| G1[Delete File]
        F1 -->|No| H1[Keep File]
        G1 --> I1[Remove from TrackingQ]
    end
    
    subgraph "Campaign Files"
        C2 --> D2[Query campaign_master<br/>WHERE status=COMPLETED]
        D2 --> E2[Get Campaign Files &<br/>Completion Time]
        E2 --> F2{Age > N days<br/>from completion?}
        F2 -->|Yes| G2[Delete Campaign Files]
        F2 -->|No| H2[Keep Files]
        G2 --> I2[Update DB:<br/>files_deleted=true]
    end
    
    subgraph "Group Files"
        C3 --> D3[Query group_master<br/>WHERE status=COMPLETED]
        D3 --> E3[Get Group Files &<br/>Completion Time]
        E3 --> F3{Age > N days<br/>from completion?}
        F3 -->|Yes| G3[Delete Group Files]
        F3 -->|No| H3[Keep Files]
        G3 --> I3[Update DB:<br/>files_deleted=true]
    end
    
    subgraph "DLT Files"
        C4 --> D4[Query dlt_template_request<br/>WHERE status=COMPLETED]
        D4 --> E4[Get DLT Files &<br/>Processing Time]
        E4 --> F4{Age > N days<br/>from processing?}
        F4 -->|Yes| G4[Delete DLT Files]
        F4 -->|No| H4[Keep Files]
        G4 --> I4[Update DB:<br/>files_deleted=true]
    end
    
    I1 --> J[Generate Cleanup Report]
    I2 --> J
    I3 --> J
    I4 --> J
    H1 --> J
    H2 --> J
    H3 --> J
    H4 --> J
    
    J --> K[Log Statistics:<br/>- Files Deleted<br/>- Space Freed<br/>- Files Retained]
    
    K --> L[Sleep Until Next Run]
    L --> B
    
    style G1 fill:#ffe1e1
    style G2 fill:#ffe1e1
    style G3 fill:#ffe1e1
    style G4 fill:#ffe1e1
    style J fill:#e1ffe1
```

---

## 14. Monitoring & Heartbeat System

```mermaid
graph TB
    subgraph "Module Instances"
        M1[fb-splitstage Instance 1]
        M2[fb-splitstage Instance 2]
        M3[fb-handoverstage Instance 1]
        M4[fb-handoverstage Instance 2]
        M5[fb-excludeprocessor Instance 1]
        MN[... N More Instances]
    end
    
    subgraph "Heartbeat Collection"
        M1 -->|Every 30s| HB1[heartbeat:FP-SplitStage:FileSplitQConsumer:instance1:thread1]
        M2 -->|Every 30s| HB2[heartbeat:FP-SplitStage:FileSplitQConsumer:instance2:thread1]
        M3 -->|Every 30s| HB3[heartbeat:FP-HandoverStage:SplitFileConsumer:instance1:thread1]
        M4 -->|Every 30s| HB4[heartbeat:FP-HandoverStage:SplitFileConsumer:instance2:thread1]
        M5 -->|Every 30s| HB5[heartbeat:FP-ExcludeProcessor:ExcludeConsumer:instance1:thread1]
        MN -->|Every 30s| HBN[heartbeat:...]
    end
    
    subgraph "Redis Heartbeat Storage"
        HB1 --> R[Redis<br/>TTL: 120 seconds]
        HB2 --> R
        HB3 --> R
        HB4 --> R
        HB5 --> R
        HBN --> R
    end
    
    subgraph "Monitoring System"
        R --> MS[Heartbeat Monitor]
        MS --> C1{Check TTL}
        C1 -->|Key Exists| OK[Module Healthy]
        C1 -->|Key Missing| FAIL[Module Down]
        
        OK --> DASH[Monitoring Dashboard]
        FAIL --> ALERT[Alert System]
        
        ALERT --> EMAIL[Email Notification]
        ALERT --> SLACK[Slack Notification]
        ALERT --> SMS[SMS Alert]
    end
    
    subgraph "Prometheus Metrics"
        M1 --> P1[/metrics endpoint]
        M2 --> P2[/metrics endpoint]
        M3 --> P3[/metrics endpoint]
        
        P1 --> PROM[Prometheus Server]
        P2 --> PROM
        P3 --> PROM
        
        PROM --> GRAF[Grafana Dashboard]
    end
    
    DASH --> V[Ops Team View]
    GRAF --> V
    
    style FAIL fill:#ff9999
    style ALERT fill:#ffcccc
    style OK fill:#99ff99
    style DASH fill:#e1f5ff
```

---

## 15. Database Schema - Key Tables

```mermaid
erDiagram
    campaign_master ||--o{ campaign_file_splits : contains
    campaign_master {
        bigint cm_id PK
        varchar username
        varchar file_location
        varchar status
        timestamp created_date
        timestamp completed_date
        int total_records
        int sent_count
        int failed_count
        int exclude_count
        varchar campaign_type
        varchar exclude_group_ids
        int retry_count
        text campaign_message
    }
    
    campaign_file_splits {
        bigint c_f_s_id PK
        bigint cm_id FK
        varchar file_location
        varchar status
        int total_records
        int sent_count
        int failed_count
        int exclude_count
        varchar exclude_file_location
        int retry_count
        timestamp created_date
        timestamp processed_date
    }
    
    group_master ||--o{ group_files : contains
    group_master {
        bigint g_id PK
        varchar username
        varchar group_name
        varchar file_location
        varchar status
        int total_records
        timestamp created_date
        timestamp completed_date
    }
    
    group_files {
        bigint g_f_id PK
        bigint g_id FK
        varchar file_location
        varchar status
        int record_count
        timestamp created_date
        timestamp processed_date
    }
    
    dlt_template_request ||--o{ dlt_templates : generates
    dlt_template_request {
        bigint req_id PK
        varchar username
        varchar file_location
        varchar status
        int template_count
        timestamp created_date
        timestamp processed_date
    }
    
    dlt_templates {
        bigint template_id PK
        bigint req_id FK
        varchar dlt_entity_id
        varchar dlt_template_id
        text template_content
        varchar template_type
        varchar status
        timestamp created_date
    }
    
    download_request {
        bigint download_id PK
        varchar username
        varchar csv_file_path
        varchar excel_file_path
        varchar status
        bigint file_size
        timestamp requested_date
        timestamp completed_date
    }
    
    campaign_master ||--o{ unprocess_numbers : tracks
    unprocess_numbers {
        bigint id PK
        bigint cm_id FK
        varchar mobile_number
        varchar reason
        varchar status
        timestamp created_date
    }
```

---

## 16. Configuration Management Flow

```mermaid
flowchart TD
    A[Configuration Sources] --> B1[properties/common/<br/>Global Properties]
    A --> B2[properties/fileprocessor/<br/>Module Properties]
    A --> B3[properties/profile/<br/>Environment Properties]
    
    B1 --> C[Property Files Loader]
    B2 --> C
    B3 --> C
    
    C --> D{Load Time}
    
    D -->|Application Startup| E1[ConfigParamsTon<br/>Singleton Init]
    D -->|Runtime| E2[Database config_params<br/>Table]
    
    E1 --> F[In-Memory Cache<br/>Properties Configuration]
    E2 --> F
    
    F --> G[Redis Cache<br/>config:params Hash]
    
    G --> H{Access Method}
    
    H -->|Module Needs Config| I[ConfigParamsTon<br/>.getInstance()<br/>.getConfigurationFromconfigParams]
    
    I --> J[Return Map<<br/>String, String>]
    
    J --> K[Module Uses Config]
    
    subgraph "Refresh Mechanism"
        L[fb-inmemoryrefresh<br/>HTTP Endpoint] --> M[/refresh API called]
        M --> N[Clear Cache]
        N --> O[Reload from DB]
        O --> P[Update Redis]
        P --> Q[Notify All Modules]
    end
    
    subgraph "Auto Refresh"
        R[Scheduled Job] --> S[Every N minutes]
        S --> N
    end
    
    L -.->|Triggers| N
    R -.->|Triggers| N
    
    Q --> F
    
    style F fill:#e1f5ff
    style G fill:#fff4e1
    style N fill:#ffe1e1
    style Q fill:#e1ffe1
```

---

## 17. Deployment Architecture

```mermaid
graph TB
    subgraph "Load Balancer Layer"
        LB[Nginx Load Balancer<br/>:80 / :443]
    end
    
    subgraph "Application Layer - Singleton Modules"
        FU1[fb-fileupload:8080]
        IS1[fb-initialstage:8081]
        CF1[fb-campaignfinisher:8087]
        DH1[fb-downloadhandler:8088]
        CJ1[fb-cronjobs:8089]
        IMR1[fb-inmemoryrefresh:8090]
    end
    
    subgraph "Application Layer - Scalable Modules"
        SS1[fb-splitstage:8082]
        SS2[fb-splitstage:8082]
        SS3[fb-splitstage:8082]
        
        EP1[fb-excludeprocessor:8083]
        EP2[fb-excludeprocessor:8083]
        
        HS1[fb-handoverstage:8084]
        HS2[fb-handoverstage:8084]
        HS3[fb-handoverstage:8084]
        HS4[fb-handoverstage:8084]
        HS5[fb-handoverstage:8084]
        
        GP1[fb-groupsprocessor:8085]
        GP2[fb-groupsprocessor:8085]
        
        DP1[fb-dltfileprocessor:8086]
    end
    
    subgraph "Data Layer"
        subgraph "Redis Cluster"
            R1[(Redis Master 1<br/>:6379)]
            R2[(Redis Replica 1<br/>:6380)]
            R3[(Redis Master 2<br/>:6381)]
            R4[(Redis Replica 2<br/>:6382)]
        end
        
        subgraph "Database Cluster"
            DB1[(MariaDB Master<br/>:3306)]
            DB2[(MariaDB Slave 1<br/>:3307)]
            DB3[(MariaDB Slave 2<br/>:3308)]
        end
    end
    
    subgraph "Storage Layer"
        NFS[NFS Shared Storage<br/>File Repository]
    end
    
    subgraph "Monitoring Layer"
        PROM[Prometheus<br/>:9090]
        GRAF[Grafana<br/>:3000]
        ES[Elasticsearch<br/>:9200]
        KB[Kibana<br/>:5601]
    end
    
    LB --> FU1
    LB --> IMR1
    
    FU1 --> R1
    IS1 --> R1
    SS1 --> R1
    SS2 --> R3
    SS3 --> R1
    EP1 --> R1
    EP2 --> R3
    HS1 --> R1
    HS2 --> R3
    HS3 --> R1
    HS4 --> R3
    HS5 --> R1
    GP1 --> R1
    GP2 --> R3
    DP1 --> R1
    CF1 --> R1
    DH1 --> R1
    
    R1 --> R2
    R3 --> R4
    
    FU1 --> DB1
    IS1 --> DB1
    SS1 --> DB1
    EP1 --> DB1
    HS1 --> DB1
    GP1 --> DB1
    DP1 --> DB1
    CF1 --> DB1
    
    DB1 --> DB2
    DB1 --> DB3
    
    FU1 --> NFS
    SS1 --> NFS
    SS2 --> NFS
    SS3 --> NFS
    EP1 --> NFS
    EP2 --> NFS
    HS1 --> NFS
    HS2 --> NFS
    HS3 --> NFS
    HS4 --> NFS
    HS5 --> NFS
    GP1 --> NFS
    GP2 --> NFS
    DP1 --> NFS
    DH1 --> NFS
    
    FU1 -.->|metrics| PROM
    SS1 -.->|metrics| PROM
    HS1 -.->|metrics| PROM
    
    PROM --> GRAF
    
    FU1 -.->|logs| ES
    SS1 -.->|logs| ES
    HS1 -.->|logs| ES
    
    ES --> KB
    
    style LB fill:#e1f5ff
    style R1 fill:#fff4e1
    style DB1 fill:#ffe1e1
    style NFS fill:#e1ffe1
    style PROM fill:#f0e1ff
```

---

These diagrams provide a comprehensive visual representation of the Beacon File Processor system architecture, data flows, and operational workflows. You can copy these Mermaid diagrams into any Mermaid-compatible viewer to see them rendered as visual diagrams.
