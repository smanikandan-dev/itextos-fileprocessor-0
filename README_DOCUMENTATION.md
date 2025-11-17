# Beacon File Processor - Complete Documentation Suite

## Welcome!

This documentation suite provides comprehensive information about the **Beacon File Processor** system - a distributed bulk SMS campaign processing platform built with Java, Redis, Kafka, and MariaDB.

---

## Documentation Overview

This repository contains detailed documentation across multiple files:

### 📋 [SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md)
**Complete system documentation including:**
- System overview and architecture
- Detailed module descriptions
- Technology stack
- Database schema
- Queue architecture
- Configuration details
- Error handling
- Deployment instructions

**Best for**: Understanding the overall system, architecture decisions, and module responsibilities.

---

### 📊 [FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md)
**Visual flow diagrams showing:**
- Campaign processing sequence
- Module interaction diagrams
- File upload flow
- Split stage flow
- Handover stage flow
- Group processing flow
- Error handling flow
- Queue management patterns

**Best for**: Understanding data flow, process sequences, and how components interact.

---

### 🔧 [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md)
**Practical reference including:**
- Module quick reference table
- Queue reference
- API endpoints
- Database table schemas
- Common operations
- Troubleshooting guide
- Configuration parameters
- Quick commands

**Best for**: Day-to-day operations, troubleshooting, and configuration reference.

---

## Quick Start

### Understanding the System

1. **Start Here**: Read the [System Overview](#system-overview) section below
2. **Architecture**: See [SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md#architecture)
3. **Flow**: Review diagrams in [FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md#campaign-processing-sequence)
4. **Operations**: Use [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md) for daily tasks

---

## System Overview

### What is Beacon File Processor?

Beacon File Processor is an enterprise-grade **bulk SMS campaign processing system** that handles:

- ✅ **File Upload & Validation**: Multiple formats (CSV, TXT, XLS, XLSX, ZIP)
- ✅ **Campaign Management**: OTM, MTM, Template-based, Quick, and Group campaigns
- ✅ **Intelligent File Splitting**: Breaks large files into manageable chunks
- ✅ **Parallel Processing**: Distributed architecture for high throughput
- ✅ **Queue Management**: Redis-based message queuing
- ✅ **SMS Delivery**: Kafka integration for message handover
- ✅ **Error Handling**: Automatic retry with configurable limits
- ✅ **Monitoring**: Real-time heartbeat and health checks

---

### System Architecture (High-Level)

```
┌───────────────────────────────────────────────────────────────┐
│                    BEACON FILE PROCESSOR                       │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐     │
│  │   File     │      │  Initial   │      │   Split    │     │
│  │  Upload    │─────▶│   Stage    │─────▶│   Stage    │     │
│  └────────────┘      └────────────┘      └────────────┘     │
│                                                  │             │
│                                                  ▼             │
│                                          ┌────────────┐       │
│                                          │  Handover  │       │
│                                          │   Stage    │       │
│                                          └─────┬──────┘       │
│                                                │               │
└────────────────────────────────────────────────┼──────────────┘
                                                 ▼
                                        ┌─────────────────┐
                                        │ Kafka / Delivery│
                                        │     Engine      │
                                        └─────────────────┘
```

For detailed diagrams, see [FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md).

---

### Key Features

#### 1. Multiple Campaign Types
- **OTM (One-to-Many)**: Single message to multiple recipients
- **MTM (Many-to-Many)**: Different messages for different recipients
- **TEM (Template)**: Template-based with dynamic placeholders
- **QUICK**: Quick SMS campaigns
- **GROUP**: Group-based campaigns

#### 2. File Format Support
- CSV (Comma-separated values)
- TXT (Plain text)
- XLS (Excel 2003 and earlier)
- XLSX (Excel 2007 and later)
- ZIP (Compressed archives - auto-extract)

#### 3. Intelligent Processing
- **Volume-based Queuing**: Low/Medium/High volume queues
- **Priority Handling**: Priority assignment based on volume
- **Duplicate Removal**: Optional duplicate number filtering
- **Unicode Support**: Multi-language message support
- **Template Processing**: Dynamic placeholder replacement

#### 4. Reliability
- **Automatic Retries**: Up to 5 retries (configurable)
- **Atomic Operations**: RPOPLPUSH for queue safety
- **Crash Recovery**: Processing queues prevent data loss
- **Status Tracking**: Complete audit trail in database

---

## Module Overview

The system consists of 14+ modules, each with a specific responsibility:

| Module | Responsibility |
|--------|---------------|
| **fb-fileupload** | HTTP endpoint for file uploads |
| **fb-initialstage** | Polls database for new campaigns |
| **fb-splitstage** | Splits large files into chunks |
| **fb-handoverstage** | Processes split files and sends to Kafka |
| **fb-groupsprocessor** | Handles group-based campaigns |
| **fb-excludeprocessor** | Filters excluded numbers |
| **fb-dltfileprocessor** | DLT template validation |
| **fb-campaignfinisher** | Marks campaigns as completed |
| **fb-utils** | Shared utilities (DB, Redis, validators) |
| **fb-logger** | Custom logging |
| **fb-fileparser** | File format parsers |
| **fb-cronjobs** | Scheduled maintenance tasks |
| **fb-scheduleprocessor** | Scheduled campaign handling |
| **fb-downloadhandler** | File download functionality |

For detailed module information, see [SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md#module-details).

---

## Data Flow Summary

### Complete Processing Flow

```
1. User uploads file via API (fb-fileupload)
   ↓
2. Campaign created in database with status='queued'
   ↓
3. Initial stage polls database and finds queued campaigns (fb-initialstage)
   ↓
4. Campaign pushed to FileSplitQ (Redis)
   ↓
5. Split stage consumes from FileSplitQ (fb-splitstage)
   ↓
6. File is parsed, validated, and split into chunks
   ↓
7. Split files inserted into database (campaign_file_splits)
   ↓
8. Split metadata pushed to DeliveryQ (Redis)
   ↓
9. Handover stage consumes from DeliveryQ (fb-handoverstage)
   ↓
10. Split file processed based on campaign type (OTM/MTM/TEM)
   ↓
11. Messages pushed to Kafka for delivery
   ↓
12. Status updated to 'completed' in database
   ↓
13. Campaign finisher checks all splits completed (fb-campaignfinisher)
   ↓
14. Campaign marked as 'completed'
```

For visual diagrams, see [FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md#campaign-processing-sequence).

---

## Technology Stack

| Component | Technology | Version |
|-----------|------------|---------|
| **Language** | Java | 21 |
| **Build Tool** | Maven | 3.x |
| **Web Container** | Jetty / Tomcat | 9.4.x |
| **Database** | MariaDB/MySQL | 10.x |
| **Cache/Queue** | Redis | 6.x |
| **Message Broker** | Kafka | 2.8.0 |
| **Logging** | Log4j | 2.17.0 |
| **Containerization** | Docker | Latest |

For complete technology details, see [SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md#technology-stack).

---

## Getting Started

### Prerequisites
- Java 21+
- Maven 3.x
- MariaDB/MySQL 10.x
- Redis 6.x
- Kafka 2.8.x (optional, for full flow)
- Docker & Docker Compose (for containerized deployment)

### Building the Project
```bash
# Clone the repository
git clone <repository-url>
cd beacon-fileprocessor

# Build all modules
mvn clean install

# Build individual module
cd fb-splitstage
mvn clean package
```

### Running Locally
```bash
# Start Redis
docker run -d -p 6379:6379 redis:6-alpine

# Start MariaDB
docker run -d -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=campaigndb \
  mariadb:10

# Configure properties
# Edit properties/common/*.properties

# Deploy WAR files to Tomcat
# Or run using Docker Compose
docker-compose up -d
```

### Deploying with Docker
```bash
# Build Docker images
docker-compose build

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f fb-splitstage
```

---

## Common Operations

### 1. Upload a File
```bash
curl -X POST http://localhost:8080/save \
  -F "username=john@example.com" \
  -F "frompage=campaign" \
  -F "file=@contacts.csv"
```

### 2. Check Campaign Status
```sql
SELECT 
    cm.c_name,
    cm.status as campaign_status,
    cf.filename_ori,
    cf.status as file_status,
    cf.total,
    cf.retry_count
FROM campaign_master cm
JOIN campaign_files cf ON cm.id = cf.c_id
WHERE cm.id = 'cm_12345';
```

### 3. Monitor Queue
```bash
redis-cli LLEN FileSplitQ
redis-cli LLEN DeliveryQ
```

For more operations, see [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md#common-operations).

---

## Troubleshooting

### Common Issues

1. **Files not processing**: Check fb-initialstage logs, verify database connection
2. **Queue not consuming**: Check consumer module logs, verify Redis connection
3. **High retry count**: Check file paths, database health, specific error in logs
4. **Deadlock errors**: System auto-retries, check for concurrent updates

For detailed troubleshooting, see [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md#troubleshooting-guide).

---

## Monitoring

### Health Checks
- **Heartbeat**: Each consumer pushes heartbeat to Redis every 10 seconds
- **Queue Lengths**: Monitor Redis queue lengths
- **Database Status**: Check campaigns in 'inprogress' status
- **Error Logs**: Monitor logs for exceptions

### Key Metrics
- Campaign processing time
- Queue throughput
- Retry rates
- Success/failure rates
- File processing rate

---

## Configuration

### Important Parameters
```properties
# File Processing
SMS_SPLIT_LIMIT=10000
SPLIT_FILE_DELIMITER=~
LINE_BREAK_REPLACER={br}

# Queue Names
FILE_SPLIT_QUEUE_NAME=FileSplitQ
GROUP_QUEUE_NAME=GroupQ

# Retry Configuration
MAX_RETRY_COUNT=5

# Thread Configuration
CONSUMER_SLEEP_TIME=1000
THREAD_SLEEP_TIME=5000
```

For complete configuration, see [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md#configuration-parameters).

---

## API Reference

### File Upload Endpoint

**POST** `/save`

**Parameters**:
- `username` (required): User identifier
- `frompage` (required): "campaign" or "group"
- `file` (required): File to upload

**Response**:
```json
{
  "statusCode": 200,
  "total": 15000,
  "total_human": "15.0K",
  "uploaded_files": {
    "success": [{
      "filename": "contacts.csv",
      "count": 15000
    }],
    "failed": []
  }
}
```

---

## Database Schema

### Key Tables
- `campaign_master`: Campaign metadata
- `campaign_files`: File details
- `campaign_file_splits`: Split file metadata
- `groups_master`: Group information
- `group_contacts`: Group members

For detailed schema, see [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md#database-tables).

---

## Performance Tuning

### Increase Throughput
1. Increase consumer thread count
2. Optimize split limit size
3. Scale Redis connection pool
4. Horizontal scaling (multiple instances)

### Reduce Latency
1. Decrease sleep times
2. Optimize file I/O (use SSD)
3. Co-locate services
4. Database query optimization

For detailed performance tips, see [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md#performance-tuning).

---

## Best Practices

1. **File Naming**: Always use UUID to avoid collisions
2. **Connection Management**: Close connections in finally blocks
3. **Error Handling**: Log with file_id for tracking
4. **Monitoring**: Implement continuous health checks
5. **Configuration**: Externalize all configurations
6. **Security**: Validate user permissions
7. **Logging**: Use appropriate log levels
8. **Testing**: Test with various file formats and sizes

---

## Contributing

When working with this codebase:

1. **Understand the flow**: Read the documentation first
2. **Module isolation**: Each module is independent
3. **Shared utilities**: Use fb-utils for common functionality
4. **Queue patterns**: Follow existing Redis queue patterns
5. **Error handling**: Implement retry logic consistently
6. **Logging**: Use module-specific loggers
7. **Testing**: Test edge cases (large files, empty files, etc.)

---

## Support & Documentation

### Documentation Files
1. **[SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md)**: Complete system reference
2. **[FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md)**: Visual flow diagrams
3. **[QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md)**: Operational reference

### Getting Help
- Check troubleshooting guide
- Review logs for specific errors
- Verify configuration parameters
- Check queue and database status

---

## System Requirements

### Minimum Requirements
- **CPU**: 4 cores
- **RAM**: 8 GB
- **Storage**: 100 GB SSD
- **Network**: 1 Gbps

### Recommended for Production
- **CPU**: 8+ cores
- **RAM**: 16+ GB
- **Storage**: 500 GB SSD
- **Network**: 10 Gbps
- **Database**: Dedicated server
- **Redis**: Dedicated server with persistence
- **Monitoring**: Prometheus + Grafana

---

## Security Considerations

1. **User Authentication**: Validate username/credentials
2. **File Validation**: Validate file types and sizes
3. **SQL Injection**: Use prepared statements
4. **Path Traversal**: Validate file paths
5. **Redis Security**: Use authentication
6. **Database Security**: Use least privilege
7. **Network Security**: Use firewalls and VPNs

---

## Roadmap

Potential future enhancements:
- [ ] Real-time dashboard
- [ ] Advanced analytics
- [ ] Multi-tenant support
- [ ] API rate limiting
- [ ] Webhook notifications
- [ ] S3 integration for file storage
- [ ] Elasticsearch integration for search
- [ ] GraphQL API

---

## License

[Specify license information]

---

## Contact

[Specify contact information]

---

## Acknowledgments

Built with:
- Java
- Redis
- Kafka
- MariaDB
- Docker
- Maven

---

## Document Information

- **Version**: 1.0
- **Last Updated**: 2025-11-17
- **Status**: Production Ready

---

## Next Steps

1. ✅ Read [SYSTEM_DOCUMENTATION.md](SYSTEM_DOCUMENTATION.md) for architecture details
2. ✅ Review [FLOW_DIAGRAMS.md](FLOW_DIAGRAMS.md) to understand data flow
3. ✅ Use [QUICK_REFERENCE_GUIDE.md](QUICK_REFERENCE_GUIDE.md) for operations
4. ✅ Set up development environment
5. ✅ Run sample campaign
6. ✅ Monitor and optimize

---

**Happy Processing! 🚀**
