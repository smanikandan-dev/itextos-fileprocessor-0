# Beacon File Processor - Sequence Diagrams

## Table of Contents
1. [File Upload Sequence](#file-upload-sequence)
2. [Campaign Processing Sequence](#campaign-processing-sequence)
3. [Split File Processing Sequence](#split-file-processing-sequence)
4. [Handover to Delivery Sequence](#handover-to-delivery-sequence)
5. [Error Handling Sequence](#error-handling-sequence)
6. [Campaign Completion Sequence](#campaign-completion-sequence)

---

## File Upload Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FILE UPLOAD SEQUENCE DIAGRAM                           │
└─────────────────────────────────────────────────────────────────────────────┘

User/Client         FilesSaver          FileSystem        Redis          Database
     │                  │                    │               │                │
     │  POST /save      │                    │               │                │
     │  (multipart)     │                    │               │                │
     ├─────────────────►│                    │               │                │
     │                  │                    │               │                │
     │                  │ 1. Validate params │               │                │
     │                  │    (username,      │               │                │
     │                  │     frompage)      │               │                │
     │                  │                    │               │                │
     │                  │ 2. Extract files   │               │                │
     │                  │    from parts      │               │                │
     │                  │                    │               │                │
     │                  │ 3. Generate UUID   │               │                │
     │                  │    for each file   │               │                │
     │                  │                    │               │                │
     │                  │  4. Write file     │               │                │
     │                  ├───────────────────►│               │                │
     │                  │                    │               │                │
     │                  │  5. If ZIP, extract│               │                │
     │                  │     contents       │               │                │
     │                  │◄───────────────────┤               │                │
     │                  │                    │               │                │
     │                  │ 6. Parse & count   │               │                │
     │                  │    records         │               │                │
     │                  │◄───────────────────┤               │                │
     │                  │    (via           │               │                │
     │                  │     FileReadService)│               │                │
     │                  │                    │               │                │
     │                  │ 7. Track files     │               │                │
     │                  │    for cleanup     │               │                │
     │                  ├───────────────────────────────────►│                │
     │                  │    LPUSH           │               │                │
     │                  │    tracking:       │               │                │
     │                  │    {type}:{user}   │               │                │
     │                  │◄───────────────────────────────────┤                │
     │                  │                    │               │                │
     │                  │ 8. Store metadata  │               │                │
     │                  ├───────────────────────────────────────────────────►│
     │                  │    INSERT INTO     │               │                │
     │                  │    campaign_files  │               │                │
     │                  │◄───────────────────────────────────────────────────┤
     │                  │                    │               │                │
     │                  │ 9. Build response  │               │                │
     │                  │    with counts     │               │                │
     │                  │                    │               │                │
     │  JSON Response   │                    │               │                │
     │  {total, files}  │                    │               │                │
     │◄─────────────────┤                    │               │                │
     │                  │                    │               │                │
```

**Key Points:**
- Multiple files can be uploaded in single request
- ZIP files are extracted automatically
- Each file gets a UUID to prevent collisions
- File counts are calculated asynchronously using FutureTask
- Files are tracked in Redis for cleanup
- Response includes total count and per-file details

---

## Campaign Processing Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CAMPAIGN PROCESSING SEQUENCE DIAGRAM                      │
└─────────────────────────────────────────────────────────────────────────────┘

Database            CampaignMasterPoller      Redis(FileSplitQ)      Database
   │                         │                       │                    │
   │                         │ 1. Poll every N sec   │                    │
   │  2. SELECT * FROM       │                       │                    │
   │     campaign_master     │                       │                    │
   │     WHERE status=       │                       │                    │
   │     'ready_to_process'  │                       │                    │
   │     LIMIT 100           │                       │                    │
   │◄────────────────────────┤                       │                    │
   │                         │                       │                    │
   │  3. Campaign records    │                       │                    │
   ├────────────────────────►│                       │                    │
   │                         │                       │                    │
   │                         │ 4. For each campaign  │                    │
   │                         │    Build JSON payload │                    │
   │                         │    {                  │                    │
   │                         │      cm_id,           │                    │
   │                         │      cli_id,          │                    │
   │                         │      file_path,       │                    │
   │                         │      c_type,          │                    │
   │                         │      message,         │                    │
   │                         │      ...              │                    │
   │                         │    }                  │                    │
   │                         │                       │                    │
   │                         │ 5. LPUSH FileSplitQ   │                    │
   │                         ├──────────────────────►│                    │
   │                         │                       │                    │
   │                         │ 6. Update status      │                    │
   │                         ├───────────────────────────────────────────►│
   │                         │    UPDATE              │                    │
   │                         │    campaign_master    │                    │
   │                         │    SET status=        │                    │
   │                         │    'queued_for_split' │                    │
   │                         │    WHERE cm_id=?      │                    │
   │                         │◄───────────────────────────────────────────┤
   │                         │                       │                    │
   │                         │ 7. Continue polling   │                    │
   │                         │    (loop)             │                    │
   │                         │                       │                    │

```

**Key Points:**
- Polling interval is configurable (default: continuous with sleep)
- Batch processing of multiple campaigns
- Status updated to prevent re-processing
- JSON payload includes all campaign metadata
- Retry mechanism if Redis push fails

---

## Split File Processing Sequence

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       SPLIT FILE PROCESSING SEQUENCE                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

Redis         FileSplitQConsumer    MasterFileSplitHandler   FileSystem    Database    Redis(DeliveryQ)
  │                  │                        │                  │             │               │
  │ 1. RPOP         │                        │                  │             │               │
  │  FileSplitQ     │                        │                  │             │               │
  ├────────────────►│                        │                  │             │               │
  │                  │                        │                  │             │               │
  │ 2. Campaign JSON │                        │                  │             │               │
  │  metadata        │                        │                  │             │               │
  │◄─────────────────┤                        │                  │             │               │
  │                  │                        │                  │             │               │
  │                  │ 3. Parse JSON          │                  │             │               │
  │                  │    Extract cm_id,      │                  │             │               │
  │                  │    file_path, etc.     │                  │             │               │
  │                  │                        │                  │             │               │
  │                  │ 4. Call handler        │                  │             │               │
  │                  ├───────────────────────►│                  │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ 5. Read master   │             │               │
  │                  │                        │    file          │             │               │
  │                  │                        ├─────────────────►│             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ 6. Get FileParser│             │               │
  │                  │                        │    (CSV/XLS/     │             │               │
  │                  │                        │     XLSX)        │             │               │
  │                  │                        │◄─────────────────┤             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ 7. Split logic   │             │               │
  │                  │                        │    (100K chunks) │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ ┌──────────────┐ │             │               │
  │                  │                        │ │ For each     │ │             │               │
  │                  │                        │ │ 100K records │ │             │               │
  │                  │                        │ └──────────────┘ │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ 8. Write split   │             │               │
  │                  │                        │    file          │             │               │
  │                  │                        ├─────────────────►│             │               │
  │                  │                        │   split_N.csv    │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │ 9. Insert split  │             │               │
  │                  │                        │    metadata      │             │               │
  │                  │                        ├─────────────────────────────►│               │
  │                  │                        │    INSERT INTO   │             │               │
  │                  │                        │    campaign_     │             │               │
  │                  │                        │    file_splits   │             │               │
  │                  │                        │    (c_f_s_id,    │             │               │
  │                  │                        │     cm_id,       │             │               │
  │                  │                        │     fileloc,     │             │               │
  │                  │                        │     total,       │             │               │
  │                  │                        │     status=      │             │               │
  │                  │                        │     'queued')    │             │               │
  │                  │                        │◄─────────────────────────────┤               │
  │                  │                        │    c_f_s_id      │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │10. Build split   │             │               │
  │                  │                        │   JSON           │             │               │
  │                  │                        │   {c_f_s_id,     │             │               │
  │                  │                        │    cm_id,        │             │               │
  │                  │                        │    fileloc,...}  │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │11. LPUSH to      │             │               │
  │                  │                        │   DeliveryQ_     │             │               │
  │                  │                        │   {cm_id}        │             │               │
  │                  │                        ├─────────────────────────────────────────────►│
  │                  │                        │                  │             │               │
  │                  │                        │ ┌──────────────┐ │             │               │
  │                  │                        │ │ End loop     │ │             │               │
  │                  │                        │ │ for splits   │ │             │               │
  │                  │                        │ └──────────────┘ │             │               │
  │                  │                        │                  │             │               │
  │                  │                        │12. Update        │             │               │
  │                  │                        │   campaign       │             │               │
  │                  │                        │   status         │             │               │
  │                  │                        ├─────────────────────────────►│               │
  │                  │                        │   UPDATE         │             │               │
  │                  │                        │   campaign_master│             │               │
  │                  │                        │   SET status=    │             │               │
  │                  │                        │   'split_done'   │             │               │
  │                  │◄───────────────────────┤                  │             │               │
  │                  │                        │                  │             │               │
  │                  │13. Continue consuming  │                  │             │               │
  │                  │    (loop)              │                  │             │               │
  │                  │                        │                  │             │               │
```

**Key Points:**
- File split size is configurable (default: 100,000 records)
- Each split gets a unique ID (c_f_s_id)
- Split files stored with parent campaign reference
- Separate delivery queue created per campaign (DeliveryQ_{cm_id})
- Database tracks each split for completion monitoring
- Supports CSV, XLS, XLSX formats through FileParser factory

---

## Handover to Delivery Sequence

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     HANDOVER TO DELIVERY SEQUENCE                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

Redis(DeliveryQ)  SplitFileConsumer   ProcessOTM/MTM/TEM   KafkaHandover   Kafka    Redis    Database
      │                  │                    │                   │           │        │          │
      │ 1. RPOPLPUSH     │                    │                   │           │        │          │
      │  DeliveryQ_{id}  │                    │                   │           │        │          │
      │  to DeliveryQ    │                    │                   │           │        │          │
      ├─────────────────►│                    │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │ 2. cm_id         │                    │                   │           │        │          │
      │◄─────────────────┤                    │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │ 3. RPOPLPUSH     │                    │                   │           │        │          │
      │  {cm_id} to      │                    │                   │           │        │          │
      │  processing:     │                    │                   │           │        │          │
      │  {cm_id}         │                    │                   │           │        │          │
      ├─────────────────►│                    │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │ 4. Split file    │                    │                   │           │        │          │
      │  metadata JSON   │                    │                   │           │        │          │
      │◄─────────────────┤                    │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │ 5. Parse JSON      │                   │           │        │          │
      │                  │    Extract c_f_s_id│                   │           │        │          │
      │                  │    c_type, etc.    │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │ 6. Route by type   │                   │           │        │          │
      │                  │    OTM/MTM/TEM     │                   │           │        │          │
      │                  ├───────────────────►│                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │ 7. Read split file│           │        │          │
      │                  │                    │    line by line   │           │        │          │
      │                  │                    │◄──────────────────┤           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │ 8. For each line  │           │        │          │
      │                  │                    │ ┌──────────────┐  │           │        │          │
      │                  │                    │ │ Validate     │  │           │        │          │
      │                  │                    │ │ mobile       │  │           │        │          │
      │                  │                    │ │              │  │           │        │          │
      │                  │                    │ │ Apply        │  │           │        │          │
      │                  │                    │ │ template if  │  │           │        │          │
      │                  │                    │ │ TEM type     │  │           │        │          │
      │                  │                    │ │              │  │           │        │          │
      │                  │                    │ │ Check DLT    │  │           │        │          │
      │                  │                    │ │              │  │           │        │          │
      │                  │                    │ │ Check        │  │           │        │          │
      │                  │                    │ │ exclusions   │  │           │        │          │
      │                  │                    │ │              │  │           │        │          │
      │                  │                    │ │ Build        │  │           │        │          │
      │                  │                    │ │ message      │  │           │        │          │
      │                  │                    │ └──────────────┘  │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │ 9. Handover to    │           │        │          │
      │                  │                    │    Kafka          │           │        │          │
      │                  │                    ├──────────────────►│           │        │          │
      │                  │                    │    {mobile,       │           │        │          │
      │                  │                    │     message,      │           │        │          │
      │                  │                    │     sender_id,    │           │        │          │
      │                  │                    │     dlt_info,...} │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │                   │10. Send to│        │          │
      │                  │                    │                   │   Kafka   │        │          │
      │                  │                    │                   │   topic   │        │          │
      │                  │                    │                   ├──────────►│        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │ 11. Increment     │           │        │          │
      │                  │                    │     success count │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │ ┌──────────────┐  │           │        │          │
      │                  │                    │ │ End loop     │  │           │        │          │
      │                  │                    │ └──────────────┘  │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │                    │12. Update stats   │           │        │          │
      │                  │                    ├───────────────────────────────────────────────►│
      │                  │                    │    UPDATE         │           │        │          │
      │                  │                    │    campaign_      │           │        │          │
      │                  │                    │    file_splits    │           │        │          │
      │                  │                    │    SET status=    │           │        │          │
      │                  │                    │    'COMPLETED',   │           │        │          │
      │                  │                    │    sent_count=?   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │◄───────────────────┤                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │13. Remove from     │                   │           │        │          │
      │                  │    processing Q    │                   │           │        │          │
      │                  ├────────────────────────────────────────────────────────────►│          │
      │                  │    LREM            │                   │           │        │          │
      │                  │    processing:     │                   │           │        │          │
      │                  │    {cm_id}         │                   │           │        │          │
      │                  │                    │                   │           │        │          │
      │                  │14. Continue        │                   │           │        │          │
      │                  │    consuming       │                   │           │        │          │
      │                  │    (loop)          │                   │           │        │          │
      │                  │                    │                   │           │        │          │
```

**Key Points:**
- Uses RPOPLPUSH for atomic queue operations
- Temporary processing queue prevents data loss
- Line-by-line processing for memory efficiency
- Different processors for OTM/MTM/TEM types
- Validation and enrichment before Kafka send
- Stats updated in database after completion
- Processing queue cleaned after success

---

## Error Handling Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ERROR HANDLING SEQUENCE                                │
└─────────────────────────────────────────────────────────────────────────────┘

Consumer          Exception         Redis           Database          Redis(Queue)
   │                  │                 │                │                  │
   │ 1. Processing    │                 │                │                  │
   │    record        │                 │                │                  │
   │──────────►       │                 │                │                  │
   │                  │                 │                │                  │
   │ 2. Exception!    │                 │                │                  │
   │◄─────────────────┤                 │                │                  │
   │                  │                 │                │                  │
   │ 3. Catch         │                 │                │                  │
   │    exception     │                 │                │                  │
   │                  │                 │                │                  │
   │ 4. Get retry     │                 │                │                  │
   │    count from    │                 │                │                  │
   │    Redis         │                 │                │                  │
   ├─────────────────────────────────►│                │                  │
   │    GET           │                 │                │                  │
   │    retry:{id}    │                 │                │                  │
   │◄─────────────────────────────────┤                │                  │
   │    count=2       │                 │                │                  │
   │                  │                 │                │                  │
   │ 5. Check if      │                 │                │                  │
   │    retry < MAX   │                 │                │                  │
   │    (default: 5)  │                 │                │                  │
   │                  │                 │                │                  │
   ├─► YES ───────────┼─────────────────┼────────────────┼──────────────────┤
   │                  │                 │                │                  │
   │ 6. Increment     │                 │                │                  │
   │    retry count   │                 │                │                  │
   ├─────────────────────────────────►│                │                  │
   │    INCR          │                 │                │                  │
   │    retry:{id}    │                 │                │                  │
   │                  │                 │                │                  │
   │ 7. Re-push to    │                 │                │                  │
   │    same queue    │                 │                │                  │
   ├───────────────────────────────────────────────────────────────────────►│
   │    LPUSH queue   │                 │                │                  │
   │    {metadata}    │                 │                │                  │
   │                  │                 │                │                  │
   │ 8. Log error     │                 │                │                  │
   │                  │                 │                │                  │
   │                  │                 │                │                  │
   ├─► NO (MAX)───────┼─────────────────┼────────────────┼──────────────────┤
   │                  │                 │                │                  │
   │ 9. Mark as       │                 │                │                  │
   │    FAILED        │                 │                │                  │
   ├─────────────────────────────────────────────────►│                  │
   │    UPDATE        │                 │                │                  │
   │    campaign_     │                 │                │                  │
   │    file_splits   │                 │                │                  │
   │    SET status=   │                 │                │                  │
   │    'FAILED'      │                 │                │                  │
   │    WHERE id=?    │                 │                │                  │
   │                  │                 │                │                  │
   │10. Remove from   │                 │                │                  │
   │    processing Q  │                 │                │                  │
   ├─────────────────────────────────►│                │                  │
   │    LREM          │                 │                │                  │
   │    processing:Q  │                 │                │                  │
   │                  │                 │                │                  │
   │11. Log failure   │                 │                │                  │
   │                  │                 │                │                  │
   │12. Sleep thread  │                 │                │                  │
   │    (30 seconds)  │                 │                │                  │
   │                  │                 │                │                  │
   │13. Restart       │                 │                │                  │
   │    consumer      │                 │                │                  │
   │                  │                 │                │                  │
```

**Key Points:**
- Retry count tracked per record in Redis
- Configurable max retry count (default: 5)
- Failed records marked in database
- Processing queue cleaned up
- Thread sleeps on error to prevent tight loop
- Detailed error logging for troubleshooting

---

## Campaign Completion Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CAMPAIGN COMPLETION SEQUENCE                             │
└─────────────────────────────────────────────────────────────────────────────┘

Database(Splits)  PollerFiles      Database(Campaign)   PollerCampaign   Redis
      │              Completed             │                Master        Cleaner
      │                  │                  │              Completed         │
      │                  │                  │                  │             │
      │ 1. Poll splits   │                  │                  │             │
      │  SELECT * FROM   │                  │                  │             │
      │  campaign_file_  │                  │                  │             │
      │  splits WHERE    │                  │                  │             │
      │  status=         │                  │                  │             │
      │  'COMPLETED'     │                  │                  │             │
      │  GROUP BY cm_id  │                  │                  │             │
      │◄─────────────────┤                  │                  │             │
      │                  │                  │                  │             │
      │ 2. Check if all  │                  │                  │             │
      │    splits done   │                  │                  │             │
      │    for cm_id     │                  │                  │             │
      ├─────────────────►│                  │                  │             │
      │                  │                  │                  │             │
      │                  │ 3. Aggregate     │                  │             │
      │                  │    stats         │                  │             │
      │                  │    - Total sent  │                  │             │
      │                  │    - Total failed│                  │             │
      │                  │    - Time taken  │                  │             │
      │                  │                  │                  │             │
      │                  │ 4. Update master │                  │             │
      │                  ├─────────────────►│                  │             │
      │                  │    UPDATE        │                  │             │
      │                  │    campaign_     │                  │             │
      │                  │    master        │                  │             │
      │                  │    SET           │                  │             │
      │                  │    total_sent=?, │                  │             │
      │                  │    total_failed=?│                  │             │
      │                  │    WHERE cm_id=? │                  │             │
      │                  │                  │                  │             │
      │                  │ 5. Poll completed│                  │             │
      │                  │    campaigns     │                  │             │
      │                  │                  │◄─────────────────┤             │
      │                  │                  │  SELECT * FROM   │             │
      │                  │                  │  campaign_master │             │
      │                  │                  │  WHERE all       │             │
      │                  │                  │  splits done     │             │
      │                  │                  │                  │             │
      │                  │                  │ 6. Mark as       │             │
      │                  │                  │    COMPLETED     │             │
      │                  │                  ├─────────────────►│             │
      │                  │                  │    UPDATE        │             │
      │                  │                  │    campaign_     │             │
      │                  │                  │    master        │             │
      │                  │                  │    SET status=   │             │
      │                  │                  │    'COMPLETED',  │             │
      │                  │                  │    completed_ts= │             │
      │                  │                  │    NOW()         │             │
      │                  │                  │                  │             │
      │                  │                  │ 7. Trigger       │             │
      │                  │                  │    cleanup       │             │
      │                  │                  ├─────────────────────────────►│
      │                  │                  │                  │             │
      │                  │                  │                  │ 8. Clean    │
      │                  │                  │                  │    DeliveryQ│
      │                  │                  │                  │    {cm_id}  │
      │                  │                  │                  │             │
      │                  │                  │                  │ 9. Clean    │
      │                  │                  │                  │    processing│
      │                  │                  │                  │    queues   │
      │                  │                  │                  │             │
      │                  │                  │                  │10. Clean    │
      │                  │                  │                  │    retry    │
      │                  │                  │                  │    counters │
      │                  │                  │                  │             │
      │                  │                  │11. Optional:     │             │
      │                  │                  │    Send          │             │
      │                  │                  │    notification  │             │
      │                  │                  │    to user       │             │
      │                  │                  │                  │             │
```

**Key Points:**
- Polling-based completion detection
- Aggregated statistics calculated
- Master campaign marked as completed
- Redis queues cleaned up automatically
- Completion timestamp recorded
- Optional user notifications
- Campaign reports can be generated

---

## Summary

These sequence diagrams illustrate the complete lifecycle of campaign processing in the Beacon File Processor system:

1. **File Upload**: User uploads files, system stores and tracks them
2. **Campaign Processing**: Poller picks up campaigns and queues them for processing
3. **Split File Processing**: Large files are split into manageable chunks
4. **Handover to Delivery**: Split files are processed and sent to Kafka for delivery
5. **Error Handling**: Robust retry mechanism with failure tracking
6. **Campaign Completion**: Automated detection and cleanup of completed campaigns

The system is designed for:
- **Reliability**: Retry mechanisms and error handling
- **Scalability**: Queue-based architecture enables parallel processing
- **Traceability**: Every step is logged and tracked in database
- **Efficiency**: Streaming file processing and optimized database queries
- **Maintainability**: Clear separation of concerns across modules

