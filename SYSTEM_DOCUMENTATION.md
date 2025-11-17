# Beacon File Processor - Complete System Documentation

## 📋 Table of Contents
1. [System Overview](#system-overview)
2. [Architecture Overview](#architecture-overview)
3. [Module Descriptions](#module-descriptions)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [Detailed Flow Walkthrough](#detailed-flow-walkthrough)
6. [Technology Stack](#technology-stack)
7. [Queue Architecture](#queue-architecture)

---

## 🎯 System Overview

**Beacon File Processor** is a distributed, multi-stage SMS campaign processing system built with Java. It handles bulk SMS campaigns by processing uploaded files containing mobile numbers and messages, splitting them into manageable chunks, and handing them over to delivery engines.

### Key Features
- Multi-format file support (CSV, XLS, XLSX, ZIP)
- Distributed processing using Redis queues
- Campaign type support: OTM (One-To-Many), MTM (Many-To-Many), Template-based
- Group-based campaigns
- Exclude list processing
- DLT (Do Not Disturb) file processing
- Retry mechanisms with configurable limits
- Real-time heartbeat monitoring
- Duplicate detection and removal

---

## 🏗️ Architecture Overview

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        BEACON FILE PROCESSOR SYSTEM                      │
└─────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │   User/API   │
    └──────┬───────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          1. FILE UPLOAD STAGE                             │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  fb-fileupload: Receives files, extracts ZIPs, validates   │        │
│  │  Supports: CSV, XLS, XLSX, TXT, ZIP                         │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ Store files
                                  ▼
                           ┌──────────────┐
                           │  File System  │
                           │  + Database   │
                           └──────┬───────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        2. INITIAL STAGE (Polling)                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  fb-initialstage: Polls campaign_master & campaign_files    │        │
│  │  - CampaignMasterPoller: Fetches new campaigns              │        │
│  │  - GroupCampaignPoller: Fetches group-based campaigns       │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ Push to FileSplitQ
                                  ▼
                           ┌──────────────┐
                           │ Redis Queue: │
                           │ FileSplitQ   │
                           └──────┬───────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          3. SPLIT STAGE                                   │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  fb-splitstage: Splits large files into smaller chunks      │        │
│  │  - Reads master file                                         │        │
│  │  - Splits based on SMS_SPLIT_LIMIT                          │        │
│  │  - Creates child records in campaign_file_splits             │        │
│  │  - Handles OTM, MTM, Template campaigns                      │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ Push to DQ or ExcludeQ
                                  ▼
            ┌─────────────────────────────────────┐
            │                                     │
            ▼                                     ▼
    ┌──────────────┐                    ┌──────────────────┐
    │ Redis Queue: │                    │  Redis Queue:    │
    │ ExcludeQ     │                    │  DeliveryQ (DQ)  │
    └──────┬───────┘                    └────────┬─────────┘
           │                                     │
           ▼                                     │
┌──────────────────────────────────────┐        │
│   4. EXCLUDE PROCESSOR               │        │
│  ┌──────────────────────────────┐   │        │
│  │  fb-excludeprocessor         │   │        │
│  │  - Removes exclude groups    │   │        │
│  │  - Filters numbers           │   │        │
│  │  - Creates filtered files    │   │        │
│  └──────────────────────────────┘   │        │
└─────────────────┬────────────────────┘        │
                  │ Push to DQ                  │
                  └─────────────────────────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         5. HANDOVER STAGE                                 │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  fb-handoverstage: Final processing before delivery          │        │
│  │  - ProcessOTM: One-To-Many campaigns                         │        │
│  │  - ProcessMTM: Many-To-Many campaigns                        │        │
│  │  - ProcessTEM: Template-based campaigns                      │        │
│  │  - Reads split files line by line                           │        │
│  │  - Sends to Delivery Engine Platform Queue                   │        │
│  └─────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ Push to Platform Delivery Engine
                                  ▼
                        ┌──────────────────┐
                        │  Beacon Platform │
                        │ Delivery Engine  │
                        └──────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                        SUPPORTING MODULES                                 │
│                                                                           │
│  fb-groupsprocessor    : Processes group-based campaigns                 │
│  fb-dltfileprocessor   : Processes DLT files                             │
│  fb-campaignfinisher   : Marks campaigns as completed                    │
│  fb-cronjobs           : Scheduled tasks and cleanup                     │
│  fb-scheduleprocessor  : Handles scheduled campaigns                     │
│  fb-downloadhandler    : Handles file downloads/conversions              │
│  fb-inmemoryrefresh    : Refreshes in-memory cache                       │
│  fb-logger             : Centralized logging                             │
│  fb-utils              : Common utilities and DTOs                       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Module Descriptions

### 1. **fb-fileupload** (File Upload Module)
**Purpose**: Entry point for file uploads

**Key Components**:
- `FilesSaver.java`: HTTP servlet that receives file uploads
- `FileParser`: Parses different file formats (CSV, XLS, XLSX)
- `ZipHandler`: Extracts ZIP files
- `MobileValidator`: Validates mobile numbers

**Key Functions**:
- Accepts multi-part file uploads via HTTP POST
- Supports multiple file formats
- Extracts and processes ZIP archives
- Validates file contents asynchronously using FutureTasks
- Stores files in user-specific directories
- Tracks uploaded files in Redis for cleanup

**API Endpoint**:
```
POST /save
Parameters:
  - username: User identifier
  - frompage: Source (campaign/group/template)
  - files: Multiple file uploads
```

---

### 2. **fb-initialstage** (Initial Stage - Polling)
**Purpose**: Polls database for new campaigns and initiates processing

**Key Components**:
- `CampaignMasterPoller.java`: Polls `campaign_master` and `campaign_files` tables
- `GroupCampaignPoller.java`: Polls group-based campaigns
- `CampaignMasterDAO.java`: Database operations

**Key Functions**:
- Continuously polls database for campaigns with status "queued"
- Retrieves campaign metadata (message, header, type, DLT info)
- Validates user cluster (rejects 'otp' cluster)
- Updates campaign status to "inprogress"
- Pushes campaign data to FileSplitQ (Redis queue)
- Implements retry mechanism with configurable retry counts
- Sorts campaigns by client ID and file size

**Database Tables**:
- `campaign_master`: Main campaign information
- `campaign_files`: File details for each campaign

---

### 3. **fb-splitstage** (Split Stage)
**Purpose**: Splits large files into manageable chunks

**Key Components**:
- `FileSplitQConsumer.java`: Consumes from FileSplitQ
- `MasterFileSplitHandler.java`: Core splitting logic
- `FileParser`: Parses files (via fb-fileparser)

**Key Functions**:
- Consumes campaign data from FileSplitQ
- Parses uploaded files (CSV, XLS, XLSX, TXT)
- Converts files to standardized TXT format with delimiter
- Splits files based on SMS_SPLIT_LIMIT (configurable)
- Creates child records in `campaign_file_splits` table
- Handles different campaign types:
  - **OTM (One-To-Many)**: Single message to multiple numbers
  - **MTM (Many-To-Many)**: Custom message per number
  - **Template**: Message with placeholders
- Determines delivery queue based on file size:
  - Low volume → High priority
  - Medium volume → Medium priority
  - High volume → Low priority
- Handles Unicode conversion
- Pushes split files to DeliveryQ or ExcludeQ

**Split File Format**:
```
For OTM: mobile_number
For MTM: mobile_number|message
For Template: mobile_number|field1|field2|...
```

---

### 4. **fb-excludeprocessor** (Exclude Processor)
**Purpose**: Filters out excluded groups/numbers

**Key Components**:
- `ExcludeConsumer.java`: Consumes from ExcludeQ
- `FileDataExtractor.java`: Extracts and filters data

**Key Functions**:
- Consumes from ExcludeQ (queue name ends with "_exclude")
- Reads exclude group IDs from campaign metadata
- Filters mobile numbers against exclude lists
- Creates two files:
  - **Non-exclude file**: Numbers to be sent
  - **Exclude file**: Filtered numbers (for tracking)
- Updates `campaign_file_splits` with exclude count
- Pushes non-exclude file to DeliveryQ
- Handles retry mechanism on failure

---

### 5. **fb-handoverstage** (Handover Stage)
**Purpose**: Final processing and handover to delivery engine

**Key Components**:
- `SplitFileConsumer.java`: Consumes from DeliveryQ
- `ProcessOTM.java`: Processes OTM campaigns
- `ProcessMTM.java`: Processes MTM campaigns
- `ProcessTEM.java`: Processes Template campaigns

**Key Functions**:
- Consumes from DeliveryQ using `rpoplpush` (atomic operation)
- Reads split files line by line
- Performs final validations:
  - Mobile number validation
  - Message validation
  - DLT template validation
- Constructs message payloads
- Pushes to Beacon Platform Delivery Engine queue
- Updates processing status in database
- Implements retry mechanism with processing queue

**Processing Flow**:
```
1. Pop campaign ID from DeliveryQ → Processing Queue
2. Pop split file metadata from campaign ID queue
3. Process based on campaign type (OTM/MTM/Template)
4. Read file and send each record to Platform Queue
5. Remove from processing queue on success
6. On failure: Retry or mark as FAILED
```

---

### 6. **fb-groupsprocessor** (Groups Processor)
**Purpose**: Processes group-based campaigns

**Key Components**:
- `GroupsQConsumer.java`: Consumes from GroupQ
- `GroupsCampaignQConsumer.java`: Processes group campaigns
- `GroupsFileSplitQConsumer.java`: Splits group files
- `MasterFileSplitHandler.java`: Handles file splitting

**Key Functions**:
- Processes campaigns targeting predefined groups
- Fetches group member lists from database
- Creates files with group members
- Similar splitting logic as fb-splitstage
- Pushes to DeliveryQ for handover

---

### 7. **fb-dltfileprocessor** (DLT File Processor)
**Purpose**: Processes DLT (Do Not Disturb) registry files

**Key Components**:
- `DltFileQConsumer.java`: Consumes DLT file requests
- File processing and validation logic

**Key Functions**:
- Processes DLT template files
- Validates DLT entity IDs and template IDs
- Updates DLT information in campaigns
- Ensures compliance with telecom regulations

---

### 8. **fb-campaignfinisher** (Campaign Finisher)
**Purpose**: Marks campaigns as completed

**Key Components**:
- Campaign completion polling and status updates

**Key Functions**:
- Monitors campaign progress
- Updates campaign status to "completed"
- Generates campaign reports
- Triggers notifications

---

### 9. **fb-cronjobs** (Cron Jobs)
**Purpose**: Scheduled maintenance tasks

**Key Functions**:
- File cleanup (old uploaded files)
- Database maintenance
- Log rotation
- Statistics aggregation

---

### 10. **fb-scheduleprocessor** (Schedule Processor)
**Purpose**: Handles scheduled campaigns

**Key Functions**:
- Processes campaigns with future scheduled_ts
- Pushes to processing queues at scheduled time
- Handles timezone conversions

---

### 11. **fb-downloadhandler** (Download Handler)
**Purpose**: Handles file download and conversion requests

**Key Components**:
- `CsvToExcelConvertionRequestConsumer.java`: CSV to Excel conversion

**Key Functions**:
- Converts CSV files to Excel format
- Handles download requests
- File format conversions

---

### 12. **fb-utils** (Utilities)
**Purpose**: Common utilities, DTOs, and singletons

**Key Components**:
- `Constants.java`: Application constants
- `Utility.java`: Common utility methods
- DTOs: `FileDataBean`, `SplitFileData`, `Templates`
- Singletons: Database connection factories, Redis connection pools
- `HeartBeatMonitoring.java`: Pushes heartbeat for monitoring

---

### 13. **fb-logger** (Logger)
**Purpose**: Centralized logging

**Key Components**:
- Module-specific log instances
- Log4j2 configuration

---

### 14. **fb-fileparser** (File Parser)
**Purpose**: Parses different file formats

**Key Components**:
- `FileParser.java`: Interface
- `CsvFileParser.java`: CSV parser
- `XlsFileParser.java`: XLS parser
- `XlsxFileParser.java`: XLSX parser
- `FileChopHandler.java`: Splits files into chunks

---

### 15. **fb-inmemoryrefresh** (In-Memory Refresh)
**Purpose**: Refreshes in-memory configuration cache

**Key Functions**:
- Reloads configuration without restart
- Refreshes Redis connection pools
- Updates application properties

---

## 📊 Data Flow Diagrams

### Complete Campaign Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CAMPAIGN PROCESSING FLOW                          │
└─────────────────────────────────────────────────────────────────────────┘

Step 1: FILE UPLOAD
═══════════════════
User → HTTP POST /save → FilesSaver Servlet
                              │
                              ├─→ Validate username & frompage
                              ├─→ Store file in: /path/{username}/filename_{uuid}.ext
                              ├─→ If ZIP: Extract contents
                              ├─→ Parse file (async using FutureTask)
                              ├─→ Count records
                              ├─→ Track files in Redis (for cleanup)
                              └─→ Return JSON response with file count


Step 2: DATABASE INSERT
═══════════════════════
User/API → Insert into campaign_master (campaign details)
User/API → Insert into campaign_files (file details)
Status: QUEUED


Step 3: INITIAL STAGE (Polling)
════════════════════════════════
CampaignMasterPoller (Thread) → Poll every X milliseconds
    │
    ├─→ SELECT from campaign_master & campaign_files 
    │   WHERE status='queued' AND retry_count <= MAX_RETRY
    │   ORDER BY cli_id, total
    │
    ├─→ Get user details & validate cluster
    │
    ├─→ UPDATE status='inprogress'
    │
    └─→ LPUSH to Redis → FileSplitQ
        Format: {cm_id, cf_id, cli_id, fileloc, msg, header, c_type, ...}


Step 4: SPLIT STAGE
═══════════════════
FileSplitQConsumer (Thread) → RPOP from FileSplitQ
    │
    ├─→ Check if already split (SELECT from campaign_file_splits)
    │
    ├─→ If NOT split:
    │   │
    │   ├─→ Parse master file
    │   │   ├─→ FileParserFactory.get(fileloc)
    │   │   ├─→ Read file line by line
    │   │   └─→ Convert to TXT format with delimiter
    │   │
    │   ├─→ Split file into chunks (based on SMS_SPLIT_LIMIT)
    │   │   Example: 100,000 records → 10 files of 10,000 each
    │   │
    │   └─→ INSERT into campaign_file_splits
    │       Format: {c_f_s_id, cm_id, cf_id, fileloc, total, status='queued', ...}
    │
    ├─→ Determine queue based on total count:
    │   ├─→ < 10,000: LowVolumeDQ (Priority: 1)
    │   ├─→ < 100,000: MediumVolumeDQ (Priority: 2)
    │   └─→ >= 100,000: HighVolumeDQ (Priority: 4)
    │
    ├─→ If exclude_group_ids present:
    │   └─→ Queue = Queue + "_exclude"
    │
    └─→ LPUSH to Redis → DeliveryQ or ExcludeQ
        For each split file:
        LPUSH {cm_id}:Q → split file metadata
        LPUSH DQ → {cm_id}:Q


Step 5: EXCLUDE PROCESSING (If needed)
═══════════════════════════════════════
ExcludeConsumer (Thread) → RPOPLPUSH from ExcludeQ
    │
    ├─→ RPOPLPUSH ExcludeQ → ExcludeQ (get cm_id:Q)
    │
    ├─→ RPOP {cm_id}_exclude:Q → split file metadata
    │
    ├─→ Read split file
    │
    ├─→ Filter numbers against exclude groups
    │   ├─→ Query exclude group members
    │   └─→ Remove matching numbers
    │
    ├─→ Create two files:
    │   ├─→ non_exclude_file.txt (numbers to send)
    │   └─→ exclude_file.txt (filtered numbers)
    │
    ├─→ UPDATE campaign_file_splits 
    │   SET fileloc=non_exclude_file, exclude_count=X
    │
    └─→ LPUSH to DeliveryQ (without _exclude suffix)
        LPUSH {cm_id}:Q → updated metadata
        LPUSH DQ → {cm_id}:Q


Step 6: HANDOVER STAGE (Final Processing)
══════════════════════════════════════════
SplitFileConsumer (Thread) → RPOPLPUSH from DeliveryQ
    │
    ├─→ RPOPLPUSH DQ → DQ (get cm_id:Q, keep in queue)
    │
    ├─→ RPOPLPUSH {cm_id}:Q → processing:{cm_id}:Q
    │   (Move to processing queue for retry safety)
    │
    ├─→ Parse split file metadata
    │
    ├─→ Determine campaign type & process:
    │   │
    │   ├─→ OTM (One-To-Many):
    │   │   └─→ ProcessOTM.processData()
    │   │       ├─→ Open split file
    │   │       ├─→ Read line by line: mobile_number
    │   │       ├─→ Validate mobile number
    │   │       ├─→ Create JSON payload:
    │   │       │   {mobile, msg, header, cli_id, ...}
    │   │       └─→ LPUSH to Platform Delivery Queue
    │   │
    │   ├─→ MTM (Many-To-Many):
    │   │   └─→ ProcessMTM.processData()
    │   │       ├─→ Open split file
    │   │       ├─→ Read line by line: mobile_number|message
    │   │       ├─→ Validate mobile & message
    │   │       ├─→ Replace line breaks with LINE_BREAK_REPLACER
    │   │       ├─→ Create JSON payload:
    │   │       │   {mobile, msg, header, cli_id, ...}
    │   │       └─→ LPUSH to Platform Delivery Queue
    │   │
    │   └─→ TEMPLATE:
    │       └─→ ProcessTEM.processData()
    │           ├─→ Open split file
    │           ├─→ Read line by line: mobile|field1|field2|...
    │           ├─→ Parse template placeholders
    │           ├─→ Replace {field1}, {field2} with actual values
    │           ├─→ Create JSON payload:
    │           │   {mobile, msg, header, cli_id, ...}
    │           └─→ LPUSH to Platform Delivery Queue
    │
    ├─→ On success:
    │   └─→ LREM processing:{cm_id}:Q (remove from processing)
    │
    └─→ On failure:
        ├─→ If retry_count < MAX_RETRY:
        │   └─→ LPUSH back to DQ with retry_count++
        └─→ Else:
            └─→ UPDATE campaign_file_splits SET status='FAILED'


Step 7: PLATFORM DELIVERY ENGINE
═════════════════════════════════
Beacon Platform → RPOP from Platform Delivery Queue
    │
    ├─→ Route to appropriate SMS gateway
    ├─→ Send SMS
    └─→ Update delivery status
```

---

### Redis Queue Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        REDIS QUEUE ARCHITECTURE                          │
└─────────────────────────────────────────────────────────────────────────┘

┌────────────────────┐
│    FileSplitQ      │  ← Initial Stage pushes campaign metadata
└─────────┬──────────┘
          │ LPUSH/RPOP
          ▼
    ┌──────────┐
    │ Consumer │ (FileSplitQConsumer)
    └──────────┘
          │
          │ After splitting
          ▼
┌──────────────────────────────────────────────────────────────┐
│                    Delivery Queues (DQ)                       │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ LowVolumeDQ      │  │ LowVolumeDQ      │                 │
│  │                  │  │ _exclude         │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ MediumVolumeDQ   │  │ MediumVolumeDQ   │                 │
│  │                  │  │ _exclude         │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ HighVolumeDQ     │  │ HighVolumeDQ     │                 │
│  │                  │  │ _exclude         │                 │
│  └──────────────────┘  └──────────────────┘                 │
└──────────────────────────────────────────────────────────────┘
          │
          │ RPOPLPUSH (atomic)
          ▼
┌──────────────────────────────────────────────────────────────┐
│              Campaign ID Queues (cm_id:Q)                     │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  {cm_id}:Q  →  [metadata1, metadata2, metadata3, ...]   ││
│  └──────────────────────────────────────────────────────────┘│
└───────────────────────────────┬──────────────────────────────┘
                                │ RPOPLPUSH
                                ▼
┌──────────────────────────────────────────────────────────────┐
│           Processing Queue (processing:cm_id:Q)               │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  FP:processing:{cm_id}:Q  →  [metadata (temp)]          ││
│  └──────────────────────────────────────────────────────────┘│
│                                                               │
│  Purpose: Holds metadata during processing for retry safety  │
│  On Success: LREM (remove)                                   │
│  On Failure: Push back to DQ or mark FAILED                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  Supporting Queues                            │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ GroupQ           │  │ GroupCampaignQ   │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ GroupFileSplitQ  │  │ DltFileQ         │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ ExcludeNumberQ   │  │ UpdateSQLQ       │                 │
│  └──────────────────┘  └──────────────────┘                 │
└──────────────────────────────────────────────────────────────┘

Queue Operations:
─────────────────
LPUSH: Add to left (head) of queue
RPOP: Remove from right (tail) of queue
RPOPLPUSH: Atomically pop from one queue and push to another
LREM: Remove specific item from queue
```

---

### Campaign Types Processing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CAMPAIGN TYPES                                   │
└─────────────────────────────────────────────────────────────────────────┘

1. OTM (One-To-Many) Campaign
══════════════════════════════
Description: Single message sent to multiple recipients
Use Case: Promotional campaigns, announcements

Upload File Format (CSV/XLS):
  mobile_number
  9876543210
  9876543211
  9876543212

Processing:
  ├─→ Read file line by line
  ├─→ Extract mobile number
  ├─→ Use campaign message from campaign_master.msg
  └─→ Send: {mobile: 9876543210, msg: "Campaign Message", header: "SENDER"}

Split File Format:
  9876543210
  9876543211
  9876543212


2. MTM (Many-To-Many) Campaign
═══════════════════════════════
Description: Different message for each recipient
Use Case: Personalized messages, invoices

Upload File Format (CSV/XLS):
  mobile_number | message
  9876543210 | Hello John, your order #123 is ready
  9876543211 | Hello Jane, your invoice #456 is attached
  9876543212 | Hello Bob, your appointment is confirmed

Processing:
  ├─→ Read file line by line
  ├─→ Split by delimiter: mobile|message
  ├─→ Validate both fields
  └─→ Send: {mobile: 9876543210, msg: "Hello John...", header: "SENDER"}

Split File Format:
  9876543210|Hello John, your order #123 is ready
  9876543211|Hello Jane, your invoice #456 is attached


3. TEMPLATE Campaign
════════════════════
Description: Template with dynamic placeholders
Use Case: Transactional messages with variable fields

Upload File Format (CSV/XLS):
  mobile_number | name | amount | date
  9876543210 | John | 1500 | 2024-01-15
  9876543211 | Jane | 2500 | 2024-01-16

Template Definition:
  "Dear {name}, your payment of Rs.{amount} is due on {date}"

Processing:
  ├─→ Read file line by line
  ├─→ Parse columns based on template fields
  ├─→ Replace placeholders: {name} → John, {amount} → 1500, {date} → 2024-01-15
  ├─→ Final message: "Dear John, your payment of Rs.1500 is due on 2024-01-15"
  └─→ Send: {mobile: 9876543210, msg: "Dear John...", header: "SENDER"}

Split File Format:
  9876543210|John|1500|2024-01-15
  9876543211|Jane|2500|2024-01-16


4. GROUP Campaign
═════════════════
Description: Campaign targeting predefined groups
Use Case: Sending to saved contact groups

Processing:
  ├─→ Fetch group members from database
  ├─→ Create file with mobile numbers
  ├─→ Process as OTM campaign
  └─→ Send same message to all group members


5. QUICK Campaign
═════════════════
Description: Quick send without file upload (API-based)
Use Case: Instant notifications

Processing:
  ├─→ Receive numbers via API (comma-separated or JSON array)
  ├─→ Create temporary file
  ├─→ Process as OTM campaign
  └─→ Send immediately
```

---

## 🔄 Detailed Flow Walkthrough

### Example: Processing a 50,000 Record OTM Campaign

```
Step-by-Step Execution:
═══════════════════════

[T=0] User uploads file via UI
      POST /save?username=testuser&frompage=campaign
      File: contacts.csv (50,000 records)

[T=1] FilesSaver Servlet
      ├─→ Save to: /files/testuser/contacts_uuid-123.csv
      ├─→ Parse file (count records): 50,000
      ├─→ Track in Redis: uploaded_files:testuser
      └─→ Return: {total: 50000, status: "success"}

[T=2] User creates campaign via UI/API
      INSERT INTO campaign_master
        (cli_id, username, c_name, msg, header, c_type, status)
      VALUES
        ('CLI001', 'testuser', 'Promo Campaign', 'Get 50% off!', 
         'PROMO', 'otm', 'queued')
      → cm_id = 12345

      INSERT INTO campaign_files
        (c_id, filename_ori, fileloc, status, total)
      VALUES
        (12345, 'contacts.csv', '/files/testuser/contacts_uuid-123.csv',
         'queued', 50000)
      → cf_id = 67890

[T=3] CampaignMasterPoller (runs every 1 second)
      ├─→ SELECT * FROM campaign_master cm, campaign_files cf
          WHERE cm.status='queued' AND cf.status='queued'
      ├─→ Found: cm_id=12345, cf_id=67890
      ├─→ Validate user cluster: NOT 'otp' ✓
      ├─→ UPDATE campaign_master SET status='inprogress'
      ├─→ UPDATE campaign_files SET status='inprogress'
      └─→ LPUSH FileSplitQ → {cm_id:12345, cf_id:67890, ...}

[T=4] FileSplitQConsumer
      ├─→ RPOP FileSplitQ → Get campaign data
      ├─→ Check campaign_file_splits: No records found
      ├─→ Parse file: /files/testuser/contacts_uuid-123.csv
      │   ├─→ FileParserFactory.get() → CsvFileParser
      │   ├─→ Read 50,000 lines
      │   └─→ Validate mobile numbers
      │
      ├─→ Split file (SMS_SPLIT_LIMIT = 10,000)
      │   ├─→ Create 5 split files:
      │   │   - split_1.txt (10,000 records)
      │   │   - split_2.txt (10,000 records)
      │   │   - split_3.txt (10,000 records)
      │   │   - split_4.txt (10,000 records)
      │   │   - split_5.txt (10,000 records)
      │   │
      │   └─→ INSERT INTO campaign_file_splits (5 records)
      │       c_f_s_id | cm_id | cf_id | fileloc        | total  | status
      │       ─────────┼───────┼───────┼────────────────┼────────┼────────
      │       1001     | 12345 | 67890 | split_1.txt    | 10000  | queued
      │       1002     | 12345 | 67890 | split_2.txt    | 10000  | queued
      │       1003     | 12345 | 67890 | split_3.txt    | 10000  | queued
      │       1004     | 12345 | 67890 | split_4.txt    | 10000  | queued
      │       1005     | 12345 | 67890 | split_5.txt    | 10000  | queued
      │
      ├─→ Determine queue: 50,000 → MediumVolumeDQ
      ├─→ Set PRIORITY = 2
      └─→ LPUSH to Redis:
          LPUSH 12345:Q → {c_f_s_id:1001, fileloc:split_1.txt, ...}
          LPUSH 12345:Q → {c_f_s_id:1002, fileloc:split_2.txt, ...}
          LPUSH 12345:Q → {c_f_s_id:1003, fileloc:split_3.txt, ...}
          LPUSH 12345:Q → {c_f_s_id:1004, fileloc:split_4.txt, ...}
          LPUSH 12345:Q → {c_f_s_id:1005, fileloc:split_5.txt, ...}
          LPUSH MediumVolumeDQ → "12345:Q"

[T=5] SplitFileConsumer (Thread 1)
      ├─→ RPOPLPUSH MediumVolumeDQ → MediumVolumeDQ (get "12345:Q")
      ├─→ RPOPLPUSH 12345:Q → FP:processing:12345:Q
      │   → Get {c_f_s_id:1001, fileloc:split_1.txt, total:10000, ...}
      │
      ├─→ Campaign type = OTM → ProcessOTM.processData()
      │   ├─→ Open file: split_1.txt
      │   ├─→ Read line 1: 9876543210
      │   │   ├─→ Validate mobile: ✓
      │   │   ├─→ Create payload: 
      │   │   │   {mobile:9876543210, msg:"Get 50% off!", 
      │   │   │    header:"PROMO", cli_id:"CLI001", ...}
      │   │   └─→ LPUSH Platform:DeliveryEngine:Queue
      │   │
      │   ├─→ Read line 2: 9876543211
      │   │   └─→ LPUSH Platform:DeliveryEngine:Queue
      │   │
      │   ├─→ ... (repeat for all 10,000 records)
      │   │
      │   └─→ File processed ✓
      │
      ├─→ LREM FP:processing:12345:Q (remove metadata)
      └─→ UPDATE campaign_file_splits SET status='completed'
          WHERE c_f_s_id=1001

[T=6] SplitFileConsumer (Thread 2-5) - Process remaining files
      ├─→ Thread 2 processes c_f_s_id:1002
      ├─→ Thread 3 processes c_f_s_id:1003
      ├─→ Thread 4 processes c_f_s_id:1004
      └─→ Thread 5 processes c_f_s_id:1005

[T=7] All split files completed
      ├─→ campaign_file_splits: All 5 records status='completed'
      └─→ CampaignFinisher updates:
          UPDATE campaign_master SET status='completed'
          UPDATE campaign_files SET status='completed'

[T=8] Platform Delivery Engine
      ├─→ Processes 50,000 messages from Platform:DeliveryEngine:Queue
      ├─→ Routes to appropriate SMS gateways
      └─→ Updates delivery status

Total Time: ~2-5 minutes (depending on system load)
```

---

## 🛠️ Technology Stack

### Core Technologies
- **Language**: Java 21
- **Build Tool**: Maven 3.x
- **Web Container**: Jetty 9.4+ / Tomcat (embedded)
- **Servlet API**: 4.0.1

### Frameworks & Libraries
- **Commons Libraries**:
  - Apache Commons Configuration 1.10
  - Apache Commons IO
  - Apache Commons Lang
  - Apache Commons Net 3.6
  - Apache Commons CSV 1.8
  - Apache Commons DBCP2 2.8.0
  - Apache Commons Validator 1.6
  - Apache Commons Codec 1.10
  - Apache Commons Math3 3.6.1

- **Logging**:
  - Log4j 2.17.0
  - Commons Logging 1.2

- **Database**:
  - MariaDB Java Client 2.3.0
  - Connection Pooling (DBCP2)

- **Caching & Queuing**:
  - Redis (Jedis 3.6.0)
  - Kafka Clients 2.8.0

- **Data Processing**:
  - Jackson Databind 2.12.1
  - JSON Simple 1.1.1
  - Google Gson 2.8.8
  - Apache POI (for XLS/XLSX parsing)

- **Scheduling**:
  - Quartz Scheduler 2.3.2

- **Rules Engine**:
  - Drools 5.4.0.Final

- **Monitoring**:
  - Prometheus (simpleclient 0.9.0)

- **HTTP Client**:
  - Apache HTTP Client 4.5.13

- **Search**:
  - Elasticsearch High-Level Client 7.12.0

- **Other**:
  - SMPP (Cloudhopper SMPP 5.0.0)
  - JAXB API 2.2.11
  - UserAgent Parser (Bitwalker) 1.21

### Database Schema (Key Tables)

```sql
-- Campaign Master Table
CREATE TABLE campaign_master (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  cli_id VARCHAR(50),
  username VARCHAR(100),
  c_name VARCHAR(255),
  msg TEXT,
  header VARCHAR(15),
  intl_header VARCHAR(15),
  template_id VARCHAR(50),
  template_type VARCHAR(20),
  template_mobile_column VARCHAR(50),
  dlt_entity_id VARCHAR(50),
  dlt_template_id VARCHAR(50),
  c_type VARCHAR(20),  -- otm, mtm, template, group, quick
  c_lang_type VARCHAR(20),  -- text, unicode
  remove_dupe_yn CHAR(1),
  scheduled_ts DATETIME,
  status VARCHAR(20),  -- queued, inprogress, completed, failed
  reason TEXT,
  created_ts DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Campaign Files Table
CREATE TABLE campaign_files (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  c_id BIGINT,  -- FK to campaign_master.id
  group_id BIGINT,
  filename_ori VARCHAR(255),
  fileloc VARCHAR(500),
  exclude_group_ids TEXT,
  retry_count INT DEFAULT 0,
  total INT,
  status VARCHAR(20),
  reason TEXT,
  instance_id VARCHAR(50),
  started_ts DATETIME,
  created_ts DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (c_id) REFERENCES campaign_master(id)
);

-- Campaign File Splits Table
CREATE TABLE campaign_file_splits (
  c_f_s_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  cm_id BIGINT,  -- FK to campaign_master.id
  cf_id BIGINT,  -- FK to campaign_files.id
  fileloc VARCHAR(500),
  total INT,
  exclude_count INT DEFAULT 0,
  retry_count INT DEFAULT 0,
  status VARCHAR(20),
  reason TEXT,
  created_ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Redis Data Structures

```
# Queue for file splitting
FileSplitQ → List
  Format: JSON string of campaign metadata

# Delivery Queues (multiple based on volume)
LowVolumeDQ → List (campaign IDs)
MediumVolumeDQ → List (campaign IDs)
HighVolumeDQ → List (campaign IDs)

# Campaign-specific queues
{cm_id}:Q → List
  Format: JSON string of split file metadata

# Processing queues (for retry safety)
FP:processing:{cm_id}:Q → List
  Format: Temporary copy during processing

# Exclude queues
LowVolumeDQ_exclude → List
MediumVolumeDQ_exclude → List
HighVolumeDQ_exclude → List

# File tracking (for cleanup)
uploaded_files:{username} → Set
  Format: List of file paths

# Heartbeat monitoring
heartbeat:{module}:{instance}:{thread} → String (timestamp)

# Configuration cache
config_params → Hash
  Fields: Key-value pairs of configuration
```

---

## 📈 Queue Architecture Details

### Queue Selection Logic

```java
// Based on total count in campaign
if (total < 10000) {
    queueName = "LowVolumeDQ";
    priority = 1;  // High priority
} else if (total < 100000) {
    queueName = "MediumVolumeDQ";
    priority = 2;  // Medium priority
} else {
    queueName = "HighVolumeDQ";
    priority = 4;  // Low priority
}

// Add exclude suffix if needed
if (hasExcludeGroups) {
    queueName = queueName + "_exclude";
}
```

### Retry Mechanism

```java
// Configuration
MAX_RETRY_COUNT = 5  // Configurable

// On failure
if (retry_count <= MAX_RETRY_COUNT) {
    retry_count++;
    // Push back to queue with incremented retry_count
    pushBackToQueue(metadata, retry_count);
} else {
    // Max retries exceeded
    updateStatus(id, "FAILED", "Max retries exceeded");
}
```

### Atomic Operations

```java
// RPOPLPUSH ensures atomicity
// Campaign ID stays in DeliveryQ until processing completes
String campIdQueue = jedis.rpoplpush(deliveryQ, deliveryQ);

// Get metadata and move to processing queue
String metadata = jedis.rpoplpush(
    campIdQueue, 
    "FP:processing:" + campIdQueue
);

// Process data...

// On success: Remove from processing queue
jedis.lrem("FP:processing:" + campIdQueue, 0, metadata);

// On failure: Push back to original queue
jedis.lpush(campIdQueue, metadata);
jedis.lpush(deliveryQ, campIdQueue);
```

---

## 🔍 Monitoring & Observability

### Heartbeat Monitoring

```java
// Each consumer thread pushes heartbeat every iteration
HeartBeatMonitoring.pushConsumersHeartBeat(
    "FP-SplitStage",           // Module name
    "FileSplitQConsumer",      // Consumer name
    instanceId,                // Instance ID
    threadName,                // Thread name
    timestamp                  // Current timestamp
);

// Stored in Redis as:
// heartbeat:FP-SplitStage:instance-1:FileSplitQConsumer-Thread-1 → "2024-01-15 10:30:45"
```

### Logging

```java
// Module-specific loggers
SplitStageLog.getInstance().debug("Processing file: " + filename);
HandoverStageLog.getInstance().error("Failed to process", exception);

// Log4j2 configuration with separate log files per module
// - split-stage.log
// - handover-stage.log
// - initial-stage.log
// etc.
```

### Metrics (Prometheus)

```java
// Exposed on /metrics endpoint
// - Consumer lag
// - Queue sizes
// - Processing rates
// - Error rates
// - Retry counts
```

---

## 🚀 Deployment Architecture

### Docker Deployment

```yaml
# docker-compose-singleton.yml
services:
  fileprocessor-singleton:
    build:
      context: .
      dockerfile: Dockerfile_singleton
    ports:
      - "8080:8080"
      - "9090:9090"  # Metrics
    environment:
      - INSTANCE_ID=fp-singleton-01
      - REDIS_HOST=redis
      - DB_HOST=mariadb
    volumes:
      - ./properties:/app/properties
      - ./files:/app/files
```

### Scaling

```
┌───────────────────────────────────────────────────────────┐
│                    SCALING ARCHITECTURE                    │
└───────────────────────────────────────────────────────────┘

Load Balancer
      │
      ├──────────────┬──────────────┬──────────────┐
      ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Upload   │  │ Upload   │  │ Upload   │  │ Upload   │
│ Instance │  │ Instance │  │ Instance │  │ Instance │
│    1     │  │    2     │  │    3     │  │    4     │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
      │              │              │              │
      └──────────────┴──────────────┴──────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │  Redis   │
                    │ FileSplitQ│
                    └─────┬────┘
                          │
      ┌───────────────────┼───────────────────┐
      ▼                   ▼                   ▼
┌──────────┐        ┌──────────┐        ┌──────────┐
│ Split    │        │ Split    │        │ Split    │
│ Instance │        │ Instance │        │ Instance │
│    1     │        │    2     │        │    3     │
└──────────┘        └──────────┘        └──────────┘
      │                   │                   │
      └───────────────────┴───────────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │  Redis   │
                    │ DeliveryQ│
                    └─────┬────┘
                          │
      ┌───────────────────┼───────────────────┐
      ▼                   ▼                   ▼
┌──────────┐        ┌──────────┐        ┌──────────┐
│ Handover │        │ Handover │        │ Handover │
│ Instance │        │ Instance │        │ Instance │
│    1     │        │    2     │        │    3     │
└──────────┘        └──────────┘        └──────────┘

Note: Each stage can scale independently based on load
```

---

## 🔐 Security Considerations

1. **File Validation**: Validates file types, sizes, and content
2. **SQL Injection Prevention**: Uses PreparedStatements
3. **Path Traversal Prevention**: Validates file paths
4. **User Isolation**: Files stored in user-specific directories
5. **DLT Compliance**: Validates DLT entity and template IDs
6. **Rate Limiting**: Configurable processing rates
7. **Retry Limits**: Prevents infinite retry loops

---

## 🎯 Best Practices & Configuration

### Key Configuration Parameters

```properties
# Split limit (records per split file)
SMS_SPLIT_LIMIT=10000

# Retry configuration
MAX_RETRY_COUNT=5

# Queue names
FILE_SPLIT_QUEUE_NAME=FileSplitQ
GROUP_QUEUE_NAME=GroupQ

# File paths
FILE_STORE_PATH=/app/files/
CAMPAIGNS_FILE_STORE_PATH=/app/files/campaigns/
GROUP_FILE_STORE_PATH=/app/files/groups/

# Delimiters
SPLIT_FILE_DELIMITER=|
LINE_BREAK_REPLACER=<br>

# Sleep times (milliseconds)
CONSUMER_SLEEP_TIME=1000
THREAD_SLEEP_TIME=5000

# Processing delays
DE_NEXT_REQUEST_POP_DELAY=1000
EE_NEXT_REQUEST_POP_DELAY=1000
```

---

## 📝 Summary

The Beacon File Processor is a robust, scalable system for processing SMS campaigns with the following characteristics:

✅ **Distributed Architecture**: Uses Redis queues for decoupled processing
✅ **Multi-Stage Pipeline**: Upload → Poll → Split → Exclude → Handover
✅ **Fault Tolerant**: Retry mechanisms at every stage
✅ **Scalable**: Each stage can scale independently
✅ **Flexible**: Supports multiple campaign types and file formats
✅ **Observable**: Heartbeat monitoring and comprehensive logging
✅ **Performant**: Parallel processing with configurable thread pools

The system efficiently handles millions of SMS messages daily with high throughput and reliability.

---

**Document Version**: 1.0  
**Last Updated**: 2024-01-15  
**Author**: System Documentation Team
