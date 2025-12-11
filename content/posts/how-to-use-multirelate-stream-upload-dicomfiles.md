---
title: "构建可扩展的云DICOM-WEB服务的前期准备工作"
date: 2025-12-11T15:21:56+08:00
keywords: "DICOM-WEB , medical imaging,healthcare cloud,DICOM storage"
description: "利用CURL脚步或是NETCORE应用程序实现多文件上传."
draft: false
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "STOW-RS"]
---

### 利用CURL脚本或NETCORE应用程序实现多文件上传。

在开发STOW-RS的过程中, 为了测试RESTFul API接口， 创建了一个CURL脚本， 该脚本用于上传多个DICOM文件。

需要注意的是,RESTFul 接口要求上传文件必须是multipart/related。Content-Type 为: multipart/related;boundary=DICOM_BOUNDARY;type=application/dicom" 的格式.

```bash
#!/bin/bash

# 检查参数
if [ $# -eq 0 ]; then
    echo "Usage: $0 <dicom_directory>"
    echo "Example: $0 /home/dhz/amprData"
    exit 1
fi

DICOM_DIR="$1"

# 检查目录是否存在
if [ ! -d "$DICOM_DIR" ]; then
    echo "Error: Directory '$DICOM_DIR' does not exist"
    exit 1
fi

BOUNDARY="DICOM_BOUNDARY"
TEMP_FILE="multipart_request_largdata.tmp"

# 检查是否有DICOM文件
DICOM_FILES=($(find "$DICOM_DIR" -type f -name "*.dcm"))
if [ ${#DICOM_FILES[@]} -eq 0 ]; then
    echo "Warning: No DICOM files found in '$DICOM_DIR'"
    exit 0
fi

echo "Found ${#DICOM_FILES[@]} DICOM files"

# 1. 初始化文件（不包含JSON部分）
> "$TEMP_FILE"

# 2. 循环处理所有DICOM文件（第一个文件不需要前置分隔符）
for i in "${!DICOM_FILES[@]}"; do
    dicom_file="${DICOM_FILES[$i]}"

    # 除了第一个文件，其他文件都需要前置分隔符
    if [ $i -gt 0 ]; then
        printf -- "\r\n--%s\r\n" "$BOUNDARY" >> "$TEMP_FILE"
    else
        # 第一个文件需要起始分隔符
        printf -- "--%s\r\n" "$BOUNDARY" >> "$TEMP_FILE"
    fi

    printf -- "Content-Type: application/dicom\r\n\r\n" >> "$TEMP_FILE"

    # 附加 DICOM 文件的内容
    cat "$dicom_file" >> "$TEMP_FILE"

    echo "Added file: $(basename "$dicom_file")"
done

# 3. 写入请求体的结束分隔符
printf -- "\r\n--%s--\r\n" "$BOUNDARY" >> "$TEMP_FILE"

# 4. 计算文件大小
CONTENT_LENGTH=$(wc -c < "$TEMP_FILE" | tr -d ' ')

echo "Total content length: $CONTENT_LENGTH bytes"

# 5. 发送请求
curl -X POST http://localhost:9000/stow-rs/v1/studies \
     -H "Content-Type: multipart/related; boundary=$BOUNDARY; type=application/dicom" \
     -H "Accept: application/json" \
     -H "x-tenant: 1234567890" \
     -H "Content-Length: $CONTENT_LENGTH" \
     --data-binary @"$TEMP_FILE"

# 6. 清理临时文件
rm "$TEMP_FILE"

echo "Upload completed"

```

```csharp
using System.Text;

namespace MakeMultirelate
{
    public class ConstructPostRequest : IDisposable
    {
        private const string Boundary = "DICOM_BOUNDARY";
        private readonly HttpClient _httpClient = new();

        public async Task SendDicomFilesAsync(List<string> dicomFilePaths, string url, string tenantId)
        {
            // 预估内存流大小以优化性能
            long estimatedSize = 0;
            foreach (var filePath in dicomFilePaths)
            {
                if (!File.Exists(filePath))
                    throw new FileNotFoundException($"DICOM file not found: {filePath}");

                // 获取文件大小并累加
                var fileInfo = new FileInfo(filePath);
                estimatedSize += fileInfo.Length;
            }

            // 添加边界和头部的预估大小(每个文件大约需要额外200字节用于分隔符和头部)
            estimatedSize += dicomFilePaths.Count * 200;
            // 添加结束边界大小
            estimatedSize += Boundary.Length + 10;

            // 创建内存流来构建 multipart 内容，使用预估大小初始化容量
            using var memoryStream = new MemoryStream((int)Math.Min(estimatedSize, int.MaxValue));

            // 构建 multipart 内容
            foreach (var filePath in dicomFilePaths)
            {
                if (!File.Exists(filePath))
                    throw new FileNotFoundException($"DICOM file not found: {filePath}");

                // 添加分隔符和头部
                var separator = Encoding.UTF8.GetBytes($"\r\n--{Boundary}\r\n");
                var header = Encoding.UTF8.GetBytes("Content-Type: application/dicom\r\n\r\n");

                if (memoryStream.Length == 0)
                {
                    // 第一个部分不需要前导分隔符
                    separator = Encoding.UTF8.GetBytes($"--{Boundary}\r\n");
                }

                await memoryStream.WriteAsync(separator, 0, separator.Length);
                await memoryStream.WriteAsync(header, 0, header.Length);

                // 读取并添加 DICOM 文件内容
                var fileBytes = await File.ReadAllBytesAsync(filePath);
                await memoryStream.WriteAsync(fileBytes, 0, fileBytes.Length);
            }

            // 添加结束分隔符
            var endBoundary = Encoding.UTF8.GetBytes($"\r\n--{Boundary}--\r\n");
            await memoryStream.WriteAsync(endBoundary, 0, endBoundary.Length);

            // 准备请求内容
            var content = new ByteArrayContent(memoryStream.ToArray());
            content.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue("multipart/related");
            content.Headers.ContentType.Parameters.Add(
                new System.Net.Http.Headers.NameValueHeaderValue("boundary", Boundary));
            content.Headers.ContentType.Parameters.Add(
                new System.Net.Http.Headers.NameValueHeaderValue("type", "application/dicom"));

            // 设置请求头
            _httpClient.DefaultRequestHeaders.Clear();
            _httpClient.DefaultRequestHeaders.Add("Accept", "application/json");
            _httpClient.DefaultRequestHeaders.Add("x-tenant", tenantId);


            // 发送请求
            var response = await _httpClient.PostAsync(url, content);

            // 处理响应
            var responseContent = await response.Content.ReadAsStringAsync();
            Console.WriteLine($"Status Code: {response.StatusCode}");
            Console.WriteLine($"Response: {responseContent}");
        }


        // 新增的重载函数：基于目录递归查找DICOM文件
        public async Task SendDicomFilesAsync(string dicomDirectory, string url, string tenantId)
        {
            if (!Directory.Exists(dicomDirectory))
                throw new DirectoryNotFoundException($"DICOM directory not found: {dicomDirectory}");

            // 递归查找所有.dcm扩展名的文件
            var dicomFiles = Directory.GetFiles(dicomDirectory, "*.dcm", SearchOption.AllDirectories);

            // 如果没有找到文件，抛出异常
            if (dicomFiles.Length == 0)
                throw new FileNotFoundException($"No DICOM files found in directory: {dicomDirectory}");

            // 调用原始方法发送文件
            await SendDicomFilesAsync(dicomFiles.ToList(), url, tenantId);
        }

        public void Dispose()
        {
            _httpClient.Dispose();
        }
    }
}

```

