Middlwares Source Code
==============================

A. `StaticFileMiddleware`
B. `ExceptionHandlerMiddleware`
C. `CookiePolicyMiddleware`
D. `SessionMiddleware`


## A-StaticFileMiddleware
-------------------------
```C#
public class Startup {

   public void ConfigureServices(IServiceCollection services) {  

   }

   public void Configure(IApplicationBuilder app, IWebHostEnvironment env) {
      if (env.IsDevelopment()) {
         app.UseDeveloperExceptionPage();
      }
                        
      app.UseStaticFiles(new StaticFileOptions {   // <--------------------------------------------------------
         // ContentRootPath is the project path
         FileProvider = new PhysicalFileProvider($"{env.ContentRootPath}/staticfiles"),  // overwrite ContentRootPath so it doesn't need to be wwwroot
         RequestPath = "/files"  // match urls such as http://localhost:5000/files/hello.html
      });

      app.UseRouting();
      app.UseEndpoints(endpoints => {   
         endpoints.MapGet("/", async context => {   
            await context.Response.WriteAsync("Hello World!");        
      });
   }
}
```
```C#
//--------------------------------------V
public static class StaticFileExtensions {
   // ...
   public static IApplicationBuilder UseStaticFiles(this IApplicationBuilder app) {  
      return app.UseMiddleware<StaticFileMiddleware>();
   }

   public static IApplicationBuilder UseStaticFiles(this IApplicationBuilder app, StaticFileOptions options) {
      return app.UseMiddleware<StaticFileMiddleware>(Options.Create(options));
   }
}
//-------------------------------------Ʌ
//-------------------------------V
public class StaticFileMiddleware {    // <------------------------------------a
   private readonly StaticFileOptions _options;
   private readonly PathString _matchUrl;
   private readonly RequestDelegate _next;
   private readonly ILogger _logger;
   private readonly IFileProvider _fileProvider;
   private readonly IContentTypeProvider _contentTypeProvider; 

   public StaticFileMiddleware(RequestDelegate next, IWebHostEnvironment hostingEnv, IOptions<StaticFileOptions> options, ILoggerFactory loggerFactory) {
      _next = next;
      _options = options.Value;
      _contentTypeProvider = _options.ContentTypeProvider ?? new FileExtensionContentTypeProvider();
      _fileProvider = _options.FileProvider ?? Helpers.ResolveFileProvider(hostingEnv);
      _matchUrl = _options.RequestPath;
      _logger = loggerFactory.CreateLogger<StaticFileMiddleware>();
   }

   public Task Invoke(HttpContext context) {
      if (!ValidateNoEndpoint(context)) {
         _logger.EndpointMatched();
      } 
      else if (!ValidateMethod(context)) {
         _logger.RequestMethodNotSupported(context.Request.Method);
      }
      else if (!ValidatePath(context, _matchUrl, out var subPath)) {
         _logger.PathMismatch(subPath);
      }
      else if (!LookupContentType(_contentTypeProvider, _options, subPath, out var contentType)) {
         _logger.FileTypeNotSupported(subPath);
      }
      else {
         // If we get here, we can try to serve the file
         return TryServeStaticFile(context, contentType, subPath);
      }

      return _next(context);
   }
   // ...  
}
//-------------------------------Ʌ
//----------------------------V
public class StaticFileOptions : SharedOptionsBase {
   internal static readonly Action<StaticFileResponseContext> _defaultOnPrepareResponse = _ => { };
   public StaticFileOptions() : this(new SharedOptions()) {

   }
   // ...
   public IContentTypeProvider ContentTypeProvider { get; set; } = default!;
   public string? DefaultContentType { get; set; }
   public bool ServeUnknownFileTypes { get; set; }
   public HttpsCompressionMode HttpsCompression { get; set; } = HttpsCompressionMode.Compress;
   // called after the status code and headers have been set, but before the body has been written, can be used to add or change the response headers
   public Action<StaticFileResponseContext> OnPrepareResponse { get; set; }
}
//----------------------------Ʌ
//----------------------------V
public class SharedOptions {
   private PathString _requestPath;
   public SharedOptions() {
      RequestPath = PathString.Empty;
   }

   public PathString RequestPath { get; set; }
   public IFileProvider? FileProvider { get; set; }
   public bool RedirectToAppendTrailingSlash { get; set; } = true;
}
//----------------------------Ʌ
```

## B-ExceptionHandlerMiddleware 
-------------------------------
```C#
public class Startup {

   public void ConfigureServices(IServiceCollection services) {  
   }

   public void Configure(IApplicationBuilder app, IWebHostEnvironment env) {
      if (env.IsProduction()) {
         app.UseExceptionHandler("/error.html");  // <------------------
      } else {
         app.UseDeveloperExceptionPage();  // we don't want to use it in production as we want to provide an custom error page for users
      }
                              
      app.UseRouting();
      app.UseEndpoints(endpoints => {   
         endpoints.MapGet("/", async context => {   
            await context.Response.WriteAsync("Hello World!");        
      });
   }
}
```
When using an exception thrown, `ExceptionHandlerMiddleware` (need to be place in very beginning of request pipeline) will capture it and change the current request's url and then re-execute the pipleline (4.4), so there is no brownser redirection involved, only one http request with request pipeline executed twice.
```C#
//-------------------------------------------VV
public static class ExceptionHandlerExtensions {
   //...
   public static IApplicationBuilder UseExceptionHandler(this IApplicationBuilder app, string errorHandlingPath) {
      return app.UseExceptionHandler(new ExceptionHandlerOptions 
      {
         ExceptionHandlingPath = new PathString(errorHandlingPath)
      });
   }

   public static IApplicationBuilder UseExceptionHandler(this IApplicationBuilder app, ExceptionHandlerOptions options) {
      OptionsWrapper<ExceptionHandlerOptions> iOptions = Options.Create(options);
      return app.UseMiddleware<ExceptionHandlerMiddleware>(options);  // *
   }
}

public class ExceptionHandlerOptions {
   public PathString ExceptionHandlingPath { get; set; }
   public RequestDelegate? ExceptionHandler { get; set; }
   public bool AllowStatusCode404Response { get; set; }  // controls whether the ExceptionHandlerMiddleware should consider 404 status response code to be a valid result of
                                                         // executing the ExceptionHandler; The default value is false, server and will therefore rethrow the original exception
}
//-------------------------------------------ɅɅ
//-------------------------------------V
public class ExceptionHandlerMiddleware 
{
   private readonly RequestDelegate _next;
   private readonly ExceptionHandlerOptions _options;
   private readonly Func<object, Task> _clearCacheHeadersDelegate;

   public ExceptionHandlerMiddleware(RequestDelegate next, IOptions<ExceptionHandlerOptions> options) 
   {
      _next = next;
      _options = options.Value;
      _clearCacheHeadersDelegate = ClearCacheHeaders;
      if (_options.ExceptionHandler == null) {
         if (_options.ExceptionHandlingPath == null) {
            throw new InvalidOperationException(Resources.ExceptionHandlerOptions_NotConfiguredCorrectly);
         }
         else 
         {
            _options.ExceptionHandler = _next;   <------------------1 : save the next RequestDelegate so it can be executed after catching the exception
         }
      }
   }

   public Task Invoke(HttpContext context) {
      try {
         var task = _next(context);
         if (!task.IsCompletedSuccessfully) {
            return Awaited(this, context, task);   <-----------------2.0
         }

         return Task.CompletedTask;
      }
      catch (Exception exception) {
         edi = ExceptionDispatchInfo.Capture(exception);
      }

      return HandleException(context, edi);

      static async Task Awaited(ExceptionHandlerMiddleware middleware, HttpContext context, Task task) {
         ExceptionDispatchInfo? edi = null;
         try {
            await task;   <--------------------------2.1
         }
         catch (Exception exception) {
            edi = ExceptionDispatchInfo.Capture(exception);   <------------------------3.1
         }

         if (edi != null) {
            await middleware.HandleException(context, edi);  <------------------------3.2
         }
      }
   }

   private async Task HandleException(HttpContext context, ExceptionDispatchInfo edi) {
      // we can't do anything if the response has already started, just abort
      if (context.Response.HasStarted) {
         edi.Throw();
      }

      PathString originalPath = context.Request.Path;
      if (_options.ExceptionHandlingPath.HasValue) {
         context.Request.Path = _options.ExceptionHandlingPath;  <------------------------4.1
      }
      try {
         var exceptionHandlerFeature = new ExceptionHandlerFeature() {
            Error = edi.SourceException,
            Path = originalPath.Value!,
            Endpoint = context.GetEndpoint(),
            RouteValues = context.Features.Get<IRouteValuesFeature>()?.RouteValues  <------------------------4.2
         }

         ClearHttpContext(context);

         context.Features.Set<IExceptionHandlerFeature>(exceptionHandlerFeature);
         context.Features.Set<IExceptionHandlerPathFeature>(exceptionHandlerFeature);
         context.Response.StatusCode = StatusCodes.Status500InternalServerError;   <------------------------4.3
         context.Response.OnStarting(_clearCacheHeadersDelegate, context.Response);

         await _options.ExceptionHandler!(context);   <-------------------------4.4

         // if the response has already started, assume exception handler was successful
         if (context.Response.HasStarted || context.Response.StatusCode != StatusCodes.Status404NotFound || _options.AllowStatusCode404Response) {
            return;
         }

         edi = ExceptionDispatchInfo.Capture(new InvalidOperationException($"The exception handler configured on ExceptionHandlerOptions produced a 404 status response. This InvalidOperationException containing the original exception was thrown since this is often due to a misconfigured ExceptionHandlerOptions.ExceptionHandlingPath. If the exception handler is expected to return 404 status responses then set ExceptionHandlerOptions.AllowStatusCode404Response to true.", edi.SourceException));
      }
      catch (Exception ex2) {
         //.. suppress secondary exceptions, re-throw the original.
      }

      edi.Throw(); // re-throw wrapped exception or the original if we couldn't handle it
   }

   private static void ClearHttpContext(HttpContext context) {
      context.Response.Clear();

      // an endpoint may have already been set, since we will re-invoke the middleware pipeline we need to reset the endpoint and route values to ensure things are re-calculated
      context.SetEndpoint(endpoint: null);
      var routeValuesFeature = context.Features.Get<IRouteValuesFeature>();
      if (routeValuesFeature != null) {
         routeValuesFeature.RouteValues = null!;
      }
   }
   
   private static Task ClearCacheHeaders(object state) {
      var headers = ((HttpResponse)state).Headers;
        headers.CacheControl = "no-cache,no-store";
        headers.Pragma = "no-cache";
        headers.Expires = "-1";
        headers.ETag = default;
        return Task.CompletedTask;
   }
}
//-------------------------------------Ʌ
```

## C-CookiePolicyMiddleware
----------------------------

```C#
public class Startup {
   public void ConfigureServices(IServiceCollection services) {
      services.Configure<CookiePolicyOptions>(opts => {  // <--------------B0
         opts.CheckConsentNeeded = context => true;
      });
   }

   public void Configure(IApplicationBuilder app, IWebHostEnvironment env) {
      if (env.IsDevelopment()) {
         app.UseDeveloperExceptionPage();
      }

      app.UseCookiePolicy(); // <------------------------------------------B1

      app.UseRouting();  
      // ...
   }
}
```
```C#
//--------------------------------------------------V
public static class CookiePolicyAppBuilderExtensions {
   
   public static IApplicationBuilder UseCookiePolicy(this IApplicationBuilder app) {
      return app.UseMiddleware<CookiePolicyMiddleware>();  // <------------------------------------------b1
   }

   public static IApplicationBuilder UseCookiePolicy(this IApplicationBuilder app, CookiePolicyOptions options) {
      return app.UseMiddleware<CookiePolicyMiddleware>(Options.Create(options));  // manually create new OptionsWrapper<TOptions>(options)
   }
}
//--------------------------------------------------Ʌ
//--------------------------------------------------V
public class CookiePolicyMiddleware {
   private readonly RequestDelegate _next;
   private readonly ILogger _logger;
                                                       // <-------------------------------b0
   public CookiePolicyMiddleware(RequestDelegate next, IOptions<CookiePolicyOptions> options, ILoggerFactory factory) {
      Options = options.Value;
      _next = next;
      _logger = factory.CreateLogger<CookiePolicyMiddleware>();
   }

   public CookiePolicyOptions Options { get; set; }

   public Task Invoke(HttpContext context) {
      var feature = context.Features.Get<IResponseCookiesFeature>() ?? new ResponseCookiesFeature(context.Features) // <-------------b2
      var wrapper = new ResponseCookiesWrapper(context, Options, feature, _logger);
      context.Features.Set<IResponseCookiesFeature>(new CookiesWrapperFeature(wrapper));
      context.Features.Set<ITrackingConsentFeature>(wrapper);

      return _next(context);
   }

   private class CookiesWrapperFeature : IResponseCookiesFeature {
      public CookiesWrapperFeature(ResponseCookiesWrapper wrapper) {
         Cookies = wrapper;
      }

      public IResponseCookies Cookies { get; }
   }
}
//--------------------------------------------------Ʌ
//--------------------------------------V
public interface IResponseCookiesFeature {  // a helper for creating the response Set-Cookie header
   IResponseCookies Cookies { get; }
}

public class ResponseCookiesFeature : IResponseCookiesFeature {
   private readonly IFeatureCollection _features;
   private IResponseCookies? _cookiesCollection;

   public ResponseCookiesFeature(IFeatureCollection features) {
      _features = features ?? throw new ArgumentNullException(nameof(features));
   }

   public IResponseCookies Cookies {
      get {
         if (_cookiesCollection == null) {
            _cookiesCollection = new ResponseCookies(_features);   // <-----------------------------b3
         }

         return _cookiesCollection;
      }
   }
}
//--------------------------------------Ʌ
//------------------------------VV
public interface IResponseCookies {
   void Append(string key, string value);
   void Append(string key, string value, CookieOptions options);
   void Delete(string key);
   void Delete(string key, CookieOptions options);
}

public interface ITrackingConsentFeature {
   bool IsConsentNeeded { get; }
   bool HasConsent { get; }
   bool CanTrack { get; }   // indicates either if consent has been given or if consent is not required
   void GrantConsent();
   void WithdrawConsent();
   string CreateConsentCookie();
}
//------------------------------ɅɅ
```
```C#
//------------------------------------V
internal partial class ResponseCookies : IResponseCookies {
   internal const string EnableCookieNameEncoding = "Microsoft.AspNetCore.Http.EnableCookieNameEncoding";
   internal bool _enableCookieNameEncoding = AppContext.TryGetSwitch(EnableCookieNameEncoding, out var enabled) && enabled;

   private readonly IFeatureCollection _features;
   private IHeaderDictionary Headers { get; set; }

   internal ResponseCookies(IFeatureCollection features) {
      _features = features;
      Headers = _features.GetRequiredFeature<IHttpResponseFeature>().Headers;
   }
   // ...
   public void Append(string key, string value, CookieOptions options) {
      var setCookieHeaderValue = new SetCookieHeaderValue(_enableCookieNameEncoding ? Uri.EscapeDataString(key) : key, Uri.EscapeDataString(value)) {
         Domain = options.Domain,
         Path = options.Path,
         Expires = options.Expires,
         MaxAge = options.MaxAge,
         Secure = options.Secure,
         SameSite = (Net.Http.Headers.SameSiteMode)options.SameSite,
         HttpOnly = options.HttpOnly
      };

      var cookieValue = setCookieHeaderValue.ToString();

      Headers.SetCookie = StringValues.Concat(Headers.SetCookie, cookieValue);
   }

   public void Delete(string key) {
      Delete(key, new CookieOptions() { Path = "/" });
   }

   public void Delete(string key, CookieOptions options) {
      // ...
   }
}
//------------------------------------Ʌ
```
```C#
//-----------------------------------V
internal class ResponseCookiesWrapper : IResponseCookies, ITrackingConsentFeature {
   private const string ConsentValue = "yes";
   private readonly ILogger _logger;
   private bool? _isConsentNeeded;
   private bool? _hasConsent;

   public ResponseCookiesWrapper(HttpContext context, CookiePolicyOptions options, IResponseCookiesFeature feature, ILogger logger) {
      Context = context;
      Feature = feature;
      Options = options;
      _logger = logger;
   }
   
   private HttpContext Context { get; }

   private IResponseCookiesFeature Feature { get; }

   private IResponseCookies Cookies => Feature.Cookies;

   private CookiePolicyOptions Options { get; }

   public bool IsConsentNeeded {
      get {
         if (!_isConsentNeeded.HasValue) {
            _isConsentNeeded = Options.CheckConsentNeeded == null ? false : Options.CheckConsentNeeded(Context);
         }
         return _isConsentNeeded.Value;
      }
   }

   public bool HasConsent {
      get {
         if (!_hasConsent.HasValue) {
            var cookie = Context.Request.Cookies[Options.ConsentCookie.Name!];
            _hasConsent = string.Equals(cookie, ConsentValue, StringComparison.Ordinal);
         }
         return _hasConsent.Value;
      }
   }

   public bool CanTrack => !IsConsentNeeded || HasConsent;

   public void GrantConsent() {
      if (!HasConsent && !Context.Response.HasStarted) {
         var cookieOptions = Options.ConsentCookie.Build(Context);
         // Note policy will be applied. We don't want to bypass policy because we want HttpOnly, Secure, etc. to apply.
         Append(Options.ConsentCookie.Name!, ConsentValue, cookieOptions);      
      }
      _hasConsent = true;
   }

   public void WithdrawConsent() {
      // ... same as above except using Delete(Options.ConsentCookie.Name!, cookieOptions);
   }

   public string CreateConsentCookie() {
      var key = Options.ConsentCookie.Name;
      var value = ConsentValue;
      var options = Options.ConsentCookie.Build(Context);

      ApplyAppendPolicy(ref key, ref value, options);

      var setCookieHeaderValue = new Net.Http.Headers.SetCookieHeaderValue(Uri.EscapeDataString(key), Uri.EscapeDataString(value)) {
         Domain = options.Domain,
         Path = options.Path,
         Expires = options.Expires,
         MaxAge = options.MaxAge,
         Secure = options.Secure,
         SameSite = (Net.Http.Headers.SameSiteMode)options.SameSite,
         HttpOnly = options.HttpOnly
      };

      return setCookieHeaderValue.ToString();
   }

   private bool CheckPolicyRequired() {
      return !CanTrack || Options.MinimumSameSitePolicy != SameSiteMode.Unspecified || Options.HttpOnly != HttpOnlyPolicy.None || Options.Secure != CookieSecurePolicy.None;
   }

   public void Append(string key, string value) {
      if (CheckPolicyRequired() || Options.OnAppendCookie != null) {
         Append(key, value, new CookieOptions());
      } else {
         Cookies.Append(key, value);
      }
   }

   public void Append(string key, string value, CookieOptions options) {
      if (ApplyAppendPolicy(ref key, ref value, options)) {
         Cookies.Append(key, value, options);
      }
   }

   public void Delete(string key) {
      if (CheckPolicyRequired() || Options.OnDeleteCookie != null) {
         Delete(key, new CookieOptions());
      } else {
         Cookies.Delete(key);
      }
   }

   public void Delete(string key, CookieOptions options) {
      // Assume you can always delete cookies unless directly overridden in the user event
      var issueCookie = true;
      ApplyPolicy(key, options);
      if (Options.OnDeleteCookie != null) {
         var context = new DeleteCookieContext(Context, options, key) {
            IsConsentNeeded = IsConsentNeeded,
            HasConsent = HasConsent,
            IssueCookie = issueCookie
         };
         Options.OnDeleteCookie(context);

         key = context.CookieName;
         issueCookie = context.IssueCookie;
      }

      if (issueCookie) {
         Cookies.Delete(key, options);
      }
   }

   private bool ApplyAppendPolicy(ref string key, ref string value, CookieOptions options) {
      var issueCookie = CanTrack || options.IsEssential;
      ApplyPolicy(key, options);
      if (Options.OnAppendCookie != null) {
         var context = new AppendCookieContext(Context, options, key, value) {
            IsConsentNeeded = IsConsentNeeded,
            HasConsent = HasConsent,
            IssueCookie = issueCookie,
         };
         Options.OnAppendCookie(context);

         key = context.CookieName;
         value = context.CookieValue;
         issueCookie = context.IssueCookie;
      }

      return issueCookie;
   }

   private void ApplyPolicy(string key, CookieOptions options) {
      switch (Options.Secure) {
         case CookieSecurePolicy.Always:
            if (!options.Secure) {
               options.Secure = true;           
            }
            break;
         case CookieSecurePolicy.SameAsRequest:
            // Never downgrade a cookie
            if (Context.Request.IsHttps && !options.Secure) {
               options.Secure = true;
            }
            break;
         case CookieSecurePolicy.None:
            break;
         default:
            throw new InvalidOperationException();
      }

      if (options.SameSite < Options.MinimumSameSitePolicy) {
         options.SameSite = Options.MinimumSameSitePolicy;
      }

      switch (Options.HttpOnly) {
         case HttpOnlyPolicy.Always:
            if (!options.HttpOnly) {
               options.HttpOnly = true;
            }
            break;
         case HttpOnlyPolicy.None:
            break;
         default:
            throw new InvalidOperationException($"Unrecognized {nameof(HttpOnlyPolicy)} value {Options.HttpOnly.ToString()}");
      }
   }
}
//-----------------------------------Ʌ
//------------------------------V
public class CookiePolicyOptions {
   public SameSiteMode MinimumSameSitePolicy { get; set; } = SameSiteMode.Unspecified;

   public HttpOnlyPolicy HttpOnly { get; set; } = HttpOnlyPolicy.None;

   public CookieSecurePolicy Secure { get; set; } = CookieSecurePolicy.None;

   public CookieBuilder ConsentCookie { get; set; } = new CookieBuilder() {
      Name = ".AspNet.Consent",
      Expiration = TimeSpan.FromDays(365),
      IsEssential = true
   };

   // checks if consent policies should be evaluated on this request. The default is false
   public Func<HttpContext, bool> CheckConsentNeeded { get; set; }

   // called when a cookie is appended
   public Action<AppendCookieContext>? OnAppendCookie { get; set; }

   // called when a cookie is deleted 
   public Action<DeleteCookieContext>? OnDeleteCookie { get; set; }
}
//------------------------------Ʌ
//------------------------V
public class CookieBuilder {
   private string? _name;

   public virtual string? Name {
      get => _name;
      set => _name = !string.IsNullOrEmpty(value) ? value : throw new ArgumentException(Resources.ArgumentCannotBeNullOrEmpty, nameof(value));
   }

   public virtual string? Path { get; set; }

   public virtual string? Domain { get; set; }

   public virtual bool HttpOnly { get; set; }

   public virtual SameSiteMode SameSite { get; set; } = SameSiteMode.Unspecified;

   public virtual CookieSecurePolicy SecurePolicy { get; set; }

   public virtual TimeSpan? Expiration { get; set; }

   public virtual TimeSpan? MaxAge { get; set; }

   public virtual bool IsEssential { get; set; }

   public CookieOptions Build(HttpContext context) => Build(context, DateTimeOffset.Now);

   public virtual CookieOptions Build(HttpContext context, DateTimeOffset expiresFrom) {
      return new CookieOptions {
         Path = Path ?? "/",
         SameSite = SameSite,
         HttpOnly = HttpOnly,
         MaxAge = MaxAge,
         Domain = Domain,
         IsEssential = IsEssential,
         Secure = SecurePolicy == CookieSecurePolicy.Always || (SecurePolicy == CookieSecurePolicy.SameAsRequest && context.Request.IsHttps),
         Expires = Expiration.HasValue ? expiresFrom.Add(Expiration.GetValueOrDefault()) : default(DateTimeOffset?)
      };
   }
}
```

## D-SessionMiddleware
----------------------

```C#
public class Startup {

   public void ConfigureServices(IServiceCollection services) 
   {  
      services.AddDistributedMemoryCache();  // we need it to store session

      services.AddSession(options => {  // <---------------------------
            options.IdleTimeout = TimeSpan.FromMinutes(30);          
            options.Cookie.IsEssential = true;  // session cookie isn't denoted as essential by default, which can cause problems when cookie consent is used
      });
   }

   public void Configure(IApplicationBuilder app, IWebHostEnvironment env) 
   {
      if (env.IsDevelopment()) {
         app.UseDeveloperExceptionPage();
      }                        
      
      app.UseCookiePolicy(); // session depends on cookies, so have to use CookiePolicyMiddleware
      app.UseSession();  // <---------------------------

      app.UseRouting();
      app.UseEndpoints(endpoints => {   
         endpoints.MapGet("/", async context => {   
            await context.Response.WriteAsync("Hello World!");        
      });
   }
}
```
```C#
public static class MemoryCacheServiceCollectionExtensions 
{
   public static IServiceCollection AddMemoryCache(this IServiceCollection services!!) {
      services.AddOptions();
      services.TryAdd(ServiceDescriptor.Singleton<IMemoryCache, MemoryCache>());
      
      return services;
   }

   public static IServiceCollection AddDistributedMemoryCache(this IServiceCollection services!!){
      services.AddOptions();
      services.TryAdd(ServiceDescriptor.Singleton<IDistributedCache, MemoryDistributedCache>());

      return services;
   }
}
/*  public interface IDistributedCache derived:

Microsoft.Extensions.Caching.Distributed.MemoryDistributedCache
Microsoft.Extensions.Caching.SqlServer.SqlServerCache
Microsoft.Extensions.Caching.Redis.RedisCache
Microsoft.Extensions.Caching.StackExchangeRedis.RedisCache
*/
//--------------------------------V
public interface IDistributedCache {
   byte[]? Get(string key);
   Task<byte[]?> GetAsync(string key, CancellationToken token = default(CancellationToken));
   void Set(string key, byte[] value, DistributedCacheEntryOptions options);
   Task SetAsync(string key, byte[] value, DistributedCacheEntryOptions options, CancellationToken token = default(CancellationToken));
   void Refresh(string key);
   Task RefreshAsync(string key, CancellationToken token = default(CancellationToken));
   void Remove(string key);
   Task RemoveAsync(string key, CancellationToken token = default(CancellationToken));
}

public class MemoryDistributedCache : IDistributedCache 
{
   private readonly IMemoryCache _memCache;

   public MemoryDistributedCache(IOptions<MemoryDistributedCacheOptions> optionsAccessor!!, ILoggerFactory loggerFactory!!) {
      _memCache = new MemoryCache(optionsAccessor.Value, loggerFactory);
   }

   public byte[]? Get(string key!!) {
      return (byte[]?)_memCache.Get(key);
   }

   public void Remove(string key!!) {
      _memCache.Remove(key);
   }

   public void Set(string key!!, byte[] value!!, DistributedCacheEntryOptions options!!) {
      var memoryCacheEntryOptions = new MemoryCacheEntryOptions();
      memoryCacheEntryOptions.AbsoluteExpiration = options.AbsoluteExpiration;
      memoryCacheEntryOptions.AbsoluteExpirationRelativeToNow = options.AbsoluteExpirationRelativeToNow;
      memoryCacheEntryOptions.SlidingExpiration = options.SlidingExpiration;
      memoryCacheEntryOptions.Size = value.Length;

      _memCache.Set(key, value, memoryCacheEntryOptions);
   }  
   // ...
}
//--------------------------------Ʌ
//---------------------------------------------------VVV
public static class SessionServiceCollectionExtensions 
{
   public static IServiceCollection AddSession(this IServiceCollection services, Action<SessionOptions> configure) {
      services.Configure(configure);
      services.AddSession();
      return services;
   }

   public static IServiceCollection AddSession(this IServiceCollection services) {
      services.TryAddTransient<ISessionStore, DistributedSessionStore>();
      services.AddDataProtection();
      return services;
   }
}
//---------------------------------------------------ɅɅɅ
public class SessionOptions 
{
   private CookieBuilder _cookieBuilder = new SessionCookieBuilder();

   public CookieBuilder Cookie {
      get => _cookieBuilder;
      set => _cookieBuilder = value ?? throw new ArgumentNullException(nameof(value));
   }

   // how long the session can be idle before its contents are abandoned. Each session access resets the timeout
   // note this only applies to the content of the session, not the cookie
   public TimeSpan IdleTimeout { get; set; } = TimeSpan.FromMinutes(20);
   
   // maximum amount of time allowed to load a session from the store or to commit it back to the store
   public TimeSpan IOTimeout { get; set; } = TimeSpan.FromMinutes(1);

   private class SessionCookieBuilder : CookieBuilder {
      public SessionCookieBuilder() {
         Name = SessionDefaults.CookieName;
         Path = SessionDefaults.CookiePath;
         SecurePolicy = CookieSecurePolicy.None;
         SameSite = SameSiteMode.Lax;
         HttpOnly = true;
         IsEssential = false;  // session is considered non-essential as it's designed for ephemeral data
      }

      public override TimeSpan? Expiration {
         get => null;
         set => throw new InvalidOperationException(nameof(Expiration) + " cannot be set for the cookie defined by " + nameof(SessionOptions));
      }
   }
}
//---------------------------------------------------ɅɅ
public static class SessionMiddlewareExtensions 
{
   public static IApplicationBuilder UseSession(this IApplicationBuilder app) {
      return app.UseMiddleware<SessionMiddleware>();
   }

   public static IApplicationBuilder UseSession(this IApplicationBuilder app, SessionOptions options) {
      return app.UseMiddleware<SessionMiddleware>(Options.Create(options));
   }
}
//---------------------------------------------------Ʌ
//----------------------------V
public class SessionMiddleware 
{
   private const int SessionKeyLength = 36; // "382c74c3-721d-4f34-80e5-57657b6cbc27"
   private static readonly Func<bool> ReturnTrue = () => true;
   private readonly RequestDelegate _next;
   private readonly SessionOptions _options;
   private readonly ILogger _logger;
   private readonly ISessionStore _sessionStore;
   private readonly IDataProtector _dataProtector;

   public SessionMiddleware(
        RequestDelegate next,
        ILoggerFactory loggerFactory,
        IDataProtectionProvider dataProtectionProvider,
        ISessionStore sessionStore,
        IOptions<SessionOptions> options)
   {
      _next = next;
      _logger = loggerFactory.CreateLogger<SessionMiddleware>();
      _dataProtector = dataProtectionProvider.CreateProtector(nameof(SessionMiddleware));
      _options = options.Value;
      _sessionStore = sessionStore;
   }

   public async Task Invoke(HttpContext context) 
   {
       var isNewSessionKey = false;
       Func<bool> tryEstablishSession = ReturnTrue;
       var cookieValue = context.Request.Cookies[_options.Cookie.Name!];
       var sessionKey = CookieProtection.Unprotect(_dataProtector, cookieValue, _logger);
       if (string.IsNullOrWhiteSpace(sessionKey) || sessionKey.Length != SessionKeyLength) 
       {
          // no valid cookie, new session
          var guidBytes = new byte[16];
          RandomNumberGenerator.Fill(guidBytes);
          sessionKey = new Guid(guidBytes).ToString();
          cookieValue = CookieProtection.Protect(_dataProtector, sessionKey);
          var establisher = new SessionEstablisher(context, cookieValue, _options);
          tryEstablishSession = establisher.TryEstablishSession;
          isNewSessionKey = true;
       }

       SessionFeature feature = new SessionFeature();
       feature.Session = _sessionStore.Create(sessionKey, _options.IdleTimeout, _options.IOTimeout, tryEstablishSession, isNewSessionKey);
       context.Features.Set<ISessionFeature>(feature);

       try 
       {
          await _next(context);
       }
       fially 
       {
          context.Features.Set<ISessionFeature?>(null);

          if (feature.Session != null)
          {
             try
             {
                await feature.Session.CommitAsync();
             }
             catch (OperationCanceledException)
             {
                _logger.SessionCommitCanceled();
             }
             catch (Exception ex)
             {
                _logger.ErrorClosingTheSession(ex);
             }
          }
       }
   }

   private class SessionEstablisher
   {
      private readonly HttpContext _context;
      private readonly string _cookieValue;
      private readonly SessionOptions _options;
      private bool _shouldEstablishSession;

      public SessionEstablisher(HttpContext context, string cookieValue, SessionOptions options)
      {
         _context = context;
         _cookieValue = cookieValue;
         _options = options;
         context.Response.OnStarting(OnStartingCallback, state: this);
      }

      private static Task OnStartingCallback(object state)
      {
         var establisher = (SessionEstablisher)state;
         if (establisher._shouldEstablishSession)
         {
            establisher.SetCookie();
         }
         return Task.CompletedTask;
      }

      private void SetCookie()
      {
         var cookieOptions = _options.Cookie.Build(_context);

         var response = _context.Response;
         response.Cookies.Append(_options.Cookie.Name!, _cookieValue, cookieOptions);

         var responseHeaders = response.Headers;
         responseHeaders.CacheControl = "no-cache,no-store";
         responseHeaders.Pragma = "no-cache";
         responseHeaders.Expires = "-1";
      }

      internal bool TryEstablishSession()
      {
         return (_shouldEstablishSession |= !_context.Response.HasStarted);
      }
   }
}
//----------------------------Ʌ
```





<!-- <div class="alert alert-info p-1" role="alert">
    
</div> -->

<!-- ![alt text](./zImages/17-6.png "Title") -->

<!-- <code>&lt;T&gt;</code> -->

<!-- <div class="alert alert-info pt-2 pb-0" role="alert">
    <ul class="pl-1">
      <li></li>
      <li></li>
    </ul>  
</div> -->

<!-- <ul>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
</ul>  -->

<!-- <span style="color:red">hurt</span> -->

<style type="text/css">
.markdown-body {
  max-width: 1800px;
  margin-left: auto;
  margin-right: auto;
}
</style>

<link rel="stylesheet" href="./zCSS/bootstrap.min.css">
<script src="./zCSS/jquery-3.3.1.slim.min.js"></script>
<script src="./zCSS/popper.min.js"></script>
<script src="./zCSS/bootstrap.min.js"></script>