# Beacon File Processor - System Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Module Details](#module-details)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)
6. [Detailed Component Description](#detailed-component-description)

---

## System Overview

**Beacon File Processor** is a comprehensive, multi-module Java-based messaging/SMS campaign file processing system designed to handle large-scale file uploads, processing, and delivery orchestration for SMS/messaging campaigns.

### Primary Purpose
- Process uploaded campaign files (CSV, XLS, XLSX, ZIP formats)
- Split large files into manageable chunks
- Validate and enrich data with templates
- Hand over processed data to delivery engines via Kafka
- Track campaign completion and cleanup

### Key Characteristics
- **Multi-tenant**: Supports multiple users/clients
- **Distributed**: Uses Redis for queueing and coordination
- **Scalable**: Designed with consumer-producer patterns
- **Resilient**: Built-in retry mechanisms and error handling
- **Asynchronous**: Event-driven architecture with queues

---

## Architecture Diagram

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BEACON FILE PROCESSOR SYSTEM                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   External   │         │   External   │         │   External   │
│   HTTP API   │────────▶│  File Upload │────────▶│  Storage     │
│   Clients    │         │  (WAR)       │         │  (File Sys)  │
└──────────────┘         └──────────────┘         └──────────────┘
                                │
                                │ Metadata
                                ▼
                    ┌───────────────────────┐
                    │   MariaDB Database    │◀──────────┐
                    │  - Campaigns          │           │
                    │  - Files Metadata     │           │
                    │  - Templates          │           │
                    │  - Groups             │           │
                    └───────────────────────┘           │
                                │                       │
                                │ Polls                 │ Updates
                                ▼                       │
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PROCESSING PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐              │
│  │   Initial    │─────▶│    Split     │─────▶│   Handover   │              │
│  │    Stage     │      │    Stage     │      │    Stage     │              │
│  │  (Polling)   │      │  (Split Qs)  │      │  (Delivery)  │              │
│  └──────────────┘      └──────────────┘      └──────────────┘              │
│         │                      │                      │                      │
│         │                      │                      │                      │
│         ▼                      ▼                      ▼                      │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │                    Redis Queue System                        │            │
│  │  - FileSplitQ (initial to split)                            │            │
│  │  - DeliveryQ_{type} (split to handover)                     │            │
│  │  - Campaign Queues                                           │            │
│  │  - Processing Queues                                         │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ Hands over to
                                ▼
                    ┌───────────────────────┐
                    │   Kafka Topics        │
                    │  - Delivery Engine    │
                    │  - Message Processing │
                    └───────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  SMS/Messaging        │
                    │  Delivery System      │
                    └───────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        SUPPORTING MODULES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Groups    │  │   Campaign   │  │     DLT      │  │   Exclude    │     │
│  │ Processor  │  │   Finisher   │  │  Processor   │  │  Processor   │     │
│  └────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
│       │                  │                  │                  │             │
│       └──────────────────┴──────────────────┴──────────────────┘            │
│                              Support                                         │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Details

### Core Processing Modules (WAR Deployments)

#### 1. **fb-fileupload** 
**Purpose**: Entry point for file uploads
- **Type**: Web Application (WAR)
- **Key Components**:
  - `FilesSaver` servlet: Handles multipart file uploads
  - Supports CSV, XLS, XLSX, ZIP formats
  - Validates file structure and counts records
  - Stores files with UUID-based naming
- **Flow**:
  1. Receives file via HTTP POST
  2. Validates and stores file
  3. Extracts ZIP if needed
  4. Counts total records
  5. Tracks files in Redis for cleanup
- **Dependencies**: fb-utils

#### 2. **fb-initialstage**
**Purpose**: Polls database for new campaigns and initiates processing
- **Type**: Web Application (WAR)
- **Key Components**:
  - `CampaignMasterPoller`: Polls `campaign_master` table
  - `GroupCampaignDAO`: Database operations
  - `FileSender`: Pushes metadata to Redis queues
- **Flow**:
  1. Continuously polls database for campaigns in "ready" state
  2. Fetches campaign metadata
  3. Pushes to `FileSplitQ` in Redis
  4. Updates campaign status
- **Dependencies**: fb-utils

#### 3. **fb-splitstage**
**Purpose**: Splits large campaign files into smaller chunks for parallel processing
- **Type**: Web Application (WAR)
- **Key Components**:
  - `FileSplitQConsumer`: Consumes from FileSplitQ
  - `MasterFileSplitHandler`: Core splitting logic
  - `FileParser`: Various format parsers
- **Flow**:
  1. Consumes campaign metadata from FileSplitQ
  2. Reads master file
  3. Splits into configurable chunk sizes (e.g., 100K records each)
  4. Creates split file metadata entries in DB
  5. Pushes split metadata to DeliveryQ_{cm_id}
- **Dependencies**: fb-utils, fb-fileparser
- **Configuration**: Split size, file paths

#### 4. **fb-handoverstage**
**Purpose**: Final processing stage that hands over data to delivery engines
- **Type**: Web Application (WAR)
- **Key Components**:
  - `SplitFileConsumer`: Consumes from delivery queues
  - `ProcessOTM`: One-to-Many message processing
  - `ProcessMTM`: Many-to-Many message processing
  - `ProcessTEM`: Template-based message processing
  - `KafkaHandover`: Sends to Kafka
- **Flow**:
  1. Consumes split file metadata from DeliveryQ
  2. Reads split file
  3. Applies business rules (templates, DLT, excludes)
  4. Validates mobile numbers
  5. Sends to Kafka for delivery
  6. Removes from processing queue
- **Campaign Types**:
  - **OTM** (One-to-Many): Single message to multiple recipients
  - **MTM** (Many-to-Many): Different messages per recipient
  - **TEM** (Template): Template-based personalized messages
  - **GROUP**: Group-based campaigns
  - **QUICK**: Quick send campaigns
- **Dependencies**: fb-utils

#### 5. **fb-campaignfinisher**
**Purpose**: Monitors and finalizes completed campaigns
- **Type**: Web Application (WAR)
- **Key Components**:
  - `PollerCampaignFilesCompleted`: Tracks file completion
  - `PollerCampaignMasterCompleted`: Tracks campaign completion
  - `DQRedisCleaner`: Cleans up Redis queues
  - `QueryExecutor`: Database updates
- **Flow**:
  1. Polls for completed split files
  2. Aggregates completion status
  3. Marks campaigns as completed
  4. Cleans up Redis queues
  5. Updates statistics
- **Dependencies**: fb-utils

---

### Supporting Modules

#### 6. **fb-groupsprocessor**
**Purpose**: Handles group-based campaigns where groups have predefined members
- **Key Functions**:
  - Polls for group campaigns
  - Generates campaign files from group members
  - Splits group files
  - Processes group metadata
- **Dependencies**: fb-utils, fb-fileparser

#### 7. **fb-dltfileprocessor**
**Purpose**: Processes DLT (Do Not Disturb List/Telecom Templates) template files
- **Key Functions**:
  - Uploads DLT template files
  - Validates template formats
  - Maps templates to campaigns
  - Stores template configurations
- **Dependencies**: fb-utils

#### 8. **fb-excludeprocessor**
**Purpose**: Manages exclusion lists (numbers to be excluded from campaigns)
- **Key Functions**:
  - Uploads exclusion lists
  - Validates excluded numbers
  - Integrates with campaign processing
- **Dependencies**: fb-utils

#### 9. **fb-downloadhandler**
**Purpose**: Handles download requests for reports and processed files
- **Dependencies**: fb-utils

#### 10. **fb-scheduleprocessor**
**Purpose**: Manages scheduled campaigns
- **Key Functions**:
  - Polls for scheduled campaigns
  - Triggers campaigns at scheduled time
  - Manages recurring schedules
- **Dependencies**: fb-utils

#### 11. **fb-cronjobs**
**Purpose**: Scheduled maintenance and cleanup tasks
- **Functions**:
  - File cleanup
  - Redis cleanup
  - Log rotation
  - Statistics aggregation
- **Dependencies**: fb-utils

---

### Utility and Library Modules

#### 12. **fb-utils**
**Purpose**: Core shared utilities used by all modules
- **Components**:
  - Database connection factories (MariaDB)
  - Redis connection management
  - DTOs (Data Transfer Objects)
  - Configuration management
  - Common utilities (date, validation, JSON)
  - Heartbeat monitoring
  - Kafka queue management
- **Key Classes**:
  - `ConfigParamsTon`: Configuration singleton
  - `RedisConnectionFactory`: Redis connection pool
  - `ConnectionFactoryForCMDB`: Campaign master DB
  - `Utility`: Common utility functions
  - `HeartBeatMonitoring`: Health monitoring
  - `MobileValidator`: Mobile number validation

#### 13. **fb-fileparser**
**Purpose**: File parsing library for various formats
- **Parsers**:
  - CSV (with Unicode support)
  - XLS (Excel 97-2003)
  - XLSX (Excel 2007+)
  - TXT
  - ZIP (extraction and parsing)
- **Features**:
  - Streaming for large files
  - Line-by-line processing
  - Format detection
  - Error handling

#### 14. **fb-logger**
**Purpose**: Centralized logging framework
- **Loggers**:
  - FileUploadLog
  - InitialStageLog
  - SplitStageLog
  - HandoverStageLog
  - GroupProcessorLog
  - DLTFileLog
- **Features**:
  - Custom formatters
  - Separate log files per module
  - Log rotation support

#### 15. **fb-inmemoryrefresh**
**Purpose**: In-memory cache refresh for configuration and lookups
- **Functions**:
  - Refreshes Redis cache
  - Updates configuration
  - Hot-reload capabilities

#### 16. **beaconlib**
**Purpose**: Standalone library/JAR with common functionality
- **Dependencies**: fb-utils, fb-fileparser
- **Build**: Shaded JAR with all dependencies

---

## Data Flow

### Detailed Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DETAILED DATA FLOW DIAGRAM                            │
└─────────────────────────────────────────────────────────────────────────────┘

                    USER / EXTERNAL SYSTEM
                            │
                            │ 1. Upload File (POST /save)
                            ▼
                ┌───────────────────────────┐
                │   fb-fileupload           │
                │   (FilesSaver Servlet)    │
                └───────────────────────────┘
                            │
                            │ 2. Validate & Store File
                            │    - Extract ZIP if needed
                            │    - Count records
                            │    - Generate UUID
                            ▼
                ┌───────────────────────────┐
                │   File System Storage     │
                │   /path/username/         │
                │   file_UUID.csv           │
                └───────────────────────────┘
                            │
                            │ 3. Store metadata
                            ▼
                ┌───────────────────────────┐
                │   MariaDB                 │
                │   - campaign_master       │
                │   - campaign_files        │
                │   status = 'UPLOADED'     │
                └───────────────────────────┘
                            │
                            │ 4. Poll for campaigns (status='ready to process')
                            ▼
                ┌───────────────────────────┐
                │   fb-initialstage         │
                │   CampaignMasterPoller    │
                └───────────────────────────┘
                            │
                            │ 5. Push to Redis
                            ▼
                ┌───────────────────────────┐
                │   Redis Queue             │
                │   FileSplitQ              │
                │   {cm_id, file_path, ...} │
                └───────────────────────────┘
                            │
                            │ 6. Consume from FileSplitQ
                            ▼
                ┌───────────────────────────┐
                │   fb-splitstage           │
                │   FileSplitQConsumer      │
                └───────────────────────────┘
                            │
                            │ 7. Split file logic
                            ▼
                ┌───────────────────────────────────────┐
                │   MasterFileSplitHandler              │
                │   - Read master file                  │
                │   - Parse records                     │
                │   - Split into chunks (100K each)     │
                │   - Write split files                 │
                │   - Create DB entries                 │
                └───────────────────────────────────────┘
                            │
                            │ 8. For each split file
                            ▼
                ┌───────────────────────────┐
                │   MariaDB                 │
                │   campaign_file_splits    │
                │   - c_f_s_id              │
                │   - file_path             │
                │   - total_count           │
                │   - status='queued'       │
                └───────────────────────────┘
                            │
                            │ 9. Push split metadata
                            ▼
                ┌───────────────────────────┐
                │   Redis Queue             │
                │   DeliveryQ_{cm_id}       │
                │   {c_f_s_id, ...}         │
                └───────────────────────────┘
                            │
                            │ 10. Consume from DeliveryQ
                            ▼
                ┌───────────────────────────┐
                │   fb-handoverstage        │
                │   SplitFileConsumer       │
                └───────────────────────────┘
                            │
                            │ 11. Process based on type
                            ▼
        ┌───────────────────┴───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ ProcessOTM   │    │ ProcessMTM   │    │ ProcessTEM   │
│ (One-to-Many)│    │(Many-to-Many)│    │ (Template)   │
└──────────────┘    └──────────────┘    └──────────────┘
        │                   │                   │
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            │ 12. Read split file & process
                            │     - Apply templates
                            │     - Validate mobiles
                            │     - Apply DLT rules
                            │     - Check exclusions
                            ▼
                ┌───────────────────────────┐
                │   KafkaHandover           │
                │   - Format message        │
                │   - Build Kafka payload   │
                │   - Send to Kafka         │
                └───────────────────────────┘
                            │
                            │ 13. Send to delivery
                            ▼
                ┌───────────────────────────┐
                │   Kafka Topic             │
                │   delivery-engine-topic   │
                └───────────────────────────┘
                            │
                            │ 14. Update status
                            ▼
                ┌───────────────────────────┐
                │   MariaDB                 │
                │   campaign_file_splits    │
                │   status = 'COMPLETED'    │
                └───────────────────────────┘
                            │
                            │ 15. Monitor completion
                            ▼
                ┌───────────────────────────┐
                │   fb-campaignfinisher     │
                │   - Poll completion       │
                │   - Aggregate stats       │
                │   - Mark campaign done    │
                │   - Clean Redis queues    │
                └───────────────────────────┘
                            │
                            │ 16. Final status
                            ▼
                ┌───────────────────────────┐
                │   MariaDB                 │
                │   campaign_master         │
                │   status = 'COMPLETED'    │
                │   total_sent = count      │
                └───────────────────────────┘
```

---

## Technology Stack

### Languages & Frameworks
- **Java 21**: Primary language
- **Maven**: Build tool and dependency management
- **Java Servlet API 4.0**: Web layer

### Databases
- **MariaDB**: Primary relational database
  - Campaign metadata
  - File tracking
  - Templates
  - Groups
  - Statistics

### Caching & Queuing
- **Redis**: 
  - Queue management (FileSplitQ, DeliveryQs)
  - Campaign queues
  - Processing queues
  - File tracking
  - Heartbeat monitoring
  - Configuration caching
- **Jedis 3.6.0**: Redis Java client

### Messaging
- **Kafka 2.8.0**:
  - Message delivery handover
  - Event streaming
  - Inter-service communication

### File Processing
- **Apache Commons CSV 1.8**: CSV parsing
- **Apache POI** (via dependencies): Excel parsing
- **Commons IO**: File operations

### Utilities
- **Apache Commons Configuration 1.10**: Configuration management
- **Apache Commons Lang**: String utilities
- **Apache Commons Validator 1.6**: Validation
- **Apache Commons DBCP2 2.8.0**: Database connection pooling
- **Log4j2 2.17.0**: Logging
- **Jackson 2.12.1**: JSON processing
- **Gson 2.8.8**: JSON serialization
- **Quartz 2.3.2**: Job scheduling
- **HttpClient 4.5.13**: HTTP communication

### Build & Deployment
- **Maven Shade Plugin**: Fat JAR creation
- **Maven WAR Plugin**: WAR packaging
- **Docker**: Containerization (docker-fileprocessor module)
- **WildFly/JBoss**: Application server (inferred from log paths)

---

## Detailed Component Description

### Queue System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    REDIS QUEUE ARCHITECTURE                       │
└──────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────┐
                    │   FileSplitQ        │ ◄─── Initial Stage (Producer)
                    │   (Global Queue)    │
                    └──────────┬──────────┘
                               │
                               │ Campaign Metadata
                               │
                    ┌──────────▼──────────┐
                    │   Split Stage       │
                    │   (Consumer)        │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
    ┌───────────▼─────────┐       ┌──────────▼────────┐
    │ DeliveryQ_{cm_id_1} │       │ DeliveryQ_{cm_id_2}│
    │  Campaign 1 Queue   │       │  Campaign 2 Queue  │
    └───────────┬─────────┘       └──────────┬─────────┘
                │                            │
                │                            │
                │ ┌──────────────────────┐   │
                └─►  {cm_id}:processing  │◄──┘
                  │  Temp Queue          │
                  └──────────┬───────────┘
                             │
                             │ Split File Metadata
                             │
                  ┌──────────▼───────────┐
                  │   Handover Stage     │
                  │   (Consumer)         │
                  └──────────┬───────────┘
                             │
                             │ To Kafka
                             ▼
                  ┌──────────────────────┐
                  │   Delivery Engine    │
                  └──────────────────────┘

Additional Queues:
┌──────────────────────────────────────────┐
│  SQLUpdateQ: Database update queries     │
│  GroupsQ: Group processing queue         │
│  DltQ: DLT processing queue              │
│  ScheduleQ: Scheduled campaigns queue    │
│  tracking:{type}:{username}: File track  │
└──────────────────────────────────────────┘
```

### Campaign Types and Processing

```
┌──────────────────────────────────────────────────────────────────┐
│                      CAMPAIGN TYPE PROCESSING                     │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  OTM (One-to-Many) / QUICK / GROUP                              │
├─────────────────────────────────────────────────────────────────┤
│  Input File Format:                                             │
│    mobile_number                                                │
│    9876543210                                                   │
│    9876543211                                                   │
│                                                                  │
│  Campaign Config:                                               │
│    - Single message text                                        │
│    - Sender ID                                                  │
│    - DLT template ID                                            │
│                                                                  │
│  Processing:                                                    │
│    1. Read mobile numbers from file                             │
│    2. Validate each mobile                                      │
│    3. Apply same message to all                                 │
│    4. Send to Kafka                                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  MTM (Many-to-Many)                                             │
├─────────────────────────────────────────────────────────────────┤
│  Input File Format:                                             │
│    mobile_number,message                                        │
│    9876543210,Hello John your balance is 100                    │
│    9876543211,Hello Jane your balance is 200                    │
│                                                                  │
│  Campaign Config:                                               │
│    - Sender ID                                                  │
│    - DLT template ID (optional)                                 │
│                                                                  │
│  Processing:                                                    │
│    1. Read mobile and message from each line                    │
│    2. Validate mobile                                           │
│    3. Use line-specific message                                 │
│    4. Send to Kafka                                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  TEM (Template-based)                                           │
├─────────────────────────────────────────────────────────────────┤
│  Input File Format:                                             │
│    mobile_number,name,balance,due_date                          │
│    9876543210,John,100,2024-01-15                               │
│    9876543211,Jane,200,2024-01-20                               │
│                                                                  │
│  Campaign Config:                                               │
│    - Template: "Hello {name} your balance is {balance}"         │
│    - Column mapping                                             │
│    - Sender ID                                                  │
│    - DLT template ID                                            │
│                                                                  │
│  Processing:                                                    │
│    1. Read data columns from file                               │
│    2. Apply template with variable substitution                 │
│    3. Validate mobile                                           │
│    4. Generate personalized message                             │
│    5. Send to Kafka                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Error Handling and Retry Logic

```
┌──────────────────────────────────────────────────────────────────┐
│                    ERROR HANDLING FLOW                            │
└──────────────────────────────────────────────────────────────────┘

                Processing Stage (any module)
                            │
                            │
                ┌───────────▼──────────┐
                │  Exception Occurs    │
                └───────────┬──────────┘
                            │
                ┌───────────▼──────────┐
                │ Get retry count      │
                │ from Redis           │
                └───────────┬──────────┘
                            │
                    ┌───────┴────────┐
                    │                │
         ┌──────────▼──────┐    ┌───▼────────────┐
         │ retry < MAX     │    │ retry >= MAX   │
         │ (default: 5)    │    │ (default: 5)   │
         └──────────┬──────┘    └───┬────────────┘
                    │               │
         ┌──────────▼──────┐        │
         │ Increment retry │        │
         │ counter         │        │
         └──────────┬──────┘        │
                    │               │
         ┌──────────▼──────┐        │
         │ Re-push to      │        │
         │ same queue      │        │
         └─────────────────┘        │
                                    │
                         ┌──────────▼──────────┐
                         │ Mark as FAILED      │
                         │ in database         │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │ Remove from         │
                         │ processing queue    │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │ Log error details   │
                         │ Notify admins       │
                         └─────────────────────┘

Thread Sleep on Error:
┌────────────────────────────────────────────────────────┐
│ Exception → Sleep (configurable, default 30s)         │
│           → Restart consumer thread                    │
│           → Continue polling                           │
└────────────────────────────────────────────────────────┘
```

### Monitoring and Health

```
┌──────────────────────────────────────────────────────────────────┐
│                    MONITORING ARCHITECTURE                        │
└──────────────────────────────────────────────────────────────────┘

Each Consumer Thread:
    │
    │ Every iteration
    ▼
┌─────────────────────────────┐
│ HeartBeatMonitoring         │
│ - Module name               │
│ - Consumer type             │
│ - Instance ID               │
│ - Thread name               │
│ - Timestamp                 │
└─────────────┬───────────────┘
              │
              │ Push to Redis
              ▼
┌─────────────────────────────┐
│ Redis HeartBeat Keys        │
│ FP-{Module}:{Consumer}:     │
│   {instance}:{thread}       │
│ TTL: configured             │
└─────────────┬───────────────┘
              │
              │ Monitor checks
              ▼
┌─────────────────────────────┐
│ External Monitoring         │
│ - Prometheus (port 1075)    │
│ - Custom health checks      │
│ - Alert on missing HB       │
└─────────────────────────────┘

Metrics Tracked:
┌────────────────────────────────────────────┐
│ - Consumer thread status                   │
│ - Queue depths (Redis)                     │
│ - Processing rates                         │
│ - Error rates                              │
│ - File processing times                    │
│ - Database connection pool status          │
│ - Campaign completion rates                │
└────────────────────────────────────────────┘
```

### Database Schema (Key Tables)

```
┌──────────────────────────────────────────────────────────────────┐
│                      KEY DATABASE TABLES                          │
└──────────────────────────────────────────────────────────────────┘

campaign_master
├─ cm_id (PK)                   Campaign ID
├─ cli_id                       Client ID
├─ username                     User who created
├─ c_name                       Campaign name
├─ c_type                       Campaign type (OTM/MTM/TEM/GROUP)
├─ c_lang_type                  Language (English/Unicode)
├─ sender_id                    Sender ID
├─ message                      Message text
├─ total                        Total count
├─ exclude_group_ids            Exclusion groups
├─ dlt_entity_id                DLT entity
├─ dlt_template_id              DLT template
├─ status                       Status (uploaded/processing/completed)
├─ scheduled_time               Schedule time
├─ created_ts                   Created timestamp
└─ completed_ts                 Completion timestamp

campaign_files
├─ c_f_id (PK)                  File ID
├─ cm_id (FK)                   Campaign ID
├─ filename                     Original filename
├─ fileloc                      File storage location
├─ total                        Total records
├─ status                       Status
├─ retry_count                  Retry attempts
└─ created_ts                   Created timestamp

campaign_file_splits
├─ c_f_s_id (PK)                Split file ID
├─ cm_id (FK)                   Campaign ID
├─ c_f_id (FK)                  Parent file ID
├─ filename                     Split filename
├─ fileloc                      Split file location
├─ total                        Records in split
├─ status                       Status (queued/processing/completed)
├─ retry_count                  Retry attempts
├─ redis_id                     Redis server ID
├─ sent_count                   Successfully sent
├─ failed_count                 Failed count
└─ created_ts                   Created timestamp

group_master
├─ g_id (PK)                    Group ID
├─ cli_id                       Client ID
├─ g_name                       Group name
├─ total                        Total members
├─ status                       Status
└─ created_ts                   Created timestamp

group_files
├─ g_f_id (PK)                  Group file ID
├─ g_id (FK)                    Group ID
├─ filename                     Filename
├─ fileloc                      File location
├─ total                        Total records
└─ status                       Status

dlt_template_master
├─ dlt_temp_id (PK)             Template ID
├─ cli_id                       Client ID
├─ dlt_entity_id                Entity ID
├─ dlt_template_id              Template ID
├─ template_content             Template text
├─ template_type                Type
└─ status                       Status

exclude_group_master
├─ e_g_id (PK)                  Exclude group ID
├─ cli_id                       Client ID
├─ e_g_name                     Group name
├─ total                        Total excludes
└─ status                       Status

config_params (Configuration)
├─ key                          Config key
└─ value                        Config value
```

### Configuration Management

```
┌──────────────────────────────────────────────────────────────────┐
│                   CONFIGURATION ARCHITECTURE                      │
└──────────────────────────────────────────────────────────────────┘

Application Startup
    │
    ├─► Load global.properties
    │   ├─ Database connections
    │   ├─ Redis connections
    │   ├─ Kafka settings
    │   └─ Module-specific refs
    │
    ├─► Load module.properties
    │   ├─ Module-specific settings
    │   ├─ Queue names
    │   ├─ Thread counts
    │   └─ File paths
    │
    ├─► Load from Database
    │   ├─ ConfigParamsTon reads config_params table
    │   ├─ Cache in memory
    │   └─ Refresh periodically
    │
    └─► Load from Redis
        ├─ Runtime configurations
        ├─ Dynamic settings
        └─ Feature flags

Key Configuration Areas:
┌────────────────────────────────────────────────────────┐
│ FILE_STORE_PATH:          /opt/files/campaigns/       │
│ CAMPAIGNS_FILE_STORE_PATH: /opt/files/campaigns/      │
│ GROUP_FILE_STORE_PATH:     /opt/files/groups/         │
│ SPLIT_FILE_PATH:           /opt/files/splits/         │
│ FILE_SPLIT_QUEUE_NAME:     FileSplitQ                 │
│ MAX_RETRY_COUNT:           5                          │
│ SPLIT_FILE_SIZE:           100000                     │
│ CONSUMER_SLEEP_TIME:       1000                       │
│ THREAD_SLEEP_TIME:         30000                      │
│ LINE_BREAK_REPLACER:       <br>                       │
│ SPLIT_FILE_DELIMITER:      ~                          │
└────────────────────────────────────────────────────────┘
```

### Deployment Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT ARCHITECTURE                        │
└──────────────────────────────────────────────────────────────────┘

                    ┌───────────────────────┐
                    │   Load Balancer       │
                    │   (Nginx/HAProxy)     │
                    └───────────┬───────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
        ┌───────────▼─────────┐ ┌──────────▼──────────┐
        │ WildFly Instance 1  │ │ WildFly Instance 2  │
        │ (Application Server)│ │ (Application Server)│
        ├─────────────────────┤ ├─────────────────────┤
        │ fb-fileupload.war   │ │ fb-fileupload.war   │
        │ fb-initialstage.war │ │ fb-splitstage.war   │
        │ fb-handoverstage.war│ │ fb-campaignfinisher │
        │ ...                 │ │ ...                 │
        └───────────┬─────────┘ └──────────┬──────────┘
                    │                       │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Shared Resources    │
                    ├───────────────────────┤
                    │ - fb-utils.jar        │
                    │ - fb-fileparser.jar   │
                    │ - fb-logger.jar       │
                    │ - Third-party libs    │
                    └───────────┬───────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
    ┌───────────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
    │ MariaDB Cluster  │ │Redis Cluster│ │Kafka Cluster│
    │ - Master         │ │- Master     │ │- Broker 1   │
    │ - Slaves         │ │- Slaves     │ │- Broker 2   │
    │ - Replication    │ │- Sentinel   │ │- Broker 3   │
    └──────────────────┘ └─────────────┘ └─────────────┘

                    ┌───────────────────────┐
                    │ Shared File System    │
                    │ (NFS/GlusterFS)       │
                    │ - Campaign files      │
                    │ - Split files         │
                    │ - Logs                │
                    └───────────────────────┘
```

---

## Key Design Patterns

### 1. **Producer-Consumer Pattern**
- Producers push to Redis queues
- Multiple consumers poll and process
- Decouples components
- Enables horizontal scaling

### 2. **Polling Pattern**
- Database polling for new campaigns
- Avoids webhook/callback complexity
- Simple and reliable
- Configurable polling intervals

### 3. **Splitting Pattern**
- Large files split into manageable chunks
- Enables parallel processing
- Better error isolation
- Improved throughput

### 4. **Singleton Pattern**
- Connection factories
- Configuration managers
- Ensures single instance
- Resource optimization

### 5. **Factory Pattern**
- File parser factory (CSV/XLS/XLSX)
- Creates appropriate parser based on file type
- Extensible design

### 6. **Retry Pattern**
- Built-in retry mechanism
- Exponential backoff (via sleep)
- Max retry limit
- Failure handling

---

## Performance Considerations

### Scalability
1. **Horizontal Scaling**:
   - Deploy multiple instances of each WAR
   - Redis-based coordination prevents conflicts
   - Stateless design enables easy scaling

2. **File Splitting**:
   - Configurable split size (default 100K)
   - Parallel processing of splits
   - Reduces memory footprint

3. **Consumer Threads**:
   - Multiple consumer threads per module
   - Configurable thread count
   - Thread-safe operations

### Optimization
1. **Streaming**:
   - Files processed line-by-line
   - Minimal memory usage
   - Handles large files efficiently

2. **Connection Pooling**:
   - Database connection pools (DBCP2)
   - Redis connection pooling
   - Reduces connection overhead

3. **Caching**:
   - Configuration cached in memory
   - Redis for distributed caching
   - Reduces database load

---

## Security Considerations

### File Upload
- File type validation
- Size limits
- Malicious file detection
- User-based isolation (separate folders)

### Database
- Connection pooling
- Prepared statements (SQL injection prevention)
- User-based data access control

### Redis
- Connection authentication
- Network security
- Queue isolation per campaign

### Kafka
- Secure topic configuration
- Message encryption (configurable)
- Access control lists

---

## Operational Aspects

### Monitoring
- Heartbeat monitoring via Redis
- Prometheus metrics (port 1075)
- Custom logging per module
- Campaign status tracking

### Logging
- Separate logs per module
- Log rotation (rotate_logs.sh)
- Configurable log levels
- Error tracking

### Maintenance
- Automated file cleanup (cronjobs)
- Redis queue cleanup
- Database archival
- Log rotation

### Troubleshooting
- Detailed error logging
- Redis queue inspection
- Database status queries
- Thread monitoring

---

## API Endpoints

### File Upload Module
```
POST /fb-fileupload/save
Parameters:
  - username (required)
  - frompage (required): campaign/group/template
  - files (multipart): CSV/XLS/XLSX/ZIP
Response:
  {
    "statusCode": 200,
    "total": 50000,
    "total_human": "50K",
    "uploaded_files": {
      "success": [{
        "filename": "contacts.csv",
        "r_filename": "contacts_uuid.csv",
        "count": 50000
      }],
      "failed": []
    }
  }
```

### Other Servlets
- **InitializePoller**: Starts polling threads (fb-initialstage)
- **SplitStageServlet**: Initializes split consumers (fb-splitstage)
- **InitializeConsumersServlet**: Starts handover consumers (fb-handoverstage)
- **ServletInitializer**: Initializes campaign finisher (fb-campaignfinisher)

---

## Future Enhancements

### Potential Improvements
1. **Real-time Status Updates**:
   - WebSocket-based live tracking
   - Progress bars
   - Real-time statistics

2. **Advanced Scheduling**:
   - Cron-based scheduling
   - Recurring campaigns
   - Time-zone support

3. **Better Analytics**:
   - Campaign performance metrics
   - Delivery rates
   - Cost analysis
   - A/B testing

4. **Enhanced Security**:
   - OAuth2/JWT authentication
   - Encryption at rest
   - Audit logging
   - GDPR compliance

5. **Cloud Native**:
   - Kubernetes deployment
   - Auto-scaling
   - Cloud storage integration
   - Managed services

---

## Conclusion

The **Beacon File Processor** is a robust, scalable, enterprise-grade system designed for high-volume SMS/messaging campaign processing. Its modular architecture, queue-based design, and comprehensive error handling make it suitable for production environments handling millions of messages.

### Key Strengths
✅ Modular and maintainable
✅ Horizontally scalable
✅ Fault-tolerant with retry mechanisms
✅ Supports multiple campaign types
✅ Comprehensive monitoring
✅ Well-structured codebase

### Use Cases
- Bulk SMS campaigns
- Promotional messaging
- Transactional notifications
- OTP delivery
- Marketing campaigns
- Customer engagement

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Author**: System Documentation Generator
