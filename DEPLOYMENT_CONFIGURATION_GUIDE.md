# Deployment & Configuration Guide - Beacon File Processor

## 📋 Table of Contents
1. [System Requirements](#system-requirements)
2. [Architecture Components](#architecture-components)
3. [Configuration Files](#configuration-files)
4. [Deployment Options](#deployment-options)
5. [Monitoring & Operations](#monitoring--operations)
6. [Troubleshooting](#troubleshooting)
7. [Performance Tuning](#performance-tuning)

---

## 🖥️ System Requirements

### Minimum Hardware Requirements

#### Production Environment
```
Component                  | Minimum     | Recommended
───────────────────────────┼─────────────┼──────────────
CPU                        | 8 cores     | 16 cores
RAM                        | 16 GB       | 32 GB
Disk Space                 | 500 GB SSD  | 1 TB SSD
Network                    | 1 Gbps      | 10 Gbps
```

#### Development Environment
```
Component                  | Minimum     
───────────────────────────┼─────────────
CPU                        | 4 cores     
RAM                        | 8 GB        
Disk Space                 | 100 GB SSD  
Network                    | 100 Mbps    
```

### Software Requirements
```
Software                   | Version     | Purpose
───────────────────────────┼─────────────┼────────────────────────
Java JDK                   | 21          | Application runtime
Maven                      | 3.6+        | Build tool
MariaDB/MySQL              | 10.5+       | Primary database
Redis                      | 6.0+        | Queue & cache
Docker (optional)          | 20.10+      | Containerization
Elasticsearch (optional)   | 7.12+       | Logging & search
Prometheus (optional)      | 2.30+       | Metrics collection
```

---

## 🏗️ Architecture Components

### Component Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DEPLOYMENT ARCHITECTURE                         │
└─────────────────────────────────────────────────────────────────────┘

                        ┌──────────────────┐
                        │   Load Balancer  │
                        │  (Nginx/HAProxy) │
                        └────────┬─────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  File Upload    │    │  File Upload    │    │  File Upload    │
│  Module         │    │  Module         │    │  Module         │
│  (Port 8080)    │    │  (Port 8081)    │    │  (Port 8082)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │  Shared File System    │
                    │  (NFS/GlusterFS)       │
                    │  /app/files/           │
                    └────────────────────────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                                               │
         ▼                                               ▼
┌─────────────────┐                            ┌─────────────────┐
│  MariaDB Master │                            │  Redis Cluster  │
│  (Primary DB)   │                            │  (Queues/Cache) │
└────────┬────────┘                            └────────┬────────┘
         │                                               │
         │                                               │
         ▼                                               ▼
┌─────────────────┐                            ┌─────────────────┐
│ MariaDB Slave   │                            │  Redis Sentinel │
│ (Read Replica)  │                            │  (HA/Failover)  │
└─────────────────┘                            └─────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                      PROCESSING MODULES                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Initial Stage  │    │  Initial Stage  │    │  Initial Stage  │
│  Poller         │    │  Poller         │    │  Poller         │
│  Instance 1     │    │  Instance 2     │    │  Instance 3     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                                 │
                                 ▼ Pushes to FileSplitQ
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Split Stage    │    │  Split Stage    │    │  Split Stage    │
│  Consumer 1     │    │  Consumer 2     │    │  Consumer 3     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                                 │
                                 ▼ Pushes to DeliveryQ
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Exclude        │    │  Handover       │    │  Handover       │
│  Processor      │    │  Stage 1        │    │  Stage 2        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                                 │
                                 ▼
                        ┌────────────────┐
                        │   Platform     │
                        │   Delivery     │
                        │   Engine       │
                        └────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                   MONITORING & LOGGING                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Prometheus     │    │  Grafana        │    │  AlertManager   │
│  (Metrics)      │───▶│  (Dashboards)   │◀───│  (Alerts)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │
         │ Scrapes /metrics
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│              All File Processor Modules                             │
│  (Each exposes Prometheus metrics on /metrics endpoint)             │
└─────────────────────────────────────────────────────────────────────┘


┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Elasticsearch  │◀───│  Logstash       │◀───│  Log Files      │
│  (Log Storage)  │    │  (Log Pipeline) │    │  (All Modules)  │
└────────┬────────┘    └─────────────────┘    └─────────────────┘
         │
         │
         ▼
┌─────────────────┐
│  Kibana         │
│  (Log Analysis) │
└─────────────────┘
```

---

## 📝 Configuration Files

### 1. Database Configuration (`properties/common/jndi.properties`)

```properties
# MariaDB Configuration
# =====================

# Campaign Management Database (CMDB)
cmdb.driver=org.mariadb.jdbc.Driver
cmdb.url=jdbc:mariadb://mariadb-master:3306/campaigndb?useUnicode=true&characterEncoding=UTF-8
cmdb.username=fpuser
cmdb.password=encrypted_password_here
cmdb.maxActive=100
cmdb.maxIdle=30
cmdb.minIdle=10
cmdb.maxWait=10000
cmdb.validationQuery=SELECT 1
cmdb.testOnBorrow=true
cmdb.testWhileIdle=true
cmdb.timeBetweenEvictionRunsMillis=30000
cmdb.minEvictableIdleTimeMillis=60000

# Accounts Database
accounts.driver=org.mariadb.jdbc.Driver
accounts.url=jdbc:mariadb://mariadb-master:3306/accountsdb?useUnicode=true&characterEncoding=UTF-8
accounts.username=fpuser
accounts.password=encrypted_password_here
accounts.maxActive=50
accounts.maxIdle=20
accounts.minIdle=5

# Configuration Database
config.driver=org.mariadb.jdbc.Driver
config.url=jdbc:mariadb://mariadb-master:3306/configdb?useUnicode=true&characterEncoding=UTF-8
config.username=fpuser
config.password=encrypted_password_here
config.maxActive=50
config.maxIdle=20
config.minIdle=5
```

### 2. Redis Configuration (`properties/common/redis.properties`)

```properties
# Redis Configuration
# ===================

# Primary Redis Instances (for queues)
redis.instances=3

# Redis Instance 1
redis.1.ip=redis-node-1
redis.1.port=6379
redis.1.id=1
redis.1.password=redis_password_here
redis.1.maxTotal=200
redis.1.maxIdle=50
redis.1.minIdle=10
redis.1.maxWaitMillis=10000
redis.1.testOnBorrow=true
redis.1.testWhileIdle=true
redis.1.timeBetweenEvictionRunsMillis=30000

# Redis Instance 2
redis.2.ip=redis-node-2
redis.2.port=6379
redis.2.id=2
redis.2.password=redis_password_here
redis.2.maxTotal=200
redis.2.maxIdle=50
redis.2.minIdle=10

# Redis Instance 3
redis.3.ip=redis-node-3
redis.3.port=6379
redis.3.id=3
redis.3.password=redis_password_here
redis.3.maxTotal=200
redis.3.maxIdle=50
redis.3.minIdle=10

# Redis for Heartbeat Monitoring
redis.heartbeat.ip=redis-heartbeat
redis.heartbeat.port=6379
redis.heartbeat.password=redis_password_here

# Redis for Unprocessed Numbers
redis.unprocess.instances=2
redis.unprocess.1.ip=redis-unprocess-1
redis.unprocess.1.port=6379
redis.unprocess.2.ip=redis-unprocess-2
redis.unprocess.2.port=6379
```

### 3. Application Configuration (`properties/common/config_params.properties`)

```properties
# Application Configuration
# =========================

# File Paths
FILE_STORE_PATH=/app/files/
CAMPAIGNS_FILE_STORE_PATH=/app/files/campaigns/
GROUP_FILE_STORE_PATH=/app/files/groups/
TEMPLATE_FILE_STORE_PATH=/app/files/templates/
DLT_FILE_STORE_PATH=/app/files/dlt/

# Processing Limits
SMS_SPLIT_LIMIT=10000
MAX_FILE_SIZE_MB=50
MAX_RECORDS_PER_FILE=1000000

# Queue Names
FILE_SPLIT_QUEUE_NAME=FileSplitQ
GROUP_QUEUE_NAME=GroupQ
GROUP_CAMPAIGN_QUEUE_NAME=GroupCampaignQ
GROUP_FILE_SPLIT_QUEUE_NAME=GroupFileSplitQ
DLT_FILE_QUEUE_NAME=DltFileQ
EXCLUDE_NUMBER_FILE_QUEUE_NAME=ExcludeNumberQ
STATS_UPDATE_STATUS_QUERY_QUEUE_NAME=UpdateSQLQ

# Delivery Queue Configuration
DELIVERY_QUEUE_LOW_VOLUME=LowVolumeDQ
DELIVERY_QUEUE_MEDIUM_VOLUME=MediumVolumeDQ
DELIVERY_QUEUE_HIGH_VOLUME=HighVolumeDQ

# Volume Thresholds
LOW_VOLUME_THRESHOLD=10000
MEDIUM_VOLUME_THRESHOLD=100000

# Retry Configuration
MAX_RETRY_COUNT=5
DQ_HO_INFINITE_ATTEMPTS_LOG_LIMIT=5

# Delimiter Configuration
SPLIT_FILE_DELIMITER=|
LINE_BREAK_REPLACER=<br>

# Consumer Configuration
CONSUMER_SLEEP_TIME=1000
THREAD_SLEEP_TIME=5000
DE_NEXT_REQUEST_POP_DELAY=1000
EE_NEXT_REQUEST_POP_DELAY=1000

# Thread Pool Configuration
INITIAL_STAGE_THREADS=3
SPLIT_STAGE_THREADS=10
HANDOVER_STAGE_THREADS=20
EXCLUDE_PROCESSOR_THREADS=5
GROUP_PROCESSOR_THREADS=5

# Numeric Pattern
PATTERN_DECIMALFORMATTER_FOR_DIGITS=0

# Product Name
PRODUCT_NAME=FP
```

### 4. Module-Specific Configuration

#### Initial Stage (`properties/profile/initialstage.properties`)
```properties
# Initial Stage Configuration
MONITORING_INSTANCE_ID=initial-stage-01
FILE_SMS_ALL_STATUS='queued','failed'
INITIAL_STAGE_POLL_INTERVAL=1000
CAMPAIGN_MASTER_POLLER_THREADS=3
GROUP_CAMPAIGN_POLLER_THREADS=2
```

#### Split Stage (`properties/profile/splitstage.properties`)
```properties
# Split Stage Configuration
MONITORING_INSTANCE_ID=split-stage-01
SPLIT_STAGE_CONSUMER_THREADS=10
FILE_SPLIT_QUEUE_CONSUMERS=10
```

#### Handover Stage (`properties/profile/handoverstage.properties`)
```properties
# Handover Stage Configuration
MONITORING_INSTANCE_ID=handover-stage-01
HANDOVER_CONSUMER_THREADS=20
LOW_VOLUME_DQ_CONSUMERS=5
MEDIUM_VOLUME_DQ_CONSUMERS=10
HIGH_VOLUME_DQ_CONSUMERS=15
```

#### Exclude Processor (`properties/profile/excludeprocessor.properties`)
```properties
# Exclude Processor Configuration
MONITORING_INSTANCE_ID=exclude-processor-01
EXCLUDE_CONSUMER_THREADS=5
```

### 5. Logging Configuration (`log4j2.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30">
    <Properties>
        <Property name="logPath">/app/logs</Property>
        <Property name="pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n</Property>
    </Properties>

    <Appenders>
        <!-- File Upload Log -->
        <RollingFile name="FileUploadLog" 
                     fileName="${logPath}/fileupload.log"
                     filePattern="${logPath}/fileupload-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- Initial Stage Log -->
        <RollingFile name="InitialStageLog" 
                     fileName="${logPath}/initialstage.log"
                     filePattern="${logPath}/initialstage-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- Split Stage Log -->
        <RollingFile name="SplitStageLog" 
                     fileName="${logPath}/splitstage.log"
                     filePattern="${logPath}/splitstage-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- Handover Stage Log -->
        <RollingFile name="HandoverStageLog" 
                     fileName="${logPath}/handoverstage.log"
                     filePattern="${logPath}/handoverstage-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- Error Log (All modules) -->
        <RollingFile name="ErrorLog" 
                     fileName="${logPath}/error.log"
                     filePattern="${logPath}/error-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="${pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
            <DefaultRolloverStrategy max="30"/>
            <ThresholdFilter level="ERROR"/>
        </RollingFile>

        <!-- Console Appender -->
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${pattern}"/>
        </Console>
    </Appenders>

    <Loggers>
        <Logger name="com.winnovature.fileuploads" level="debug" additivity="false">
            <AppenderRef ref="FileUploadLog"/>
            <AppenderRef ref="Console"/>
        </Logger>
        
        <Logger name="com.winnovature.initialstate" level="debug" additivity="false">
            <AppenderRef ref="InitialStageLog"/>
            <AppenderRef ref="Console"/>
        </Logger>
        
        <Logger name="com.winnovature.splitstage" level="debug" additivity="false">
            <AppenderRef ref="SplitStageLog"/>
            <AppenderRef ref="Console"/>
        </Logger>
        
        <Logger name="com.winnovature.handoverstage" level="debug" additivity="false">
            <AppenderRef ref="HandoverStageLog"/>
            <AppenderRef ref="Console"/>
        </Logger>

        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="ErrorLog"/>
        </Root>
    </Loggers>
</Configuration>
```

---

## 🚀 Deployment Options

### Option 1: Standalone Deployment (Single Server)

```bash
# Build all modules
cd /workspace
mvn clean package

# Deploy modules
# 1. File Upload Module
cd fb-fileupload/target
java -jar fb-fileupload.war --server.port=8080

# 2. Initial Stage
cd ../../fb-initialstage/target
java -jar fb-initialstage.war

# 3. Split Stage
cd ../../fb-splitstage/target
java -jar fb-splitstage.war

# 4. Handover Stage
cd ../../fb-handoverstage/target
java -jar fb-handoverstage.war

# 5. Exclude Processor
cd ../../fb-excludeprocessor/target
java -jar fb-excludeprocessor.war
```

### Option 2: Docker Deployment

#### docker-compose.yml
```yaml
version: '3.8'

services:
  # MariaDB Master
  mariadb-master:
    image: mariadb:10.5
    container_name: fp-mariadb-master
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: campaigndb
      MYSQL_USER: fpuser
      MYSQL_PASSWORD: fp_password
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"
    networks:
      - fp-network

  # Redis Cluster Node 1
  redis-1:
    image: redis:6.2-alpine
    container_name: fp-redis-1
    command: redis-server --requirepass redis_password --maxmemory 2gb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis1_data:/data
    networks:
      - fp-network

  # Redis Cluster Node 2
  redis-2:
    image: redis:6.2-alpine
    container_name: fp-redis-2
    command: redis-server --requirepass redis_password --maxmemory 2gb --maxmemory-policy allkeys-lru
    ports:
      - "6380:6379"
    volumes:
      - redis2_data:/data
    networks:
      - fp-network

  # Redis Cluster Node 3
  redis-3:
    image: redis:6.2-alpine
    container_name: fp-redis-3
    command: redis-server --requirepass redis_password --maxmemory 2gb --maxmemory-policy allkeys-lru
    ports:
      - "6381:6379"
    volumes:
      - redis3_data:/data
    networks:
      - fp-network

  # File Upload Module
  fileupload:
    build:
      context: .
      dockerfile: fb-fileupload/Dockerfile
    container_name: fp-fileupload
    ports:
      - "8080:8080"
      - "9090:9090"  # Prometheus metrics
    environment:
      - INSTANCE_ID=fileupload-01
      - JAVA_OPTS=-Xms2g -Xmx4g -XX:+UseG1GC
    volumes:
      - ./properties:/app/properties
      - file_storage:/app/files
    depends_on:
      - mariadb-master
      - redis-1
    networks:
      - fp-network

  # Initial Stage
  initialstage:
    build:
      context: .
      dockerfile: fb-initialstage/Dockerfile
    container_name: fp-initialstage
    ports:
      - "8081:8080"
      - "9091:9090"
    environment:
      - INSTANCE_ID=initialstage-01
      - JAVA_OPTS=-Xms1g -Xmx2g -XX:+UseG1GC
    volumes:
      - ./properties:/app/properties
    depends_on:
      - mariadb-master
      - redis-1
    networks:
      - fp-network

  # Split Stage
  splitstage:
    build:
      context: .
      dockerfile: fb-splitstage/Dockerfile
    container_name: fp-splitstage
    ports:
      - "8082:8080"
      - "9092:9090"
    environment:
      - INSTANCE_ID=splitstage-01
      - JAVA_OPTS=-Xms4g -Xmx8g -XX:+UseG1GC
    volumes:
      - ./properties:/app/properties
      - file_storage:/app/files
    depends_on:
      - mariadb-master
      - redis-1
    networks:
      - fp-network
    deploy:
      replicas: 3  # Scale split stage for performance

  # Handover Stage
  handoverstage:
    build:
      context: .
      dockerfile: fb-handoverstage/Dockerfile
    container_name: fp-handoverstage
    ports:
      - "8083:8080"
      - "9093:9090"
    environment:
      - INSTANCE_ID=handoverstage-01
      - JAVA_OPTS=-Xms4g -Xmx8g -XX:+UseG1GC
    volumes:
      - ./properties:/app/properties
      - file_storage:/app/files
    depends_on:
      - mariadb-master
      - redis-1
    networks:
      - fp-network
    deploy:
      replicas: 5  # Scale handover stage for high throughput

  # Exclude Processor
  excludeprocessor:
    build:
      context: .
      dockerfile: fb-excludeprocessor/Dockerfile
    container_name: fp-excludeprocessor
    ports:
      - "8084:8080"
      - "9094:9090"
    environment:
      - INSTANCE_ID=excludeprocessor-01
      - JAVA_OPTS=-Xms2g -Xmx4g -XX:+UseG1GC
    volumes:
      - ./properties:/app/properties
      - file_storage:/app/files
    depends_on:
      - mariadb-master
      - redis-1
    networks:
      - fp-network

  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    container_name: fp-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - fileupload
    networks:
      - fp-network

  # Prometheus for Metrics
  prometheus:
    image: prom/prometheus:latest
    container_name: fp-prometheus
    ports:
      - "9095:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - fp-network

  # Grafana for Dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: fp-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana-dashboards:/etc/grafana/provisioning/dashboards
    depends_on:
      - prometheus
    networks:
      - fp-network

volumes:
  mariadb_data:
  redis1_data:
  redis2_data:
  redis3_data:
  file_storage:
  prometheus_data:
  grafana_data:

networks:
  fp-network:
    driver: bridge
```

#### Start Docker Deployment
```bash
# Build and start all services
docker-compose up -d

# Scale specific services
docker-compose up -d --scale splitstage=5 --scale handoverstage=10

# Check logs
docker-compose logs -f splitstage

# Stop all services
docker-compose down
```

### Option 3: Kubernetes Deployment

#### k8s-deployment.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: beacon-fileprocessor

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fp-config
  namespace: beacon-fileprocessor
data:
  config_params.properties: |
    # Configuration content here
    ...

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fp-splitstage
  namespace: beacon-fileprocessor
spec:
  replicas: 5
  selector:
    matchLabels:
      app: fp-splitstage
  template:
    metadata:
      labels:
        app: fp-splitstage
    spec:
      containers:
      - name: splitstage
        image: beacon-fileprocessor/splitstage:latest
        ports:
        - containerPort: 8080
        - containerPort: 9090
        env:
        - name: INSTANCE_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: JAVA_OPTS
          value: "-Xms4g -Xmx8g -XX:+UseG1GC"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        volumeMounts:
        - name: config
          mountPath: /app/properties
        - name: file-storage
          mountPath: /app/files
      volumes:
      - name: config
        configMap:
          name: fp-config
      - name: file-storage
        persistentVolumeClaim:
          claimName: fp-file-storage-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: fp-splitstage-svc
  namespace: beacon-fileprocessor
spec:
  selector:
    app: fp-splitstage
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fp-splitstage-hpa
  namespace: beacon-fileprocessor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fp-splitstage
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 📊 Monitoring & Operations

### Prometheus Metrics Configuration

#### prometheus.yml
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'fileupload'
    static_configs:
      - targets: ['fileupload:9090']
        labels:
          module: 'fileupload'
          
  - job_name: 'initialstage'
    static_configs:
      - targets: ['initialstage:9091']
        labels:
          module: 'initialstage'
          
  - job_name: 'splitstage'
    static_configs:
      - targets: ['splitstage:9092']
        labels:
          module: 'splitstage'
          
  - job_name: 'handoverstage'
    static_configs:
      - targets: ['handoverstage:9093']
        labels:
          module: 'handoverstage'
          
  - job_name: 'excludeprocessor'
    static_configs:
      - targets: ['excludeprocessor:9094']
        labels:
          module: 'excludeprocessor'
```

### Key Metrics to Monitor

```
┌─────────────────────────────────────────────────────────────────┐
│                     METRICS DASHBOARD                           │
└─────────────────────────────────────────────────────────────────┘

System Metrics:
───────────────
- CPU Usage (per module)
- Memory Usage (heap/non-heap)
- GC Pause Time
- Thread Count (active/total)
- Disk I/O
- Network I/O

Application Metrics:
────────────────────
- Campaigns Processed (per hour)
- Files Split (per hour)
- Messages Sent (per second)
- Queue Sizes (all queues)
- Consumer Lag
- Processing Time (per stage)
- Error Rate (per module)
- Retry Count

Database Metrics:
─────────────────
- Active Connections
- Query Execution Time
- Deadlock Count
- Row Count (campaign_file_splits)

Redis Metrics:
──────────────
- Connected Clients
- Queue Lengths
- Memory Usage
- Hit Rate
- Commands Per Second
```

### Health Check Endpoints

```bash
# File Upload Module
curl http://localhost:8080/health
# Response: {"status": "UP", "components": {...}}

# Initial Stage
curl http://localhost:8081/health

# Split Stage
curl http://localhost:8082/health

# Handover Stage
curl http://localhost:8083/health

# Exclude Processor
curl http://localhost:8084/health
```

---

## 🔧 Troubleshooting

### Common Issues and Solutions

#### Issue 1: FileSplitQ Consumer Not Processing
```bash
# Symptoms:
- Campaigns stuck in "inprogress" status
- FileSplitQ size increasing

# Diagnosis:
1. Check Redis queue size:
   redis-cli LLEN FileSplitQ

2. Check Split Stage logs:
   tail -f /app/logs/splitstage.log

3. Check for errors:
   grep ERROR /app/logs/splitstage.log

# Solutions:
1. Restart Split Stage consumers
2. Increase SPLIT_STAGE_THREADS
3. Check file system permissions
4. Verify database connectivity
```

#### Issue 2: Out of Memory Error
```bash
# Symptoms:
- java.lang.OutOfMemoryError in logs
- Module crashes

# Diagnosis:
1. Check heap usage:
   jstat -gc <pid>

2. Generate heap dump:
   jmap -dump:live,format=b,file=heap.hprof <pid>

# Solutions:
1. Increase -Xmx in JAVA_OPTS
2. Reduce SMS_SPLIT_LIMIT
3. Reduce thread pool sizes
4. Add more instances (scale out)
```

#### Issue 3: Database Connection Pool Exhausted
```bash
# Symptoms:
- "Could not get JDBC Connection" errors
- Slow processing

# Diagnosis:
1. Check active connections:
   SHOW PROCESSLIST;

2. Check pool configuration in jndi.properties

# Solutions:
1. Increase maxActive in connection pool
2. Fix connection leaks (ensure connections are closed)
3. Add database read replicas
4. Optimize slow queries
```

#### Issue 4: Redis Connection Timeout
```bash
# Symptoms:
- "JedisConnectionException: Could not get resource"
- Queue operations failing

# Diagnosis:
1. Check Redis connectivity:
   redis-cli -h redis-node-1 -p 6379 PING

2. Check Redis memory:
   redis-cli INFO memory

3. Check slow log:
   redis-cli SLOWLOG GET 10

# Solutions:
1. Increase maxTotal in Redis pool config
2. Add more Redis instances
3. Increase Redis maxmemory
4. Optimize Redis commands (use pipelines)
```

---

## ⚡ Performance Tuning

### JVM Tuning

```bash
# Recommended JVM Options for Production

# File Upload Module (4GB heap)
JAVA_OPTS="-Xms2g -Xmx4g 
           -XX:+UseG1GC 
           -XX:MaxGCPauseMillis=200 
           -XX:+ParallelRefProcEnabled 
           -XX:+UnlockExperimentalVMOptions 
           -XX:+AggressiveOpts 
           -XX:+UseLargePages 
           -XX:+AlwaysPreTouch 
           -Xlog:gc*:file=/app/logs/gc.log:time,uptime:filecount=5,filesize=100M"

# Split Stage (8GB heap - high file processing)
JAVA_OPTS="-Xms4g -Xmx8g 
           -XX:+UseG1GC 
           -XX:MaxGCPauseMillis=200 
           -XX:G1HeapRegionSize=16m 
           -XX:+ParallelRefProcEnabled 
           -XX:+UnlockExperimentalVMOptions 
           -XX:+AggressiveOpts"

# Handover Stage (8GB heap - high throughput)
JAVA_OPTS="-Xms4g -Xmx8g 
           -XX:+UseG1GC 
           -XX:MaxGCPauseMillis=100 
           -XX:G1HeapRegionSize=16m 
           -XX:ConcGCThreads=4 
           -XX:ParallelGCThreads=8"
```

### Thread Pool Configuration

```properties
# Optimize based on available CPU cores

# Split Stage (CPU-intensive)
SPLIT_STAGE_THREADS=<number_of_cores * 2>

# Handover Stage (I/O-intensive)
HANDOVER_STAGE_THREADS=<number_of_cores * 3>

# Initial Stage (low load)
INITIAL_STAGE_THREADS=<number_of_cores>
```

### Database Optimization

```sql
-- Add indexes for frequently queried columns
CREATE INDEX idx_cm_status ON campaign_master(status, cli_id);
CREATE INDEX idx_cf_status ON campaign_files(status, c_id);
CREATE INDEX idx_cfs_status ON campaign_file_splits(status, cm_id, cf_id);
CREATE INDEX idx_cm_scheduled ON campaign_master(scheduled_ts, status);

-- Optimize table structure
ALTER TABLE campaign_file_splits 
  ENGINE=InnoDB 
  ROW_FORMAT=COMPRESSED;

-- Enable query cache
SET GLOBAL query_cache_size = 268435456;  -- 256MB
SET GLOBAL query_cache_type = ON;
```

### Redis Optimization

```bash
# redis.conf optimizations

# Increase max clients
maxclients 10000

# Enable lazy free
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes

# Optimize for throughput
tcp-backlog 511
timeout 0
tcp-keepalive 300

# Persistence (adjust based on needs)
save 900 1
save 300 10
save 60 10000
```

---

## 📈 Capacity Planning

### Estimated Throughput

```
┌─────────────────────────────────────────────────────────────────┐
│               CAPACITY & THROUGHPUT ESTIMATES                   │
└─────────────────────────────────────────────────────────────────┘

Configuration: 5 Split Stage + 10 Handover Stage Instances
Hardware: 16 Core, 32GB RAM per instance

Stage                    | Throughput        | Bottleneck
─────────────────────────┼──────────────────┼─────────────────
File Upload              | 100 files/min     | Disk I/O
Initial Stage (Polling)  | 1000 campaigns/min| Database
Split Stage              | 500K msgs/min     | CPU + File I/O
Exclude Processing       | 200K msgs/min     | Database
Handover Stage           | 1M msgs/min       | Redis + Network
─────────────────────────┴──────────────────┴─────────────────

Overall System Capacity:
- Campaigns per hour: 60,000
- Messages per hour: 30 million
- Peak messages per second: 15,000
```

---

## 🔒 Security Best Practices

1. **Network Security**
   - Use VPN/Private network for internal communication
   - Enable SSL/TLS for Redis connections
   - Firewall rules to restrict access

2. **Database Security**
   - Use encrypted passwords
   - Implement role-based access control
   - Regular security audits

3. **File Security**
   - Validate file uploads (size, type, content)
   - Scan for viruses
   - Isolate user files

4. **API Security**
   - Implement authentication & authorization
   - Rate limiting
   - Input validation

---

**End of Deployment & Configuration Guide**
