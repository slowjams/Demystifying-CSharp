```C#

public class Startup
{
   public void ConfigureServices(IServiceCollection services)
   {     
      services.AddControllers();
   }
   public void Configure(IApplicationBuilder app, DataContext context)
   {    
      // ...
      app.UseEndpoints(endpoints =>   // <----------------endpoints is IEndpointRouteBuilder
      {                                         
         //endpoints.MapDefaultControllerRoute();
         endpoints.MapControllerRoute("Default", "{controller=Home}/{action=Index}/{id?}");
      });
   }
}

```


```C#

public interface IEndpointRouteBuilder 
{    
   IApplicationBuilder CreateApplicationBuilder();
   IServiceProvider ServiceProvider { get; }
   ICollection<EndpointDataSource> DataSources { get; }  
}

internal class DefaultEndpointRouteBuilder : IEndpointRouteBuilder 
{
   public DefaultEndpointRouteBuilder(IApplicationBuilder applicationBuilder) 
   {
      ApplicationBuilder = applicationBuilder;
      DataSources = new List<EndpointDataSource>();
   }

   public IApplicationBuilder ApplicationBuilder { get; }

   public IApplicationBuilder CreateApplicationBuilder() => ApplicationBuilder.New();
  
   public ICollection<EndpointDataSource> DataSources { get; }   

   public IServiceProvider ServiceProvider => ApplicationBuilder.ApplicationServices;
}

```





```C#

public static class ControllerEndpointRouteBuilderExtensions 
{
   public static ControllerActionEndpointConventionBuilder MapControllers(this IEndpointRouteBuilder endpointsBuilder)  
   {                                                                                                             
      return GetOrCreateDataSource(endpointsBuilder).DefaultBuilder;
   }

   public static ControllerActionEndpointConventionBuilder MapDefaultControllerRoute(this IEndpointRouteBuilder endpointsBuilder)
   {
      var dataSource = GetOrCreateDataSource(endpointsBuilder);
      
      return dataSource.AddRoute("default", "{controller=Home}/{action=Index}/{id?}", defaults: null, constraints: null, dataTokens: null);
   }

   private static ControllerActionEndpointDataSource GetOrCreateDataSource(IEndpointRouteBuilder endpointsBuilder)
   {
      var dataSource = endpointsBuilder.DataSources.OfType<ControllerActionEndpointDataSource>().FirstOrDefault();
      
      if (dataSource == null) 
      {
         var orderProvider = endpointsBuilder.ServiceProvider.GetRequiredService<OrderedEndpointsSequenceProviderCache>();
         var factory = endpointsBuilder.ServiceProvider.GetRequiredService<ControllerActionEndpointDataSourceFactory>();    
         dataSource = factory.Create(orderProvider.GetOrCreateOrderedEndpointsSequenceProvider(endpointsBuilder));         
         endpointsBuilder.DataSources.Add(dataSource);  // <--------- add ControllerActionEndpointDataSource to DefaultEndpointRouteBuilder
      }

      return dataSource;
   }
}

internal sealed class ControllerActionEndpointDataSourceFactory   
{
   private readonly ControllerActionEndpointDataSourceIdProvider _dataSourceIdProvider;
   private readonly IActionDescriptorCollectionProvider _actions;
   private readonly ActionEndpointFactory _factory;
 
   public ControllerActionEndpointDataSourceFactory(
      ControllerActionEndpointDataSourceIdProvider dataSourceIdProvider,
      IActionDescriptorCollectionProvider actions,     
      ActionEndpointFactory factory)                  
   {
      _dataSourceIdProvider = dataSourceIdProvider;
      _actions = actions;
      _factory = factory;
   }
 
   public ControllerActionEndpointDataSource Create(OrderedEndpointsSequenceProvider orderProvider)
   {
       return new ControllerActionEndpointDataSource(_dataSourceIdProvider, _actions, _factory, orderProvider); 
   }
}



```







```C#

internal sealed class ActionEndpointFactory
{
   private readonly RoutePatternTransformer _routePatternTransformer;
   private readonly RequestDelegate _requestDelegate;
   private readonly IRequestDelegateFactory[] _requestDelegateFactories;
   private readonly IServiceProvider _serviceProvider;

   public ActionEndpointFactory(RoutePatternTransformer routePatternTransformer, 
                                IEnumerable<IRequestDelegateFactory> requestDelegateFactories, 
                                IServiceProvider serviceProvider);

   public void AddEndpoints(List<Endpoint> endpoints, 
                            HashSet<string> routeNames,
                            ActionDescriptor action, 
                            IReadOnlyList<ConventionalRouteEntry> routes, ...)
   {
      if (action.AttributeRouteInfo?.Template == null) {
         foreach (var route in routes)
         {      
            var updatedRoutePattern = _routePatternTransformer
                                      .SubstituteRequiredValues(route.Pattern, action.RouteValues);
            if (updatedRoutePattern == null)
            {
               continue;
            }

            updatedRoutePattern = RoutePatternFactory.Combine(groupPrefix, updatedRoutePattern);

            var requestDelegate = CreateRequestDelegate(action, route.DataTokens) ?? _requestDelegate;  

            var builder = new RouteEndpointBuilder(requestDelegate, updatedRoutePattern, route.Order)
            {
               DisplayName = action.DisplayName,
            };
            
            AddActionDataToBuilder(builder, routeNames, action, route.RouteName ...);

            endpoints.Add(builder.Build());
         }
      }
      else
      {
         var requestDelegate = CreateRequestDelegate(action) ?? _requestDelegate;
         var attributeRoutePattern = RoutePatternFactory.Parse(action.AttributeRouteInfo.Template);
         var (resolvedRoutePattern, resolvedRouteValues) = 
            ResolveDefaultsAndRequiredValues(action, attributeRoutePattern);
         var updatedRoutePattern = _routePatternTransformer
                                   .SubstituteRequiredValues(resolvedRoutePattern, resolvedRouteValues);
         // ...
         updatedRoutePattern = RoutePatternFactory.Combine(groupPrefix, updatedRoutePattern);
         var builder = new RouteEndpointBuilder(requestDelegate, 
                                                updatedRoutePattern, 
                                                action.AttributeRouteInfo.Order)
         {
            DisplayName = action.DisplayName,
         };

         AddActionDataToBuilder(builder, routeNames, action, route.RouteName ...);

         endpoints.Add(builder.Build());
      }
   }

   private RequestDelegate? CreateRequestDelegate(ActionDescriptor action, 
                                                  RouteValueDictionary? dataTokens = null)
   {
      foreach (var factory in _requestDelegateFactories)
      {
         var requestDelegate = factory.CreateRequestDelegate(action, dataTokens);
         if (requestDelegate != null)
         {
            return requestDelegate;
         }
      }  
      return null;
   }

   private static RequestDelegate CreateRequestDelegate() 
   {
      IActionInvokerFactory invokerFactory = null;

      return (context) =>
      {
         var endpoint = context.GetEndpoint()!;
         var dataTokens = endpoint.Metadata.GetMetadata<IDataTokensMetadata>();
 
         var routeData = new RouteData();
         routeData.PushState(router: null, context.Request.RouteValues, new RouteValueDictionary(dataTokens?.DataTokens));
 
         var action = endpoint.Metadata.GetMetadata<ActionDescriptor>()!;
         var actionContext = new ActionContext(context, routeData, action);
 
         if (invokerFactory == null)
         {
            invokerFactory = context.RequestServices.GetRequiredService<IActionInvokerFactory>(); 
         }                                                                                        
 
         var invoker = invokerFactory.CreateInvoker(actionContext);  
         
         // invoker is ControllerActionInvoker, but InvokeAsync() is from ResourceInvoker
         return invoker.InvokeAsync();                                
      };                                                              
   }
}


```



```C#

//
public interface IActionDescriptorCollectionProvider
{
   ActionDescriptorCollection ActionDescriptors { get; }
}

public abstract class ActionDescriptorCollectionProvider : IActionDescriptorCollectionProvider  
{
   public abstract ActionDescriptorCollection ActionDescriptors { get; }
   public abstract IChangeToken GetChangeToken();
}
//

internal sealed partial class DefaultActionDescriptorCollectionProvider : ActionDescriptorCollectionProvider   
{
   private readonly IActionDescriptorProvider[] _actionDescriptorProviders;
   private readonly IActionDescriptorChangeProvider[] _actionDescriptorChangeProviders;
   private ActionDescriptorCollection? _collection;  // <-----------------------a
   private readonly ILogger _logger;
   // ...

   public DefaultActionDescriptorCollectionProvider(
      IEnumerable<IActionDescriptorProvider> actionDescriptorProviders,
      IEnumerable<IActionDescriptorChangeProvider> actionDescriptorChangeProviders,
      ILogger<DefaultActionDescriptorCollectionProvider> logger)
   {
      _actionDescriptorProviders = actionDescriptorProviders
         .OrderBy(p => p.Order)
         .ToArray();

      _actionDescriptorChangeProviders = actionDescriptorChangeProviders.ToArray();

      ChangeToken.OnChange(GetCompositeChangeToken, UpdateCollection);
   }

   public override ActionDescriptorCollection ActionDescriptors
   {
      get {
         Initialize();
         return _collection;
      }
   }

   private void Initialize()
   {
      if (_collection == null)
      {
         // ... call UpdateCollection();
      }
   }

   private void UpdateCollection()
   {
      // ...
      var context = new ActionDescriptorProviderContext();
      for (var i = 0; i < _actionDescriptorProviders.Length; i++)
      {
         _actionDescriptorProviders[i].OnProvidersExecuting(context);
      }

      for (var i = _actionDescriptorProviders.Length - 1; i >= 0; i--)
      {
         _actionDescriptorProviders[i].OnProvidersExecuted(context);
      }
      
      var oldCancellationTokenSource = _cancellationTokenSource;

      _collection = new ActionDescriptorCollection(   // <-----------------------b
         new ReadOnlyCollection<ActionDescriptor>(context.Results), _version++
      );

      // ...
   }
}


public class ActionDescriptorCollection
{
   public ActionDescriptorCollection(IReadOnlyList<ActionDescriptor> items, 
                                     int version)
   {
      Items = items;
      Version = version;
   }

   public IReadOnlyList<ActionDescriptor> Items { get; }
   
   public int Version { get; }
}


```