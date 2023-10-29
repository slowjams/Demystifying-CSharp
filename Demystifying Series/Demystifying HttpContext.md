Demystifying HttpContext
==============================

```C#
//-------------------------------V
public abstract class HttpContext {   // namespace Microsoft.AspNetCore.Http
   // ...
   public abstract HttpRequest Request { get; }
   public abstract HttpResponse Response { get; }
   public abstract ConnectionInfo Connection { get; }
   public abstract ISession Session { get; set; }
   public abstract void Abort();

   public abstract ClaimsPrincipal User { get; set; }

   public abstract IFeatureCollection Features { get; }   // allow access to the low-level aspects of request handling

   public abstract IDictionary<object, object> Items { get; set; }  // gets or sets a key/value collection that shares data within the scope of this request.

   public abstract IServiceProvider RequestServices { get; set; }   // gets or sets the IServiceProvider that can access to the request's service container.

   public abstract string TraceIdentifier { get; set; }
}
//-------------------------------Ʌ

//------------------------------------V
public sealed class DefaultHttpContext : HttpContext  // in a nutshell, HttpContext stores everything like RouteValues in a FeatureCollection instance which is managed 
{                                                     // FeatureReferences<FeatureInterfaces>, that's how you can access HttpContext.Features.              
   private const int DefaultFeatureCollectionSize = 10;   //based on the number of most common features
   
   private static readonly Func<IFeatureCollection, IItemsFeature> _newItemsFeature = f => new ItemsFeature();
   private static readonly Func<DefaultHttpContext, IServiceProvidersFeature> _newServiceProvidersFeature 
      = context => new RequestServicesFeature(context, context.ServiceScopeFactory);
   private static readonly Func<IFeatureCollection, IHttpAuthenticationFeature> _newHttpAuthenticationFeature = f => new HttpAuthenticationFeature();
   private static readonly Func<IFeatureCollection, IHttpRequestLifetimeFeature> _newHttpRequestLifetimeFeature = f => new HttpRequestLifetimeFeature();
   private static readonly Func<IFeatureCollection, ISessionFeature> _newSessionFeature = f => new DefaultSessionFeature();
   private static readonly Func<IFeatureCollection, ISessionFeature?> _nullSessionFeature = f => null;
   private static readonly Func<IFeatureCollection, IHttpRequestIdentifierFeature> _newHttpRequestIdentifierFeature = f => new HttpRequestIdentifierFeature();
     
   private readonly DefaultHttpRequest _request;
   private readonly DefaultHttpResponse _response;
   
   // ...
   public DefaultHttpContext() : this(new FeatureCollection(DefaultFeatureCollectionSize)) {  // <----------------------a1.0

      Features.Set<IHttpRequestFeature>(new HttpRequestFeature());     // <<<--------------------------a3

      Features.Set<IHttpResponseFeature>(new HttpResponseFeature());

      Features.Set<IHttpResponseBodyFeature>(new StreamResponseBodyFeature(Stream.Null));     
   }

   public DefaultHttpContext(IFeatureCollection features) {   // <----------------------a1.1
      _features.Initalize(features);              // <----------------------------------a2.1
      _request = new DefaultHttpRequest(this);    // <----------------------------------a4
      _response = new DefaultHttpResponse(this);  // <----------------------------------a4
   }
   
   public IServiceScopeFactory ServiceScopeFactory { get; set; } = default!;

   // both of FeatureReferences and FeatureInterfaces are struct
   private FeatureReferences<FeatureInterfaces> _features;    // <-------a2.1--------->   

   /*
      The reason we store everything such as IHttpRequestFeature, IRouteValuesFeature etc into a IFeatureCollection instead of defining property directly like 
      `RouteValueDictionary RouteValues { get; set; }` on HttpContext is probably we need extensibility
   */
   public override IFeatureCollection Features => _features.Collection;   // <---------a3.0---------->

   public override HttpRequest Request => _request;
   public override HttpResponse Response => _response;

   private IItemsFeature ItemsFeature => _features.Fetch(ref _features.Cache.Items, _newItemsFeature)!;

   private IServiceProvidersFeature ServiceProvidersFeature => _features.Fetch(ref _features.Cache.ServiceProviders, this, _newServiceProvidersFeature)!;

   private IHttpAuthenticationFeature HttpAuthenticationFeature => _features.Fetch(ref _features.Cache.Authentication, _newHttpAuthenticationFeature)!;

   private IHttpRequestLifetimeFeature LifetimeFeature => _features.Fetch(ref _features.Cache.Lifetime, _newHttpRequestLifetimeFeature)!;

   private ISessionFeature SessionFeature => _features.Fetch(ref _features.Cache.Session, _newSessionFeature)!;

   private ISessionFeature? SessionFeatureOrNull => _features.Fetch(ref _features.Cache.Session, _nullSessionFeature);

   private IHttpRequestIdentifierFeature RequestIdentifierFeature => _features.Fetch(ref _features.Cache.RequestIdentifier, _newHttpRequestIdentifierFeature)!;

   public override IDictionary<object, object?> Items {
      get { return ItemsFeature.Items; }
      set { ItemsFeature.Items = value; }
   }
   
   public override string TraceIdentifier
   {
      get { return RequestIdentifierFeature.TraceIdentifier; }
      set { RequestIdentifierFeature.TraceIdentifier = value; }
   }
    
   // ...
   public override ISession Session {
      get {
         return feature.Session;
      }
      set {
         SessionFeature.Session = value;
      }
   }

   public override IServiceProvider RequestServices {
      get { return ServiceProvidersFeature.RequestServices; }
      set { ServiceProvidersFeature.RequestServices = value; }
   }

   struct FeatureInterfaces {
      public IItemsFeature? Items;
      public IServiceProvidersFeature? ServiceProviders;
      public IHttpAuthenticationFeature? Authentication;
      public IHttpRequestLifetimeFeature? Lifetime;
      public ISessionFeature? Session;
      public IHttpRequestIdentifierFeature? RequestIdentifier;
   }
}
//------------------------------------Ʌ

//---------------------------------VVV
public interface IFeatureCollection : IEnumerable<KeyValuePair<Type, object>> {   // represents a collection of HTTP features
   
   bool IsReadOnly { get; }
   
   // incremented for each modification and can be used to verify cached results
   int Revision { get; }
  
   // setting a null value removes the feature
   object? this[Type key] { get; set; }  

   TFeature? Get<TFeature>();

   void Set<TFeature>(TFeature? instance);
}
//----------------------------------ɅɅɅ
//----------------------------VV
public class FeatureCollection : IFeatureCollection {  //<-----------------------------------a1.2
   private static readonly KeyComparer FeatureKeyComparer = new KeyComparer();
   private readonly IFeatureCollection? _defaults;
   private readonly int _initialCapacity;
   private IDictionary<Type, object>? _features;  // <<----------------this is the property that stores/retrieves features
                                                  // for cookie, Type is ResponseCookiesFeature,
   private volatile int _containerRevision;

   public FeatureCollection() { }

   public FeatureCollection(int initialCapacity) {   //<-----------------------------------a1.3
      _initialCapacity = initialCapacity;
   }

   public FeatureCollection(IFeatureCollection defaults) {
      _defaults = defaults;
   }

   public virtual int Revision {
      get { return _containerRevision + (_defaults?.Revision ?? 0); }
   }

   public bool IsReadOnly { get { return false; } }

   public object? this[Type key] {
      get {
         return _features != null && _features.TryGetValue(key, out var result) ? result : _defaults?[key];
      }
      set {
         if (key == null) {
            throw new ArgumentNullException(nameof(key));
         }
         if (value == null) {
            if (_features != null && _features.Remove(key)) {
               _containerRevision++;
            }
            return;
         }

         if (_features == null) {
            _features = new Dictionary<Type, object>(_initialCapacity);
         }
         _features[key] = value;
         _containerRevision++;   // <----------------------- once we set a new feature, the revision increments
      }
   }

   public TFeature? Get<TFeature>() {
      return (TFeature?)this[typeof(TFeature)];
   }
 
   public void Set<TFeature>(TFeature? instance) {   // <--------------------------a3.1
        this[typeof(TFeature)] = instance;
   }
   // ... implements IEnumerable<KeyValuePair<Type, object>> because of IFeatureCollection
}
//----------------------------ɅɅ
//-------------------------------------V
public struct FeatureReferences<TCache> {

   public FeatureReferences(IFeatureCollection collection) {
      Collection = collection;
      Cache = default;
      Revision = collection.Revision;
   }
   
   public void Initalize(IFeatureCollection collection) {  // <------------------------------a2.1
      Revision = collection.Revision;  // 0 initially
      Collection = collection;                             // <------------------------------a2.2
   }

   public void Initalize(IFeatureCollection collection, int revision) {
      Revision = revision;
      Collection = collection;
   }

   public IFeatureCollection Collection { get; private set; }   // <-----------a2.2------------>  stores the FeatureCollection instance passed from DefaultHttpContext
                                                                // <--------------------------a3.0
   public int Revision { get; private set; }

   public TCache? Cache;

   public TFeature? Fetch<TFeature>(ref TFeature? cached, Func<IFeatureCollection, TFeature?> factory) => Fetch(ref cached, Collection, factory);

   public TFeature? Fetch<TFeature, TState>(ref TFeature? cached, TState state, Func<TState, TFeature?> factory) where TFeature : class?) {
      var flush = false;
      var revision = Collection?.Revision ?? ContextDisposed();  // throw new ObjectDisposedException
      if (Revision != revision) {
         // clear cached value to force call to UpdateCached
         cached = null!;
         // collection changed, clear whole feature cache
         flush = true;
      }

      return cached ?? UpdateCached(ref cached!, state, factory, revision, flush);   // ?? also apply when first time TFeature is accessed
   }

   private TFeature? UpdateCached<TFeature, TState>(ref TFeature? cached, TState state, Func<TState, TFeature?> factory, int revision, bool flush) {
      if (flush) {
         // Collection detected as changed, clear cache
         Cache = default;
      }

      cached = Collection.Get<TFeature>();
      if (cached == null) {
         // item not in collection, create it with factory
         cached = factory(state);
         // add item to IFeatureCollection
         Collection.Set(cached);
         // revision changed by .Set, update revision to new value
         Revision = Collection.Revision;
      } 
      else if (flush) {
         // cache was cleared, but item retrieved from current Collection for version, so use passed in revision rather than making another virtual call
         Revision = revision;
      }

      return cached;
   }
}
//-----------------------------------Ʌ
//--------------------------------VV
public abstract class HttpResponse {
   public abstract long? ContentLength { get; set; }
   public abstract string ContentType { get; set; }
   public abstract IResponseCookies Cookies { get; }
   public abstract bool HasStarted { get; }
   public abstract IHeaderDictionary Headers { get; }
   public abstract int StatusCode { get; set; }
   public virtual void Redirect(string location);

   public abstract HttpContext HttpContext { get; }
}
//--------------------------------ɅɅ
//---------------------------------------V
internal sealed class DefaultHttpResponse : HttpResponse {
   private static readonly Func<IFeatureCollection, IHttpResponseFeature?> _nullResponseFeature = f => null;
   private static readonly Func<IFeatureCollection, IHttpResponseBodyFeature?> _nullResponseBodyFeature = f => null;
   private static readonly Func<IFeatureCollection, IResponseCookiesFeature?> _newResponseCookiesFeature = f => new ResponseCookiesFeature(f);

   private readonly DefaultHttpContext _context;
   private FeatureReferences<FeatureInterfaces> _features;

   public DefaultHttpResponse(DefaultHttpContext context) { 
      _context = context;
      _features.Initalize(context.Features);
   }

   public void Initialize() {
      _features.Initalize(_context.Features);
   }

   public void Initialize(int revision) {
      _features.Initalize(_context.Features, revision);
   }

   public void Uninitialize() {
      _features = default;
   }

   private IHttpResponseFeature HttpResponseFeature => _features.Fetch(ref _features.Cache.Response, _nullResponseFeature)!;

   private IHttpResponseBodyFeature HttpResponseBodyFeature =>  _features.Fetch(ref _features.Cache.ResponseBody, _nullResponseBodyFeature)!;

   private IResponseCookiesFeature ResponseCookiesFeature => _features.Fetch(ref _features.Cache.Cookies, _newResponseCookiesFeature)!;

   public override IResponseCookies Cookies {
      get { return ResponseCookiesFeature.Cookies; }
   }
   
   public override HttpContext HttpContext { get { return _context; } }

   public override int StatusCode {
      get { return HttpResponseFeature.StatusCode; }
      set { HttpResponseFeature.StatusCode = value; }
   }

   public override IHeaderDictionary Headers {
      get { return HttpResponseFeature.Headers; }
   }

   public override Stream Body {
      get { return HttpResponseBodyFeature.Stream; }
      set {
         // ...
         _features.Collection.Set<IHttpResponseBodyFeature>(new StreamResponseBodyFeature(value, otherFeature));
      }
   }

   public override long? ContentLength {
      get { return Headers.ContentLength; }
      set { Headers.ContentLength = value; }
   }

   public override string? ContentType {
      get {
         return Headers.ContentType;
      }
      set {
         if (string.IsNullOrEmpty(value)) {
            HttpResponseFeature.Headers.ContentType = default;
         } 
         else {
            HttpResponseFeature.Headers.ContentType = value;
         }
      }
   }

   public override void Redirect(string location, bool permanent) {
      if (permanent) {
         HttpResponseFeature.StatusCode = 301;
      }
      else 
      {
         HttpResponseFeature.StatusCode = 302;
      }

      Headers.Location = location;
   }

   struct FeatureInterfaces {
      public IHttpResponseFeature? Response;
      public IHttpResponseBodyFeature? ResponseBody;
      public IResponseCookiesFeature? Cookies;
   }
}
//-------------------------------------Ʌ
//-------------------------------VV
public abstract class HttpRequest {
   ...
   public abstract Stream Body { get; set; }
   public abstract string ContentType { get; set; }
   public abstract long? ContentLength { get; set; }
   public abstract IRequestCookieCollection Cookies { get; set; }
   public abstract IFormCollection Form { get; set; }
   public abstract IHeaderDictionary Headers { get; }
   public abstract bool IsHttps { get; set; }
   public abstract string Method { get; set; }
   public abstract PathString Path { get; set; }
   public abstract IQueryCollection Query { get; set; }
   public virtual RouteValueDictionary RouteValues { get; set; }
   
   public abstract HttpContext HttpContext { get; }
}
//-------------------------------ɅɅ
//--------------------------------------V
internal sealed class DefaultHttpRequest : HttpRequest {
   private const string Http = "http";
   private const string Https = "https";

   private static readonly Func<IFeatureCollection, IHttpRequestFeature?> _nullRequestFeature = f => null;
   private static readonly Func<IFeatureCollection, IQueryFeature?> _newQueryFeature = f => new QueryFeature(f);
   private static readonly Func<DefaultHttpRequest, IFormFeature> _newFormFeature = r => new FormFeature(r, r._context.FormOptions ?? FormOptions.Default);
   private static readonly Func<IFeatureCollection, IRequestCookiesFeature> _newRequestCookiesFeature = f => new RequestCookiesFeature(f);
   private static readonly Func<IFeatureCollection, IRouteValuesFeature> _newRouteValuesFeature = f => new RouteValuesFeature();

   private readonly DefaultHttpContext _context;
   private FeatureReferences<FeatureInterfaces> _features;

   public DefaultHttpRequest(DefaultHttpContext context) {
      _context = context;
      _features.Initalize(context.Features);
   }

   public void Initialize() {
      _features.Initalize(_context.Features);
   }

   public void Uninitialize() {
      _features.Initalize(_context.Features, revision);
   }

   public override HttpContext HttpContext => _context;

   private IHttpRequestFeature HttpRequestFeature => _features.Fetch(ref _features.Cache.Request, _nullRequestFeature)!;

   private IQueryFeature QueryFeature => _features.Fetch(ref _features.Cache.Query, _newQueryFeature)!;

   private IFormFeature FormFeature => _features.Fetch(ref _features.Cache.Form, this, _newFormFeature)!;

   private IRequestCookiesFeature RequestCookiesFeature => _features.Fetch(ref _features.Cache.Cookies, _newRequestCookiesFeature)!;   // <--------------------c1

   private IRouteValuesFeature RouteValuesFeature =>  _features.Fetch(ref _features.Cache.RouteValues, _newRouteValuesFeature)!;

   // ...

   public override string Method {
      get { return HttpRequestFeature.Method; }
      set { HttpRequestFeature.Method = value; }
   }

   public override IHeaderDictionary Headers {
      get { return HttpRequestFeature.Headers; }
   }

   public override IRequestCookieCollection Cookies {   // <----------------------------c1
      get { return RequestCookiesFeature.Cookies; }
      set { RequestCookiesFeature.Cookies = value; }
   }

   public override RouteValueDictionary RouteValues {
      get { return RouteValuesFeature.RouteValues; }
      set { RouteValuesFeature.RouteValues = value; }
   }

   struct FeatureInterfaces {
      public IHttpRequestFeature? Request;
      public IQueryFeature? Query;
      public IFormFeature? Form;
      public IRequestCookiesFeature? Cookies;
      public IRouteValuesFeature? RouteValues;
      public IRequestBodyPipeFeature? BodyPipe;
   }
}
//--------------------------------------Ʌ
//-----------------------------V
public class HttpRequestFeature : IHttpRequestFeature {
   public HttpRequestFeature() {
      Headers = new HeaderDictionary();
      Body = Stream.Null;
      Method = string.Empty;
      // ...
      Path = string.Empty;
   }

   public string Protocol { get; set; }
   public string Scheme { get; set; }
   public string Method { get; set; }
   public string PathBase { get; set; }
   public string Path { get; set; }
   public string QueryString { get; set; }
   public string RawTarget { get; set; }
   public IHeaderDictionary Headers { get; set; }
   public Stream Body { get; set; }
}
//-----------------------------Ʌ
//--------------------------------V
public class RequestCookiesFeature : IRequestCookiesFeature {  // this is the one that get stored in FeatureCollection's IDictionary<Type, object> property
   private static readonly Func<IFeatureCollection, IHttpRequestFeature?> _nullRequestFeature = f => null;
   
   private StringValues _original;
   private IRequestCookieCollection? _parsedValues;

   private FeatureReferences<IHttpRequestFeature> _features;

   public RequestCookiesFeature(IRequestCookieCollection cookies) {
      _parsedValues = cookies;
   }

   public RequestCookiesFeature(IFeatureCollection features) {
      _features.Initalize(features);
   }

   private IHttpRequestFeature HttpRequestFeature => _features.Fetch(ref _features.Cache, _nullRequestFeature)!;   // <-----------c1.2----------->

   public IRequestCookieCollection Cookies {   // <---------------------------------c1.1
      get {
         if (_features.Collection == null) {
            if (_parsedValues == null) {
               _parsedValues = RequestCookieCollection.Empty;
            }
            return _parsedValues;
         }

         var headers = HttpRequestFeature.Headers;   // <---------------------------------c1.2
         var current = headers.Cookie;

         if (_parsedValues == null || _original != current) {
            _original = current;
            _parsedValues = RequestCookieCollection.Parse(current);   // <---------------------------------c1.2
         }

         return _parsedValues;
      }

      set {
         _parsedValues = value;
         _original = StringValues.Empty;
         if (_features.Collection != null) {
            if (_parsedValues == null || _parsedValues.Count == 0) {
               HttpRequestFeature.Headers.Cookie = default;
            }
            else {
               var headers = new List<string>(_parsedValues.Count);
               foreach (var pair in _parsedValues) {
                  headers.Add(new CookieHeaderValue(pair.Key, pair.Value).ToString());
               }
               _original = headers.ToArray();
               HttpRequestFeature.Headers.Cookie = _original;
            }
         }
      }
   }
}
//--------------------------------Ʌ
//------------------------------V
public class HttpResponseFeature : IHttpResponseFeature {

   public HttpResponseFeature() {
      StatusCode = 200;
      Headers = new HeaderDictionary();
      Body = Stream.Null;
   }

   public int StatusCode { get; set; }
   public string? ReasonPhrase { get; set; }
   public IHeaderDictionary Headers { get; set; }
   public Stream Body { get; set; }
   public virtual bool HasStarted => false;

   public virtual void OnStarting(Func<object, Task> callback, object state) { }

   public virtual void OnCompleted(Func<object, Task> callback, object state) { }
}
//------------------------------Ʌ
//---------------------------V
public class HeaderDictionary : IHeaderDictionary {

   private static readonly string[] EmptyKeys = Array.Empty<string>();
   private static readonly StringValues[] EmptyValues = Array.Empty<StringValues>();

   public HeaderDictionary() { }

   public HeaderDictionary(Dictionary<string, StringValues>? store) {
      Store = store;
   }

   private Dictionary<string, StringValues>? Store { get; set; }  // this dictionary stores all header, for example Set-Cookie: sessionId=38afes7a8
   public StringValues this[string key] {
      get {
         return Store.TryGetValue(key, out value);  // *
      }
      set {
         Store[key] = value;  // *
      }
   }
   // all the operations operates on Store
}

public partial interface IHeaderDictionary {
   // ...
   StringValues Cookie { get => this[HeaderNames.Cookie]; set => this[HeaderNames.Cookie] = value; }
   StringValues SetCookie { get => this[HeaderNames.SetCookie]; set => this[HeaderNames.SetCookie] = value; }
}
//---------------------------Ʌ
//-----------------------------V
public static class HeaderNames {   // defines constants for well-known HTTP headers
   // ...
   public static readonly string Accept = "Accept";
   public static readonly string Status = ":status";
   public static readonly string Cookie = "Cookie";
   public static readonly string SetCookie = "Set-Cookie";
}
//-----------------------------Ʌ
```

## Demystify HttpContext and Feature

Note that it's asp.net core web server (Kestrel) that constructs a `DefaultHttpContext` (`HttpContext`)
-------------------------------------------------------------------------------------------------
**Part A**-`DefaultHttpContext`, `FeatureCollection` and `FeatureReferences<TCache>`

**a1**: 

`DefaultHttpContext`'s default constructor `public DefaultHttpContext() : this(new FeatureCollection(DefaultFeatureCollectionSize))` invokes another constructor `public DefaultHttpContext(IFeatureCollection features)` that receives a `new FeatureCollection(10)`.

`FeatureCollection` has `_features` property ( it is different to `DefaultHttpContext` who also has a `_features` property) which is `IDictionary<Type, object>` which is used to store/retrieve features.

**a2**: 

a2.1-`FeatureReferences<FeatureInterfaces>` (both `FeatureReferences<TCache>` and `FeatureInterfaces` are struct, this is very important) kicks in. Firstly, it calls `Initalize()` that takes the new `FeatureCollection` instance as argument. 

a2.2-`FeatureReferences<FeatureInterfaces>.Collection` property is used to store the `FeatureCollection` instance that is passed from `DefaultHttpContext`'s constructor. 

**a3**: 

`DefaultHttpContext.Features.Set<T>(new T())` is called to setup `IHttpRequestFeature`, `IHttpResponseFeature`, `IHttpResponseBodyFeature`, which results in 
DefaultHttpContext's `FeatureReferences<FeatureInterfaces>` property's Collection will be set with `HttpRequestFeature`, `HttpResponseFeature` and `StreamResponseBodyFeature`. Th e collection's revision will be set to 0 initially, and note that every time the collection is changed, the revision will increment.

At this stage `DefaultHttpContext`'s _features property (`FeatureReferences<FeatureInterfaces>`) has the `new FeatureCollection(DefaultFeatureCollectionSize)` instance

**a4**: `DefaultHttpRequest` and `DefaultHttpResponse` instances are created. Note that both of them receives `DefaultHttpContext` as:

```C#
public DefaultHttpContext(IFeatureCollection features) { 
   // ...  
   _request  = new DefaultHttpRequest(this);   // `this` is DefaultHttpContext
   _response = new DefaultHttpResponse(this); 
}
```
that's why you can access `HttpContext` from `HttpRequest` and `Response`


**The key take away is, asp.net core store everything in "Feature" (probably for extensibility) which is a `FeatureCollection` associated with `HttpContext`, and this `IFeatureCollection` is managed by a "cache" structure (`FeatureReferences<FeatureInterfaces>`) and this cache collection get passed to `HttpRequest` and `HttpResponse` (along with ``HttpContext` itself), so if you add/modify a feature in one party (e.g `HttpRequest`), and the change will be refelected to the rest of two parties (e.g `HttpContext` and `HttpResponse`) because of the cache**

**The cache work like this: some common features like `IRequestCookiesFeature`, `IRouteValuesFeature` etc are defined in a struct, if the `FeatureCollection` is changed by any other party, the Collection's Revision number will change, so the party can compare the Revision number to know it should get the latest feature from `FeatureCollection`**
---------------------------------------------------------------------------------------------------------

## Demystify CookiePolicyMiddleware

To see the effect, restart ASP.NET Core and request http://localhost/consent and then http://localhost/cookie. When consent is granted, nonessential cookies (counter1) are allowed, and both the counters in the example will work. Repeat the process to withdraw consent, and you will find that only the counter whose cookie has been denoted as essential (counter2) works.

**Grant Consent**:

![alt text](./zImages/zDemystifyASP.NETCore-1.png "consent is granted")

**Request After Consent Granted**:

![alt text](./zImages/zDemystifyASP.NETCore-3.png "Request after consent granted")

**Withdrawn Consent**:

![alt text](./zImages/zDemystifyASP.NETCore-2.png "consent is withdrawn")

**Request After Consent Withdrawn**:

![alt text](./zImages/zDemystifyASP.NETCore-4.png "Request when consent is withdrawn")

```C#
//------------------V
public class Startup {

   public void ConfigureServices(IServiceCollection services) {
      
      services.Configure<CookiePolicyOptions>(opts => {  // <----------------b1
         opts.CheckConsentNeeded = context => true;
      });
   }

   public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger) {
      // ...
      app.UseCookiePolicy();   // <--------------------b1
      app.UseMiddleware<ConsentMiddleware>();   
      app.UseRouting();

      app.UseEndpoints(endpoints => {
         endpoints.MapGet("/cookie", async context => {        
            int counter1 = int.Parse(context.Request.Cookies["counter1"] ?? "0") + 1;   // <---------------------------------c1
            context.Response.Cookies.Append("counter1", counter1.ToString(),            // <---------------------------------c2
               new CookieOptions {
                  MaxAge = TimeSpan.FromMinutes(30),
                  IsEssential = true
               });

            int counter2 = int.Parse(context.Request.Cookies["counter2"] ?? "0") + 1;
            context.Response.Cookies.Append("counter2", counter1.ToString(),
               new CookieOptions { MaxAge = TimeSpan.FromMinutes(30) });

            await context.Response.WriteAsync($"Counter1: {counter1}, Counter2: {counter2}");
         });

         endpoints.MapGet("clear", context => {
            context.Response.Cookies.Delete("counter1");
            context.Response.Cookies.Delete("counter2");
            context.Response.Redirect("/");
            return Task.CompletedTask;
         });

         endpoints.MapFallback(async context =>
         await context.Response.WriteAsync("Hello World!"));
      });
   }
}
//------------------Ʌ
public class CookiePolicyOptions {
   private string _consentCookieValue = "yes";

   public CookieBuilder ConsentCookie { get; set; } = new CookieBuilder() {
      Name = ".AspNet.Consent",
      Expiration = TimeSpan.FromDays(365),
      IsEssential = true
   };
   
   public string ConsentCookieValue {
      get => _consentCookieValue;
      set => _consentCookieValue = value;
   }

   public Func<HttpContext, bool>? CheckConsentNeeded { get; set; }
   // ...
}

public class CookieBuilder {
   private string? _name;   // the name of the cookie

   public virtual string? Name {
      get => _name;
      set => _name = value;
   }

   public virtual string? Path { get; set; }
   public virtual string? Domain { get; set; }
   public virtual bool HttpOnly { get; set; }
   public virtual TimeSpan? Expiration { get; set; }
   public virtual TimeSpan? MaxAge { get; set; }
   public virtual bool IsEssential { get; set; }
   // ...

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
//------------------Ʌ
//---------------------------------V
public class CookiePolicyMiddleware {
   private readonly RequestDelegate _next;
   private readonly ILogger _logger;

   public CookiePolicyMiddleware(RequestDelegate next, IOptions<CookiePolicyOptions> options, ILoggerFactory factory) {
      Options = options.Value;
      _next = next;
      _logger = factory.CreateLogger<CookiePolicyMiddleware>();
   }

   public CookiePolicyOptions Options { get; set; }

   public Task Invoke(HttpContext context) {
      var feature = context.Features.Get<IResponseCookiesFeature>() ?? new ResponseCookiesFeature(context.Features)  // <---------------------b2
      var wrapper = new ResponseCookiesWrapper(context, Options, feature, _logger);        // <--------------------b3
      context.Features.Set<IResponseCookiesFeature>(new CookiesWrapperFeature(wrapper));   // <--------------------b4
      context.Features.Set<ITrackingConsentFeature>(wrapper);                              // <--------------------b4

      return _next(context);
   }

   private class CookiesWrapperFeature : IResponseCookiesFeature {
      public CookiesWrapperFeature(ResponseCookiesWrapper wrapper) {
         Cookies = wrapper;
      }

      public IResponseCookies Cookies { get; }
   }
}
//---------------------------------Ʌ
//----------------------------V
public class ConsentMiddleware {
   private RequestDelegate next;

   public ConsentMiddleware(RequestDelegate nextDelgate) {
      next = nextDelgate;
   }

   public async Task Invoke(HttpContext context) {
      if (context.Request.Path == "/consent") {
         ITrackingConsentFeature consentFeature = context.Features.Get<ITrackingConsentFeature>();   // ITrackingConsentFeature is ResponseCookiesWrapper that is 
         if (!consentFeature.HasConsent) {                                                           // set in CookiePolicyMiddleware
            consentFeature.GrantConsent();
         } else {
            consentFeature.WithdrawConsent();
         }
         await context.Response.WriteAsync(consentFeature.HasConsent ? "Consent Granted \n" : "Consent Withdrawn\n");
      }
      await next(context);
   }
}
//----------------------------Ʌ
//---------------------------------------V
internal sealed class DefaultHttpResponse : HttpResponse {
   // ...
   private static readonly Func<IFeatureCollection, IResponseCookiesFeature?> _newResponseCookiesFeature = f => new ResponseCookiesFeature(f);

   private readonly DefaultHttpContext _context;
   private FeatureReferences<FeatureInterfaces> _features;    

   public DefaultHttpResponse(DefaultHttpContext context) {
      _context = context;
      _features.Initalize(context.Features);
   }

   public void Initialize() {
      _features.Initalize(_context.Features);
   }

   public void Initialize(int revision) {
      _features.Initalize(_context.Features, revision);
   }

   public void Uninitialize() {
      _features = default;
   }
                                                                                
   private IResponseCookiesFeature ResponseCookiesFeature => _features.Fetch(ref _features.Cache.Cookies, _newResponseCookiesFeature)!;  

   public override IResponseCookies Cookies { 
      get { return ResponseCookiesFeature.Cookies; }
   }

   struct FeatureInterfaces {
      public IHttpResponseFeature? Response;
      public IHttpResponseBodyFeature? ResponseBody;
      public IResponseCookiesFeature? Cookies;   
   }
}
//---------------------------------------Ʌ
//-------------------------------------V
public struct FeatureReferences<TCache> {    // TCache is FeatureInterfaces

   public FeatureReferences(IFeatureCollection collection) {
      Collection = collection;
      Cache = default;   // <----------------
      Revision = collection.Revision;
   }
   
   public void Initalize(IFeatureCollection collection) { 
      Revision = collection.Revision;
      Collection = collection;                             
   }

   public void Initalize(IFeatureCollection collection, int revision) {
      Revision = revision;
      Collection = collection;
   }

   public IFeatureCollection Collection { get; private set; }   
                                                               
   public int Revision { get; private set; }

   public TCache? Cache;   // TCache is FeatureInterfaces

                                     // TFeature is IResponseCookiesFeature, from DefaultHttpResponse._features.Cookies (_features is FeatureReferences<FeatureInterfaces>)
   public TFeature? Fetch<TFeature>(ref TFeature? cached, Func<IFeatureCollection, TFeature?> factory) {   // <------------------------------c1.1
      Fetch(ref cached, Collection, factory);   
   } 
                                                                      // state is FeatureCollection
   public TFeature? Fetch<TFeature, TState>(ref TFeature? cached, TState state, Func<TState, TFeature?> factory) where TFeature : class?) {  // <-------c1.1------->
      var flush = false;
      var revision = Collection?.Revision ?? ContextDisposed();  
      if (Revision != revision) {
         cached = null!;
         flush = true;
      }

      return cached ?? UpdateCached(ref cached!, state, factory, revision, flush);   // <------------------------------c1.2
   }

   private TFeature? UpdateCached<TFeature, TState>(ref TFeature? cached, TState state, Func<TState, TFeature?> factory, int revision, bool flush) {
      if (flush) {
         Cache = default;
      }
                           // TFeature is IResponseCookiesFeature
      cached = Collection.Get<TFeature>();   // <---------------------------------c1.3
      if (cached == null) {
         cached = factory(state);
         Collection.Set(cached);
         Revision = Collection.Revision;
      } 
      else if (flush) {
         Revision = revision;
      }

      return cached;
   }
}
//-------------------------------------Ʌ
```
```C#
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
            _cookiesCollection = new ResponseCookies(_features);   // <-----------------------------c2
         }

         return _cookiesCollection;
      }
   }
}
//--------------------------------------Ʌ
//-------------------------------V
public interface IResponseCookies {
   void Append(string key, string value);
   void Append(string key, string value, CookieOptions options);
   void Delete(string key);
   void Delete(string key, CookieOptions options);
}

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
//------------------------V
public class CookieOptions {
   public CookieOptions() {m 
      Path = "/";
   }
   public string? Domain { get; set; }
   public string? Path { get; set; }
   public DateTimeOffset? Expires { get; set; }
   public bool Secure { get; set; }
   public SameSiteMode SameSite { get; set; } = SameSiteMode.Unspecified;
   public bool HttpOnly { get; set; }
   public TimeSpan? MaxAge { get; set; }
   public bool IsEssential { get; set; }
}
//------------------------Ʌ
//-------------------------------V
public class SetCookieHeaderValue {
   private const string ExpiresToken = "expires";
   private const string MaxAgeToken = "max-age";
   private const string DomainToken = "domain";
   // ...
   private StringSegment _name;
   private StringSegment _value;

   public SetCookieHeaderValue(StringSegment name, StringSegment value) {
      Name = name;
      Value = value;
   }

   public DateTimeOffset? Expires { get; set; }
   public TimeSpan? MaxAge { get; set; }
   public StringSegment Domain { get; set; }
   // ...

   public override string ToString() {
      // return example: "<cookie-name>=<cookie-value>; expires=Sun, 06 Nov 1994 08:49:37 GMT; max-age=86400; domain=domain1;"
   }
}
//-------------------------------Ʌ
```
```C#
//--------------------------------------V
public interface ITrackingConsentFeature {
   bool IsConsentNeeded { get; }
   bool HasConsent { get; }
   bool CanTrack { get; }   // indicates either if consent has been given or if consent is not required
   void GrantConsent();
   void WithdrawConsent();
   string CreateConsentCookie();
}

internal class ResponseCookiesWrapper : IResponseCookies, ITrackingConsentFeature {
   private const string ConsentValue = "yes";
   private readonly ILogger _logger;
   private bool? _isConsentNeeded;
   private bool? _hasConsent;

   public ResponseCookiesWrapper(HttpContext context, CookiePolicyOptions options, IResponseCookiesFeature feature, ILogger logger) { // <-----------------------b3
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
         Append(Options.ConsentCookie.Name!, ConsentValue, cookieOptions);   // Options.ConsentCookie.Name is ".AspNet.Consent"; ConsentValue is const string value "yes"
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
```

**b1**: 

`CookiePolicyOptions` value is provided to `CookiePolicyMiddleware`

**b2**:

 `DefaultHttpContext.Feature` (which is `FeatureReferences<FeatureInterfaces>`) is queried, since it is the first time of the query, an instance of `ResponseCookiesFeature` is created:

```C#
public class ResponseCookiesFeature : IResponseCookiesFeature {
   private readonly IFeatureCollection _features;
   private IResponseCookies? _cookiesCollection;

   public ResponseCookiesFeature(IFeatureCollection features) {
      _features = features ?? throw new ArgumentNullException(nameof(features));
   }

   public IResponseCookies Cookies {
      get {
         if (_cookiesCollection == null) {
            _cookiesCollection = new ResponseCookies(_features);            
         }

         return _cookiesCollection;
      }
   }
}

// check source code for details
internal partial class ResponseCookies : IResponseCookies {   // IResponseCookies defines Append(...), Delete(...)
   private readonly IFeatureCollection _features;

   internal ResponseCookies(IFeatureCollection features) {
      _features = features;                                                     
      Headers = _features.GetRequiredFeature<IHttpResponseFeature>().Headers;   // GetRequiredFeature is just an extension method of `featureCollection.Get<TFeature>()` 
                                                                                // IHttpResponseFeature (HttpResponseFeature) is initialized in DefaultHttpContext's constructor
   }

   private IHeaderDictionary Headers { get; set; }

   public void Append(string key, string value) {  // also can taks CookieOptions options
      var setCookieHeaderValue = new SetCookieHeaderValue(_enableCookieNameEncoding ? Uri.EscapeDataString(key) : key, Uri.EscapeDataString(value)) {
         Path = "/"
      }
      var cookieValue = setCookieHeaderValue.ToString();
      Headers.SetCookie = StringValues.Concat(Headers.SetCookie, cookieValue);
   }

   public void Delete(string key) {
      // ...
   }
   // ...
}
```

**b3**:   

An instance of `ResponseCookiesWrapper` is created, and this wrapper implments `ITrackingConsentFeature` in contrast to `ResponseCookies`:

```C#
internal class ResponseCookiesWrapper : IResponseCookies, ITrackingConsentFeature {
   private const string ConsentValue = "yes";
   private bool? _isConsentNeeded;
   private bool? _hasConsent;

   public ResponseCookiesWrapper(HttpContext context, CookiePolicyOptions options, IResponseCookiesFeature feature) { 
      Feature = feature;
      Options = options;
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
   // ... other ITrackingConsentFeature's methods
   
   public void Append(string key, string value) {
      // ... check some logics
      Cookies.Append(key, value);
   }
   // ... other IResponseCookies's methods
}
```

Note that the "unwrapper" type is `ResponseCookies`:

```C#
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
   // ...
}
```

**c1**:  RequestCookiesFeature

When `context.Request.Cookies["counter"]` is accessed:

- `RequestCookiesFeature` is accessed, a new instance is created
- `RequestCookiesFeature` uses `HttpRequestFeature` via (its `HeaderDictionary` property) internally to create  an `RequestCookieCollection` instance who has `AdaptiveCapacityDictionary` internally that store real cookie data

```C#
//--------------------------------------V
internal sealed class DefaultHttpRequest : HttpRequest {
   private static readonly Func<IFeatureCollection, IHttpRequestFeature?> _nullRequestFeature = f => null;
   private static readonly Func<IFeatureCollection, IRequestCookiesFeature> _newRequestCookiesFeature = f => new RequestCookiesFeature(f);

   private IHttpRequestFeature HttpRequestFeature => _features.Fetch(ref _features.Cache.Request, _nullRequestFeature)!;
   private IRequestCookiesFeature RequestCookiesFeature => _features.Fetch(ref _features.Cache.Cookies, _newRequestCookiesFeature)!;  // <------------2

   private FeatureReferences<FeatureInterfaces> _features;

   public override IRequestCookieCollection Cookies {    // <-----------------------------0
      get { return RequestCookiesFeature.Cookies; }      // <-----------------------------1
      set { RequestCookiesFeature.Cookies = value; }
   }

   struct FeatureInterfaces {
      public IHttpRequestFeature? Request;
      public IRequestCookiesFeature? Cookies;
   }
}
//------------------------------------Ʌ
//--------------------------------V
public class RequestCookiesFeature : IRequestCookiesFeature { 
   private static readonly Func<IFeatureCollection, IHttpRequestFeature?> _nullRequestFeature = f => null;

   private FeatureReferences<IHttpRequestFeature> _features;
   private StringValues _original;
   private IRequestCookieCollection? _parsedValues;   // <--------------- store property (AdaptiveCapacityDictionary) that contains the real cookies data!

   public RequestCookiesFeature(IRequestCookieCollection cookies) {
      _parsedValues = cookies;
   }

   public RequestCookiesFeature(IFeatureCollection features) {
      _features.Initalize(features);
   }

   private IHttpRequestFeature HttpRequestFeature => _features.Fetch(ref _features.Cache, _nullRequestFeature)!;    // <------------------------4

   public IRequestCookieCollection Cookies {   // <------------------------3
      get {
         if (_features.Collection == null) {
            if (_parsedValues == null) {
               _parsedValues = RequestCookieCollection.Empty;
            }
            return _parsedValues;
         }

         HeaderDictionary headers = HttpRequestFeature.Headers;   // <------------------------4.1
         StringValues current = headers.Cookie;                   // <------------------------4.2
                                                                  // HeaderDictionary is a collection of `public StringValues this[string key]`, which is sth like raw data
         if (_parsedValues == null || _original != current) {
            _original = current;                                  
            _parsedValues = RequestCookieCollection.Parse(current);   // <------------------------5
         }                                                            // every time when cookie is accessed `context.Request.Cookies["xxx"], HttpRequestFeature's HeaderDictionary
                                                                      // is "copied/parsed" into _parsedValues who has AdaptiveCapacityDictionary property to deal with cookie data
         return _parsedValues;   // _parsedValues is `RequestCookieCollection` that users can use index method as `context.Request.Cookies["counter"]`, 
                                 // dfdfhdjfhdjfhjd is it the place that cooke get stored? AdaptiveCapacityDictionary?
      }

      set {
         _parsedValues = value;
         _original = StringValues.Empty;
         if (_features.Collection != null) {
            if (_parsedValues == null || _parsedValues.Count == 0) {
               HttpRequestFeature.Headers.Cookie = default;
            }
            else {
               var headers = new List<string>(_parsedValues.Count);
               foreach (var pair in _parsedValues) {
                  headers.Add(new CookieHeaderValue(pair.Key, pair.Value).ToString());
               }
               _original = headers.ToArray();                    // every time when cookie is set `context.Request.Cookies["xxx"] = "XXX", RequestCookieCollection is copied into
               HttpRequestFeature.Headers.Cookie = _original;    // HttpRequestFeature's Header
            }
         }
      }
   }
}
//---------------------------------Ʌ
//------------------------------------V
internal class RequestCookieCollection : IRequestCookieCollection {
   public static readonly RequestCookieCollection Empty = new RequestCookieCollection();
   // ...
   private AdaptiveCapacityDictionary<string, string> Store { get; set; }   // this is the data structure that holds the cookie data

   public TValue this[TKey key] {   // there is no setter, so you can't do `context.Request.Cookies["xxx"] = "XXX"`, you can only read from it
      get {
         if (TryGetValue(key, out var value)) {
            return value;
         }
         return null
      }
   }

   public bool TryGetValue(string key, [MaybeNullWhen(false)] out string? value) {
      return Store.TryGetValue(key, out value);
   }

   public static RequestCookieCollection Parse(StringValues values) {
      var collection = new RequestCookieCollection();
      // return a new RequestCookieCollection based on values, and setup this RequestCookieCollection instance's Store property
      return collection;
   }
}
//------------------------------------Ʌ
```

**c2**:  ResponseCookiesFeature

When `context.Response.Cookies["counter"]` is accessed:

```C#
//---------------------------------------V
internal sealed class DefaultHttpResponse : HttpResponse {
   // ...
   private FeatureReferences<FeatureInterfaces> _features; 

   private IResponseCookiesFeature ResponseCookiesFeature => _features.Fetch(ref _features.Cache.Cookies, _newResponseCookiesFeature)!;   // <---------------------------1.1  

   public override IResponseCookies Cookies {          // <--------------------------0
      get { return ResponseCookiesFeature.Cookies; }   // <--------------------------1.0, 2.0
   }

   struct FeatureInterfaces {
      public IHttpResponseFeature? Response;
      public IHttpResponseBodyFeature? ResponseBody;
      public IResponseCookiesFeature? Cookies;   
   }
}
//--------------------------------------Ʌ
//---------------------------------V
public class ResponseCookiesFeature : IResponseCookiesFeature {
   private readonly IFeatureCollection _features;
   private IResponseCookies? _cookiesCollection;

   public ResponseCookiesFeature(IFeatureCollection features) {
      _features = features ?? throw new ArgumentNullException(nameof(features));
   }

   public IResponseCookies Cookies {   // <--------------------------2.1
      get {
         if (_cookiesCollection == null) {
            _cookiesCollection = new ResponseCookies(_features);   // <--------------------------2.2
         }
         return _cookiesCollection;
      }
   }
}
//----------------------------------Ʌ
//------------------------------------V
internal partial class ResponseCookies : IResponseCookies {
   // ...
   private readonly IFeatureCollection _features;
   private IHeaderDictionary Headers { get; set; }   // <-------------- Headers stores the response cookie

   internal ResponseCookies(IFeatureCollection features) {   // <--------------------------2.2
      _features = features;
      Headers = _features.GetRequiredFeature<IHttpResponseFeature>().Headers;   // use HttpResponseFeature
   }

   public void Append(string key, string value, CookieOptions options) {   // <--------------------------3      
      var setCookieHeaderValue = new SetCookieHeaderValue(key, value) {  // *
         Expires = options.Expires,
         MaxAge = options.MaxAge,
         // ...
      }
      var cookieValue = setCookieHeaderValue.ToString();
      Headers.SetCookie = StringValues.Concat(Headers.SetCookie, cookieValue);   // new cookie value is set to HttpResponseFeature's Headers
   }

   public void Delete(string key, CookieOptions options) {
      // ...
      Append(key, string.Empty, new CookieOptions {   // when we delete a cookie, we set its value to null but keep the cookie name
         // ...
      });
   }

}
```
Note that `SetCookieHeaderValue` is like an helper class for `CookieOptions`, so users just needs to focus on creating an instance of `CookieOptions`, and met core uses `SetCookieHeaderValue` to generate the cookie string. And ResponseCookies doesn't have index method, which makes sense, you don't need to read cookie from it

**The key take away is, when we read/set a cookie from response or request, we operate on the `HeaderDictionary` of `HttpRequestFeature` and `HttpResponseFeature`**

## Demystify ...

<!-- <code>&lt;T&gt;<code> -->

<!-- <div class="alert alert-info p-1" role="alert">
    
</div> -->

<!-- <div class="alert alert-info pt-2 pb-0" role="alert">
    <ul class="pl-1">
      <li></li>
      <li></li>
    </ul>  
</div> -->

<!-- <ul>
  <li></li>
  <li></li>
  <li></li>
  <li></li>
</ul>  -->

<!-- <ul>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
</ul>  -->

<!-- ![alt text](./zImages/16-1.png "Title") -->

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