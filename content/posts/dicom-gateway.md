---
title: "如何用FoDICOM构建DICOM CStoreSCU 工具,批量发送DICOM文件"
date: 2025-11-20T09:33:23+08:00
keywords: "DICOM Gateway, medical imaging,healthcare cloud,DICOM 网关"
description: "DICOM网关概念, DICOM Gateway具体需要实现那些功能,有哪些开源的DICOM网关可用来参考"
draft: false
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "DICOM storage"]
---

#### DICOM 网关（DICOM Gateway）是一种专门用于在不同医疗信息系统之间转换、路由、适配或处理 DICOM数据流的中间件系统。

它通常部署在网络边界或系统集成点，起到“协议转换器”、“数据路由器”和“数据预处理器”的作用。

一、DICOM 网关是什么？

从概念上讲，DICOM 网关是传统网络“网关”在医学影像领域的具体应用：

连接不同协议/格式系统：例如将传统的 DICOM C-STORE（基于 TCP 的 DIMSE 协议）转换为现代的 DICOMweb（基于 HTTP/REST 的 WADO/QIDO/STOW）。

实现跨系统互操作性：如连接老旧 PACS 与云原生影像平台（如 Azure Health Data Services、Google Healthcare API）。

执行数据预处理：如去标识化（de-identification）、元数据标准化、图像压缩、格式转换等。

二、DICOM 网关需要实现哪些核心功能？

一个完整的 DICOM 网关通常需支持以下功能模块：

1. DICOM DIMSE 支持（传统协议）
-  接收/发送 C-ECHO、C-STORE、C-FIND、C-MOVE 等服务类用户（SCU/SCP）请求。
-  支持 AETitle（应用实体标题）认证与过滤。
-  处理 DICOM 关联（Association）建立与释放。
  
2. DICOMweb 支持（现代 Web API）
- 实现 STOW-RS（上传）、WADO-RS（下载）、QIDO-RS（查询）。
- 支持 multipart/related 或 single-part 上传。
- 兼容 FHIR ImagingStudy 等扩展标准（可选）。
  
3. 数据路由与转发
- 根据规则（如 Modality、StudyInstanceUID、AETitle）将 DICOM 实例转发到一个或多个目标（PACS、云服务、AI引擎等）。
- 支持失败重试、队列管理、流量控制。
  
4. 数据处理与转换
- 去标识化（De-identification）：移除或替换患者隐私信息（如 PatientName, PatientID），符合 HIPAA 或 GDPR。
- 元数据标准化：统一不同设备厂商的标签写法（如将 Siemens 的私有标签映射为标准标签）。
- 图像格式转换：如将 JPEG2000 转为 JPEG-LS，或将多帧转为单帧序列。

5. 日志、监控与审计
- 记录所有接收/转发事件。
- 提供健康检查接口（如 /healthz）。
- 支持 Prometheus/Grafana 监控（现代网关常见）。

6. 安全机制
- TLS 加密（DICOM over TLS、HTTPS）。
- 身份认证（API Key、OAuth2、JWT）。
- IP 白名单/AETitle 白名单。

三、有哪些开源的 DICOM 网关项目可供参考？

1. Orthanc：一个开源的 DICOM 网关，支持 DICOM DIMSE 和 DICOMweb。
   C++ 开发。 功能强大的开源 PACS 服务器，也可作为网关使用。
   适合小型医院 PACS 替代、DICOM 中转站、AI 集成入口。
   https://www.orthanc-server.com/

2. dcm4chee-arc-light + dcm4che toolkit
   java 开发。企业级开源 PACS（dcm4chee）的轻量版，配合 dcm4che 工具包可构建网关。
   适用大型机构、需符合 IHE 标准的集成项目。

3. fo-dicom + 自定义服务（.NET 生态）
   C#开发。fo-dicom 是 .NET 平台最流行的 DICOM 库，虽非完整网关，但可用于快速构建自定义网关。
   适用.NET 技术栈团队、Azure 医疗云集成。
   https://github.com/fo-dicom/fo-dicom


四、典型应用场景举例

| 场景 | 网关作用 |
|------|----------|
| 医院 → 云 PACS | 将本地 DICOM 设备数据通过网关转为 DICOMweb 上传至 Azure/Google Cloud |
| AI 模型接入 | 网关接收影像后，转发副本给 AI 推理服务，并回传结构化结果 |
| 多院区数据汇聚 | 各分院 DICOM 数据经网关去标识化后汇总至中心研究平台 |
| 老旧设备对接新系统 | 旧 CT 机只支持 C-STORE，网关将其转为 REST API 供新 HIS 调用 |


#### 总结

DICOM 网关本质是一个智能中介，解决医学影像系统间的“语言不通”问题。其核心价值在于：

- 协议适配（DIMSE ↔ DICOMweb）
- 数据治理（去标识、标准化）
- 灵活路由（一对多、条件转发）
