## Demystifying Logging


```C#
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args)
    {
        Host.CreateDefaultBuilder(args)
           .ConfigureLogging((hostBuilderContext, loggingBuilder) =>
           {
               loggingBuilder.Configure(options => options.ActivityTrackingOptions = ActivityTrackingOptions.TraceId | ActivityTrackingOptions.SpanId);  // <------scolog
               loggingBuilder.AddConsole(options =>
               {
                   options.IncludeScopes = true;   // <----------fis
               });
           })
           .ConfigureWebHostDefaults(webBuilder =>
           {
               webBuilder.UseStartup<Startup>();
           });
    }
       
}
```

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "CarvedRock": "Debug"
    },
    "Console": {
      "FormatterName": "json",
      "FormatterOptions": {
        "SingleLine": true,
        "IncludeScopes": true,  // <----------------------fis
        "TimestampFormat": "HH:mm:ss ",
        "UseUtcTimestamp": true,
        "JsonWriterOptions": {
          "Indented": true
        }
      },
      "AllowedHosts": "*"
    }
  }
}
```

Instrumentation is code that is added to a software project to record what it is doing. This information can then be collected in files, databases, or in-memory and analyzed to understand how a software program is operating.


```C#
//!!!!!!!!!!!!!!!!!!!!trace id, span id, parent id, request id, correlation id---------------------------------------------------------------------to work on

public class HomeController : Controller
{
   private readonly ILogger<HomeController> _logger;

   public HomeController(ILogger<HomeController> logger)
   {
      _logger = logger;
     
   }

   public HomeController(ILoggerFactory factory)
   {
       // Unless you're using heavily customized categories for some reason, favor injecting ILogger<T> over ILoggerFactory 
       _logger = factory.CreateLogger("DemystifyingLogging.NotCalledHomeController");  // for heavily customized categories
       //_logger = factory.CreateLogger<HomeController>(); // same as inject ILogger<HomeController>
   }

   public IActionResult Index()
   {
      return View();
   }

   public IActionResult Privacy()
   {
      return View();
   }

   [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
   public IActionResult Error()
   {
      return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });  // <---------------------
   }
}

/*
Request ID: 00-c659e64609ac37959252c779655da6d3-5213a5e1fbfcea4a-00
HTTP TraceId: 0HMNGJOQ3VBIF:00000009
Activity.Id: 00-c659e64609ac37959252c779655da6d3-5213a5e1fbfcea4a-00
Activity.SpanId: 5213a5e1fbfcea4a
Activity.TraceId: c659e64609ac37959252c779655da6d3
Activity.Parent:
Activity.ParentId:
Activity.ParentSpanId: 0000000000000000
Activity.RootId: c659e64609ac37959252c779655da6d3
Activity.Kind: Internal
*/
```




Below are buildt-in logger provider:

1. `ConsoleLoggerProvider` (`ConsoleLogger`) : writes messages to the console
2. `DebugLoggerProvider` (`DebugLogger`) : writes messages to the debug window when debugging an app in VS
3. `EventLogLoggerProvider` (`EventLogLogger`) : Windows-only as it requires Windows-specific APIs
4. `EventSourceLoggerProvider` (`EventSourceLogger`) : writes messages using Event Tracing for Windows (ETW) or LTTng tracing on Linux

Note that .NET Core doesn't provder "FileLogger", which you have to use a third-party libary, in fact you can associate a file to the trace listener to write logs on files as

```C#
using System.Diagnostics;

var path = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
var tracePath = Path.Join(path, $"Log_Boc_{DateTime.Now.ToString("yyyyMMdd-HHmm")}.txt");
Trace.Listeners.Add(new TextWriterTraceListener(System.IO.File.CreateText(tracePath)));
Trace.AutoFlush = true;
```

```C#
//------------------V
public class Program {
   public static void Main(string[] args) {
      CreateHostBuilder(args).Build().Run();
   }

   public static IHostBuilder CreateHostBuilder(string[] args) =>
       Host.CreateDefaultBuilder(args)
           .ConfigureWebHostDefaults(webBuilder => {
              webBuilder.UseStartup<Startup>();
           })
           .ConfigureLogging(logger =>  // logger is ILoggingBuilder
           {       
              logger.AddConsole(  // <-----------------------------------------1
                 options => // options is ConsoleLoggerOptions
                 {   
                    options.IncludeScopes = true;
                 }
               ).AddFilter("Boc.Domain", LogLevel.Debug);  // you can also configure the logging category/level via code, not just from appsetting.json
           });
}
//------------------Ʌ

//-------------------------V example
public class HomeController {

   public HomeController(ILogger<MyService> logger) {  // <------------------------4_
      // ...
   }
   
   public void MethodToBeCalled()
   {
      _logger.LogTrace("Loaded Trace");
      _logger.LogDebug("Loaded Debug");
      _logger.LogInformation(1001, "Loaded Information {param}", "MyService");  // 1001 is the Event ID
      // _logger.LogInformation("Loaded Information MyService");  // don't do this, which is not structure loggin
      _logger.LogWarning("Loaded Warning");
      _logger.LogError("Loaded Error");
      _logger.LogCritical("Loaded Critical");
      
      //-------------------------------------->>
      using(_logger.BeginScope("Scope value"))
      using(_logger.BeginScope(new Dictionary<string, object> { { "CustomValue1", 12345 } }))
      {
         _logger.LogWarning("Yes, I have the scope!");   
         _logger.LogWarning("again, I have the scope!");
      }

      _logger.LogWarning("No, I lost it again");
      //--------------------------------------<< logging output as below
      /* you have to set IncludeScopes to true to be able to log and see ConnectionId, SpanId, TraceId, ParentId etc
      warn: DemystifyingLogging.Controllers.HomeController[0]
            => ConnectionId:0HN1LUP1092ME => RequestPath:/ RequestId:0HN1LUP1092ME:00000001, SpanId:|8909bd52-4f55d689b02d8489., TraceId:8909bd52-4f55d689b02d8489, ParentId: => DemystifyingLogging.Controllers.HomeController.Index (DemystifyingLogging) => Scope value => System.Collections.Generic.Dictionary`2[System.String,System.Object]
            
            Yes, I have the scope!
      warn: DemystifyingLogging.Controllers.HomeController[0]
            => ConnectionId:0HN1LUP1092ME => RequestPath:/ RequestId:0HN1LUP1092ME:00000001, SpanId:|8909bd52-4f55d689b02d8489., TraceId:8909bd52-4f55d689b02d8489, ParentId: => 
            DemystifyingLogging.Controllers.HomeController.Index (DemystifyingLogging) => Scope value => System.Collections.Generic.Dictionary`2[System.String,System.Object]
            
            again, I have the scope!
      warn: DemystifyingLogging.Controllers.HomeController[0]
            => ConnectionId:0HN1LUP1092ME => RequestPath:/ RequestId:0HN1LUP1092ME:00000001, SpanId:|8909bd52-4f55d689b02d8489., TraceId:8909bd52-4f55d689b02d8489, ParentId: => DemystifyingLogging.Controllers.HomeController.Index (DemystifyingLogging)
            
            No, I lost it again
      
      also check https://nblumhardt.com/2016/11/ilogger-beginscope/
      */
   }
}
//-------------------------Ʌ 

// example
public class EmployeeRepository : IEmployeeRepository
{
   private readonly ILogger<EmployeeRepository> _logger;
   private readonly ILogger _factoryLogger;


   public EmployeeRepository(ILogger<EmployeeRepository> logger, ILoggerFactory loggerFactory)
   {
      _logger = logger;  // category name is Boc.Data.EmployeeRepository (namespace name + class name)
      _factoryLogger = loggerFactory.CreateLogger("DataAccessLayer");  // specify a custom category name
   }
}

```

Quick dependencies simplified code:

```C#
//--------------------------------------------------->>
public interface ILogger<TCategoryName> : ILogger { }

public interface ILogger
{
   void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter);
   
   bool IsEnabled(LogLevel logLevel);

   IDisposable BeginScope<TState>(TState state) ;
}
//---------------------------------------------------<<

//---------------------------------V
public class Logger<T> : ILogger<T>  // <-----------------------------so inject `Logger<T>` is just an indirect call of injecting `ILoggerFactory`
{
   private readonly ILogger _logger;

   public Logger(ILoggerFactory factory) 
   {   
      _logger = factory.CreateLogger(TypeNameHelper.GetTypeDisplayName(typeof(T), includeGenericParameters: false, nestedTypeDelimiter: '.')); 
   }

   // ...
}
//---------------------------------Ʌ

//-----------------------------------------V
public class LoggerFactory : ILoggerFactory   // contains `ILoggerProvider`
{
   private readonly Dictionary<string, Logger> _loggers = new Dictionary<string, Logger>(StringComparer.Ordinal);

   public LoggerFactory(IEnumerable<ILoggerProvider> providers, ...)
   {
      foreach (ILoggerProvider provider in providers)
      {
         AddProviderRegistration(provider, dispose: false);
      }
   }

   private void AddProviderRegistration(ILoggerProvider provider, bool dispose) 
   {
      _providerRegistrations.Add(new ProviderRegistration
      {
         Provider = provider,   // <------------
         ShouldDispose = dispose
      });
 
      // ...
   }

   public ILogger CreateLogger(string categoryName)
   {
      if (!_loggers.TryGetValue(categoryName, out Logger? logger))  //  only a single Logger instance is needed for a specific category
      {
         logger = new Logger(CreateLoggers(categoryName));   // <----------------------------------------------create a Logger instance
         (logger.MessageLoggers, logger.ScopeLoggers) = ApplyFilters(logger.Loggers);
         _loggers[categoryName] = logger;
      }

       return logger;
   }

   private LoggerInformation[] CreateLoggers(string categoryName)  
   {
      var loggers = new LoggerInformation[_providerRegistrations.Count];
      for (int i = 0; i < _providerRegistrations.Count; i++)
      {
         loggers[i] = new LoggerInformation(_providerRegistrations[i].Provider,   // <-----------pass ILoggerProvider to LoggerInformation
                                            categoryName);
      }
      return loggers;
   }
}
//-----------------------------------------Ʌ

//----------------------------------------V
internal readonly struct LoggerInformation  // has dependency of ILoggerProvider
{
   public LoggerInformation(ILoggerProvider provider, string category) : this()
   {
      ProviderType = provider.GetType();
      Logger = provider.CreateLogger(category);  // <--------------use ILoggerProvider to create an ILogger, e.g ConsoleLoggerProvider which calls `new ConsoleLogger(...)`
      Category = category;
   }
 
   public ILogger Logger { get; }   // <----------------create concrete ILogger, e.g `ConsoleLogger`
 
   public string Category { get; }
 
   public Type ProviderType { get; }
}
//----------------------------------------Ʌ

//--------------------------V
internal sealed class Logger : ILogger   // Logger is a wrapper so it contains multiple ILoggers via `LoggerInformation`
{
   public Logger(LoggerInformation[] loggers) 
   {
      Loggers = loggers;
   } 

   public LoggerInformation[] Loggers { get; set; }   // contains multiple ILoggers, such as `ConsoleLogger`, `DebugLogger` etc

   public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter)
   {
      // ...
      for (int i = 0; i < loggers.Length; i++)   // that's why muliple sinks can be logged in once
      {
         LoggerLog(logLevel,
                   eventId,
                   loggerInfo.Logger,  // <---------------
                   ...
                  );
      }
   }

   static void LoggerLog(LogLevel logLevel, EventId eventId, ILogger logger, ...)
   {
      logger.Log(logLevel, eventId, ...);
   }
}
//--------------------------Ʌ
```


=========================================================================================================================================


## Source Code

```C#
//----------------------------------------------------V
public static class LoggingServiceCollectionExtensions
{
   public static IServiceCollection AddLogging(this IServiceCollection services)
   {
      return AddLogging(services, builder => { });
   }

   public static IServiceCollection AddLogging(this IServiceCollection services, Action<ILoggingBuilder> configure)
   {
      services.AddOptions();
                                                                                          // note it uses `TryAdd`, that's how you can register your own Serilog later
      services.TryAdd(ServiceDescriptor.Singleton<ILoggerFactory, LoggerFactory>());      // <-------------------------
      services.TryAdd(ServiceDescriptor.Singleton(typeof(ILogger<>), typeof(Logger<>)));  // <-------------------------
 
      services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<LoggerFilterOptions>>(new DefaultLoggerLevelConfigureOptions(LogLevel.Information)));
 
      configure(new LoggingBuilder(services));
      return services;
   }
}
//----------------------------------------------------Ʌ

//------------------------------------------V
public static class LoggingBuilderExtensions
{
   public static ILoggingBuilder SetMinimumLevel(this ILoggingBuilder builder, LogLevel level)
   {
      builder.Services.Add(ServiceDescriptor.Singleton<IConfigureOptions<LoggerFilterOptions>>(new DefaultLoggerLevelConfigureOptions(level)));
      return builder;
   }

   public static ILoggingBuilder AddProvider(this ILoggingBuilder builder, ILoggerProvider provider)
   {
      builder.Services.AddSingleton(provider);
      return builder;
   }

   public static ILoggingBuilder ClearProviders(this ILoggingBuilder builder)
   {
      builder.Services.RemoveAll<ILoggerProvider>();
      return builder;
   }

   public static ILoggingBuilder Configure(this ILoggingBuilder builder, Action<LoggerFactoryOptions> action)
   {
      builder.Services.Configure(action);
      return builder;
   }
}
//------------------------------------------Ʌ

//-------------------------------------------------V
public static partial class ConsoleLoggerExtensions
{
    public static ILoggingBuilder AddConsole(this ILoggingBuilder builder)
    {
        builder.AddConfiguration();
 
        builder.AddConsoleFormatter<JsonConsoleFormatter, JsonConsoleFormatterOptions, ConsoleFormatterConfigureOptions>();
        builder.AddConsoleFormatter<SystemdConsoleFormatter, ConsoleFormatterOptions, ConsoleFormatterConfigureOptions>();
        builder.AddConsoleFormatter<SimpleConsoleFormatter, SimpleConsoleFormatterOptions, ConsoleFormatterConfigureOptions>();
 
        builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ILoggerProvider, ConsoleLoggerProvider>());
 
        builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<ConsoleLoggerOptions>, ConsoleLoggerConfigureOptions>());
        builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IOptionsChangeTokenSource<ConsoleLoggerOptions>, LoggerProviderOptionsChangeTokenSource<ConsoleLoggerOptions, ConsoleLoggerProvider>>());
 
        return builder;
    }

    public static ILoggingBuilder AddConsole(this ILoggingBuilder builder, Action<ConsoleLoggerOptions> configure)
    { 
        builder.AddConsole();
        builder.Services.Configure(configure);
 
        return builder;
    }
}
//-------------------------------------------------Ʌ

//-------------------------------------------------V
public static partial class ConsoleLoggerExtensions
{
   public static ILoggingBuilder AddConsole(this ILoggingBuilder builder, Action<ConsoleLoggerOptions> configure)  // <----------------------------2
   {
      builder.AddConsole();   // <-------------------3
      builder.Services.Configure(configure);
 
      return builder;
   }
   
   public static ILoggingBuilder AddConsole(this ILoggingBuilder builder)
   {
      builder.AddConfiguration();  // <-----------------------------3.1, register LoggerProviderConfigurationFactory and LoggerProviderConfiguration

      builder.AddConsoleFormatter<JsonConsoleFormatter, JsonConsoleFormatterOptions>();   
      builder.AddConsoleFormatter<SystemdConsoleFormatter, ConsoleFormatterOptions>();
      builder.AddConsoleFormatter<SimpleConsoleFormatter, SimpleConsoleFormatterOptions>();
                                                                                              // <-------------------3.2

      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ILoggerProvider, ConsoleLoggerProvider>());   // <----------------3.3!
                                                                                                                  // notice the usage of TryAddEnumerable  
      LoggerProviderOptions
         .RegisterProviderOptions<ConsoleLoggerOptions, ConsoleLoggerProvider>(builder.Services);  // <--------------------------------------------3.4->, register below
                                                                                                   // LoggerProviderConfigureOptions<ConsoleLoggerOptions, ConsoleLoggerProvider>
      return builder;
   }

   public static ILoggingBuilder AddSimpleConsole(this ILoggingBuilder builder)
   {
      return builder.AddFormatterWithName(ConsoleFormatterNames.Simple);
   }

   public static ILoggingBuilder AddJsonConsole(this ILoggingBuilder builder)
   {
      return builder.AddFormatterWithName(ConsoleFormatterNames.Json);
   }
            
   public static ILoggingBuilder AddJsonConsole(this ILoggingBuilder builder, Action<JsonConsoleFormatterOptions> configure)
   {
      return builder.AddConsoleWithFormatter<JsonConsoleFormatterOptions>(ConsoleFormatterNames.Json, configure);
   } 

   public static ILoggingBuilder AddSystemdConsole(this ILoggingBuilder builder, Action<ConsoleFormatterOptions> configure)
   {
      return builder.AddConsoleWithFormatter<ConsoleFormatterOptions>(ConsoleFormatterNames.Systemd, configure);
   }    

   internal static ILoggingBuilder AddConsoleWithFormatter<TOptions>(this ILoggingBuilder builder, string name, Action<TOptions> configure)
   {
      builder.AddFormatterWithName(name);
      builder.Services.Configure(configure);
 
      return builder;
   }

   private static ILoggingBuilder AddFormatterWithName(this ILoggingBuilder builder, string name)
   {
      return builder.AddConsole((ConsoleLoggerOptions options) => options.FormatterName = name);
   }

   public static ILoggingBuilder AddConsoleFormatter<TFormatter, TOptions>(this ILoggingBuilder builder)  // <-----------------------3
   {
      builder.AddConfiguration();
 
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ConsoleFormatter, TFormatter>());
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<TOptions>, ConsoleLoggerFormatterConfigureOptions<TFormatter, TOptions>>());
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IOptionsChangeTokenSource<TOptions>, ConsoleLoggerFormatterOptionsChangeTokenSource<TFormatter, TOptions>>());
 
      return builder;
   }

   public static ILoggingBuilder AddConsoleFormatter<TFormatter, TOptions>(this ILoggingBuilder builder, Action<TOptions> configure)
   { 
      builder.AddConsoleFormatter<TFormatter, TOptions>();
      builder.Services.Configure(configure);
      return builder;
   }
}
//-------------------------------------------------Ʌ

//----------------------------------------------V
public static class DebugLoggerFactoryExtensions
{
   public static ILoggingBuilder AddDebug(this ILoggingBuilder builder)
   {
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ILoggerProvider, DebugLoggerProvider>());
      return builder;
   }
}
//----------------------------------------------Ʌ

public static class LoggerProviderOptions
{
   public static void RegisterProviderOptions<TOptions, TProvider>(IServiceCollection services)
   {
      services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<TOptions>, LoggerProviderConfigureOptions<TOptions, TProvider>>());  // <---------3.4.1_
      services.TryAddEnumerable(ServiceDescriptor.Singleton<IOptionsChangeTokenSource<TOptions>, LoggerProviderOptionsChangeTokenSource<TOptions, TProvider>>());
   }  
}

//----------------------------------------------V
public static class EventLoggerFactoryExtensions
{
   public static ILoggingBuilder AddEventLog(this ILoggingBuilder builder)
   { 
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ILoggerProvider, EventLogLoggerProvider>());
 
      return builder;
   }

   public static ILoggingBuilder AddEventLog(this ILoggingBuilder builder, EventLogSettings settings)
   {
      builder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<ILoggerProvider>(new EventLogLoggerProvider(settings)));
 
      return builder;
   }

   public static ILoggingBuilder AddEventLog(this ILoggingBuilder builder, Action<EventLogSettings> configure)
   {
      builder.AddEventLog();
      builder.Services.Configure(configure);
 
      return builder;
   }
}

//--------------------------->>>
public class EventLogSettings
{
   private IEventLog? _eventLog;
   public string? LogName { get; set; }
   public string? SourceName { get; set; }
   public string? MachineName { get; set; }
   public Func<string, LogLevel, bool>? Filter { get; set; }

   internal IEventLog EventLog
   {
      get => _eventLog ??= CreateDefaultEventLog();
 
      // for unit testing purposes only.
      set => _eventLog = value;
   }

   private IEventLog CreateDefaultEventLog()
   {
      string logName = string.IsNullOrEmpty(LogName) ? "Application" : LogName;
      string machineName = string.IsNullOrEmpty(MachineName) ? "." : MachineName;
      string sourceName = string.IsNullOrEmpty(SourceName) ? ".NET Runtime" : SourceName;
      int? defaultEventId = null;
 
      if (string.IsNullOrEmpty(SourceName))
      {
         sourceName = ".NET Runtime";
         defaultEventId = 1000;
      }
 
      return new WindowsEventLog(logName, machineName, sourceName) { DefaultEventId = defaultEventId };
   }
}
//---------------------------<<<
//----------------------------------------------Ʌ

//---------------------------------->>
public class ConsoleFormatterOptions
{
   public ConsoleFormatterOptions() { }
   public bool IncludeScopes { get; set; }
   public string? TimestampFormat { get; set; }
   public bool UseUtcTimestamp { get; set; }
}
//----------------------------------<<

//-------------------------------------->>
public class JsonConsoleFormatterOptions : ConsoleFormatterOptions
{
   public JsonConsoleFormatterOptions() { }
   public JsonWriterOptions JsonWriterOptions { get; set; }
}
//--------------------------------------<<

//----------------------------->>
public struct JsonWriterOptions
{
   internal const int DefaultMaxDepth = 1000;
 
   private int _maxDepth;
   private int _optionsMask;
   public JavaScriptEncoder? Encoder { get; set; }

   public bool Indented
   {
      get
      {
         return (_optionsMask & IndentBit) != 0;
      }
      set
      {
         if (value)
            _optionsMask |= IndentBit;
         else
            _optionsMask &= ~IndentBit;
      }
   }

   public int MaxDepth { get; set; }

   public bool SkipValidation
   {
      get
      {
         return (_optionsMask & SkipValidationBit) != 0;
      }
      set
      {
         if (value)
            _optionsMask |= SkipValidationBit;
         else
            _optionsMask &= ~SkipValidationBit;
      }
   }

   internal bool IndentedOrNotSkipValidation => _optionsMask != SkipValidationBit; // Equivalent to: Indented || !SkipValidation;
 
   private const int IndentBit = 1;
   private const int SkipValidationBit = 2;
}
//-----------------------------<<

//------------------------------------------------------V
public static class LoggingBuilderConfigurationExtensions
{
   public static void AddConfiguration(this ILoggingBuilder builder)   // <-----------------------2
   {
      builder.Services.TryAddSingleton<ILoggerProviderConfigurationFactory, LoggerProviderConfigurationFactory>();
      builder.Services.TryAddSingleton(typeof(ILoggerProviderConfiguration<>), typeof(LoggerProviderConfiguration<>));
   }
}
//------------------------------------------------------Ʌ

//-------------------------------------------------->>
public interface ILoggerProviderConfigurationFactory 
{
   IConfiguration GetConfiguration(Type providerType);
}
//--------------------------------------------------<<

//------------------------------------------------------V
internal sealed class LoggerProviderConfigurationFactory : ILoggerProviderConfigurationFactory  
{
   private readonly IEnumerable<LoggingConfiguration> _configurations;
 
   public LoggerProviderConfigurationFactory(IEnumerable<LoggingConfiguration> configurations)
   {
      _configurations = configurations;
   }

   public IConfiguration GetConfiguration(Type providerType)
   {
      string fullName = providerType.FullName!;
      string? alias = ProviderAliasUtilities.GetAlias(providerType);
      ConfigurationBuilder configurationBuilder = new ConfigurationBuilder();

      foreach (LoggingConfiguration configuration in _configurations)
      {
         IConfigurationSection sectionFromFullName = configuration.Configuration.GetSection(fullName);
         configurationBuilder.AddConfiguration(sectionFromFullName);
 
         if (!string.IsNullOrWhiteSpace(alias))
         {
            IConfigurationSection sectionFromAlias = configuration.Configuration.GetSection(alias);
            configurationBuilder.AddConfiguration(sectionFromAlias);
         }
      }
      return configurationBuilder.Build();
   }
}
//------------------------------------------------------Ʌ

//---------------------------------------------->>
public interface ILoggerProviderConfiguration<T>
{
   IConfiguration Configuration { get; }
}
//----------------------------------------------<<

//--------------------------------------------------V
internal sealed class LoggerProviderConfiguration<T> : ILoggerProviderConfiguration<T>
{
   public LoggerProviderConfiguration(ILoggerProviderConfigurationFactory providerConfigurationFactory)
   {
      Configuration = providerConfigurationFactory.GetConfiguration(typeof(T));
   }
 
   public IConfiguration Configuration { get; }
}
//--------------------------------------------------Ʌ

//------------------------------------>>
public interface ISupportExternalScope
{
   void SetScopeProvider(IExternalScopeProvider scopeProvider);
}
//------------------------------------<<

//------------------------------------->>
public interface IExternalScopeProvider
{
   void ForEachScope<TState>(Action<object?, TState> callback, TState state);
   IDisposable Push(object? state);
}
//-------------------------------------<<

//------------------------------>>
public interface ILoggerProvider : IDisposable
{
   ILogger CreateLogger(string categoryName);
}
//------------------------------<<

//--------------------------------V
public class ConsoleLoggerProvider : ILoggerProvider, ISupportExternalScope   // ConsoleLoggerProvider implements ISupportExternalScope
{
   private readonly IOptionsMonitor<ConsoleLoggerOptions> _options;
   private readonly ConcurrentDictionary<string, ConsoleLogger> _loggers;
   private ConcurrentDictionary<string, ConsoleFormatter> _formatters;
   private readonly ConsoleLoggerProcessor _messageQueue;

   private IDisposable? _optionsReloadToken;
   private IExternalScopeProvider _scopeProvider = NullExternalScopeProvider.Instance;

   public ConsoleLoggerProvider(IOptionsMonitor<ConsoleLoggerOptions> options) : this(options, Array.Empty<ConsoleFormatter>()) { }

   public ConsoleLoggerProvider(IOptionsMonitor<ConsoleLoggerOptions> options, IEnumerable<ConsoleFormatter>? formatters)
   {
       _options = options;
      _loggers = new ConcurrentDictionary<string, ConsoleLogger>();
      SetFormatters(formatters);
      IConsole? console;
      IConsole? errorConsole;
      if (DoesConsoleSupportAnsi())
      {
         console = new AnsiLogConsole();
         errorConsole = new AnsiLogConsole(stdErr: true);
      }
      else
      {
         console = new AnsiParsingLogConsole();
         errorConsole = new AnsiParsingLogConsole(stdErr: true);
      }
      _messageQueue = new ConsoleLoggerProcessor(console, errorConsole, options.CurrentValue.QueueFullMode, options.CurrentValue.MaxQueueLength);
 
      ReloadLoggerOptions(options.CurrentValue);
      _optionsReloadToken = _options.OnChange(ReloadLoggerOptions);
   }

   private static bool DoesConsoleSupportAnsi() { ... }

   private void SetFormatters(IEnumerable<ConsoleFormatter>? formatters = null)
   {
      var cd = new ConcurrentDictionary<string, ConsoleFormatter>(StringComparer.OrdinalIgnoreCase);
 
      bool added = false;
      if (formatters != null)
      {
         foreach (ConsoleFormatter formatter in formatters)
         {
            cd.TryAdd(formatter.Name, formatter);
            added = true;
         }
      }
 
      if (!added)
      {
         cd.TryAdd(ConsoleFormatterNames.Simple, new SimpleConsoleFormatter(new FormatterOptionsMonitor<SimpleConsoleFormatterOptions>(new SimpleConsoleFormatterOptions())));
         cd.TryAdd(ConsoleFormatterNames.Systemd, new SystemdConsoleFormatter(new FormatterOptionsMonitor<ConsoleFormatterOptions>(new ConsoleFormatterOptions())));
         cd.TryAdd(ConsoleFormatterNames.Json, new JsonConsoleFormatter(new FormatterOptionsMonitor<JsonConsoleFormatterOptions>(new JsonConsoleFormatterOptions())));
      }
 
      _formatters = cd;
   }

   private void ReloadLoggerOptions(ConsoleLoggerOptions options)
   {
      if (options.FormatterName == null || !_formatters.TryGetValue(options.FormatterName, out ConsoleFormatter? logFormatter))
      {
         logFormatter = options.Format switch
         {
            ConsoleLoggerFormat.Systemd => _formatters[ConsoleFormatterNames.Systemd],
            _ => _formatters[ConsoleFormatterNames.Simple],
         };
         if (options.FormatterName == null)
         {
            UpdateFormatterOptions(logFormatter, options);
         }
      }

      _messageQueue.FullMode = options.QueueFullMode;
      _messageQueue.MaxQueueLength = options.MaxQueueLength;
 
      foreach (KeyValuePair<string, ConsoleLogger> logger in _loggers)
      {
         logger.Value.Options = options;
         logger.Value.Formatter = logFormatter;
      }
   }

   public ILogger CreateLogger(string name)
   {
      if (_options.CurrentValue.FormatterName == null || !_formatters.TryGetValue(_options.CurrentValue.FormatterName, out ConsoleFormatter? logFormatter))
      {
         logFormatter = _options.CurrentValue.Format switch
         {
            ConsoleLoggerFormat.Systemd => _formatters[ConsoleFormatterNames.Systemd],
            _ => _formatters[ConsoleFormatterNames.Simple],
         };

         if (_options.CurrentValue.FormatterName == null)
         {
            UpdateFormatterOptions(logFormatter, _options.CurrentValue);
         }
      }

      return _loggers.TryGetValue(name, out ConsoleLogger? logger) ? logger : 
         _loggers.GetOrAdd(name, new ConsoleLogger(name, _messageQueue, logFormatter, _scopeProvider, _options.CurrentValue));
   }

   private static void UpdateFormatterOptions(ConsoleFormatter formatter, ConsoleLoggerOptions deprecatedFromOptions)
   {
      // kept for deprecated apis:
      if (formatter is SimpleConsoleFormatter defaultFormatter)
      {
         defaultFormatter.FormatterOptions = new SimpleConsoleFormatterOptions()
         {
            ColorBehavior = deprecatedFromOptions.DisableColors ? LoggerColorBehavior.Disabled : LoggerColorBehavior.Default,
            IncludeScopes = deprecatedFromOptions.IncludeScopes,
            TimestampFormat = deprecatedFromOptions.TimestampFormat,
            UseUtcTimestamp = deprecatedFromOptions.UseUtcTimestamp,
         };
      }
      else if (formatter is SystemdConsoleFormatter systemdFormatter)
      {
         systemdFormatter.FormatterOptions = new ConsoleFormatterOptions()
         {
            IncludeScopes = deprecatedFromOptions.IncludeScopes,
            TimestampFormat = deprecatedFromOptions.TimestampFormat,
            UseUtcTimestamp = deprecatedFromOptions.UseUtcTimestamp,
         };
      }
   }

   public void Dispose()
   {
      _optionsReloadToken?.Dispose();
      _messageQueue.Dispose();
   }

   public void SetScopeProvider(IExternalScopeProvider scopeProvider)
   {
      _scopeProvider = scopeProvider;
      foreach (System.Collections.Generic.KeyValuePair<string, ConsoleLogger> logger in _loggers)
      {
         logger.Value.ScopeProvider = _scopeProvider;
      }
   }
}
//--------------------------------Ʌ

//-----------------------------V
internal sealed class NullScope : IDisposable
{
   public static NullScope Instance { get; } = new NullScope();
   private NullScope() { }
   public void Dispose() { }
}
//-----------------------------Ʌ

//---------------------------------------------V
internal sealed class NullExternalScopeProvider : IExternalScopeProvider
{
   private NullExternalScopeProvider() { }

   public static IExternalScopeProvider Instance { get; } = new NullExternalScopeProvider();

   void IExternalScopeProvider.ForEachScope<TState>(Action<object?, TState> callback, TState state) 
   { 

   }

   IDisposable IExternalScopeProvider.Push(object? state)
   {
      return NullScope.Instance;
   }
}
//---------------------------------------------Ʌ

//-------------------------------------V
public interface IExternalScopeProvider
{
   void ForEachScope<TState>(Action<object?, TState> callback, TState state);
   IDisposable Push(object? state);
}
//-------------------------------------Ʌ

//----------------------------------------------V
internal sealed class LoggerFactoryScopeProvider : IExternalScopeProvider
{
    private readonly AsyncLocal<Scope?> _currentScope = new AsyncLocal<Scope?>();
    private readonly ActivityTrackingOptions _activityTrackingOption;

    public LoggerFactoryScopeProvider(ActivityTrackingOptions activityTrackingOption) => _activityTrackingOption = activityTrackingOption;

    public IDisposable Push(object? state)
    {
        Scope? parent = _currentScope.Value;
        var newScope = new Scope(this, state, parent);
        _currentScope.Value = newScope;
 
        return newScope;
    }

    public void ForEachScope<TState>(Action<object?, TState> callback, TState state)
    {
        void Report(Scope? current)
        {
            if (current == null)
            {
                return;
            }
            Report(current.Parent);
            callback(current.State, state);
        }

        if (_activityTrackingOption != ActivityTrackingOptions.None)  // <----------------------scolog
        {
            Activity? activity = Activity.Current;   // <----------------------scolog
            if (activity != null)
            {
                const string propertyKey = "__ActivityLogScope__";

                ActivityLogScope? activityLogScope = activity.GetCustomProperty(propertyKey) as ActivityLogScope;
                if (activityLogScope == null)
                {
                    activityLogScope = new ActivityLogScope(activity, _activityTrackingOption);   // <----------------------scolog
                    activity.SetCustomProperty(propertyKey, activityLogScope);
                }

                callback(activityLogScope, state);  // <----------------------scolog

                // Tags and baggage are opt-in and thus we assume that most of the time it will not be used.
                if ((_activityTrackingOption & ActivityTrackingOptions.Tags) != 0 && activity.TagObjects.GetEnumerator().MoveNext())
                {
                    // As TagObjects is a IEnumerable<KeyValuePair<string, object?>> this can be used directly as a scope.
                    // We do this to safe the allocation of a wrapper object.
                    callback(activity.TagObjects, state);
                }

                if ((_activityTrackingOption & ActivityTrackingOptions.Baggage) != 0)
                {
                    // Only access activity.Baggage as every call leads to an allocation
                    IEnumerable<KeyValuePair<string, string?>> baggage = activity.Baggage;
                    if (baggage.GetEnumerator().MoveNext())
                    {
                        // For the baggage a wrapper object is necessary because we need to be able to overwrite ToString().
                        // In contrast to the TagsObject, Baggage doesn't have one underlining type where we can do this overwrite.
                        ActivityBaggageLogScopeWrapper scope = GetOrCreateActivityBaggageLogScopeWrapper(activity, baggage);
                        callback(scope, state);
                    }
                }
            }
        }

        Report(_currentScope.Value);
    }

    private sealed class Scope : IDisposable
    {
        private readonly LoggerFactoryScopeProvider _provider;
        private bool _isDisposed;

        internal Scope(LoggerFactoryScopeProvider provider, object? state, Scope? parent)
        {
            _provider = provider;
            State = state;
            Parent = parent;
        }

        public Scope? Parent { get; }

        public object? State { get; }

        public override string? ToString()
        {
                return State?.ToString();
        }

        public void Dispose()  // <-----------------------no need to understand, just get the idea that need to call Dispose on Scope
        {
            if (!_isDisposed)
            {
                _provider._currentScope.Value = Parent;
                _isDisposed = true;
            }
        }
    }

    private sealed class ActivityLogScope : IReadOnlyList<KeyValuePair<string, object?>>
    {
        private string? _cachedToString;
        private const int MaxItems = 5;
        private readonly KeyValuePair<string, object?>[] _items = new KeyValuePair<string, object?>[MaxItems];

        public ActivityLogScope(Activity activity, ActivityTrackingOptions activityTrackingOption)
        {
            int count = 0;
            if ((activityTrackingOption & ActivityTrackingOptions.SpanId) != 0)
            {
                _items[count++] = new KeyValuePair<string, object?>("SpanId", activity.GetSpanId());  // <-----------------scolog
            }

            if ((activityTrackingOption & ActivityTrackingOptions.TraceId) != 0)
            {
                _items[count++] = new KeyValuePair<string, object?>("TraceId", activity.GetTraceId());  // <-----------------scolog
            }

            if ((activityTrackingOption & ActivityTrackingOptions.ParentId) != 0)
            {
                _items[count++] = new KeyValuePair<string, object?>("ParentId", activity.GetParentId());  // <-----------------scolog
            }

            if ((activityTrackingOption & ActivityTrackingOptions.TraceState) != 0)
            {
                _items[count++] = new KeyValuePair<string, object?>("TraceState", activity.TraceStateString);  // <-----------------scolog
            }

            if ((activityTrackingOption & ActivityTrackingOptions.TraceFlags) != 0)
            {
                _items[count++] = new KeyValuePair<string, object?>("TraceFlags", activity.ActivityTraceFlags);  // <-----------------scolog
            }

            Count = count;
        }

        public int Count { get; }

        public KeyValuePair<string, object?> this[int index]
        {
            get
            {
                if (index >= Count)
                {
                    throw new ArgumentOutOfRangeException(nameof(index));
                }

                return _items[index];
            }
        }
     
        public IEnumerator<KeyValuePair<string, object?>> GetEnumerator()
        {
            for (int i = 0; i < Count; ++i)
            {
                yield return this[i];
            }
        }

        IEnumerator IEnumerable.GetEnumerator()
        {
            return GetEnumerator();
        }
    }
    // ...

    internal static class ActivityExtensions
    {
        public static string GetSpanId(this Activity activity)
        {
            return activity.IdFormat switch
            {
                ActivityIdFormat.Hierarchical => activity.Id,
                ActivityIdFormat.W3C => activity.SpanId.ToHexString(),
                _ => null,
            } ?? string.Empty;
        }
 
        public static string GetTraceId(this Activity activity)
        {
            return activity.IdFormat switch
            {
                ActivityIdFormat.Hierarchical => activity.RootId,
                ActivityIdFormat.W3C => activity.TraceId.ToHexString(),
                _ => null,
            } ?? string.Empty;
        }
 
        public static string GetParentId(this Activity activity)
        {
            return activity.IdFormat switch
            {
                ActivityIdFormat.Hierarchical => activity.ParentId,
                ActivityIdFormat.W3C => activity.ParentSpanId.ToHexString(),
                _ => null,
            } ?? string.Empty;
        }
    }
}
//----------------------------------------------Ʌ

//-------------------------------------V
public readonly struct LogEntry<TState>
{
   public LogEntry(LogLevel logLevel, string category, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter)
   {
      LogLevel = logLevel;
      Category = category;
      EventId = eventId;
      State = state;
      Exception = exception;
      Formatter = formatter;
   }

   public LogLevel LogLevel { get; }
   public string Category { get; }
   public EventId EventId { get; }
   public TState State { get; }
   public Exception? Exception { get; }
   public Func<TState, Exception?, string> Formatter { get; }
}
//-------------------------------------Ʌ

//------------------------------------V
public abstract class ConsoleFormatter
{
   protected ConsoleFormatter(string name)
   {
      Name = name;
   }

   public string Name { get; }

   public abstract void Write<TState>(in LogEntry<TState> logEntry, IExternalScopeProvider? scopeProvider, TextWriter textWriter);
}
//------------------------------------Ʌ

//------------------------------------------V this is the default formatter being used when calling AddLogging()
internal sealed class SimpleConsoleFormatter : ConsoleFormatter, IDisposable
{
    private const string LoglevelPadding = ": ";
    private static readonly string _messagePadding = new string(' ', GetLogLevelString(LogLevel.Information).Length + LoglevelPadding.Length);
    private static readonly string _newLineWithMessagePadding = Environment.NewLine + _messagePadding;
    private readonly IDisposable? _optionsReloadToken;

    public SimpleConsoleFormatter(IOptionsMonitor<SimpleConsoleFormatterOptions> options) : base(ConsoleFormatterNames.Simple)
    {
        ReloadLoggerOptions(options.CurrentValue);
        _optionsReloadToken = options.OnChange(ReloadLoggerOptions);
    }

    // ...

    public override void Write<TState>(in LogEntry<TState> logEntry, IExternalScopeProvider? scopeProvider, TextWriter textWriter)
    {
        string message = logEntry.Formatter(logEntry.State, logEntry.Exception);
        if (logEntry.Exception == null && message == null)
        {
            return;
        }
        LogLevel logLevel = logEntry.LogLevel;
        ConsoleColors logLevelColors = GetLogLevelConsoleColors(logLevel);
        string logLevelString = GetLogLevelString(logLevel);
 
        string? timestamp = null;
        string? timestampFormat = FormatterOptions.TimestampFormat;
        if (timestampFormat != null)
        {
            DateTimeOffset dateTimeOffset = GetCurrentDateTime();
            timestamp = dateTimeOffset.ToString(timestampFormat);
        }
        if (timestamp != null)
        {
            textWriter.Write(timestamp);
        }
        if (logLevelString != null)
        {
            textWriter.WriteColoredMessage(logLevelString, logLevelColors.Background, logLevelColors.Foreground);
        }
        CreateDefaultLogMessage(textWriter, logEntry, message, scopeProvider);
    }

    private void CreateDefaultLogMessage<TState>(TextWriter textWriter, in LogEntry<TState> logEntry, string message, IExternalScopeProvider? scopeProvider)
    {
        bool singleLine = FormatterOptions.SingleLine;
        int eventId = logEntry.EventId.Id;
        Exception? exception = logEntry.Exception;
 
        // Example:
        // info: ConsoleApp.Program[10]
        //       Request received
 
        // category and event id
        textWriter.Write(LoglevelPadding);
        textWriter.Write(logEntry.Category);
        textWriter.Write('[');
        // ...
        WriteScopeInformation(textWriter, scopeProvider, singleLine);
        WriteMessage(textWriter, message, singleLine);
        // ...
    }

    private void WriteScopeInformation(TextWriter textWriter, IExternalScopeProvider? scopeProvider, bool singleLine)  // IExternalScopeProvider is LoggerFactoryScopeProvider
    {
        if (FormatterOptions.IncludeScopes && scopeProvider != null)  // <-----------------fis, that's how IncludeScopes is checked
        {
            bool paddingNeeded = !singleLine;
            scopeProvider.ForEachScope((scope, state) =>  // <--------------------------------------scolog
            {
                if (paddingNeeded)
                {
                    paddingNeeded = false;
                    state.Write(_messagePadding);
                    state.Write("=> ");  // <----------------------scobol, now you know why scope in the console print "=>",
                }
                else
                {
                    state.Write(" => ");
                }
                state.Write(scope);  // <--------------------------------------scolog, state is textWriter, scope is ActivityLogScope
            }, textWriter);
 
            if (!paddingNeeded && !singleLine)
            {
                textWriter.Write(Environment.NewLine);
            }
        }
    }

    private static void WriteMessage(TextWriter textWriter, string message, bool singleLine)
    {
        if (!string.IsNullOrEmpty(message))
        {
            if (singleLine)
            {
                textWriter.Write(' ');
                WriteReplacing(textWriter, Environment.NewLine, " ", message);
            }
            else
            {
                textWriter.Write(_messagePadding);
                WriteReplacing(textWriter, Environment.NewLine, _newLineWithMessagePadding, message);
                textWriter.Write(Environment.NewLine);
            }
        }
 
        static void WriteReplacing(TextWriter writer, string oldValue, string newValue, string message)
        {
            string newMessage = message.Replace(oldValue, newValue);
            writer.Write(newMessage);
        }
    }

} 
//------------------------------------------Ʌ

//----------------------------------------V
internal sealed class JsonConsoleFormatter : ConsoleFormatter, IDisposable
{
   private IDisposable? _optionsReloadToken;
 
   public JsonConsoleFormatter(IOptionsMonitor<JsonConsoleFormatterOptions> options) : base(ConsoleFormatterNames.Json)
   {
      ReloadLoggerOptions(options.CurrentValue);
      _optionsReloadToken = options.OnChange(ReloadLoggerOptions);
   }

   public override void Write<TState>(in LogEntry<TState> logEntry, IExternalScopeProvider? scopeProvider, TextWriter textWriter)
   {
      string message = logEntry.Formatter(logEntry.State, logEntry.Exception);

      if (logEntry.Exception == null && message == null)
         return;

      LogLevel logLevel = logEntry.LogLevel;
      string category = logEntry.Category;
      int eventId = logEntry.EventId.Id;
      Exception? exception = logEntry.Exception;
      const int DefaultBufferSize = 1024;

      using (var output = new PooledByteBufferWriter(DefaultBufferSize))
      {
         using (var writer = new Utf8JsonWriter(output, FormatterOptions.JsonWriterOptions))
         {
            writer.WriteStartObject();
            var timestampFormat = FormatterOptions.TimestampFormat;
            if (timestampFormat != null)
            {
               DateTimeOffset dateTimeOffset = FormatterOptions.UseUtcTimestamp ? DateTimeOffset.UtcNow : DateTimeOffset.Now;
               writer.WriteString("Timestamp", dateTimeOffset.ToString(timestampFormat));
            }
            writer.WriteNumber(nameof(logEntry.EventId), eventId);
            writer.WriteString(nameof(logEntry.LogLevel), GetLogLevelString(logLevel));
            writer.WriteString(nameof(logEntry.Category), category);
            writer.WriteString("Message", message);
 
            if (exception != null)
            {
               string exceptionMessage = exception.ToString();
               if (!FormatterOptions.JsonWriterOptions.Indented)
               {
                  exceptionMessage = exceptionMessage.Replace(Environment.NewLine, " ");
               }
               writer.WriteString(nameof(Exception), exceptionMessage);
            }

            if (logEntry.State != null)
            {
               writer.WriteStartObject(nameof(logEntry.State));
               writer.WriteString("Message", logEntry.State.ToString());
               if (logEntry.State is IReadOnlyCollection<KeyValuePair<string, object>> stateProperties)
               {
                  foreach (KeyValuePair<string, object> item in stateProperties)
                  {
                     WriteItem(writer, item);
                  }
               }
               writer.WriteEndObject();
            }
            WriteScopeInformation(writer, scopeProvider);
            writer.WriteEndObject();
            writer.Flush();
         }

         textWriter.Write(Encoding.UTF8.GetString(output.WrittenMemory.Span));
      }

      textWriter.Write(Environment.NewLine);
   }

   private static string GetLogLevelString(LogLevel logLevel)
   {
      return logLevel switch
      {
         LogLevel.Trace => "Trace",
         LogLevel.Debug => "Debug",
         LogLevel.Information => "Information",
         LogLevel.Warning => "Warning",
         LogLevel.Error => "Error",
         LogLevel.Critical => "Critical",
         _ => throw new ArgumentOutOfRangeException(nameof(logLevel))
      };
   }

   private void WriteScopeInformation(Utf8JsonWriter writer, IExternalScopeProvider? scopeProvider)
   {
      if (FormatterOptions.IncludeScopes && scopeProvider != null)
      {
         writer.WriteStartArray("Scopes");
         scopeProvider.ForEachScope((scope, state) =>
         {
            if (scope is IEnumerable<KeyValuePair<string, object>> scopeItems)
            {
               state.WriteStartObject();
               state.WriteString("Message", scope.ToString());
               foreach (KeyValuePair<string, object> item in scopeItems)
               {
                  WriteItem(state, item);
               }
               state.WriteEndObject();
            }
            else
            {
               state.WriteStringValue(ToInvariantString(scope));
            }
         }, writer);
         writer.WriteEndArray();
      }
   }

   private static void WriteItem(Utf8JsonWriter writer, KeyValuePair<string, object> item)
   {
      var key = item.Key;
      switch (item.Value)
      {
         case bool boolValue:
            writer.WriteBoolean(key, boolValue);
            break;
         case byte byteValue:
            writer.WriteNumber(key, byteValue);
            break;
         case sbyte sbyteValue:
            writer.WriteNumber(key, sbyteValue);
            break;
         case char charValue:
            writer.WriteString(key, charValue.ToString());
            break;
         case int intValue:
            writer.WriteNumber(key, intValue);
            break;
         ...
         case null:
            writer.WriteNull(key);
            break;
         default:
            writer.WriteString(key, ToInvariantString(item.Value));
            break;
      }
   }

   private static string? ToInvariantString(object? obj) => Convert.ToString(obj, CultureInfo.InvariantCulture);
 
   internal JsonConsoleFormatterOptions FormatterOptions { get; set; }
 
   private void ReloadLoggerOptions(JsonConsoleFormatterOptions options)
   {
      FormatterOptions = options;
   }
 
   public void Dispose()
   {
      _optionsReloadToken?.Dispose();
   }
}
//----------------------------------------Ʌ
```


```C#
//------------------V
public enum LogLevel 
{

   Trace = 0,        // logs that contain the most detailed messages. These messages may contain sensitive application data
                     // these messages are disabled by default and should never be enabled in a production environment

   Debug = 1,        // logs that are used for interactive investigation during development
                     // These logs should primarily contain information useful for debugging and have no long-term value

   Information = 2,  // logs that track the general flow of the application. These logs should have long-term value
   
   Warning = 3,      // logs that highlight an abnormal or unexpected event in the application flow, but do not otherwise cause the application to stop

   Error = 4,        // logs that highlight when the current flow of execution is stopped due to a failure 
                     // these should indicate a failure in the current activity, not an application-wide failure

   Critical = 5,     // logs that describe an unrecoverable application or system crash, or a catastrophic failure that requires immediate attention

   None = 6          // not used for writing log messages. Specifies that a logging category should not write any message
}
//------------------Ʌ

//----------------------------------V
public static class LoggerExtensions
{
   private static readonly Func<FormattedLogValues, Exception?, string> _messageFormatter = MessageFormatter;

   public static void LogDebug(this ILogger logger, EventId eventId, string? message, params object?[] args)
   {
      logger.Log(LogLevel.Debug, eventId, message, args);
   }

   public static void LogDebug(this ILogger logger, EventId eventId, Exception? exception, string? message, params object?[] args)
   {
      logger.Log(LogLevel.Debug, eventId, exception, message, args);
   }
     
   // ...

   public static void Log(this ILogger logger, LogLevel logLevel, EventId eventId, Exception? exception, string? message, params object?[] args)
   {
      logger.Log(logLevel, eventId, new FormattedLogValues(message, args), exception, _messageFormatter);
   }

   public static IDisposable? BeginScope(this ILogger logger, string messageFormat, params object?[] args)
   {
      return logger.BeginScope(new FormattedLogValues(messageFormat, args));
   }

   private static string MessageFormatter(FormattedLogValues state, Exception? error)
   {
      return state.ToString();
   }
}
//----------------------------------Ʌ

//------------------------------>>
public interface ILoggingBuilder   // an interface for configuring logging providers
{
   IServiceCollection Services { get; }
}
//------------------------------<<

//----------------------------------V
internal sealed class LoggingBuilder : ILoggingBuilder
{
   public LoggingBuilder(IServiceCollection services)
   {
      Services = services;
   }
 
   public IServiceCollection Services { get; }
}
//----------------------------------Ʌ

//-------------------------------V
public class ConsoleLoggerOptions
{
   internal const int DefaultMaxQueueLengthValue = 2500;
   private int _maxQueuedMessages = DefaultMaxQueueLengthValue;

   public bool DisableColors { get; set; }
   
   private ConsoleLoggerFormat _format = ConsoleLoggerFormat.Default;
   
   public ConsoleLoggerFormat Format
   {
      get => _format;
      set
      {
         if (value < ConsoleLoggerFormat.Default || value > ConsoleLoggerFormat.Systemd)
         {
            throw new ArgumentOutOfRangeException(nameof(value));
         }
         _format = value;
      }
   }

   public string? FormatterName { get; set; }
   
   public bool IncludeScopes { get; set; }

   public LogLevel LogToStandardErrorThreshold { get; set; } = LogLevel.None;

   public string? TimestampFormat { get; set; }

   public bool UseUtcTimestamp { get; set; }

   private ConsoleLoggerQueueFullMode _queueFullMode = ConsoleLoggerQueueFullMode.Wait;

   public ConsoleLoggerQueueFullMode QueueFullMode
   {
      get => _queueFullMode;
      set
      {
         if (value != ConsoleLoggerQueueFullMode.Wait && value != ConsoleLoggerQueueFullMode.DropWrite)
         {
            throw new ArgumentOutOfRangeException(SR.Format(SR.QueueModeNotSupported, nameof(value)));
         }
         _queueFullMode = value;
      }
   }

   public int MaxQueueLength
   {
      get => _maxQueuedMessages;
      set
      {
         if (value <= 0)
         {
            throw new ArgumentOutOfRangeException(SR.Format(SR.MaxQueueLengthBadValue, nameof(value)));
         }
 
         _maxQueuedMessages = value;
      }
   }
}
//--------------------------------Ʌ

//----------------------V
public interface ILogger 
{
   void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter);

   bool IsEnabled(LogLevel logLevel);
 
   IDisposable BeginScope<TState>(TState state);
}
//----------------------Ʌ

//----------------------------V
public readonly struct EventId : IEquatable<EventId>
{
   public EventId(int id, string? name = null)
   {
      Id = id;
      Name = name;
   }

   public int Id { get; }

   public string? Name { get; }

   public static implicit operator EventId(int i)
   {
      return new EventId(i);
   }

   public static bool operator ==(EventId left, EventId right)
   {
      return left.Equals(right);
   }

   // ...
}
//----------------------------Ʌ

//------------------------------>>
public class LoggerFilterOptions
{
   public LoggerFilterOptions() { }
   public bool CaptureScopes { get; set; } = true;
   public LogLevel MinLevel { get; set; }
   public IList<LoggerFilterRule> Rules => RulesInternal;
   internal List<LoggerFilterRule> RulesInternal { get; } = new List<LoggerFilterRule>();
}
//------------------------------<<

//--------------------------->>
public class LoggerFilterRule
{
   public LoggerFilterRule(string? providerName, string? categoryName, LogLevel? logLevel, Func<string?, string?, LogLevel, bool>? filter)
   {
      ProviderName = providerName;
      CategoryName = categoryName;
      LogLevel = logLevel;
      Filter = filter;
   }

   public string? ProviderName { get; }
   public string? CategoryName { get; }
   public LogLevel? LogLevel { get; }
   public Func<string?, string?, LogLevel, bool>? Filter { get; }

   public override string ToString()
   {
      return $"{nameof(ProviderName)}: '{ProviderName}', {nameof(CategoryName)}: '{CategoryName}', {nameof(LogLevel)}: '{LogLevel}', {nameof(Filter)}: '{Filter}'";
   }
}
//---------------------------<<

//----------------------------------------->>
public static class LoggerFactoryExtensions
{
   public static ILogger<T> CreateLogger<T>(this ILoggerFactory factory)
   {
      return new Logger<T>(factory);
   }

   public static ILogger CreateLogger(this ILoggerFactory factory, Type type)
   {
      return factory.CreateLogger(TypeNameHelper.GetTypeDisplayName(type, includeGenericParameters: false, nestedTypeDelimiter: '.'));
   }
}
//-----------------------------------------<<

//-----------------------------V
public interface ILoggerFactory : IDisposable
{
   ILogger CreateLogger(string categoryName);
   void AddProvider(ILoggerProvider provider);
}
//-----------------------------Ʌ

//-------------------------------V
public class LoggerFactoryOptions
{
   public LoggerFactoryOptions() { }

   public ActivityTrackingOptions ActivityTrackingOptions { get; set; }
}
//-------------------------------Ʌ

//---------------------------------V
[Flags]
public enum ActivityTrackingOptions
{
   None        = 0x0000,
   SpanId      = 0x0001,
   TraceId     = 0x0002,
   ParentId    = 0x0004,
   TraceState  = 0x0008,
   TraceFlags  = 0x0010,
   Tags        = 0x0020,
   Baggage     = 0x0040
}
//---------------------------------Ʌ

//------------------------V
public class LoggerFactory : ILoggerFactory   // <-------------------------5.0
{
   private readonly Dictionary<string, Logger> _loggers = new Dictionary<string, Logger>(StringComparer.Ordinal);
   private readonly List<ProviderRegistration> _providerRegistrations = new List<ProviderRegistration>();
   private readonly object _sync = new object();
   private volatile bool _disposed;
   private IDisposable? _changeTokenRegistration;
   private LoggerFilterOptions _filterOptions;
   private IExternalScopeProvider? _scopeProvider;
   private LoggerFactoryOptions _factoryOptions;

   public LoggerFactory() : this(Array.Empty<ILoggerProvider>()) { }
   public LoggerFactory(IEnumerable<ILoggerProvider> providers) : this(providers, new StaticFilterOptionsMonitor(new LoggerFilterOptions())) { }
   public LoggerFactory(IEnumerable<ILoggerProvider> providers, LoggerFilterOptions filterOptions) : this(providers, new StaticFilterOptionsMonitor(filterOptions)) { }
   // ...

   public LoggerFactory(IEnumerable<ILoggerProvider> providers, IOptionsMonitor<LoggerFilterOptions> filterOption, IOptions<LoggerFactoryOptions>? options = null, IExternalScopeProvider? scopeProvider = null)
   {
      _scopeProvider = scopeProvider;
 
      _factoryOptions = options == null || options.Value == null ? new LoggerFactoryOptions() : options.Value;
 
      const ActivityTrackingOptions ActivityTrackingOptionsMask = ~(ActivityTrackingOptions.SpanId | ActivityTrackingOptions.TraceId | ActivityTrackingOptions.ParentId |
                                                                          ActivityTrackingOptions.TraceFlags | ActivityTrackingOptions.TraceState | ActivityTrackingOptions.Tags
                                                                          | ActivityTrackingOptions.Baggage);
 
 
      if ((_factoryOptions.ActivityTrackingOptions & ActivityTrackingOptionsMask) != 0)
      {
         throw new ArgumentException(SR.Format(SR.InvalidActivityTrackingOptions, _factoryOptions.ActivityTrackingOptions), nameof(options));
      }
 
      foreach (ILoggerProvider provider in providers)
      {
         AddProviderRegistration(provider, dispose: false);  // <-------------------------5.0->
      }
 
      _changeTokenRegistration = filterOption.OnChange(RefreshFilters);
      RefreshFilters(filterOption.CurrentValue);
   }

   public static ILoggerFactory Create(Action<ILoggingBuilder> configure)
   {
      var serviceCollection = new ServiceCollection();  // <------------------------------logging uses a separate ServiceCollection
      serviceCollection.AddLogging(configure);          // <------------------------------register
      ServiceProvider serviceProvider = serviceCollection.BuildServiceProvider();
      ILoggerFactory loggerFactory = serviceProvider.GetRequiredService<ILoggerFactory>();
      return new DisposingLoggerFactory(loggerFactory, serviceProvider);
   }

   public ILogger CreateLogger(string categoryName)  // <--------------------5.2
   {
      if (CheckDisposed())
      {
         throw new ObjectDisposedException(nameof(LoggerFactory));
      }
 
      lock (_sync)
      {
         if (!_loggers.TryGetValue(categoryName, out Logger? logger))
         {
            // Logger is a wrapper that contains muliple ILogger such as ConsoleLogger, DebugLogger etc, so only one instance of Logger is needed
            logger = new Logger  
            (
               CreateLoggers(categoryName)  // <------------------5.3
            );  
 
            (logger.MessageLoggers, logger.ScopeLoggers) = ApplyFilters(logger.Loggers);  // <--------------------5.7

            _loggers[categoryName] = logger;
         }
 
         return logger;
      }
   }

   private LoggerInformation[] CreateLoggers(string categoryName)  // <----------------5.4
   {
      var loggers = new LoggerInformation[_providerRegistrations.Count];
      for (int i = 0; i < _providerRegistrations.Count; i++)
      {
         loggers[i] = new LoggerInformation(_providerRegistrations[i].Provider, categoryName);  // <----------------5.5
      }
      return loggers;
   }

   private (MessageLogger[] MessageLoggers, ScopeLogger[]? ScopeLoggers) ApplyFilters(LoggerInformation[] loggers)  // <-----------------5.7
   {
      var messageLoggers = new List<MessageLogger>();
      List<ScopeLogger>? scopeLoggers = _filterOptions.CaptureScopes ? new List<ScopeLogger>() : null;
 
      foreach (LoggerInformation loggerInformation in loggers)
      {
         LoggerRuleSelector.Select(_filterOptions,
                    loggerInformation.ProviderType,
                    loggerInformation.Category,
                    out LogLevel? minLevel,
                    out Func<string?, string?, LogLevel, bool>? filter);
 
         if (minLevel is not null and > LogLevel.Critical)
            continue;
 
         messageLoggers.Add(new MessageLogger(loggerInformation.Logger, loggerInformation.Category, loggerInformation.ProviderType.FullName, minLevel, filter));
 
         if (!loggerInformation.ExternalScope)
            scopeLoggers?.Add(new ScopeLogger(logger: loggerInformation.Logger, externalScopeProvider: null));
      }
 
      if (_scopeProvider != null)
      {
         scopeLoggers?.Add(new ScopeLogger(logger: null, externalScopeProvider: _scopeProvider));
      }
 
      return (messageLoggers.ToArray(), scopeLoggers?.ToArray());
   }

   public void AddProvider(ILoggerProvider provider)
   {
      if (CheckDisposed())
         throw new ObjectDisposedException(nameof(LoggerFactory));
  
      lock (_sync)
      {
         AddProviderRegistration(provider, dispose: true);
 
         foreach (KeyValuePair<string, Logger> existingLogger in _loggers)
         {
            Logger logger = existingLogger.Value;
            LoggerInformation[] loggerInformation = logger.Loggers;
 
            int newLoggerIndex = loggerInformation.Length;
            Array.Resize(ref loggerInformation, loggerInformation.Length + 1);
            loggerInformation[newLoggerIndex] = new LoggerInformation(provider, existingLogger.Key);
 
            logger.Loggers = loggerInformation;
            (logger.MessageLoggers, logger.ScopeLoggers) = ApplyFilters(logger.Loggers);
         }
      }
   }

   private void AddProviderRegistration(ILoggerProvider provider, bool dispose)   // <--------------5.0.1
   {
      _providerRegistrations.Add(new ProviderRegistration
      {
         Provider = provider,
         ShouldDispose = dispose
      });
 
      if (provider is ISupportExternalScope supportsExternalScope)
      {
         _scopeProvider ??= new LoggerFactoryScopeProvider(_factoryOptions.ActivityTrackingOptions);
 
         supportsExternalScope.SetScopeProvider(_scopeProvider);
      }
   }

   private void RefreshFilters(LoggerFilterOptions filterOptions)
   {
      lock (_sync)
      {
         _filterOptions = filterOptions;
         foreach (KeyValuePair<string, Logger> registeredLogger in _loggers)
         {
            Logger logger = registeredLogger.Value;
            (logger.MessageLoggers, logger.ScopeLoggers) = ApplyFilters(logger.Loggers);
         }
      }
   }

   protected virtual bool CheckDisposed() => _disposed;

   public void Dispose()
   {
      if (!_disposed)
      {
         _disposed = true;
 
         _changeTokenRegistration?.Dispose();
 
         foreach (ProviderRegistration registration in _providerRegistrations)
         {
            try
            {
               if (registration.ShouldDispose)
                  registration.Provider.Dispose();
                        
            }
            catch
            {
               // swallow exceptions on dispose
            }
         }
      }
   }

   private struct ProviderRegistration
   {
      public ILoggerProvider Provider;
      public bool ShouldDispose;
   }

   private sealed class DisposingLoggerFactory : ILoggerFactory
   {
      private readonly ILoggerFactory _loggerFactory;
 
      private readonly ServiceProvider _serviceProvider;
 
      public DisposingLoggerFactory(ILoggerFactory loggerFactory, ServiceProvider serviceProvider)
      {
         _loggerFactory = loggerFactory;
         _serviceProvider = serviceProvider;
      }
 
      public void Dispose()
      {
         _serviceProvider.Dispose();
      }
 
      public ILogger CreateLogger(string categoryName)
      {
         return _loggerFactory.CreateLogger(categoryName);
      }
 
      public void AddProvider(ILoggerProvider provider)
      {
         _loggerFactory.AddProvider(provider);
      }
   }
}
//------------------------Ʌ

//----------------------------------------V
internal readonly struct LoggerInformation
{
   public LoggerInformation(ILoggerProvider provider, string category) : this()
   {
      ProviderType = provider.GetType();
      Logger = provider.CreateLogger(category);  // <----------------------5.6, create an instance of `Logger`
      Category = category;
      ExternalScope = provider is ISupportExternalScope;
   }
 
   public ILogger Logger { get; }   // <--------------------
 
   public string Category { get; }
 
   public Type ProviderType { get; }
 
   public bool ExternalScope { get; }
}
//----------------------------------------Ʌ

//---------------------------------V
internal sealed class ConsoleLogger : ILogger
{
   private readonly string _name;
   private readonly ConsoleLoggerProcessor _queueProcessor;

   internal ConsoleLogger(string name, ConsoleLoggerProcessor loggerProcessor, ConsoleFormatter formatter, IExternalScopeProvider? scopeProvider, ConsoleLoggerOptions options)
   {
      _name = name;
      _queueProcessor = loggerProcessor;
      Formatter = formatter;
      ScopeProvider = scopeProvider;
      Options = options;
   }

   internal ConsoleFormatter Formatter { get; set; }
   internal IExternalScopeProvider? ScopeProvider { get; set; }  // <---------------LoggerFactoryScopeProvider
   internal ConsoleLoggerOptions Options { get; set; }
   private static StringWriter? t_stringWriter;

   public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter)
   {
      if (!IsEnabled(logLevel))
         return;
  
      t_stringWriter ??= new StringWriter();
      LogEntry<TState> logEntry = new LogEntry<TState>(logLevel, _name, eventId, state, exception, formatter);
      Formatter.Write(in logEntry, ScopeProvider, t_stringWriter);  // <----------------------------------------! important this is the actual method that log/write your message
 
      var sb = t_stringWriter.GetStringBuilder();
      if (sb.Length == 0)
         return;

      string computedAnsiString = sb.ToString();
      sb.Clear();
      if (sb.Capacity > 1024)
      {
         sb.Capacity = 1024;
      }
      _queueProcessor.EnqueueMessage(new LogMessageEntry(computedAnsiString, logAsError: logLevel >= Options.LogToStandardErrorThreshold));
   }

   public bool IsEnabled(LogLevel logLevel)
   {
      return logLevel != LogLevel.None;
   }

   public IDisposable BeginScope<TState>(TState state)  // <-------------------------------------
   {
      return ScopeProvider?.Push(state) ?? NullScope.Instance;  // return LoggerFactoryScopeProvider.Scope instance
   } 
}
//---------------------------------Ʌ

//------------------------------V
public class DebugLoggerProvider : ILoggerProvider
{
   public ILogger CreateLogger(string name)
   {
      return new DebugLogger(name);
   }
 
   public void Dispose() { }
}
//------------------------------Ʌ

//---------------------------------------V
internal sealed partial class DebugLogger : ILogger  // a logger that writes messages in the debug output window only when a debugger is attached
{
   private readonly string _name;

   public DebugLogger(string name)
   {
      _name = name;
   }

   public IDisposable BeginScope<TState>(TState state)
   {
      return NullScope.Instance;
   }

   public bool IsEnabled(LogLevel logLevel)
   {
      return Debugger.IsAttached && logLevel != LogLevel.None;
   }

   public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter)
   {
      if (!IsEnabled(logLevel))
         return;
      
      string message = formatter(state, exception);
 
      if (string.IsNullOrEmpty(message))
         return;
 
      message = $"{ logLevel }: {message}";
 
      if (exception != null)
      {
         message += Environment.NewLine + Environment.NewLine + exception;
      }
 
      DebugWriteLine(message, _name);
   }
}
//---------------------------------------Ʌ

//--------------------------V
internal sealed class Logger : ILogger
{
   public Logger(LoggerInformation[] loggers) 
   {
      Loggers = loggers;
   } 

   public LoggerInformation[] Loggers { get; set; }
   public MessageLogger[] MessageLoggers { get; set; }
   public ScopeLogger[] ScopeLoggers { get; set; }

   public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter) 
   {
      MessageLogger[] loggers = MessageLoggers;

      List<Exception> exceptions = null;
      for (int i = 0; i < loggers.Length; i++) 
      {
         ref readonly MessageLogger loggerInfo = ref loggers[i];
         if (!loggerInfo.IsEnabled(logLevel))
            continue;

         LoggerLog(logLevel, eventId, loggerInfo.Logger, exception, formatter, ref exceptions, state);
      }

      if (exceptions != null && exceptions.Count > 0) 
      {
         ThrowLoggingError(exceptions);
      }

      static void LoggerLog(LogLevel logLevel, EventId eventId, ILogger logger, Exception exception, Func<TState, Exception, string> formatter, ref List<Exception> exceptions, in TState state) {
         try 
         {
            logger.Log(logLevel, eventId, state, exception, formatter);
         }
         catch (Exception ex) {
            if (exceptions == null)
               exceptions = new List<Exception>();
            exceptions.Add(ex);
         }
      }
   }

   public bool IsEnabled(LogLevel logLevel) {
      MessageLogger[] loggers = MessageLoggers;

      List<Exception> exceptions = null;
      int i = 0;
      for (; i < loggers.Length; i++) 
      {
         ref readonly MessageLogger loggerInfo = ref loggers[i];
         if (!loggerInfo.IsEnabled(logLevel)) {
            continue;
         }
         if (LoggerIsEnabled(logLevel, loggerInfo.Logger, ref exceptions)) {
            break;
         }
      }

      if (exceptions != null && exceptions.Count > 0) 
         ThrowLoggingError(exceptions);
      

      return i < loggers.Length ? true : false;

      static bool LoggerIsEnabled(LogLevel logLevel, ILogger logger, ref List<Exception> exceptions) {
         try {
            if (logger.IsEnabled(logLevel)) {
               return true;
            }
         }
         catch (Exception ex) {
            if (exceptions == null)
               exceptions = new List<Exception>();
            exceptions.Add(ex);
         }

         return false;
      }
   }

   public IDisposable BeginScope<TState>(TState state) 
   { 
      ScopeLogger[] loggers = ScopeLoggers;
      
      if (loggers.Length == 1) {
         return loggers[0].CreateScope(state);
      }

      var scope = new Scope(loggers.Length);
      List<Exception> exceptions = null;
      for (int i = 0; i < loggers.Length; i++) {
         ref readonly ScopeLogger scopeLogger = ref loggers[i];

         try {
            scope.SetDisposable(i, scopeLogger.CreateScope(state));
         }
         // catch         
      }

      return scope;
   }

   private sealed class Scope : IDisposable 
   {
      private bool _isDisposed;

      private IDisposable _disposable0;
      private IDisposable _disposable1;
      private readonly IDisposable[] _disposable;

      public Scope(int count) {
         if (count > 2) {
            _disposable = new IDisposable[count - 2];
         }
      }

      public void SetDisposable(int index, IDisposable disposable) 
      {
         switch (index) {
            case 0:
               _disposable0 = disposable;
            case 1:
               _disposable1 = disposable;
               break;
            default:
               _disposable[index - 2] = disposable;
               break;
         }
      }

      public void Dispose() 
      {
         if (!_isDisposed) {
            _disposable0?.Dispose();
            _disposable1?.Dispose();

            if (_disposable != null) {
               int count = _disposable.Length;
               for (int index = 0; index != count; ++index) {
                  if (_disposable[index] != null) {
                     _disposable[index].Dispose();
                  }
               }
            }

            _isDisposed = true;
         }
      }
   }
}
//--------------------------Ʌ

//---------------------------------V
public class Logger<T> : ILogger<T>    // a wrapper of Logger
{
   private readonly ILogger _logger;   // _logger is Logger

   public Logger(ILoggerFactory factory)  // <-------------------------------5
   {   
      _logger = factory.CreateLogger(     // <-------------------------------5.1
         TypeNameHelper.GetTypeDisplayName(typeof(T), includeGenericParameters: false, nestedTypeDelimiter: '.')
      ); 
   }

   IDisposable ILogger.BeginScope<TState>(TState state) {
      return _logger.BeginScope(state);
   }

   bool ILogger.IsEnabled(LogLevel logLevel) {
      return _logger.IsEnabled(logLevel);
   }

   void ILogger.Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter) {
      _logger.Log(logLevel, eventId, state, exception, formatter);
   }
}
//---------------------------------Ʌ

//------------------------------------V
internal readonly struct MessageLogger
{
   public MessageLogger(ILogger logger, string? category, string? providerTypeFullName, LogLevel? minLevel, Func<string?, string?, LogLevel, bool>? filter)
   {
      Logger = logger;
      Category = category;
      ProviderTypeFullName = providerTypeFullName;
      MinLevel = minLevel;
      Filter = filter;
   }

   public ILogger Logger { get; }   // <-----------contains concrete logger e.g `ConsoleLogger`
   public string? Category { get; }
   private string? ProviderTypeFullName { get; }
   public LogLevel? MinLevel { get; }
   public Func<string?, string?, LogLevel, bool>? Filter { get; }
   public bool IsEnabled(LogLevel level)
   {
      if (MinLevel != null && level < MinLevel)
         return false;
 
      if (Filter != null)
         return Filter(ProviderTypeFullName, Category, level);
 
      return true;
   }
}
//------------------------------------Ʌ

//----------------------------------V
internal readonly struct ScopeLogger
{
   public ScopeLogger(ILogger? logger, IExternalScopeProvider? externalScopeProvider)
   {
      Debug.Assert(logger != null || externalScopeProvider != null, "Logger can't be null when there isn't an ExternalScopeProvider");
 
      Logger = logger;
      ExternalScopeProvider = externalScopeProvider;
   }

   public ILogger? Logger { get; }

   public IExternalScopeProvider? ExternalScopeProvider { get; }

   public IDisposable? CreateScope<TState>(TState state) where TState : notnull
   {
      if (ExternalScopeProvider != null)
         return ExternalScopeProvider.Push(state);
         
 
      Debug.Assert(Logger != null);
      return Logger.BeginScope<TState>(state);
   }
}
//----------------------------------Ʌ
```


## Logging with IHttpClientFactory

```C#
//----------------------------------------------------------V
internal sealed class LoggingHttpMessageHandlerBuilderFilter : IHttpMessageHandlerBuilderFilter
{
    // we want to prevent a circular depencency between ILoggerFactory and IHttpMessageHandlerBuilderFilter, in case
    // any of ILoggerProvider instances use IHttpClientFactory to send logs to an external server
    private ILoggerFactory? _loggerFactory;
    private ILoggerFactory LoggerFactory => _loggerFactory ??= _serviceProvider.GetRequiredService<ILoggerFactory>();
    private readonly IServiceProvider _serviceProvider;
    private readonly IOptionsMonitor<HttpClientFactoryOptions> _optionsMonitor;

    public LoggingHttpMessageHandlerBuilderFilter(IServiceProvider serviceProvider, IOptionsMonitor<HttpClientFactoryOptions> optionsMonitor)
    {

        _serviceProvider = serviceProvider;
        _optionsMonitor = optionsMonitor;
    }

    public Action<HttpMessageHandlerBuilder> Configure(Action<HttpMessageHandlerBuilder> next)
    {
        return (builder) =>
        {
            // Run other configuration first, we want to decorate.
            next(builder);

            HttpClientFactoryOptions options = _optionsMonitor.Get(builder.Name);
            if (options.SuppressDefaultLogging)
            {
                return;
            }

            string loggerName = !string.IsNullOrEmpty(builder.Name) ? builder.Name : "Default";

            // We want all of our logging message to show up as-if they are coming from HttpClient,
            // but also to include the name of the client for more fine-grained control.
            ILogger outerLogger = LoggerFactory.CreateLogger($"System.Net.Http.HttpClient.{loggerName}.LogicalHandler");   // <-----------------------loh
            ILogger innerLogger = LoggerFactory.CreateLogger($"System.Net.Http.HttpClient.{loggerName}.ClientHandler");    // <-----------------------clh

            // The 'scope' handler goes first so it can surround everything.
            builder.AdditionalHandlers.Insert(0, new LoggingScopeHttpMessageHandler(outerLogger, options));   // <-----------------------lsch0.

            // We want this handler to be last so we can log details about the request after service discovery and security happen.
            builder.AdditionalHandlers.Add(new LoggingHttpMessageHandler(innerLogger, options));
        };
    }
}
//----------------------------------------------------------Ʌ

//-----------------------------------------V
public class LoggingScopeHttpMessageHandler : DelegatingHandler
{
    private readonly ILogger _logger;
    private readonly HttpClientFactoryOptions? _options;

    private static readonly Func<string, bool> _shouldNotRedactHeaderValue = (header) => false;

    public LoggingScopeHttpMessageHandler(ILogger logger)
    {
        _logger = logger;
    }

    public LoggingScopeHttpMessageHandler(ILogger logger, HttpClientFactoryOptions options)
    {
        _logger = logger;
        _options = options;
    }

    private Task<HttpResponseMessage> SendCoreAsync(HttpRequestMessage request, bool useAsync, CancellationToken cancellationToken)
    {
        return Core(request, useAsync, cancellationToken);

        async Task<HttpResponseMessage> Core(HttpRequestMessage request, bool useAsync, CancellationToken cancellationToken)
        {
            var stopwatch = ValueStopwatch.StartNew();

            Func<string, bool> shouldRedactHeaderValue = _options?.ShouldRedactHeaderValue ?? _shouldNotRedactHeaderValue;

            using (Log.BeginRequestPipelineScope(_logger, request))  // <-----------------------lsch1.0
            {
                Log.RequestPipelineStart(_logger, request, shouldRedactHeaderValue);  // <-----------------------lsch2.0
                HttpResponseMessage response = useAsync
                    ? await base.SendAsync(request, cancellationToken).ConfigureAwait(false) : base.Send(request, cancellationToken);

                Log.RequestPipelineEnd(_logger, response, stopwatch.GetElapsedTime(), shouldRedactHeaderValue);

                return response;
            }
        }
    }

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => SendCoreAsync(request, useAsync: true, cancellationToken);

    protected override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
        => SendCoreAsync(request, useAsync: false, cancellationToken).GetAwaiter().GetResult();

    // Used in tests
    internal static class Log
    {
        public static class EventIds
        {
            public static readonly EventId PipelineStart = new EventId(100, "RequestPipelineStart");
            public static readonly EventId PipelineEnd = new EventId(101, "RequestPipelineEnd");

            public static readonly EventId RequestHeader = new EventId(102, "RequestPipelineRequestHeader");
            public static readonly EventId ResponseHeader = new EventId(103, "RequestPipelineResponseHeader");
        }

        private static readonly Func<ILogger, HttpMethod, string?, IDisposable?> _beginRequestPipelineScope = LoggerMessage.DefineScope<HttpMethod, string?>("HTTP {HttpMethod} {Uri}");   // <-----------------------lsch1.1

        /*
        public static class LoggerMessage
        {
            // ...
            public static Func<ILogger, T1, T2, IDisposable?> DefineScope<T1, T2>(string formatString)
            {
                LogValuesFormatter formatter = CreateLogValuesFormatter(formatString, expectedNamedParameterCount: 2);
      
                return (logger, arg1, arg2) => logger.BeginScope(new LogValues<T1, T2>(formatter, arg1, arg2));  // <-----------------------lsch1.2.
            }
        }       
        */

        private static readonly Action<ILogger, HttpMethod, string?, Exception?> _requestPipelineStart = LoggerMessage.Define<HttpMethod, string?>(
            LogLevel.Information,
            EventIds.PipelineStart,
            "Start processing HTTP request {HttpMethod} {Uri}");   // <-----------------------lsch2.3

        /*
        public static Action<ILogger, T1, T2, Exception?> Define<T1, T2>(LogLevel logLevel, EventId eventId, string formatString)
        {
            // ...
            LogValuesFormatter formatter = CreateLogValuesFormatter(formatString, expectedNamedParameterCount: 2);
 
            void Log(ILogger logger, T1 arg1, T2 arg2, Exception? exception)
            {
                logger.Log(logLevel, eventId, new LogValues<T1, T2>(formatter, arg1, arg2), exception, LogValues<T1, T2>.Callback);
            }
 
            if (options != null && options.SkipEnabledCheck)
            {
                return Log;
            }
 
            return (logger, arg1, arg2, exception) =>
            {
                if (logger.IsEnabled(logLevel))
                {
                    Log(logger, arg1, arg2, exception);   // <-----------------------lsch2.4
                }
            }          
        }          
        */

        private static readonly Action<ILogger, double, int, Exception?> _requestPipelineEnd = LoggerMessage.Define<double, int>(
            LogLevel.Information,
            EventIds.PipelineEnd,
            "End processing HTTP request after {ElapsedMilliseconds}ms - {StatusCode}");

        public static IDisposable? BeginRequestPipelineScope(ILogger logger, HttpRequestMessage request)   // <-----------------------lsch1.1
        {
            return _beginRequestPipelineScope(logger, request.Method, GetUriString(request.RequestUri));
        }

        public static void RequestPipelineStart(ILogger logger, HttpRequestMessage request, Func<string, bool> shouldRedactHeaderValue)  // <-----------------------lsch2.1
        {
            _requestPipelineStart(logger, request.Method, GetUriString(request.RequestUri), null);  // <-----------------------lsch2.2

            if (logger.IsEnabled(LogLevel.Trace))
            {
                logger.Log(
                    LogLevel.Trace,
                    EventIds.RequestHeader,
                    new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Request, request.Headers, request.Content?.Headers, shouldRedactHeaderValue),
                    null,
                    (state, ex) => state.ToString());
            }
        }

        public static void RequestPipelineEnd(ILogger logger, HttpResponseMessage response, TimeSpan duration, Func<string, bool> shouldRedactHeaderValue)
        {
            _requestPipelineEnd(logger, duration.TotalMilliseconds, (int)response.StatusCode, null);

            if (logger.IsEnabled(LogLevel.Trace))
            {
                logger.Log(
                    LogLevel.Trace,
                    EventIds.ResponseHeader,
                    new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Response, response.Headers, response.Content?.Headers, shouldRedactHeaderValue),
                    null,
                    (state, ex) => state.ToString());
            }
        }

        private static string? GetUriString(Uri? requestUri);
    }
}
//-----------------------------------------Ʌ

//------------------------------------V
public class LoggingHttpMessageHandler : DelegatingHandler
{
    private readonly ILogger _logger;
    private readonly HttpClientFactoryOptions? _options;

    private static readonly Func<string, bool> _shouldNotRedactHeaderValue = (header) => false;

    public LoggingHttpMessageHandler(ILogger logger)
    {
        _logger = logger;
    }

    public LoggingHttpMessageHandler(ILogger logger, HttpClientFactoryOptions options)
    {
        _logger = logger;
        _options = options;
    }

    private Task<HttpResponseMessage> SendCoreAsync(HttpRequestMessage request, bool useAsync, CancellationToken cancellationToken)
    {
        return Core(request, useAsync, cancellationToken);

        async Task<HttpResponseMessage> Core(HttpRequestMessage request, bool useAsync, CancellationToken cancellationToken)
        {
            Func<string, bool> shouldRedactHeaderValue = _options?.ShouldRedactHeaderValue ?? _shouldNotRedactHeaderValue;

            // not using a scope here because we always expect this to be at the end of the pipeline, thus there's  not really anything to surround.
            Log.RequestStart(_logger, request, shouldRedactHeaderValue);   // <----------------log "Sending HTTP request {HttpMethod} {Uri}" and 
                                                                           // if logLevel is Trace, log request.Headers
            var stopwatch = ValueStopwatch.StartNew();
            HttpResponseMessage response = useAsync ? await base.SendAsync(request, cancellationToken).ConfigureAwait(false)

            Log.RequestEnd(_logger, response, stopwatch.GetElapsedTime(), shouldRedactHeaderValue);  // <--------------------------------log StatusCode and if logLevel
                                                                                                     // is Trace, log response.Headers
            return response;
        }
    }

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        => SendCoreAsync(request, useAsync: true, cancellationToken);
    
    protected override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
        => SendCoreAsync(request, useAsync: false, cancellationToken).GetAwaiter().GetResult();

}
//------------------------------------Ʌ
```

```C#
//-----------------------------V
internal static class LogHelper
{
   private static readonly LogDefineOptions s_skipEnabledCheckLogDefineOptions = new LogDefineOptions() { SkipEnabledCheck = true };

   private static class EventIds
   {
      public static readonly EventId RequestStart = new EventId(100, "RequestStart");
      public static readonly EventId RequestEnd = new EventId(101, "RequestEnd");

      public static readonly EventId RequestHeader = new EventId(102, "RequestHeader");
      public static readonly EventId ResponseHeader = new EventId(103, "ResponseHeader");

      public static readonly EventId RequestFailed = new EventId(104, "RequestFailed");

      public static readonly EventId PipelineStart = new EventId(100, "RequestPipelineStart");
      public static readonly EventId PipelineEnd = new EventId(101, "RequestPipelineEnd");

      public static readonly EventId RequestPipelineRequestHeader = new EventId(102, "RequestPipelineRequestHeader");
      public static readonly EventId RequestPipelineResponseHeader = new EventId(103, "RequestPipelineResponseHeader");

      public static readonly EventId PipelineFailed = new EventId(104, "RequestPipelineFailed");
   }

   public static readonly Func<string, bool> ShouldRedactHeaderValue = (header) => true;

   private static readonly Action<ILogger, HttpMethod, string?, Exception?> _requestStart = LoggerMessage.Define<HttpMethod, string?>(
      LogLevel.Information,
      EventIds.RequestStart,
      "Sending HTTP request {HttpMethod} {Uri}",
      s_skipEnabledCheckLogDefineOptions);

   private static readonly Action<ILogger, double, int, Exception?> _requestEnd = LoggerMessage.Define<double, int>(
      LogLevel.Information,
      EventIds.RequestEnd,
      "Received HTTP response headers after {ElapsedMilliseconds}ms - {StatusCode}");

   private static readonly Action<ILogger, double, Exception?> _requestFailed = LoggerMessage.Define<double>(
      LogLevel.Information,
      EventIds.RequestFailed,
      "HTTP request failed after {ElapsedMilliseconds}ms");

   private static readonly Func<ILogger, HttpMethod, string?, IDisposable?> _beginRequestPipelineScope = LoggerMessage.DefineScope<HttpMethod, string?>("HTTP {HttpMethod} {Uri}");

   private static readonly Action<ILogger, HttpMethod, string?, Exception?> _requestPipelineStart = LoggerMessage.Define<HttpMethod, string?>(
      LogLevel.Information,
      EventIds.PipelineStart,
      "Start processing HTTP request {HttpMethod} {Uri}");

   private static readonly Action<ILogger, double, int, Exception?> _requestPipelineEnd = LoggerMessage.Define<double, int>(
      LogLevel.Information,
      EventIds.PipelineEnd,
      "End processing HTTP request after {ElapsedMilliseconds}ms - {StatusCode}");

   private static readonly Action<ILogger, double, Exception?> _requestPipelineFailed = LoggerMessage.Define<double>(
      LogLevel.Information,
      EventIds.PipelineFailed,
      "HTTP request failed after {ElapsedMilliseconds}ms");

   public static void LogRequestStart(this ILogger logger, HttpRequestMessage request, Func<string, bool> shouldRedactHeaderValue)
   {
      // We check here to avoid allocating in the GetRedactedUriString call unnecessarily
      if (logger.IsEnabled(LogLevel.Information))
      {
            _requestStart(logger, request.Method, UriRedactionHelper.GetRedactedUriString(request.RequestUri), null);
      }

      if (logger.IsEnabled(LogLevel.Trace))
      {
            logger.Log(
               LogLevel.Trace,
               EventIds.RequestHeader,
               new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Request, request.Headers, request.Content?.Headers, shouldRedactHeaderValue),
               null,
               (state, ex) => state.ToString());
      }
   }

   public static void LogRequestEnd(this ILogger logger, HttpResponseMessage response, TimeSpan duration, Func<string, bool> shouldRedactHeaderValue)
   {
      _requestEnd(logger, duration.TotalMilliseconds, (int)response.StatusCode, null);

      if (logger.IsEnabled(LogLevel.Trace))
      {
            logger.Log(
               LogLevel.Trace,
               EventIds.ResponseHeader,
               new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Response, response.Headers, response.Content?.Headers, shouldRedactHeaderValue),
               null,
               (state, ex) => state.ToString());
      }
   }

   public static void LogRequestFailed(this ILogger logger, TimeSpan duration, HttpRequestException exception) =>
      _requestFailed(logger, duration.TotalMilliseconds, exception);

   public static IDisposable? BeginRequestPipelineScope(this ILogger logger, HttpRequestMessage request, out string? formattedUri)
   {
      formattedUri = UriRedactionHelper.GetRedactedUriString(request.RequestUri);
      return _beginRequestPipelineScope(logger, request.Method, formattedUri);
   }

   public static void LogRequestPipelineStart(this ILogger logger, HttpRequestMessage request, string? formattedUri, Func<string, bool> shouldRedactHeaderValue)
   {
      _requestPipelineStart(logger, request.Method, formattedUri, null);

      if (logger.IsEnabled(LogLevel.Trace))
      {
            logger.Log(
               LogLevel.Trace,
               EventIds.RequestPipelineRequestHeader,
               new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Request, request.Headers, request.Content?.Headers, shouldRedactHeaderValue),
               null,
               (state, ex) => state.ToString());
      }
   }

   public static void LogRequestPipelineEnd(this ILogger logger, HttpResponseMessage response, TimeSpan duration, Func<string, bool> shouldRedactHeaderValue)
   {
      _requestPipelineEnd(logger, duration.TotalMilliseconds, (int)response.StatusCode, null);

      if (logger.IsEnabled(LogLevel.Trace))
      {
            logger.Log(
               LogLevel.Trace,
               EventIds.RequestPipelineResponseHeader,
               new HttpHeadersLogValue(HttpHeadersLogValue.Kind.Response, response.Headers, response.Content?.Headers, shouldRedactHeaderValue),
               null,
               (state, ex) => state.ToString());
      }
   }

   public static void LogRequestPipelineFailed(this ILogger logger, TimeSpan duration, HttpRequestException exception) =>
      _requestPipelineFailed(logger, duration.TotalMilliseconds, exception);
}
//-----------------------------Ʌ
```
=================================================================================================================================

## Serilog


```C#
//------------------V
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args)
    {
        Host.CreateDefaultBuilder(args)
           .UseSerilog((context, configuration) =>  // (HostBuilderContext context, Serilog.LoggerConfiguration configuration)
           {
               configuration
                   .Enrich.FromLogContext() // allow you use add custom field by `using (LogContext.PushProperty("CustomField", "Hello World"))`, check s1
                   .Enrich.WithMachineName()
                   .WriteTo.Console()
                   .WriteTo.Elasticsearch(
                       new ElasticsearchSinkOptions(new Uri(context.Configuration["ElasticConfiguration:Uri"]))
                       {
                           IndexFormat = $"applogs-{context.HostingEnvironment.ApplicationName?.ToLower().Replace(".", "-")}-{context.HostingEnvironment.EnvironmentName?.ToLower().Replace(".", "-")}-{DateTime.UtcNow:yyyy-MM}",
                           AutoRegisterTemplate = true,
                           NumberOfShards = 2,
                           NumberOfReplicas = 1
                       }
                   )
                   .Enrich.WithProperty("Environment", context.HostingEnvironment.EnvironmentName)
                   .Enrich.WithProperty("Application", context.HostingEnvironment.ApplicationName)
                   .ReadFrom.Configuration(context.Configuration);  // read "Serilog" configuration in appsetting.json
           })
           .ConfigureWebHostDefaults(webBuilder =>
           {
               webBuilder.UseStartup<Startup>();
           });
    }      
}
//------------------Ʌ
```

An Elasticsearch **index** is a logical namespace that holds a collection of documents, where each document is a collection of fields — which, in turn, are key-value pairs that contain your data. Elasticsearch indices are not the same as you'd find in a relational database. Think of an Elasticsearch cluster as a database that can contain many indices you can consider as a table, and within each index, you have many documents.

```C#
public class XXXService : IXXXService
{
    private readonly ILogger<XXXService> _logger;

    public CatalogService(ILogger<XXXService> logger)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task<IEnumerable<XXX>> GetXXX()
    {
        using (_logger.BeginScope(new Dictionary<string, object> { { "name", "John" }, { "age", 21 } }))
        using (LogContext.PushProperty("CustomField", "Hello World"))  // <----------------------s1
        {
            _logger.LogInformation("Getting Catalog Products from url: {url} and custom property : {customProperty}", _client.BaseAddress, 6);
        }

        // ...
      
    }
}
```

```json
{
  "_index": "applogs-aspnetrunbasics-development-2024-02",
  "_type": "logevent",
  "_id": "ZS1Z740BlWyeECfksfGE",
  "_version": 1,
  "_score": null,
  "_source": {
    "@timestamp": "2024-02-28T21:54:07.9163655+11:00",
    "level": "Information",
    "messageTemplate": "Getting Catalog Products from url: {url} and custom property : {customProperty}",
    "message": "Getting Catalog Products from url: http://localhost:8010/ and custom property : 6",
    "fields": {
      "url": "http://localhost:8010/",
      "customProperty": 6,
      "SourceContext": "AspnetRunBasics.Services.CatalogService",
      "name": "John",  // <------------------------from BeginScope()
      "age": 21,       // <------------------------from BeginScope()
      "ActionId": "09d701e2-37f7-45f3-9d6b-f1d396948533",
      "ActionName": "/Index",
      "RequestId": "0HN1O9EB3MKEB:00000002",
      "RequestPath": "/",
      "ConnectionId": "0HN1O9EB3MKEB",
      "CustomField": "Hello World",  // <-------------------------s1
      "MachineName": "DESKTOP-KLM1TNG",
      "Environment": "Development",
      "Application": "AspnetRunBasics"
    }
  },
  "fields": {
    "@timestamp": [
      "2024-02-28T10:54:07.916Z"
    ]
  },
  "sort": [
    1709117647916
  ]
}
```
