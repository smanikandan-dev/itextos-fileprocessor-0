# Beacon File Processor - Detailed Flow Diagrams

## Table of Contents
1. [Campaign Processing Sequence](#campaign-processing-sequence)
2. [Module Interaction Diagram](#module-interaction-diagram)
3. [File Upload Flow](#file-upload-flow)
4. [Split Stage Flow](#split-stage-flow)
5. [Handover Stage Flow](#handover-stage-flow)
6. [Group Processing Flow](#group-processing-flow)
7. [Error Handling Flow](#error-handling-flow)
8. [Queue Management Flow](#queue-management-flow)

---

## Campaign Processing Sequence

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  User/   │     │   File   │     │ Initial  │     │  Split   │     │ Handover │
│   API    │     │  Upload  │     │  Stage   │     │  Stage   │     │  Stage   │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │                │
     │  Upload File   │                │                │                │
     │───────────────>│                │                │                │
     │                │                │                │                │
     │                │ Save File      │                │                │
     │                │ Count Records  │                │                │
     │                │ Store Metadata │                │                │
     │                │                │                │                │
     │  Response      │                │                │                │
     │<───────────────│                │                │                │
     │  {total:1000}  │                │                │                │
     │                │                │                │                │
     │                │                │                │                │
     │  Create Campaign (via UI/API)    │                │                │
     │─────────────────────────────────>│                │                │
     │                │                │                │                │
     │                │    Insert into campaign_master  │                │
     │                │    Insert into campaign_files   │                │
     │                │    Status: 'queued'             │                │
     │                │                │                │                │
     │                │                │  Poll DB       │                │
     │                │                │  (every 1s)    │                │
     │                │                │  SELECT WHERE  │                │
     │                │                │  status='queued'                │
     │                │                │                │                │
     │                │                │  Found Data    │                │
     │                │                │  Update status │                │
     │                │                │  ='inprogress' │                │
     │                │                │                │                │
     │                │                │  Push to       │                │
     │                │                │  FileSplitQ    │                │
     │                │                │───────────────>│                │
     │                │                │                │                │
     │                │                │                │ Consume from   │
     │                │                │                │ FileSplitQ     │
     │                │                │                │ (RPOP)         │
     │                │                │                │                │
     │                │                │                │ Parse File     │
     │                │                │                │ Split into     │
     │                │                │                │ Chunks         │
     │                │                │                │ (10K records)  │
     │                │                │                │                │
     │                │                │                │ Insert into    │
     │                │                │                │ campaign_file_ │
     │                │                │                │ splits         │
     │                │                │                │                │
     │                │                │                │ Push to        │
     │                │                │                │ DeliveryQ      │
     │                │                │                │───────────────>│
     │                │                │                │                │
     │                │                │                │                │ Consume
     │                │                │                │                │ (RPOPLPUSH)
     │                │                │                │                │
     │                │                │                │                │ Read Split
     │                │                │                │                │ File
     │                │                │                │                │
     │                │                │                │                │ Process
     │                │                │                │                │ by Type
     │                │                │                │                │ (OTM/MTM/
     │                │                │                │                │  TEM)
     │                │                │                │                │
     │                │                │                │                │ Push to
     │                │                │                │                │ Kafka
     │                │                │                │                │────────>
     │                │                │                │                │
     │                │                │                │                │ Update
     │                │                │                │                │ Status
     │                │                │                │                │ ='completed'
     │                │                │                │                │
     │                │                │  Check All     │                │
     │                │                │  Files Done?   │                │
     │                │                │  If YES:       │                │
     │                │                │  Update        │                │
     │                │                │  campaign_     │                │
     │                │                │  master        │                │
     │                │                │  status=       │                │
     │                │                │  'completed'   │                │
     │                │                │                │                │
```

---

## Module Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MODULE INTERACTION ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────────────┘

                                    ┌──────────────┐
                                    │   Database   │
                                    │   (MariaDB)  │
                                    │              │
                                    │ - campaign_  │
                                    │   master     │
                                    │ - campaign_  │
                                    │   files      │
                                    │ - campaign_  │
                                    │   file_splits│
                                    └──────┬───────┘
                                           │
                         Read/Write        │        Read/Write
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
         ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
         │ fb-initialstage│    │  fb-splitstage │    │fb-handoverstage│
         │                │    │                │    │                │
         │ - Polls DB     │    │ - Splits Files │    │ - Processes    │
         │ - Validates    │    │ - Inserts to DB│    │   Split Files  │
         │ - Pushes to Q  │    │ - Pushes to Q  │    │ - Pushes to    │
         │                │    │                │    │   Kafka        │
         └───────┬────────┘    └───────┬────────┘    └───────┬────────┘
                 │                     │                     │
                 │ LPUSH               │ LPUSH               │ RPOPLPUSH
                 │                     │                     │
                 ▼                     ▼                     ▼
         ┌──────────────────────────────────────────────────────────┐
         │                    REDIS (Queue Layer)                    │
         │                                                            │
         │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
         │  │ FileSplitQ   │  │  DeliveryQ   │  │  ExcludeQ    │   │
         │  │              │  │  - Low       │  │              │   │
         │  │              │  │  - Medium    │  │              │   │
         │  │              │  │  - High      │  │              │   │
         │  └──────────────┘  └──────────────┘  └──────────────┘   │
         │                                                            │
         │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
         │  │   GroupQ     │  │ cm_id_queue  │  │ processing:  │   │
         │  │              │  │              │  │ cm_id_queue  │   │
         │  │              │  │              │  │              │   │
         │  └──────────────┘  └──────────────┘  └──────────────┘   │
         └──────────────────────────────────────────────────────────┘
                 │                     │                     │
                 │ RPOP                │ RPOP                │ RPOP
                 │                     │                     │
                 ▼                     ▼                     ▼
         ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
         │fb-groupsprocessor   │fb-excludeprocessor   │fb-campaignfinisher
         │                │    │                │    │                │
         │ - Processes    │    │ - Filters      │    │ - Marks        │
         │   Groups       │    │   Excludes     │    │   Completed    │
         │ - Generates    │    │ - Pushes to    │    │ - Cleans Redis │
         │   Campaign     │    │   DeliveryQ    │    │                │
         └────────────────┘    └────────────────┘    └────────────────┘


                    ┌────────────────────────────────┐
                    │        fb-utils (Shared)       │
                    │                                │
                    │ - Database Connections         │
                    │ - Redis Connections            │
                    │ - JSON Utilities               │
                    │ - Validators                   │
                    │ - Constants                    │
                    │ - Logging                      │
                    └────────────────────────────────┘
                              ▲  ▲  ▲  ▲  ▲
                              │  │  │  │  │
                    Used by All Modules (Dependency)


                    ┌────────────────────────────────┐
                    │      fb-fileupload (HTTP)      │
                    │                                │
                    │ - POST /save                   │
                    │ - Receives File Upload         │
                    │ - Stores File                  │
                    │ - Returns Metadata             │
                    └────────────────────────────────┘
                                ▲
                                │ HTTP
                                │
                    ┌────────────────────────────────┐
                    │      User / External API       │
                    └────────────────────────────────┘


                    ┌────────────────────────────────┐
                    │     Kafka / Delivery Engine    │
                    │                                │
                    │ - Receives Messages            │
                    │ - Sends SMS                    │
                    │ - Tracks Delivery              │
                    └────────────────────────────────┘
                                ▲
                                │ Kafka Producer
                                │
                    ┌────────────────────────────────┐
                    │      fb-handoverstage          │
                    └────────────────────────────────┘
```

---

## File Upload Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         FILE UPLOAD DETAILED FLOW                         │
└──────────────────────────────────────────────────────────────────────────┘

User/API                    FilesSaver Servlet              Filesystem/Redis
   │                              │                              │
   │  HTTP POST /save             │                              │
   │  multipart/form-data         │                              │
   │  - username                  │                              │
   │  - frompage (campaign/group) │                              │
   │  - file(s)                   │                              │
   │─────────────────────────────>│                              │
   │                              │                              │
   │                              │ Validate Parameters          │
   │                              │ - username required?         │
   │                              │ - frompage required?         │
   │                              │                              │
   │                              │ Determine File Store Path    │
   │                              │ based on frompage:           │
   │                              │ - campaign → CAMPAIGNS_PATH  │
   │                              │ - group → GROUP_PATH         │
   │                              │                              │
   │                              │ Create Directory             │
   │                              │ {path}/{username}/           │
   │                              │─────────────────────────────>│
   │                              │                              │
   │                              │ For Each Part (File):        │
   │                              │                              │
   │                              │ 1. Extract Original Filename │
   │                              │    (e.g., "contacts.csv")    │
   │                              │                              │
   │                              │ 2. Generate UUID             │
   │                              │    uuid = UUID.randomUUID()  │
   │                              │                              │
   │                              │ 3. Create Stored Filename    │
   │                              │    "contacts_uuid-xxx.csv"   │
   │                              │                              │
   │                              │ 4. Write File to Disk        │
   │                              │─────────────────────────────>│
   │                              │                              │
   │                              │ 5. If ZIP File:              │
   │                              │    Extract Contents          │
   │                              │    ├─ file1.csv              │
   │                              │    ├─ file2.txt              │
   │                              │    └─ file3.xlsx             │
   │                              │                              │
   │                              │ 6. Start FileReadService     │
   │                              │    Thread for Each File      │
   │                              │    (Parallel Processing)     │
   │                              │                              │
   │                              ├─> Thread 1: Read file1.csv   │
   │                              │   - Determine file type      │
   │                              │   - Get Parser (CSV/XLS/TXT) │
   │                              │   - Count Records            │
   │                              │   - Return count             │
   │                              │                              │
   │                              ├─> Thread 2: Read file2.txt   │
   │                              │   - Count lines              │
   │                              │   - Validate format          │
   │                              │   - Return count             │
   │                              │                              │
   │                              ├─> Thread 3: Read file3.xlsx  │
   │                              │   - Parse Excel              │
   │                              │   - Count rows               │
   │                              │   - Return count             │
   │                              │                              │
   │                              │ Wait for All Threads         │
   │                              │ to Complete                  │
   │                              │                              │
   │                              │ Push Filenames to Redis      │
   │                              │ for Cleanup Tracking         │
   │                              │─────────────────────────────>│
   │                              │ LPUSH tracking_redis         │
   │                              │ Key: {frompage}:{username}   │
   │                              │                              │
   │                              │ Aggregate Results            │
   │                              │ - Success files              │
   │                              │ - Failed files               │
   │                              │ - Total count                │
   │                              │                              │
   │                              │ Build JSON Response          │
   │                              │ {                            │
   │                              │   "statusCode": 200,         │
   │                              │   "total": 15000,            │
   │                              │   "total_human": "15.0K",    │
   │                              │   "uploaded_files": {        │
   │                              │     "success": [             │
   │                              │       {                      │
   │                              │         "filename": "c.csv", │
   │                              │         "r_filename": "c_...",
   │                              │         "count": 15000       │
   │                              │       }                      │
   │                              │     ],                       │
   │                              │     "failed": []             │
   │                              │   }                          │
   │                              │ }                            │
   │                              │                              │
   │  HTTP 200 OK                 │                              │
   │  JSON Response               │                              │
   │<─────────────────────────────│                              │
   │                              │                              │

Error Handling:
───────────────
If any error occurs:
   │                              │
   │                              │ Catch Exception              │
   │                              │ - Log error                  │
   │                              │ - Push files to Redis        │
   │                              │   (for cleanup)              │
   │                              │ - Set HTTP 500               │
   │                              │ - Build error response       │
   │                              │                              │
   │  HTTP 500 Error              │                              │
   │  {                           │                              │
   │    "statusCode": 500,        │                              │
   │    "error": "Internal Error",│                              │
   │    "message": "..."          │                              │
   │  }                           │                              │
   │<─────────────────────────────│                              │
```

---

## Split Stage Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SPLIT STAGE DETAILED FLOW                          │
└──────────────────────────────────────────────────────────────────────────┘

FileSplitQ           FileSplitQConsumer        MasterFileSplitHandler        DB/Filesystem
   │                        │                           │                          │
   │  Message: {            │                           │                          │
   │    cm_id: "123",       │                           │                          │
   │    cf_id: "456",       │                           │                          │
   │    fileloc: "...",     │                           │                          │
   │    total: "50000",     │                           │                          │
   │    c_type: "OTM"       │                           │                          │
   │  }                     │                           │                          │
   │                        │                           │                          │
   │<───────── RPOP ────────│                           │                          │
   │                        │                           │                          │
   │                        │ Parse JSON                │                          │
   │                        │ Convert to Map            │                          │
   │                        │                           │                          │
   │                        │ Create Handler            │                          │
   │                        │ Pass Map                  │                          │
   │                        │──────────────────────────>│                          │
   │                        │                           │                          │
   │                        │                           │ Get User Details         │
   │                        │                           │ (cli_id)                 │
   │                        │                           │ - platform_cluster       │
   │                        │                           │ - unicode settings       │
   │                        │                           │                          │
   │                        │                           │ Check if Already Split   │
   │                        │                           │ SELECT FROM              │
   │                        │                           │ campaign_file_splits     │
   │                        │                           │ WHERE c_f_id=?           │
   │                        │                           │─────────────────────────>│
   │                        │                           │                          │
   │                        │                           │ If Records Found:        │
   │                        │                           │ - Use existing splits    │
   │                        │                           │ - Skip file processing   │
   │                        │                           │                          │
   │                        │                           │ If NOT Found:            │
   │                        │                           │ ┌──────────────────────┐ │
   │                        │                           │ │  File Splitting      │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 1. Determine Type:   │ │
   │                        │                           │ │    - OTM/QUICK/GROUP │ │
   │                        │                           │ │    - MTM             │ │
   │                        │                           │ │    - TEM (Template)  │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 2. Get File Parser:  │ │
   │                        │                           │ │    FileParserFactory │ │
   │                        │                           │ │    .get(filename)    │ │
   │                        │                           │ │    Returns:          │ │
   │                        │                           │ │    - CsvFileParser   │ │
   │                        │                           │ │    - XlsFileParser   │ │
   │                        │                           │ │    - XlsxFileParser  │ │
   │                        │                           │ │    - TextFileParser  │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 3. Set Parser Config:│ │
   │                        │                           │ │    - delimiter       │ │
   │                        │                           │ │    - split limit     │ │
   │                        │                           │ │    - message         │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 4. Parse Master File:│ │
   │                        │                           │ │    fileParser.parse()│ │
   │                        │                           │ │                      │ │
   │                        │                           │ │    For OTM:          │ │
   │                        │                           │ │    - Read mobiles    │ │
   │                        │                           │ │    - Validate format │ │
   │                        │                           │ │    - Write to split  │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │    For MTM:          │ │
   │                        │                           │ │    - Read mobile~msg │ │
   │                        │                           │ │    - Parse delimiter │ │
   │                        │                           │ │    - Write to split  │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │    For TEM:          │ │
   │                        │                           │ │    - Get template    │ │
   │                        │                           │ │    - Extract fields  │ │
   │                        │                           │ │    - Read data       │ │
   │                        │                           │ │    - Map placeholders│ │
   │                        │                           │ │    - Write to split  │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 5. Split into Chunks:│ │
   │                        │                           │ │    Every 10,000 recs │ │
   │                        │                           │ │    Create new file:  │ │
   │                        │                           │ │    split_1.txt       │ │
   │                        │                           │ │    split_2.txt       │ │
   │                        │                           │ │    ...               │ │
   │                        │                           │ │                      │ │
   │                        │                           │ │ 6. Store Split Files:│ │
   │                        │                           │ │    Write to disk     │ │
   │                        │                           │ └──────────────────────┘ │
   │                        │                           │─────────────────────────>│
   │                        │                           │                          │
   │                        │                           │ Insert Split Records     │
   │                        │                           │ to DB                    │
   │                        │                           │                          │
   │                        │                           │ INSERT INTO              │
   │                        │                           │ campaign_file_splits     │
   │                        │                           │ (c_f_id, filename,       │
   │                        │                           │  fileloc, total,         │
   │                        │                           │  status, ...)            │
   │                        │                           │ VALUES (?, ?, ?, ?, ...) │
   │                        │                           │─────────────────────────>│
   │                        │                           │                          │
   │                        │                           │ Determine Queue:         │
   │                        │                           │ - Check total count      │
   │                        │                           │   < 10K → LowVolumeQ     │
   │                        │                           │   10K-100K → MediumQ     │
   │                        │                           │   > 100K → HighVolumeQ   │
   │                        │                           │                          │
   │                        │                           │ - Check exclude_group_ids│
   │                        │                           │   if present → ExcludeQ  │
   │                        │                           │                          │
   │                        │                           │ Set Priority:            │
   │                        │                           │ - Low Vol: Priority=1    │
   │                        │                           │ - Med Vol: Priority=2    │
   │                        │                           │ - High Vol: Priority=4   │
   │                        │                           │                          │
   │                        │                           │ Push to Redis Queue      │
   │                        │                           │ For Each Split File:     │
   │                        │                           │ {                        │
   │                        │                           │   c_f_s_id: "789",       │
   │                        │                           │   cm_id: "123",          │
   │                        │                           │   fileloc: "split_1.txt",│
   │                        │                           │   total: "10000",        │
   │                        │                           │   PRIORITY: "1"          │
   │                        │                           │ }                        │
   │                        │                           │                          │
   │                        │                           │ LPUSH to DeliveryQ       │
   │                        │                           │ or ExcludeQ              │
   │                        │                           │                          │

Error Handling:
───────────────
   │                        │                           │                          │
   │                        │                           │ If Error Occurs:         │
   │                        │                           │ - Increment retry_count  │
   │                        │                           │ - Check vs MAX_RETRY     │
   │                        │                           │                          │
   │                        │                           │ If retry_count <= MAX:   │
   │                        │                           │ - Push back to FileSplitQ│
   │                        │                           │                          │
   │                        │                           │ If retry_count > MAX:    │
   │                        │                           │ - Update status='FAILED' │
   │                        │                           │ - Update reason          │
   │                        │                           │─────────────────────────>│
   │                        │                           │                          │
```

---

## Handover Stage Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       HANDOVER STAGE DETAILED FLOW                        │
└──────────────────────────────────────────────────────────────────────────┘

DeliveryQ        SplitFileConsumer       ProcessOTM/MTM/TEM      KafkaHandover      Kafka
   │                    │                        │                      │              │
   │  Queue Structure:  │                        │                      │              │
   │  ┌──────────────┐  │                        │                      │              │
   │  │ DeliveryQ    │  │                        │                      │              │
   │  │  ├─ cm_123   │  │                        │                      │              │
   │  │  ├─ cm_456   │  │                        │                      │              │
   │  │  └─ cm_789   │  │                        │                      │              │
   │  │              │  │                        │                      │              │
   │  │ cm_123       │  │                        │                      │              │
   │  │  ├─ split1   │  │                        │                      │              │
   │  │  ├─ split2   │  │                        │                      │              │
   │  │  └─ split3   │  │                        │                      │              │
   │  └──────────────┘  │                        │                      │              │
   │                    │                        │                      │              │
   │<── RPOPLPUSH ──────│                        │                      │              │
   │    (DeliveryQ,     │                        │                      │              │
   │     DeliveryQ)     │                        │                      │              │
   │                    │                        │                      │              │
   │    Returns:        │                        │                      │              │
   │    "cm_123"        │                        │                      │              │
   │                    │                        │                      │              │
   │<── RPOPLPUSH ──────│                        │                      │              │
   │    (cm_123,        │                        │                      │              │
   │     processing:    │                        │                      │              │
   │     cm_123)        │                        │                      │              │
   │                    │                        │                      │              │
   │    Returns:        │                        │                      │              │
   │    split_metadata  │                        │                      │              │
   │                    │                        │                      │              │
   │                    │ Parse JSON             │                      │              │
   │                    │ {                      │                      │              │
   │                    │   c_f_s_id: "789",     │                      │              │
   │                    │   cm_id: "123",        │                      │              │
   │                    │   c_type: "OTM",       │                      │              │
   │                    │   fileloc: "split1.txt"│                      │              │
   │                    │ }                      │                      │              │
   │                    │                        │                      │              │
   │                    │ Determine Campaign Type│                      │              │
   │                    │ Based on c_type        │                      │              │
   │                    │                        │                      │              │
   │                    │ If "OTM" or "QUICK":   │                      │              │
   │                    │────────────────────────>│                      │              │
   │                    │                        │                      │              │
   │                    │                        │ Open Split File      │              │
   │                    │                        │ /path/split1.txt     │              │
   │                    │                        │                      │              │
   │                    │                        │ Read Line by Line:   │              │
   │                    │                        │ 919876543210         │              │
   │                    │                        │ 919876543211         │              │
   │                    │                        │ 919876543212         │              │
   │                    │                        │ ...                  │              │
   │                    │                        │                      │              │
   │                    │                        │ For Each Mobile:     │              │
   │                    │                        │ 1. Validate format   │              │
   │                    │                        │    (10-15 digits)    │              │
   │                    │                        │                      │              │
   │                    │                        │ 2. Build Message:    │              │
   │                    │                        │    {                 │              │
   │                    │                        │      mobile: "91...", │              │
   │                    │                        │      message: "...",  │              │
   │                    │                        │      header: "ABC",   │              │
   │                    │                        │      cli_id: "123",   │              │
   │                    │                        │      campaign_id:"cm_123"           │
   │                    │                        │    }                 │              │
   │                    │                        │                      │              │
   │                    │                        │ 3. Push to Kafka     │              │
   │                    │                        │────────────────────────────────────>│
   │                    │                        │                      │              │
   │                    │                        │ 4. Increment Counter │              │
   │                    │                        │    success_count++   │              │
   │                    │                        │                      │              │
   │                    │                        │ Close File           │              │
   │                    │                        │                      │              │
   │                    │                        │ Return success       │              │
   │                    │                        │<─────────────────────│              │
   │                    │                        │                      │              │
   │                    │ If "MTM":              │                      │              │
   │                    │────────────────────────>│                      │              │
   │                    │                        │                      │              │
   │                    │                        │ Open Split File      │              │
   │                    │                        │                      │              │
   │                    │                        │ Read Line by Line:   │              │
   │                    │                        │ 91987654~Hi {NAME}   │              │
   │                    │                        │ 91987655~Hello Sir   │              │
   │                    │                        │                      │              │
   │                    │                        │ For Each Line:       │              │
   │                    │                        │ 1. Split by delimiter│              │
   │                    │                        │    mobile = parts[0] │              │
   │                    │                        │    message = parts[1]│              │
   │                    │                        │                      │              │
   │                    │                        │ 2. Replace linebreaks│              │
   │                    │                        │    {br} → \n         │              │
   │                    │                        │                      │              │
   │                    │                        │ 3. Unicode check     │              │
   │                    │                        │    if unicode:       │              │
   │                    │                        │      toHex(message)  │              │
   │                    │                        │                      │              │
   │                    │                        │ 4. Build & Push      │              │
   │                    │                        │────────────────────────────────────>│
   │                    │                        │                      │              │
   │                    │ If "TEM" (Template):   │                      │              │
   │                    │────────────────────────>│                      │              │
   │                    │                        │                      │              │
   │                    │                        │ Get Template Details │              │
   │                    │                        │ template_id: "T123"  │              │
   │                    │                        │ msg: "Hi {NAME}"     │              │
   │                    │                        │ mobile_col: "MOBILE" │              │
   │                    │                        │                      │              │
   │                    │                        │ Open Split File      │              │
   │                    │                        │ Header:              │              │
   │                    │                        │ MOBILE,NAME,AMOUNT   │              │
   │                    │                        │ Data:                │              │
   │                    │                        │ 91987654,John,1000   │              │
   │                    │                        │ 91987655,Jane,2000   │              │
   │                    │                        │                      │              │
   │                    │                        │ For Each Row:        │              │
   │                    │                        │ 1. Parse columns     │              │
   │                    │                        │    mobile="91987654" │              │
   │                    │                        │    NAME="John"       │              │
   │                    │                        │    AMOUNT="1000"     │              │
   │                    │                        │                      │              │
   │                    │                        │ 2. Replace template  │              │
   │                    │                        │    "Hi {NAME}"       │              │
   │                    │                        │    → "Hi John"       │              │
   │                    │                        │                      │              │
   │                    │                        │ 3. Build & Push      │              │
   │                    │                        │────────────────────────────────────>│
   │                    │                        │                      │              │
   │                    │                        │ All Messages Sent    │              │
   │                    │                        │<─────────────────────│              │
   │                    │                        │                      │              │
   │                    │ Remove from Processing │                      │              │
   │                    │ Queue                  │                      │              │
   │────── LREM ────────│                        │                      │              │
   │   (processing:     │                        │                      │              │
   │    cm_123,         │                        │                      │              │
   │    split_metadata) │                        │                      │              │
   │                    │                        │                      │              │
   │                    │ Update Status in DB    │                      │              │
   │                    │ UPDATE                 │                      │              │
   │                    │ campaign_file_splits   │                      │              │
   │                    │ SET status='completed' │                      │              │
   │                    │ WHERE c_f_s_id='789'   │                      │              │
   │                    │                        │                      │              │

Error Handling:
───────────────
   │                    │                        │                      │              │
   │                    │ If Error Occurs:       │                      │              │
   │                    │ - Get retry_count      │                      │              │
   │                    │ - Check vs MAX_RETRY   │                      │              │
   │                    │                        │                      │              │
   │                    │ If retry <= MAX:       │                      │              │
   │                    │ - Increment retry      │                      │              │
   │                    │ - Push back to queue   │                      │              │
   │<─── LPUSH ─────────│   (cm_123, metadata)   │                      │              │
   │                    │                        │                      │              │
   │                    │ If retry > MAX:        │                      │              │
   │                    │ - Update status=FAILED │                      │              │
   │                    │ - Remove from queue    │                      │              │
   │                    │                        │                      │              │
```

---

## Group Processing Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        GROUP PROCESSING FLOW                              │
└──────────────────────────────────────────────────────────────────────────┘

Database         GroupsMasterPoller    GroupsQConsumer    GroupsCampaignFileGenerator
   │                    │                     │                      │
   │                    │                     │                      │
   │ groups_master      │                     │                      │
   │ - id               │                     │                      │
   │ - name             │                     │                      │
   │ - status='queued'  │                     │                      │
   │                    │                     │                      │
   │ group_contacts     │                     │                      │
   │ - group_id         │                     │                      │
   │ - mobile           │                     │                      │
   │                    │                     │                      │
   │<─── Poll (1s) ─────│                     │                      │
   │                    │                     │                      │
   │    SELECT *        │                     │                      │
   │    FROM groups_    │                     │                      │
   │    master          │                     │                      │
   │    WHERE status=   │                     │                      │
   │    'queued'        │                     │                      │
   │                    │                     │                      │
   │ Returns: Group ID  │                     │                      │
   │ group_id: "G123"   │                     │                      │
   │────────────────────>│                     │                      │
   │                    │                     │                      │
   │                    │ Push to GroupQ      │                      │
   │                    │ {                   │                      │
   │                    │   group_id: "G123", │                      │
   │                    │   name: "VIP",      │                      │
   │                    │   ...               │                      │
   │                    │ }                   │                      │
   │                    │─────────────────────>│                      │
   │                    │                     │                      │
   │                    │                     │ RPOP from GroupQ     │
   │                    │                     │ Get group_id         │
   │                    │                     │                      │
   │<──────── SELECT ───────────────────────────────────────────────│
   │    group_contacts  │                     │                      │
   │    WHERE group_id  │                     │                      │
   │    = 'G123'        │                     │                      │
   │                    │                     │                      │
   │ Returns:           │                     │                      │
   │ 919876543210       │                     │                      │
   │ 919876543211       │                     │                      │
   │ 919876543212       │                     │                      │
   │ ...                │                     │                      │
   │────────────────────────────────────────────────────────────────>│
   │                    │                     │                      │
   │                    │                     │                      │ Generate File
   │                    │                     │                      │ /path/group_
   │                    │                     │                      │ G123.txt
   │                    │                     │                      │
   │                    │                     │                      │ Write mobiles:
   │                    │                     │                      │ 919876543210
   │                    │                     │                      │ 919876543211
   │                    │                     │                      │ 919876543212
   │                    │                     │                      │ ...
   │                    │                     │                      │
   │<──── INSERT ───────────────────────────────────────────────────│
   │  campaign_files    │                     │                      │
   │  {                 │                     │                      │
   │    group_id: "G123",                     │                      │
   │    fileloc: "/path/group_G123.txt",      │                      │
   │    total: 10000    │                     │                      │
   │  }                 │                     │                      │
   │                    │                     │                      │
   │                    │                     │                      │ Push to
   │                    │                     │                      │ FileSplitQ
   │                    │                     │                      │ or
   │                    │                     │                      │ GroupsSplitQ
   │                    │                     │                      │
   │                    │                     │                      │ (Same flow as
   │                    │                     │                      │  campaign files)
```

---

## Error Handling Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ERROR HANDLING FLOW                               │
└──────────────────────────────────────────────────────────────────────────┘

Consumer Thread         Error Detection           Retry Logic         Database
      │                        │                       │                  │
      │                        │                       │                  │
      │  Process Request       │                       │                  │
      │  from Queue            │                       │                  │
      │                        │                       │                  │
      │  [ERROR OCCURS]        │                       │                  │
      │─────────────────────────>│                       │                  │
      │                        │                       │                  │
      │                        │ Catch Exception       │                  │
      │                        │ - Log error           │                  │
      │                        │ - Extract metadata    │                  │
      │                        │                       │                  │
      │                        │ Get retry_count       │                  │
      │                        │ from metadata         │                  │
      │                        │                       │                  │
      │                        │ Check Exception Type  │                  │
      │                        │                       │                  │
      │                        ├─ If Deadlock:         │                  │
      │                        │  Don't increment      │                  │
      │                        │  retry_count          │                  │
      │                        │  (database issue)     │                  │
      │                        │                       │                  │
      │                        ├─ If Other Error:      │                  │
      │                        │  Increment            │                  │
      │                        │  retry_count          │                  │
      │                        │                       │                  │
      │                        │ Check Retry Count     │                  │
      │                        │────────────────────────>│                  │
      │                        │                       │                  │
      │                        │                       │ retry_count <=   │
      │                        │                       │ MAX_RETRY?       │
      │                        │                       │                  │
      │                        │                       ├─ YES:            │
      │                        │                       │  ┌────────────┐  │
      │                        │                       │  │ Push back  │  │
      │                        │                       │  │ to Queue   │  │
      │                        │                       │  │            │  │
      │                        │                       │  │ Update     │  │
      │                        │                       │  │ retry_count│  │
      │                        │                       │  │            │  │
      │                        │                       │  │ Will be    │  │
      │                        │                       │  │ retried    │  │
      │                        │                       │  │ later      │  │
      │                        │                       │  └────────────┘  │
      │                        │                       │                  │
      │                        │                       ├─ NO:             │
      │                        │                       │  ┌────────────┐  │
      │                        │                       │  │ Update DB  │  │
      │                        │                       │  │ status =   │  │
      │                        │                       │  │ 'FAILED'   │  │
      │                        │                       │  │            │  │
      │                        │                       │  │ Set reason │  │
      │                        │                       │  │            │  │
      │                        │                       │  │ Remove from│  │
      │                        │                       │  │ queue      │  │
      │                        │                       │  │            │  │
      │                        │                       │  │ Send       │  │
      │                        │                       │  │ notification│  │
      │                        │                       │  └────────────┘  │
      │                        │                       │                  │
      │                        │                       │ UPDATE          │
      │                        │                       │ campaign_files  │
      │                        │                       │ SET status=     │
      │                        │                       │ 'FAILED',       │
      │                        │                       │ reason='...',   │
      │                        │                       │ retry_count=?   │
      │                        │                       │ WHERE id=?      │
      │                        │                       │─────────────────>│
      │                        │                       │                  │
      │  Sleep & Continue      │                       │                  │
      │  to Next Request       │                       │                  │
      │                        │                       │                  │

Specific Error Types:
─────────────────────

1. FileNotFoundException
   → Update status='FAILED'
   → Reason: 'File not found'
   → No retry

2. Database Deadlock
   → Don't increment retry
   → Push back to queue
   → Will be retried immediately

3. Redis Connection Error
   → Reconnect
   → Retry operation
   → If fails again, sleep & retry

4. Kafka Push Failure
   → Increment retry
   → Push back to queue
   → If max retries exceeded, mark as FAILED

5. Invalid File Format
   → Update status='INVALIDFILE'
   → Reason: 'Invalid file format'
   → No retry

6. Zero Records in File
   → Update status='INVALIDFILE'
   → Reason: 'File contains 0 records'
   → No retry

7. Mobile Number Validation Error
   → Skip record
   → Continue with next
   → Log error
   → Decrement total count

8. Template Not Found (TEM)
   → Update status='FAILED'
   → Reason: 'Template not found'
   → No retry
```

---

## Queue Management Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        REDIS QUEUE MANAGEMENT                             │
└──────────────────────────────────────────────────────────────────────────┘

Producer Thread          Redis Queue           Consumer Thread         Backup Logic
      │                       │                       │                      │
      │                       │                       │                      │
      │  Prepare Data         │                       │                      │
      │  Convert to JSON      │                       │                      │
      │                       │                       │                      │
      │  LPUSH                │                       │                      │
      │──────────────────────>│                       │                      │
      │  (Queue Name, JSON)   │                       │                      │
      │                       │                       │                      │
      │                       │  Queue Structure:     │                      │
      │                       │  ┌─────────────────┐  │                      │
      │                       │  │ [0] newest data │  │                      │
      │                       │  │ [1] older data  │  │                      │
      │                       │  │ [2] oldest data │  │                      │
      │                       │  └─────────────────┘  │                      │
      │                       │                       │                      │
      │                       │                       │  RPOP               │
      │                       │                       │<─────────────────────│
      │                       │                       │  (Queue Name)        │
      │                       │                       │                      │
      │                       │  Returns oldest       │                      │
      │                       │  data (FIFO)          │                      │
      │                       │─────────────────────────>│                      │
      │                       │                       │                      │
      │                       │                       │  Process Data        │
      │                       │                       │  - Parse JSON        │
      │                       │                       │  - Execute Logic     │
      │                       │                       │                      │

Atomic Processing (RPOPLPUSH):
───────────────────────────────

Consumer                 Main Queue          Processing Queue        Completion
    │                       │                       │                    │
    │                       │                       │                    │
    │  RPOPLPUSH            │                       │                    │
    │  (mainQ, procQ)       │                       │                    │
    │──────────────────────>│                       │                    │
    │                       │                       │                    │
    │                       │  Atomically:          │                    │
    │                       │  1. POP from mainQ    │                    │
    │                       │  2. PUSH to procQ     │                    │
    │                       │────────────────────────>│                    │
    │                       │                       │                    │
    │  Returns data         │                       │                    │
    │<──────────────────────────────────────────────│                    │
    │                       │                       │                    │
    │  Process safely       │                       │                    │
    │  (data is in procQ,   │                       │                    │
    │   safe if crash)      │                       │                    │
    │                       │                       │                    │
    │  [Processing Success] │                       │                    │
    │                       │                       │                    │
    │  LREM                 │                       │                    │
    │  (procQ, 1, data)     │                       │                    │
    │────────────────────────────────────────────────>│                    │
    │                       │                       │                    │
    │                       │                       │  Remove from       │
    │                       │                       │  processing queue  │
    │                       │                       │                    │
    │                       │                       │  Update DB         │
    │                       │                       │──────────────────────>│
    │                       │                       │  status='completed'│
    │                       │                       │                    │

Crash Recovery:
───────────────

    │  [CRASH OCCURS]       │                       │                    │
    │  ✗                    │                       │                    │
    │                       │                       │                    │
    │  Service Restart      │                       │                    │
    │                       │                       │                    │
    │  Check Processing Q   │                       │                    │
    │────────────────────────────────────────────────>│                    │
    │                       │                       │                    │
    │  Get all items        │                       │                    │
    │  in processing queue  │                       │                    │
    │<────────────────────────────────────────────────│                    │
    │                       │                       │                    │
    │  RPOPLPUSH            │                       │                    │
    │  (procQ, mainQ)       │                       │                    │
    │  Move back to main Q  │                       │                    │
    │<──────────────────────────────────────────────>│                    │
    │                       │                       │                    │
    │  Items will be        │                       │                    │
    │  reprocessed          │                       │                    │
    │                       │                       │                    │

Round-Robin Queue Selection:
────────────────────────────

Producer              Redis Instance 1    Redis Instance 2    Redis Instance 3
    │                       │                   │                   │
    │  Request 1            │                   │                   │
    │──────────────────────>│                   │                   │
    │  Push to Redis 1      │                   │                   │
    │                       │                   │                   │
    │  Request 2            │                   │                   │
    │────────────────────────────────────────────>│                   │
    │  Push to Redis 2      │                   │                   │
    │                       │                   │                   │
    │  Request 3            │                   │                   │
    │──────────────────────────────────────────────────────────────>│
    │  Push to Redis 3      │                   │                   │
    │                       │                   │                   │
    │  Request 4            │                   │                   │
    │──────────────────────>│                   │                   │
    │  Push to Redis 1      │                   │                   │
    │  (round-robin)        │                   │                   │
    │                       │                   │                   │

Queue Monitoring:
─────────────────

    │  Get Queue Length     │                   │                   │
    │  LLEN (queueName)     │                   │                   │
    │──────────────────────>│                   │                   │
    │  Returns: 1523        │                   │                   │
    │<──────────────────────│                   │                   │
    │                       │                   │                   │
    │  Check if Stuck       │                   │                   │
    │  (length not         │                   │                   │
    │   decreasing)         │                   │                   │
    │                       │                   │                   │
    │  Alert if Queue       │                   │                   │
    │  Length > Threshold   │                   │                   │
```

---

## Summary

These diagrams provide a comprehensive view of:

1. **End-to-End Campaign Processing**: From file upload to SMS delivery
2. **Module Interactions**: How different components communicate
3. **Detailed File Upload**: Complete flow with error handling
4. **Split Stage Logic**: File parsing, splitting, and queue management
5. **Handover Processing**: Campaign type-specific processing (OTM/MTM/TEM)
6. **Group Processing**: Special handling for group-based campaigns
7. **Error Handling**: Retry logic and failure scenarios
8. **Queue Management**: Redis queue patterns and atomic operations

Each diagram shows the sequential flow with detailed steps, making it easy to understand:
- **What** each component does
- **When** actions occur
- **How** data flows through the system
- **Why** certain decisions are made
- **Where** errors can occur and how they're handled

These diagrams serve as both documentation and troubleshooting guides for the Beacon File Processor system.
