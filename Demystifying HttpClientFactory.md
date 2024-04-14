## Demystifying HttpClientFactory


```C#
public class Startup
{

    public void ConfigureServices(IServiceCollection services)
    {     
        services
            .AddHttpClient<IBasketService, BasketService>(c => c.BaseAddress = new Uri(Configuration["ApiSettings:BasketUrl"]))
            .AddHttpMessageHandler<LoggingDelegatingHandler>()
            .AddTransientHttpErrorPolicy(policy => policy.WaitAndRetryAsync(retryCount: 3, sleepDurationProvider: _ => TimeSpan.FromSeconds(2)))
            .AddTransientHttpErrorPolicy(policy => policy.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));

        // ...
    }
}
```

https://medium.com/@organicprogrammer/why-tcp-connection-termination-needs-four-way-handshake-90d68bb82816
https://www.stevejgordon.co.uk/introduction-to-httpclientfactory-aspnetcore
https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests

basd on the articles above, if you new up a `HttpClient` instance and use it directly, `HttpClient`'s `HttpClientHandler` (which derives from `HttpMessageHandler`) causes the issue (it's the HttpClientHandler which it uses to make the HTTP calls that is the actual issue. It's this which opens the connections to the external services that will then remain open and block sockets)

By using `IHttpClientFactory`, it assigns an `HttpMessageHandler` from a pool to the HttpClient. `HttpClient` may (or may not) use an existing HttpClientHandler from the pool and therefore use an existing open connection.

==========================================================================================================================



# Source Code

```C#
public static partial class HttpClientBuilderExtensions
{
    public static IHttpClientBuilder ConfigureHttpClient(this IHttpClientBuilder builder, Action<HttpClient> configureClient)
    { 
        builder.Services.Configure<HttpClientFactoryOptions>(builder.Name, options => options.HttpClientActions.Add(configureClient));
 
        return builder;
    }

    // ...

    public static IHttpClientBuilder AddHttpMessageHandler<THandler>(this IHttpClientBuilder builder) where THandler : DelegatingHandler
    { 
        builder.Services.Configure<HttpClientFactoryOptions>(builder.Name, options =>
        {
            options.HttpMessageHandlerBuilderActions.Add(b => b.AdditionalHandlers.Add(b.Services.GetRequiredService<THandler>()));
        });
 
        return builder;
    }

    public static IHttpClientBuilder ConfigurePrimaryHttpMessageHandler(this IHttpClientBuilder builder, Func<HttpMessageHandler> configureHandler)
    { 
        builder.Services.Configure<HttpClientFactoryOptions>(builder.Name, options =>
        {
            options.HttpMessageHandlerBuilderActions.Add(b => b.PrimaryHandler = configureHandler());
        });
 
        return builder;
    }

    public static IHttpClientBuilder AddTypedClient<TClient>(this IHttpClientBuilder builder) where TClient : class
    {
        return AddTypedClientCore<TClient>(builder, validateSingleType: false);
    }

    internal static IHttpClientBuilder AddTypedClientCore<TClient>(this IHttpClientBuilder builder, bool validateSingleType) where TClient : class
    {
        ReserveClient(builder, typeof(TClient), builder.Name, validateSingleType);
 
        builder.Services.AddTransient(s => AddTransientHelper<TClient>(s, builder));
 
        return builder;
    }

    public static IHttpClientBuilder SetHandlerLifetime(this IHttpClientBuilder builder, TimeSpan handlerLifetime)
    { 
        if (handlerLifetime != Timeout.InfiniteTimeSpan && handlerLifetime < HttpClientFactoryOptions.MinimumHandlerLifetime)
        {
            throw new ArgumentException(SR.HandlerLifetime_InvalidValue, nameof(handlerLifetime));
        }
 
        builder.Services.Configure<HttpClientFactoryOptions>(builder.Name, options => options.HandlerLifetime = handlerLifetime);
        return builder;
    }

    public static IHttpClientBuilder ConfigureAdditionalHttpMessageHandlers(this IHttpClientBuilder builder, Action<IList<DelegatingHandler>, IServiceProvider> configureAdditionalHandlers)
    { 
        builder.Services.Configure<HttpClientFactoryOptions>(builder.Name, options =>
        {
            options.HttpMessageHandlerBuilderActions.Add(b => configureAdditionalHandlers(b.AdditionalHandlers, b.Services));
        });
 
        return builder;
    }

    private static void ReserveClient(IHttpClientBuilder builder, Type type, string name, bool validateSingleType)
    {
        var registry = (HttpClientMappingRegistry?)builder.Services.Single(sd => sd.ServiceType == typeof(HttpClientMappingRegistry)).ImplementationInstance;
 
        // check for same name registered to two types. This won't work because we rely on named options for the configuration.
        if (registry.NamedClientRegistrations.TryGetValue(name, out Type? otherType) &&
            // allow using the same name with multiple types in some cases (see callers).
            validateSingleType &&
            // allow registering the same name twice to the same type.
            type != otherType)
        {
            string message =
                $"The HttpClient factory already has a registered client with the name '{name}', bound to the type '{otherType.FullName}'. " +
                $"Client names are computed based on the type name without considering the namespace ('{otherType.Name}'). " +
                $"Use an overload of AddHttpClient that accepts a string and provide a unique name to resolve the conflict.";
            throw new InvalidOperationException(message);
        }
 
        if (validateSingleType)
        {
            registry.NamedClientRegistrations[name] = type;
        }
    }
}
```


```C#
//--------------------------------------V
public abstract class HttpMessageHandler : IDisposable
{
    protected HttpMessageHandler()
    {
        if (NetEventSource.Log.IsEnabled()) NetEventSource.Info(this);
    }

    protected internal virtual HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        throw new NotSupportedException(SR.Format(SR.net_http_missing_sync_implementation, GetType(), nameof(HttpMessageHandler), nameof(Send)));
    }

    protected internal abstract Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken);

    protected virtual void Dispose(bool disposing)
    {
        // Nothing to do in base class.
    }
 
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
//--------------------------------------Ʌ

//-------------------------------------V
public abstract class DelegatingHandler : HttpMessageHandler
{
    private HttpMessageHandler? _innerHandler;
    private volatile bool _operationStarted;
    private volatile bool _disposed;

    protected DelegatingHandler() { }

    protected DelegatingHandler(HttpMessageHandler innerHandler)
    {
        InnerHandler = innerHandler;
    }

    public HttpMessageHandler? InnerHandler
    {
        get {
            return _innerHandler;
        }
        set {
            CheckDisposedOrStarted();
 
            if (NetEventSource.Log.IsEnabled()) NetEventSource.Associate(this, value);
                _innerHandler = value;
        }
    }

    protected internal override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    { 
        SetOperationStarted();
        return _innerHandler!.Send(request, cancellationToken);
    }

    protected internal override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    { 
        SetOperationStarted();
        return _innerHandler!.SendAsync(request, cancellationToken);
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing && !_disposed)
        {
            _disposed = true;
            _innerHandler?.Dispose();
        }
 
        base.Dispose(disposing);
    }
}
//-------------------------------------Ʌ

//------------------------------------V
public partial class HttpClientHandler : HttpMessageHandler
{
    private readonly SocketsHttpHandler _underlyingHandler;

    private HttpHandlerType Handler => _underlyingHandler;

    private volatile bool _disposed;
 
    public HttpClientHandler()
    {
        _underlyingHandler = new HttpHandlerType();
 
        ClientCertificateOptions = ClientCertificateOption.Manual;
    }
 
    protected override void Dispose(bool disposing)
    {
        if (disposing && !_disposed)
        {
            _disposed = true;
            _underlyingHandler.Dispose();
        }
 
        base.Dispose(disposing);
    }
 
    private HttpHandlerType Handler => _underlyingHandler;

    public virtual bool SupportsAutomaticDecompression => HttpHandlerType.SupportsAutomaticDecompression;
    public virtual bool SupportsProxy => HttpHandlerType.SupportsProxy;
    public virtual bool SupportsRedirectConfiguration => HttpHandlerType.SupportsRedirectConfiguration;

    public IMeterFactory? MeterFactory
    {
        get => _underlyingHandler.MeterFactory;
        set => _underlyingHandler.MeterFactory = value;
    }

    public bool UseCookies
    {
        get => _underlyingHandler.UseCookies;
        set => _underlyingHandler.UseCookies = value;
    }

    public CookieContainer CookieContainer
    {
        get => _underlyingHandler.CookieContainer;
        set =>  _underlyingHandler.CookieContainer = value;
    }
    public DecompressionMethods AutomaticDecompression { get; set; }
    public bool UseProxy { get; set; }
    public IWebProxy? Proxy { get; set; }
    public public ICredentials? DefaultProxyCredentials { get; set; }
    public bool PreAuthenticate { get; set; }
    public DecompressionMethods AutomaticDecompression { get; set; }
    public X509CertificateCollection ClientCertificates { get; set; }
    public IDictionary<string, object?> Properties => _underlyingHandler.Properties;
    // ... all based on _underlyingHandler

    protected internal override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        return Handler.Send(request, cancellationToken);
    }

    protected internal override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        return Handler.SendAsync(request, cancellationToken);
    }
}
//------------------------------------Ʌ

//------------------------------------V
public sealed class SocketsHttpHandler : HttpMessageHandler
{
    private readonly HttpConnectionSettings _settings = new HttpConnectionSettings();
    private HttpMessageHandlerStage? _handler;
    private Func<HttpConnectionSettings, HttpMessageHandlerStage, HttpMessageHandlerStage>? _decompressionHandlerFactory;
    private bool _disposed;

    public bool UseCookies
    {
        get => _settings._useCookies;
        set => _settings._useCookies = value;
    }
    // ...

    private HttpMessageHandlerStage SetupHandlerChain()
    {
        HttpConnectionSettings settings = _settings.CloneAndNormalize();
 
        HttpConnectionPoolManager poolManager = new HttpConnectionPoolManager(settings);
 
        HttpMessageHandlerStage handler;
 
        if (settings._credentials == null)
        {
            handler = new HttpConnectionHandler(poolManager);
        }
        else
        {
            handler = new HttpAuthenticatedConnectionHandler(poolManager);
        }

        // DiagnosticsHandler is inserted before RedirectHandler so that trace propagation is done on redirects as well
        if (DiagnosticsHandler.IsGloballyEnabled() && settings._activityHeadersPropagator is DistributedContextPropagator propagator)
        {
            handler = new DiagnosticsHandler(handler, propagator, settings._allowAutoRedirect);
        }
 
        handler = new MetricsHandler(handler, settings._meterFactory, out Meter meter);
 
        settings._metrics = new SocketsHttpHandlerMetrics(meter);
 
        if (settings._allowAutoRedirect)
        {
            // Just as with WinHttpHandler, for security reasons, we do not support authentication on redirects if the credential is anything other than a CredentialCache.
            // We allow credentials in a CredentialCache since they are specifically tied to URIs.
            HttpMessageHandlerStage redirectHandler =
                (settings._credentials == null || settings._credentials is CredentialCache) ?
                handler :
                new HttpConnectionHandler(poolManager);  // will not authenticate
 
            handler = new RedirectHandler(settings._maxAutomaticRedirections, handler, redirectHandler);
        }
 
        if (settings._automaticDecompression != DecompressionMethods.None)
        {
            handler = _decompressionHandlerFactory(settings, handler);
        }
 
        // Ensure a single handler is used for all requests.
        if (Interlocked.CompareExchange(ref _handler, handler, null) != null)
        {
            handler.Dispose();
        }
 
        return _handler;
    }

    private void EnsureDecompressionHandlerFactory()
    {
        _decompressionHandlerFactory ??= (settings, handler) => new DecompressionHandler(settings._automaticDecompression, handler);
    }

    protected internal override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();
 
        HttpMessageHandlerStage handler = _handler ?? SetupHandlerChain();
 
        Exception? error = ValidateAndNormalizeRequest(request);
        if (error != null) 
            throw error;
            
 
        return handler.Send(request, cancellationToken);
    }

    protected internal override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    { 
        if (cancellationToken.IsCancellationRequested)
            return Task.FromCanceled<HttpResponseMessage>(cancellationToken);
 
        HttpMessageHandlerStage handler = _handler ?? SetupHandlerChain();
 
        Exception? error = ValidateAndNormalizeRequest(request);
        if (error != null)
            return Task.FromException<HttpResponseMessage>(error);
 
        return handler.SendAsync(request, cancellationToken);
    }
}
//------------------------------------Ʌ

//---------------------------------------------V
internal abstract class HttpMessageHandlerStage : HttpMessageHandler
{
    protected internal sealed override HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        ValueTask<HttpResponseMessage> sendTask = SendAsync(request, async: false, cancellationToken);
        
        return sendTask.IsCompleted ? sendTask.Result : sendTask.AsTask().GetAwaiter().GetResult();
    }
 
    protected internal sealed override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken) 
        => SendAsync(request, async: true, cancellationToken).AsTask();
 
    internal abstract ValueTask<HttpResponseMessage> SendAsync(HttpRequestMessage request, bool async, CancellationToken cancellationToken);  
}
//---------------------------------------------Ʌ

internal sealed class HttpConnectionPoolManager : IDisposable
{
    private readonly TimeSpan _cleanPoolTimeout;  // how frequently an operation should be initiated to clean out old pools and connections in those pools
    private readonly ConcurrentDictionary<HttpConnectionKey, HttpConnectionPool> _pools;  // the pools, indexed by endpoint
    private readonly Timer? _cleaningTimer;  // timer used to initiate cleaning of the pools
    private readonly Timer? _heartBeatTimer;  // heart beat timer currently used for Http2 ping only
 
    private readonly HttpConnectionSettings _settings;
    private readonly IWebProxy? _proxy;
    private readonly ICredentials? _proxyCredentials;

    private bool _timerIsRunning;  // keeps track of whether or not the cleanup timer is running. It helps us avoid the expensive ConcurrentDictionary{TKey,TValue}.IsEmpty call
    private object SyncObj => _pools;  // object used to synchronize access to state in the pool

    public HttpConnectionPoolManager(HttpConnectionSettings settings)
    {
        _settings = settings;
        _pools = new ConcurrentDictionary<HttpConnectionKey, HttpConnectionPool>();

        // ...
    }
    // ...
    public ValueTask<HttpResponseMessage> SendAsyncCore(HttpRequestMessage request, Uri? proxyUri, bool async, bool doRequestAuth, bool isProxyConnect, CancellationToken cancellationToken)
    {
        HttpConnectionKey key = GetConnectionKey(request, proxyUri, isProxyConnect);
 
        HttpConnectionPool? pool;
        while (!_pools.TryGetValue(key, out pool))
        {
            pool = new HttpConnectionPool(this, key.Kind, key.Host, key.Port, key.SslHostName, key.ProxyUri);
 
            if (_cleaningTimer == null)
            {
                // there's no cleaning timer, which means we're not adding connections into pools, but we still need the pool object for this request.  We don't need or want to add the pool 
                // to the pools, though,  since we don't want it to sit there forever, which it would without the cleaning timer.
                break;
            }
 
            if (_pools.TryAdd(key, pool))
            {
                // we need to ensure the cleanup timer is running if it isn't already now that we added a new connection pool.
                lock (SyncObj)
                {
                    if (!_timerIsRunning)
                    {
                        SetCleaningTimer(_cleanPoolTimeout);
                    }
                }
                break;
            }
 
            // we created a pool and tried to add it to our pools, but some other thread got there before us. We don't need to Dispose the pool,
            // as that's only needed when it contains connections that need to be closed.
        }
 
        return pool.SendAsync(request, async, doRequestAuth, cancellationToken);
    }

    public ValueTask<HttpResponseMessage> SendAsync(HttpRequestMessage request, bool async, bool doRequestAuth, CancellationToken cancellationToken)
    {
        if (_proxy == null)
            return SendAsyncCore(request, null, async, doRequestAuth, isProxyConnect: false, cancellationToken);
 
        // do proxy lookup.
        Uri? proxyUri = null;
        try
        {
            if (!_proxy.IsBypassed(request.RequestUri))
            {
                if (_proxy is IMultiWebProxy multiWebProxy)
                {
                    MultiProxy multiProxy = multiWebProxy.GetMultiProxy(request.RequestUri);
 
                    if (multiProxy.ReadNext(out proxyUri, out bool isFinalProxy) && !isFinalProxy)
                    {
                        return SendAsyncMultiProxy(request, async, doRequestAuth, multiProxy, proxyUri, cancellationToken);
                    }
                }
                else
                {
                    proxyUri = _proxy.GetProxy(request.RequestUri);
                }
            }
        }
        catch (Exception ex)
        {
            // eat any exception from the IWebProxy and just treat it as no proxy. This matches the behavior of other handlers.
            if (NetEventSource.Log.IsEnabled()) NetEventSource.Error(this, $"Exception from {_proxy.GetType().Name}.GetProxy({request.RequestUri}): {ex}");
        }
 
        if (proxyUri != null && !HttpUtilities.IsSupportedProxyScheme(proxyUri.Scheme))
        {
            throw new NotSupportedException(SR.net_http_invalid_proxy_scheme);
        }
 
        return SendAsyncCore(request, proxyUri, async, doRequestAuth, isProxyConnect: false, cancellationToken);
    }
}
```


```C#
//--------------------------------->>
public interface IHttpClientFactory
{
    HttpClient CreateClient(string name);
}
//---------------------------------<<

//--------------------------------------------------------------V
public static class HttpClientFactoryServiceCollectionExtensions
{
    public static IServiceCollection AddHttpClient(this IServiceCollection services)
    {
        services.AddLogging();
        services.AddOptions();
        services.AddMetrics();
 
        services.TryAddTransient<HttpMessageHandlerBuilder, DefaultHttpMessageHandlerBuilder>();
        services.TryAddSingleton<DefaultHttpClientFactory>();
        services.TryAddSingleton<IHttpClientFactory>(serviceProvider => serviceProvider.GetRequiredService<DefaultHttpClientFactory>());
        services.TryAddSingleton<IHttpMessageHandlerFactory>(serviceProvider => serviceProvider.GetRequiredService<DefaultHttpClientFactory>());
 
        // Typed Clients
        services.TryAdd(ServiceDescriptor.Transient(typeof(ITypedHttpClientFactory<>), typeof(DefaultTypedHttpClientFactory<>)));
        services.TryAdd(ServiceDescriptor.Singleton(typeof(DefaultTypedHttpClientFactory<>.Cache), typeof(DefaultTypedHttpClientFactory<>.Cache)));
 
        // Misc infrastructure
        services.TryAddEnumerable(ServiceDescriptor.Singleton<IHttpMessageHandlerBuilderFilter, LoggingHttpMessageHandlerBuilderFilter>());
        services.TryAddEnumerable(ServiceDescriptor.Singleton<IHttpMessageHandlerBuilderFilter, MetricsFactoryHttpMessageHandlerFilter>());
 
        // This is used to track state and report errors **DURING** service registration. This has to be an instance because we access it by reaching into the service collection.
        services.TryAddSingleton(new HttpClientMappingRegistry());
 
        // This is used to store configuration for the default builder.
        services.TryAddSingleton(new DefaultHttpClientConfigurationTracker());
 
        // Register default client as HttpClient
        services.TryAddTransient(s =>
        {
            return s.GetRequiredService<IHttpClientFactory>().CreateClient(string.Empty);  // <----------------------------------------------
        });
 
        return services;
    }
}
//--------------------------------------------------------------Ʌ
```

```C#
//--------------------------------->>
public interface IHttpClientFactory
{
    HttpClient CreateClient(string name);
}
//---------------------------------<<

//----------------------------------------->>
public interface IHttpMessageHandlerFactory
{
    HttpMessageHandler CreateHandler(string name);
}
//-----------------------------------------<<

//-------------------------------------V
internal class DefaultHttpClientFactory : IHttpClientFactory, IHttpMessageHandlerFactory
{
    private static readonly TimerCallback _cleanupCallback = (s) => ((DefaultHttpClientFactory)s!).CleanupTimer_Tick();
    private readonly IServiceProvider _services;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IOptionsMonitor<HttpClientFactoryOptions> _optionsMonitor;
    private readonly IHttpMessageHandlerBuilderFilter[] _filters;
    private readonly Func<string, Lazy<ActiveHandlerTrackingEntry>> _entryFactory;
    private readonly Lazy<ILogger> _logger;
 
    // default time of 10s for cleanup seems reasonable. Quick math: 
    // 10 distinct named clients * expiry time >= 1s = approximate cleanup queue of 100 items, this seems frequent enough
    private readonly TimeSpan DefaultCleanupInterval = TimeSpan.FromSeconds(10);
 
    // we use a new timer for each regular cleanup cycle, protected with a lock. Note that this scheme
    // doesn't give us anything to dispose, as the timer is started/stopped as needed.
    // there's no need for the factory itself to be disposable. If you stop using it, eventually everything will get reclaimed.
    private Timer? _cleanupTimer;
    private readonly object _cleanupTimerLock;
    private readonly object _cleanupActiveLock;
 
    // collection of 'active' handlers. Using lazy for synchronization to ensure that only one instance of HttpMessageHandler is created for each name. internal for tests
    internal readonly ConcurrentDictionary<string, Lazy<ActiveHandlerTrackingEntry>> _activeHandlers;
 
    // collection of 'expired' but not yet disposed handlers, used when we're rotating handlers so that we can dispose HttpMessageHandler instances once they
    // are eligible for garbage collection, internal for tests
    internal readonly ConcurrentQueue<ExpiredHandlerTrackingEntry> _expiredHandlers;
    private readonly TimerCallback _expiryCallback;

    public DefaultHttpClientFactory(
        IServiceProvider services,
        IServiceScopeFactory scopeFactory,
        IOptionsMonitor<HttpClientFactoryOptions> optionsMonitor,
        IEnumerable<IHttpMessageHandlerBuilderFilter> filters)
    {
        _services = services;
        _scopeFactory = scopeFactory;
        _optionsMonitor = optionsMonitor;
        _filters = filters.ToArray();
 
        // case-sensitive because named options is.
        _activeHandlers = new ConcurrentDictionary<string, Lazy<ActiveHandlerTrackingEntry>>(StringComparer.Ordinal);
        _entryFactory = (name) =>
        {
            return new Lazy<ActiveHandlerTrackingEntry>(() =>
            {
                return CreateHandlerEntry(name);
            }, LazyThreadSafetyMode.ExecutionAndPublication);
        };
 
        _expiredHandlers = new ConcurrentQueue<ExpiredHandlerTrackingEntry>();
        _expiryCallback = ExpiryTimer_Tick;
 
        _cleanupTimerLock = new object();
        _cleanupActiveLock = new object();
 
        // we want to prevent a circular dependency between ILoggerFactory and IHttpClientFactory, 
        // in case any of ILoggerProvider instances use IHttpClientFactory to send logs to an external server.
        // Logger will be created during the first ExpiryTimer_Tick execution. Lazy guarantees thread safety
        // to prevent creation of unnecessary ILogger objects in case several handlers expired at the same time.
        _logger = new Lazy<ILogger>(
            () => _services.GetRequiredService<ILoggerFactory>().CreateLogger<DefaultHttpClientFactory>(),
            LazyThreadSafetyMode.ExecutionAndPublication);
    }

    public HttpClient CreateClient(string name)
    { 
        HttpMessageHandler handler = CreateHandler(name);
        var client = new HttpClient(handler, disposeHandler: false);
 
        HttpClientFactoryOptions options = _optionsMonitor.Get(name);
        for (int i = 0; i < options.HttpClientActions.Count; i++)
        {
            options.HttpClientActions[i](client);
        }
 
        return client;
    }

    public HttpMessageHandler CreateHandler(string name)
    { 
        ActiveHandlerTrackingEntry entry = _activeHandlers.GetOrAdd(name, _entryFactory).Value;
 
        StartHandlerEntryTimer(entry);
 
        return entry.Handler;
    }

    // internal for tests
    internal ActiveHandlerTrackingEntry CreateHandlerEntry(string name)
    {
        IServiceProvider services = _services;
        var scope = (IServiceScope?)null;
 
        HttpClientFactoryOptions options = _optionsMonitor.Get(name);
        if (!options.SuppressHandlerScope)
        {
            scope = _scopeFactory.CreateScope();
            services = scope.ServiceProvider;
        }
 
        try
        {
            HttpMessageHandlerBuilder builder = services.GetRequiredService<HttpMessageHandlerBuilder>();
            builder.Name = name;
 
            Action<HttpMessageHandlerBuilder> configure = Configure;
            for (int i = _filters.Length - 1; i >= 0; i--)
            {
                configure = _filters[i].Configure(configure);
            }
 
            configure(builder);
 
            // wrap the handler so we can ensure the inner handler outlives the outer handler.
            var handler = new LifetimeTrackingHttpMessageHandler(builder.Build());
 
            // note that we can't start the timer here. That would introduce a very very subtle race condition
            // with very short expiry times. We need to wait until we've actually handed out the handler once to start the timer.
            // otherwise it would be possible that we start the timer here, immediately expire it (very short  timer) and then dispose it without ever creating a client
            // that would be bad. it's unlikely this would happen, but we want to be sure
            return new ActiveHandlerTrackingEntry(name, handler, scope, options.HandlerLifetime);
 
            void Configure(HttpMessageHandlerBuilder b)
            {
                for (int i = 0; i < options.HttpMessageHandlerBuilderActions.Count; i++)
                {
                    options.HttpMessageHandlerBuilderActions[i](b);
                }
 
                // Logging is added separately in the end. But for now it should be still possible to override it via filters...
                foreach (Action<HttpMessageHandlerBuilder> action in options.LoggingBuilderActions)
                {
                    action(b);
                }
            }
        }
        catch
        {           
            scope?.Dispose();  // if something fails while creating the handler, dispose the services.
            throw;
        }
    }

    // internal for tests
    internal void ExpiryTimer_Tick(object? state)
    {
        var active = (ActiveHandlerTrackingEntry)state!;
 
        // the timer callback should be the only one removing from the active collection. If we can't find  our entry in the collection, then this is a bug.
        bool removed = _activeHandlers.TryRemove(active.Name, out Lazy<ActiveHandlerTrackingEntry>? found);
 
        // at this point the handler is no longer 'active' and will not be handed out to any new clients. However we haven't dropped our strong reference to the handler,
        // so we can't yet determine if  there are still any other outstanding references (we know there is at least one).
        // we use a different state object to track expired handlers. This allows any other thread that acquired the 'active' entry to use it without safety problems.
        var expired = new ExpiredHandlerTrackingEntry(active);
        _expiredHandlers.Enqueue(expired);
 
        StartCleanupTimer();
    }

     // internal so it can be overridden in tests
    internal virtual void StartHandlerEntryTimer(ActiveHandlerTrackingEntry entry)
    {
        entry.StartExpiryTimer(_expiryCallback);
    }
 
    // internal so it can be overridden in tests
    internal virtual void StartCleanupTimer()
    {
        lock (_cleanupTimerLock)
        {
            _cleanupTimer ??= NonCapturingTimer.Create(_cleanupCallback, this, DefaultCleanupInterval, Timeout.InfiniteTimeSpan);
        }
    }
 
    // internal so it can be overridden in tests
    internal virtual void StopCleanupTimer()
    {
        lock (_cleanupTimerLock)
        {
            _cleanupTimer!.Dispose();
            _cleanupTimer = null;
        }
    }

    internal void CleanupTimer_Tick();
}
//-------------------------------------Ʌ

//---------------------------------------------V
public abstract class HttpMessageHandlerBuilder
{
    public abstract string? Name { get; set; }
    public abstract HttpMessageHandler PrimaryHandler { get; set; }
    public abstract IList<DelegatingHandler> AdditionalHandlers { get; }
    public virtual IServiceProvider Services { get; } = null!;

    public abstract HttpMessageHandler Build();

    protected internal static HttpMessageHandler CreateHandlerPipeline(HttpMessageHandler primaryHandler, IEnumerable<DelegatingHandler> additionalHandlers)
    {
        IReadOnlyList<DelegatingHandler> additionalHandlersList = additionalHandlers as IReadOnlyList<DelegatingHandler> ?? additionalHandlers.ToArray();
 
        HttpMessageHandler next = primaryHandler;
        for (int i = additionalHandlersList.Count - 1; i >= 0; i--)
        {
            DelegatingHandler handler = additionalHandlersList[i];  // <--------------------------------
            if (handler == null)
            {
                throw new InvalidOperationException("...");
            }
 
            // checking for this allows us to catch cases where someone has tried to re-use a handler. That really won't
            // work the way you want and it can be tricky for callers to figure out.
            if (handler.InnerHandler != null)
            {
                throw new InvalidOperationException("...");
            }
 
            handler.InnerHandler = next;
            next = handler;
        }
 
        return next;
    }
}
//---------------------------------------------Ʌ

//----------------------------------------------------V
internal sealed class DefaultHttpMessageHandlerBuilder : HttpMessageHandlerBuilder
{
    public DefaultHttpMessageHandlerBuilder(IServiceProvider services)
    {
        Services = services;
    }
 
    private string? _name;
 
    public override string? Name
    {
        get => _name;
        set {
            _name = value;
        }
    }

    public override HttpMessageHandler PrimaryHandler { get; set; } = new HttpClientHandler();

    public override IList<DelegatingHandler> AdditionalHandlers { get; } = new List<DelegatingHandler>();
 
    public override IServiceProvider Services { get; }
 
    public override HttpMessageHandler Build()
    {
        if (PrimaryHandler == null)
        {             
            throw new InvalidOperationException("...");
        }
 
        return CreateHandlerPipeline(PrimaryHandler, AdditionalHandlers);
    }
}
//----------------------------------------------------Ʌ

//-----------------------------------V
public class HttpClientFactoryOptions
{
    internal static readonly TimeSpan MinimumHandlerLifetime = TimeSpan.FromSeconds(1);
 
    private TimeSpan _handlerLifetime = TimeSpan.FromMinutes(2);  // <-------------------------------

    public IList<Action<HttpMessageHandlerBuilder>> HttpMessageHandlerBuilderActions { get; } = new List<Action<HttpMessageHandlerBuilder>>();

    public IList<Action<HttpClient>> HttpClientActions { get; } = new List<Action<HttpClient>>();

    public TimeSpan HandlerLifetime { get; set; }

    public Func<string, bool> ShouldRedactHeaderValue { get; set; } = (header) => false;

    public bool SuppressHandlerScope { get; set; }
 
    internal bool SuppressDefaultLogging { get; set; }
    internal List<Action<HttpMessageHandlerBuilder>> LoggingBuilderActions { get; } = new List<Action<HttpMessageHandlerBuilder>>(); 
}

//-----------------------------------Ʌ
```


```C#
//-----------------------------V
public partial class HttpClient : HttpMessageInvoker
{
    private static IWebProxy? s_defaultProxy;
    private static readonly TimeSpan s_defaultTimeout = TimeSpan.FromSeconds(100);
    private static readonly TimeSpan s_maxTimeout = TimeSpan.FromMilliseconds(int.MaxValue);
    private static readonly TimeSpan s_infiniteTimeout = Threading.Timeout.InfiniteTimeSpan;
    private const HttpCompletionOption DefaultCompletionOption = HttpCompletionOption.ResponseContentRead;
 
    private volatile bool _operationStarted;
    private volatile bool _disposed;
 
    private CancellationTokenSource _pendingRequestsCts;
    private HttpRequestHeaders? _defaultRequestHeaders;
    private Version _defaultRequestVersion = HttpRequestMessage.DefaultRequestVersion;
    private HttpVersionPolicy _defaultVersionPolicy = HttpRequestMessage.DefaultVersionPolicy;
 
    private Uri? _baseAddress;
    private TimeSpan _timeout;
    private int _maxResponseContentBufferSize;

    public HttpRequestHeaders DefaultRequestHeaders => _defaultRequestHeaders ??= new HttpRequestHeaders();
    public static IWebProxy DefaultProxy { get; set; }
    public Version DefaultRequestVersion { get; set; }
    public HttpVersionPolicy DefaultVersionPolicy { get; set; }
    public Uri? BaseAddress { get; set; }
    public TimeSpan Timeout { get; set; }
    public long MaxResponseContentBufferSize { get; set; }

    public HttpClient() : this(new HttpClientHandler()) { }

    public HttpClient(HttpMessageHandler handler) : this(handler, true) { }

    public HttpClient(HttpMessageHandler handler, bool disposeHandler) : base(handler, disposeHandler)
    {
        _timeout = s_defaultTimeout;
        _maxResponseContentBufferSize = HttpContent.MaxBufferSize;
        _pendingRequestsCts = new CancellationTokenSource();
    }

    public Task<string> GetStringAsync(Uri? requestUri, CancellationToken cancellationToken)
    {
        HttpRequestMessage request = CreateRequestMessage(HttpMethod.Get, requestUri);
 
        // called outside of async state machine to propagate certain exception even without awaiting the returned task.
        CheckRequestBeforeSend(request);
 
        return GetStringAsyncCore(request, cancellationToken);
    }

    private async Task<string> GetStringAsyncCore(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        bool telemetryStarted = StartSend(request);
        bool responseContentTelemetryStarted = false;
 
        (CancellationTokenSource cts, bool disposeCts, CancellationTokenSource pendingRequestsCts) = PrepareCancellationTokenSource(cancellationToken);
        HttpResponseMessage? response = null;

        try 
        {
             // wait for the response message and make sure it completed successfully.
            response = await base.SendAsync(request, cts.Token).ConfigureAwait(false);
            ThrowForNullResponse(response);
            response.EnsureSuccessStatusCode();
 
            // get the response content.
            HttpContent c = response.Content;
            if (HttpTelemetry.Log.IsEnabled() && telemetryStarted)
            {
                HttpTelemetry.Log.ResponseContentStart();
                responseContentTelemetryStarted = true;
            }

            // since the underlying byte[] will never be exposed, we use an ArrayPool-backed stream to which we copy all of the data from the response.
            using Stream responseStream = c.TryReadAsStream() ?? await c.ReadAsStreamAsync(cts.Token).ConfigureAwait(false);
            using var buffer = new HttpContent.LimitArrayPoolWriteStream(_maxResponseContentBufferSize, (int)c.Headers.ContentLength.GetValueOrDefault());
 
            try
            {
                await responseStream.CopyToAsync(buffer, cts.Token).ConfigureAwait(false);
            }
            catch (Exception e) when (HttpContent.StreamCopyExceptionNeedsWrapping(e))
            {
                throw HttpContent.WrapStreamCopyException(e);
            }
 
            if (buffer.Length > 0)
            {
                // decode and return the data from the buffer.
                    return HttpContent.ReadBufferAsString(buffer.GetBuffer(), c.Headers);
            }
 
            // no content to return.
            return string.Empty;
        }
        catch (Exception e)
        {
            HandleFailure(e, telemetryStarted, response, cts, cancellationToken, pendingRequestsCts);
            throw;
        }
        finally
        {
            FinishSend(response, cts, disposeCts, telemetryStarted, responseContentTelemetryStarted);
        }
    }

    // ...
}
//-----------------------------Ʌ

//-----------------------------V
public class HttpMessageInvoker : IDisposable
{
    private volatile bool _disposed;
    private readonly bool _disposeHandler;
    private readonly HttpMessageHandler _handler;
 
    public HttpMessageInvoker(HttpMessageHandler handler) : this(handler, true) { }

    public HttpMessageInvoker(HttpMessageHandler handler, bool disposeHandler)
    { 
        _handler = handler;
        _disposeHandler = disposeHandler;
    }

    public virtual HttpResponseMessage Send(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        if (ShouldSendWithTelemetry(request))
        {
            HttpTelemetry.Log.RequestStart(request);
 
            HttpResponseMessage? response = null;
            try
            {
                response = _handler.Send(request, cancellationToken);
                return response;
            }
            catch (Exception ex) when (LogRequestFailed(ex, telemetryStarted: true))
            {
                // unreachable as LogRequestFailed will return false
                throw;
            }
            finally
            {
                HttpTelemetry.Log.RequestStop(response);
            }
        }
        else
        {
            return _handler.Send(request, cancellationToken);
        }
    }

    public virtual Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken);

    // ...
}
//-----------------------------Ʌ
```