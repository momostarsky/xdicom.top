---
title: "How to Use NLog in .NET Core"
description: "Leverage the NLog library to record logs for easier debugging and viewing of runtime logs in .NET applications"
date: 2025-11-16T16:17:56+08:00
keywords: "NLog .NET Core logging framework structured-logging cross-platform"
draft: false
tags: ["logging", "dotnet-core", "application-monitoring", "structured-logging"]
---

## Benefits of Using NLog for Logging in .NET Projects

NLog is a powerful, flexible, and high-performance logging framework widely used in .NET applications (including .NET Framework, .NET Core, and .NET 5/6/7/8+). Integrating NLog into .NET projects can significantly enhance system observability, maintainability, and debugging efficiency. Here are the main advantages of using NLog.

---

## I. Core Advantages

### 1. **High Performance with Low Overhead**
- NLog is highly optimized, utilizing asynchronous writing, buffering, and batch processing mechanisms that have minimal impact on application performance.
- Supports asynchronous logging (`async="true"`) to prevent I/O from blocking the main thread.

### 2. **Flexible Configuration Options**
- Supports externalized configuration through **XML configuration files** (like `nlog.config`), allowing adjustments to logging behavior without recompiling code.
- Also supports **code-based configuration** for dynamic scenarios or cloud-native environments.

### 3. **Rich Output Targets**
NLog supports writing logs to multiple targets simultaneously, including:
- Files (with automatic rolling by date/size)
- Console
- Databases (SQL Server, PostgreSQL, etc.)
- Windows Event Log
- Logging aggregation platforms like Elasticsearch, Graylog, and Seq
- Email, HTTP endpoints, message queues, and more

### 4. **Powerful Log Filtering and Routing**
- Can route logs based on log level (Trace, Debug, Info, Warn, Error, Fatal), source (Logger name), and custom conditions with fine-grained control.
- Supports "rule chains" for implementing complex log distribution logic.

### 5. **Structured Logging Support**
- Native support for structured logging, making integration with modern log analysis tools (like Serilog + Seq, ELK) seamless.
- Uses layout renderers like `${message}`, `${event-properties:item=...}` to extract structured fields.

### 6. **Automatic Log File Management**
- Built-in log file archiving strategies: by time (daily/hourly) or by size (e.g., 10MB) with automatic splitting.
- Configurable maximum number of retained files or total size to prevent disk overflow.

### 7. **Cross-Platform Compatibility**
- Fully supports .NET Standard, running on Windows, Linux, and macOS.
- Suitable for various .NET projects including ASP.NET Core, WPF, WinForms, console applications, Azure Functions, and MAUI.

### 8. **Active Community and Ongoing Maintenance**
- Open source (MIT license), well-documented, frequently updated, with a large user community and rich plugin ecosystem.

---

## II. Typical Use Cases

| Scenario | NLog Advantage |
|----------|----------------|
| **Production Environment Monitoring** | Asynchronous writing + file rolling + error log highlighting ensures system stability |
| **Microservice Debugging** | Structured logging + TraceID correlation enables distributed tracing |
| **Compliance Auditing** | Log content encryption, tamper-proof configuration, and long-term archiving support |
| **Development Phase Issue Resolution** | Real-time console output + detailed debug information |

---

## III. Simple Implementation Example

### 1. Install NuGet Packages
```bash
dotnet add package NLog
dotnet add package NLog.Web.AspNetCore  # Recommended for ASP.NET Core projects
```

### 2. Create Configuration File `nlog.config`
```xml
<?xml version="1.0" encoding="utf-8"?>

<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      internalLogLevel="Info"
      internalLogFile=".\internal-nlog.txt">

    <!-- Define log output targets -->
    <targets>
        <!-- Output to file with size-based rolling -->
        <target name="logfile"
                xsi:type="File"
                fileName="logs/logfile.log"
                layout="${longdate}|${level:uppercase=true}|${logger}|${message} ${exception:format=tostring}"
                maxArchiveFiles="10"
                archiveAboveSize="5242880"
                archiveFileName="logs/archived/logfile.{#}.log"
                archiveNumbering="Sequence"
                keepFileOpen="false"
                concurrentWrites="true"
                enableFileDelete="true" />
                
        <!-- Output to file for errors with size-based rolling -->
        <target name="errorFile"
                xsi:type="File"
                fileName="logs/error.log"
                layout="${longdate}|${level:uppercase=true}|${logger}|${message} ${exception:format=tostring}"
                maxArchiveFiles="10"
                archiveAboveSize="5242880"
                archiveFileName="logs/archived/logfile.{#}.log"
                archiveNumbering="Sequence"
                keepFileOpen="false"
                concurrentWrites="true"
                enableFileDelete="true" />

        <target name="console"
                xsi:type="ColoredConsole"
                layout="${longdate}|${level:uppercase=true}|${logger}|${message} ${exception:format=tostring}">
            <!-- Configure different colors for different log levels -->
            <highlight-row condition="level == LogLevel.Debug" foregroundColor="DarkGray" />
            <highlight-row condition="level == LogLevel.Info" foregroundColor="Gray" />
            <highlight-row condition="level == LogLevel.Warn" foregroundColor="Yellow" />
            <highlight-row condition="level == LogLevel.Error" foregroundColor="Red" />
            <highlight-row condition="level == LogLevel.Fatal" foregroundColor="Red" backgroundColor="White" />

        </target>
    </targets>

    <rules>
        <!-- Rule: All logs output to logfile -->
        <logger name="*" minlevel="Info" writeTo="logfile" />
        <logger name="*" minlevel="Debug" writeTo="console" />
        <logger name="*" minlevel="Error" writeTo="errorFile" />
    </rules>
</nlog>
```

### 3. Using NLog in Your Application

```csharp
using Microsoft.Extensions.Configuration;
using NLog;

namespace YourApplication;

internal class Program
{
    // Get logger instance for current class
    private static readonly Logger Logger = LogManager.GetCurrentClassLogger();

    static async Task Main()
    {
        // Add this before DicomSetupBuilder if using DICOM
        System.Text.Encoding.RegisterProvider(System.Text.CodePagesEncodingProvider.Instance); 
        
        // Create CancellationTokenSource to handle Ctrl+C signals
        var cancellationTokenSource = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true; // Prevent immediate program exit
            Logger.Info("Ctrl+C signal received, gracefully shutting down...");
            cancellationTokenSource.Cancel();
        };
        
        try
        {
            // Register DICOM services if needed
            // new DicomSetupBuilder()
            //     .RegisterServices(s =>
            //         s.AddFellowOakDicom()
            //             .AddTranscoderManager<FellowOakDicom.Imaging.NativeCodec.NativeTranscoderManager>())
            //     .SkipValidation()
            //     .Build();

            // Ensure log directories exist
            var logDir = Path.Combine(AppContext.BaseDirectory, "logs");
            var archivedDir = Path.Combine(logDir, "archived");

            if (!Directory.Exists(logDir))
                Directory.CreateDirectory(logDir);

            if (!Directory.Exists(archivedDir))
                Directory.CreateDirectory(archivedDir);

            Logger.Info("🚀 Application started, NLog logging configured successfully!");

            // Create configuration builder to read appsettings.json
            var configuration = new ConfigurationBuilder()
                .SetBasePath(AppContext.BaseDirectory)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .Build(); 

            Console.WriteLine("Application starting...");
 
            Logger.Info("Application running... Press Ctrl+C to exit");

            // Wait for cancellation signal
            await Task.Delay(Timeout.Infinite, cancellationTokenSource.Token);
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation, no need to log as error
            Logger.Info("Application is shutting down...");
        }
        catch (Exception ex)
        {
            // Log internal exception to NLog (if internalLogFile is configured, NLog's own errors will also be recorded)
            Logger.Error(ex, "Application stopped due to exception");
            throw;
        }
        finally
        { 
            // Ensure all logs are written to disk and resources are closed
            LogManager.Shutdown();
        }
    }
}
```

When running the application, NLog will automatically create log directories and files, recording application startup and runtime information. When the program receives a Ctrl+C signal, NLog will log information about the application shutting down. If the program stops due to an exception, NLog will record the exception information.

---

## IV. Best Practices for NLog Implementation

### 1. **Performance Optimization**
- Use async targets to prevent blocking
- Configure appropriate log levels in production
- Implement log file rotation to manage disk space

### 2. **Security Considerations**
- Avoid logging sensitive information like passwords or personal data
- Implement log file access controls
- Consider log encryption for sensitive environments

### 3. **Monitoring and Alerting**
- Set up log monitoring for critical error patterns
- Implement structured logging for better searchability
- Configure appropriate log retention policies

### 4. **Development vs Production Configurations**
- Use more verbose logging in development
- Optimize for performance in production
- Different log formats and targets based on environment

By following these practices, you can effectively implement NLog in your .NET applications to create a robust logging system that supports debugging, monitoring, and operational excellence.