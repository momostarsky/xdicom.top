---
title: "构建可扩展的云DICOM-WEB服务的前期准备工作"
date: 2025-11-19T15:21:56+08:00
keywords: "DICOM-WEB , medical imaging,healthcare cloud,DICOM storage"
description: "构建云DICOM-WEB服务的前期准备工作,主要是数据库,消息队列,缓存,开发语言,服务框架等配置."
draft: false
---

本文介绍了一个基于 Rust 语言开发的 DICOM 医疗影像系统架构设计，该系统采用现代化的技术栈，包括 PostgreSQL 作为主索引数据库、Apache Doris 用于日志存储、RedPanda 作为消息队列以及 Redis 作为缓存。系统设计支持单机运行和分布式扩展，充分利用了 Rust 语言的安全性和性能优势。

DICOM 医疗影像系统架构概览及运行环境介绍

核心组件
-   PostgreSQL: 主索引数据库，存储患者、检查、序列等核心元数据
-   Apache Doris: 日志存储，记录 DICOM CStoreSCP 服务和 WADO-RS 服务访问日志
-   RedPanda: 消息队列，处理系统间异步通信
-   Redis: 缓存层，提升系统响应速度
-   Rust: 开发语言，利用 dicom-rs 库处理 DICOM 数据

服务模块

-   wado-storescp: DICOM CStoreSCP 服务，接收 DICOM 文件并写入磁盘
-   wado-consumer: 消费消息队列中的存储事件，提取元数据并写入数据库
-   wado-server: DICOM WEB WADO-RS API 接口实现
-   wado-webworker: 定期生成 JSON 格式的元数据用于加速访问
  
数据库设计
-   PostgreSQL 主索引数据库
-   PostgreSQL 作为主索引数据库存储核心元数据，包括:
-   dicom_state_meta: 存储患者、检查、序列级别的元数据
-   dicom_json_meta: 记录需要生成 JSON 格式元数据的序列信息
  
Apache Doris 日志存储
-   Apache Doris 用于存储各种服务日志:
-   DicomStateMeta: DICOM 状态元数据
-   DicomImageMeta: DICOM 影像元数据
-   WadoAccessLog: WADO 访问日志
  
Docker Compose 脚步

我们假设你的数据库服务器IP地址为 192.168.1.14

操作系统 Ubuntu 22.04.5  LTS

- PostgreSQL
```yaml
version: '3'

services:
  pgdb:
    image: ankane/pgvector:latest
    container_name: pgappx  # 你写的是 container:pgappx，应为 container_name
    restart: always
    environment:
      POSTGRES_PASSWORD: "xDicom123"
      POSTGRES_USER: "root"
      PGTZ: "Asia/Shanghai"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
      - ./pg_hba.conf:/var/lib/postgresql/data/pg_hba.conf  # ✅ 正确路径
    ports:
      - "5432:5432"
```

- Redis And  PgAdmin
  
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
    image: rabbitmq:management # 使用官方带有管理插件的RabbitMQ镜像
    ports:
      - "5672:5672" # 暴露RabbitMQ AMQP协议端口
      - "15672:15672" # 暴露RabbitMQ管理界面端口
    environment:
      RABBITMQ_DEFAULT_USER: admin   # 设置默认用户名
      RABBITMQ_DEFAULT_PASS: xDicom123  # 设置默认密码
    volumes:
      - ./rabbitmq/data:/var/lib/rabbitmq # 将宿主机的目录映射到容器内存储消息队列的数据目录（可选）


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

  - Doris  And  RedPanda
  
  参考Doris 和 RedPanda 的官方文档
  
  启动 Doris

```bash
./Doris3.X/3.1.0/fe/bin/start_fe.sh --daemon
./Doris3.X/3.1.0/be/bin/start_be.sh --daemon
```
  
  RedPanda  操作消息队列
 - 创建队列
```bash 
rpk topic create dicom_image_queue  --partitions 1 --replicas 1
rpk topic create dicom_state_queue  --partitions 1 --replicas 1
rpk topic create log_queue          --partitions 1 --replicas 1
rpk topic create storage_queue      --partitions 1 --replicas 1
```

- 清空队列
```bash
rpk topic trim-prefix dicom_image_queue  -p 0 --offset end --no-confirm
rpk topic trim-prefix dicom_state_queue  -p 0 --offset end --no-confirm
rpk topic trim-prefix log_queue          -p 0 --offset end --no-confirm
rpk topic trim-prefix storage_queue      -p 0 --offset end --no-confirm
```

### 数据库初始化脚步

- 主数据库Postgresql 创建脚步
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

-  Doris 数据库表及构建Stream_load 配置
```sql
drop  table IF   EXISTS  dicom_object_meta;
create table IF NOT EXISTS  dicom_object_meta
(
    tenant_id           varchar(64)   not null comment '租户ID',
    patient_id          varchar(64)   not null comment '患者ID',
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
    trace_id            varchar(36)   not null comment '全局唯一追踪ID，作为主键',
    worker_node_id      varchar(64)   not null comment '工作节点 ID'
)
ENGINE=OLAP
DUPLICATE KEY(tenant_id,patient_id,study_uid,series_uid,sop_uid)  -- 逻辑主键，自动去重
DISTRIBUTED BY HASH(tenant_id) BUCKETS 1
PROPERTIES("replication_num" = "1");


DROP TABLE IF  EXISTS dicom_state_meta;
CREATE TABLE IF NOT EXISTS dicom_state_meta (
    -- 基本标识信息
                                                tenant_id VARCHAR(64) NOT NULL,
    patient_id VARCHAR(64) NOT NULL,
    study_uid VARCHAR(64) NOT NULL,
    series_uid VARCHAR(64) NOT NULL,
    study_uid_hash  VARCHAR(20)  NOT NULL,
    series_uid_hash  VARCHAR(20)   NOT NULL,
    study_date_origin VARCHAR(8) NOT NULL,

    -- 患者信息
    patient_name VARCHAR(64) NULL,
    patient_sex VARCHAR(1) NULL,
    patient_birth_date DATE NULL,
    patient_birth_time VARCHAR(16) NULL,
    patient_age VARCHAR(16) NULL,
    patient_size DOUBLE NULL,
    patient_weight DOUBLE NULL,
    pregnancy_status INT NULL,

    -- 检查信息
    study_date DATE NOT NULL,
    study_time VARCHAR(16) NULL,
    accession_number VARCHAR(16) NOT NULL,
    study_id VARCHAR(16) NULL,
    study_description VARCHAR(64) NULL,


    -- 序列信息
    modality VARCHAR(16) NULL,
    series_number INT NULL,
    series_date DATE NULL,
    series_time VARCHAR(16) NULL,
    series_description VARCHAR(256) NULL,
    body_part_examined VARCHAR(64) NULL,
    protocol_name VARCHAR(64) NULL,
    -- 时间戳
    created_time DATETIME NULL,
    updated_time DATETIME NULL
    )
ENGINE=OLAP
UNIQUE KEY(tenant_id, patient_id, study_uid, series_uid)
DISTRIBUTED BY HASH(tenant_id) BUCKETS 1
PROPERTIES("replication_num" = "1");


DROP TABLE IF   EXISTS dicom_image_meta ;
CREATE TABLE IF NOT EXISTS dicom_image_meta (
    -- 基本标识信息
                                                tenant_id VARCHAR(64) NOT NULL COMMENT "租户ID",
    patient_id VARCHAR(64) NOT NULL COMMENT "患者ID",
    study_uid VARCHAR(64) NOT NULL COMMENT "检查UID",
    series_uid VARCHAR(64) NOT NULL COMMENT "序列UID",
    sop_uid VARCHAR(64) NOT NULL COMMENT "实例UID",

    -- 哈希值
    study_uid_hash  VARCHAR(20) NOT NULL COMMENT "检查UID哈希值",
    series_uid_hash   VARCHAR(20)   NOT NULL COMMENT "序列UID哈希值",

    -- 时间相关
    study_date_origin DATE NOT NULL COMMENT "检查日期(原始格式)",
    content_date DATE COMMENT "内容日期",
    content_time VARCHAR(32) COMMENT "内容时间",


    -- 图像基本信息
    instance_number INT COMMENT "实例编号",
    image_type VARCHAR(128) COMMENT "图像类型",
    image_orientation_patient VARCHAR(128) COMMENT "图像方向(患者坐标系)",
    image_position_patient VARCHAR(64) COMMENT "图像位置(患者坐标系)",

    -- 图像尺寸参数
    slice_thickness DOUBLE COMMENT "层厚",
    spacing_between_slices DOUBLE COMMENT "层间距",
    slice_location DOUBLE COMMENT "切片位置",

    -- 像素数据属性
    samples_per_pixel INT COMMENT "每个像素采样数",
    photometric_interpretation VARCHAR(32) COMMENT "光度解释",
    width INT COMMENT "图像行数",
    columns INT COMMENT "图像列数",
    bits_allocated INT COMMENT "分配位数",
    bits_stored INT COMMENT "存储位数",
    high_bit INT COMMENT "高比特位",
    pixel_representation INT COMMENT "像素表示法",

    -- 重建参数
    rescale_intercept DOUBLE COMMENT "重建截距",
    rescale_slope DOUBLE COMMENT "重建斜率",
    rescale_type VARCHAR(64) COMMENT "重建类型",
    window_center VARCHAR(64) COMMENT "窗位中心",
    window_width VARCHAR(64) COMMENT "窗宽",

    -- 传输和分类信息
    transfer_syntax_uid VARCHAR(64) NOT NULL COMMENT "传输语法UID",
    pixel_data_location VARCHAR(512) COMMENT "像素数据位置",
    thumbnail_location VARCHAR(512) COMMENT "缩略图位置",
    sop_class_uid VARCHAR(64) NOT NULL COMMENT "SOP类UID",
    image_status VARCHAR(32) COMMENT "图像状态",
    space_size BIGINT COMMENT "占用空间大小",
    created_time DATETIME COMMENT "创建时间",
    updated_time DATETIME COMMENT "更新时间",
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

此处创建了3个ROUTINE LOAD任务，分别对应了DICOM对象的元数据、DICOM状态元数据和DICOM图像元数据。
WADO-RS ACCESS_LOG 可以同样处理.看实际需求.

GoTo  Summary  : [how-to-build-cloud-dicom](/posts/how-to-build-cloud-dicom)