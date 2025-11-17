# Detailed Flow Diagrams - Beacon File Processor

## 📊 Sequence Diagrams

### 1. Complete Campaign Processing Sequence

```
┌────────┐  ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  User  │  │   UI   │  │ Upload   │  │ Database │  │ Initial  │  │  Split   │  │ Exclude  │  │ Handover │
│        │  │        │  │ Service  │  │          │  │  Stage   │  │  Stage   │  │  Stage   │  │  Stage   │
└───┬────┘  └───┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
    │           │            │             │             │             │             │             │
    │ Upload    │            │             │             │             │             │             │
    │ File      │            │             │             │             │             │             │
    ├──────────>│            │             │             │             │             │             │
    │           │ POST /save │             │             │             │             │             │
    │           ├───────────>│             │             │             │             │             │
    │           │            │ Validate    │             │             │             │             │
    │           │            │ & Store     │             │             │             │             │
    │           │            │ File        │             │             │             │             │
    │           │            ├─────────┐   │             │             │             │             │
    │           │            │         │   │             │             │             │             │
    │           │            │<────────┘   │             │             │             │             │
    │           │            │             │             │             │             │             │
    │           │<───────────┤             │             │             │             │             │
    │           │ Success    │             │             │             │             │             │
    │<──────────┤            │             │             │             │             │             │
    │           │            │             │             │             │             │             │
    │ Create    │            │             │             │             │             │             │
    │ Campaign  │            │             │             │             │             │             │
    ├──────────>│            │             │             │             │             │             │
    │           │ INSERT     │             │             │             │             │             │
    │           │ Campaign   │             │             │             │             │             │
    │           ├────────────┼────────────>│             │             │             │             │
    │           │            │ campaign_   │             │             │             │             │
    │           │            │ master &    │             │             │             │             │
    │           │            │ files       │             │             │             │             │
    │           │            │             │             │             │             │             │
    │           │            │<────────────┤             │             │             │             │
    │           │<───────────┤             │             │             │             │             │
    │           │            │             │             │             │             │             │
    │           │            │             │ Poll DB    │             │             │             │
    │           │            │             │ (every 1s) │             │             │             │
    │           │            │             │<───────────┤             │             │             │
    │           │            │             │            │             │             │             │
    │           │            │             │ SELECT     │             │             │             │
    │           │            │             │ WHERE      │             │             │             │
    │           │            │             │ status=    │             │             │             │
    │           │            │             │ 'queued'   │             │             │             │
    │           │            │             ├───────────>│             │             │             │
    │           │            │             │            │             │             │             │
    │           │            │             │<───────────┤             │             │             │
    │           │            │             │ Campaign   │             │             │             │
    │           │            │             │ Data       │             │             │             │
    │           │            │             │            │             │             │             │
    │           │            │             │ UPDATE     │             │             │             │
    │           │            │             │ status=    │             │             │             │
    │           │            │             │ 'inprogress'│            │             │             │
    │           │            │             ├───────────>│             │             │             │
    │           │            │             │            │             │             │             │
    │           │            │             │ LPUSH      │             │             │             │
    │           │            │             │ FileSplitQ │             │             │             │
    │           │            │             ├────────────┼────────────>│             │             │
    │           │            │             │            │             │             │             │
    │           │            │             │            │ RPOP       │             │             │
    │           │            │             │            │ FileSplitQ │             │             │
    │           │            │             │            │<───────────┤             │             │
    │           │            │             │            │            │             │             │
    │           │            │             │            │ Parse &    │             │             │
    │           │            │             │            │ Split File │             │             │
    │           │            │             │            │ (10k each) │             │             │
    │           │            │             │            ├────────┐   │             │             │
    │           │            │             │            │        │   │             │             │
    │           │            │             │            │<───────┘   │             │             │
    │           │            │             │            │            │             │             │
    │           │            │             │ INSERT     │            │             │             │
    │           │            │             │ campaign_  │            │             │             │
    │           │            │             │ file_splits│            │             │             │
    │           │            │             │<───────────┤            │             │             │
    │           │            │             │            │            │             │             │
    │           │            │             │            │ Has        │             │             │
    │           │            │             │            │ Exclude?   │             │             │
    │           │            │             │            ├────────┐   │             │             │
    │           │            │             │            │  YES   │   │             │             │
    │           │            │             │            │<───────┘   │             │             │
    │           │            │             │            │            │             │             │
    │           │            │             │            │ LPUSH      │             │             │
    │           │            │             │            │ ExcludeQ   │             │             │
    │           │            │             │            ├────────────┼────────────>│             │
    │           │            │             │            │            │             │             │
    │           │            │             │            │            │ RPOPLPUSH  │             │
    │           │            │             │            │            │ ExcludeQ   │             │
    │           │            │             │            │            │<───────────┤             │
    │           │            │             │            │            │            │             │
    │           │            │             │            │            │ Filter     │             │
    │           │            │             │            │            │ Exclude    │             │
    │           │            │             │            │            │ Numbers    │             │
    │           │            │             │            │            ├────────┐   │             │
    │           │            │             │            │            │        │   │             │
    │           │            │             │            │            │<───────┘   │             │
    │           │            │             │            │            │            │             │
    │           │            │             │ UPDATE     │            │            │             │
    │           │            │             │ exclude_   │            │            │             │
    │           │            │             │ count      │            │            │             │
    │           │            │             │<───────────┼────────────┤            │             │
    │           │            │             │            │            │            │             │
    │           │            │             │            │            │ LPUSH      │             │
    │           │            │             │            │            │ DeliveryQ  │             │
    │           │            │             │            │            ├────────────┼────────────>│
    │           │            │             │            │            │            │             │
    │           │            │             │            │            │            │ RPOPLPUSH  │
    │           │            │             │            │            │            │ DeliveryQ  │
    │           │            │             │            │            │            │<───────────┤
    │           │            │             │            │            │            │            │
    │           │            │             │            │            │            │ Process    │
    │           │            │             │            │            │            │ Split File │
    │           │            │             │            │            │            │ Line by    │
    │           │            │             │            │            │            │ Line       │
    │           │            │             │            │            │            ├────────┐   │
    │           │            │             │            │            │            │        │   │
    │           │            │             │            │            │            │<───────┘   │
    │           │            │             │            │            │            │            │
    │           │            │             │            │            │            │ LPUSH      │
    │           │            │             │            │            │            │ Platform   │
    │           │            │             │            │            │            │ Queue      │
    │           │            │             │            │            │            ├──────────> │
    │           │            │             │            │            │            │ (to Delivery
    │           │            │             │            │            │            │  Engine)   │
    │           │            │             │            │            │            │            │
    │           │            │             │ UPDATE     │            │            │            │
    │           │            │             │ status=    │            │            │            │
    │           │            │             │ 'completed'│            │            │            │
    │           │            │             │<───────────┼────────────┼────────────┤            │
    │           │            │             │            │            │            │            │
    │<──────────┼────────────┼─────────────┤ Notify     │            │            │            │
    │ Campaign  │            │             │ Complete   │            │            │            │
    │ Complete  │            │             │            │            │            │            │
    │           │            │             │            │            │            │            │
```

---

## 2. File Upload Detailed Flow

```
┌──────────┐
│   User   │
└────┬─────┘
     │
     │ POST /save (multipart/form-data)
     │ - username: testuser
     │ - frompage: campaign
     │ - files: [file1.csv, file2.xlsx, archive.zip]
     │
     ▼
┌─────────────────────────────────────────────────────┐
│           FilesSaver Servlet                        │
│                                                     │
│  1. Validate Parameters                             │
│     ├─→ username required?  ✓                       │
│     └─→ frompage required?  ✓                       │
│                                                     │
│  2. Determine Storage Location                      │
│     ├─→ frompage = "campaign"                       │
│     │   → /files/campaigns/testuser/                │
│     ├─→ frompage = "group"                          │
│     │   → /files/groups/testuser/                   │
│     └─→ frompage = "template"                       │
│         → /files/templates/testuser/                │
│                                                     │
│  3. Create Directory                                │
│     └─→ Files.createDirectories(path)               │
│                                                     │
│  4. Process Each File (Loop)                        │
│     │                                               │
│     ├─→ file1.csv                                   │
│     │   ├─→ Generate UUID: abc-123                  │
│     │   ├─→ Store as: file1_abc-123_timestamp.csv  │
│     │   ├─→ Convert encoding (UTF-8)                │
│     │   └─→ Response: {filename, r_filename, count} │
│     │                                               │
│     ├─→ file2.xlsx                                  │
│     │   ├─→ Generate UUID: def-456                  │
│     │   ├─→ Store as: file2_def-456.xlsx           │
│     │   └─→ Response: {filename, r_filename, count} │
│     │                                               │
│     └─→ archive.zip                                 │
│         ├─→ Generate UUID: ghi-789                  │
│         ├─→ Store as: archive_ghi-789.zip          │
│         ├─→ Extract ZIP contents                    │
│         │   ├─→ Contains: data1.csv, data2.txt     │
│         │   ├─→ Store extracted files               │
│         │   └─→ Delete original ZIP                 │
│         └─→ Response: [{file1}, {file2}]            │
│                                                     │
│  5. Parse Files (Async - FutureTask)                │
│     │                                               │
│     └─→ For each file:                              │
│         ├─→ Create FileReadService(fileInfo)        │
│         ├─→ Create FutureTask(callable)             │
│         ├─→ Start Thread                            │
│         └─→ Store in taskList                       │
│                                                     │
│  6. Track Files in Redis                            │
│     │                                               │
│     └─→ SADD uploaded_files:testuser                │
│         ├─→ /files/campaigns/testuser/file1_abc... │
│         ├─→ /files/campaigns/testuser/file2_def... │
│         └─→ ... (for cleanup later)                 │
│                                                     │
│  7. Wait for All Tasks to Complete                  │
│     │                                               │
│     └─→ While (completedTasks < totalTasks)         │
│         ├─→ Check each FutureTask.isDone()          │
│         └─→ Sleep 100ms between checks              │
│                                                     │
│  8. Collect Results                                 │
│     │                                               │
│     ├─→ Success:                                    │
│     │   ├─→ {filename: "file1.csv",                │
│     │   │    r_filename: "file1_abc-123.csv",      │
│     │   │    count: 10000,                          │
│     │   │    count_human: "10,000"}                 │
│     │   │                                           │
│     │   └─→ {filename: "file2.xlsx",               │
│     │        r_filename: "file2_def-456.xlsx",     │
│     │        count: 5000,                           │
│     │        count_human: "5,000"}                  │
│     │                                               │
│     └─→ Failed:                                     │
│         └─→ {filename: "invalid.txt",               │
│              error: "UNSUPPORTED_FILE_TYPE"}        │
│                                                     │
│  9. Build Response                                  │
│     │                                               │
│     └─→ {                                           │
│         "statusCode": 200,                          │
│         "total": 15000,                             │
│         "total_human": "15,000",                    │
│         "uploaded_files": {                         │
│           "success": [file1, file2],                │
│           "failed": [invalid]                       │
│         }                                           │
│       }                                             │
│                                                     │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌──────────┐
│   User   │ ← JSON Response
└──────────┘
```

---

## 3. Split Stage Detailed Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                     SPLIT STAGE PROCESSING                         │
└────────────────────────────────────────────────────────────────────┘

┌────────────────┐
│  FileSplitQ    │ ← {cm_id: 12345, cf_id: 67890, fileloc: "...", ...}
└────────┬───────┘
         │ RPOP
         ▼
┌──────────────────────────────────────────────────────────────────┐
│        FileSplitQConsumer Thread                                 │
│                                                                  │
│  Step 1: Validate Campaign Data                                 │
│  ───────────────────────────────                                │
│  ├─→ cm_id present? ✓                                           │
│  ├─→ cf_id present? ✓                                           │
│  ├─→ fileloc exists? ✓                                          │
│  └─→ Get user account details                                   │
│                                                                  │
│  Step 2: Check if Already Split                                 │
│  ──────────────────────────────                                 │
│  ├─→ SELECT * FROM campaign_file_splits                         │
│  │   WHERE cf_id = 67890 AND retry_count <= 5                   │
│  │                                                              │
│  └─→ If found (count > 0):                                      │
│      ├─→ Use existing split files                               │
│      └─→ Skip to Step 6                                         │
│                                                                  │
│  Step 3: Parse & Split File (NEW)                               │
│  ────────────────────────────────                               │
│  ├─→ reWriteAndSplitUploadedFiles()                            │
│  │                                                              │
│  │   A. Determine File Type                                     │
│  │   ───────────────────                                       │
│  │   ├─→ Extension: .csv → CsvFileParser                       │
│  │   ├─→ Extension: .xls → XlsFileParser                       │
│  │   ├─→ Extension: .xlsx → XlsxFileParser                     │
│  │   └─→ Extension: .txt → TextFileParser                      │
│  │                                                              │
│  │   B. Configure Parser                                        │
│  │   ─────────────────                                         │
│  │   ├─→ campaign_type = "otm"                                 │
│  │   │   ├─→ SendSMSTypes = OTM                                │
│  │   │   ├─→ MsgTypeCheck = true                               │
│  │   │   └─→ columnsLimit = 1 (only mobile)                    │
│  │   │                                                          │
│  │   ├─→ campaign_type = "mtm"                                 │
│  │   │   ├─→ SendSMSTypes = MTM                                │
│  │   │   ├─→ MsgTypeCheck = false                              │
│  │   │   ├─→ columnsLimit = 2 (mobile|message)                 │
│  │   │   └─→ delimiter = "|"                                   │
│  │   │                                                          │
│  │   └─→ campaign_type = "template"                            │
│  │       ├─→ SendSMSTypes = TEMPLATE                           │
│  │       ├─→ MsgTypeCheck = false                              │
│  │       ├─→ Get template details                              │
│  │       │   ├─→ template_id, template_type                    │
│  │       │   ├─→ mobile_column_name                            │
│  │       │   └─→ template_fields: {name, amount, date}         │
│  │       ├─→ Parse placeholders: {field1}, {field2}...         │
│  │       └─→ columnsLimit = field count                        │
│  │                                                              │
│  │   C. Open File & Parse                                       │
│  │   ──────────────────────                                    │
│  │   ├─→ fileParser.parse()                                    │
│  │   │                                                          │
│  │   │   Original File: contacts.csv (50,000 records)          │
│  │   │   ─────────────────────────────────────────             │
│  │   │   9876543210                                            │
│  │   │   9876543211                                            │
│  │   │   9876543212                                            │
│  │   │   ...                                                   │
│  │   │   9876593209 (record 50,000)                            │
│  │   │                                                          │
│  │   └─→ FileChopHandler (splits based on limit)               │
│  │       ├─→ SMS_SPLIT_LIMIT = 10,000                          │
│  │       │                                                      │
│  │       ├─→ Create Split File 1: split_1.txt                  │
│  │       │   ├─→ Records 1-10,000                              │
│  │       │   └─→ Path: /files/campaigns/testuser/split_1.txt  │
│  │       │                                                      │
│  │       ├─→ Create Split File 2: split_2.txt                  │
│  │       │   ├─→ Records 10,001-20,000                         │
│  │       │   └─→ Path: /files/campaigns/testuser/split_2.txt  │
│  │       │                                                      │
│  │       ├─→ Create Split File 3: split_3.txt                  │
│  │       │   ├─→ Records 20,001-30,000                         │
│  │       │                                                      │
│  │       ├─→ Create Split File 4: split_4.txt                  │
│  │       │   ├─→ Records 30,001-40,000                         │
│  │       │                                                      │
│  │       └─→ Create Split File 5: split_5.txt                  │
│  │           ├─→ Records 40,001-50,000                         │
│  │           └─→ Total: 10,000 records                         │
│  │                                                              │
│  │   D. Unicode Conversion (if needed)                          │
│  │   ────────────────────────────────                          │
│  │   └─→ If c_lang_type = "unicode"                            │
│  │       ├─→ Convert message to hex                            │
│  │       └─→ Update msg field                                  │
│  │                                                              │
│  └─→ Return FileDataBean                                        │
│      ├─→ totalNumbers: 50,000                                  │
│      └─→ splitFiles: List<SplitFileData>                       │
│          [                                                      │
│            {fileloc: split_1.txt, count: 10000},               │
│            {fileloc: split_2.txt, count: 10000},               │
│            {fileloc: split_3.txt, count: 10000},               │
│            {fileloc: split_4.txt, count: 10000},               │
│            {fileloc: split_5.txt, count: 10000}                │
│          ]                                                      │
│                                                                  │
│  Step 4: Insert into Database                                   │
│  ─────────────────────────                                      │
│  └─→ INSERT INTO campaign_file_splits                           │
│      (c_f_s_id, cm_id, cf_id, fileloc, total, status)          │
│      VALUES                                                      │
│      (1001, 12345, 67890, '/path/split_1.txt', 10000, 'queued'),│
│      (1002, 12345, 67890, '/path/split_2.txt', 10000, 'queued'),│
│      (1003, 12345, 67890, '/path/split_3.txt', 10000, 'queued'),│
│      (1004, 12345, 67890, '/path/split_4.txt', 10000, 'queued'),│
│      (1005, 12345, 67890, '/path/split_5.txt', 10000, 'queued') │
│                                                                  │
│  Step 5: Determine Queue                                        │
│  ───────────────────                                            │
│  ├─→ total = 50,000                                             │
│  ├─→ if (total < 10,000)                                        │
│  │   ├─→ queue = "LowVolumeDQ"                                 │
│  │   └─→ priority = 1                                           │
│  ├─→ else if (total < 100,000)                                  │
│  │   ├─→ queue = "MediumVolumeDQ"  ← Selected                  │
│  │   └─→ priority = 2                                           │
│  └─→ else                                                        │
│      ├─→ queue = "HighVolumeDQ"                                 │
│      └─→ priority = 4                                           │
│                                                                  │
│  Step 6: Check Exclude Groups                                   │
│  ────────────────────────                                       │
│  ├─→ exclude_group_ids = "101,102"  (present)                   │
│  └─→ queue = queue + "_exclude"                                 │
│      = "MediumVolumeDQ_exclude"                                 │
│                                                                  │
│  Step 7: Push to Redis Queue                                    │
│  ────────────────────────                                       │
│  └─→ RedisQueueSender.sendToRedis()                            │
│      │                                                          │
│      ├─→ For each split file:                                   │
│      │   │                                                      │
│      │   ├─→ Create metadata JSON:                              │
│      │   │   {                                                  │
│      │   │     "c_f_s_id": 1001,                                │
│      │   │     "cm_id": 12345,                                  │
│      │   │     "cf_id": 67890,                                  │
│      │   │     "fileloc": "/path/split_1.txt",                 │
│      │   │     "total": 10000,                                  │
│      │   │     "msg": "Campaign message",                       │
│      │   │     "header": "SENDER",                              │
│      │   │     "cli_id": "CLI001",                              │
│      │   │     "c_type": "otm",                                 │
│      │   │     "exclude_group_ids": "101,102",                  │
│      │   │     "PRIORITY": "2",                                 │
│      │   │     ...                                              │
│      │   │   }                                                  │
│      │   │                                                      │
│      │   └─→ LPUSH 12345:Q_exclude → metadata_json             │
│      │                                                          │
│      └─→ LPUSH MediumVolumeDQ_exclude → "12345:Q_exclude"      │
│          (Push campaign ID queue name to main queue)            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────┐
│ MediumVolumeDQ_exclude │ ← "12345:Q_exclude"
└────────────────────────┘
         │
┌────────────────────────┐
│   12345:Q_exclude      │ ← [metadata1, metadata2, ..., metadata5]
└────────────────────────┘
```

---

## 4. Exclude Processing Detailed Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                   EXCLUDE PROCESSING FLOW                          │
└────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐
│ MediumVolumeDQ_exclude  │
└────────┬────────────────┘
         │ RPOPLPUSH (atomic: pop and keep in queue)
         ▼
┌──────────────────────────────────────────────────────────────────┐
│        ExcludeConsumer Thread                                    │
│                                                                  │
│  Step 1: Get Campaign ID Queue                                  │
│  ──────────────────────────────                                 │
│  └─→ RPOPLPUSH MediumVolumeDQ_exclude → MediumVolumeDQ_exclude  │
│      Result: "12345:Q_exclude"                                   │
│      (Keeps cm_id in queue until all files processed)            │
│                                                                  │
│  Step 2: Get Split File Metadata                                │
│  ───────────────────────────────                                │
│  └─→ RPOP 12345:Q_exclude                                       │
│      Result: {c_f_s_id: 1001, fileloc: "/path/split_1.txt",    │
│               exclude_group_ids: "101,102", total: 10000, ...}  │
│                                                                  │
│  Step 3: Fetch Exclude Groups                                   │
│  ────────────────────────────                                   │
│  └─→ SELECT mobile FROM group_members                           │
│      WHERE group_id IN (101, 102)                               │
│      Result: Set of 500 numbers                                 │
│      [                                                           │
│        9876501111,                                              │
│        9876501112,                                              │
│        9876501113,                                              │
│        ...                                                       │
│        9876501610  (500 numbers total)                          │
│      ]                                                           │
│                                                                  │
│  Step 4: Read & Filter Split File                               │
│  ─────────────────────────────────                              │
│  └─→ FileDataExtractor.process()                               │
│      │                                                          │
│      ├─→ Open: /path/split_1.txt (10,000 records)              │
│      │                                                          │
│      ├─→ Create two temp files:                                 │
│      │   ├─→ non_exclude_file.txt  (filtered)                  │
│      │   └─→ exclude_file.txt      (excluded numbers)          │
│      │                                                          │
│      ├─→ Read line by line:                                     │
│      │   │                                                      │
│      │   ├─→ Line 1: 9876543210                                │
│      │   │   ├─→ Check if in exclude set? NO                   │
│      │   │   └─→ Write to: non_exclude_file.txt                │
│      │   │                                                      │
│      │   ├─→ Line 2: 9876543211                                │
│      │   │   ├─→ Check if in exclude set? NO                   │
│      │   │   └─→ Write to: non_exclude_file.txt                │
│      │   │                                                      │
│      │   ├─→ Line 50: 9876501111                               │
│      │   │   ├─→ Check if in exclude set? YES (in group 101)   │
│      │   │   ├─→ Write to: exclude_file.txt                    │
│      │   │   └─→ excludeCount++                                │
│      │   │                                                      │
│      │   ├─→ ... (continue for all 10,000 records)             │
│      │   │                                                      │
│      │   └─→ Statistics:                                        │
│      │       ├─→ Total processed: 10,000                        │
│      │       ├─→ Excluded: 50 numbers                           │
│      │       └─→ To send: 9,950 numbers                         │
│      │                                                          │
│      └─→ Return Map:                                            │
│          {                                                       │
│            "total": "9950",                                      │
│            "file_loc": "/path/non_exclude_1.txt",               │
│            "exc_file_loc": "/path/exclude_1.txt",               │
│            "excludeCnt": "50"                                    │
│          }                                                       │
│                                                                  │
│  Step 5: Update Database                                        │
│  ───────────────────────                                        │
│  └─→ UPDATE campaign_file_splits                                │
│      SET fileloc = '/path/non_exclude_1.txt',                   │
│          exclude_count = 50,                                     │
│          total = 9950                                            │
│      WHERE c_f_s_id = 1001                                       │
│                                                                  │
│  Step 6: Hand Over to Delivery Queue                            │
│  ───────────────────────────────────                            │
│  └─→ pushToDeliveryQueue()                                      │
│      │                                                          │
│      ├─→ Remove "_exclude" suffix from queue name:              │
│      │   "MediumVolumeDQ_exclude" → "MediumVolumeDQ"           │
│      │   "12345:Q_exclude" → "12345:Q"                          │
│      │                                                          │
│      ├─→ Update metadata with new file location:                │
│      │   {                                                      │
│      │     "c_f_s_id": 1001,                                    │
│      │     "fileloc": "/path/non_exclude_1.txt",  ← Updated    │
│      │     "total": 9950,                          ← Updated    │
│      │     "exclude": 50,                          ← Added      │
│      │     "EXCLUDE_FILE_LOC": "/path/exclude_1.txt", ← Added  │
│      │     ...                                                  │
│      │   }                                                      │
│      │                                                          │
│      ├─→ LPUSH 12345:Q → updated_metadata                       │
│      │                                                          │
│      └─→ LPUSH MediumVolumeDQ → "12345:Q"                       │
│                                                                  │
│  Step 7: Track Exclude File (Optional)                          │
│  ──────────────────────────────────────                         │
│  └─→ pushToExcludeNumberQueue()                                │
│      LPUSH ExcludeNumberQ → {                                   │
│        "c_f_s_id": 1001,                                        │
│        "EXCLUDE_FILE": "/path/exclude_1.txt",                   │
│        "cli_id": "CLI001",                                      │
│        "fileid": 67890                                          │
│      }                                                           │
│      (For reporting/tracking excluded numbers)                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────┐
│  MediumVolumeDQ  │ ← "12345:Q" (without _exclude)
└──────────────────┘
         │
┌──────────────────┐
│    12345:Q       │ ← Updated metadata with filtered file
└──────────────────┘
```

---

## 5. Handover Stage Detailed Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                    HANDOVER STAGE PROCESSING                       │
└────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│  MediumVolumeDQ  │
└────────┬─────────┘
         │ RPOPLPUSH (atomic: keep in queue during processing)
         ▼
┌──────────────────────────────────────────────────────────────────┐
│        SplitFileConsumer Thread                                  │
│                                                                  │
│  Step 1: Get Campaign ID Queue (Atomic)                         │
│  ──────────────────────────────────────                         │
│  └─→ RPOPLPUSH MediumVolumeDQ → MediumVolumeDQ                  │
│      Result: "12345:Q"                                           │
│      (Campaign ID stays in queue until all files complete)       │
│                                                                  │
│  Step 2: Get Split File Metadata (With Safety)                  │
│  ─────────────────────────────────────────                      │
│  └─→ RPOPLPUSH 12345:Q → FP:processing:12345:Q                  │
│      Result: {                                                   │
│        "c_f_s_id": 1001,                                         │
│        "cm_id": 12345,                                           │
│        "cf_id": 67890,                                           │
│        "fileloc": "/path/non_exclude_1.txt",                    │
│        "total": 9950,                                            │
│        "msg": "Get 50% off!",                                    │
│        "header": "PROMO",                                        │
│        "cli_id": "CLI001",                                       │
│        "c_type": "otm",                                          │
│        "c_lang_type": "text",                                    │
│        "delimiter": "|",                                         │
│        "PRIORITY": "2",                                          │
│        "APP_INSTANCE_ID": "beacon-platform-01",                 │
│        ...                                                       │
│      }                                                           │
│      (Metadata moved to processing queue for safety)             │
│                                                                  │
│  Step 3: Determine Campaign Type & Process                       │
│  ─────────────────────────────────────────                      │
│  └─→ c_type = "otm" → ProcessOTM.processData()                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                   ProcessOTM (One-To-Many)                       │
│                                                                  │
│  Step 1: Open Split File                                        │
│  ───────────────────────                                        │
│  └─→ File: /path/non_exclude_1.txt (9,950 records)             │
│      Format: Plain text, one mobile per line                    │
│      9876543210                                                  │
│      9876543211                                                  │
│      9876543212                                                  │
│      ...                                                         │
│                                                                  │
│  Step 2: Initialize Platform Connection                         │
│  ─────────────────────────────────                              │
│  ├─→ Get APP_INSTANCE_ID: "beacon-platform-01"                  │
│  ├─→ Get Platform Queue name from DeliveryEngineQueueTon        │
│  └─→ Platform Queue: "beacon:platform:01:queue"                 │
│                                                                  │
│  Step 3: Process Each Line                                      │
│  ──────────────────────────                                     │
│  └─→ Read line by line with BufferedReader:                     │
│      │                                                          │
│      ├─→ Line 1: "9876543210"                                   │
│      │   │                                                      │
│      │   ├─→ Validate Mobile Number                             │
│      │   │   ├─→ Length check: 10 digits ✓                     │
│      │   │   ├─→ Starts with valid prefix (6-9) ✓              │
│      │   │   └─→ Numeric only ✓                                 │
│      │   │                                                      │
│      │   ├─→ Construct JSON Payload:                            │
│      │   │   {                                                  │
│      │   │     "mobile": "9876543210",                          │
│      │   │     "msg": "Get 50% off!",                           │
│      │   │     "header": "PROMO",                               │
│      │   │     "cli_id": "CLI001",                              │
│      │   │     "username": "testuser",                          │
│      │   │     "c_name": "Promo Campaign",                      │
│      │   │     "c_type": "otm",                                 │
│      │   │     "c_lang_type": "text",                           │
│      │   │     "cm_id": 12345,                                  │
│      │   │     "cf_id": 67890,                                  │
│      │   │     "c_f_s_id": 1001,                                │
│      │   │     "priority": 2,                                   │
│      │   │     "dlt_entity_id": "1001234567890123",            │
│      │   │     "dlt_template_id": "1234567890123456789",       │
│      │   │     "scheduled_ts": null,                            │
│      │   │     "app_instance_id": "beacon-platform-01",        │
│      │   │     "timestamp": "2024-01-15 10:30:45.123"          │
│      │   │   }                                                  │
│      │   │                                                      │
│      │   └─→ LPUSH beacon:platform:01:queue → json_payload     │
│      │       (Push to Platform Delivery Engine)                 │
│      │                                                          │
│      ├─→ Line 2: "9876543211"                                   │
│      │   └─→ ... (repeat same process)                          │
│      │                                                          │
│      ├─→ Line 3: "9876543212"                                   │
│      │   └─→ ...                                                │
│      │                                                          │
│      ├─→ ... (continue for all 9,950 records)                   │
│      │                                                          │
│      └─→ Line 9950: "9876593209"                                │
│          └─→ ...                                                 │
│                                                                  │
│  Step 4: Track Progress (Optional)                              │
│  ──────────────────────────                                     │
│  ├─→ Every 1000 records:                                        │
│  │   └─→ Log progress: "Processed 1000/9950 records"           │
│  │                                                              │
│  └─→ Update Redis counter (optional):                           │
│      HINCRBY campaign:12345:stats processed_count 1000          │
│                                                                  │
│  Step 5: File Processing Complete                               │
│  ─────────────────────────────                                  │
│  └─→ Close file                                                  │
│      All 9,950 records sent to Platform ✓                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│              Back to SplitFileConsumer                           │
│                                                                  │
│  Step 4: Cleanup & Update                                       │
│  ───────────────────────                                        │
│  │                                                              │
│  ├─→ Remove from Processing Queue:                              │
│  │   LREM FP:processing:12345:Q 0 metadata                      │
│  │   (Remove the metadata we were processing)                   │
│  │                                                              │
│  ├─→ Update Database:                                           │
│  │   UPDATE campaign_file_splits                                │
│  │   SET status = 'completed',                                  │
│  │       completed_ts = NOW()                                   │
│  │   WHERE c_f_s_id = 1001                                      │
│  │                                                              │
│  └─→ Check if all split files completed:                        │
│      SELECT COUNT(*) FROM campaign_file_splits                  │
│      WHERE cm_id = 12345 AND status != 'completed'              │
│      Result: 4 files remaining                                  │
│      → Campaign not yet complete                                │
│                                                                  │
│  (If all files complete, CampaignFinisher will mark done)       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────┐
│ beacon:platform:01:queue │ ← 9,950 JSON payloads queued
└──────────────────────────┘
         │
         ▼
┌──────────────────────────┐
│   Platform Delivery      │
│   Engine                 │ → Routes to SMS Gateways
└──────────────────────────┘
```

---

## 6. Error Handling & Retry Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                     ERROR HANDLING & RETRY                         │
└────────────────────────────────────────────────────────────────────┘

Scenario: Handover Stage Processing Fails
═══════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│        SplitFileConsumer - Error During Processing              │
│                                                                  │
│  Processing: c_f_s_id = 1002                                     │
│  ───────────────────────────                                    │
│                                                                  │
│  try {                                                           │
│    ├─→ RPOPLPUSH 12345:Q → FP:processing:12345:Q               │
│    │   Metadata for c_f_s_id 1002 obtained                      │
│    │                                                            │
│    ├─→ Open file: /path/non_exclude_2.txt                       │
│    │   ❌ FileNotFoundException thrown!                         │
│    │                                                            │
│    └─→ Exception caught                                         │
│                                                                  │
│  } catch (Exception e) {                                         │
│    │                                                            │
│    ├─→ Log error:                                               │
│    │   "Exception processing c_f_s_id:1002 - FileNotFound"     │
│    │                                                            │
│    ├─→ Get retry_count from metadata:                           │
│    │   current_retry = 0                                        │
│    │                                                            │
│    ├─→ Get MAX_RETRY_COUNT from config:                         │
│    │   MAX_RETRY = 5                                            │
│    │                                                            │
│    └─→ Check: retry_count (0) <= MAX_RETRY (5) ? YES           │
│                                                                  │
│        Retry Logic:                                             │
│        ────────────                                             │
│        ├─→ Increment retry_count: 0 → 1                         │
│        │                                                        │
│        ├─→ Update metadata:                                     │
│        │   {                                                    │
│        │     "c_f_s_id": 1002,                                  │
│        │     "retry_count": 1,  ← Updated                       │
│        │     ...                                                │
│        │   }                                                    │
│        │                                                        │
│        ├─→ Push back to queue:                                  │
│        │   LPUSH 12345:Q → updated_metadata                     │
│        │   LPUSH MediumVolumeDQ → "12345:Q"                     │
│        │                                                        │
│        └─→ Remove from processing queue:                        │
│            LREM FP:processing:12345:Q 0 metadata                │
│                                                                  │
│        Result: Retry will be attempted later ✓                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Later: Retry Attempt
═══════════════════

┌──────────────────────────────────────────────────────────────────┐
│        SplitFileConsumer - Retry #1                             │
│                                                                  │
│  ├─→ RPOPLPUSH MediumVolumeDQ → MediumVolumeDQ                  │
│  │   Get "12345:Q" again                                        │
│  │                                                              │
│  ├─→ RPOPLPUSH 12345:Q → FP:processing:12345:Q                 │
│  │   Get metadata for c_f_s_id 1002 (retry_count = 1)          │
│  │                                                              │
│  ├─→ Open file: /path/non_exclude_2.txt                         │
│  │   ❌ Still FileNotFoundException!                            │
│  │                                                              │
│  └─→ Increment retry_count: 1 → 2                              │
│      Push back to queue again                                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

After Multiple Retries: Max Retry Exceeded
═══════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│        SplitFileConsumer - Retry #5 (Final)                     │
│                                                                  │
│  ├─→ RPOPLPUSH 12345:Q → FP:processing:12345:Q                 │
│  │   Get metadata (retry_count = 5)                             │
│  │                                                              │
│  ├─→ Open file: /path/non_exclude_2.txt                         │
│  │   ❌ Still FileNotFoundException!                            │
│  │                                                              │
│  └─→ Check: retry_count (5) <= MAX_RETRY (5) ? YES (last try)  │
│                                                                  │
│      But this was the final retry!                              │
│      ─────────────────────────────                              │
│      Next time retry_count will be 6 > MAX_RETRY                │
│                                                                  │
│      ├─→ Mark as FAILED:                                        │
│      │   UPDATE campaign_file_splits                            │
│      │   SET status = 'FAILED',                                 │
│      │       reason = 'Maximum retries exceeded - FileNotFound',│
│      │       retry_count = 6                                    │
│      │   WHERE c_f_s_id = 1002                                  │
│      │                                                          │
│      ├─→ Remove from processing queue:                          │
│      │   LREM FP:processing:12345:Q 0 metadata                  │
│      │                                                          │
│      ├─→ Do NOT push back to queue                              │
│      │                                                          │
│      └─→ Send notification (optional):                          │
│          Email/Webhook to user about failure                    │
│                                                                  │
│      Result: File marked as FAILED in database ✓                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Deadlock Detection & Handling
══════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│        Split Stage - Database Deadlock                          │
│                                                                  │
│  Processing campaign cf_id = 67890                              │
│                                                                  │
│  ├─→ INSERT INTO campaign_file_splits (multiple records)        │
│  │   ❌ SQLException: Deadlock found when trying to get lock   │
│  │                                                              │
│  ├─→ Catch SQLException:                                        │
│  │   if (exception.message.contains("deadlock")) {             │
│  │     deadLockFound = true                                     │
│  │   }                                                          │
│  │                                                              │
│  └─→ pushBacktoFileSplitQ(deadLockFound = true)                │
│      │                                                          │
│      └─→ Retry Logic:                                           │
│          ├─→ Get retry_count: 2                                 │
│          ├─→ Do NOT increment (since it's not application error)│
│          │   retry_count stays 2                                │
│          └─→ LPUSH FileSplitQ → metadata (same retry_count)    │
│                                                                  │
│      Result: Will retry immediately without penalty ✓           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. MTM & Template Processing Comparison

```
┌────────────────────────────────────────────────────────────────────┐
│            MTM (Many-To-Many) Processing                           │
└────────────────────────────────────────────────────────────────────┘

File Format: mobile|message
────────────────────────────
9876543210|Hi John, your order #123 is ready for pickup
9876543211|Dear Jane, your invoice amount is Rs.5000
9876543212|Bob, your appointment is confirmed for tomorrow

Processing in ProcessMTM.processData():
────────────────────────────────────────

Read line: "9876543210|Hi John, your order #123 is ready for pickup"
│
├─→ Split by delimiter "|":
│   ├─→ mobile = "9876543210"
│   └─→ message = "Hi John, your order #123 is ready for pickup"
│
├─→ Validate mobile: ✓
│
├─→ Validate message: ✓
│
├─→ Replace line breaks (if any):
│   message = message.replace("\n", "<br>")
│
└─→ Create payload & send:
    {
      "mobile": "9876543210",
      "msg": "Hi John, your order #123 is ready for pickup",
      "header": "SENDER",
      ...
    }


┌────────────────────────────────────────────────────────────────────┐
│              TEMPLATE Processing                                   │
└────────────────────────────────────────────────────────────────────┘

Template Definition:
────────────────────
Template ID: T12345
Message: "Dear {name}, your payment of Rs.{amount} is due on {date}"
Mobile Column: "mobile"
Fields: name, amount, date

File Format: mobile|name|amount|date
─────────────────────────────────────
9876543210|John Smith|1500|2024-01-20
9876543211|Jane Doe|2500|2024-01-22
9876543212|Bob Johnson|3000|2024-01-25

Processing in ProcessTEM.processData():
────────────────────────────────────────

Read line: "9876543210|John Smith|1500|2024-01-20"
│
├─→ Split by delimiter "|":
│   ├─→ [0] mobile = "9876543210"
│   ├─→ [1] name = "John Smith"
│   ├─→ [2] amount = "1500"
│   └─→ [3] date = "2024-01-20"
│
├─→ Validate mobile: ✓
│
├─→ Get template message:
│   "Dear {name}, your payment of Rs.{amount} is due on {date}"
│
├─→ Replace placeholders:
│   ├─→ {name} → "John Smith"
│   ├─→ {amount} → "1500"
│   └─→ {date} → "2024-01-20"
│
├─→ Final message:
│   "Dear John Smith, your payment of Rs.1500 is due on 2024-01-20"
│
└─→ Create payload & send:
    {
      "mobile": "9876543210",
      "msg": "Dear John Smith, your payment of Rs.1500 is due on 2024-01-20",
      "header": "SENDER",
      "template_id": "T12345",
      "dlt_template_id": "1234567890123456789",
      ...
    }


┌────────────────────────────────────────────────────────────────────┐
│         Index-Based vs Name-Based Templates                        │
└────────────────────────────────────────────────────────────────────┘

NAME-BASED Template (Column Headers Required):
───────────────────────────────────────────────
Template: "Hi {name}, OTP is {otp}"
File has headers: mobile,name,otp
9876543210,John,123456

Processing:
├─→ Read header row: mobile,name,otp
├─→ Map columns: {mobile: col0, name: col1, otp: col2}
├─→ Read data row: 9876543210,John,123456
└─→ Replace: {name}→John, {otp}→123456
    Result: "Hi John, OTP is 123456"


INDEX-BASED Template (No Headers Required):
────────────────────────────────────────────
Template: "Hi {0}, OTP is {1}"
File without headers: mobile,field1,field2
9876543210,John,123456

Processing:
├─→ No header row expected
├─→ Read data row: 9876543210,John,123456
├─→ Split: [0]=9876543210, [1]=John, [2]=123456
└─→ Replace: {0}→John, {1}→123456
    Result: "Hi John, OTP is 123456"
```

---

## 8. Database Schema Relationships

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DATABASE SCHEMA DIAGRAM                          │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐
│   campaign_master        │
├──────────────────────────┤
│ id (PK)                  │◄─────┐
│ cli_id                   │      │
│ username                 │      │ ONE campaign_master
│ c_name                   │      │ HAS MANY campaign_files
│ msg                      │      │
│ header                   │      │
│ intl_header              │      │
│ template_id              │      │
│ template_type            │      │
│ template_mobile_column   │      │
│ dlt_entity_id            │      │
│ dlt_template_id          │      │
│ c_type                   │      │
│ c_lang_type              │      │
│ remove_dupe_yn           │      │
│ scheduled_ts             │      │
│ status                   │      │
│ reason                   │      │
│ created_ts               │      │
└──────────────────────────┘      │
                                  │
                                  │
                    ┌─────────────┴──────────────┐
                    │                            │
                    │                            │
┌───────────────────▼───────┐  ┌────────────────▼─────────┐
│   campaign_files           │  │   campaign_schedules     │
├───────────────────────────┤  │   (for scheduled camps)  │
│ id (PK)                   │  └──────────────────────────┘
│ c_id (FK) → cm.id         │◄─────┐
│ group_id                  │      │
│ filename_ori              │      │ ONE campaign_files
│ fileloc                   │      │ HAS MANY campaign_file_splits
│ exclude_group_ids         │      │
│ retry_count               │      │
│ total                     │      │
│ status                    │      │
│ reason                    │      │
│ instance_id               │      │
│ started_ts                │      │
│ created_ts                │      │
└───────────────────────────┘      │
                                   │
                                   │
              ┌────────────────────┴─────────────┐
              │                                  │
┌─────────────▼───────────────┐                 │
│  campaign_file_splits       │                 │
├─────────────────────────────┤                 │
│ c_f_s_id (PK)               │                 │
│ cm_id (FK) → cm.id          │─────────────────┘
│ cf_id (FK) → cf.id          │─────────────────┘
│ fileloc                     │  (child split files)
│ total                       │
│ exclude_count               │
│ retry_count                 │
│ status                      │
│ reason                      │
│ created_ts                  │
│ completed_ts                │
└─────────────────────────────┘


Status Flow:
════════════

campaign_master.status:
  queued → inprogress → completed/failed

campaign_files.status:
  queued → inprogress → completed/failed

campaign_file_splits.status:
  queued → inprogress → completed/failed


Example Data:
═════════════

campaign_master (id: 12345)
├─→ c_name: "Summer Sale Campaign"
├─→ msg: "Get 50% off on all products!"
├─→ header: "SALE"
├─→ c_type: "otm"
└─→ status: "inprogress"

campaign_files (id: 67890, c_id: 12345)
├─→ filename_ori: "contacts.csv"
├─→ fileloc: "/files/campaigns/user/contacts_uuid.csv"
├─→ total: 50000
├─→ exclude_group_ids: "101,102"
└─→ status: "inprogress"

campaign_file_splits (cf_id: 67890)
├─→ (c_f_s_id: 1001) fileloc: split_1.txt, total: 9950, status: completed
├─→ (c_f_s_id: 1002) fileloc: split_2.txt, total: 9950, status: completed
├─→ (c_f_s_id: 1003) fileloc: split_3.txt, total: 9950, status: inprogress
├─→ (c_f_s_id: 1004) fileloc: split_4.txt, total: 9950, status: queued
└─→ (c_f_s_id: 1005) fileloc: split_5.txt, total: 10200, status: queued
```

---

## Summary

These detailed diagrams provide a comprehensive view of the Beacon File Processor system, showing:

1. **Complete sequence flows** from user upload to SMS delivery
2. **Module-by-module processing** with step-by-step breakdowns
3. **Error handling** and retry mechanisms
4. **Different campaign types** (OTM, MTM, Template) processing
5. **Database relationships** and status transitions
6. **Redis queue operations** with atomic operations
7. **File format transformations** at each stage

Each stage is designed for **fault tolerance**, **scalability**, and **observability**, ensuring reliable processing of millions of messages daily.
