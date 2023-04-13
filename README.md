# Batch-Process-Automation
This is a repository where settings, examples, and configuration will be added. We created a dotnet/,Net set of applications for discovering and executing processes in an unopinionated way.

As always, check out our website www.inforhino.co.uk to find out more about our work and projects.

## Processor and Executor Processor automation overview
Automating processing of applications and processes - orchestration is very complicated, highly semantic, and challenging when trying to reach consensus on the right way forward. Furthermore, maintaining it can be very complicated.  From experience of consulting to enterprises - orchestration is the last thing developers care about is orchestrating their processes. At the same time, developers present blackboxes to testers, DevOps, and Ops/Operations and don't like it when their carefully laid plans gets ruined on. 

We must remember that whilst there are some great dotnet/dotnetcore process automation tools, they are few. Furthermore, they are opinionated and often require writing code to add jobs and processes. Our current version is still opinionated in some respects but it avoids having to invest heavily in batch automation software. If you are viewing this readme online, it won't contain client specific configuration, we will set up a test version.

### Executor Processor
This is an application that reads configuration and generates, discovers artifacts to create the necessary artifacts that a processor instance can discover and generate artifacts for the processor to run its processes. We run the Executor Processor, typically through a generate.bat we define to create the artifacts for the Processor.

### Processor
The Processor is relatively simple code that interprets execution information created by the Executor Processor to set about running applications. It tries to detect success but we feel that it would be better to have independent applications verify success as in "Promise Theory" than to keep adding more to this lightweight application.

We can either run, a processor, a scheduled processor, or a file watcher processor.

## The technology stack
- SQL Server database for storing configuration
- Dotnet Framework Executor Procecssor application for generating configuration
- Dotnet Framework Processor application for running processes

### Future plans
- Move this to dotnet core
- Either remove SQL Server configuration or add file based configuration (Note, we can avoid licensing issues as we have shared SQL Server hosting or can use the basic version of SQL Server but some clients may not have this luxury
- Add a front-end to manage the configuration
- Add a front-end and scanner for checking processes and statuses
- Not to add too much complexity to this solution, there are plenty of more weighty batch automation solutions such as control-M, autosys, Dollar Universe
- Add more Processor types
## Headliners, thoughts, and caveats
Note - it should be the case that the SQL server database is not needed for execution, simply containing the configuration. We "Generate" configuration.

## Basic tables and setup

### Running the following sql statements
```sql
SELECT * FROM [job].[BatchRoot] for json auto
SELECT * FROM [job].[Executable] for json auto 
SELECT * FROM [job].[ExecutableCommandlineParameter] for json auto
SELECT * FROM [job].[ExecutableGroup] for json auto
SELECT * FROM [job].[SimpleProcess] for json auto
SELECT CONVERT(nvarchar(max) ,(SELECT * FROM [job].[ExecutableGroupProcess] for json auto)) data
```
### Returns the following data

(Not publishing this on a public website)

### Explanations and relationships

Whilst maybe not immediately obvious, running this sql will show the data from their tables.
```sql 
SELECT * FROM [job].[BatchRoot]
SELECT * FROM [job].[Executable] 
SELECT * FROM [job].[ExecutableCommandlineParameter] 
SELECT * FROM [job].[ExecutableGroup] 
SELECT * FROM [job].[SimpleProcess] 
SELECT * FROM [job].[ExecutableGroupProcess]
```
Shows this relationship 
[ExecutableCommandlineParameter]  < [Executable] <  [ExecutableGroup] 
> [ExecutableGroupProcess] > [ExecutableGroup]

## Some ideas on executing batch files or executables
### Batch files
You may decide to put the arguments for an executable all inside a batch file. This partially depends upon who is managing this process. Perhaps your "operations/ops" teams prefer to maintain this in batch files, perhaps developers like to see these common configurations making them part of the CI process?

### Set up the Executable and commandline parameters inside the database
Executable and Executable Command line Parameter may be preferable as it is centrally configured inside the database and has greater visibility than batch files. Alternatively, you may find it cumbersome to maintain inside the database. Rest assured, we have flexibility.

### Executable Group Process, Parallel execution, executable discovery
One very important principles of the Executable Group Process it to set the max threads setting. This will move towards async parallel processes, but importantly we can allow processes to be batched together and ran in parallel. More importantly is the *Number of Children Per Group setting*, The Executable Processor application can create batches of discovered executables. This allows us to create lots of executables and allow the executor to discover the binaries saving on configuration time. This potentially allows for auto scaling if controlled externally.

## Typical Setup Process

 - Define an Executable Group, this can be used one or more times
 - Define your Executables, whether they are wildcards for discovering and grouping, or whether they have parameters or not
 - Manage the Executable Group Process, this helps define the shape of execution, note - we can only have one parallel set of executions per matched sets of executable group/executable combination. i.e. if we wanted to find all .bat files in a folder and group them we could not do this within the same executable group. We may need a separate executable group.

## Application Configuration 

### Sample config for a screenscraping application

### 
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <startup> 
        <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6.1" />
    </startup>
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
      <dependentAssembly>
        <assemblyIdentity name="Newtonsoft.Json" publicKeyToken="30ad4fe6b2a6aeed" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-11.0.0.0" newVersion="11.0.0.0" />
      </dependentAssembly>
      <dependentAssembly>
        <assemblyIdentity name="System.ComponentModel.Annotations" publicKeyToken="b03f5f7f11d50a3a" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-4.1.1.0" newVersion="4.1.1.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
  <appSettings>
    
    <add key="LocationOfApplicationExecutionDefinition" value="C:\InfoRhino\Test\Processor\TESTSIMPLE\ApplicationExecution.json" />
    <!--<add key="LocationOfApplicationExecutionDefinition" value="C:\ScreenScraper\Processing\ApplicationExecution.json" />-->
  <!--Note, we only place a file in the same location IF we do indeed want to run a specific type of processor-->
    <add key="FileWatcherConfig" value="FileWatcherConfig.json" /> <!--Flename that will be in the same folder as LocationOfApplicationExecutionDefinition-->
    <add key="TimedEventConfig" value="TimedEventConfig.json" /><!--Flename that will be in the same folder as LocationOfApplicationExecutionDefinition-->
  </appSettings>
  <connectionStrings>
    <add name="JobRepoConnection" connectionString="Data Source=someserver\somedb;Database=somedatabase;Trusted_Connection=True;Connection Timeout=0;" />
  </connectionStrings>
</configuration>
 ```

### Generating the ApplicationExecution.json
By setting up the database, most of the processing uses generated file based  json. The intention is to configure a database, and to then generate the settings which the processor can use.

## Process Modes within the Processor
As mentioned, there are scheduling tools in .Net. However, as we already have the Processor it isn't too hard to add a basic schedule based processor and a file watcher based processor. Here are some basic points on how to set this up.

### Main app settings within the Processor app
We need to point the exe.config to a valid location of a ApplicationExecution.json file.
That folder path is used to search for the possible existence of a FileWatcherConfig or a TimedEventConfig. From the absence/existence of these files, the application will either run a;

 - ProcessorFileWatcher
 - ProcessorTimedEvents
 - Processor (Default original version)
 
```xml
  <appSettings>
    <add key="LocationOfApplicationExecutionDefinition" value="C:\InfoRhino\Test\Processor\TESTSIMPLE\ApplicationExecution.json" />
    <!--Note, we only place a file in the same location IF we do indeed want to run a specific type of processor-->
    <add key="FileWatcherConfig" value="FileWatcherConfig.json" /> <!--Flename that will be in the same folder as LocationOfApplicationExecutionDefinition-->
    <add key="TimedEventConfig" value="TimedEventConfig.json" /><!--Flename that will be in the same folder as LocationOfApplicationExecutionDefinition-->
  </appSettings>
```

### Configuration classes and json
We expect either no json files in the "LocationOfApplicationExecutionDefinition" location or a single file.

#### TimedEventConfig
```csharp
    public class TimedEventConfig
    {

        public TimeLength timeLength { get; set; }
        public TimedInstruction every { get; set; }
        public TimedInstruction on { get; set; }
        public int secondsDelay { get; set; }
    }
    public class TimeLength
    {
        [JsonConverter(typeof(StringEnumConverter))]
        public Period Period { get; set; }

        public int Length { get; set; }
    }
    
    public enum Period
    {
        Years,
        Months,
        Weeks,
        Days,
        DayOfMonth,
        Hours,
        Minutes,
        Seconds
    }
    public class TimedInstruction
    {
        [JsonConverter(typeof(StringEnumConverter))]
        public Period Period { get; set; }
        public int[] When { get; set; }
    }
```

#### FileWatcherConfig
```csharp
    public class FileWatcherConfig
    {

        public string FolderPath { get; set; }

        public string FilePattern { get; set; }
        /// <summary>
        /// Any files found matching
        /// </summary>
        public string ProcessedFileExtension { get; set; }

        public TimeLength DurationLimit { get; set; }
        public TimeLength Delay { get; set; }
        //public int SecondsDelay { get; set; }
    }
    public class TimeLength
    {
        [JsonConverter(typeof(StringEnumConverter))]
        public Period Period { get; set; }

        public int Length { get; set; }
    }

    public enum Period
    {
        Years,
        Months,
        Weeks,
        Days,
        DayOfMonth,
        Hours,
        Minutes,
        Seconds
    }
```

> Written with [StackEdit](https://stackedit.io/).
