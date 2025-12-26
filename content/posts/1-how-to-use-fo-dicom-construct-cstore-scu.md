---
title: "How to Use fo-dicom to Build DICOM C-Store SCU Tool for Batch Sending DICOM Files"
date: 2025-11-19T11:23:56+08:00
keywords: "DICOM C-Store SCU, medical imaging, healthcare cloud, DICOM storage, fo-dicom, DICOM C-Store client, medical imaging software"
description: "Learn how to use fo-dicom to build a DICOM C-Store SCU tool for batch sending DICOM files. Complete guide for testing and validation of medical imaging systems."
draft: false
tags: ["DICOM", "medical imaging", "healthcare cloud", "DICOM storage", "fo-dicom", "C-Store SCU"]
categories: ["Medical Imaging", "DICOM Development", "Healthcare Technology"]
slug: "how-to-use-fo-dicom-build-dicom-cstore-scu-batch-send"
---

## How to Use fo-dicom to Build DICOM C-Store SCU Tool for Batch Sending DICOM Files

Building a DICOM C-Store SCU (Service Class User) tool is essential for medical imaging applications that require sending DICOM files to storage systems. This tutorial will guide you through creating a robust batch DICOM file sender using fo-dicom, a powerful open-source DICOM library written in C#.

Fo-DICOM provides comprehensive features for working with DICOM data, including support for reading, writing, and manipulating DICOM files. This guide demonstrates how to create a DICOM C-Store SCU tool capable of batch sending DICOM files for testing and verification purposes in medical imaging environments.

### Installing fo-dicom Dependencies

To begin, install the required fo-dicom packages using NuGet Package Manager in Visual Studio:

```bash
dotnet add package fo-dicom             
dotnet add package fo-dicom.Codecs
dotnet add package fo-dicom.Imaging.ImageSharp
```

These packages provide the core DICOM functionality, codec support, and imaging capabilities needed for our C-Store SCU tool.

### Setting Up Logging with NLog
First, create an NLog.config file to manage application logging:
```xml
<?xml version="1.0" encoding="utf-8"?>

<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      internalLogLevel="Info"
      internalLogFile=".\internal-nlog.txt">

    <!-- Define log output targets -->
    <targets>
        <!-- File output with size-based rolling -->
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
        <!-- Error file output -->
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
        <!-- Rules: all logs to logfile -->
        <logger name="*" minlevel="Info" writeTo="logfile" />
        <logger name="*" minlevel="Debug" writeTo="console" />
        <logger name="*" minlevel="Error" writeTo="errorFile" />
    </rules>
</nlog>
```
### Configuring Application Settings
Create an appsettings.json file to manage application configuration:
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

### Creating the DICOM Sender Class
Create a DcmSender.cs file containing the core sending logic:
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

        // Get all DICOM files in the directory
        var dicomFiles = GetDicomFiles(dir);
        _logger.Info($"Found Files Count: {dicomFiles.Count} ");

        // Create batches of files
        var batches = CreateBatches(dicomFiles, BatchSize);
        
        // Send files using multiple threads
        var tasks = new List<Task>();
        var ge = batches.GetEnumerator();
        while (ge.MoveNext())
        {
            if (CancellationToken.IsCancellationRequested)
                break;

            var batch = ge.Current;
            var task = Task.Run(async () => { await SendDicomFile(batch); }, CancellationToken);

            tasks.Add(task);

            // Control concurrent thread count
            if (tasks.Count < Threads) continue;
            await Task.WhenAny(tasks);
            tasks.RemoveAll(t => t.IsCompleted);
        }

        // Wait for all tasks to complete
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
            _logger.Error(ex, $"Error occurred when enumerating files in directory: {directory} ");
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
            _logger.Debug($"Beginning to send files: {string.Join(", ", filePath)}");
            var client = DicomClientFactory.Create(Ip, Port, false, MyAet, Aet);
            client.NegotiateAsyncOps();
            // When StoreSCP uses DICOM-rs implementation
            client.ServiceOptions.MaxPDULength = 16384;
            foreach (var file in filePath)
            {
                var dicomFile = await DicomFile.OpenAsync(file);
                var request = new DicomCStoreRequest(dicomFile);
                // You can add your own tags
                request.Command.Add(DicomVR.LO, new DicomTag(0x1211, 0x0001), "xdicom.com-dicomgate-2026");
                request.Command.Add(DicomVR.LO, new DicomTag(0x1211, 0x1217), "1234567890");
                request.OnResponseReceived += (_, args) =>
                {
                    _logger.Info($"Received response status: {args.Status}");
                };
                await client.AddRequestAsync(request);
            }

            await client.SendAsync(CancellationToken);
            _logger.Info($"Successfully sent files: {string.Join(", ", filePath)}");
        }
        catch (Exception ex)
        {
            _logger.Error(ex, $"Error occurred when sending files: {string.Join(", ", filePath)} ");
        }
    }
}
```

### Implementing the Main Program
Modify your Program.cs file to include command-line interface functionality:
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
    // Get logger for current class
    private static readonly Logger Logger = LogManager.GetCurrentClassLogger();

    private static readonly SenderDefaultOptions DefaultOptions = new SenderDefaultOptions();

    static async Task Main(string[] args)
    {
        // Register encoding provider before DicomSetupBuilder
        System.Text.Encoding.RegisterProvider(System.Text.CodePagesEncodingProvider.Instance);

        var rootCommand = new RootCommand(description: "DICOM Batch Sending Tool");
        
        Option<string> hostOption = new("--host", "-h")
        {
            Description = "Store-SCP Server IP Address.",
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
            Description = "Local AeTitle.",
            DefaultValueFactory = _ => DefaultOptions.DefaultMyAet,
        };
        Option<int> batchSizeOption = new("--batch-size")
        {
            Description = "How many files to send per connection.",
            DefaultValueFactory = _ => DefaultOptions.DefaultBatchSize,
        };
        Option<int> threadsOption = new("--threads", "-t")
        {
            Description = "Maximum number of threads to run.",
            DefaultValueFactory = _ => DefaultOptions.DefaultThreads,
        };

        Option<string> dirsOption = new("--files")
        {
            Description = "Directory containing DICOM files to send.",
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
                Console.WriteLine("\nOptions:");
                Console.WriteLine(
                    $"  --host, -h <host>         Store-SCP Server IP Address. Default: {DefaultOptions.DefaultHost}");
                Console.WriteLine(
                    $"  --port, -P <port>         Store-SCP Server Port. Default: {DefaultOptions.DefaultPort}");
                Console.WriteLine(
                    $"  --aet <aet>               Store-SCP Server AeTitle. Default: {DefaultOptions.DefaultAet}");
                Console.WriteLine($"  --myaet <myaet>           Local AeTitle. Default: {DefaultOptions.DefaultMyAet}");
                Console.WriteLine(
                    $"  --batch-size <batch-size> Files to send per connection. Default: {DefaultOptions.DefaultBatchSize}");
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
        // Create CancellationTokenSource to handle Ctrl+C signals
        var cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        Console.CancelKeyPress += (_, e)
        {
            e.Cancel = true; // Prevent immediate program exit
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

            // Ensure log directories exist
            var logDir = Path.Combine(AppContext.BaseDirectory, "logs");
            var archivedDir = Path.Combine(logDir, "archived");

            if (!Directory.Exists(logDir))
                Directory.CreateDirectory(logDir);

            if (!Directory.Exists(archivedDir))
                Directory.CreateDirectory(archivedDir);

            Logger.Info("🚀 Application Started, NLog configuration is OK！");

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
        }
        catch (OperationCanceledException)
        {
            // Normal cancellation, no need to log error
            Logger.Info("Application is shutting down...");
        }
        catch (Exception ex)
        {
            Logger.Error(ex, "Application Crashed !");
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

### Project Configuration
Here's the final .csproj file:
```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
    </PropertyGroup>
    <ItemGroup>
        <!-- Explicitly upgrade versions to resolve downgrade warnings -->
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

###  Key Features and Notes
-   The project targets .NET 8.0, as indicated by <TargetFramework>net8.0</TargetFramework> in the csproj file.
-   The project uses NLog for comprehensive logging. Install NLog and NLog.Extensions.Logging packages to use it.
-   client.ServiceOptions.MaxPDULength = 16384; sets the maximum length of the PDU (Protocol Data Unit) to 16384 bytes for optimal performance.
-   The application supports concurrent file sending with configurable thread count and batch size.
-   Command-line interface allows flexible configuration of DICOM server parameters.
###  Next Steps
After implementing this C-Store SCU client, you'll need to create a C-Store SCP (Service Class Provider) to receive DICOM files. This completes the DICOM communication cycle for medical imaging systems.

For more information about building cloud-based DICOM platforms, see our guide on how to build cloud DICOM systems.

This guide provides a complete implementation of a DICOM C-Store SCU tool using fo-dicom, suitable for testing and validating medical imaging systems. The batch sending capability makes it ideal for large-scale DICOM file transfers in healthcare environments.