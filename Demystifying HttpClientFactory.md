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


# Source Code

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
            return s.GetRequiredService<IHttpClientFactory>().CreateClient(string.Empty);
        });
 
        return services;
    }
}
//--------------------------------------------------------------Ʌ
```

```C#
//-------------------------------------V
internal class DefaultHttpClientFactory : IHttpClientFactory, IHttpMessageHandlerFactory
{

}
//-------------------------------------Ʌ
```