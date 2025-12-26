---
title: "How to Build Scalable Cloud DICOM-WEB Services | DICOM Cloud"
date: 2025-11-19T10:23:56+08:00
keywords: "DICOM-WEB, medical imaging, healthcare cloud, DICOM storage, cloud DICOM platform, DICOM services, medical DICOM platform"
description: "Complete guide to building scalable cloud DICOM-WEB services using open-source projects. Learn architecture, implementation, and deployment strategies for medical imaging platforms."
weight: 1
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "DICOM storage", "cloud DICOM platform"]
categories: ["Medical Imaging", "DICOM Development", "Healthcare Technology"]
slug: "how-to-build-scalable-cloud-dicom-web-services"
---

# How to Build Scalable Cloud DICOM-WEB Services

Learn how to build distributed DICOM-WEB services using open-source projects.

## Overall Architecture

1. **Apache Kafka** as message queue. RedPanda can be used as alternative during development.
2. **Apache Doris** as data warehouse. Provides storage for DicomStateMeta, DicomImageMeta, and WadoAccessLog, offering query and statistical analysis.
3. **PostgreSQL** as database. Provides data storage and indexing functions. Stores only patient, study, and series level metadata to fully leverage ACID properties of relational databases. Can be scaled later with Citus.
4. **Redis** as cache. Provides data caching functionality.
5. **Nginx** as reverse proxy server. Provides load balancing, static files, and TLS termination.

Files received by the storage service are first stored locally, then sent to the message queue via Kafka.

Message Body = {
    TransferSyntax, SopInstanceUID, StudyInstanceUID, SeriesInstanceUID, PatientID, FileName, FileSize, FilePath
}

Messages are distributed to multiple queues:
1. **Storage Queue**: Stores file information, file storage path, file size.
2. **Index Queue**: Extracts file TAG information, including PatientInformation, StudyInformation, SeriesInformation, ImageInformation, and writes to Doris database.
3. **Conversion Queue**: For some transfer syntaxes that Cornerstone3D cannot parse, convert to formats that CornerstoneJS can parse. Failed conversions are written to Doris conversion record table.

## Service Purposes and Descriptions

| Service | Purpose |
|---------|---------|
| wado-server | DICOM Web WADO-RS API implementation, supports OAuth2. |
| wado-storescp | CStoreSCP Provider, writes DICOM files to disk and publishes messages to Kafka: storage_queue, log_queue |
| wado-consumer | Consumes storage-queue, and publishes messages to Kafka: dicom_state_queue, dicom_image_queue, uses stream_load to write to Doris |
| wado-webworker | Generates metadata for wado-server and updates related instances for series and study. |

### wado-server

DICOM Web WADO-RS API implementation, with OAuth2 authentication enabled based on configuration options.

### wado-storescp

DICOM CStoreSCP service that receives DICOM files and writes them to disk, simultaneously distributing messages to Kafka: storage_queue, log_queue
- 1. Storage file path: ${ROOT}/<TenantID>/<StudyInstanceUID>/<SeriesInstanceUID>/<SopInstanceUID>.dcm
- 2. Send messages to message queues: storage_queue, log_queue

### wado-consumer

Consumes Kafka message queue storage_queue, extracts DicomStateMeta to main database, and publishes DicomStateMeta and DicomImageMeta to Doris database via message queues: dicom_state_queue, dicom_image_queue
- 1. Read from storage-queue, write to dicom_state_queue, dicom_image_queue
- 2. Extract DicomStateMeta to main database
- 3. Publish DicomStateMeta and DicomImageMeta to Doris database using Stream_Load mode via message queues: dicom_state_queue, dicom_image_queue

### wado-webworker

Periodically scans database and generates JSON-formatted metadata based on last update time to accelerate WADO-Server access

## Summary

Subsequent related subsystem introductions will all revolve around this foundation.

***Storage file and metadata paths:***

1. Storage file path: ${ROOT}/TenantID/StudyInstanceUID/SeriesInstanceUID/SopInstanceUID.dcm
2. StudyMetadata storage path: ${ROOT}/metadata/StudyInstanceUID/SeriesInstanceUID.json
3. SeriesMetadata storage path: ${ROOT}/metadata/StudyInstanceUID.json

***Tools needed during development:***

1. [DICOMwebJS](https://github.com/cornerstonejs/dicomweb-client)
2. [Dcm4che](https://dcm4che.org/)
3. [Wireshark](https://www.wireshark.org/)
4. [DVTk](https://www.dvtk.org)
5. [DCMTK](https://dcmtk.org/en/dcmtk/dcmtk-tools/)
6. [fo-dicom](https://github.com/fo-dicom/fo-dicom)
7. [Cornerstone3D](https://www.cornerstonejs.org/)
8. [MicroDicom DICOM Viewer](https://www.microdicom.com/)
9. [RadiAnt DICOM Viewer](https://www.radiantviewer.com/)

***Of course, don't forget the most important DICOM standard:***

[DICOM Standard](https://www.dicomstandard.org/current)

***Additionally, you need to master the following skills:***

1. Docker skills
2. Nginx skills
3. Linux skills
4. PostgreSQL skills

How To Build A Cloud DICOM-WEB Service. Follow Me:

** Part One: Database Design and Database Interface **

1. [How to use fo-dicom to construct CStore SCU](/posts/1-how-to-use-fo-dicom-construct-cstore-scu)
   
2. [DICOM Cloud Prepare](/posts/2-dicom-cloud-prepare)

3. [DICOM Cloud Part ONE Database Design](/posts/3.1-dicom-database-design)
   
4. [DICOM Cloud Part ONE Database Type Defines](/posts/3.2-dicom-database-types)

5. [DICOM Cloud Part ONE DICOM Database Interface](/posts/3.3-dicom-database-interface)

** Part Two: Common Function and Configuration Manager for Services Building **

1. [DICOM Cloud Part Two 4.1 Common Module Introduction](/posts/4.1-common-index)
2. [DICOM Cloud Part Two 4.2 Storage Configuration](/posts/4.2-common-configuration)
3. [DICOM Cloud Part Two 4.3 Database Factory](/posts/4.3-common-database)
4. [DICOM Cloud Part Two 4.4 Redis](/posts/4.4-common-redis)
5. [DICOM Cloud Part Two 4.4 Kafka Message Publish](/posts/4.5-common-messaging)
6. [DICOM Cloud Part Two 4.6 Extract DICOM Information](/posts/4.6-common-dicom-processing)
7. [DICOM Cloud Part Two 4.7 Security](/posts/4.7-common-security)
8. [DICOM Cloud Part Two 4.8 Write File To Storage](/posts/4.8-common-storage) 
9. [DICOM Cloud Part Two 5.1 wado-storescp Implementation](/posts/5.1-wado-storescp-implemention) 
10. [DICOM Cloud Part Two 6.1 wado-consumer Service](/posts/6.1-wado-consumer) 

Use Cornerstone3D to View DICOM Images in a Web Browser.

And finally:

![Image Display How to load DICOM Image With WADO-RS](/1.png)

![Invert Color Function DICOM Viewer Invert Color](/2.png)

![DICOM View Load Default CT Images](/3.png)

![Window Adjustment DICOM Viewer Adjust WINDOW Level/WINDOW Center](/4.png)

![Scroll DICOM Viewer Scroll](/5.png)

![Measurement Tools DICOM Viewer Measurement Tools](/6.png)

![Invert Color Function Invert Color2](/7.png)

![MPR DICOM MPR Reconstructor](/10.png)

![MP2 DICOM Viewer MPR With CrossHair](/11.png)

---

*This comprehensive guide provides everything you need to know about building scalable cloud DICOM-WEB services, from architecture design to implementation and deployment. Perfect for healthcare technology professionals looking to implement modern medical imaging solutions.*