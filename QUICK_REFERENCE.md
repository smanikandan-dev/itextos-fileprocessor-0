# Beacon File Processor - Quick Reference Guide

## 🎯 System Purpose
A high-volume SMS/messaging campaign file processing system that handles bulk file uploads, splits them into manageable chunks, applies business rules, and hands them over to delivery engines.

---

## 📦 Module Quick Reference

| Module | Type | Purpose | Dependencies |
|--------|------|---------|-------------|
| **fb-utils** | JAR | Core utilities, DB, Redis, DTOs | Base (none) |
| **fb-fileparser** | JAR | CSV/XLS/XLSX/ZIP parsing | - |
| **fb-logger** | JAR | Centralized logging | - |
| **fb-fileupload** | WAR | File upload API | fb-utils |
| **fb-initialstage** | WAR | DB polling, queue initialization | fb-utils |
| **fb-splitstage** | WAR | File splitting (100K chunks) | fb-utils, fb-fileparser |
| **fb-handoverstage** | WAR | Process & send to Kafka | fb-utils |
| **fb-campaignfinisher** | WAR | Completion tracking, cleanup | fb-utils |
| **fb-groupsprocessor** | WAR | Group campaign handling | fb-utils, fb-fileparser |
| **fb-dltfileprocessor** | WAR | DLT template management | fb-utils |
| **fb-excludeprocessor** | WAR | Exclusion list handling | fb-utils |
| **fb-scheduleprocessor** | WAR | Scheduled campaigns | fb-utils |
| **fb-downloadhandler** | WAR | File/report downloads | fb-utils |
| **fb-cronjobs** | WAR | Maintenance tasks | fb-utils |
| **fb-inmemoryrefresh** | WAR | Cache refresh | fb-utils |
| **beaconlib** | JAR | Standalone library (shaded) | fb-utils, fb-fileparser |

---

## 🔄 Processing Flow (Simple)

```
Upload → Store → Poll → Split → Process → Kafka → Complete
  ↓        ↓       ↓      ↓       ↓         ↓        ↓
 API      File    DB    Redis   Redis    Kafka    Cleanup
```

---

## 🔄 Processing Flow (Detailed)

```
1. FILE UPLOAD
   User → POST /fb-fileupload/save → File stored → DB entry

2. INITIAL STAGE
   Poll DB → campaign_master (status='ready') → Push to FileSplitQ

3. SPLIT STAGE
   Consume FileSplitQ → Read file → Split (100K) → Push to DeliveryQ_{cm_id}

4. HANDOVER STAGE
   Consume DeliveryQ → Process (OTM/MTM/TEM) → Send to Kafka → Update DB

5. CAMPAIGN FINISHER
   Poll completed splits → Aggregate → Mark campaign complete → Cleanup queues
```

---

## 🗄️ Database Tables (Key)

| Table | Purpose |
|-------|---------|
| **campaign_master** | Campaign metadata, status |
| **campaign_files** | Uploaded file info |
| **campaign_file_splits** | Split file tracking |
| **group_master** | Group definitions |
| **group_files** | Group member files |
| **dlt_template_master** | DLT templates |
| **exclude_group_master** | Exclusion lists |
| **config_params** | Configuration KV store |

---

## 🔴 Redis Queues

| Queue | Purpose | Producer | Consumer |
|-------|---------|----------|----------|
| **FileSplitQ** | Campaign metadata for splitting | InitialStage | SplitStage |
| **DeliveryQ_{cm_id}** | Split file metadata per campaign | SplitStage | HandoverStage |
| **SQLUpdateQ** | Database update queries | Various | QueryExecutor |
| **tracking:{type}:{user}** | File cleanup tracking | FileUpload | CronJobs |
| **{product}:processing:{cm_id}** | Temp processing queue | HandoverStage | HandoverStage |

---

## 📊 Campaign Types

| Type | Description | File Format |
|------|-------------|-------------|
| **OTM** | One-to-Many: Same message to all | `mobile_number` |
| **MTM** | Many-to-Many: Different message per mobile | `mobile_number,message` |
| **TEM** | Template: Personalized messages | `mobile_number,var1,var2,...` |
| **GROUP** | Group-based: Predefined groups | Managed in group_master |
| **QUICK** | Quick send: Fast OTM | `mobile_number` |

---

## ⚙️ Configuration (Key Properties)

```properties
# File Paths
FILE_STORE_PATH=/opt/files/campaigns/
CAMPAIGNS_FILE_STORE_PATH=/opt/files/campaigns/
GROUP_FILE_STORE_PATH=/opt/files/groups/
SPLIT_FILE_PATH=/opt/files/splits/

# Queue Names
FILE_SPLIT_QUEUE_NAME=FileSplitQ

# Processing
MAX_RETRY_COUNT=5
SPLIT_FILE_SIZE=100000
CONSUMER_SLEEP_TIME=1000
THREAD_SLEEP_TIME=30000

# Delimiters
LINE_BREAK_REPLACER=<br>
SPLIT_FILE_DELIMITER=~

# Product
PRODUCT_NAME=FileBeacon
```

---

## 🛠️ Technology Stack

| Category | Technology |
|----------|-----------|
| **Language** | Java 21 |
| **Build** | Maven 3.x |
| **Server** | WildFly/JBoss |
| **Database** | MariaDB 10.x |
| **Cache/Queue** | Redis 6.x (Jedis 3.6.0) |
| **Messaging** | Kafka 2.8.0 |
| **Logging** | Log4j2 2.17.0 |
| **Connection Pool** | Apache DBCP2 2.8.0 |
| **File Parsing** | Commons CSV 1.8, Apache POI |
| **JSON** | Jackson 2.12.1, Gson 2.8.8 |
| **Scheduling** | Quartz 2.3.2 |

---

## 🔌 API Endpoints

### File Upload
```http
POST /fb-fileupload/save
Content-Type: multipart/form-data

Parameters:
  - username (required): User identifier
  - frompage (required): campaign|group|template
  - files: CSV/XLS/XLSX/ZIP files

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

### Initialize Modules
- `/fb-initialstage/init` - Start polling threads
- `/fb-splitstage/init` - Start split consumers
- `/fb-handoverstage/init` - Start handover consumers
- `/fb-campaignfinisher/init` - Start completion pollers

---

## 🔧 Common Operations

### Check Campaign Status
```sql
SELECT cm_id, c_name, status, total, created_ts 
FROM campaign_master 
WHERE cm_id = ?;
```

### Check Split Status
```sql
SELECT c_f_s_id, status, total, sent_count, failed_count, retry_count
FROM campaign_file_splits 
WHERE cm_id = ?;
```

### Check Redis Queue
```bash
# Queue length
redis-cli LLEN FileSplitQ

# View queue contents
redis-cli LRANGE FileSplitQ 0 -1

# Check specific delivery queue
redis-cli LLEN DeliveryQ_{cm_id}
```

### Check Heartbeat
```bash
# View heartbeats
redis-cli KEYS "HB:*"

# Check specific module
redis-cli GET "HB:FP-SplitStage:FileSplitQConsumer:instance1:Thread-1"
```

---

## 🐛 Troubleshooting

### Issue: Files not processing
1. Check database status: `SELECT * FROM campaign_master WHERE status != 'COMPLETED'`
2. Check Redis queues: `redis-cli LLEN FileSplitQ`
3. Check consumer logs: `/opt/jboss/wildfly/logs/application/`
4. Check heartbeat: `redis-cli KEYS "HB:*"`

### Issue: Split stage stuck
1. Check FileSplitQ: `redis-cli LLEN FileSplitQ`
2. Check consumer threads: View logs
3. Check file accessibility: Verify file paths
4. Check database connections: Connection pool status

### Issue: Handover stage failing
1. Check DeliveryQ: `redis-cli LLEN DeliveryQ_{cm_id}`
2. Check Kafka connectivity: Kafka logs
3. Check processing queues: `redis-cli LLEN {product}:processing:{cm_id}`
4. Check retry counts: `redis-cli GET retry:{c_f_s_id}`

### Issue: Campaign not completing
1. Check split statuses: `SELECT status, COUNT(*) FROM campaign_file_splits WHERE cm_id=? GROUP BY status`
2. Check finisher logs: `/opt/jboss/wildfly/logs/application/campaignfinisher.log`
3. Check for stuck processing: `redis-cli KEYS "*processing*"`

---

## 📝 Log Locations

```
/opt/jboss/wildfly/logs/
├── application/
│   ├── fileupload.log
│   ├── initialstage.log
│   ├── splitstage.log
│   ├── handoverstage.log
│   ├── campaignfinisher.log
│   └── groupprocessor.log
├── consumer/
│   └── consumer.log
├── producer/
│   └── producer.log
├── kafkasender/
│   └── kafkasender.log
└── error/
    └── error.log
```

---

## 🔒 Security Considerations

1. **File Upload**: Type validation, size limits, user isolation
2. **Database**: Connection pooling, prepared statements
3. **Redis**: Authentication, network isolation
4. **Kafka**: ACLs, encryption (configurable)
5. **File Storage**: User-based folders, permission checks

---

## 📈 Performance Tuning

### Database
- Connection pool size: Adjust based on load
- Query optimization: Add indexes on status, cm_id
- Batch updates: Use SQLUpdateQ

### Redis
- Multiple instances: Load balancing
- Pipeline operations: Batch Redis commands
- Connection pooling: Jedis pool configuration

### File Processing
- Split size: Adjust based on memory (default 100K)
- Consumer threads: Increase for higher throughput
- Streaming: Line-by-line processing for large files

### Kafka
- Batch size: Configure producer batch size
- Compression: Enable compression for bandwidth
- Partitioning: Use campaign ID for partitioning

---

## 🚀 Deployment Checklist

- [ ] MariaDB configured and accessible
- [ ] Redis instances up and running
- [ ] Kafka cluster configured
- [ ] File storage paths created with permissions
- [ ] Log directories created
- [ ] Configuration files updated (global.properties, module.properties)
- [ ] Database tables created
- [ ] config_params table populated
- [ ] WildFly/JBoss configured
- [ ] WAR files deployed
- [ ] Servlets initialized (/init endpoints called)
- [ ] Monitoring configured (Prometheus, Grafana)
- [ ] Log rotation configured
- [ ] Backup strategy in place

---

## 📞 Monitoring & Alerts

### Key Metrics to Monitor
- Queue depths (Redis)
- Consumer thread status (Heartbeat)
- Campaign processing rate
- Error rates
- Database connection pool utilization
- File storage usage
- Kafka lag

### Health Checks
- HTTP endpoint: `/health` (if implemented)
- Redis heartbeat keys: `HB:*`
- Database connectivity
- Kafka producer status
- File system read/write

---

## 🔑 Key Design Principles

1. **Loose Coupling**: Queue-based communication between modules
2. **Scalability**: Horizontal scaling via multiple instances
3. **Resilience**: Retry mechanisms, error handling
4. **Traceability**: Every step logged and tracked
5. **Modularity**: Clear separation of concerns
6. **Configuration-Driven**: Externalized configuration

---

## 💡 Best Practices

### Development
- Use connection pooling for all external resources
- Implement proper exception handling with retries
- Log important events at appropriate levels
- Use prepared statements for SQL
- Validate all inputs

### Operations
- Monitor queue depths regularly
- Set up alerts for stale heartbeats
- Rotate logs regularly
- Clean up old files periodically
- Monitor database growth
- Backup configuration and data

### Performance
- Tune connection pool sizes based on load
- Adjust split file size for optimal memory usage
- Use batch operations where possible
- Cache frequently accessed configuration
- Optimize database queries with indexes

---

## 📚 Additional Resources

- **System Documentation**: See `SYSTEM_DOCUMENTATION.md`
- **Sequence Diagrams**: See `SEQUENCE_DIAGRAMS.md`
- **Module Relationships**: See `MODULE_RELATIONSHIPS.md`
- **Property Files**: `/properties/` directory
- **Docker Setup**: `/docker-fileprocessor/` directory

---

## 🆘 Support

For issues or questions:
1. Check logs in `/opt/jboss/wildfly/logs/`
2. Verify database and Redis connectivity
3. Check configuration in `config_params` table
4. Review module-specific documentation
5. Contact system administrator

---

**Last Updated**: 2024  
**Version**: 1.0  
**System**: Beacon File Processor
