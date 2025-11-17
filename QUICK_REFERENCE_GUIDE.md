# Beacon File Processor - Quick Reference Guide

## Table of Contents
1. [System At a Glance](#system-at-a-glance)
2. [Module Quick Reference](#module-quick-reference)
3. [Queue Reference](#queue-reference)
4. [API Endpoints](#api-endpoints)
5. [Database Tables](#database-tables)
6. [Common Operations](#common-operations)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [Configuration Parameters](#configuration-parameters)

---

## System At a Glance

### What is Beacon File Processor?
A **distributed bulk SMS campaign processing system** that:
- Accepts file uploads containing mobile numbers and messages
- Splits large files into manageable chunks
- Processes different campaign types (OTM, MTM, Template)
- Hands over to SMS delivery engine via Kafka
- Handles groups, exclusions, and DLT validation

### Key Statistics
| Metric | Value |
|--------|-------|
| Modules | 14+ |
| File Types Supported | CSV, TXT, XLS, XLSX, ZIP |
| Campaign Types | 5 (OTM, MTM, TEM, QUICK, GROUP) |
| Redis Queues | 6+ main queues |
| Default Split Size | 10,000 records/file |
| Max Retry Count | 5 (configurable) |
| Programming Language | Java 21 |

### Processing Capacity
- **High Volume**: > 100,000 records per campaign
- **Medium Volume**: 10,000 - 100,000 records
- **Low Volume**: < 10,000 records
- **Concurrent Processing**: Multiple threads per module
- **Scalability**: Horizontal scaling via Docker

---

## Module Quick Reference

### Core Processing Modules

| Module | Purpose | Input | Output | Key Files |
|--------|---------|-------|--------|-----------|
| **fb-fileupload** | File upload handler | HTTP multipart files | File metadata JSON | `FilesSaver.java` |
| **fb-initialstage** | Campaign poller | Database (campaign_master) | FileSplitQ messages | `CampaignMasterPoller.java` |
| **fb-splitstage** | File splitter | FileSplitQ | DeliveryQ/ExcludeQ | `MasterFileSplitHandler.java` |
| **fb-handoverstage** | Message processor | DeliveryQ | Kafka messages | `SplitFileConsumer.java`, `ProcessOTM/MTM/TEM.java` |
| **fb-groupsprocessor** | Group handler | Database (groups_master) | GroupQ/FileSplitQ | `GroupsMasterPoller.java` |
| **fb-excludeprocessor** | Exclusion filter | ExcludeQ | Filtered DeliveryQ | `ExcludeConsumer.java` |
| **fb-dltfileprocessor** | DLT validator | DLT files | Template validation | `DltTemplateRequestPoller.java` |
| **fb-campaignfinisher** | Completion tracker | Database | Status updates | `PollerCampaignFilesCompleted.java` |

### Support Modules

| Module | Purpose | Details |
|--------|---------|---------|
| **fb-utils** | Shared utilities | DB connections, Redis, validators, constants |
| **fb-logger** | Logging | Custom logging implementation |
| **fb-fileparser** | File parsing | Parsers for CSV, XLS, XLSX, TXT |
| **fb-cronjobs** | Scheduled tasks | Cleanup, archival, maintenance |
| **fb-scheduleprocessor** | Scheduled campaigns | Time-based campaign triggering |
| **fb-downloadhandler** | File downloads | Download processed files |

---

## Queue Reference

### Main Queues

| Queue Name | Type | Producer | Consumer | Message Format |
|------------|------|----------|----------|----------------|
| **FileSplitQ** | LIST | fb-initialstage | fb-splitstage | Campaign metadata (cm_id, cf_id, fileloc, total, msg, c_type) |
| **DeliveryQ** | LIST | fb-splitstage | fb-handoverstage | Split file metadata (c_f_s_id, cm_id, fileloc, total) |
| **ExcludeQ** | LIST | fb-splitstage | fb-excludeprocessor | Split file metadata + exclude_group_ids |
| **GroupQ** | LIST | fb-initialstage | fb-groupsprocessor | Group metadata (group_id, name, status) |
| **LowVolumeDeliveryQ** | LIST | fb-splitstage | fb-handoverstage | Split files with < 10K records |
| **MediumVolumeDeliveryQ** | LIST | fb-splitstage | fb-handoverstage | Split files with 10K-100K records |
| **HighVolumeDeliveryQ** | LIST | fb-splitstage | fb-handoverstage | Split files with > 100K records |

### Queue Patterns

**Producer (LPUSH)**:
```java
jedis.lpush("FileSplitQ", jsonData);
```

**Consumer (RPOP)**:
```java
String data = jedis.rpop("FileSplitQ");
```

**Atomic Consumer (RPOPLPUSH)**:
```java
String campId = jedis.rpoplpush("DeliveryQ", "DeliveryQ");
String splitData = jedis.rpoplpush(campId, "processing:" + campId);
// Process...
jedis.lrem("processing:" + campId, 1, splitData);
```

---

## API Endpoints

### File Upload API

**Endpoint**: `POST /save`  
**Content-Type**: `multipart/form-data`  
**Module**: fb-fileupload

**Parameters**:
```
username (required): User identifier
frompage (required): "campaign" or "group"
file (required): File to upload (CSV/TXT/XLS/XLSX/ZIP)
```

**Success Response** (200):
```json
{
  "statusCode": 200,
  "total": 15000,
  "total_human": "15.0K",
  "uploaded_files": {
    "success": [
      {
        "filename": "contacts.csv",
        "r_filename": "contacts_uuid-xxx.csv",
        "count": 15000
      }
    ],
    "failed": []
  }
}
```

**Error Response** (500):
```json
{
  "statusCode": 500,
  "code": 500,
  "error": "Internal Server Error",
  "message": "Error processing file"
}
```

---

## Database Tables

### campaign_master
**Purpose**: Stores campaign metadata

| Column | Type | Description |
|--------|------|-------------|
| id | VARCHAR(50) | Primary key, campaign ID |
| cli_id | VARCHAR(50) | Client ID |
| username | VARCHAR(100) | Username |
| c_name | VARCHAR(200) | Campaign name |
| msg | TEXT | Message content |
| header | VARCHAR(20) | SMS header |
| template_id | VARCHAR(50) | Template ID (for TEM) |
| c_type | VARCHAR(10) | Campaign type (OTM/MTM/TEM/QUICK/GROUP) |
| c_lang_type | VARCHAR(20) | Language type (text/unicode) |
| status | VARCHAR(20) | Status (queued/inprogress/completed/failed) |
| created_ts | TIMESTAMP | Creation timestamp |

**Indexes**:
- `idx_status` on (status)
- `idx_cli_id` on (cli_id)

---

### campaign_files
**Purpose**: Stores file details linked to campaigns

| Column | Type | Description |
|--------|------|-------------|
| id | VARCHAR(50) | Primary key, file ID |
| c_id | VARCHAR(50) | Foreign key to campaign_master |
| filename_ori | VARCHAR(500) | Original filename |
| fileloc | VARCHAR(1000) | File location path |
| total | INT | Total records in file |
| status | VARCHAR(20) | Status |
| retry_count | INT | Retry attempts |
| instance_id | VARCHAR(50) | Processing instance ID |
| started_ts | TIMESTAMP | Processing start time |
| completed_ts | TIMESTAMP | Processing end time |

**Indexes**:
- `idx_status` on (status)
- `idx_c_id` on (c_id)

---

### campaign_file_splits
**Purpose**: Stores split file metadata

| Column | Type | Description |
|--------|------|-------------|
| id | VARCHAR(50) | Primary key, split file ID |
| c_f_id | VARCHAR(50) | Foreign key to campaign_files |
| filename | VARCHAR(500) | Split filename |
| fileloc | VARCHAR(1000) | Split file location |
| total | INT | Records in this split |
| status | VARCHAR(20) | Status |
| retry_count | INT | Retry attempts |
| created_ts | TIMESTAMP | Creation timestamp |

**Indexes**:
- `idx_status` on (status)
- `idx_c_f_id` on (c_f_id)

---

### groups_master
**Purpose**: Stores group information

| Column | Type | Description |
|--------|------|-------------|
| id | VARCHAR(50) | Primary key, group ID |
| cli_id | VARCHAR(50) | Client ID |
| name | VARCHAR(200) | Group name |
| total | INT | Total members |
| status | VARCHAR(20) | Status |
| created_ts | TIMESTAMP | Creation timestamp |

---

### group_contacts
**Purpose**: Stores group member details

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT | Primary key |
| group_id | VARCHAR(50) | Foreign key to groups_master |
| mobile | VARCHAR(20) | Mobile number |
| name | VARCHAR(100) | Contact name |
| created_ts | TIMESTAMP | Creation timestamp |

**Indexes**:
- `idx_group_id` on (group_id)

---

## Common Operations

### 1. Upload a File
```bash
curl -X POST http://localhost:8080/save \
  -F "username=john@example.com" \
  -F "frompage=campaign" \
  -F "file=@contacts.csv"
```

---

### 2. Create a Campaign (Manual DB Insert)
```sql
-- Insert campaign
INSERT INTO campaign_master (
    id, cli_id, username, c_name, msg, header, 
    c_type, c_lang_type, status, created_ts
) VALUES (
    'cm_12345', '1001', 'john@example.com', 
    'Test Campaign', 'Hello World', 'SMSHEADER',
    'OTM', 'text', 'queued', NOW()
);

-- Insert campaign file
INSERT INTO campaign_files (
    id, c_id, filename_ori, fileloc, total, 
    status, retry_count, created_ts
) VALUES (
    'cf_67890', 'cm_12345', 'contacts.csv',
    '/path/to/contacts.csv', 10000,
    'queued', 0, NOW()
);
```

---

### 3. Check Campaign Status
```sql
SELECT 
    cm.id as campaign_id,
    cm.c_name,
    cm.status as campaign_status,
    cf.id as file_id,
    cf.filename_ori,
    cf.status as file_status,
    cf.total,
    cf.retry_count
FROM campaign_master cm
JOIN campaign_files cf ON cm.id = cf.c_id
WHERE cm.id = 'cm_12345';
```

---

### 4. Check Split Files
```sql
SELECT 
    id,
    filename,
    total,
    status,
    retry_count,
    created_ts
FROM campaign_file_splits
WHERE c_f_id = 'cf_67890'
ORDER BY created_ts;
```

---

### 5. Monitor Queue Length
```bash
# Connect to Redis
redis-cli

# Check FileSplitQ length
LLEN FileSplitQ

# Check DeliveryQ length
LLEN LowVolumeDeliveryQ
LLEN MediumVolumeDeliveryQ
LLEN HighVolumeDeliveryQ

# View queue contents (without removing)
LRANGE FileSplitQ 0 -1
```

---

### 6. Check Processing Status
```sql
-- Get campaigns in progress
SELECT * FROM campaign_master 
WHERE status = 'inprogress';

-- Get files being processed
SELECT * FROM campaign_files 
WHERE status = 'inprogress';

-- Get split files pending/processing
SELECT status, COUNT(*) 
FROM campaign_file_splits 
GROUP BY status;
```

---

### 7. Retry Failed Campaign
```sql
-- Reset campaign status
UPDATE campaign_master 
SET status = 'queued', reason = NULL 
WHERE id = 'cm_12345';

-- Reset file status
UPDATE campaign_files 
SET status = 'queued', retry_count = 0, reason = NULL 
WHERE c_id = 'cm_12345';
```

---

### 8. Check Heartbeat (Monitoring)
```bash
# Redis keys for heartbeat
redis-cli

# List all heartbeat keys
KEYS *heartbeat*

# Get specific consumer heartbeat
GET FP-SplitStage:FileSplitQConsumer:instance-01:thread-01

# Check timestamp
# Format: yyyy-MM-dd HH:mm:ss
```

---

## Troubleshooting Guide

### Issue: Files Not Being Processed

**Symptoms**:
- Campaigns remain in 'queued' status
- No logs in initialstage

**Checks**:
1. Check if fb-initialstage is running
2. Check database connection
3. Check if polls are happening (check logs)
4. Verify status in database is 'queued' or 'new'

**Solution**:
```bash
# Check logs
tail -f logs/initialstage.log

# Restart initialstage
docker restart fb-initialstage
```

---

### Issue: FileSplitQ Messages Not Consumed

**Symptoms**:
- FileSplitQ has messages but not being processed
- Campaigns stuck in 'inprogress'

**Checks**:
1. Check if fb-splitstage is running
2. Check Redis connection
3. Check queue name configuration
4. Check file paths exist

**Solution**:
```bash
# Check queue
redis-cli LLEN FileSplitQ

# Check splitstage logs
tail -f logs/splitstage.log

# Restart splitstage
docker restart fb-splitstage
```

---

### Issue: Split Files Not Being Handed Over

**Symptoms**:
- Split files created but not sent to Kafka
- DeliveryQ has messages but not consumed

**Checks**:
1. Check if fb-handoverstage is running
2. Check Kafka connection
3. Check split file paths
4. Check campaign type processing

**Solution**:
```bash
# Check handoverstage logs
tail -f logs/handoverstage.log

# Check Kafka connectivity
# (depends on your Kafka setup)

# Restart handoverstage
docker restart fb-handoverstage
```

---

### Issue: High Retry Count

**Symptoms**:
- retry_count reaching MAX_RETRY
- Files marked as 'FAILED'

**Root Causes**:
- File not found (path issue)
- Database deadlocks
- Redis connection issues
- Invalid file format

**Solution**:
1. Check logs for specific error
2. Verify file exists at specified path
3. Check database health
4. Verify Redis connectivity
5. Fix root cause and reset retry_count

---

### Issue: Deadlock in Database

**Symptoms**:
- "Deadlock found when trying to get lock" in logs
- Transactions rolling back

**Solution**:
- System automatically retries without incrementing retry_count
- If persistent, check for:
  - Long-running transactions
  - Lock timeout settings
  - Concurrent updates to same records

---

### Issue: Redis Connection Timeout

**Symptoms**:
- "JedisConnectionException" in logs
- "Could not get a resource from the pool"

**Solution**:
```bash
# Check Redis health
redis-cli PING

# Check connection pool settings
# Increase pool size in configuration

# Restart affected module
docker restart <module-name>
```

---

### Issue: File Parsing Errors

**Symptoms**:
- "Unsupported file type" errors
- Zero records after parsing

**Checks**:
1. Verify file format (CSV/XLS/XLSX/TXT)
2. Check file encoding (UTF-8 expected)
3. Check delimiter configuration
4. Verify file is not corrupted

**Solution**:
- Re-upload file with correct format
- Convert file to supported format
- Check file encoding: `file -i filename.csv`

---

## Configuration Parameters

### File Processing

| Parameter | Default | Description |
|-----------|---------|-------------|
| SMS_SPLIT_LIMIT | 10000 | Records per split file |
| SPLIT_FILE_DELIMITER | ~ | Delimiter for split files |
| LINE_BREAK_REPLACER | {br} | Line break replacement string |
| FILE_STORE_PATH | /opt/files/ | Base file storage path |
| CAMPAIGNS_FILE_STORE_PATH | /opt/files/campaigns/ | Campaign files path |
| GROUP_FILE_STORE_PATH | /opt/files/groups/ | Group files path |

---

### Queue Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| FILE_SPLIT_QUEUE_NAME | FileSplitQ | File split queue name |
| GROUP_QUEUE_NAME | GroupQ | Group queue name |
| MAX_RETRY_COUNT | 5 | Maximum retry attempts |
| DE_NEXT_REQUEST_POP_DELAY | 1000 | Delay between requests (ms) |

---

### Thread Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| CONSUMER_SLEEP_TIME | 1000 | Sleep when no data (ms) |
| THREAD_SLEEP_TIME | 5000 | Sleep on error (ms) |
| HEART_BEAT_INTERVAL | 10000 | Heartbeat interval (ms) |

---

### Redis Configuration

| Parameter | Example | Description |
|-----------|---------|-------------|
| REDIS_HOST | localhost | Redis server host |
| REDIS_PORT | 6379 | Redis server port |
| REDIS_PASSWORD | (optional) | Redis authentication |
| REDIS_POOL_SIZE | 50 | Connection pool size |

---

### Database Configuration

| Parameter | Example | Description |
|-----------|---------|-------------|
| CMDB_URL | jdbc:mysql://localhost:3306/campaigndb | Campaign DB URL |
| CMDB_USER | dbuser | Database username |
| CMDB_PASSWORD | dbpass | Database password |
| DB_POOL_SIZE | 20 | Connection pool size |

---

### Kafka Configuration

| Parameter | Example | Description |
|-----------|---------|-------------|
| KAFKA_BROKERS | localhost:9092 | Kafka broker list |
| KAFKA_TOPIC | sms-delivery | Kafka topic name |
| KAFKA_PRODUCER_BATCH_SIZE | 16384 | Batch size |
| KAFKA_LINGER_MS | 10 | Linger time (ms) |

---

## Performance Tuning

### Increase Throughput

1. **Increase Consumer Threads**:
   - Modify thread pool size in servlet configuration
   - More consumers = higher parallel processing

2. **Increase Split Limit**:
   - Larger split files = fewer split files
   - Trade-off: larger files take longer to process

3. **Redis Connection Pool**:
   - Increase pool size for high concurrency
   - Configure connection timeout

4. **Database Connection Pool**:
   - Increase pool size for high DB operations
   - Tune query performance with indexes

---

### Reduce Latency

1. **Decrease Sleep Times**:
   - Reduce CONSUMER_SLEEP_TIME (but avoid busy-waiting)
   - Reduce DE_NEXT_REQUEST_POP_DELAY

2. **Optimize File I/O**:
   - Use SSD for file storage
   - Enable file system caching

3. **Network Optimization**:
   - Co-locate services (reduce network hops)
   - Use dedicated network for Redis/Kafka

---

## Monitoring Checklist

### Daily Checks
- [ ] Check queue lengths (should be decreasing)
- [ ] Check failed campaigns (retry_count > MAX_RETRY)
- [ ] Check heartbeat for all consumers
- [ ] Check disk space for file storage
- [ ] Check error logs for exceptions

### Weekly Checks
- [ ] Analyze processing times (identify bottlenecks)
- [ ] Review database slow queries
- [ ] Check Redis memory usage
- [ ] Archive old completed campaigns
- [ ] Clean up old files

### Monthly Checks
- [ ] Database maintenance (optimize tables)
- [ ] Log rotation
- [ ] Capacity planning (based on growth)
- [ ] Performance baseline updates

---

## Quick Commands Reference

### Docker Commands
```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Restart specific service
docker restart fb-splitstage

# View logs
docker logs -f fb-splitstage

# Check service status
docker ps
```

### Redis Commands
```bash
# Connect to Redis
redis-cli

# Check queue length
LLEN FileSplitQ

# View queue contents
LRANGE FileSplitQ 0 10

# Delete queue
DEL FileSplitQ

# Flush all data (CAUTION!)
FLUSHALL
```

### Database Commands
```bash
# Connect to database
mysql -u root -p campaigndb

# Check table sizes
SELECT 
    table_name, 
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size (MB)"
FROM information_schema.TABLES
WHERE table_schema = 'cm'
ORDER BY (data_length + index_length) DESC;

# Count records
SELECT COUNT(*) FROM campaign_master;
```

---

## Conclusion

This quick reference guide provides:
- **At-a-glance information** for common operations
- **Troubleshooting steps** for typical issues
- **Configuration reference** for tuning
- **Database and queue commands** for monitoring
- **Performance tips** for optimization

Keep this guide handy for day-to-day operations and troubleshooting!

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-17
