## Demystifying Distributed Tracing

```
```

======================================================================================================


## Source Code

```C#
//-------------------V
public class Activity : IDisposable
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
    public ActivitySource(string name, string? version = "");

	public string Name { get; }
	public string? Version { get; }

	public static void AddActivityListener(ActivityListener listener);
	public bool HasListeners();

	public Activity? StartActivity(string name, ActivityKind kind = ActivityKind.Internal);
	public Activity? StartActivity(string name, 
	                               ActivityKind kind,
	                               ActivityContext parentContext, 
								   IEnumerable<KeyValuePair<string, object?>>? tags = null,                   
	                               IEnumerable<ActivityLink>? links = null, 
								   DateTimeOffset startTime = default);
    public Activity? StartActivity(string name, 
	                               ActivityKind kind, 
								   string parentId, 
								   IEnumerable<KeyValuePair<string, object?>>? tags = null, 
	                               IEnumerable<ActivityLink>? links = null,
								    DateTimeOffset startTime = default);
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