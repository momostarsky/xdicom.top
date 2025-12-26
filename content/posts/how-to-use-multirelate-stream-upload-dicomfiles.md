---
title: "How to Use Multi-Related Stream to Upload DICOM Files | Cloud DICOM-WEB Service"
date: 2025-12-11T15:21:56+08:00
keywords: "DICOM-WEB, medical imaging, healthcare cloud, DICOM storage, STOW-RS, multipart upload, DICOM file upload"
description: "Learn how to implement multi-file upload using CURL scripts or .NET Core applications for DICOM-WEB services. Complete guide for STOW-RS implementation with multipart/related streaming."
draft: false
tags: ["DICOM-WEB", "medical imaging", "healthcare cloud", "STOW-RS", "multipart upload"]
categories: ["Medical Imaging", "DICOM Development", "Healthcare Technology"]
slug: "how-to-use-multi-related-stream-upload-dicom-files"
---

# How to Use Multi-Related Stream to Upload DICOM Files

## Implementing Multi-File Upload with CURL Scripts or .NET Core Applications

During STOW-RS development, to test RESTful API interfaces, we created a CURL script that uploads multiple DICOM files.

It's important to note that RESTful interfaces require uploads to be in `multipart/related` format with Content-Type: `multipart/related;boundary=DICOM_BOUNDARY;type=application/dicom`.

### CURL Script Implementation

```bash
#!/bin/bash

# Check parameters
if [ $# -eq 0 ]; then
    echo "Usage: $0 <dicom_directory>"
    echo "Example: $0 ~/amprData"
    exit 1
fi

DICOM_DIR="$1"

# Check if directory exists
if [ ! -d "$DICOM_DIR" ]; then
    echo "Error: Directory '$DICOM_DIR' does not exist"
    exit 1
fi

BOUNDARY="DICOM_BOUNDARY"
TEMP_FILE="multipart_request_largdata.tmp"

# Check for DICOM files
DICOM_FILES=($(find "$DICOM_DIR" -type f -name "*.dcm"))
if [ ${#DICOM_FILES[@]} -eq 0 ]; then
    echo "Warning: No DICOM files found in '$DICOM_DIR'"
    exit 0
fi

echo "Found ${#DICOM_FILES[@]} DICOM files"

# 1. Initialize file (without JSON part)
> "$TEMP_FILE"

# 2. Loop through all DICOM files (first file doesn't need prefix separator)
for i in "${!DICOM_FILES[@]}"; do
    dicom_file="${DICOM_FILES[$i]}"

    # Add prefix separator for all files except the first
    if [ $i -gt 0 ]; then
        printf -- "\r\n--%s\r\n" "$BOUNDARY" >> "$TEMP_FILE"
    else
        # First file needs starting separator
        printf -- "--%s\r\n" "$BOUNDARY" >> "$TEMP_FILE"
    fi

    printf -- "Content-Type: application/dicom\r\n\r\n" >> "$TEMP_FILE"

    # Append DICOM file content
    cat "$dicom_file" >> "$TEMP_FILE"

    echo "Added file: $(basename "$dicom_file")"
done

# 3. Write ending separator for request body
printf -- "\r\n--%s--\r\n" "$BOUNDARY" >> "$TEMP_FILE"

# 4. Calculate file size
CONTENT_LENGTH=$(wc -c < "$TEMP_FILE" | tr -d ' ')

echo "Total content length: $CONTENT_LENGTH bytes"

# 5. Send request
curl -X POST http://localhost:9000/stow-rs/v1/studies \
     -H "Content-Type: multipart/related; boundary=$BOUNDARY; type=application/dicom" \
     -H "Accept: application/json" \
     -H "x-tenant: 1234567890" \
     -H "Content-Length: $CONTENT_LENGTH" \
     --data-binary @"$TEMP_FILE"

# 6. Clean up temporary file
rm "$TEMP_FILE"

echo "Upload completed"

```
### .NET Core Implementation

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
            // Estimate memory stream size for performance optimization
            long estimatedSize = 0;
            foreach (var filePath in dicomFilePaths)
            {
                if (!File.Exists(filePath))
                    throw new FileNotFoundException($"DICOM file not found: {filePath}");

                // Get file size and accumulate
                var fileInfo = new FileInfo(filePath);
                estimatedSize += fileInfo.Length;
            }

            // Add estimated size for boundaries and headers (approximately 200 bytes per file for separators and headers)
            estimatedSize += dicomFilePaths.Count * 200;
            // Add end boundary size
            estimatedSize += Boundary.Length + 10;

            // Create memory stream to build multipart content with estimated size initialization
            using var memoryStream = new MemoryStream((int)Math.Min(estimatedSize, int.MaxValue));

            // Build multipart content
            foreach (var filePath in dicomFilePaths)
            {
                if (!File.Exists(filePath))
                    throw new FileNotFoundException($"DICOM file not found: {filePath}");

                // Add separator and header
                var separator = Encoding.UTF8.GetBytes($"\r\n--{Boundary}\r\n");
                var header = Encoding.UTF8.GetBytes("Content-Type: application/dicom\r\n\r\n");

                if (memoryStream.Length == 0)
                {
                    // First part doesn't need leading separator
                    separator = Encoding.UTF8.GetBytes($"--{Boundary}\r\n");
                }

                await memoryStream.WriteAsync(separator, 0, separator.Length);
                await memoryStream.WriteAsync(header, 0, header.Length);

                // Read and add DICOM file content
                var fileBytes = await File.ReadAllBytesAsync(filePath);
                await memoryStream.WriteAsync(fileBytes, 0, fileBytes.Length);
            }

            // Add ending separator
            var endBoundary = Encoding.UTF8.GetBytes($"\r\n--{Boundary}--\r\n");
            await memoryStream.WriteAsync(endBoundary, 0, endBoundary.Length);

            // Prepare request content
            var content = new ByteArrayContent(memoryStream.ToArray());
            content.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue("multipart/related");
            content.Headers.ContentType.Parameters.Add(
                new System.Net.Http.Headers.NameValueHeaderValue("boundary", Boundary));
            content.Headers.ContentType.Parameters.Add(
                new System.Net.Http.Headers.NameValueHeaderValue("type", "application/dicom"));

            // Set request headers
            _httpClient.DefaultRequestHeaders.Clear();
            _httpClient.DefaultRequestHeaders.Add("Accept", "application/json");
            _httpClient.DefaultRequestHeaders.Add("x-tenant", tenantId);

            // Send request
            var response = await _httpClient.PostAsync(url, content);

            // Process response
            var responseContent = await response.Content.ReadAsStringAsync();
            Console.WriteLine($"Status Code: {response.StatusCode}");
            Console.WriteLine($"Response: {responseContent}");
        }

        // Overloaded function: Recursively find DICOM files based on directory
        public async Task SendDicomFilesAsync(string dicomDirectory, string url, string tenantId)
        {
            if (!Directory.Exists(dicomDirectory))
                throw new DirectoryNotFoundException($"DICOM directory not found: {dicomDirectory}");

            // Recursively find all files with .dcm extension
            var dicomFiles = Directory.GetFiles(dicomDirectory, "*.dcm", SearchOption.AllDirectories);

            // If no files found, throw exception
            if (dicomFiles.Length == 0)
                throw new FileNotFoundException($"No DICOM files found in directory: {dicomDirectory}");

            // Call original method to send files
            await SendDicomFilesAsync(dicomFiles.ToList(), url, tenantId);
        }

        public void Dispose()
        {
            _httpClient.Dispose();
        }
    }
}
```

### Key Implementation Points
1. Multipart/Related Format
Content-Type must be multipart/related;boundary=DICOM_BOUNDARY;type=application/dicom
Each DICOM file requires proper boundary delimiters
First file needs a starting boundary, subsequent files need separator boundaries

2. Performance Considerations
Estimate memory usage for large file uploads
Use streaming approaches for memory efficiency
Handle large files without loading everything into memory

3. Error Handling
Validate file existence before processing
Handle directory traversal properly
Provide meaningful error messages for debugging

### Use Cases
This implementation is particularly useful for:

- STOW-RS (Store Over the Web): DICOM file storage via REST API
- Batch DICOM uploads: Multiple files in a single request
- Cloud DICOM platforms: Scalable medical imaging solutions
- DICOM integration testing: API validation and testing
  
This comprehensive guide provides both CURL and .NET Core implementations for multi-related stream uploads of DICOM files, essential for building robust DICOM-WEB services and cloud-based medical imaging platforms.