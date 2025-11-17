# Beacon File Processor - System Architecture & Code Flow

## 📋 Table of Contents
1. [System Overview](#system-overview)
2. [Module Architecture](#module-architecture)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [Detailed Module Descriptions](#detailed-module-descriptions)
5. [Technology Stack](#technology-stack)
6. [Queue Architecture](#queue-architecture)

---

## 🎯 System Overview

**Beacon File Processor** is a distributed, multi-module Java-based system designed for processing bulk SMS campaigns, groups, and DLT (Distributed Ledger Technology) template files. The system follows a queue-based microservices architecture where each module performs specific tasks and communicates via Redis queues.

### Key Characteristics:
- **Architecture**: Microservices with queue-based communication
- **Primary Technology**: Java 21, Maven multi-module project
- **Message Broker**: Redis (for queues and caching)
- **Database**: MariaDB (via JDBC)
- **File Processing**: Supports CSV, XLS, XLSX, TXT, and ZIP files
- **Campaign Types**: OTM (One-to-Many), MTM (Many-to-Many), TEM (Template), GROUP, QUICK

---

## 🏗️ Module Architecture

### Core Modules (15 modules)

```
beacon-fileprocessor (Parent)
├── fb-logger              → Centralized logging framework
├── fb-utils               → Common utilities and DTOs
├── fb-fileupload          → HTTP servlet for file uploads
├── fb-inmemoryrefresh     → In-memory cache refresh servlet
├── fb-initialstage        → Campaign polling and initialization
├── fb-fileparser          → File parsing utilities (CSV, XLS, XLSX)
├── fb-splitstage          → File splitting into chunks
├── fb-excludeprocessor    → Number exclusion processing
├── fb-handoverstage       → Handover to delivery engine
├── fb-groupsprocessor     → Group campaign processing
├── fb-dltfileprocessor    → DLT template file processing
├── fb-scheduleprocessor   → Scheduled campaign processor
├── fb-campaignfinisher    → Campaign completion tracking
├── fb-downloadhandler     → CSV to Excel conversion
├── fb-cronjobs            → Maintenance tasks
├── beaconlib              → Beacon library dependencies
├── properties             → Configuration files
├── docker-fileprocessor   → Docker deployment configs
└── thirtypartyjar         → Third-party JAR dependencies
```

---

## 📊 Data Flow Diagrams

### 1. High-Level System Flow

```
┌─────────────────┐
│   User/Client   │
│      (UI)       │
└────────┬────────┘
         │ HTTP POST (multipart/form-data)
         ↓
┌────────────────────────────────────────────────────────────┐
│                    FB-FILEUPLOAD                           │
│  - FilesSaver Servlet                                      │
│  - Parses CSV/XLS/XLSX/ZIP files                          │
│  - Validates file format                                   │
│  - Stores files in filesystem                              │
│  - Tracks files in Redis for cleanup                       │
└────────┬───────────────────────────────────────────────────┘
         │
         ↓ (File stored, Campaign created in DB)
┌────────────────────────────────────────────────────────────┐
│                  FB-INITIALSTAGE                           │
│  - CampaignMasterPoller (Thread)                          │
│  - CampaignGroupsPoller (Thread)                          │
│  - Polls DB for new campaigns                              │
│  - Pushes to FileSplitQ                                    │
└────────┬───────────────────────────────────────────────────┘
         │
         ↓ JSON payload pushed to Redis
         │
    ┌────┴─────────────┬──────────────┬──────────────┐
    ↓                  ↓              ↓              ↓
┌───────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐
│ FileSplitQ│  │   GroupQ     │  │  DltFileQ  │  │  ScheduleQ   │
│  (Redis)  │  │   (Redis)    │  │  (Redis)   │  │   (Redis)    │
└─────┬─────┘  └──────┬───────┘  └─────┬──────┘  └──────┬───────┘
      │               │                │                 │
      ↓               ↓                ↓                 ↓
┌────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐
│FB-SPLITSTG │  │FB-GROUPSPROC │  │FB-DLTPROC  │  │FB-SCHEDPROC  │
│FileSplitQ  │  │GroupsQ       │  │DltFileQ    │  │ScheduleProc  │
│Consumer    │  │Consumer      │  │Consumer    │  │              │
└─────┬──────┘  └──────┬───────┘  └─────┬──────┘  └──────────────┘
      │                │                 │
      ↓                ↓                 ↓
      └────────────────┴─────────────────┘
                       │
                       ↓ (Split files generated)
          ┌────────────────────────┐
          │   ExcludeQ (Redis)     │
          │  [campaign_id_exclude] │
          └────────────┬───────────┘
                       │
                       ↓
          ┌────────────────────────┐
          │   FB-EXCLUDEPROCESSOR  │
          │  - ExcludeConsumer     │
          │  - Filters numbers     │
          │  - Removes exclusions  │
          └────────────┬───────────┘
                       │
                       ↓ (Non-excluded numbers)
          ┌────────────────────────┐
          │   DeliveryQ (Redis)    │
          │   [campaign_id]        │
          └────────────┬───────────┘
                       │
                       ↓
          ┌────────────────────────┐
          │   FB-HANDOVERSTAGE     │
          │  - SplitFileConsumer   │
          │  - ProcessOTM          │
          │  - ProcessMTM          │
          │  - ProcessTEM          │
          └────────────┬───────────┘
                       │
                       ↓ (Sends to Delivery Engine)
          ┌────────────────────────┐
          │  Platform Delivery     │
          │      Engine            │
          │  (External Platform)   │
          └────────────────────────┘
```

### 2. File Upload Flow (Detailed)

```
┌───────────────────────────────────────────────────────────────────┐
│                     FILE UPLOAD FLOW                              │
└───────────────────────────────────────────────────────────────────┘

User uploads file(s) → FilesSaver Servlet
                              │
                              ↓
                    ┌─────────────────────┐
                    │   Validate Request  │
                    │  - username param   │
                    │  - frompage param   │
                    └──────────┬──────────┘
                              │
                              ↓
                    ┌─────────────────────┐
                    │  Create Directory   │
                    │  /files/username/   │
                    └──────────┬──────────┘
                              │
                    ┌─────────┴──────────┐
                    │                    │
        ┌───────────↓─────────┐   ┌─────↓──────────┐
        │  ZIP File?          │   │  Regular File  │
        │  Extract contents   │   │  Save directly │
        └───────────┬─────────┘   └─────┬──────────┘
                    │                    │
                    └─────────┬──────────┘
                              │
                              ↓
                    ┌─────────────────────┐
                    │ FileReadService     │
                    │ (Concurrent Tasks)  │
                    │ - Count records     │
                    │ - Validate format   │
                    └──────────┬──────────┘
                              │
                              ↓
                    ┌─────────────────────┐
                    │ Push to Redis       │
                    │ (Tracking Queue)    │
                    │ For cleanup         │
                    └──────────┬──────────┘
                              │
                              ↓
                    ┌─────────────────────┐
                    │ Return JSON Response│
                    │ - Total count       │
                    │ - File names        │
                    │ - Success/Failed    │
                    └─────────────────────┘
```

### 3. Campaign Processing Flow (Detailed)

```
┌───────────────────────────────────────────────────────────────────┐
│                   CAMPAIGN PROCESSING FLOW                        │
└───────────────────────────────────────────────────────────────────┘

Database: campaign_master (status='QUEUED')
           │
           ↓ (Polled every N seconds)
┌──────────────────────────┐
│  FB-INITIALSTAGE         │
│  CampaignMasterPoller    │
│  - Queries campaign_master
│  - Gets pending campaigns│
│  - maxRetryCount check   │
└──────────┬───────────────┘
           │
           ↓ Push JSON to Redis
┌──────────────────────────┐
│   FileSplitQ (Redis)     │
│   List<campaign_json>    │
└──────────┬───────────────┘
           │
           ↓ rpop() by Consumer
┌──────────────────────────┐
│  FB-SPLITSTAGE           │
│  FileSplitQConsumer      │
│  - MasterFileSplitHandler│
│  - Read master file      │
│  - Split into chunks     │
│  - Chunk size: N records │
└──────────┬───────────────┘
           │
           ↓ Generate split files
Database: campaign_file_splits
   ┌───────┴────────┬─────────────────┐
   │                │                 │
Split File 1    Split File 2    Split File N
   │                │                 │
   └────────────────┴─────────────────┘
                    │
                    ↓ Check if exclusion needed
          ┌─────────┴──────────┐
          │  Has exclude_group_ids?
          └─────────┬──────────┘
                    │
        ┌───────────┴────────────┐
        ↓ YES                    ↓ NO
┌────────────────┐      ┌────────────────┐
│  Push to       │      │  Push to       │
│  ExcludeQ      │      │  DeliveryQ     │
│  (Redis)       │      │  (Redis)       │
└────────┬───────┘      └────────┬───────┘
         │                       │
         ↓                       │
┌────────────────┐               │
│FB-EXCLUDEPROC  │               │
│ExcludeConsumer │               │
│- Filter numbers│               │
│- Create 2 files│               │
│  1. Excluded   │               │
│  2. Valid      │               │
└────────┬───────┘               │
         │                       │
         ↓ Push valid numbers    │
         └───────────────────────┘
                    │
                    ↓
         ┌──────────────────┐
         │   DeliveryQ      │
         │   (Redis)        │
         │  Per campaign ID │
         └──────────┬───────┘
                    │
                    ↓ rpoplpush()
         ┌──────────────────┐
         │ FB-HANDOVERSTAGE │
         │SplitFileConsumer │
         │- ProcessOTM      │
         │- ProcessMTM      │
         │- ProcessTEM      │
         │- Read split file │
         │- Send to Platform│
         └──────────┬───────┘
                    │
                    ↓
         ┌──────────────────┐
         │ Delivery Engine  │
         │  (Platform Q)    │
         └──────────────────┘
```

### 4. Group Processing Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                     GROUP PROCESSING FLOW                         │
└───────────────────────────────────────────────────────────────────┘

Database: group_master (status='QUEUED')
           │
           ↓ (Polled by InitialStage)
┌──────────────────────────┐
│  FB-INITIALSTAGE         │
│  CampaignGroupsPoller    │
└──────────┬───────────────┘
           │
           ↓ Push to GroupQ
┌──────────────────────────┐
│    GroupQ (Redis)        │
└──────────┬───────────────┘
           │
           ↓ rpop() by Consumer
┌──────────────────────────┐
│  FB-GROUPSPROCESSOR      │
│  GroupsQConsumer         │
│  - MasterFileSplitHandler│
│  - Process group file    │
│  - Split into chunks     │
└──────────┬───────────────┘
           │
           ↓
Database: group_files (status updated)
           │
           ↓ When group complete
┌──────────────────────────┐
│  GroupsCampaignQ         │
│  (Create campaigns from  │
│   completed groups)      │
└──────────┬───────────────┘
           │
           ↓
┌──────────────────────────┐
│  GroupsCampaignQConsumer │
│  - Create campaign_master│
│  - Push to FileSplitQ    │
└──────────┬───────────────┘
           │
           ↓ (Joins main campaign flow)
           [Continue with Campaign Processing Flow]
```

### 5. DLT Template Processing Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                 DLT TEMPLATE PROCESSING FLOW                      │
└───────────────────────────────────────────────────────────────────┘

User uploads DLT Template File
           │
           ↓
┌──────────────────────────┐
│  FB-FILEUPLOAD           │
│  CampaignTemplateDlt     │
│  FilesSaver              │
└──────────┬───────────────┘
           │
           ↓ Create DLT request in DB
Database: dlt_template_request
           │
           ↓ Push to DltFileQ
┌──────────────────────────┐
│   DltFileQ (Redis)       │
└──────────┬───────────────┘
           │
           ↓ rpop() by Consumer
┌──────────────────────────┐
│  FB-DLTFILEPROCESSOR     │
│  DltFileQConsumer        │
│  - MasterFileSplitHandler│
│  - Read template file    │
│  - Process templates     │
│  - Store in DB           │
└──────────┬───────────────┘
           │
           ↓
Database: dlt_templates (Updated)
```

---

## 🔍 Detailed Module Descriptions

### 1. fb-fileupload
**Purpose**: HTTP endpoint for file uploads via web UI

**Key Classes**:
- `FilesSaver`: Main servlet handling multipart file uploads
- `CampaignTemplateFilesSaver`: Template-specific file uploads
- `CampaignTemplateDltFilesSaver`: DLT template file uploads
- `FileReadService`: Concurrent file reading and validation
- `XlsFileParser`, `XlsxFileParser`, `CsvReaderFileParser`: File parsers

**Workflow**:
1. Receives HTTP POST with multipart/form-data
2. Validates username and frompage parameters
3. Creates user-specific directory
4. Handles ZIP extraction if needed
5. Spawns concurrent tasks for file validation
6. Returns JSON response with file statistics
7. Tracks files in Redis for cleanup

**Key Features**:
- Supports CSV, XLS, XLSX, TXT, ZIP formats
- Concurrent file processing using FutureTask
- File tracking for cleanup (abandoned files)
- Mobile number validation during upload

---

### 2. fb-initialstage
**Purpose**: Polls database for new campaigns/groups and initializes processing

**Key Classes**:
- `CampaignMasterPoller`: Polls campaign_master table
- `CampaignGroupsPoller`: Polls for group campaigns
- `CampaignMasterDAO`: Database operations
- `FileSender`: Pushes campaigns to Redis queues

**Workflow**:
1. Polls DB every N seconds (configurable)
2. Fetches campaigns with status='QUEUED'
3. Checks retry count < maxRetryCount
4. Converts campaign to JSON
5. Pushes to FileSplitQ in Redis
6. Updates campaign status in DB
7. Sends heartbeat to monitoring

**Configuration**:
- Polling interval
- Max retry count
- Redis connection details

---

### 3. fb-splitstage
**Purpose**: Splits large campaign files into smaller chunks for parallel processing

**Key Classes**:
- `FileSplitQConsumer`: Consumes from FileSplitQ
- `MasterFileSplitHandler`: Core splitting logic
- `SplitStageServlet`: Initialization servlet

**Workflow**:
1. Consumes campaign JSON from FileSplitQ
2. Reads master file from filesystem
3. Splits file into chunks (configurable size)
4. Creates campaign_file_splits records in DB
5. Generates split files on filesystem
6. Pushes split file metadata to ExcludeQ or DeliveryQ
7. Updates campaign status

**Split Logic**:
- Chunk size: N records per file (e.g., 10,000)
- Maintains file format (CSV/TXT)
- Preserves delimiter settings
- Handles different campaign types (OTM, MTM, TEM, GROUP)

---

### 4. fb-excludeprocessor
**Purpose**: Filters excluded numbers from campaign files

**Key Classes**:
- `ExcludeConsumer`: Main consumer thread
- `FileDataExtractor`: Extracts and filters numbers
- `CampaignDAO`: Database operations

**Workflow**:
1. Consumes from ExcludeQ (campaign_id_exclude)
2. Reads split file
3. Fetches exclude_group_ids from campaign
4. Filters out excluded numbers
5. Creates two files:
   - Valid numbers file (for delivery)
   - Excluded numbers file (for tracking)
6. Updates DB with exclude count
7. Pushes valid file to DeliveryQ
8. Tracks excluded numbers

**Exclusion Sources**:
- Group-based exclusions
- DND (Do Not Disturb) lists
- Previous campaign exclusions

---

### 5. fb-handoverstage
**Purpose**: Final processing before handing over to delivery engine

**Key Classes**:
- `SplitFileConsumer`: Consumes from DeliveryQ
- `ProcessOTM`: One-to-Many campaign processing
- `ProcessMTM`: Many-to-Many campaign processing
- `ProcessTEM`: Template campaign processing

**Workflow**:
1. Uses rpoplpush() to safely consume from DeliveryQ
2. Maintains processing queue for failure recovery
3. Reads split file records
4. Processes based on campaign type:
   - **OTM**: Single message to multiple numbers
   - **MTM**: Multiple messages to multiple numbers
   - **TEM**: Template-based messages with substitutions
5. Sends to Platform Delivery Engine
6. Removes from processing queue on success
7. Handles retry logic on failure

**Reliability Features**:
- Atomic queue operations (rpoplpush)
- Processing queue for tracking
- Retry mechanism with backoff
- Failure tracking in DB

---

### 6. fb-groupsprocessor
**Purpose**: Processes group-based campaigns

**Key Classes**:
- `GroupsQConsumer`: Consumes from GroupQ
- `GroupsFileSplitQConsumer`: Splits group files
- `GroupsCampaignQConsumer`: Creates campaigns from groups
- `MasterFileSplitHandler`: Group file splitting

**Workflow**:
1. Consumes group request from GroupQ
2. Reads group master file
3. Splits into smaller files
4. Updates group_master status
5. When all group files complete:
   - Creates campaign_master records
   - Pushes to FileSplitQ
   - Group numbers become campaign targets

**Group Types**:
- Static groups (predefined numbers)
- Dynamic groups (query-based)
- Merged groups (multiple groups combined)

---

### 7. fb-dltfileprocessor
**Purpose**: Processes DLT (Distributed Ledger Technology) template files

**Key Classes**:
- `DltFileQConsumer`: Consumes from DltFileQ
- `MasterFileSplitHandler`: DLT file processing
- `DltTemplateRequestDAO`: Database operations

**Workflow**:
1. Consumes DLT file request from queue
2. Reads template file (CSV/XLSX)
3. Validates template structure
4. Parses template variables
5. Stores templates in dlt_templates table
6. Updates request status
7. Makes templates available for campaigns

**Template Structure**:
- Template ID
- Template content
- Variables/placeholders
- DLT entity information
- Approval status

---

### 8. fb-campaignfinisher
**Purpose**: Tracks campaign completion and cleanup

**Key Classes**:
- `PollerCampaignFilesCompleted`: Polls for completed split files
- `PollerCampaignMasterCompleted`: Polls for completed campaigns
- `QueryExecutor`: Executes update queries from queue
- `DQRedisCleaner`: Cleans up delivery queues

**Workflow**:
1. **File Completion Poller**:
   - Checks campaign_file_splits status
   - When all splits complete, marks campaign as complete
   - Aggregates statistics (sent, failed, etc.)

2. **Campaign Completion Poller**:
   - Checks campaign_master status
   - Updates final statistics
   - Triggers notifications
   - Archives campaign data

3. **Query Executor**:
   - Consumes SQL queries from UpdateSQLQueue
   - Executes batch updates
   - Handles failures

4. **DQ Cleaner**:
   - Removes empty campaign queues from Redis
   - Prevents memory leaks

---

### 9. fb-downloadhandler
**Purpose**: Converts CSV reports to Excel format

**Key Classes**:
- `CsvToExcelConvertionRequestConsumer`: Queue consumer
- `CsvToExcelConvertor`: Conversion logic
- `DownloadReqDAO`: Database operations

**Workflow**:
1. Consumes conversion request from queue
2. Reads CSV file
3. Converts to Excel format (.xlsx)
4. Handles large files (pagination if needed)
5. Stores Excel file
6. Updates download_request table
7. Notifies user (via email/notification)

**Features**:
- Handles large CSV files
- Row limit per Excel sheet (1M rows)
- Multiple sheet support
- Retry mechanism on failure

---

### 10. fb-cronjobs
**Purpose**: Scheduled maintenance tasks

**Key Classes**:
- `UnwantedFilesRemoval`: Removes old/abandoned files
- `CurrencyRatesUpdater`: Updates currency exchange rates

**Scheduled Jobs**:

**1. File Cleanup** (runs daily):
- Removes abandoned uploaded files (not used in campaigns)
- Removes campaign files older than N days
- Removes group files older than N days
- Removes DLT files older than N days
- Clears tracking queues in Redis

**2. Currency Rate Update** (runs periodically):
- Fetches latest exchange rates
- Updates wallet_currency_rates table
- Used for billing and reporting

---

### 11. fb-scheduleprocessor
**Purpose**: Handles scheduled campaigns

**Key Classes**:
- Schedule processing logic for future campaigns
- Time zone handling
- Campaign queue scheduling

**Workflow**:
1. Polls for campaigns with scheduled_time set
2. Waits until scheduled_time arrives
3. Pushes to FileSplitQ when time reaches
4. Handles time zone conversions
5. Supports recurring schedules

---

### 12. fb-inmemoryrefresh
**Purpose**: Refreshes in-memory caches

**Key Classes**:
- `InMemoryRefreshServlet`: HTTP endpoint for cache refresh

**Cached Data**:
- Configuration parameters
- User details
- DLT templates
- Route information
- Rate cards

**Refresh Triggers**:
- Manual via HTTP endpoint
- Scheduled refresh (cron)
- On-demand from admin UI

---

### 13. fb-utils
**Purpose**: Shared utilities across all modules

**Key Packages**:

**DTOs**:
- `FileDataBean`: File metadata
- `SplitFileData`: Split file information
- `Templates`: DLT template structure
- `QueueType`: Queue enumeration
- `RedisServerDetailsBean`: Redis connection details

**Singletons**:
- `ConfigParamsTon`: Configuration parameters cache
- `ConnectionFactoryForAccountsDB`: Account DB connections
- `ConnectionFactoryForCMDB`: Campaign DB connections
- `RedisConnectionTon`: Redis connection pool
- `GlobalPropertiesTon`: Global properties

**Utilities**:
- `Utility`: General utility methods
- `JsonUtility`: JSON parsing/generation
- `MobileValidator`: Mobile number validation
- `EmailValidator`: Email validation
- `HeartBeatMonitoring`: Heartbeat tracking
- `UploadedFilesTrackingUtility`: File tracking

---

### 14. fb-logger
**Purpose**: Centralized logging framework

**Key Classes**:
- `InitialStageLog`: InitialStage module logger
- `SplitStageLog`: SplitStage module logger
- `HandoverStageLog`: HandoverStage module logger
- `FileUploadLog`: FileUpload module logger
- Custom log formatters and handlers

**Logging Features**:
- Module-specific log files
- Log rotation (daily/size-based)
- Different log levels per module
- Structured logging (JSON format option)
- Correlation IDs for request tracking

---

## 🛠️ Technology Stack

### Core Technologies
- **Java**: 21 (JDK 21)
- **Build Tool**: Maven 3.x (Multi-module)
- **Application Server**: Jetty 9.4 / JBoss/WildFly (configurable)

### Frameworks & Libraries
- **Servlet API**: 4.0.1
- **Log4j**: 2.17.0 (Logging)
- **Apache Commons**: Multiple libraries
  - commons-configuration: 1.10
  - commons-dbcp2: 2.8.0 (Connection pooling)
  - commons-csv: 1.8
  - commons-validator: 1.6
  - commons-codec: 1.10

### Data Storage
- **Database**: MariaDB 2.3.0 (JDBC driver)
- **Cache/Queue**: Redis (Jedis 3.6.0 client)
- **Elasticsearch**: 7.12.0 (for analytics)

### Messaging
- **Kafka**: 2.8.0 (for event streaming)
- **Queue Pattern**: Redis Lists (RPOP/LPUSH)

### File Processing
- **Apache POI**: For Excel (XLS/XLSX) processing
- **OpenCSV**: For CSV parsing
- **Commons Compress**: For ZIP handling

### Monitoring
- **Prometheus**: Metrics collection (simpleclient 0.9.0)
- **Custom Heartbeat**: Module health monitoring

### Other Libraries
- **Jackson**: 2.12.1 (JSON processing)
- **Gson**: 2.8.8 (JSON serialization)
- **HTTP Client**: Apache HttpClient 4.5.13
- **Quartz**: 2.3.2 (Scheduling)
- **Drools**: 5.4.0 (Business rules engine)

---

## 🔄 Queue Architecture

### Redis Queue Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS QUEUE ARCHITECTURE                   │
└─────────────────────────────────────────────────────────────────┘

Work Queues (Task Distribution):
├── FileSplitQ                    → Campaign splitting tasks
├── GroupQ                        → Group processing tasks
├── DltFileQ                      → DLT template processing
├── ScheduleQ                     → Scheduled campaign tasks
├── GroupsFileSplitQ              → Group file splitting
├── GroupsCampaignQ               → Group-to-campaign conversion
└── CsvToExcelConversionQ         → Report conversion

Processing Queues (Per Campaign):
├── DeliveryQ:[campaign_id]       → Ready for delivery
├── ExcludeQ:[campaign_id]_exclude → Needs exclusion filtering
├── Processing:[campaign_id]      → Currently processing (rpoplpush)
└── DupCheck:[campaign_id]        → Duplicate tracking

Status/Update Queues:
├── StatsUpdateStatusQueryQ       → SQL update queries
├── ExcludeNumberFileQ            → Excluded numbers tracking
└── FileTrackingQ:[username]      → Uploaded files tracking

Monitoring Queues:
├── HeartBeat:[module]:[instance] → Module health status
└── PrometheusMetricsQ            → Metrics for Prometheus

Cache Keys:
├── config:params                 → Configuration cache
├── user:details:[user_id]        → User information
├── dlt:template:[template_id]    → DLT templates
└── retry:count:[file_id]         → Retry attempt tracking
```

### Queue Operations Pattern

**1. Simple Queue (FIFO)**:
```
Producer: LPUSH queue_name json_payload
Consumer: RPOP queue_name
```

**2. Reliable Queue (with processing tracking)**:
```
Consumer: RPOPLPUSH source_queue processing_queue
Process: [Do work]
On Success: LREM processing_queue json_payload
On Failure: [Retry or DLT]
```

**3. Per-Campaign Queue (hierarchical)**:
```
Structure:
  - DeliveryQ (main) → List of campaign_id keys
  - campaign_id_XXX → List of split file metadata

Consumer:
  1. RPOPLPUSH DeliveryQ DeliveryQ (get campaign_id)
  2. RPOPLPUSH campaign_id_XXX processing:campaign_id_XXX (get file)
  3. Process file
  4. LREM processing:campaign_id_XXX metadata (remove on success)
  5. Check if campaign_id_XXX is empty → LREM DeliveryQ campaign_id
```

---

## 🔐 Configuration Management

### Configuration Hierarchy

```
properties/
├── common/                       → Shared configs
│   ├── global.properties         → Global settings
│   ├── jndi.properties           → JNDI/Database
│   ├── common.properties         → Common params
│   └── [28 property files]
├── fileprocessor/                → Module-specific
│   └── [22 property files]
└── profile/                      → Environment-specific
    ├── *.properties_do1          → Dev Ocean 1
    ├── *.properties_do2          → Dev Ocean 2
    ├── *.properties_staging      → Staging
    └── *.properties_production   → Production
```

### Key Configuration Parameters

**Redis Configuration**:
- `redis.server.details`: Redis server IPs/ports
- `redis.pool.max.total`: Connection pool size
- `redis.timeout`: Connection timeout

**Queue Names**:
- `file.split.queue.name`: FileSplitQ
- `group.queue.name`: GroupQ
- `dlt.file.queue.name`: DltFileQ
- `stats.update.status.query.queue.name`: Update query queue

**Processing Parameters**:
- `file.split.chunk.size`: Records per split file
- `max.retry.count`: Maximum retry attempts
- `consumer.sleep.time`: Consumer polling interval
- `thread.sleep.time`: Thread restart delay

**File Paths**:
- `file.store.path`: Base file storage
- `campaigns.file.store.path`: Campaign files
- `group.file.store.path`: Group files

**Database**:
- JNDI data source names
- Connection pool settings
- Query timeout values

---

## 🚀 Deployment Architecture

### Docker Deployment

**Singleton Mode** (Single instance per module):
```
docker-compose-singleton.yml
├── fb-fileupload:8080
├── fb-initialstage:8081
├── fb-splitstage:8082
├── fb-excludeprocessor:8083
├── fb-handoverstage:8084
├── fb-groupsprocessor:8085
├── fb-dltfileprocessor:8086
├── fb-campaignfinisher:8087
├── fb-downloadhandler:8088
├── fb-cronjobs:8089
└── fb-inmemoryrefresh:8090
```

**Other Mode** (Multiple instances for high throughput):
```
docker-compose-other.yml
├── fb-splitstage (3 instances)
├── fb-excludeprocessor (2 instances)
├── fb-handoverstage (5 instances)
└── fb-groupsprocessor (2 instances)
```

### High Availability Features
- Multiple Redis instances (master-replica)
- Connection pooling
- Retry mechanisms
- Heartbeat monitoring
- Graceful degradation
- Circuit breaker pattern (for external services)

---

## 📈 Monitoring & Observability

### Metrics Collection
- **Prometheus Endpoints**: Each module exposes `/metrics`
- **Custom Metrics**:
  - Queue lengths
  - Processing time
  - Success/failure rates
  - File processing statistics
  - Database connection pool stats

### Logging
- **Module-specific logs**: Each module has its own logger
- **Log levels**: DEBUG, INFO, WARN, ERROR
- **Log rotation**: Daily or size-based (100MB)
- **Structured logging**: JSON format for easy parsing

### Health Checks
- **Heartbeat System**:
  - Each consumer thread sends heartbeat every N seconds
  - Stored in Redis with timestamp
  - Monitoring system checks for stale heartbeats
  - Alerts on missing heartbeats

---

## 🔄 Error Handling & Reliability

### Retry Mechanism
1. **Automatic Retry**: Failed tasks retry up to `max.retry.count`
2. **Exponential Backoff**: Increasing delays between retries
3. **Dead Letter Queue**: After max retries, move to DLT
4. **Database Tracking**: All retry attempts logged in DB

### Failure Recovery
- **Processing Queue**: rpoplpush ensures no message loss
- **Status Tracking**: DB maintains current status
- **Idempotency**: Operations can be safely retried
- **Cleanup Jobs**: Removes stale entries

### Data Integrity
- **Atomic Operations**: Redis transactions for critical ops
- **Database Transactions**: ACID compliance
- **File Locking**: Prevents concurrent file access
- **Checksum Validation**: Ensures file integrity

---

## 📊 Performance Optimization

### Concurrency
- **Multi-threaded Consumers**: N consumers per queue
- **Async File Processing**: FutureTask for parallel ops
- **Connection Pooling**: DB and Redis connections
- **Batch Operations**: Bulk inserts/updates

### Caching
- **In-Memory Cache**: Frequently accessed config
- **Redis Cache**: User details, templates, routes
- **Cache Invalidation**: Manual + scheduled refresh

### File Handling
- **Streaming**: Large files processed in streams
- **Chunking**: Files split for parallel processing
- **Compression**: ZIP support reduces storage
- **Cleanup**: Automated old file removal

---

## 🎯 Key Design Patterns

1. **Producer-Consumer**: Queue-based task distribution
2. **Singleton**: Shared resources (connections, config)
3. **Factory**: Connection factories for DB/Redis
4. **Strategy**: Different processors for OTM/MTM/TEM
5. **Template Method**: Common processing skeleton
6. **Observer**: Heartbeat monitoring
7. **Circuit Breaker**: External service calls
8. **Retry**: Resilient operations

---

## 📝 Data Model (Key Tables)

```sql
-- Campaign management
campaign_master (cm_id, username, file_location, status, ...)
campaign_file_splits (c_f_s_id, cm_id, file_location, status, ...)

-- Group management
group_master (g_id, username, file_location, status, ...)
group_files (g_f_id, g_id, file_location, status, ...)

-- DLT templates
dlt_template_request (req_id, username, file_location, status, ...)
dlt_templates (template_id, content, variables, ...)

-- Tracking
download_request (req_id, csv_file, excel_file, status, ...)
unprocess_numbers (number, campaign_id, reason, ...)
```

---

## 🔒 Security Considerations

1. **File Upload Validation**: Whitelist file types, size limits
2. **SQL Injection Prevention**: Prepared statements
3. **Access Control**: User-based file isolation
4. **Data Encryption**: Sensitive data encrypted at rest
5. **Secure Connections**: SSL/TLS for Redis/DB
6. **Input Sanitization**: Validate all user inputs
7. **Rate Limiting**: Prevent abuse of APIs

---

## 🚦 Campaign Lifecycle States

```
Campaign States:
QUEUED → PROCESSING → SPLITTING → EXCLUDING → 
DELIVERY_QUEUED → DELIVERING → COMPLETED / FAILED

Group States:
QUEUED → PROCESSING → SPLITTING → COMPLETED / FAILED

Split File States:
PENDING → EXCLUDING → QUEUED → DELIVERING → 
COMPLETED / FAILED / PARTIAL

DLT Request States:
QUEUED → PROCESSING → COMPLETED / FAILED
```

---

## 📚 API Endpoints

### File Upload
- `POST /save`: Upload campaign files
- `POST /template-save`: Upload template files
- `POST /dlt-save`: Upload DLT template files

### Cache Management
- `GET /refresh`: Refresh in-memory cache
- `POST /refresh-config`: Reload configuration

### Monitoring
- `GET /metrics`: Prometheus metrics
- `GET /health`: Health check endpoint

---

## 🎓 Best Practices Implemented

1. **Separation of Concerns**: Each module has single responsibility
2. **Fail-Fast**: Early validation and error detection
3. **Graceful Degradation**: System continues with reduced functionality
4. **Observability**: Comprehensive logging and metrics
5. **Scalability**: Horizontal scaling via multiple instances
6. **Maintainability**: Clear module boundaries and documentation
7. **Testability**: Dependency injection for easier testing
8. **Configuration Management**: Externalized configuration

---

## 🔧 Development Workflow

1. **Build**: `mvn clean install` (builds all modules)
2. **Run Locally**: Deploy individual WARs to Jetty/Tomcat
3. **Docker Build**: `docker build -f Dockerfile_singleton`
4. **Deploy**: `docker-compose up -d`
5. **Monitor**: Check logs in `/logs` directory
6. **Troubleshoot**: Check Redis queues, DB status, logs

---

## 📞 Module Communication Summary

```
HTTP → fb-fileupload
DB Poll → fb-initialstage
Redis Q → fb-splitstage
Redis Q → fb-groupsprocessor
Redis Q → fb-dltfileprocessor
Redis Q → fb-excludeprocessor
Redis Q → fb-handoverstage → External Platform
DB Poll → fb-campaignfinisher
Redis Q → fb-downloadhandler
Scheduled → fb-cronjobs
HTTP → fb-inmemoryrefresh
```

---

## 🎯 Critical Success Factors

1. **Redis Availability**: System depends heavily on Redis
2. **Database Performance**: Campaign status tracking
3. **File Storage**: Adequate disk space for files
4. **Network Bandwidth**: File transfers between modules
5. **Monitoring**: Early detection of issues
6. **Configuration**: Proper tuning of queue sizes, retries
7. **Scaling**: Adding consumers based on load

---

This architecture provides a robust, scalable, and maintainable solution for processing large-scale SMS campaigns with various campaign types, exclusion rules, and delivery requirements. The queue-based design ensures loose coupling between modules and enables horizontal scaling for high throughput scenarios.
