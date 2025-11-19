---
title: "如何用FoDICOM构建DICOM CStoreSCU 工具,批量发送DICOM文件"
date: 2025-11-19T11:23:56+08:00
keywords: "DICOM CStoreSCU , medical imaging,healthcare cloud,DICOM storage"
description: "用FoDICOM构建DICOM CStoreSCU .用于批量发送DICOM文件用于测试和验证"
draft: false
---

### 1-Step How To Use FoDICOM To Construct DICOM CStoreSCU Tool, Batch Send DICOM Files For Testing And Verification

FoDICOM is a powerful open-source DICOM library written in C#. It provides a wide range of features for working with DICOM data, including support for reading, writing, and manipulating DICOM files. In this tutorial, we will use FoDICOM to construct a DICOM CStoreSCU tool, which can be used to batch send DICOM files for testing and verification purposes.

You can install FoDICOM using NuGet Package Manager in Visual Studio. Here is the command to install it:
 
```bash
dotnet add package fo-dicom             
dotnet add package fo-dicom.Codecs
dotnet add package fo-dicom.Imaging.ImageSharp
```


Now ,let's create a new console application. 

1. Add  NLog.config file.

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


2. Add appsettings.json or change default log level in appsettings.json.
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  }  
}
```

3. Add DcmSender.cs file.

```csharp
using FellowOakDicom;
using FellowOakDicom.Network;
using FellowOakDicom.Network.Client;
using NLog;

namespace DcmBatchSender;

public class DcmSender
{
    private readonly Logger _logger = LogManager.GetCurrentClassLogger();
    public CancellationToken CancellationToken { get; set; }
    public required string Ip { get; set; }
    public int Port { get; set; }
    public required string Aet { get; set; }
    public required string MyAet { get; set; }
    public int BatchSize { get; set; }
    public int Threads { get; set; }

    public async Task Start(string dir)
    {
        if (!Directory.Exists(dir))
        {
            _logger.Error($"Directory does not exist: {dir}");
            return;
        }

        // 获取目录下所有DICOM文件
        var dicomFiles = GetDicomFiles(dir);
        _logger.Info($"Found Files Count: {dicomFiles.Count} ");

        // 分批处理文件
        var batches = CreateBatches(dicomFiles, BatchSize);
        // 使用多线程发送文件
        var tasks = new List<Task>();
        var ge = batches.GetEnumerator();
        while (ge.MoveNext())
        {
            if (CancellationToken.IsCancellationRequested)
                break;

            var batch = ge.Current;
            var task = Task.Run(async () => { await SendDicomFile(batch); }, CancellationToken);

            tasks.Add(task);

            // 控制并发线程数
            if (tasks.Count < Threads) continue;
            await Task.WhenAny(tasks);
            tasks.RemoveAll(t => t.IsCompleted);
        }

        // 等待所有任务完成
        await Task.WhenAll(tasks);
    }

    private List<string> GetDicomFiles(string directory)
    {
        var files = new List<string>();
        try
        {
            var dicomExtensions = new[] { "*.dcm", "*.dicom" };
            foreach (var extension in dicomExtensions)
            {
                files.AddRange(Directory.GetFiles(directory, extension, SearchOption.AllDirectories));
            }
        }
        catch (Exception ex)
        {
            _logger.Error(ex, $"occurs error when enumerate files in directory: {directory} ");
        }

        return files;
    }

    private List<List<string>> CreateBatches(List<string> files, int batchSize)
    {
        var batches = new List<List<string>>();
        for (int i = 0; i < files.Count; i += batchSize)
        {
            batches.Add(files.GetRange(i, Math.Min(batchSize, files.Count - i)));
        }

        return batches;
    }

    private async Task SendDicomFile(List<string> filePath)
    {
        try
        {
            _logger.Debug($"begning send files: {filePath}");
            var client = DicomClientFactory.Create(Ip, Port, false, MyAet, Aet);
            client.NegotiateAsyncOps();
            //when StoreSCP use DICOM-rs implementation. 
            client.ServiceOptions.MaxPDULength = 16384;
            foreach (var file in filePath)
            {
                var dicomFile = await DicomFile.OpenAsync(file);
                var request = new DicomCStoreRequest(dicomFile);
                // you can add your own tag 
                request.Command.Add(DicomVR.LO, new DicomTag(0x1211, 0x0001), "xdicom.com-dicomgate-2026");
                request.Command.Add(DicomVR.LO, new DicomTag(0x1211, 0x1217), "1234567890");
                request.OnResponseReceived += (_, args) =>
                {
                    _logger.Info($"received response status: {args.Status}");
                };
                await client.AddRequestAsync(request);
            }

            await client.SendAsync(CancellationToken);
            _logger.Info($"send files success: {filePath}");
        }
        catch (Exception ex)
        {
            _logger.Error(ex, $"some error when send files: {filePath} ");
        }
    }
}
```

4. Modify Program.cs file.
```csharp
using FellowOakDicom;
using NLog;
using System.CommandLine;

namespace DcmBatchSender;

struct SenderDefaultOptions
{
    public readonly string DefaultHost = "192.168.1.14";
    public readonly string DefaultAet = "STORE-SCP";
    public readonly string DefaultMyAet = "STORE-SCU";
    public readonly int DefaultPort = 11111;
    public readonly int DefaultBatchSize = 100;
    public readonly int DefaultThreads = 10;
    public readonly int DefaultTimeout = 30;

    public readonly string DefaultDirs = "./";

    public SenderDefaultOptions()
    {
    }
}

class Program
{
    // 获取当前类日志记录器
    private static readonly Logger Logger = LogManager.GetCurrentClassLogger();

    private static readonly SenderDefaultOptions DefaultOptions = new SenderDefaultOptions();

    static async Task Main(string[] args)
    {
        // 在 DicomSetupBuilder 之前添加
        System.Text.Encoding.RegisterProvider(System.Text.CodePagesEncodingProvider.Instance);

        var rootCommand = new RootCommand(description: "DICOM Batch Sending Tool");
        
        Option<string> hostOption = new("--host", "-h")
        {
            Description = "Store-SCP Server  IP Address.",
            DefaultValueFactory = _ => DefaultOptions.DefaultHost,
        };
        Option<int> portOption = new("--port", "-P")
        {
            Description = "Remote Server Port.",
            DefaultValueFactory = _ => DefaultOptions.DefaultPort,
        };
        Option<string> aetOption = new("--aet")
        {
            Description = "Store-SCP Server AeTitle.",
            DefaultValueFactory = _ => DefaultOptions.DefaultAet,
        };
        Option<string> myOption = new("--myaet")
        {
            Description = "Store-SCP Server  IP Address.",
            DefaultValueFactory = _ => DefaultOptions.DefaultMyAet,
        };
        Option<int> batchSizeOption = new("--batch-size")
        {
            Description = "How Many Files Send Per Connection.",
            DefaultValueFactory = _ => DefaultOptions.DefaultBatchSize,
        };
        Option<int> threadsOption = new("--threads", "-t")
        {
            Description = "How Much Max Threads to Run",
            DefaultValueFactory = _ => DefaultOptions.DefaultThreads,
        };

        Option<string> dirsOption = new("--files")
        {
            Description = "Send DICOM Files Directory.",
            DefaultValueFactory = _ => DefaultOptions.DefaultDirs,
        };
        rootCommand.Add(hostOption);
        rootCommand.Add(portOption);
        rootCommand.Add(aetOption);
        rootCommand.Add(myOption);
        rootCommand.Add(batchSizeOption);
        rootCommand.Add(threadsOption);
        rootCommand.Add(dirsOption);

        rootCommand.SetAction(parseResult =>
        {
            if (parseResult.Errors.Count > 0)
            {
                Console.WriteLine("Parse Arguments:");
                foreach (var error in parseResult.Errors)
                {
                    Console.WriteLine($"  - {error.Message}");
                }

                Console.WriteLine("\nUsage:");
                Console.WriteLine("DcmBatchSender [Option]");
                Console.WriteLine("\noption:");
                Console.WriteLine(
                    $"  --host, -h <host>         Store-SCP Server IP Address. Default: {DefaultOptions.DefaultHost}");
                Console.WriteLine(
                    $"  --port, -P <port>         Store-SCP Server Port. Default: {DefaultOptions.DefaultPort}");
                Console.WriteLine(
                    $"  --aet <aet>               Store-SCP Server AeTitle. Default: {DefaultOptions.DefaultAet}");
                Console.WriteLine($"  --myaet <myaet>           Local AeTitle. Default: {DefaultOptions.DefaultMyAet}");
                Console.WriteLine(
                    $"  --batch-size <batch-size> Files Send Per Connection. Default: {DefaultOptions.DefaultBatchSize}");
                Console.WriteLine($"  --threads, -t <threads>   Max Threads. Default: {DefaultOptions.DefaultThreads}");
                Console.WriteLine(
                    $"  --files <files>           DICOM Files Directory. Default: {DefaultOptions.DefaultDirs}");

                Environment.Exit(1);
            }

            var host = parseResult.GetRequiredValue(hostOption);
            var port = parseResult.GetRequiredValue(portOption);
            var aet = parseResult.GetRequiredValue(aetOption);
            var myaet = parseResult.GetRequiredValue(myOption);
            var batchSize = parseResult.GetRequiredValue(batchSizeOption);
            var threads = parseResult.GetRequiredValue(threadsOption);
            var dirs = parseResult.GetRequiredValue(dirsOption);
            return RunApplication(CancellationToken.None, host, port, aet, myaet, batchSize, threads, dirs,
                DefaultOptions.DefaultTimeout);
        });

        ParseResult parseResult = rootCommand.Parse(args);

        await parseResult.InvokeAsync();
    }

    static async Task RunApplication(CancellationToken cancellationToken,
        string ip, int port, string aet, string myaet,
        int batchSize, int threads, string dir, int timeout)
    {
        // 创建 CancellationTokenSource 来处理 Ctrl+C 信号
        var cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        Console.CancelKeyPress += (_, e) =>
        {
            e.Cancel = true; // 阻止程序立即退出 
            Logger.Info("Received Ctrl+C signal, application will shut down gracefully...");
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

            Logger.Info("🚀 Application Started，NLog configuration is oK！");

            // 创建配置构建器来读取 appsettings.json
            // var configuration = new ConfigurationBuilder()
            //     .SetBasePath(AppContext.BaseDirectory)
            //     .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            //     .Build();

            // 在这里使用 updatedConfig 进行后续操作
            Logger.Info(
                $"Configuration: IP={ip}, Port={port}, AET={aet}, MyAET={myaet}, BatchSize={batchSize}, Threads={threads}, Timeout={timeout}");

            DcmSender dcmSender = new DcmSender()
            {
                CancellationToken = cancellationTokenSource.Token,
                Ip = ip,
                Port = port,
                Aet = aet,
                MyAet = myaet,
                BatchSize = batchSize,
                Threads = threads,
            };
            await dcmSender.Start(dir);
            // 应用程序主要逻辑应该在这里执行 
        }
        catch (OperationCanceledException)
        {
            // 正常的取消操作，不需要记录错误
            Logger.Info("Application is shutting down...");
        }
        catch (Exception ex)
        {
            // 记录内部异常到 NLog（如果配置了 internalLogFile 也会记录 NLog 自己的错误）
            Logger.Error(ex, "Application Crashed !");
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

5. The final csproj file:
```The csproj file:
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>
    <ItemGroup>
        <!-- 显式提升版本，解决降级警告 -->
        <PackageReference Include="StackExchange.Redis" Version="2.9.32" />
        <PackageReference Include="System.CommandLine" Version="2.0.0-rc.1.25451.107" />
        <PackageReference Include="System.Net.NameResolution" Version="4.3.0" />
        <PackageReference Include="System.Net.Primitives" Version="4.3.1" /> 
        <PackageReference Include="System.Text.Encoding.CodePages" Version="9.0.10" /> 
        <PackageReference Include="fo-dicom" Version="5.2.5" />
        <PackageReference Include="fo-dicom.Codecs" Version="5.16.4" />
        <PackageReference Include="fo-dicom.Imaging.ImageSharp" Version="5.2.4" />
        <PackageReference Include="NLog" Version="6.0.5" />
        <PackageReference Include="NLog.Extensions.Logging" Version="6.0.5" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="9.0.10" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="9.0.10" />
    </ItemGroup>
    <ItemGroup>
      <None Update="appsettings.json">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      </None>
      <None Update="NLog.config">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      </None>
    </ItemGroup> 
</Project> 
```


Note: 
-    The project target is net 8.0 ,you can find    <TargetFramework>net8.0</TargetFramework> in the csproj file.
-    The project uses NLog for logging. You need to install NLog and NLog.Extensions.Logging packages to use it.
-    client.ServiceOptions.MaxPDULength = 16384;  This line sets the maximum length of the PDU (Protocol Data Unit) to 16384 bytes.
  
OK, Next  Step is to create a CStoreSCP Provider to receive DICOM files from the CStoreSCP Client.

GoTo  Summary  : [how-to-build-cloud-dicom](/posts/how-to-build-cloud-dicom)