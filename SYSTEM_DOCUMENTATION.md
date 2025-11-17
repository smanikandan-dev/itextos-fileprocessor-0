# Beacon File Processor System - Complete Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Module Details](#module-details)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)
6. [Queue Architecture](#queue-architecture)

---

## System Overview

**Beacon File Processor** is a comprehensive, distributed file processing system designed for handling bulk SMS campaigns. The system processes uploaded files containing mobile numbers and messages, splits them into manageable chunks, and hands them over to delivery engines for SMS transmission.

### Key Capabilities:
- **File Upload & Validation**: Supports CSV, TXT, XLS, XLSX, and ZIP files
- **Campaign Management**: Handles multiple campaign types (OTM, MTM, Template-based, Quick, Group)
- **File Splitting**: Splits large files into smaller chunks for parallel processing
- **DLT Processing**: Handles DLT (Distributed Ledger Technology) template validation
- **Group Management**: Processes group-based campaigns
- **Exclude Processing**: Filters numbers based on exclusion groups
- **Duplicate Check**: Removes duplicate mobile numbers
- **Real-time Monitoring**: Heart-beat monitoring for all components

### Campaign Types:
1. **OTM (One-to-Many)**: Single message to multiple recipients
2. **MTM (Many-to-Many)**: Different messages for different recipients
3. **TEM (Template)**: Template-based campaigns with placeholders
4. **GROUP**: Group-based campaigns
5. **QUICK**: Quick SMS campaigns

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BEACON FILE PROCESSOR SYSTEM                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐          │
│  │   File       │      │   Initial    │      │    Split     │          │
│  │   Upload     │─────▶│   Stage      │─────▶│    Stage     │          │
│  │  (fb-fileupload)   │  (fb-initialstage) │  (fb-splitstage) │          │
│  └──────────────┘      └──────────────┘      └──────────────┘          │
│         │                     │                      │                   │
│         │                     │                      ▼                   │
│         │                     │              ┌──────────────┐           │
│         │                     │              │  Handover    │           │
│         │                     │              │   Stage      │           │
│         │                     │              │(fb-handoverstage)        │
│         │                     │              └──────────────┘           │
│         │                     │                      │                   │
│         ▼                     ▼                      ▼                   │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │                  REDIS QUEUES (Message Bus)               │          │
│  │  - FileSplitQ   - DeliveryQ   - ExcludeQ   - GroupQ     │          │
│  └──────────────────────────────────────────────────────────┘          │
│         │                     │                      │                   │
│         ▼                     ▼                      ▼                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐         │
│  │   DLT        │      │   Groups     │      │   Exclude    │         │
│  │  Processor   │      │  Processor   │      │  Processor   │         │
│  │(fb-dltfileprocessor)│(fb-groupsprocessor)│(fb-excludeprocessor)    │
│  └──────────────┘      └──────────────┘      └──────────────┘         │
│         │                     │                      │                   │
│         │                     └──────────┬───────────┘                  │
│         │                                │                               │
│         ▼                                ▼                               │
│  ┌──────────────┐              ┌──────────────┐                        │
│  │  Campaign    │              │   Download   │                        │
│  │  Finisher    │              │   Handler    │                        │
│  │(fb-campaignfinisher)       │(fb-downloadhandler)                    │
│  └──────────────┘              └──────────────┘                        │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          ┌──────────────────┐
                          │  Delivery Engine │
                          │   (Kafka/SMS)    │
                          └──────────────────┘
```

---

## Module Details

### 1. **fb-utils** (Shared Utilities)
**Purpose**: Common utilities shared across all modules

**Key Components**:
- Database connection factories (Campaign DB, Config DB, Accounts DB)
- Redis connection managers
- JSON utilities
- Mobile/Email validators
- Unicode utilities
- Constant definitions
- Logging utilities

**Key Classes**:
```
- Utility.java: Common utility functions
- JsonUtility.java: JSON conversion utilities
- MobileValidator.java: Mobile number validation
- Constants.java: Application-wide constants
- ConnectionFactoryForCMDB.java: Campaign database connections
- RedisConnectionTon.java: Redis connection singleton
- HeartBeatMonitoring.java: System health monitoring
```

---

### 2. **fb-fileupload** (File Upload Module)
**Purpose**: Entry point for file uploads from users

**Key Components**:
- HTTP servlet for file upload handling
- File parsers (CSV, TXT, XLS, XLSX)
- ZIP file extraction
- File validation
- Initial file processing

**Main Servlet**:
- `FilesSaver.java` - `/save` endpoint
  - Accepts multipart file uploads
  - Stores files in designated directory
  - Validates file format
  - Extracts ZIP files
  - Returns file statistics

**Supported File Types**:
- `.csv` - Comma-separated values
- `.txt` - Plain text
- `.xls` - Excel 2003 and earlier
- `.xlsx` - Excel 2007 and later
- `.zip` - Compressed archives

**Flow**:
```
1. User uploads file via HTTP POST to /save endpoint
2. File is saved with UUID-based filename
3. File is parsed to count records
4. File metadata is pushed to Redis for tracking
5. Returns JSON response with file details
```

---

### 3. **fb-initialstage** (Initial Stage)
**Purpose**: Polls database for new campaigns and initiates processing

**Key Components**:
- `CampaignMasterPoller.java`: Polls campaign_master and campaign_files tables
- `CampaignGroupsPoller.java`: Polls group-based campaigns
- Database polling mechanism
- Campaign validation

**Database Tables**:
- `campaign_master`: Campaign metadata
- `campaign_files`: File details linked to campaigns

**Polling Logic**:
```sql
SELECT cm.id, cm.cli_id, cm.username, cm.c_name, cm.msg, 
       cf.id as cf_id, cf.fileloc, cf.total, cf.retry_count
FROM campaign_master cm, campaign_files cf
WHERE cm.id = cf.c_id 
  AND cm.status IN ('queued', 'new')
  AND cf.status IN ('queued', 'new')
  AND cf.retry_count <= MAX_RETRY
ORDER BY cm.cli_id, cf.total
```

**Handover**:
- Pushes campaign data to **FileSplitQ** (Redis queue)
- Updates status to 'inprogress'

---

### 4. **fb-splitstage** (Split Stage)
**Purpose**: Splits large files into smaller chunks for parallel processing

**Key Components**:
- `FileSplitQConsumer.java`: Consumes from FileSplitQ
- `MasterFileSplitHandler.java`: Core splitting logic
- File parsing and rewriting
- Database persistence

**Splitting Logic**:
1. Reads campaign data from FileSplitQ
2. Parses master file based on campaign type
3. Splits file into chunks (configurable size, default: SMS_SPLIT_LIMIT)
4. Stores split files in filesystem
5. Inserts split metadata into `campaign_file_splits` table
6. Determines queue priority based on volume:
   - Low volume → High priority (0-1)
   - Medium volume → Medium priority (2-3)
   - High volume → Low priority (4-5)
7. Pushes split file metadata to **DeliveryQ** or **ExcludeQ**

**Queue Selection**:
```
if (exclude_group_ids present)
    → Push to ExcludeQ
else
    → Push to DeliveryQ (Low/Medium/High based on count)
```

**Database Updates**:
```sql
INSERT INTO campaign_file_splits (
    c_f_id, filename, fileloc, total, 
    status, created_ts
) VALUES (?, ?, ?, ?, 'queued', NOW())
```

---

### 5. **fb-handoverstage** (Handover Stage)
**Purpose**: Processes split files and hands over to delivery engine

**Key Components**:
- `SplitFileConsumer.java`: Consumes from DeliveryQ
- `ProcessOTM.java`: Processes One-to-Many campaigns
- `ProcessMTM.java`: Processes Many-to-Many campaigns
- `ProcessTEM.java`: Processes Template campaigns
- `KafkaHandover.java`: Hands over to Kafka/Delivery Engine

**Processing Types**:

**OTM (One-to-Many)**:
- Single message for all recipients
- Reads mobile numbers from split file
- Replaces line breaks in message
- Validates mobile numbers
- Pushes to Kafka

**MTM (Many-to-Many)**:
- Different message for each recipient
- Reads mobile~message pairs from file
- Delimiter-based parsing
- Unicode support
- Pushes to Kafka

**TEM (Template)**:
- Template-based messages with placeholders
- Reads template fields from file
- Replaces placeholders with actual values
- Validates required fields
- Pushes to Kafka

**Kafka Handover**:
```java
// Message format pushed to Kafka
{
    "mobile": "919876543210",
    "message": "Your message here",
    "header": "SMSHEADER",
    "cli_id": "123456",
    "campaign_id": "cm_12345",
    // ... other metadata
}
```

---

### 6. **fb-groupsprocessor** (Groups Processor)
**Purpose**: Processes group-based campaigns

**Key Components**:
- `GroupsMasterPoller.java`: Polls groups_master table
- `GroupsQConsumer.java`: Consumes from GroupQ
- `GroupsCampaignQConsumer.java`: Processes group campaigns
- `GroupsCampaignFileGenerator.java`: Generates campaign files from groups
- `MasterFileSplitHandler.java`: Splits group files

**Flow**:
1. Polls groups_master table for new groups
2. Retrieves group members from database
3. Generates campaign file with group members
4. Splits file if necessary
5. Pushes to DeliveryQ

**Database Tables**:
- `groups_master`: Group metadata
- `group_contacts`: Group member details

---

### 7. **fb-dltfileprocessor** (DLT File Processor)
**Purpose**: Processes DLT (Distributed Ledger Technology) templates

**Key Components**:
- `DltTemplateRequestPoller.java`: Polls DLT requests
- `DltFileQConsumer.java`: Consumes DLT processing requests
- `DltTemplateMasterDAO.java`: DLT template database operations
- File parsing for DLT templates

**DLT Processing**:
1. Polls `dlt_template_request` table
2. Validates template content
3. Parses uploaded DLT files
4. Updates template status
5. Links templates to campaigns

---

### 8. **fb-excludeprocessor** (Exclude Processor)
**Purpose**: Filters out excluded mobile numbers

**Key Components**:
- Exclude group processing
- Mobile number filtering
- Duplicate removal

**Logic**:
```
1. Read split file
2. Load exclude group member IDs
3. Filter out excluded numbers
4. Write filtered file
5. Update counts
6. Push to final DeliveryQ
```

---

### 9. **fb-campaignfinisher** (Campaign Finisher)
**Purpose**: Marks campaigns as completed

**Key Components**:
- `PollerCampaignMasterCompleted.java`: Checks campaign completion
- `PollerCampaignFilesCompleted.java`: Checks file completion
- `UpdateCampaignMasterCompletedDAO.java`: Updates campaign status
- `DQRedisCleaner.java`: Cleans up Redis queues

**Completion Logic**:
```sql
-- Check if all split files are completed
SELECT COUNT(*) as pending 
FROM campaign_file_splits 
WHERE c_f_id = ? AND status != 'completed'

-- If pending = 0, mark campaign as completed
UPDATE campaign_master SET status = 'completed' WHERE id = ?
UPDATE campaign_files SET status = 'completed' WHERE id = ?
```

---

### 10. **fb-downloadhandler** (Download Handler)
**Purpose**: Handles file download requests

**Key Components**:
- Download servlet
- File retrieval
- Access control

---

### 11. **fb-cronjobs** (Cron Jobs)
**Purpose**: Scheduled maintenance tasks

**Key Components**:
- Cleanup old files
- Archive completed campaigns
- Database maintenance
- Log rotation

---

### 12. **fb-scheduleprocessor** (Schedule Processor)
**Purpose**: Handles scheduled campaigns

**Key Components**:
- Schedule validation
- Time-based campaign triggering
- Scheduled campaign polling

---

### 13. **fb-logger** (Logger Module)
**Purpose**: Custom logging implementation

**Key Components**:
- Centralized logging
- Module-specific loggers
- Log aggregation

---

## Data Flow

### Complete Campaign Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CAMPAIGN PROCESSING FLOW                         │
└─────────────────────────────────────────────────────────────────────────┘

1. FILE UPLOAD STAGE
   ┌──────────────┐
   │   User/API   │
   │   Uploads    │
   │    File      │
   └──────┬───────┘
          │
          ▼
   ┌──────────────────────────────────────┐
   │  fb-fileupload                       │
   │  POST /save                          │
   │  - Validates file                    │
   │  - Stores in filesystem              │
   │  - Counts records                    │
   │  - Returns metadata                  │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  Database Insert                     │
   │  - campaign_master                   │
   │  - campaign_files                    │
   │  Status: 'queued'                    │
   └──────────────┬───────────────────────┘

2. POLLING STAGE
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  fb-initialstage                     │
   │  CampaignMasterPoller                │
   │  - Polls every X seconds             │
   │  - Finds status='queued'             │
   │  - Validates user account            │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  Redis Queue: FileSplitQ             │
   │  Message: {                          │
   │    cm_id, cf_id, cli_id, username,   │
   │    fileloc, total, msg, c_type, ...  │
   │  }                                   │
   └──────────────┬───────────────────────┘

3. SPLITTING STAGE
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  fb-splitstage                       │
   │  FileSplitQConsumer                  │
   │  - RPOP from FileSplitQ              │
   │  - Reads master file                 │
   │  - Validates data                    │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  MasterFileSplitHandler              │
   │  - Determines file type              │
   │  - Parses file (CSV/XLS/XLSX/TXT)    │
   │  - Splits into chunks (SMS_SPLIT_LIMIT)│
   │  - Writes split files                │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  Database Insert                     │
   │  campaign_file_splits                │
   │  - For each split file:              │
   │    c_f_s_id, filename, total, etc.   │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  Redis Queue Selection               │
   │  If exclude_group_ids present:       │
   │    → ExcludeQ                         │
   │  Else by volume:                     │
   │    → LowVolumeDeliveryQ              │
   │    → MediumVolumeDeliveryQ           │
   │    → HighVolumeDeliveryQ             │
   └──────────────┬───────────────────────┘

4. HANDOVER STAGE
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  fb-handoverstage                    │
   │  SplitFileConsumer                   │
   │  - RPOPLPUSH from DeliveryQ          │
   │  - Reads split file metadata         │
   │  - Determines campaign type          │
   └──────────────┬───────────────────────┘
                  │
          ┌───────┴───────┬────────────┐
          ▼               ▼            ▼
   ┌──────────┐   ┌──────────┐  ┌──────────┐
   │ProcessOTM│   │ProcessMTM│  │ProcessTEM│
   │          │   │          │  │          │
   │- Read    │   │- Read    │  │- Read    │
   │  mobiles │   │  mobile~ │  │  template│
   │- Validate│   │  message │  │  fields  │
   │- Format  │   │  pairs   │  │- Replace │
   └────┬─────┘   └────┬─────┘  └────┬─────┘
        │              │              │
        └──────────────┴──────────────┘
                       │
                       ▼
   ┌──────────────────────────────────────┐
   │  KafkaHandover                       │
   │  - Formats message for Kafka         │
   │  - Validates mobile numbers          │
   │  - Adds metadata                     │
   │  - Pushes to Kafka topic             │
   └──────────────┬───────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  Kafka Topic / Delivery Engine       │
   │  - Message queued for delivery       │
   │  - SMS gateway picks up              │
   │  - Delivery reports tracked          │
   └──────────────┬───────────────────────┘

5. COMPLETION STAGE
                  │
                  ▼
   ┌──────────────────────────────────────┐
   │  fb-campaignfinisher                 │
   │  PollerCampaignFilesCompleted        │
   │  - Checks all split files processed  │
   │  - Updates campaign_files status     │
   │  - Updates campaign_master status    │
   │  Status: 'completed'                 │
   └──────────────────────────────────────┘
```

---

## Technology Stack

### Programming Language
- **Java 21**: Core application language

### Frameworks & Libraries
- **Servlet API 4.0**: HTTP request handling
- **Apache Commons**: Utilities (Lang, IO, Configuration, Math, CSV)
- **Jackson**: JSON processing
- **Log4j 2.17.0**: Logging framework
- **Apache POI**: Excel file processing

### Databases
- **MariaDB/MySQL**: 
  - Campaign Master Database
  - Configuration Database
  - Accounts Database
- **JDBC/Connection Pooling**: Apache DBCP2

### Message Queue
- **Redis 6.x**:
  - Queue management (FileSplitQ, DeliveryQ, ExcludeQ, GroupQ)
  - Session storage
  - Caching
- **Kafka 2.8.0**: Message handover to delivery engine
- **Jedis 3.6.0**: Java Redis client

### Web Server
- **Jetty 9.4**: Embedded web server for servlets
- **Apache Tomcat**: Alternative deployment option

### Build & Deployment
- **Maven**: Build automation
- **Docker**: Containerization
- **Docker Compose**: Multi-container orchestration

### Monitoring
- **Prometheus**: Metrics collection
- **Custom HeartBeat Monitoring**: Service health checks

### Other
- **Quartz Scheduler 2.3.2**: Job scheduling
- **Elasticsearch 7.12**: Search and analytics (optional)
- **SMPP 5.0**: SMS protocol

---

## Queue Architecture

### Redis Queue Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS QUEUE ARCHITECTURE                    │
└─────────────────────────────────────────────────────────────────┘

1. FileSplitQ
   Purpose: Queue for file splitting requests
   Producer: fb-initialstage
   Consumer: fb-splitstage
   Message Format:
   {
     "cm_id": "12345",
     "cf_id": "67890",
     "cli_id": "1001",
     "username": "user@example.com",
     "fileloc": "/path/to/file.csv",
     "total": "10000",
     "msg": "Your message",
     "c_type": "OTM|MTM|TEM",
     "retry_count": "0"
   }

2. DeliveryQ (Volume-based)
   - LowVolumeDeliveryQ: < 10,000 records (Priority: 0-1)
   - MediumVolumeDeliveryQ: 10,000 - 100,000 (Priority: 2-3)
   - HighVolumeDeliveryQ: > 100,000 (Priority: 4-5)
   
   Purpose: Queue for split file processing
   Producer: fb-splitstage
   Consumer: fb-handoverstage
   Message Format:
   {
     "c_f_s_id": "111",
     "cm_id": "12345",
     "fileloc": "/path/to/split_file_1.txt",
     "total": "1000",
     "c_type": "OTM",
     "PRIORITY": "1"
   }

3. ExcludeQ
   Purpose: Queue for exclude group processing
   Producer: fb-splitstage (when exclude_group_ids present)
   Consumer: fb-excludeprocessor
   Message Format: Same as DeliveryQ + exclude_group_ids

4. GroupQ
   Purpose: Queue for group processing
   Producer: fb-initialstage
   Consumer: fb-groupsprocessor

5. Campaign-specific Queues
   Format: {cm_id}_queue
   Example: "12345_queue"
   Purpose: Holds split files for specific campaign
   Used in: RPOPLPUSH operations for atomic processing

6. Processing Queues
   Format: PRODUCT_NAME:processing:{cm_id}_queue
   Example: "beacon:processing:12345_queue"
   Purpose: Temporary queue while processing
   Ensures: At-least-once delivery semantics
```

### Queue Operations

**Producer Pattern (LPUSH)**:
```java
Jedis jedis = RedisConnectionTon.getInstance().getConnection();
String json = new JsonUtility().convertMapToJSON(dataMap);
jedis.lpush("FileSplitQ", json);
jedis.close();
```

**Consumer Pattern (RPOP)**:
```java
Jedis jedis = RedisConnectionTon.getInstance().getConnection();
String data = jedis.rpop("FileSplitQ");
if (data != null) {
    // Process data
    processData(data);
}
jedis.close();
```

**Atomic Consumer Pattern (RPOPLPUSH)**:
```java
// Pop from main queue and push to processing queue
String campIdQueue = jedis.rpoplpush("DeliveryQ", "DeliveryQ");
String splitFileData = jedis.rpoplpush(
    campIdQueue, 
    "beacon:processing:" + campIdQueue
);
// Process data
// On success, remove from processing queue
jedis.lrem("beacon:processing:" + campIdQueue, 1, splitFileData);
```

---

## Configuration

### Key Configuration Parameters

**File Processing**:
- `SMS_SPLIT_LIMIT`: Number of records per split file (default: 10000)
- `SPLIT_FILE_DELIMITER`: Delimiter for split files (default: `~`)
- `LINE_BREAK_REPLACER`: Character to replace line breaks (default: `{br}`)
- `FILE_STORE_PATH`: Base path for file storage
- `CAMPAIGNS_FILE_STORE_PATH`: Campaign file storage path

**Queue Configuration**:
- `FILE_SPLIT_QUEUE_NAME`: Name of file split queue
- `GROUP_QUEUE_NAME`: Name of group queue
- `MAX_RETRY_COUNT`: Maximum retry attempts (default: 5)

**Thread Configuration**:
- `CONSUMER_SLEEP_TIME`: Sleep time when no data (default: 1000ms)
- `THREAD_SLEEP_TIME`: Sleep time on error (default: 5000ms)
- `DE_NEXT_REQUEST_POP_DELAY`: Delay between requests (default: 1000ms)

**Monitoring**:
- `MONITORING_INSTANCE_ID`: Unique instance identifier
- `HEART_BEAT_INTERVAL`: HeartBeat push interval

---

## Database Schema

### Key Tables

**campaign_master**:
```sql
CREATE TABLE campaign_master (
    id VARCHAR(50) PRIMARY KEY,
    cli_id VARCHAR(50),
    username VARCHAR(100),
    c_name VARCHAR(200),
    msg TEXT,
    header VARCHAR(20),
    template_id VARCHAR(50),
    c_type VARCHAR(10),
    c_lang_type VARCHAR(20),
    status VARCHAR(20),
    created_ts TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_cli_id (cli_id)
);
```

**campaign_files**:
```sql
CREATE TABLE campaign_files (
    id VARCHAR(50) PRIMARY KEY,
    c_id VARCHAR(50),
    filename_ori VARCHAR(500),
    fileloc VARCHAR(1000),
    total INT,
    status VARCHAR(20),
    retry_count INT DEFAULT 0,
    instance_id VARCHAR(50),
    started_ts TIMESTAMP,
    completed_ts TIMESTAMP,
    FOREIGN KEY (c_id) REFERENCES campaign_master(id),
    INDEX idx_status (status),
    INDEX idx_c_id (c_id)
);
```

**campaign_file_splits**:
```sql
CREATE TABLE campaign_file_splits (
    id VARCHAR(50) PRIMARY KEY,
    c_f_id VARCHAR(50),
    filename VARCHAR(500),
    fileloc VARCHAR(1000),
    total INT,
    status VARCHAR(20),
    retry_count INT DEFAULT 0,
    created_ts TIMESTAMP,
    FOREIGN KEY (c_f_id) REFERENCES campaign_files(id),
    INDEX idx_status (status),
    INDEX idx_c_f_id (c_f_id)
);
```

---

## Error Handling & Retry Logic

### Retry Mechanism
1. **Max Retry Count**: Configurable (default: 5)
2. **Retry on Failure**: 
   - Database deadlock: Retry without incrementing count
   - Other errors: Increment retry count
3. **Retry Exhausted**: Update status to 'FAILED' with reason

### Error Scenarios
- **File Not Found**: Update status to 'FAILED'
- **Invalid File Format**: Update status to 'INVALIDFILE'
- **Zero Records**: Update status to 'INVALIDFILE'
- **Database Error**: Retry with exponential backoff
- **Redis Connection Error**: Reconnect and retry
- **Kafka Push Failure**: Retry up to MAX_RETRY_COUNT

---

## Monitoring & Health Checks

### HeartBeat Monitoring
Each consumer thread pushes heartbeat every X seconds:
```java
new HeartBeatMonitoring().pushConsumersHeartBeat(
    "FP-SplitStage",          // Module name
    "FileSplitQConsumer",      // Consumer name
    instanceId,                 // Instance ID
    this.getName(),            // Thread name
    timestampAsString          // Current timestamp
);
```

### Metrics Tracked
- Active consumer threads
- Queue lengths
- Processing rate
- Error rate
- Retry counts
- File processing time
- Campaign completion time

---

## Deployment

### Docker Deployment
```yaml
# docker-compose-singleton.yml
version: '3.8'
services:
  fileprocessor:
    build:
      context: .
      dockerfile: Dockerfile_singleton
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
      - MARIADB_HOST=mariadb
    volumes:
      - ./properties:/opt/properties
      - ./logs:/opt/logs
```

### Scaling
- **Horizontal Scaling**: Deploy multiple instances of consumers
- **Vertical Scaling**: Increase thread pool sizes
- **Redis Sharding**: Multiple Redis instances for queue distribution

---

## Best Practices

1. **File Naming**: Use UUID to avoid collisions
2. **Transaction Management**: Use database transactions for consistency
3. **Connection Management**: Always close JDBC/Redis connections in finally block
4. **Logging**: Log with file_id for tracking
5. **Error Handling**: Graceful degradation with retries
6. **Monitoring**: Continuous heartbeat monitoring
7. **Configuration**: Externalize all configurations
8. **Security**: Validate user permissions before processing

---

## Conclusion

The Beacon File Processor is a robust, scalable system designed for high-throughput SMS campaign processing. Its modular architecture, queue-based communication, and comprehensive error handling make it suitable for enterprise-level bulk messaging operations.

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-17  
**Maintained By**: Beacon File Processor Team
