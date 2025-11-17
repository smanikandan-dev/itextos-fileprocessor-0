# Beacon File Processor - Operational Guide

## 📖 Quick Reference Guide

This document provides practical guidance for operating, troubleshooting, and maintaining the Beacon File Processor system.

---

## 🚀 Getting Started

### Building the Project

```bash
# Build all modules
cd /workspace
mvn clean install

# Build specific module
cd fb-splitstage
mvn clean package

# Skip tests
mvn clean install -DskipTests
```

### Running Locally

```bash
# Deploy to Tomcat/Jetty
cp fb-fileupload/target/fb-fileupload.war /path/to/tomcat/webapps/

# Or run with embedded Jetty (if configured)
java -jar fb-splitstage/target/fb-splitstage.jar
```

### Docker Deployment

```bash
# Build Docker images
docker build -f docker-fileprocessor/Dockerfile_singleton -t fp-fileupload:latest .

# Start all services (Singleton mode)
cd docker-fileprocessor
docker-compose -f docker-compose-singleton.yml up -d

# Start scalable services (Other mode)
docker-compose -f docker-compose-other.yml up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f fb-splitstage

# Stop services
docker-compose down
```

---

## 🔍 Monitoring & Health Checks

### Service Health Check

```bash
# Check if module is running
curl http://localhost:8080/health

# Check Prometheus metrics
curl http://localhost:8080/metrics

# Check in-memory cache status
curl http://localhost:8090/refresh?action=status
```

### Redis Queue Monitoring

```bash
# Connect to Redis
redis-cli -h localhost -p 6379

# Check queue lengths
LLEN FileSplitQ
LLEN DeliveryQ
LLEN ExcludeQ

# View queue contents (first 10 items)
LRANGE FileSplitQ 0 9

# Get all keys matching pattern
KEYS campaign_id_*

# Check heartbeat keys
KEYS heartbeat:*

# Get heartbeat value
GET heartbeat:FP-SplitStage:FileSplitQConsumer:instance1:thread1

# Check TTL (time to live)
TTL heartbeat:FP-SplitStage:FileSplitQConsumer:instance1:thread1
```

### Database Monitoring

```sql
-- Check campaign status distribution
SELECT status, COUNT(*) as count 
FROM campaign_master 
GROUP BY status;

-- Check campaigns stuck in processing
SELECT cm_id, username, status, created_date, 
       TIMESTAMPDIFF(MINUTE, created_date, NOW()) as minutes_elapsed
FROM campaign_master 
WHERE status IN ('QUEUED', 'PROCESSING', 'SPLITTING')
  AND created_date < DATE_SUB(NOW(), INTERVAL 1 HOUR);

-- Check split files completion rate
SELECT cm_id, 
       COUNT(*) as total_splits,
       SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) as completed,
       SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed,
       SUM(CASE WHEN status = 'PROCESSING' THEN 1 ELSE 0 END) as processing
FROM campaign_file_splits
GROUP BY cm_id
HAVING total_splits > completed + failed;

-- Check recent failed campaigns
SELECT cm_id, username, file_location, status, 
       created_date, completed_date
FROM campaign_master
WHERE status = 'FAILED'
  AND created_date > DATE_SUB(NOW(), INTERVAL 24 HOUR)
ORDER BY created_date DESC
LIMIT 20;

-- Check processing time statistics
SELECT 
    AVG(TIMESTAMPDIFF(MINUTE, created_date, completed_date)) as avg_minutes,
    MIN(TIMESTAMPDIFF(MINUTE, created_date, completed_date)) as min_minutes,
    MAX(TIMESTAMPDIFF(MINUTE, created_date, completed_date)) as max_minutes
FROM campaign_master
WHERE status = 'COMPLETED'
  AND completed_date > DATE_SUB(NOW(), INTERVAL 7 DAY);
```

### Log Monitoring

```bash
# View real-time logs
tail -f /logs/fb-splitstage/splitstage.log

# Search for errors in last hour
grep -i "error" /logs/fb-splitstage/splitstage-$(date +%Y-%m-%d).log | tail -100

# Search for specific campaign ID
grep "cm_id:12345" /logs/fb-handoverstage/*.log

# Count errors by type
grep -i "exception" /logs/*/$(date +%Y-%m-%d)*.log | \
  awk -F: '{print $NF}' | sort | uniq -c | sort -rn

# Monitor file processing rate
grep "Request consumed" /logs/fb-splitstage/splitstage.log | \
  awk '{print $1, $2}' | uniq -c
```

---

## 🔧 Common Operations

### 1. Upload a File via API

```bash
# Upload campaign file
curl -X POST "http://localhost:8080/save" \
  -F "file=@campaign_data.csv" \
  -F "username=testuser" \
  -F "frompage=campaign"

# Expected response
{
  "statusCode": 200,
  "total": 50000,
  "total_human": "50K",
  "uploaded_files": {
    "success": [
      {
        "filename": "campaign_data.csv",
        "r_filename": "campaign_data_abc123.csv",
        "count": 50000
      }
    ],
    "failed": []
  }
}
```

### 2. Refresh In-Memory Cache

```bash
# Refresh all caches
curl "http://localhost:8090/refresh"

# Refresh specific configuration
curl "http://localhost:8090/refresh?type=config"
```

### 3. Manually Push Campaign to Queue

```bash
# Connect to Redis
redis-cli

# Push campaign to FileSplitQ
LPUSH FileSplitQ '{"cm_id":"12345","username":"testuser","file_location":"/files/campaign.csv","campaign_type":"OTM"}'

# Verify
LLEN FileSplitQ
```

### 4. Clear Stuck Queue

```bash
redis-cli

# View stuck campaign queue
LRANGE campaign_id_12345 0 -1

# Remove specific item
LREM campaign_id_12345 0 '{"c_f_s_id":"67890",...}'

# Delete entire queue
DEL campaign_id_12345

# Remove from master DeliveryQ
LREM DeliveryQ 0 "campaign_id_12345"
```

### 5. Manually Update Campaign Status

```sql
-- Mark campaign as QUEUED for reprocessing
UPDATE campaign_master 
SET status = 'QUEUED', retry_count = 0 
WHERE cm_id = 12345;

-- Mark split file for retry
UPDATE campaign_file_splits 
SET status = 'PENDING', retry_count = 0 
WHERE c_f_s_id = 67890;

-- Mark failed files as completed (skip)
UPDATE campaign_file_splits 
SET status = 'COMPLETED' 
WHERE cm_id = 12345 AND status = 'FAILED';
```

---

## 🐛 Troubleshooting Guide

### Issue 1: Campaign Stuck in QUEUED Status

**Symptoms:**
- Campaign remains in QUEUED status for extended period
- No activity in logs

**Diagnosis:**
```bash
# Check if InitialStage poller is running
grep "CampaignMasterPoller" /logs/fb-initialstage/*.log | tail -20

# Check database connection
mysql -u user -p -e "SELECT 1"

# Check if FileSplitQ is backed up
redis-cli LLEN FileSplitQ
```

**Solution:**
1. Restart fb-initialstage module
2. Check database connectivity
3. Verify polling interval configuration
4. Manually push to queue if needed

---

### Issue 2: Split Files Not Processing

**Symptoms:**
- FileSplitQ has items but not being consumed
- SplitStage logs show no activity

**Diagnosis:**
```bash
# Check SplitStage consumers
grep "FileSplitQConsumer" /logs/fb-splitstage/*.log | tail -20

# Check Redis connectivity
redis-cli PING

# Check file system space
df -h

# Verify consumer threads
ps aux | grep splitstage
```

**Solution:**
1. Restart fb-splitstage module
2. Check disk space (need space for split files)
3. Verify Redis connection pool settings
4. Check consumer sleep time configuration

---

### Issue 3: Exclude Processing Slow

**Symptoms:**
- Files stuck in ExcludeQ
- Slow exclude processing

**Diagnosis:**
```sql
-- Check exclude group sizes
SELECT g_id, COUNT(*) as number_count
FROM group_numbers
WHERE g_id IN (SELECT exclude_group_ids FROM campaign_master WHERE cm_id = 12345);

-- Check exclude consumer performance
grep "ExcludeConsumer" /logs/fb-excludeprocessor/*.log | \
  grep "time taken" | tail -20
```

**Solution:**
1. Optimize exclude group queries (add indexes)
2. Increase exclude processor instances
3. Consider caching frequently used exclude lists
4. Check file I/O performance

---

### Issue 4: Handover Stage Failures

**Symptoms:**
- Files failing at handover stage
- Retry count increasing

**Diagnosis:**
```bash
# Check handover logs
grep -i "exception" /logs/fb-handoverstage/*.log | tail -50

# Check processing queue
redis-cli KEYS "processing:campaign_id_*"

# Check file existence
ls -l /files/username/split_files/

# Check delivery engine connectivity
curl http://delivery-engine:8080/health
```

**Solution:**
1. Verify split files exist on disk
2. Check delivery engine availability
3. Verify Redis rpoplpush operations
4. Check network connectivity
5. Review retry count and max retry settings

---

### Issue 5: Memory Issues

**Symptoms:**
- OutOfMemoryError in logs
- Slow processing
- Module crashes

**Diagnosis:**
```bash
# Check JVM heap usage
jstat -gc <pid> 1000 10

# Check module memory
docker stats fb-splitstage

# Review heap dumps
jmap -dump:format=b,file=heap.bin <pid>
```

**Solution:**
1. Increase JVM heap size (-Xmx)
2. Tune garbage collection settings
3. Review connection pool sizes
4. Check for memory leaks (unclosed connections)
5. Optimize file processing (streaming vs loading)

---

### Issue 6: Redis Connection Pool Exhausted

**Symptoms:**
- "Could not get a resource from the pool" errors
- Slow queue operations

**Diagnosis:**
```bash
# Check Redis connections
redis-cli CLIENT LIST

# Check pool configuration
grep "redis.pool" properties/common/*.properties

# Monitor connection acquisition
grep "getConnection" /logs/*/*.log | tail -50
```

**Solution:**
1. Increase pool max total size
2. Reduce pool max wait time
3. Fix connection leaks (ensure .close() called)
4. Review connection timeout settings
5. Consider adding more Redis instances

---

### Issue 7: File Not Found Errors

**Symptoms:**
- FileNotFoundException in logs
- Split files cannot be read

**Diagnosis:**
```bash
# Check file existence
ls -l /files/username/

# Check file permissions
ls -la /files/username/split_file.csv

# Check NFS mount (if using)
mount | grep nfs
df -h | grep nfs

# Check file tracking
redis-cli GET file:tracking:username:filename
```

**Solution:**
1. Verify file paths in database match filesystem
2. Check file system permissions
3. Verify NFS mount is stable
4. Check for premature file deletion
5. Review file cleanup job settings

---

## 🔄 Maintenance Procedures

### Daily Maintenance

```bash
# 1. Check system health
./scripts/health_check.sh

# 2. Monitor queue lengths
redis-cli LLEN FileSplitQ
redis-cli LLEN DeliveryQ

# 3. Check disk space
df -h

# 4. Review error logs
grep -i "error" /logs/*/$(date +%Y-%m-%d)*.log | wc -l

# 5. Check stuck campaigns
# (Run SQL query from monitoring section)
```

### Weekly Maintenance

```bash
# 1. Rotate logs
./docker-fileprocessor/rotate_logs.sh

# 2. Clean old files
# (Automated by fb-cronjobs, verify it ran)

# 3. Database optimization
mysql -u user -p << EOF
OPTIMIZE TABLE campaign_master;
OPTIMIZE TABLE campaign_file_splits;
ANALYZE TABLE campaign_master;
ANALYZE TABLE campaign_file_splits;
EOF

# 4. Redis memory optimization
redis-cli BGREWRITEAOF

# 5. Review performance metrics
# (Check Grafana dashboards)
```

### Monthly Maintenance

```bash
# 1. Archive old campaigns
# Move campaigns older than 90 days to archive table

# 2. Clean Redis keys
redis-cli SCAN 0 MATCH "old:pattern:*" | xargs redis-cli DEL

# 3. Review and update configurations
# Check for new property additions

# 4. Update dependencies
mvn versions:display-dependency-updates

# 5. Performance tuning review
# Analyze slow queries, optimize indexes
```

---

## 📊 Performance Tuning

### Configuration Tuning

```properties
# Consumer Performance
consumer.sleep.time.ms=100           # Lower = faster but more CPU
thread.sleep.time.ms=5000            # Error recovery delay
consumers.count.per.queue=5          # More consumers = more throughput

# File Processing
file.split.chunk.size=10000          # Records per split file
file.read.buffer.size=8192           # File I/O buffer

# Redis Connection Pool
redis.pool.max.total=50              # Max connections
redis.pool.max.idle=20               # Idle connections
redis.pool.min.idle=10               # Min idle connections
redis.timeout.ms=3000                # Connection timeout

# Database Connection Pool
db.pool.max.total=50
db.pool.max.idle=20
db.pool.min.idle=10

# Retry Settings
max.retry.count=5                    # Max retry attempts
retry.backoff.multiplier=2           # Exponential backoff
```

### JVM Tuning

```bash
# Heap Size
-Xms2g -Xmx4g

# Garbage Collection (G1GC)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:ParallelGCThreads=8
-XX:ConcGCThreads=2
-XX:InitiatingHeapOccupancyPercent=45

# GC Logging
-Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=100M

# Heap Dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdumps/
```

### Database Optimization

```sql
-- Add indexes for common queries
CREATE INDEX idx_campaign_status ON campaign_master(status, created_date);
CREATE INDEX idx_campaign_username ON campaign_master(username, created_date);
CREATE INDEX idx_splits_campaign ON campaign_file_splits(cm_id, status);
CREATE INDEX idx_splits_status ON campaign_file_splits(status, created_date);

-- Partitioning (for large tables)
ALTER TABLE campaign_master
PARTITION BY RANGE (YEAR(created_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

## 🚨 Alert Configuration

### Prometheus Alert Rules

```yaml
groups:
  - name: fileprocessor_alerts
    rules:
      - alert: QueueSizeHigh
        expr: redis_queue_length{queue="FileSplitQ"} > 10000
        for: 5m
        annotations:
          summary: "FileSplitQ size is high"
          description: "Queue length is {{ $value }}"

      - alert: ConsumerDown
        expr: time() - heartbeat_timestamp > 120
        for: 2m
        annotations:
          summary: "Consumer heartbeat missing"
          description: "{{ $labels.module }} - {{ $labels.consumer }}"

      - alert: HighFailureRate
        expr: rate(campaign_failures_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High campaign failure rate"
          description: "Failure rate is {{ $value }}"

      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/files"} < 10737418240
        for: 5m
        annotations:
          summary: "Disk space low on /files"
          description: "Only {{ $value | humanize }}B remaining"
```

### Email Alert Template

```html
Subject: [ALERT] Beacon File Processor - {{ .alert }}

Alert: {{ .alert }}
Severity: {{ .severity }}
Module: {{ .module }}
Time: {{ .timestamp }}

Description:
{{ .description }}

Current Metrics:
- Queue Length: {{ .queue_length }}
- Processing Rate: {{ .processing_rate }}/min
- Error Rate: {{ .error_rate }}%

Action Required:
{{ .action_required }}

Dashboard: {{ .dashboard_url }}
```

---

## 📈 Capacity Planning

### Throughput Calculations

```
Single Consumer Capacity:
- Average processing time per file: 2 seconds
- Files per minute: 30
- Files per hour: 1,800
- Records per hour (10K records/file): 18M

Module Scaling:
- fb-splitstage: 1 instance handles 1M records/hour
- fb-handoverstage: 1 instance handles 500K records/hour
- fb-excludeprocessor: 1 instance handles 750K records/hour

For 10M records/hour:
- splitstage instances: 10
- handoverstage instances: 20
- excludeprocessor instances: 14
```

### Resource Requirements

```
Per Module Instance:
- CPU: 2 cores
- RAM: 4 GB
- Disk: 50 GB (for logs and temp files)
- Network: 100 Mbps

Redis Cluster:
- Master: 16 GB RAM, 4 CPU
- Replica: 16 GB RAM, 4 CPU
- Disk: 100 GB SSD

Database Cluster:
- Master: 32 GB RAM, 8 CPU, 500 GB SSD
- Slaves: 16 GB RAM, 4 CPU, 500 GB SSD

File Storage:
- NFS: 2 TB (grows with campaign files)
- Retention: 90 days
```

---

## 🔐 Security Best Practices

### 1. File Upload Security

```java
// Validate file type
String[] allowedExtensions = {".csv", ".xls", ".xlsx", ".txt", ".zip"};

// Check file size
long maxFileSize = 100 * 1024 * 1024; // 100 MB

// Sanitize filename
String safeFilename = filename.replaceAll("[^a-zA-Z0-9.-]", "_");

// Store in user-specific directory
String userDir = "/files/" + username.toLowerCase() + "/";
```

### 2. Database Security

```sql
-- Use prepared statements (already implemented)
PreparedStatement stmt = conn.prepareStatement(
    "UPDATE campaign_master SET status = ? WHERE cm_id = ?"
);
stmt.setString(1, status);
stmt.setLong(2, campaignId);

-- Grant minimal permissions
GRANT SELECT, INSERT, UPDATE ON campaign_master TO 'fp_user'@'%';
REVOKE DELETE ON campaign_master FROM 'fp_user'@'%';
```

### 3. Redis Security

```bash
# Enable authentication
requirepass your_strong_password

# Bind to specific interfaces
bind 127.0.0.1 192.168.1.10

# Disable dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
```

### 4. Access Control

```bash
# File permissions
chmod 700 /files/
chown appuser:appgroup /files/

# Log file permissions
chmod 640 /logs/*.log
chown appuser:appgroup /logs/

# Configuration files
chmod 600 properties/*.properties
```

---

## 📝 Operational Checklist

### Pre-Deployment Checklist

- [ ] All modules built successfully
- [ ] Configuration files updated for environment
- [ ] Database schema up to date
- [ ] Redis cluster configured and accessible
- [ ] File storage mounted and accessible
- [ ] Monitoring configured (Prometheus, Grafana)
- [ ] Alerts configured
- [ ] Backup procedures in place
- [ ] Documentation updated

### Post-Deployment Checklist

- [ ] All modules started successfully
- [ ] Health check endpoints responding
- [ ] Heartbeats visible in Redis
- [ ] Database connections working
- [ ] Redis connections working
- [ ] File upload working (smoke test)
- [ ] Campaign processing working (end-to-end test)
- [ ] Logs being written
- [ ] Metrics visible in Prometheus
- [ ] Dashboards showing data in Grafana

---

## 🆘 Emergency Procedures

### System-Wide Outage

1. **Check infrastructure**: Database, Redis, Network
2. **Review recent changes**: Deployments, config updates
3. **Check resource usage**: CPU, Memory, Disk
4. **Review logs**: Look for patterns
5. **Restart affected modules**: Start with core services
6. **Verify recovery**: Run smoke tests

### Data Loss Prevention

1. **Database**: Daily backups, point-in-time recovery enabled
2. **Files**: Replicated across multiple nodes
3. **Redis**: AOF persistence enabled, backups hourly
4. **Logs**: Centralized logging with retention

### Rollback Procedure

```bash
# 1. Stop current version
docker-compose down

# 2. Restore database backup (if needed)
mysql -u user -p < backup_YYYY-MM-DD.sql

# 3. Deploy previous version
docker-compose -f docker-compose-v1.0.yml up -d

# 4. Verify functionality
./scripts/smoke_test.sh

# 5. Monitor closely
tail -f /logs/*/*.log
```

---

## 📞 Support Contacts

### Escalation Matrix

**Level 1**: Operations Team
- Monitor dashboards
- Handle routine issues
- Execute runbooks

**Level 2**: Development Team
- Debug application issues
- Fix bugs
- Performance tuning

**Level 3**: Architecture Team
- System design issues
- Major incidents
- Capacity planning

---

This operational guide should help you effectively manage, monitor, and troubleshoot the Beacon File Processor system in production environments.
