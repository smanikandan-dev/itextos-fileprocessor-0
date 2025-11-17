# Beacon File Processor - Documentation Index

## 📖 Overview

Welcome to the **Beacon File Processor** documentation! This system is a comprehensive, enterprise-grade SMS/messaging campaign file processing platform designed to handle high-volume bulk messaging operations.

### What is Beacon File Processor?

Beacon File Processor is a multi-module Java application that:
- Accepts bulk file uploads (CSV, Excel, ZIP)
- Processes and validates campaign data
- Splits large files into manageable chunks
- Applies business rules (templates, DLT compliance, exclusions)
- Hands off processed data to delivery engines via Kafka
- Tracks campaign completion and provides statistics

---

## 📚 Documentation Structure

This repository contains comprehensive documentation organized into the following files:

### 1. 🏗️ **SYSTEM_DOCUMENTATION.md** (Main Reference)
**Complete system architecture and module details**

Includes:
- System overview and architecture diagrams
- Detailed module descriptions (all 16+ modules)
- Technology stack details
- Database schema
- Configuration management
- Performance considerations
- Security aspects
- Operational guidelines

**Read this first** to understand the overall system architecture.

### 2. 🔄 **SEQUENCE_DIAGRAMS.md** (Flow Details)
**Step-by-step sequence diagrams for all major flows**

Includes:
- File upload sequence
- Campaign processing sequence
- Split file processing sequence
- Handover to delivery sequence
- Error handling sequence
- Campaign completion sequence

**Read this** to understand how data flows through the system.

### 3. 🔗 **MODULE_RELATIONSHIPS.md** (Dependencies)
**Module dependencies and inter-module communication**

Includes:
- Module dependency graph
- Component layer architecture
- Redis queue topology
- Database entity relationships
- Inter-module communication patterns
- Technology integration map

**Read this** to understand how modules interact with each other.

### 4. ⚡ **QUICK_REFERENCE.md** (Cheat Sheet)
**Quick reference guide for common operations**

Includes:
- Module summary table
- Key configurations
- Common operations
- Troubleshooting guide
- API endpoints
- Log locations

**Read this** for quick lookups and troubleshooting.

---

## 🚀 Quick Start Guide

### For Developers (New to the Project)

**Step 1**: Read the system overview
```
Start with: SYSTEM_DOCUMENTATION.md
Section: "System Overview" and "Architecture Diagram"
Time: 15 minutes
```

**Step 2**: Understand the data flow
```
Continue with: SEQUENCE_DIAGRAMS.md
Focus on: "File Upload Sequence" and "Campaign Processing Sequence"
Time: 20 minutes
```

**Step 3**: Explore module dependencies
```
Review: MODULE_RELATIONSHIPS.md
Section: "Module Dependency Graph"
Time: 10 minutes
```

**Step 4**: Set up development environment
```
Reference: SYSTEM_DOCUMENTATION.md
Section: "Technology Stack" and "Deployment Architecture"
Time: 30+ minutes
```

### For Operators (System Management)

**Step 1**: Familiarize with system components
```
Start with: QUICK_REFERENCE.md
Section: "Module Quick Reference" and "Technology Stack"
Time: 10 minutes
```

**Step 2**: Learn operational procedures
```
Review: SYSTEM_DOCUMENTATION.md
Section: "Operational Aspects"
Time: 15 minutes
```

**Step 3**: Understand monitoring
```
Reference: QUICK_REFERENCE.md
Section: "Monitoring & Alerts" and "Troubleshooting"
Time: 15 minutes
```

### For Architects (System Design)

**Step 1**: Review architecture patterns
```
Start with: SYSTEM_DOCUMENTATION.md
Section: "Architecture Diagram" and "Key Design Patterns"
Time: 30 minutes
```

**Step 2**: Study component interactions
```
Continue with: MODULE_RELATIONSHIPS.md
All sections
Time: 30 minutes
```

**Step 3**: Analyze scalability and performance
```
Review: SYSTEM_DOCUMENTATION.md
Section: "Performance Considerations" and "Scalability"
Time: 20 minutes
```

---

## 🎯 Use Cases & Scenarios

### Scenario 1: New Campaign Upload
```
User Action:
  - User uploads CSV file with 500K mobile numbers
  - Campaign type: OTM (One-to-Many)
  - Single message to all

System Flow:
  1. fb-fileupload: Receives and stores file
  2. Database: Records campaign metadata
  3. fb-initialstage: Polls and queues campaign
  4. fb-splitstage: Splits into 5 files (100K each)
  5. fb-handoverstage: Processes and sends to Kafka
  6. fb-campaignfinisher: Marks complete, cleans up

Time: ~10-30 minutes (depends on volume)

Documentation: See SEQUENCE_DIAGRAMS.md
```

### Scenario 2: Template-Based Campaign
```
User Action:
  - User uploads CSV with columns: mobile, name, balance
  - Campaign type: TEM (Template)
  - Template: "Hello {name}, your balance is {balance}"

System Flow:
  1-4. Same as Scenario 1
  5. fb-handoverstage:
     - Reads each line
     - Applies template substitution
     - Sends personalized messages to Kafka
  6. Same as Scenario 1

Documentation: See SYSTEM_DOCUMENTATION.md > "Campaign Types"
```

### Scenario 3: System Scaling
```
Scenario:
  - Current: 1M messages/hour
  - Required: 5M messages/hour

Solution:
  1. Scale fb-splitstage: Add 2 more instances
  2. Scale fb-handoverstage: Add 3 more instances
  3. Add Redis slave: Distribute queue load
  4. Increase Kafka partitions: Improve throughput
  5. Tune split size: Reduce to 50K per split

Documentation: See SYSTEM_DOCUMENTATION.md > "Scalability"
```

---

## 🏛️ System Architecture at a Glance

```
┌─────────────┐
│  File Upload │  (HTTP API)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Database   │  (MariaDB)
└──────┬──────┘
       │
       ▼ (Polling)
┌─────────────┐
│ Initial Stage│
└──────┬──────┘
       │
       ▼ (Queue)
┌─────────────┐
│ Split Stage │
└──────┬──────┘
       │
       ▼ (Queue)
┌─────────────┐
│Handover Stage│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Kafka     │  (Message Delivery)
└─────────────┘
```

**Key Components:**
- **Web Layer**: File upload servlets
- **Processing Layer**: Pollers, consumers, handlers
- **Storage Layer**: Database, Redis, File system
- **Messaging Layer**: Kafka for delivery
- **Support Layer**: Monitoring, logging, configuration

---

## 🔧 Technology Overview

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Language | Java | 21 | Application development |
| Build | Maven | 3.x | Dependency & build management |
| Server | WildFly | Latest | Application server |
| Database | MariaDB | 10.x | Data persistence |
| Cache/Queue | Redis | 6.x | Queue management & caching |
| Messaging | Kafka | 2.8.0 | Message delivery |
| Logging | Log4j2 | 2.17.0 | Application logging |
| Parsing | Commons CSV, POI | Latest | File parsing |

---

## 📊 Key Metrics & Scale

| Metric | Value |
|--------|-------|
| Modules | 16+ WAR/JAR modules |
| File Formats | CSV, XLS, XLSX, ZIP |
| Campaign Types | 5 (OTM, MTM, TEM, GROUP, QUICK) |
| Split Size | 100,000 records (configurable) |
| Max Retries | 5 (configurable) |
| Databases | 3 (Campaign, Accounts, Config) |
| Queue Types | 5+ (Split, Delivery, Processing, etc.) |

---

## 🛠️ Development Setup

### Prerequisites
```bash
# Required
- Java 21 JDK
- Maven 3.6+
- MariaDB 10.x
- Redis 6.x
- Kafka 2.8.x
- WildFly/JBoss

# Optional
- Docker & Docker Compose
- Git
```

### Build Steps
```bash
# Clone repository
git clone <repository-url>
cd workspace

# Build all modules
mvn clean install

# Build specific module
cd fb-utils
mvn clean package

# Build WAR files
cd fb-fileupload
mvn clean package
# Output: target/fb-fileupload.war
```

### Configuration
```bash
# 1. Set up database
mysql -u root -p < schema.sql

# 2. Update properties
vim properties/common/global.properties
# Configure DB, Redis, Kafka endpoints

# 3. Deploy to WildFly
cp target/*.war $WILDFLY_HOME/standalone/deployments/

# 4. Initialize modules
curl -X GET http://localhost:8080/fb-initialstage/init
curl -X GET http://localhost:8080/fb-splitstage/init
curl -X GET http://localhost:8080/fb-handoverstage/init
```

---

## 🐛 Common Issues & Solutions

### Issue 1: Module Not Starting
**Symptoms**: WAR deployed but servlets not responding

**Solution**:
1. Check logs: `/opt/jboss/wildfly/logs/server.log`
2. Verify dependencies: fb-utils.jar in lib folder
3. Check database connectivity
4. Verify Redis connectivity

**Reference**: QUICK_REFERENCE.md > Troubleshooting

### Issue 2: Files Not Processing
**Symptoms**: Files uploaded but stuck in queue

**Solution**:
1. Check consumer threads are running
2. Verify Redis queue: `redis-cli LLEN FileSplitQ`
3. Check heartbeat: `redis-cli KEYS "HB:*"`
4. Review consumer logs

**Reference**: SEQUENCE_DIAGRAMS.md > Error Handling

### Issue 3: Performance Degradation
**Symptoms**: Processing slowing down

**Solution**:
1. Check database connection pool
2. Monitor Redis memory usage
3. Review Kafka lag
4. Check disk I/O for file operations
5. Increase consumer threads

**Reference**: SYSTEM_DOCUMENTATION.md > Performance Considerations

---

## 📈 Monitoring & Observability

### Health Checks
```bash
# Check Redis
redis-cli ping

# Check database
mysql -u user -p -e "SELECT 1"

# Check Kafka
kafka-topics.sh --list --bootstrap-server localhost:9092

# Check heartbeats
redis-cli KEYS "HB:*" | wc -l
```

### Key Metrics
```bash
# Queue depths
redis-cli LLEN FileSplitQ
redis-cli KEYS "DeliveryQ_*" | while read key; do echo "$key: $(redis-cli LLEN $key)"; done

# Processing status
mysql -u user -p -e "SELECT status, COUNT(*) FROM campaign_file_splits GROUP BY status"

# Error rates
tail -f /opt/jboss/wildfly/logs/error/error.log
```

### Prometheus Metrics
```
# Access Prometheus exporter
http://localhost:1075/metrics
```

---

## 🔐 Security Best Practices

1. **File Upload**: Validate file types, limit sizes, scan for malware
2. **Database**: Use prepared statements, parameterized queries
3. **Redis**: Enable authentication, use ACLs
4. **Kafka**: Configure SSL, enable ACLs
5. **API**: Implement authentication, rate limiting
6. **Logging**: Sanitize sensitive data, secure log files
7. **Files**: User-based isolation, permission checks

**Reference**: SYSTEM_DOCUMENTATION.md > Security Considerations

---

## 🚦 Deployment Checklist

Before going to production:

- [ ] Database schema created and indexed
- [ ] Redis instances configured and secured
- [ ] Kafka topics created with proper partitioning
- [ ] File storage paths with correct permissions
- [ ] Configuration files reviewed and updated
- [ ] Monitoring and alerting configured
- [ ] Log rotation set up
- [ ] Backup and disaster recovery plan
- [ ] Performance testing completed
- [ ] Security audit performed
- [ ] Documentation updated
- [ ] Team training completed

---

## 📞 Support & Contribution

### Getting Help
1. **Documentation**: Start with the relevant .md file
2. **Logs**: Check application and error logs
3. **Monitoring**: Review Prometheus/Grafana dashboards
4. **Database**: Query campaign and split status
5. **Team**: Contact system administrator

### Contributing
1. Follow existing code patterns
2. Update documentation for new features
3. Add tests for new functionality
4. Follow security best practices
5. Update CHANGELOG.md

---

## 📖 Learning Path

### Week 1: Understanding
- Day 1-2: Read SYSTEM_DOCUMENTATION.md
- Day 3-4: Study SEQUENCE_DIAGRAMS.md
- Day 5: Review MODULE_RELATIONSHIPS.md

### Week 2: Hands-On
- Day 1-2: Set up local development environment
- Day 3-4: Build and deploy modules
- Day 5: Upload test campaign and trace flow

### Week 3: Deep Dive
- Day 1-2: Study specific modules (fb-utils, fb-splitstage)
- Day 3-4: Understand error handling and retry logic
- Day 5: Performance tuning and optimization

### Week 4: Operations
- Day 1-2: Monitoring and alerting setup
- Day 3-4: Troubleshooting scenarios
- Day 5: Production deployment preparation

---

## 🎓 Additional Resources

### Internal Documentation
- `SYSTEM_DOCUMENTATION.md` - Complete system reference
- `SEQUENCE_DIAGRAMS.md` - Flow diagrams
- `MODULE_RELATIONSHIPS.md` - Architecture details
- `QUICK_REFERENCE.md` - Quick lookup guide

### Code Documentation
- JavaDoc in source code
- Inline comments for complex logic
- README files in module directories

### External Resources
- Apache Commons: https://commons.apache.org/
- Redis Documentation: https://redis.io/documentation
- Kafka Documentation: https://kafka.apache.org/documentation/
- MariaDB Documentation: https://mariadb.org/documentation/

---

## 📝 Change Log

### Version 1.0 (Current)
- Initial documentation release
- Complete system architecture documented
- All sequence diagrams created
- Module relationships mapped
- Quick reference guide added

### Future Enhancements
- API documentation (OpenAPI/Swagger)
- Video tutorials
- Interactive diagrams
- Code examples and snippets
- FAQ section

---

## 🌟 Key Features

✅ **Scalable**: Horizontal scaling support  
✅ **Resilient**: Retry mechanisms and error handling  
✅ **Flexible**: Multiple campaign types supported  
✅ **Traceable**: Comprehensive logging and monitoring  
✅ **Efficient**: Streaming file processing  
✅ **Secure**: Multiple security layers  
✅ **Maintainable**: Modular architecture  
✅ **Production-Ready**: Battle-tested design patterns  

---

## 📧 Contact

For questions, issues, or contributions:
- **Documentation Issues**: Update relevant .md file
- **Code Issues**: Check logs and monitoring first
- **Feature Requests**: Discuss with architecture team
- **Emergency Support**: Contact system administrator

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**System**: Beacon File Processor  
**Status**: Production

---

## 🎯 Next Steps

1. **Start Here**: Read SYSTEM_DOCUMENTATION.md Section 1-2
2. **Then**: Review SEQUENCE_DIAGRAMS.md for your use case
3. **Next**: Reference MODULE_RELATIONSHIPS.md for architecture
4. **Finally**: Keep QUICK_REFERENCE.md handy for daily operations

**Happy Learning! 🚀**
