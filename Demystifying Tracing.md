## Demystifying Distributed Tracing


```C#
//------------------V
class Program
{
    private static ActivitySource source = new ActivitySource("Sample.DistributedTracing", "1.0.0");

    static async Task Main(string[] args)
    {
        using var tracerProvider = Sdk.CreateTracerProviderBuilder()
            .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("MySample"))
            .AddSource("Sample.DistributedTracing")  // this has to be the same as the the name pass in the new ActivitySource(string name) above
            .AddConsoleExporter()
            .Build();  // create and return a TracerProviderSdk instance which register an ActivityListener by calling ActivitySource.AddActivityListener

        await DoSomeWork("banana", 8);
        Console.WriteLine("Example work done");

        Console.ReadLine();
    }

    static async Task DoSomeWork(string foo, int bar)
    {
        // In .NET world, a span is represented by an Activity
        using (Activity activity = source.StartActivity("SomeWork"))  // StartActivity might return null if there is no registered ActivityListener
        {
            activity?.SetTag("foo", foo);
            activity?.SetTag("bar", bar);
            await StepOne();
            activity?.AddEvent(new ActivityEvent("Part way there"));
            await StepTwo();
            activity?.AddEvent(new ActivityEvent("Done now"));

            // Pretend something went wrong
            activity?.SetTag("otel.status_code", "ERROR");
            activity?.SetTag("otel.status_description", "Use this text give more information about the error");
        }  // Activity implements IDisposable, so you don't need to call activity.Stop(), `using ()` will automatically close it for you
    }

    static async Task StepOne()
    {
        using (Activity activity = source.StartActivity("StepOne"))
        {
            await Task.Delay(500);
            await InnerOne();  // <----------------------create a child span
        }
    }

    static async Task StepTwo()
    {
        using (Activity activity = source.StartActivity("StepTwo"))
        {
            await Task.Delay(1000);
        }

    }

    static async Task InnerOne()
    {
        using (Activity activity = source.StartActivity("StepOneInner"))
        {
            await Task.Delay(1000);
        }

    }
}
//------------------Ʌ 
```

```yml
Activity.Id:          00-82cf6ea92661b84d9fd881731741d04e-33fff2835a03c041-01
Activity.DisplayName: SomeWork
Activity.Kind:        Internal
Activity.StartTime:   2021-03-18T10:39:10.6902609Z
Activity.Duration:    00:00:01.5147582
Activity.TagObjects:
    foo: banana
    bar: 8
Activity.Events:
    Part way there [3/18/2021 10:39:11 AM +00:00]
    Done now [3/18/2021 10:39:12 AM +00:00]
Resource associated with Activity:
    service.name: MySample
    service.instance.id: ea7f0fcb-3673-48e0-b6ce-e4af5a86ce4f

Example work done

DoSomeWork

Id       00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-72dc8c0dc24fc406-01  ## leading 00 is the version, and 01 is the trace flag, meaning "sampled" (tailing 00 mean not sampled), 
TraceId     bbd8c75b04cda7f7fe9ee03b54b2d7ba                      ## TraceId is 16 bytes in hex-encoded, so it has 32 characters
SpanId                                       72dc8c0dc24fc406
ParentId     null
ParentSpanId 0000000000000000


StepOne

Id         00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-51de8ae5367e022e-01
TraceId       bbd8c75b04cda7f7fe9ee03b54b2d7ba
SpanId                                         51de8ae5367e022e
ParentId    00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-72dc8c0dc24fc406-01
ParentSpanId 72dc8c0dc24fc406


InnerOne

Id         00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-0ff550a48811adfa-01
TraceId       bbd8c75b04cda7f7fe9ee03b54b2d7ba
SpanId                                         0ff550a48811adfa
ParentId    00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-51de8ae5367e022e-01
ParentSpanId 51de8ae5367e022e


StepTwo
Id         00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-750752b828701ec1-01
TraceId       bbd8c75b04cda7f7fe9ee03b54b2d7ba
SpanId                                         750752b828701ec1
ParentId    00-bbd8c75b04cda7f7fe9ee03b54b2d7ba-72dc8c0dc24fc406-01
ParentSpanId 72dc8c0dc24fc406
```

======================================================================================================


## OpenTelemetry

The word "telemetry" originates from the Greek roots "tele" meaning "remote" or "far," and "metron," meaning "measure". Oxford Advanced Learner's Dictionary defines *Telemetry* as: the process of using special equipment to send, receive and measure scientific data over long distances

`Telemetry` in .NET refers to the process of collecting and analyzing data about an application's behavior and performance.



```C#
public static class Sdk
{
    public static TracerProviderBuilder CreateTracerProviderBuilder() => new TracerProviderBuilderBase();

    // ...
}

public class TracerProviderBuilderBase : TracerProviderBuilder, ITracerProviderBuilder
{
    private readonly bool allowBuild;
    private readonly TracerProviderServiceCollectionBuilder innerBuilder;

    public override TracerProviderBuilder AddInstrumentation<TInstrumentation>(Func<TInstrumentation> instrumentationFactory)
    {
        this.innerBuilder.AddInstrumentation(instrumentationFactory);

        return this;
    }

    public override TracerProviderBuilder AddSource(params string[] names)
    {
        this.innerBuilder.AddSource(names);

        return this;
    }

    protected TracerProvider Build()  // <-------------------------------
    {
        // ...
        var serviceProvider = services.BuildServiceProvider(validateScopes);

        return new TracerProviderSdk(serviceProvider, ownsServiceProvider: true);
    }

    // ...
}

internal sealed class TracerProviderSdk : TracerProvider
{
    internal readonly IServiceProvider ServiceProvider;
    internal readonly IDisposable? OwnedServiceProvider;
    internal int ShutdownCount;
    internal bool Disposed;

    private readonly List<object> instrumentations = new();
    private readonly ActivityListener listener;  // <----------------------------------
    private readonly Sampler sampler;
    private readonly Action<Activity> getRequestedDataAction;
    private readonly bool supportLegacyActivity;
    private BaseProcessor<Activity>? processor;

    internal TracerProviderSdk(IServiceProvider serviceProvider, bool ownsServiceProvider)
    {
        // ...
        var listener = new ActivityListener();  // <-------------------------------------

        listener.ActivityStarted = activity =>
        {
            OpenTelemetrySdkEventSource.Log.ActivityStarted(activity);

            if (activity.IsAllDataRequested && SuppressInstrumentationScope.IncrementIfTriggered() == 0)
            {
                this.processor?.OnStart(activity);
            }
        };

        listener.ActivityStopped = activity =>
        {
            OpenTelemetrySdkEventSource.Log.ActivityStopped(activity);

            if (!activity.IsAllDataRequested)
            {
                return;
            }

            if (SuppressInstrumentationScope.DecrementIfTriggered() == 0)
            {
                this.processor?.OnEnd(activity);
            }
        };

        ActivitySource.AddActivityListener(listener);  // <----------------------------------
        this.listener = listener;
    }
}
```


## Source Code

```C#
//-------------------V
public class Activity : IDisposable  // In .NET world, a span is represented by an Activity
{
    public Activity(string operationName);

	 public static Activity? Current { get; set; }
	 public static ActivityIdFormat DefaultIdFormat { get; set; }
	 public static bool ForceDefaultIdFormat { get; set; }
	 public IEnumerable<KeyValuePair<string, object?>> TagObjects { get; }
	 public ActivityTraceId TraceId { get; }
	 public string? Id { get; }
	 public bool IsAllDataRequested { get; set; }
	 public ActivityIdFormat IdFormat { get; }
	 public ActivityKind Kind { get; }
	 public string OperationName { get; }
	 public string DisplayName { get; set; }
	 public IEnumerable<ActivityEvent> Events { get; }
	 public ActivitySource Source { get; }
	 public IEnumerable<ActivityLink> Links { get; }
	 public ActivitySpanId ParentSpanId { get; }
	 public bool Recorded { get; }
	 public string? RootId { get; }
	 public ActivitySpanId SpanId { get; }
	 public DateTime StartTimeUtc { get; }
	 public IEnumerable<KeyValuePair<string, string?>> Tags { get; }
	 public Activity? Parent { get; }
	 public string? ParentId { get; }
    public TimeSpan Duration { get; }
	 public IEnumerable<KeyValuePair<string, string?>> Baggage { get; }
	 public string? TraceStateString { get; set; }
	 public ActivityContext Context { get; }
    public ActivityTraceFlags ActivityTraceFlags { get; set; }

	 public Activity AddBaggage(string key, string? value);
    public Activity AddEvent(ActivityEvent e);
	 public Activity AddTag(string key, string? value);
	 public Activity AddTag(string key, object? value);
	 public string? GetBaggageItem(string key);
	 public object? GetCustomProperty(string propertyName);
	 public void SetCustomProperty(string propertyName, object? propertyValue);
	 public Activity SetEndTime(DateTime endTimeUtc);
	 public Activity SetIdFormat(ActivityIdFormat format);
	 public Activity SetParentId(string parentId);
	 public Activity SetParentId(ActivityTraceId traceId, ActivitySpanId spanId, ActivityTraceFlags activityTraceFlags = ActivityTraceFlags.None);
	 public Activity SetStartTime(DateTime startTimeUtc);
	 public Activity SetTag(string key, object? value);
	 public Activity Start();
	 public void Stop();
}
//-------------------Ʌ 
```

```C#
//--------------------------------V
public sealed class ActivitySource : IDisposable
{
    private static readonly SynchronizedList<ActivitySource> s_activeSources = new SynchronizedList<ActivitySource>();     // <-------------------static
    private static readonly SynchronizedList<ActivityListener> s_allListeners = new SynchronizedList<ActivityListener>();  // <-------------------static
    private SynchronizedList<ActivityListener>? _listeners;  // <--------------non static listener
    
    public ActivitySource(string name, string? version = "")
    {
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Version = version;
 
        s_activeSources.Add(this);  // <------------------------add itself
 
        if (s_allListeners.Count > 0)
        {
            s_allListeners.EnumWithAction((listener, source) =>
            {
                Func<ActivitySource, bool>? shouldListenTo = listener.ShouldListenTo;
                if (shouldListenTo != null)
                {
                    var activitySource = (ActivitySource)source;
                    if (shouldListenTo(activitySource))
                    {
                        activitySource.AddListener(listener);  // <-------------------------
                    }
                }
            }, this);
        }
 
        GC.KeepAlive(DiagnosticSourceEventSource.Log);
    }

	public string Name { get; }
	public string? Version { get; }

	public static void AddActivityListener(ActivityListener listener)  // <-----------------------------static
    {
        if (s_allListeners.AddIfNotExist(listener))
        {
            s_activeSources.EnumWithAction((source, obj) => {
                var shouldListenTo = ((ActivityListener)obj).ShouldListenTo;
                if (shouldListenTo != null && shouldListenTo(source))
                {
                    source.AddListener((ActivityListener)obj);
                }
            }, listener);
        }
    }

	public bool HasListeners();

	public Activity? StartActivity(string name, ActivityKind kind = ActivityKind.Internal);
	public Activity? StartActivity(string name, ActivityKind kind, ActivityContext parentContext,  IEnumerable<KeyValuePair<string, object?>>? tags = null,                   
	                               IEnumerable<ActivityLink>? links = null, DateTimeOffset startTime = default);
	public Activity? CreateActivity(string name, ActivityKind kind) => CreateActivity(name, kind, default, null, null, null, default, startIt: false);
	private Activity? CreateActivity(string name, ActivityKind kind, ActivityContext context, string? parentId, IEnumerable<KeyValuePair<string, object?>>? tags,
                                    IEnumerable<ActivityLink>? links, DateTimeOffset startTime, bool startIt = true, ActivityIdFormat idFormat = ActivityIdFormat.Unknown)
}
//--------------------------------Ʌ

//------------------------------------V
public readonly struct ActivityTraceId : IEquatable<ActivityTraceId>
{
	public static ActivityTraceId CreateFromBytes(ReadOnlySpan<byte> idData);
	public static ActivityTraceId CreateFromString(ReadOnlySpan<char> idData);
	public static ActivityTraceId CreateFromUtf8String(ReadOnlySpan<byte> idData);
	public static ActivityTraceId CreateRandom();
	public void CopyTo(Span<byte> destination);
	public bool Equals(ActivityTraceId traceId);

	public static bool operator ==(ActivityTraceId traceId1, ActivityTraceId traceId2);
	public static bool operator !=(ActivityTraceId traceId1, ActivityTraceId traceId2);
}
//------------------------------------Ʌ

//------------------------------------V
public readonly struct ActivitySpanId : IEquatable<ActivitySpanId>
{
	public static ActivitySpanId CreateFromBytes(ReadOnlySpan<byte> idData);
	public static ActivitySpanId CreateFromString(ReadOnlySpan<char> idData);
	public static ActivitySpanId CreateFromUtf8String(ReadOnlySpan<byte> idData);
	public static ActivitySpanId CreateRandom();
	public void CopyTo(Span<byte> destination);
	public bool Equals(ActivitySpanId traceId);

	public static bool operator ==(ActivitySpanId traceId1, ActivitySpanId traceId2);
	public static bool operator !=(ActivitySpanId traceId1, ActivitySpanId traceId2);
}
//------------------------------------Ʌ

//------------------------------------V
public readonly struct ActivityContext : IEquatable<ActivityContext>
{
    public ActivityContext(ActivityTraceId traceId, ActivitySpanId spanId, ActivityTraceFlags traceFlags, string? traceState = null, bool isRemote = false);

	public ActivityTraceId TraceId { get; }
	public ActivitySpanId SpanId { get; }
	public ActivityTraceFlags TraceFlags { get; }
	public string? TraceState { get; }
	public bool IsRemote { get; }

	public static ActivityContext Parse(string traceParent, string? traceState);
	public static bool TryParse(string traceParent, string? traceState, out ActivityContext context);
	public bool Equals(ActivityContext value);
    
	public static bool operator ==(ActivityContext left, ActivityContext right);
	public static bool operator !=(ActivityContext left, ActivityContext right);
}
//------------------------------------Ʌ

//----------------------------------V
public readonly struct ActivityEvent
{
    public ActivityEvent(string name);
	public ActivityEvent(string name, DateTimeOffset timestamp = default, ActivityTagsCollection? tags = null);

	public string Name { get; }
	public DateTimeOffset Timestamp { get; }
	public IEnumerable<KeyValuePair<string, object?>> Tags { get; }
}
//----------------------------------Ʌ

//---------------------------------V
public readonly struct ActivityLink : IEquatable<ActivityLink>
{
    public ActivityLink(ActivityContext context, ActivityTagsCollection? tags = null);

	public ActivityContext Context { get; }
    public IEnumerable<KeyValuePair<string, object?>>? Tags { get; }
    public bool Equals(ActivityLink value);

	public static bool operator ==(ActivityLink left, ActivityLink right);
    public static bool operator !=(ActivityLink left, ActivityLink right);
}
//---------------------------------Ʌ

//----------------------------------V
public sealed class ActivityListener : IDisposable
{
    public ActivityListener();

	public Action<Activity>? ActivityStarted { get; set; }
	public Action<Activity>? ActivityStopped { get; set; }
	public Func<ActivitySource, bool>? ShouldListenTo { get; set; }
	public SampleActivity<string>? SampleUsingParentId { get; set; }
	public SampleActivity<ActivityContext>? Sample { get; set; }

	public delegate ActivitySamplingResult SampleActivity<T>(ref ActivityCreationOptions<T> options);
}
//----------------------------------Ʌ

//-----------------------------------------------V
public readonly struct ActivityCreationOptions<T>
{
    public ActivitySource Source { get; }
	public string Name { get; }
	public ActivityKind Kind { get; }
	public T Parent { get; }
	public IEnumerable<KeyValuePair<string, object?>>? Tags { get; }
	public IEnumerable<ActivityLink>? Links { get; }
	public ActivityTagsCollection SamplingTags { get; }
	public ActivityTraceId TraceId { get; }
}
//-----------------------------------------------Ʌ

//---------------------------------V
public class ActivityTagsCollection : IDictionary<string, object?>, ICollection<KeyValuePair<string, object?>>, IEnumerable<KeyValuePair<string, object?>>, IEnumerable
{
    public ActivityTagsCollection();
	public ActivityTagsCollection(IEnumerable<KeyValuePair<string, object?>> list);

	public object? this[string key];
	public bool IsReadOnly { get; }
	public int Count { get; }
	public ICollection<object?> Values { get; }
	public ICollection<string> Keys { get; }

	public void Add(string key, object? value);
	public void Add(KeyValuePair<string, object?> item);
	public bool Contains(KeyValuePair<string, object?> item);
	public bool ContainsKey(string key);
	public void Clear();
	// ...
}
//---------------------------------Ʌ

//----------------------V
public enum ActivityKind
{
	Internal = 0,
	Server = 1,
	Client = 2,
	Producer = 3,
	Consumer = 4
}
//----------------------Ʌ

//--------------------------V
public enum ActivityIdFormat
{
	Unknown = 0,
	Hierarchical = 1,
	W3C = 2
}
//--------------------------Ʌ

//----------------------------V
public enum ActivityTraceFlags
{
	None = 0,
	Recorded = 1
}
//----------------------------Ʌ
```