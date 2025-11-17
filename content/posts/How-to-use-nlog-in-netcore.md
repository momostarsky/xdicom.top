---
title: "DOTNET CORE使用NLOG"
description: "利用NLog库來記錄Log.便于调试及查看运行日志"
date: 2025-11-16T16:17:56+08:00
keywords: "Fo-DICOM NLog"
draft: false
---

# 在 .NET 项目中使用 NLog 记录日志的好处

NLog 是一个功能强大、灵活且高性能的日志记录框架，广泛应用于 .NET（包括 .NET Framework、.NET Core 和 .NET 5/6/7/8+）应用程序中。在 .NET 项目中集成 NLog 可显著提升系统的可观测性、可维护性和调试效率。以下是使用 NLog 的主要优势说明。

---

## 一、核心优势

### 1. **高性能与低开销**
- NLog 经过高度优化，采用异步写入、缓冲和批处理机制，对应用程序性能影响极小。
- 支持异步日志（`async="true"`），避免 I/O 阻塞主线程。

### 2. **灵活的配置方式**
- 支持通过 **XML 配置文件**（如 `nlog.config`）进行外部化配置，无需重新编译代码即可调整日志行为。
- 也支持 **代码方式配置**，适用于动态场景或云原生环境。

### 3. **丰富的输出目标（Targets）**
NLog 支持将日志同时写入多种目标，包括：
- 文件（按日期/大小自动滚动）
- 控制台
- 数据库（SQL Server、PostgreSQL 等）
- Windows 事件日志
- Elasticsearch、Graylog、Seq 等日志聚合平台
- 电子邮件、HTTP 端点、消息队列等

### 4. **强大的日志过滤与路由**
- 可基于日志级别（Trace、Debug、Info、Warn、Error、Fatal）、日志来源（Logger 名称）、自定义条件进行精细路由。
- 支持“规则链”（Rules），实现复杂的日志分发逻辑。

### 5. **结构化日志支持**
- 原生支持结构化日志（Structured Logging），便于与现代日志分析工具（如 Serilog + Seq、ELK）集成。
- 使用 `${message}`、`${event-properties:item=...}` 等布局渲染器提取结构化字段。

### 6. **自动日志文件管理**
- 内置日志文件归档策略：按时间（每日/每小时）、按大小（如 10MB）自动分割。
- 可配置最大保留文件数或总大小，防止磁盘爆满。

### 7. **跨平台兼容性**
- 完全支持 .NET Standard，可在 Windows、Linux、macOS 上运行。
- 适用于 ASP.NET Core、WPF、WinForms、控制台应用、Azure Functions、MAUI 等各类 .NET 项目。

### 8. **活跃的社区与持续维护**
- 开源（MIT 许可）、文档完善、更新频繁，拥有庞大的用户社区和丰富的插件生态。

---

## 二、典型应用场景

| 场景 | NLog 优势体现 |
|------|----------------|
| **生产环境监控** | 异步写入 + 文件滚动 + 错误日志高亮，保障系统稳定 |
| **微服务调试** | 结构化日志 + TraceID 关联，便于分布式追踪 |
| **合规审计** | 日志内容加密、防篡改配置、长期归档支持 |
| **开发阶段快速定位问题** | 控制台实时输出 + 详细 Debug 信息 |

---

## 三、简单示例

### 1. 安装 NuGet 包
```bash
dotnet add package NLog
dotnet add package NLog.Web.AspNetCore  # ASP.NET Core 项目推荐
```
## 2. 创建配置文件 `nlog.config`
```xml
<?xml version="1.0" encoding="utf-8"?>

<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      internalLogLevel="Info"
      internalLogFile=".\internal-nlog.txt">

    <!-- 定义日志输出的 target -->
    <targets>
        <!-- 输出到文件，按大小滚动 -->
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
        <!-- 输出到文件，按大小滚动 -->
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
            <!-- 为不同日志级别配置不同颜色 -->
            <highlight-row condition="level == LogLevel.Debug" foregroundColor="DarkGray" />
            <highlight-row condition="level == LogLevel.Info" foregroundColor="Gray" />
            <highlight-row condition="level == LogLevel.Warn" foregroundColor="Yellow" />
            <highlight-row condition="level == LogLevel.Error" foregroundColor="Red" />
            <highlight-row condition="level == LogLevel.Fatal" foregroundColor="Red" backgroundColor="White" />

        </target>
    </targets>

    <rules>
        <!-- 规则：所有日志都输出到 logfile -->
        <logger name="*" minlevel="Info" writeTo="logfile" />
        <logger name="*" minlevel="Debug" writeTo="console" />
        <logger name="*" minlevel="Error" writeTo="errorFile" />
    </rules>
</nlog>
```

## 3. 使用NLog

```csharp

using FellowOakDicom;
using Microsoft.Extensions.Configuration;
using NLog;

namespace change_ts;

internal class Program
{
    // 获取当前类日志记录器
    private static readonly Logger Logger = LogManager.GetCurrentClassLogger();

    static async Task Main()
    {
        // 在 DicomSetupBuilder 之前添加
        System.Text.Encoding.RegisterProvider(System.Text.CodePagesEncodingProvider.Instance); 
        // 创建 CancellationTokenSource 来处理 Ctrl+C 信号
        var cancellationTokenSource = new CancellationTokenSource();
        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true; // 阻止程序立即退出
            Logger.Info("收到 Ctrl+C 信号，正在优雅关闭...");
            cancellationTokenSource.Cancel();
        };
        try
        {
            new DicomSetupBuilder()
                .RegisterServices(s =>
                    s.AddFellowOakDicom()
                        .AddTranscoderManager<FellowOakDicom.Imaging.NativeCodec.NativeTranscoderManager>())
                .SkipValidation()
                .Build();

            // 确保日志目录存在
            var logDir = Path.Combine(AppContext.BaseDirectory, "logs");
            var archivedDir = Path.Combine(logDir, "archived");

            if (!Directory.Exists(logDir))
                Directory.CreateDirectory(logDir);

            if (!Directory.Exists(archivedDir))
                Directory.CreateDirectory(archivedDir);

            Logger.Info("🚀 应用程序启动，NLog 日志配置成功！");

            // 创建配置构建器来读取 appsettings.json
            var configuration = new ConfigurationBuilder()
                .SetBasePath(AppContext.BaseDirectory)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .Build(); 

            Console.WriteLine("XXXX启动中...");
 
            Logger.Info("XXXX已启动，服务正在运行中... 按 Ctrl+C 退出");

            // 等待取消信号
            await Task.Delay(Timeout.Infinite, cancellationTokenSource.Token);
        }
        catch (OperationCanceledException)
        {
            // 正常的取消操作，不需要记录错误
            Logger.Info("应用程序正在关闭...");
        }
        catch (Exception ex)
        {
            // 记录内部异常到 NLog（如果配置了 internalLogFile 也会记录 NLog 自己的错误）
            Logger.Error(ex, "程序因异常停止");
            throw;
        }
        finally
        { 
            // 确保所有日志都写入磁盘并关闭资源
            LogManager.Shutdown();
        }
    }
}

```

运行程序时，NLog 会自动创建日志目录和文件，并记录应用程序的启动和运行信息。当程序接收到 Ctrl+C 信号时，NLog 会记录应用程序正在关闭的信息。如果程序因为异常而停止，NLog 会记录异常信息。