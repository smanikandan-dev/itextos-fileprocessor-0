# Beacon File Processor - Module Relationships & Dependencies

## Table of Contents
1. [Module Dependency Graph](#module-dependency-graph)
2. [Component Layer Architecture](#component-layer-architecture)
3. [Redis Queue Topology](#redis-queue-topology)
4. [Database Entity Relationships](#database-entity-relationships)
5. [Inter-Module Communication](#inter-module-communication)
6. [Technology Integration Map](#technology-integration-map)

---

## Module Dependency Graph

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     MODULE DEPENDENCY GRAPH                                │
│                     (Arrows show "depends on")                             │
└───────────────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────┐
                        │   fb-utils          │
                        │   (Core Library)    │
                        │                     │
                        │ - Database access   │
                        │ - Redis connections │
                        │ - DTOs              │
                        │ - Utilities         │
                        │ - Configuration     │
                        └──────────┬──────────┘
                                   │
                                   │ (Shared by all)
                ┌──────────────────┼──────────────────┐
                │                  │                  │
                │                  │                  │
      ┌─────────▼────────┐ ┌──────▼──────┐  ┌───────▼────────┐
      │  fb-fileparser   │ │  fb-logger  │  │  beaconlib     │
      │  (File Parsing)  │ │  (Logging)  │  │  (Standalone)  │
      │                  │ │             │  │                │
      │ - CSV parser     │ │ - Formatters│  │ - JAR with     │
      │ - XLS parser     │ │ - Log mgmt  │  │   dependencies │
      │ - XLSX parser    │ │             │  │                │
      │ - ZIP handler    │ └─────────────┘  └────────────────┘
      └─────────┬────────┘
                │
                │ (Used by file processors)
                │
    ┌───────────┴───────────┬──────────────┬──────────────┐
    │                       │              │              │
┌───▼──────────────┐ ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────────────┐
│ fb-fileupload    │ │ fb-split    │ │fb-groups  │ │ fb-dltfile        │
│                  │ │ stage       │ │processor  │ │ processor         │
│ - Upload servlet │ │             │ │           │ │                   │
│ - File storage   │ │ - Consumer  │ │ - Group   │ │ - DLT upload      │
│ - Validation     │ │ - Splitter  │ │   polling │ │ - Template mgmt   │
│                  │ │             │ │ - File gen│ │                   │
│ Depends on:      │ │ Depends on: │ │           │ │ Depends on:       │
│ - fb-utils       │ │ - fb-utils  │ │Depends on:│ │ - fb-utils        │
└──────────────────┘ │ - fileparser│ │- fb-utils │ └───────────────────┘
                     └──────┬──────┘ │-fileparser│
                            │        └───────────┘
                            │
                    ┌───────▼───────┐
                    │ fb-handover   │
                    │ stage         │
                    │               │
                    │ - Consumer    │
                    │ - Processors  │
                    │ - Kafka send  │
                    │               │
                    │ Depends on:   │
                    │ - fb-utils    │
                    └───────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                    SUPPORTING MODULES                                     │
├──────────────────────────────────────────────────────────────────────────┤

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ fb-initialstage  │  │ fb-campaign      │  │ fb-exclude       │
│                  │  │ finisher         │  │ processor        │
│ - DB polling     │  │                  │  │                  │
│ - Queue push     │  │ - Completion     │  │ - Exclusion list │
│                  │  │ - Stats          │  │ - Validation     │
│ Depends on:      │  │ - Cleanup        │  │                  │
│ - fb-utils       │  │                  │  │ Depends on:      │
└──────────────────┘  │ Depends on:      │  │ - fb-utils       │
                      │ - fb-utils       │  └──────────────────┘
                      └──────────────────┘

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ fb-schedule      │  │ fb-download      │  │ fb-cronjobs      │
│ processor        │  │ handler          │  │                  │
│                  │  │                  │  │ - Cleanup        │
│ - Scheduler      │  │ - File download  │  │ - Maintenance    │
│ - Cron jobs      │  │ - Report gen     │  │ - Log rotation   │
│                  │  │                  │  │                  │
│ Depends on:      │  │ Depends on:      │  │ Depends on:      │
│ - fb-utils       │  │ - fb-utils       │  │ - fb-utils       │
└──────────────────┘  └──────────────────┘  └──────────────────┘

┌──────────────────┐
│ fb-inmemory      │
│ refresh          │
│                  │
│ - Cache refresh  │
│ - Config reload  │
│                  │
│ Depends on:      │
│ - fb-utils       │
└──────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                    BUILD & DEPLOYMENT MODULES                             │
├──────────────────────────────────────────────────────────────────────────┤

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ properties       │  │ thirtypartyjar   │  │ docker-file      │
│                  │  │                  │  │ processor        │
│ - Config files   │  │ - 3rd party deps │  │                  │
│ - Properties     │  │                  │  │ - Dockerfiles    │
│ - Profiles       │  │                  │  │ - Compose files  │
│                  │  │                  │  │ - Server configs │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

**Key Dependency Points:**
- **fb-utils**: The foundational module, used by ALL other modules
- **fb-fileparser**: Used by modules that need file parsing (split, groups, upload, dlt)
- **fb-logger**: Used by all for consistent logging
- **beaconlib**: Standalone deployable library combining utils and parser

---

## Component Layer Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    LAYERED ARCHITECTURE VIEW                               │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION LAYER                                 │
│  (HTTP Endpoints - Servlets)                                              │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  FilesSaver  │  InitializePoller  │  SplitStageServlet  │ Others...      │
│  (/save)     │  (/init)           │  (/init)            │                │
│                                                                            │
└────────────────────────────┬──────────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────────┐
│                         SERVICE LAYER                                      │
│  (Business Logic - Pollers, Consumers, Handlers)                          │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │   POLLERS       │  │   CONSUMERS     │  │   HANDLERS      │          │
│  │                 │  │                 │  │                 │          │
│  │ - CampaignMaster│  │ - FileSplitQ    │  │ - MasterFile    │          │
│  │   Poller        │  │   Consumer      │  │   SplitHandler  │          │
│  │ - GroupsMaster  │  │ - SplitFile     │  │ - ProcessOTM/   │          │
│  │   Poller        │  │   Consumer      │  │   MTM/TEM       │          │
│  │ - DltTemplate   │  │ - GroupsQ       │  │ - KafkaHandover │          │
│  │   Poller        │  │   Consumer      │  │                 │          │
│  │                 │  │                 │  │                 │          │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘          │
│                                                                            │
└────────────────────────────┬──────────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────────┐
│                         DATA ACCESS LAYER                                  │
│  (DAOs, File Parsers)                                                     │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │   DAO LAYER     │  │  FILE PARSERS   │  │ QUEUE ACCESS    │          │
│  │                 │  │                 │  │                 │          │
│  │ - GenericDAO    │  │ - CSVParser     │  │ - RedisQueue    │          │
│  │ - CampaignMaster│  │ - XLSParser     │  │   Sender        │          │
│  │   DAO           │  │ - XLSXParser    │  │ - FileSender    │          │
│  │ - GroupsMaster  │  │ - ZipHandler    │  │                 │          │
│  │   DAO           │  │                 │  │                 │          │
│  │                 │  │                 │  │                 │          │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘          │
│                                                                            │
└────────────────────────────┬──────────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────────┐
│                         INFRASTRUCTURE LAYER                               │
│  (Connections, Utilities, Singletons)                                     │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐       │
│  │   SINGLETONS     │  │   UTILITIES      │  │   CONNECTIONS    │       │
│  │                  │  │                  │  │                  │       │
│  │ - ConfigParamsTon│  │ - Utility        │  │ - ConnectionFactory│     │
│  │ - RedisConnection│  │ - JsonUtility    │  │   ForCMDB        │       │
│  │   Factory        │  │ - MobileValidator│  │ - RedisConnection│       │
│  │ - GlobalProperties│ │ - EmailValidator │  │   Factory        │       │
│  │   Ton            │  │ - HeartBeat      │  │ - Kafka          │       │
│  │                  │  │   Monitoring     │  │   Producers      │       │
│  │                  │  │                  │  │                  │       │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘       │
│                                                                            │
└────────────────────────────┬──────────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────────┐
│                         EXTERNAL SYSTEMS                                   │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │   MariaDB    │  │    Redis     │  │    Kafka     │  │ File System │ │
│  │              │  │              │  │              │  │             │ │
│  │ - Campaigns  │  │ - Queues     │  │ - Topics     │  │ - Uploads   │ │
│  │ - Files      │  │ - Cache      │  │ - Messages   │  │ - Splits    │ │
│  │ - Templates  │  │ - Counters   │  │              │  │ - Logs      │ │
│  │              │  │              │  │              │  │             │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Redis Queue Topology

```
┌───────────────────────────────────────────────────────────────────────────┐
│                       REDIS QUEUE TOPOLOGY                                 │
│                       (Queue Structure & Flow)                             │
└───────────────────────────────────────────────────────────────────────────┘

Redis Instance 1 (Primary)
├─ FileSplitQ                         [Global queue for file splitting]
│  └─ Structure: List
│     └─ Data: Campaign metadata JSON
│
├─ DeliveryQ_Manager                  [Round-robin delivery queue manager]
│  └─ Structure: List
│     └─ Data: Campaign IDs
│
├─ DeliveryQ_{cm_id_1}                [Per-campaign delivery queues]
│  └─ Structure: List
│     └─ Data: Split file metadata JSON
│
├─ {product}:processing:{cm_id}       [Temporary processing queues]
│  └─ Structure: List
│     └─ Data: Currently processing splits
│
├─ SQLUpdateQ                         [Database update queue]
│  └─ Structure: List
│     └─ Data: SQL update statements
│
├─ retry:{c_f_s_id}                   [Retry counters]
│  └─ Structure: String/Integer
│     └─ Data: Retry count
│
├─ tracking:{type}:{username}         [File tracking for cleanup]
│  └─ Structure: List
│     └─ Data: File paths
│
├─ HB:{module}:{consumer}:{instance}  [Heartbeat keys]
│  └─ Structure: String with TTL
│     └─ Data: Timestamp
│
└─ config:*                           [Configuration cache]
   └─ Structure: Hash/String
      └─ Data: Key-value config pairs

Redis Instance 2 (Secondary/Backup)
├─ [Same structure as Instance 1]
└─ Used for failover and load balancing

Redis Queue Operations by Module:

┌─────────────────────────────────────────────────────────────────────────┐
│ fb-initialstage:                                                        │
│   LPUSH → FileSplitQ                                                    │
│                                                                          │
│ fb-splitstage:                                                          │
│   RPOP  ← FileSplitQ                                                    │
│   LPUSH → DeliveryQ_{cm_id}                                             │
│   LPUSH → DeliveryQ_Manager                                             │
│                                                                          │
│ fb-handoverstage:                                                       │
│   RPOPLPUSH ← DeliveryQ_Manager → DeliveryQ_Manager                     │
│   RPOPLPUSH ← DeliveryQ_{cm_id} → {product}:processing:{cm_id}          │
│   LREM  → {product}:processing:{cm_id} (after success)                  │
│                                                                          │
│ fb-campaignfinisher:                                                    │
│   DEL   → DeliveryQ_{cm_id}                                             │
│   DEL   → {product}:processing:{cm_id}                                  │
│   DEL   → retry:*                                                       │
│                                                                          │
│ All modules:                                                            │
│   SET   → HB:{module}:{consumer}:{instance}                             │
│   GET   → config:*                                                      │
└─────────────────────────────────────────────────────────────────────────┘

Queue Flow Diagram:

         Initial Stage
               │
               │ LPUSH
               ▼
         ┌──────────┐
         │FileSplitQ│
         └────┬─────┘
              │ RPOP
              ▼
         Split Stage
              │
              │ LPUSH
              ▼
    ┌──────────────────────┐
    │ DeliveryQ_{cm_id}    │
    └──────────┬───────────┘
               │ RPOPLPUSH
               ▼
    ┌──────────────────────────┐
    │{product}:processing:     │
    │{cm_id}                   │
    └──────────┬───────────────┘
               │
               │ Process
               │
               ▼
    ┌──────────────────────────┐
    │  Handover Stage          │
    │  (Send to Kafka)         │
    └──────────┬───────────────┘
               │
               │ LREM (success)
               ▼
           Completed
               │
               ▼
    ┌──────────────────────────┐
    │ Campaign Finisher        │
    │ (Cleanup queues)         │
    └──────────────────────────┘
```

---

## Database Entity Relationships

```
┌───────────────────────────────────────────────────────────────────────────┐
│                   DATABASE ENTITY RELATIONSHIP DIAGRAM                     │
└───────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐
│   campaign_master       │
│─────────────────────────│
│ PK: cm_id               │
│ FK: cli_id → clients    │
│─────────────────────────│
│ username                │
│ c_name                  │
│ c_type (OTM/MTM/TEM)    │
│ c_lang_type             │
│ sender_id               │
│ message                 │
│ total                   │
│ status                  │
│ dlt_entity_id           │
│ dlt_template_id         │
│ exclude_group_ids       │
│ scheduled_time          │
│ created_ts              │
│ completed_ts            │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ▼
┌─────────────────────────┐
│   campaign_files        │
│─────────────────────────│
│ PK: c_f_id              │
│ FK: cm_id               │
│─────────────────────────│
│ filename                │
│ fileloc                 │
│ total                   │
│ status                  │
│ retry_count             │
│ created_ts              │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ▼
┌─────────────────────────┐
│ campaign_file_splits    │
│─────────────────────────│
│ PK: c_f_s_id            │
│ FK: cm_id               │
│ FK: c_f_id              │
│─────────────────────────│
│ filename                │
│ fileloc                 │
│ total                   │
│ status                  │
│ redis_id                │
│ sent_count              │
│ failed_count            │
│ retry_count             │
│ created_ts              │
│ completed_ts            │
└─────────────────────────┘


┌─────────────────────────┐
│   group_master          │
│─────────────────────────│
│ PK: g_id                │
│ FK: cli_id → clients    │
│─────────────────────────│
│ g_name                  │
│ total                   │
│ status                  │
│ created_ts              │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ▼
┌─────────────────────────┐
│   group_files           │
│─────────────────────────│
│ PK: g_f_id              │
│ FK: g_id                │
│─────────────────────────│
│ filename                │
│ fileloc                 │
│ total                   │
│ status                  │
└─────────────────────────┘


┌─────────────────────────┐
│ dlt_template_master     │
│─────────────────────────│
│ PK: dlt_temp_id         │
│ FK: cli_id → clients    │
│─────────────────────────│
│ dlt_entity_id           │
│ dlt_template_id         │
│ template_content        │
│ template_type           │
│ status                  │
└─────────────────────────┘


┌─────────────────────────┐
│ exclude_group_master    │
│─────────────────────────│
│ PK: e_g_id              │
│ FK: cli_id → clients    │
│─────────────────────────│
│ e_g_name                │
│ total                   │
│ status                  │
└────────┬────────────────┘
         │ 1
         │
         │ N
         ▼
┌─────────────────────────┐
│ exclude_group_numbers   │
│─────────────────────────│
│ PK: e_g_n_id            │
│ FK: e_g_id              │
│─────────────────────────│
│ mobile_number           │
└─────────────────────────┘


┌─────────────────────────┐
│   clients               │
│─────────────────────────│
│ PK: cli_id              │
│─────────────────────────│
│ username                │
│ company_name            │
│ status                  │
│ created_ts              │
└─────────────────────────┘


┌─────────────────────────┐
│   config_params         │
│─────────────────────────│
│ PK: key                 │
│─────────────────────────│
│ value                   │
│ description             │
└─────────────────────────┘

Relationship Summary:
═══════════════════════

1. One Client has Many Campaigns (1:N)
2. One Campaign has Many Files (1:N)
3. One Campaign File has Many Split Files (1:N)
4. One Client has Many Groups (1:N)
5. One Group has Many Group Files (1:N)
6. One Client has Many DLT Templates (1:N)
7. One Client has Many Exclusion Groups (1:N)
8. One Exclusion Group has Many Numbers (1:N)
```

---

## Inter-Module Communication

```
┌───────────────────────────────────────────────────────────────────────────┐
│                   INTER-MODULE COMMUNICATION PATTERNS                      │
└───────────────────────────────────────────────────────────────────────────┘

Communication Type 1: Queue-Based (Asynchronous)
═══════════════════════════════════════════════

    Producer Module          Redis Queue         Consumer Module
          │                       │                      │
          │  1. Push message      │                      │
          ├──────────────────────►│                      │
          │  (LPUSH/RPUSH)        │                      │
          │                       │  2. Poll for message │
          │                       │◄─────────────────────┤
          │                       │  (RPOP/BLPOP)        │
          │                       │                      │
          │                       │  3. Return message   │
          │                       ├─────────────────────►│
          │                       │                      │
          │                       │  4. Process          │
          │                       │                      │
          │                       │  5. Remove from Q    │
          │                       │◄─────────────────────┤
          │                       │  (implicit)          │

Examples:
  - InitialStage → FileSplitQ → SplitStage
  - SplitStage → DeliveryQ → HandoverStage


Communication Type 2: Database-Based (Polling)
═══════════════════════════════════════════════

    Poller Module              Database           Handler Module
          │                       │                      │
          │  1. SELECT with filter│                      │
          ├──────────────────────►│                      │
          │  (status='ready')     │                      │
          │                       │                      │
          │  2. Return records    │                      │
          │◄──────────────────────┤                      │
          │                       │                      │
          │  3. Process records   │                      │
          ├───────────────────────┼─────────────────────►│
          │                       │                      │
          │  4. Update status     │                      │
          ├──────────────────────►│                      │
          │  (status='processing')│                      │

Examples:
  - CampaignMasterPoller → campaign_master
  - GroupsMasterPoller → group_master
  - CampaignFinisher → campaign_file_splits


Communication Type 3: Kafka-Based (Streaming)
══════════════════════════════════════════════

    HandoverStage            Kafka Cluster       External System
          │                       │                      │
          │  1. Send message      │                      │
          ├──────────────────────►│                      │
          │  (producer.send)      │                      │
          │                       │                      │
          │                       │  2. Store in topic   │
          │                       │  (partitioned)       │
          │                       │                      │
          │                       │  3. Consumer poll    │
          │                       │◄─────────────────────┤
          │                       │                      │
          │                       │  4. Deliver messages │
          │                       ├─────────────────────►│

Example:
  - HandoverStage → Kafka → Delivery Engine


Communication Type 4: Direct HTTP (Synchronous)
════════════════════════════════════════════════

    External Client          Servlet/API          Backend Logic
          │                       │                      │
          │  1. HTTP Request      │                      │
          ├──────────────────────►│                      │
          │  (POST with data)     │                      │
          │                       │  2. Validate & Process│
          │                       ├─────────────────────►│
          │                       │                      │
          │                       │  3. Response         │
          │                       │◄─────────────────────┤
          │                       │                      │
          │  4. HTTP Response     │                      │
          │◄──────────────────────┤                      │
          │  (JSON)               │                      │

Example:
  - Client → FilesSaver → File Storage


Communication Matrix:
═════════════════════

┌──────────────────┬───────────┬─────────┬────────┬──────────┐
│ From/To          │ Database  │ Redis   │ Kafka  │ HTTP     │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ External Client  │     -     │    -    │   -    │  Upload  │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ InitialStage     │  Poll     │  Push   │   -    │    -     │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ SplitStage       │  Update   │ Pop/Push│   -    │    -     │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ HandoverStage    │  Update   │  Pop    │  Send  │    -     │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ CampaignFinisher │  Poll/Upd │ Cleanup │   -    │    -     │
├──────────────────┼───────────┼─────────┼────────┼──────────┤
│ GroupsProcessor  │  Poll/Upd │ Pop/Push│   -    │    -     │
└──────────────────┴───────────┴─────────┴────────┴──────────┘
```

---

## Technology Integration Map

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    TECHNOLOGY INTEGRATION MAP                              │
└───────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │   Java Application      │
                    │   (WildFly/JBoss)       │
                    └───────────┬─────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   JDBC          │  │   Jedis         │  │ Kafka Client    │
│   (Database)    │  │   (Redis)       │  │                 │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                     │
         ▼                    ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   MariaDB       │  │   Redis         │  │   Kafka         │
│                 │  │                 │  │                 │
│ - JDBC Driver   │  │ - Jedis 3.6.0   │  │ - Clients 2.8.0 │
│ - Connection    │  │ - Connection    │  │ - Producer API  │
│   Pool (DBCP2)  │  │   Pooling       │  │ - Topics        │
│                 │  │ - Pub/Sub       │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘

File Processing Stack:
══════════════════════

┌─────────────────────────────────────────────────────┐
│           File Format Detection                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐│
│  │   CSV   │  │   XLS   │  │  XLSX   │  │  ZIP   ││
│  └────┬────┘  └────┬────┘  └────┬────┘  └───┬────┘│
│       │            │            │            │     │
│       ▼            ▼            ▼            ▼     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐│
│  │Commons  │  │ Apache  │  │ Apache  │  │Commons ││
│  │  CSV    │  │  POI    │  │  POI    │  │  IO    ││
│  │  1.8    │  │  (old)  │  │  (new)  │  │        ││
│  └─────────┘  └─────────┘  └─────────┘  └────────┘│
└─────────────────────────────────────────────────────┘

Logging Stack:
══════════════

┌─────────────────────────────────────────────────────┐
│         Application Logging                         │
├─────────────────────────────────────────────────────┤
│  fb-logger (Custom Formatters)                      │
│            │                                         │
│            ▼                                         │
│  ┌──────────────────┐                               │
│  │   Log4j2 2.17.0  │                               │
│  └────────┬─────────┘                               │
│           │                                          │
│           ▼                                          │
│  ┌──────────────────────────────────┐               │
│  │   File Appenders                 │               │
│  │   - application.log              │               │
│  │   - consumer.log                 │               │
│  │   - producer.log                 │               │
│  │   - error.log                    │               │
│  │   - module-specific logs         │               │
│  └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────┘

Configuration Management:
═════════════════════════

┌─────────────────────────────────────────────────────┐
│         Configuration Sources                       │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ .properties  │  │   Database   │  │  Redis   │ │
│  │    files     │  │config_params │  │  cache   │ │
│  └──────┬───────┘  └──────┬───────┘  └────┬─────┘ │
│         │                 │                │       │
│         └─────────────────┴────────────────┘       │
│                           │                        │
│                           ▼                        │
│         ┌─────────────────────────────┐            │
│         │ Apache Commons Config 1.10  │            │
│         └─────────────────────────────┘            │
│                           │                        │
│                           ▼                        │
│         ┌─────────────────────────────┐            │
│         │  ConfigParamsTon (Singleton)│            │
│         │  - In-memory cache          │            │
│         │  - Auto-refresh             │            │
│         └─────────────────────────────┘            │
└─────────────────────────────────────────────────────┘

Monitoring & Observability:
═══════════════════════════

┌─────────────────────────────────────────────────────┐
│         Monitoring Stack                            │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────────────┐                           │
│  │  HeartBeat Module    │                           │
│  │  (Custom)            │                           │
│  └──────────┬───────────┘                           │
│             │                                        │
│             ├───────► Redis (HB keys with TTL)      │
│             │                                        │
│             └───────► Prometheus Exporter           │
│                      (Port 1075)                    │
│                            │                        │
│                            ▼                        │
│                   ┌─────────────────┐               │
│                   │   Prometheus    │               │
│                   │   Scraper       │               │
│                   └─────────────────┘               │
│                            │                        │
│                            ▼                        │
│                   ┌─────────────────┐               │
│                   │   Grafana       │               │
│                   │   Dashboards    │               │
│                   └─────────────────┘               │
└─────────────────────────────────────────────────────┘

Utility Libraries Integration:
══════════════════════════════

┌──────────────────────────────────────────────────────┐
│  Apache Commons Suite                                │
│  ├─ commons-lang (String utilities)                  │
│  ├─ commons-io (File operations)                     │
│  ├─ commons-codec (Encoding/decoding)                │
│  ├─ commons-csv (CSV parsing)                        │
│  ├─ commons-validator (Validation)                   │
│  ├─ commons-net (Network operations)                 │
│  └─ commons-dbcp2 (Connection pooling)               │
│                                                       │
│  JSON Processing                                     │
│  ├─ Jackson 2.12.1 (JSON parsing)                    │
│  └─ Gson 2.8.8 (JSON serialization)                  │
│                                                       │
│  Scheduling                                          │
│  └─ Quartz 2.3.2 (Job scheduling)                    │
│                                                       │
│  HTTP Client                                         │
│  └─ Apache HttpClient 4.5.13                         │
└──────────────────────────────────────────────────────┘
```

---

## Summary

This document provides a comprehensive view of the relationships and dependencies within the Beacon File Processor system:

### Key Architectural Patterns

1. **Dependency Hierarchy**: fb-utils as the foundational layer, with all other modules depending on it
2. **Layered Architecture**: Clear separation between presentation, service, data access, and infrastructure layers
3. **Queue-Based Communication**: Asynchronous processing using Redis queues for loose coupling
4. **Singleton Pattern**: Centralized resource management for connections and configurations
5. **Factory Pattern**: File parser selection based on format detection

### Integration Points

- **MariaDB**: Primary data store via JDBC with connection pooling
- **Redis**: Queue management, caching, and coordination via Jedis
- **Kafka**: Message delivery via Kafka clients
- **File System**: Direct file I/O for campaign and split files
- **HTTP**: RESTful endpoints for file upload and management

### Scalability Factors

- Multiple Redis instances for load distribution
- Connection pooling for database and Redis
- Queue-based decoupling enables horizontal scaling
- Modular design allows independent scaling of components
- File splitting enables parallel processing

This architecture supports the high-volume, mission-critical nature of SMS campaign processing while maintaining reliability, scalability, and maintainability.

