---
title: "Preparations for Building Scalable Cloud DICOM-WEB Services"
date: 2025-11-19T15:21:56+08:00
keywords: "DICOM-WEB, medical imaging, healthcare cloud, DICOM storage, cloud DICOM platform, DICOM services"
description: "Comprehensive guide to preparing infrastructure for scalable cloud DICOM-WEB services, including database, message queue, cache, programming language, and service framework configurations."
draft: false
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "DICOM storage", "cloud DICOM platform"]
categories: ["Medical Imaging", "DICOM Development", "Healthcare Technology"]
slug: "preparations-building-scalable-cloud-dicom-web-services"
---

# Preparations for Building Scalable Cloud DICOM-WEB Services

This article introduces the architectural design of a DICOM medical imaging system developed using Rust, which employs a modern technology stack including PostgreSQL as the primary index database, Apache Doris for log storage, RedPanda as the message queue, and Redis for caching. The system design supports both standalone operation and distributed scaling, fully leveraging the safety and performance advantages of the Rust programming language.

## Overview of DICOM Medical Imaging System Architecture and Runtime Environment

### Core Components

- **PostgreSQL**: Primary index database storing core metadata such as patients, studies, and series
- **Apache Doris**: Log storage for recording DICOM CStoreSCP service and WADO-RS service access logs
- **RedPanda**: Message queue for handling asynchronous communication between systems
- **Redis**: Cache layer to improve system response speed
- **Rust**: Programming language utilizing the dicom-rs library for DICOM data processing

### Service Modules

- **wado-storescp**: DICOM CStoreSCP service that receives DICOM files and writes them to disk
- **wado-consumer**: Consumes storage events from message queues, extracts metadata, and writes to databases
- **wado-server**: DICOM WEB WADO-RS API interface implementation
- **wado-webworker**: Periodically generates JSON-formatted metadata for accelerated access

## Database Design

### PostgreSQL Primary Index Database

PostgreSQL serves as the primary index database for storing core metadata, including:

- **dicom_state_meta**: Stores patient, study, and series-level metadata
- **dicom_json_meta**: Records sequence information requiring JSON-formatted metadata generation

### Apache Doris Log Storage

Apache Doris is used to store various service logs:

- **DicomStateMeta**: DICOM state metadata
- **DicomImageMeta**: DICOM image metadata
- **WadoAccessLog**: WADO access logs

## Docker Compose Scripts

We assume your database server IP address is 192.168.1.14

Operating System: Ubuntu 22.04.5 LTS

### PostgreSQL Setup

```yaml
version: '3'

services:
  pgdb:
    image: ankane/pgvector:latest
    container_name: pgappx
    restart: always
    environment:
      POSTGRES_PASSWORD: "xDicom123"
      POSTGRES_USER: "root"
      PGTZ: "Asia/Shanghai"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
      - ./pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf
    ports:
      - "5432:5432"
```

### Redis, RabbitMQ, and PgAdmin Setup
```yaml
 version: '3'

services:
  redis:
    image: redis
    restart: always
    volumes:
      - ./redata:/data
    ports:
      - "6379:6379"
  rabbitmq:
    image: rabbitmq:management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: xDicom123
    volumes:
      - ./rabbitmq/data:/var/lib/rabbitmq

  pgadmin:
    user: root
    container_name: pgadmin4_container
    image: dpage/pgadmin4:8.4
    restart: always

    environment:
      PGADMIN_DEFAULT_EMAIL: oscar.xdev@outlook.com
      PGADMIN_DEFAULT_PASSWORD: xDicom123
      PGADMIN_LISTEN_ADDRESS: 0.0.0.0
      PGADMIN_SERVER_JSON_FILE: /pgadmin4/servers.json
      TZ: Asia/shanghai
    ports:
      - "8080:80"
      - "9443:443"
    volumes:
      - ./pgadmin:/var/lib/pgadmin
```


### Apache Doris and RedPanda Setup
Refer to the official documentation for Doris and RedPanda.


  
Starting Doris

```bash
./Doris3.X/3.1.0/fe/bin/start_fe.sh --daemon
./Doris3.X/3.1.0/be/bin/start_be.sh --daemon
```
  
### RedPanda Message Queue Operations
- Creating Topics:
```bash 
rpk topic create dicom_image_queue  --partitions 1 --replicas 1
rpk topic create dicom_state_queue  --partitions 1 --replicas 1
rpk topic create log_queue          --partitions 1 --replicas 1
rpk topic create storage_queue      --partitions 1 --replicas 1
```

- Clearing Topics:
```bash
rpk topic trim-prefix dicom_image_queue  -p 0 --offset end --no-confirm
rpk topic trim-prefix dicom_state_queue  -p 0 --offset end --no-confirm
rpk topic trim-prefix log_queue          -p 0 --offset end --no-confirm
rpk topic trim-prefix storage_queue      -p 0 --offset end --no-confirm
```

### Database Initialization Scripts

- Primary Database (PostgreSQL) Creation Script
```pgsql
create table dicom_state_meta
(
    tenant_id                varchar(64) not null,
    patient_id               varchar(64) not null,
    study_uid                varchar(64) not null,
    series_uid               varchar(64) not null,
    study_uid_hash           varchar(20) not null,
    series_uid_hash          varchar(20) not null,
    patient_name             varchar(64),
    patient_sex              varchar(1),
    patient_birth_date       date,
    patient_birth_time       time,
    patient_age              varchar(16),
    patient_size             double precision,
    patient_weight           double precision,
    pregnancy_status         integer,
    study_date               date        not null,
    study_date_origin        varchar(8)  not null,
    study_time               time,
    accession_number         varchar(16) not null,
    study_id                 varchar(16),
    study_description        varchar(64),
    modality                 varchar(16),
    series_number            integer,
    series_date              date,
    series_time              time,
    series_description       varchar(256),
    body_part_examined       varchar(64),
    protocol_name            varchar(64),
    series_related_instances integer,
    created_time             timestamp,
    updated_time             timestamp,
    primary key (tenant_id, study_uid, series_uid)
);


create unique index index_state_unique
    on dicom_state_meta (tenant_id, study_uid, series_uid, accession_number);

drop table if exists dicom_json_meta;
create table  dicom_json_meta
(
    tenant_id                varchar(64) not null,
    study_uid                varchar(64) not null,
    series_uid               varchar(64) not null,
    study_uid_hash           varchar(20) not null,
    series_uid_hash          varchar(20) not null,
    study_date_origin        varchar(8)  not null,
    flag_time                timestamp   not null,
    created_time             timestamp   not null default current_timestamp(6),
    json_status              int         not null default 0,
    retry_times              int         not null default 0
);

ALTER TABLE dicom_json_meta
ADD CONSTRAINT PK_dicom_json_meta PRIMARY KEY (tenant_id, study_uid, series_uid);
```

-  Apache Doris Database Tables and Stream Load Configuration
```sql
 drop  table IF   EXISTS  dicom_object_meta;
create table IF NOT EXISTS  dicom_object_meta
(
    tenant_id           varchar(64)   not null comment 'Tenant ID',
    patient_id          varchar(64)   not null comment 'Patient ID',
    study_uid           varchar(64)   null,
    series_uid          varchar(64)   null,
    sop_uid             varchar(64)   null,
    file_size           bigint        null,
    file_path           varchar(512) null,
    transfer_syntax_uid varchar(64)   null,
    number_of_frames    int           null,
    created_time        datetime      null,
    series_uid_hash     VARCHAR(20)   null,
    study_uid_hash      VARCHAR(20)   null,
    accession_number    varchar(64)   null,
    target_ts           varchar(64)   null,
    study_date          date          null,
    transfer_status     varchar(64)   null,
    source_ip           varchar(24)   null,
    source_ae           varchar(64)   null,
    trace_id            varchar(36)   not null comment 'Globally unique trace ID, as primary key',
    worker_node_id      varchar(64)   not null comment 'Worker node ID'
)
ENGINE=OLAP
DUPLICATE KEY(tenant_id,patient_id,study_uid,series_uid,sop_uid)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 1
PROPERTIES("replication_num" = "1");


DROP TABLE IF  EXISTS dicom_state_meta;
CREATE TABLE IF NOT EXISTS dicom_state_meta (
    -- Basic identification information
    tenant_id VARCHAR(64) NOT NULL,
    patient_id VARCHAR(64) NOT NULL,
    study_uid VARCHAR(64) NOT NULL,
    series_uid VARCHAR(64) NOT NULL,
    study_uid_hash  VARCHAR(20)  NOT NULL,
    series_uid_hash  VARCHAR(20)   NOT NULL,
    study_date_origin VARCHAR(8) NOT NULL,

    -- Patient information
    patient_name VARCHAR(64) NULL,
    patient_sex VARCHAR(1) NULL,
    patient_birth_date DATE NULL,
    patient_birth_time VARCHAR(16) NULL,
    patient_age VARCHAR(16) NULL,
    patient_size DOUBLE NULL,
    patient_weight DOUBLE NULL,
    pregnancy_status INT NULL,

    -- Study information
    study_date DATE NOT NULL,
    study_time VARCHAR(16) NULL,
    accession_number VARCHAR(16) NOT NULL,
    study_id VARCHAR(16) NULL,
    study_description VARCHAR(64) NULL,

    -- Series information
    modality VARCHAR(16) NULL,
    series_number INT NULL,
    series_date DATE NULL,
    series_time VARCHAR(16) NULL,
    series_description VARCHAR(256) NULL,
    body_part_examined VARCHAR(64) NULL,
    protocol_name VARCHAR(64) NULL,
    
    -- Timestamps
    created_time DATETIME NULL,
    updated_time DATETIME NULL
)
ENGINE=OLAP
UNIQUE KEY(tenant_id, patient_id, study_uid, series_uid)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 1
PROPERTIES("replication_num" = "1");


DROP TABLE IF   EXISTS dicom_image_meta ;
CREATE TABLE IF NOT EXISTS dicom_image_meta (
    -- Basic identification information
    tenant_id VARCHAR(64) NOT NULL COMMENT "Tenant ID",
    patient_id VARCHAR(64) NOT NULL COMMENT "Patient ID",
    study_uid VARCHAR(64) NOT NULL COMMENT "Study UID",
    series_uid VARCHAR(64) NOT NULL COMMENT "Series UID",
    sop_uid VARCHAR(64) NOT NULL COMMENT "Instance UID",

    -- Hash values
    study_uid_hash  VARCHAR(20) NOT NULL COMMENT "Study UID hash value",
    series_uid_hash   VARCHAR(20)   NOT NULL COMMENT "Series UID hash value",

    -- Time related
    study_date_origin DATE NOT NULL COMMENT "Study date (original format)",
    content_date DATE COMMENT "Content date",
    content_time VARCHAR(32) COMMENT "Content time",


    -- Image basic information
    instance_number INT COMMENT "Instance number",
    image_type VARCHAR(128) COMMENT "Image type",
    image_orientation_patient VARCHAR(128) COMMENT "Image orientation (patient coordinate system)",
    image_position_patient VARCHAR(64) COMMENT "Image position (patient coordinate system)",

    -- Image dimension parameters
    slice_thickness DOUBLE COMMENT "Slice thickness",
    spacing_between_slices DOUBLE COMMENT "Spacing between slices",
    slice_location DOUBLE COMMENT "Slice location",

    -- Pixel data attributes
    samples_per_pixel INT COMMENT "Samples per pixel",
    photometric_interpretation VARCHAR(32) COMMENT "Photometric interpretation",
    width INT COMMENT "Image rows",
    columns INT COMMENT "Image columns",
    bits_allocated INT COMMENT "Bits allocated",
    bits_stored INT COMMENT "Bits stored",
    high_bit INT COMMENT "High bit",
    pixel_representation INT COMMENT "Pixel representation",

    -- Reconstruction parameters
    rescale_intercept DOUBLE COMMENT "Reconstruction intercept",
    rescale_slope DOUBLE COMMENT "Reconstruction slope",
    rescale_type VARCHAR(64) COMMENT "Reconstruction type",
    window_center VARCHAR(64) COMMENT "Window center",
    window_width VARCHAR(64) COMMENT "Window width",

    -- Transfer and classification information
    transfer_syntax_uid VARCHAR(64) NOT NULL COMMENT "Transfer syntax UID",
    pixel_data_location VARCHAR(512) COMMENT "Pixel data location",
    thumbnail_location VARCHAR(512) COMMENT "Thumbnail location",
    sop_class_uid VARCHAR(64) NOT NULL COMMENT "SOP class UID",
    image_status VARCHAR(32) COMMENT "Image status",
    space_size BIGINT COMMENT "Occupied space size",
    created_time DATETIME COMMENT "Creation time",
    updated_time DATETIME COMMENT "Update time"
)
ENGINE=OLAP
UNIQUE KEY(tenant_id, patient_id, study_uid, series_uid, sop_uid)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 1
PROPERTIES("replication_num" = "1");

---------------------------------------------------------
---------------------------------------------------------
---------------------------------------------------------
CREATE ROUTINE LOAD medical_object_load ON dicom_object_meta
COLUMNS (
    trace_id,
    worker_node_id,
    tenant_id,
    patient_id,
    study_uid,
    series_uid,
    sop_uid,
    file_size,
    file_path,
    transfer_syntax_uid,
    number_of_frames,
    created_time,
    series_uid_hash,
    study_uid_hash,
    accession_number,
    target_ts,
    study_date,
    transfer_status,
    source_ip,
    source_ae
)
PROPERTIES (
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "10",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200",
    "format" = "json",
    "max_error_number" = "1000"
)
FROM KAFKA (
    "kafka_broker_list" = "127.0.0.1:9092",
    "kafka_topic" = "log_queue",
    "kafka_partitions" = "0",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);


CREATE ROUTINE LOAD medical_state_load ON dicom_state_meta
COLUMNS (
        tenant_id  ,
        patient_id ,
        study_uid ,
        series_uid,
        study_uid_hash,
        series_uid_hash,
        study_date_origin,
        patient_name,
        patient_sex ,
        patient_birth_date ,
        patient_birth_time,
        patient_age,
        patient_size,
        patient_weight,
        pregnancy_status,      
        study_date,
        study_time,
        accession_number,
        study_id,
        study_description,
        modality,
        series_number,
        series_date,
        series_time,
        series_description,
        body_part_examined,
        protocol_name,
        series_related_instances,
        created_time,
        updated_time = NOW()
)
PROPERTIES (
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "10",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200",
    "format" = "json",
    "max_error_number" = "1000",
    "strip_outer_array" = "false"
)
FROM KAFKA (
    "kafka_broker_list" = "127.0.0.1:9092",
    "kafka_topic" = "dicom_state_queue",
    "kafka_partitions" = "0",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);



CREATE ROUTINE LOAD medical_image_load ON dicom_image_meta
COLUMNS (
    tenant_id,
    patient_id,
    study_uid,
    series_uid,
    sop_uid,
    study_uid_hash,
    series_uid_hash,
    study_date_origin,
    content_date,
    content_time,
    instance_number,
    image_type,
    image_orientation_patient,
    image_position_patient,
    slice_thickness,
    spacing_between_slices,
    slice_location,
    samples_per_pixel,
    photometric_interpretation,
    width,
    `columns`,
    bits_allocated,
    bits_stored,
    high_bit,
    pixel_representation,
    rescale_intercept,
    rescale_slope,
    rescale_type,
    window_center,
    window_width,
    transfer_syntax_uid,
    pixel_data_location,
    thumbnail_location,
    sop_class_uid,
    image_status,
    space_size,
    created_time,
    updated_time = NOW()
)
PROPERTIES (
    "desired_concurrent_number" = "3",

    "max_batch_interval" = "10",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200",
    "format" = "json",
    "max_error_number" = "1000",
    "strip_outer_array" = "false"

)
FROM KAFKA (
    "kafka_broker_list" = "127.0.0.1:9092",
    "kafka_topic" = "dicom_image_queue",
    "kafka_partitions" = "0",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

This creates 3 ROUTINE LOAD tasks corresponding to DICOM object metadata, DICOM state metadata, and DICOM image metadata. WADO-RS access logs can be handled similarly based on actual requirements.

For more information about building cloud DICOM systems, see our guide on how to build cloud DICOM systems.

This comprehensive guide provides all necessary preparations for building scalable cloud DICOM-WEB services, covering infrastructure setup, database design, and system architecture considerations for medical imaging applications.

GoTo  Summary  : [how-to-build-cloud-dicom](/posts/how-to-build-cloud-dicom)