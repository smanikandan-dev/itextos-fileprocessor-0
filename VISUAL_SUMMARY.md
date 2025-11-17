# Beacon File Processor - Visual Summary

## Complete System at a Glance

```
╔════════════════════════════════════════════════════════════════════════════╗
║                    BEACON FILE PROCESSOR SYSTEM                            ║
║                   Distributed SMS Campaign Processing                      ║
╚════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│                           STAGE 1: FILE UPLOAD                               │
└─────────────────────────────────────────────────────────────────────────────┘

    User/API
       │
       │ POST /save (multipart/form-data)
       ▼
  ┌──────────────────────┐
  │   fb-fileupload      │  ➜  Accepts: CSV, TXT, XLS, XLSX, ZIP
  │   FilesSaver servlet │  ➜  Validates and counts records
  │                      │  ➜  Stores in filesystem
  └──────────┬───────────┘  ➜  Returns metadata JSON
             │
             │ Stores file: /path/{username}/{filename_uuid}.csv
             ▼
       ┌─────────┐
       │ Database│  INSERT INTO campaign_master, campaign_files
       └─────────┘  Status: 'queued'


┌─────────────────────────────────────────────────────────────────────────────┐
│                      STAGE 2: INITIAL POLLING                                │
└─────────────────────────────────────────────────────────────────────────────┘

       ┌─────────┐
       │ Database│  campaign_master (status='queued')
       └────┬────┘  campaign_files (status='queued')
            │
            │ Polls every 1 second
            ▼
  ┌──────────────────────┐
  │  fb-initialstage     │  ➜  CampaignMasterPoller
  │                      │  ➜  Validates user account
  │                      │  ➜  Updates status='inprogress'
  └──────────┬───────────┘
             │
             │ LPUSH
             ▼
      ╔═════════════╗
      ║ FileSplitQ  ║  Redis Queue
      ║ (Redis)     ║  Message: {cm_id, cf_id, fileloc, total, msg, c_type}
      ╚═════════════╝


┌─────────────────────────────────────────────────────────────────────────────┐
│                      STAGE 3: FILE SPLITTING                                 │
└─────────────────────────────────────────────────────────────────────────────┘

      ╔═════════════╗
      ║ FileSplitQ  ║
      ╚═════╤═══════╝
            │ RPOP
            ▼
  ┌──────────────────────┐
  │   fb-splitstage      │  ➜  Reads master file
  │ FileSplitQConsumer   │  ➜  Parses based on format (CSV/XLS/XLSX/TXT)
  │                      │  ➜  Validates data
  │ MasterFileSplitHandler
  └──────────┬───────────┘
             │
             ├─────────────► Splits into chunks (10,000 records/file)
             │               - split_1.txt (10K records)
             │               - split_2.txt (10K records)
             │               - split_3.txt (5K records)
             │
             ├─────────────► INSERT INTO campaign_file_splits
             │               (c_f_id, filename, fileloc, total, status)
             │
             │ Determines queue by volume:
             │  • < 10K      → LowVolumeDeliveryQ    (Priority: 1)
             │  • 10K-100K   → MediumVolumeDeliveryQ (Priority: 2)
             │  • > 100K     → HighVolumeDeliveryQ   (Priority: 4)
             │
             │ LPUSH (for each split file)
             ▼
      ╔══════════════════╗
      ║ DeliveryQ        ║  Redis Queue
      ║ (or ExcludeQ)    ║  Message: {c_f_s_id, cm_id, fileloc, total, PRIORITY}
      ╚══════════════════╝


┌─────────────────────────────────────────────────────────────────────────────┐
│                     STAGE 4: MESSAGE HANDOVER                                │
└─────────────────────────────────────────────────────────────────────────────┘

      ╔══════════════════╗
      ║   DeliveryQ      ║
      ║   ├─ cm_123      ║  Queue Structure:
      ║   ├─ cm_456      ║  1. Campaign ID queue
      ║   └─ cm_789      ║  2. Split file metadata per campaign
      ╚══════╤═══════════╝
             │ RPOPLPUSH (atomic)
             ▼
  ┌──────────────────────┐
  │  fb-handoverstage    │  ➜  SplitFileConsumer
  │                      │  ➜  Determines campaign type
  │                      │  ➜  Processes based on type:
  └──────────┬───────────┘
             │
             ├──[OTM]────► ProcessOTM
             │             • Reads mobile numbers
             │             • Single message for all
             │             • Validates mobiles
             │
             ├──[MTM]────► ProcessMTM
             │             • Reads mobile~message pairs
             │             • Different message per mobile
             │             • Handles delimiters
             │
             ├──[TEM]────► ProcessTEM
             │             • Reads template data
             │             • Replaces placeholders
             │             • Dynamic message generation
             │
             │ For each message:
             │ Build Kafka message:
             │ {
             │   mobile: "919876543210",
             │   message: "Your message",
             │   header: "SMSHEADER",
             │   cli_id: "123456",
             │   campaign_id: "cm_12345"
             │ }
             │
             │ PUSH to Kafka
             ▼
      ╔═══════════════════╗
      ║  Kafka Topic      ║  Message Broker
      ║  sms-delivery     ║  ➜ Delivery Engine picks up
      ╚═══════════════════╝  ➜ SMS gateway processes
                             ➜ Delivery reports tracked


┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 5: COMPLETION TRACKING                              │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────┐
  │ fb-campaignfinisher  │  ➜  PollerCampaignFilesCompleted
  │                      │  ➜  Checks all split files status
  │                      │  ➜  Updates campaign_files status='completed'
  │                      │  ➜  Updates campaign_master status='completed'
  │                      │  ➜  Cleans up Redis queues
  └──────────┬───────────┘
             │
             │ UPDATE campaign_file_splits
             │ SET status='completed'
             │
             ▼
       ┌─────────┐
       │ Database│  Final Status: 'completed'
       └─────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                        PARALLEL PROCESSING PATHS                           ║
╚════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│  GROUP PROCESSING PATH                                                       │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────┐
    │ Database│  groups_master (status='queued')
    └────┬────┘  group_contacts
         │
         │ Polls every 1 second
         ▼
  ┌──────────────────────┐
  │ fb-groupsprocessor   │  ➜  GroupsMasterPoller
  │                      │  ➜  Retrieves group members
  │                      │  ➜  Generates campaign file
  └──────────┬───────────┘
             │
             ├────► INSERT INTO campaign_files
             │
             │ LPUSH
             ▼
      ╔═════════════╗
      ║   GroupQ    ║  → Joins main flow at FileSplitQ
      ╚═════════════╝


┌─────────────────────────────────────────────────────────────────────────────┐
│  EXCLUDE PROCESSING PATH                                                     │
└─────────────────────────────────────────────────────────────────────────────┘

      ╔═════════════╗
      ║  ExcludeQ   ║  (when exclude_group_ids present)
      ╚═════╤═══════╝
            │ RPOP
            ▼
  ┌──────────────────────┐
  │ fb-excludeprocessor  │  ➜  Reads split file
  │                      │  ➜  Loads exclude group members
  │                      │  ➜  Filters out excluded numbers
  └──────────┬───────────┘
             │
             ├────► Writes filtered file
             │
             │ LPUSH
             ▼
      ╔══════════════════╗
      ║   DeliveryQ      ║  → Continues to handover stage
      ╚══════════════════╝


┌─────────────────────────────────────────────────────────────────────────────┐
│  DLT PROCESSING PATH                                                         │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────┐
    │ Database│  dlt_template_request (status='queued')
    └────┬────┘
         │
         │ Polls
         ▼
  ┌──────────────────────┐
  │ fb-dltfileprocessor  │  ➜  DltTemplateRequestPoller
  │                      │  ➜  Validates DLT templates
  │                      │  ➜  Parses DLT files
  └──────────┬───────────┘
             │
             │ UPDATE dlt_template_master
             ▼
    ┌─────────┐
    │ Database│  Template linked to campaigns
    └─────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                            CAMPAIGN TYPES                                  ║
╚════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│  OTM (One-to-Many)                                                           │
│  ─────────────────────                                                       │
│  File Format:                                                                │
│  919876543210                                                                │
│  919876543211                                                                │
│  919876543212                                                                │
│                                                                              │
│  Message: Single message for all recipients                                 │
│  "Dear customer, your balance is Rs. 1000"                                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  MTM (Many-to-Many)                                                          │
│  ───────────────────                                                         │
│  File Format: (mobile~message)                                              │
│  919876543210~Dear John, your order #123 is confirmed                       │
│  919876543211~Dear Jane, your order #456 is confirmed                       │
│  919876543212~Dear Bob, your order #789 is confirmed                        │
│                                                                              │
│  Message: Different message for each recipient                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  TEM (Template-based)                                                        │
│  ─────────────────────                                                       │
│  File Format: (CSV with headers)                                            │
│  MOBILE,NAME,AMOUNT                                                          │
│  919876543210,John,1000                                                      │
│  919876543211,Jane,2000                                                      │
│  919876543212,Bob,3000                                                       │
│                                                                              │
│  Template: "Dear {NAME}, your account has Rs. {AMOUNT}"                     │
│                                                                              │
│  Result Messages:                                                            │
│  • "Dear John, your account has Rs. 1000"                                   │
│  • "Dear Jane, your account has Rs. 2000"                                   │
│  • "Dear Bob, your account has Rs. 3000"                                    │
└─────────────────────────────────────────────────────────────────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                         ERROR HANDLING FLOW                                ║
╚════════════════════════════════════════════════════════════════════════════╝

    ┌──────────────┐
    │ Error Occurs │
    └──────┬───────┘
           │
           ▼
    ┌──────────────────┐
    │ Check Error Type │
    └──────┬───────────┘
           │
           ├─[Deadlock]──────────────► Don't increment retry_count
           │                           Push back to queue immediately
           │
           ├─[File Not Found]────────► Update status='FAILED'
           │                           Reason: 'File not found'
           │                           No retry
           │
           ├─[Invalid Format]────────► Update status='INVALIDFILE'
           │                           Reason: 'Invalid file format'
           │                           No retry
           │
           └─[Other Errors]──────────► Increment retry_count
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ retry_count <=  │
                              │  MAX_RETRY?     │
                              └────┬────────┬───┘
                                   │        │
                              [YES]│        │[NO]
                                   │        │
                                   ▼        ▼
                           ┌────────────┐  ┌────────────────┐
                           │ Push back  │  │ Update status= │
                           │ to queue   │  │ 'FAILED'       │
                           │ for retry  │  │ Send notification
                           └────────────┘  │ No more retries│
                                           └────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                        QUEUE ARCHITECTURE                                  ║
╚════════════════════════════════════════════════════════════════════════════╝

Redis Queue Hierarchy:

╔═══════════════════════════════════════════════════════════════════════════╗
║ FileSplitQ                                                                ║
║ ─────────────────────────────────────────────────────────────────────────║
║ [0] {"cm_id":"cm_123", "cf_id":"cf_456", "fileloc":"...", "total":"1000"}║
║ [1] {"cm_id":"cm_124", "cf_id":"cf_457", "fileloc":"...", "total":"2000"}║
║ [2] {"cm_id":"cm_125", "cf_id":"cf_458", "fileloc":"...", "total":"3000"}║
╚═══════════════════════════════════════════════════════════════════════════╝
       │
       │ Consumed by fb-splitstage
       ▼
╔═══════════════════════════════════════════════════════════════════════════╗
║ DeliveryQ (Campaign ID Queue)                                             ║
║ ─────────────────────────────────────────────────────────────────────────║
║ [0] cm_123                                                                ║
║ [1] cm_456                                                                ║
║ [2] cm_789                                                                ║
╚═══════════════════════════════════════════════════════════════════════════╝
       │
       │ RPOPLPUSH → Get campaign ID
       ▼
╔═══════════════════════════════════════════════════════════════════════════╗
║ cm_123 (Split Files for Campaign 123)                                    ║
║ ─────────────────────────────────────────────────────────────────────────║
║ [0] {"c_f_s_id":"789", "fileloc":"split_1.txt", "total":"10000"}        ║
║ [1] {"c_f_s_id":"790", "fileloc":"split_2.txt", "total":"10000"}        ║
║ [2] {"c_f_s_id":"791", "fileloc":"split_3.txt", "total":"5000"}         ║
╚═══════════════════════════════════════════════════════════════════════════╝
       │
       │ RPOPLPUSH → Move to processing queue
       ▼
╔═══════════════════════════════════════════════════════════════════════════╗
║ processing:cm_123 (Safety Queue)                                         ║
║ ─────────────────────────────────────────────────────────────────────────║
║ [0] {"c_f_s_id":"789", "fileloc":"split_1.txt", "total":"10000"}        ║
╚═══════════════════════════════════════════════════════════════════════════╝
       │
       │ After successful processing → LREM (remove)
       │ If crash → RPOPLPUSH back to main queue
       ▼
   Processing
   Completed


╔════════════════════════════════════════════════════════════════════════════╗
║                        MODULE DEPENDENCIES                                 ║
╚════════════════════════════════════════════════════════════════════════════╝

                          ┌──────────────┐
                          │  fb-utils    │  (Shared by ALL)
                          │              │
                          │ - DB Conn    │
                          │ - Redis Conn │
                          │ - Validators │
                          │ - Constants  │
                          └──────────────┘
                                 ▲
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
    ┌───────┴───────┐    ┌───────┴───────┐    ┌─────┴─────┐
    │ fb-fileupload │    │fb-initialstage│    │fb-splitstage
    └───────────────┘    └───────────────┘    └───────────┘
                                                      │
                                                      ▼
                         ┌────────────────────────────────────┐
                         │       fb-handoverstage             │
                         └────────────────────────────────────┘
                                       │
                         ┌─────────────┼─────────────┐
                         │             │             │
                  ┌──────┴──────┐ ┌───┴────┐ ┌──────┴──────┐
                  │fb-groups    │ │fb-dlt  │ │fb-exclude   │
                  │processor    │ │processor│ │processor    │
                  └─────────────┘ └────────┘ └─────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                            KEY STATISTICS                                  ║
╚════════════════════════════════════════════════════════════════════════════╝

┌────────────────────────────────────────────────────────────────────────────┐
│ Processing Capacity                                                         │
│ ───────────────────                                                         │
│ • Default Split Size:    10,000 records/file                               │
│ • Max Retry Count:       5 attempts (configurable)                         │
│ • Consumer Sleep:        1 second (no data)                                │
│ • Thread Sleep:          5 seconds (on error)                              │
│ • Heartbeat Interval:    10 seconds                                        │
│ • Next Request Delay:    1 second                                          │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Queue Priority Assignment                                                   │
│ ─────────────────────────                                                   │
│ • Low Volume (<10K):     Priority 1 (High)                                 │
│ • Medium Volume (10K-100K): Priority 2-3 (Medium)                          │
│ • High Volume (>100K):   Priority 4-5 (Low)                                │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Database Tables Count                                                       │
│ ─────────────────────                                                       │
│ • campaign_master:       1 record per campaign                             │
│ • campaign_files:        1+ records per campaign (multiple files)          │
│ • campaign_file_splits:  N records (10K records each)                      │
│ • groups_master:         1 record per group                                │
│ • group_contacts:        N records per group                               │
└────────────────────────────────────────────────────────────────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                          MONITORING POINTS                                 ║
╚════════════════════════════════════════════════════════════════════════════╝

┌─ Redis Queue Lengths ─────────────────────────────────────────────────────┐
│  LLEN FileSplitQ              → Should be decreasing                       │
│  LLEN DeliveryQ               → Should be processing                       │
│  LLEN processing:*            → Should be minimal (active processing)      │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Database Status Checks ──────────────────────────────────────────────────┐
│  SELECT COUNT(*) FROM campaign_master WHERE status='queued'                │
│  SELECT COUNT(*) FROM campaign_master WHERE status='inprogress'            │
│  SELECT COUNT(*) FROM campaign_master WHERE status='completed'             │
│  SELECT COUNT(*) FROM campaign_master WHERE status='failed'                │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Heartbeat Monitoring ────────────────────────────────────────────────────┐
│  GET FP-SplitStage:FileSplitQConsumer:instance-01:thread-01               │
│  → Returns timestamp (check if recent)                                     │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Error Rate Monitoring ───────────────────────────────────────────────────┐
│  SELECT COUNT(*) FROM campaign_files WHERE retry_count > 3                 │
│  → High count indicates issues                                             │
└────────────────────────────────────────────────────────────────────────────┘


╔════════════════════════════════════════════════════════════════════════════╗
║                              SUMMARY                                       ║
╚════════════════════════════════════════════════════════════════════════════╝

The Beacon File Processor is a robust, scalable bulk SMS campaign processing
system with the following key characteristics:

✓ Modular Architecture:    14+ independent modules
✓ Queue-based Processing:  Redis for message queuing
✓ Parallel Processing:     Multiple consumers per queue
✓ Reliability:             Automatic retry with configurable limits
✓ Atomic Operations:       RPOPLPUSH for queue safety
✓ Crash Recovery:          Processing queues prevent data loss
✓ Flexible Campaign Types: OTM, MTM, Template, Quick, Group
✓ File Format Support:     CSV, TXT, XLS, XLSX, ZIP
✓ Monitoring:              Heartbeat and health checks
✓ Scalability:             Horizontal scaling via Docker

Flow Summary:
1. Upload → 2. Poll → 3. Split → 4. Handover → 5. Complete

Average Processing Time:
• Small file (< 10K):     1-5 minutes
• Medium file (10K-100K): 5-30 minutes
• Large file (> 100K):    30+ minutes

Key Success Factors:
• Proper configuration of all services (Redis, DB, Kafka)
• Adequate system resources (CPU, RAM, Storage)
• Regular monitoring and maintenance
• Timely cleanup of old files and logs

For detailed information, refer to:
• SYSTEM_DOCUMENTATION.md     → Complete technical reference
• FLOW_DIAGRAMS.md            → Detailed process flows
• QUICK_REFERENCE_GUIDE.md    → Operational guide
• README_DOCUMENTATION.md     → Getting started guide

═══════════════════════════════════════════════════════════════════════════════
                         END OF VISUAL SUMMARY
═══════════════════════════════════════════════════════════════════════════════
