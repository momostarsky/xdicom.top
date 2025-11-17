---
title: "DICOM文件中核心Infomation 结构及说明"
date: 2025-11-14T16:17:56+08:00
description: "DICOM文件中核心对象模型结构及说明"
keywords: "DICOM PACS"
draft: false
---

# DICOM基础概念

DICOM（Digital Imaging and Communications in Medicine）是医学影像及其相关信息的国际标准（ISO 12052），用于存储、交换和传输医学图像及相关数据。DICOM 文件不仅包含像素数据（即图像本身），还包含大量描述该图像的元数据，这些元数据以“数据元素”（Data Elements）的形式组织在“信息对象定义”（Information Object Definition, IOD）中。

- ✅ 一、DICOM 文件的核心结构
  
  - 文件头（File Meta Information）
    - 固定长度为 128 字节的前导（通常全为 0x00，可选）
    - 4 字节的 DICOM 前缀 "DICM"
    - 文件元信息组（Group 0002），包含：
      -   (0002,0000) File Meta Information Group Length
      -   (0002,0001) File Meta Information Version
      -   (0002,0002) Media Storage SOP Class UID
      -   (0002,0003) Media Storage SOP Instance UID
      -   (0002,0010) Transfer Syntax UID
      -   (0002,0012) Implementation Class UID
      -   (0002,0013) Implementation Version Name

  - 数据集（Dataset）
    - 包含实际的图像信息和元数据，遵循特定的 IOD（如 CT Image IOD、MR Image IOD 等）
    - 数据元素按标签（Tag）组织，格式为 (gggg,eeee)，其中 gggg 是组号，eeee 是元素号
    - 每个数据元素包括：Tag、VR（Value Representation）、Value Length、Value
 

 - ✅ 二、核心信息对象（IOD）结构说明

每个 DICOM 图像属于一种 SOP Class（Service-Object Pair Class），例如：

- CT Image Storage (1.2.840.10008.5.1.4.1.1.2)
- MR Image Storage (1.2.840.10008.5.1.4.1.1.4)
- Secondary Capture Image Storage (1.2.840.10008.5.1.4.1.1.7)

每种 SOP Class 对应一个 IOD，定义了必须（Mandatory）或可选（Optional）的数据元素。

  **典型 IOD 模块（Modules）包括：**

| 模块名称         | 类型     | 说明                                                       |
|------------------|----------|------------------------------------------------------------|
| Patient          | Module   | 患者基本信息（如：姓名、ID、性别、出生日期等）                     |
| Study            | Module   | 检查（Study）信息（如：Study Instance UID、检查日期/时间、医生等） |
| Series           | Module   | 序列（Series）信息（如：Series Instance UID、模态、设备等）        |
| Equipment        | Module   | 设备信息（如：制造商、型号、软件版本等）                           |
| Image            | Module   | 图像特有信息（如：图像方向、位置、像素间距、窗宽窗位等）           |
| Pixel Data       | Module   | 像素数据本身  (7fe0,0010)         |          |


- ✅ 三、关键数据元素示例

| TAG(gggg,eeee) | 名称 |      说明   |
|---------------|--------------|--------|
| (0010,0010)  | Patient's Name        | 患者姓名    |
| (0010,0020)  | Patient ID        | 患者唯一标识      |
| (0010,0030)  | Patient's Birth Date     | 出生日期  |
| (0010,0040)  | Patient's Sex          | 性别       |
| (0020,000D)  | Study Instance UID      | 检查唯一标识符    |
| (0020,000E)  | Series Instance UID       | 序列唯一标识符    |
| (0008,0018)  | SOP Instance UID       | 图像实例唯一标识符    |
| (0008,0060)  | Modality             | 成像模态（CT、MR、US 等）  |
| (0008,0020)  | Study Date           | 检查日期    |
| (0008,0030)  | Study Time         | 检查时间  |
| (0020,0032)  | Image Position (Patient)  | 图像在患者坐标系中的位置       |
| (0020,0037)  | Image Orientation (Patient)  | 图像方向（行/列向量）   |
| (0028,0030)  | Pixel Spacing  | 像素物理尺寸（mm）    |
| (0028,0010)  | Rows    | 图像行数             |
| (0028,0011)  | Columns      | 图像列数             |
| (0028,0100)  | Bits Allocated      | 每像素分配的位数      |
| (7FE0,0010)  | Pixel Data      | 实际像素数据（可能压缩）   |


- ✅ 四、传输语法（Transfer Syntax）
  
 决定了数据如何编码（字节序、是否压缩等），常见包括：

   - Implicit VR Little Endian (1.2.840.10008.1.2) —— 默认
   - Explicit VR Little Endian (1.2.840.10008.1.2.1)
   - JPEG Lossless (1.2.840.10008.1.2.4.70)
   - JPEG 2000 (1.2.840.10008.1.2.4.90)
   - Deflated Explicit VR Little Endian (1.2.840.10008.1.2.1.99) 


- ✅ 五、总结
  
DICOM 文件 = 文件元信息 + 数据集（遵循特定 IOD）

核心信息通过分层结构组织：Patient → Study → Series → Image → Pixel Data 对应的TAG为:PatientID → StudyInstanceUID → SeriesInstanceUID → SOPInstanceUID → PixelData.  在ER模型中表示为:
Patient → Study → Series → Image 
1      -> N  ---->N  -->N  
对于多帧图像 NumberOfFrames>1 的需要单独处理。多帧主要是超声,心血管造影,心电图等设备生成到的DICOM文件。

所有信息通过标准化标签（Tag）访问，支持跨厂商互操作性。

如需解析或生成 DICOM 文件，推荐使用成熟库如：

-   Python: pydicom
-   C++: DCMTK
-   Java: dcm4che
-   C#: fo-dicom
-   RUST: dicom-rs

如果是AI学习大数据分析等,推荐采用Pydicom。 开发PACS系统时，推荐采用
FO-DICOM。主要是目前dcm4che的兼容性相比fo-dicom差一些。dicom-rs 主要是难度高一些。DCMTK 对于JPEG2000传输语法的解码需要收费。dcm4che 在解析速度上目前是最慢的。但是辅助工具方面dcm4che 反而是最多的。

另外涉及到一些调试工具，如：
-   DVTk 
-   Wireshark + DICOM plugin
  
对于调试DICOM通讯比较好。
