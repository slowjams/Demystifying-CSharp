Demystifying Controller and View
==============================


```C#
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
   private DbContext _dbContext;

   public ProductsController(DbContext dbContext)
   {
      _dbContext = dbContext;
   }

   [HttpGet("{id}")]
   public async Task<IActionResult> GetProduct(long id)
   {
      Product p = await _dbContext.Products.FindAsync(id);

      if (p != null)
      {
         return Ok(p);  // <---------------------------
      } 
      else
      {
         return NotFound();   // <--------------------
      }
         
   }
   
   [HttpPost]
   public async Task<IActionResult> CreateProduct([FromBody] Product product)
   {
      _dbContext.Add<Product>(product);
      await _dbContext.SaveChangesAsync();
      return CreatedAtAction(nameof(CreateProduct), product);
   }

   [HttpPut]
   public async Task<IActionResult> UpdateProduct([FromBody] Product product)
   {
      _dbContext.Products.Update(product);
      await _dbContext.SaveChangesAsync();
      return NoContent();
   }

   [HttpDelete]
   public async Task<IActionResult> DeleteProduct([FromBody] Product product)
   {
      _dbContext.Products.Remove(product);
      await _dbContext.SaveChangesAsync();
      return NoContent();   // <---------------------------
   }
}
```


## Source Code

```C#
internal partial class ControllerActionInvoker : ResourceInvoker, IActionInvoker   
{
   private readonly ControllerActionInvokerCacheEntry _cacheEntry;
   private readonly ControllerContext _controllerContext;
   private Dictionary<string, object> _arguments;
   private ActionExecutingContextSealed _actionExecutingContext;
   private ActionExecutedContextSealed _actionExecutedContext;

   internal ControllerActionInvoker(
        ILogger logger,
        DiagnosticListener diagnosticListener,
        IActionContextAccessor actionContextAccessor,
        IActionResultTypeMapper mapper,
        ControllerContext controllerContext,
        ControllerActionInvokerCacheEntry cacheEntry,
        IFilterMetadata[] filters)
        : base(diagnosticListener, logger, actionContextAccessor, mapper, controllerContext, filters, controllerContext.ValueProviderFactories)
   {
      _cacheEntry = cacheEntry;
      _controllerContext = controllerContext;
   }

   // ...

   private Task Next(ref State next, ref Scope scope, ref object? state, ref bool isCompleted)  
   {
      switch (next)
      {
         case State.ActionBegin:   
            var controllerContext = _controllerContext;
            _cursor.Reset();

            _instance = _cacheEntry.ControllerFactory(controllerContext);   // create an instance of Controller

            _arguments = new Dictionary<string, object?>(StringComparer.OrdinalIgnoreCase); 

            var task = BindArgumentsAsync();   // <--------------------------model binding

            if (task.Status != TaskStatus.RanToCompletion)
            {
               next = State.ActionNext;
               return task;
            }

            goto case State.ActionNext;

         //...
     
         case State.ActionInside:
            var task = InvokeActionMethodAsync();   // <----------------------------------------1, important!, invoke action method
            if (task.Status != TaskStatus.RanToCompletion)
            {
               next = State.ActionEnd;
               return task;
            }
 
            goto case State.ActionEnd;

         case State.ActionEnd:
            if (scope == Scope.Action)
            {
               if (_actionExecutedContext == null)
               {
                  _actionExecutedContext = new ActionExecutedContextSealed(_controllerContext, _filters, _instance!)
                  {
                     Result = _result,
                  };
               }

               isCompleted = true;
               return Task.CompletedTask;
            }

            var actionExecutedContext = _actionExecutedContext;
            Rethrow(actionExecutedContext);

            if (actionExecutedContext != null)
            {
               _result = actionExecutedContext.Result;
            }
 
            isCompleted = true;
            return Task.CompletedTask;

         default:
            throw new InvalidOperationException();
      }
   }
   
   private Task InvokeActionMethodAsync()   // <------------------------------1.1
   {
      var objectMethodExecutor = _cacheEntry.ObjectMethodExecutor;
      var actionMethodExecutor = _cacheEntry.ActionMethodExecutor;
      var orderedArguments = PrepareArguments(_arguments, objectMethodExecutor);
 
      var actionResultValueTask = actionMethodExecutor.Execute(ControllerContext, _mapper, objectMethodExecutor, _instance!, orderedArguments);
      if (actionResultValueTask.IsCompletedSuccessfully)
      {
         _result = actionResultValueTask.Result;
      }
      else
      {
         return Awaited(this, actionResultValueTask);
      }
 
      return Task.CompletedTask;
   }

   private static object?[]? PrepareArguments(IDictionary<string, object?>? actionParameters, ObjectMethodExecutor actionMethodExecutor)
   {
      var declaredParameterInfos = actionMethodExecutor.MethodParameters;
      var count = declaredParameterInfos.Length;
      if (count == 0)
      {
         return null;
      }

      var arguments = new object?[count];
      for (var index = 0; index < count; index++)
      {
         var parameterInfo = declaredParameterInfos[index];

         if (!actionParameters.TryGetValue(parameterInfo.Name!, out var value) || value is null)
         {
            value = actionMethodExecutor.GetDefaultValueForParameter(index);
         }

         arguments[index] = value;
      }

      return arguments;
   }

   private static void Rethrow(ActionExecutedContextSealed? context)
   {
      if (context == null)
         return;
 
      if (context.ExceptionHandled)
         return;
 
      if (context.ExceptionDispatchInfo != null)
         context.ExceptionDispatchInfo.Throw();
 
      if (context.Exception != null)
         throw context.Exception;
   }
}
```














```C#
//----------------------------------V
public abstract class ControllerBase  
{
   public virtual OkObjectResult Ok([ActionResultObjectValue] object? value) 
   {
      return new OkObjectResult(value);
   }

   public virtual BadRequestResult BadRequest()
   {
      return new BadRequestResult();
   }

   public virtual NotFoundResult NotFound()
   {
      return new NotFoundResult();
   }

   public virtual NotFoundObjectResult NotFound([ActionResultObjectValue] object? value)
   {
      return new NotFoundObjectResult(value);
   }
   //...
}
//---------------------------------Ʌ

//---------------------------->>
public interface IActionResult
{
   Task ExecuteResultAsync(ActionContext context);
}
//----------------------------<<

//------------------------------------->>
public interface IStatusCodeActionResult : IActionResult
{
   int? StatusCode { get; }
}
//-------------------------------------<<

//----------------------------->>
public static class StatusCodes
{
   // ...
   public const int Status200OK = 200;
   public const int Status201Created = 201;
   public const int Status202Accepted = 202;
   public const int Status204NoContent = 204;
   public const int Status400BadRequest = 400;
   public const int Status401Unauthorized = 401;
   public const int Status404NotFound = 404;
   public const int Status405MethodNotAllowed = 405;
   public const int Status500InternalServerError = 500;
   // ...
}
//-----------------------------<<

//--------------------------------V
public abstract class ActionResult : IActionResult
{
   public virtual Task ExecuteResultAsync(ActionContext context)
   {
      ExecuteResult(context);
      return Task.CompletedTask;
   }

   public virtual void ExecuteResult(ActionContext context)
   {
      // empty
   }
}
//--------------------------------Ʌ

//-----------------------------------V
public partial class StatusCodeResult : ActionResult, IStatusCodeActionResult  // check the source again you will see
{
   public StatusCodeResult( int statusCode)
   {
      StatusCode = statusCode;
   }

   public int StatusCode { get; }

   int? IStatusCodeActionResult.StatusCode => StatusCode;

   public override void ExecuteResult(ActionContext context)
   {
      var httpContext = context.HttpContext;
      // ... logging related
      httpContext.Response.StatusCode = StatusCode;   // <------------------set status code
   }
}
//-----------------------------------Ʌ

//-----------------------V
public class ObjectResult : ActionResult, IStatusCodeActionResult
{
   private MediaTypeCollection _contentTypes;

   public ObjectResult(object? value)
   {
      Value = value;   // <--------------------------
      Formatters = new FormatterCollection<IOutputFormatter>();
      _contentTypes = new MediaTypeCollection();
   }

   public object Value { get; set; }  // <--------------------------

   public FormatterCollection<IOutputFormatter> Formatters { get; set; }

   public MediaTypeCollection ContentTypes
   {
      get => _contentTypes;
      set => _contentTypes = value ?? throw new ArgumentNullException(nameof(value));
   }

   public Type DeclaredType { get; set; }

   public int StatusCode { get; set; }  // <----------------------------------

   public override Task ExecuteResultAsync(ActionContext context)
   {
      var executor = context.HttpContext.RequestServices.GetRequiredService<IActionResultExecutor<ObjectResult>>();

      return executor.ExecuteAsync(context, this);   // <-------------------------------------------------------------
   }

   public virtual void OnFormatting(ActionContext context)
   {
      if (Value is ProblemDetails details)
      {
         if (details.Status != null && StatusCode == null)
         {
            StatusCode = details.Status;
         }
         else if (details.Status == null && StatusCode != null)
         {
            details.Status = StatusCode;
         }
      }

      if (StatusCode.HasValue)
      {
         context.HttpContext.Response.StatusCode = StatusCode.Value;
      }
   }
}
//-----------------------Ʌ

//-------------------------V
public class OkObjectResult : ObjectResult
{
   private const int DefaultStatusCode = StatusCodes.Status200OK;
   
   public OkObjectResult(object value) : base(value)
   {
      StatusCode = DefaultStatusCode;
   }
}
//-------------------------Ʌ
```

```C#
//---------------------------------------V
public partial class ObjectResultExecutor : IActionResultExecutor<ObjectResult>
{
   public ObjectResultExecutor(
      OutputFormatterSelector formatterSelector,
      IHttpResponseStreamWriterFactory writerFactory,
      ILoggerFactory loggerFactory,
      IOptions<MvcOptions> mvcOptions)
   {
      FormatterSelector = formatterSelector;
      WriterFactory = writerFactory.CreateWriter;
      Logger = loggerFactory.CreateLogger<ObjectResultExecutor>();
   }

   protected ILogger Logger { get; }

   protected OutputFormatterSelector FormatterSelector { get; }

   protected Func<Stream, Encoding, TextWriter> WriterFactory { get; }

   public virtual Task ExecuteAsync(ActionContext context, ObjectResult result)
   {
      InferContentTypes(context, result);
 
      var objectType = result.DeclaredType;
 
      if (objectType == null || objectType == typeof(object))
      {
         objectType = result.Value?.GetType();
      }

      object value = result.Value;
      return ExecuteAsyncCore(context, result, objectType, value);
   }

   private Task ExecuteAsyncCore(ActionContext context, ObjectResult result, Type? objectType, object? value)
   {
      var formatterContext = new OutputFormatterWriteContext(context.HttpContext, WriterFactory, objectType, value);

      var selectedFormatter = FormatterSelector.SelectFormatter(
         formatterContext,
         (IList<IOutputFormatter>)result.Formatters ?? Array.Empty<IOutputFormatter>(),
         result.ContentTypes);

      if (selectedFormatter == null)
      {
         // No formatter supports this.
         Log.NoFormatter(Logger, formatterContext, result.ContentTypes);
 
         const int statusCode = StatusCodes.Status406NotAcceptable;
         context.HttpContext.Response.StatusCode = statusCode;
 
         if (context.HttpContext.RequestServices.GetService<IProblemDetailsService>() is { } problemDetailsService)
         {
            return problemDetailsService.WriteAsync(new()
            {
               HttpContext = context.HttpContext,
               ProblemDetails = { Status = statusCode }
            }).AsTask();
         }
 
         return Task.CompletedTask;
      }

      Log.ObjectResultExecuting(Logger, result, value);
 
      result.OnFormatting(context);
      return selectedFormatter.WriteAsync(formatterContext);    // <------------eventually calls context.HttpContext.Response.WriteAsync
   }

   private static void InferContentTypes(ActionContext context, ObjectResult result)
   {
      // If the user sets the content type both on the ObjectResult (example: by Produces) and Response object,
      // then the one set on ObjectResult takes precedence over the Response object
      var responseContentType = context.HttpContext.Response.ContentType;
      if (result.ContentTypes.Count == 0 && !string.IsNullOrEmpty(responseContentType))
      {
         result.ContentTypes.Add(responseContentType);
      }
 
      if (result.Value is ProblemDetails)
      {
         result.ContentTypes.Add("application/problem+json");
         result.ContentTypes.Add("application/problem+xml");
      }
   }
}
//---------------------------------------Ʌ


public abstract class TextOutputFormatter : OutputFormatter { ... }
public abstract class OutputFormatter : IOutputFormatter, IApiResponseTypeMetadataProvider { ...}

//--------------------------------V
public class StringOutputFormatter : TextOutputFormatter   // most simple formatter
{
   public StringOutputFormatter()
   {
      SupportedEncodings.Add(Encoding.UTF8);
      SupportedEncodings.Add(Encoding.Unicode);
      SupportedMediaTypes.Add("text/plain");
   }

   public override bool CanWriteResult(OutputFormatterCanWriteContext context)
   {
      if (context.ObjectType == typeof(string) || context.Object is string)
      {
         // call into base to check if the current request's content type is a supported media type.
         return base.CanWriteResult(context);
      }
 
      return false;
   }

   public override Task WriteResponseBodyAsync(OutputFormatterWriteContext context, Encoding encoding)
   {
      var valueAsString = (string?)context.Object;
      if (string.IsNullOrEmpty(valueAsString))
      {
         return Task.CompletedTask;
      }
 
      var response = context.HttpContext.Response;          
      return response.WriteAsync(valueAsString, encoding);  // <---------------------
   }
}
//--------------------------------Ʌ


```






































-------------------------------------------------------------------------------------------------

## Build Your Own `IActionResult`

```C#
public static class ServiceCollectionExtensions
{
   public static IServiceCollection AddMvcControllers(this IServiceCollection services)
   {
      return services
         .AddSingleton<IActionDescriptorCollectionProvider, DefaultActionDescriptorCollectionProvider>()
         .AddSingleton<IActionInvokerFactory, ActionInvokerFactory>()
         .AddSingleton <IActionDescriptorProvider, ControllerActionDescriptorProvider>()
         .AddSingleton<ControllerActionEndpointDataSource, ControllerActionEndpointDataSource>()
         .AddSingleton<IActionResultTypeMapper, ActionResultTypeMapper>();
   }
}

public class FoobarController : Controller
{
   private static readonly string _html = @"<html>...<body><p>Hello World!</p></body></html>";

   [HttpGet("/{foo}")]
   public Task<IActionResult> FooAsync()
   {
      return Task.FromResult<IActionResult>(new ContentResult(_html, "text/html"));
   }

   [HttpGet("/bar")]
   public ValueTask<ContentResult> BarAsync() 
   {
      return new ValueTask<ContentResult>(new ContentResult(_html, "text/html"));
   }

   [HttpGet("/baz")]
   public Task<string> BazAsync() 
   {
      Task.FromResult(_html);
   }

   [HttpGet("/qux")]
   public ValueTask<string> QuxAsync() 
   {
      new ValueTask<string>(_html);
   }
}
```

**Part A**

```C#
public interface IActionResult
{
   Task ExecuteResultAsync(ActionContext context);
}

public class ContentResult : IActionResult
{
   private readonly string _content;
   private readonly string _contentType;

   public ContentResult(string content, string contentType)
   {
      _content = content;
       _contentType = contentType;
   }

   public Task ExecuteResultAsync(ActionContext context)
   {
      var response = context.HttpContext.Response;
      response.ContentType = _contentType;
      return response.WriteAsync(_content);
   }
}

public sealed class NullActionResult : IActionResult
{
   private NullActionResult() { }
   public static NullActionResult Instance { get; } = new NullActionResult();
   public Task ExecuteResultAsync(ActionContext context) => Task.CompletedTask;
}
```

**Part B**

```C#
public interface IActionInvoker
{
   Task InvokeAsync();
}

public class ControllerActionInvoker : IActionInvoker
{
   public ActionContext ActionContext { get; }

   public ControllerActionInvoker(ActionContext actionContext) => ActionContext = actionContext;
   
   public Task InvokeAsync()
   {
      var actionDescriptor = (ControllerActionDescriptor)ActionContext.ActionDescriptor;
      var controllerType = actionDescriptor.ControllerType;
      var requestServies = ActionContext.HttpContext.RequestServices;
      var controllerInstance = ActivatorUtilities.CreateInstance(requestServies, controllerType);
      
      if (controllerInstance is Controller controller)
      {
         controller.ActionContext = ActionContext;
      }

      var actionMethod = actionDescriptor.Method;

      var result = actionMethod.Invoke(controllerInstance, new object[0]);

      var actionResult = await ToActionResultAsync(result);

      await actionResult.ExecuteResultAsync(ActionContext);
   }
}

private async Task<IActionResult> ToActionResultAsync(object result)
{
   if (result == null)
      return NullActionResult.Instance;
   
   if (result is Task<IActionResult> taskOfActionResult)
      return await taskOfActionResult;

   if (result is ValueTask<IActionResult> valueTaskOfActionResult)
      return await valueTaskOfActionResult;

   if (result is IActionResult actionResult)
      return actionResult;

   if (result is Task task)
   {
      await task;
      return NullActionResult.Instance;
   }

   throw new InvalidOperationException("Action method's return value is invalid.");
}
```

**Part C**

return type can be any type:

```C#
public interface IActionResultTypeMapper
{
    IActionResult Convert(object value, Type returnType);
}

public class ActionResultTypeMapper : IActionResultTypeMapper
{
   public IActionResult Convert(object value, Type returnType)
   {
      new ContentResult(value.ToString(), "text/plain");
   }
}

//----------------------------------V
public class ControllerActionInvoker : IActionInvoker
{
   private static readonly MethodInfo _taskConvertMethod;
   private static readonly MethodInfo _valueTaskConvertMethod;

   static ControllerActionInvoker()
   {
      var bindingFlags = BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.Static;
      _taskConvertMethod = typeof(ControllerActionInvoker).GetMethod(nameof(ConvertFromTaskAsync), bindingFlags);
      _valueTaskConvertMethod = typeof(ControllerActionInvoker).GetMethod(nameof(ConvertFromValueTaskAsync), bindingFlags);
   }

   private static async Task<IActionResult> ConvertFromTaskAsync<TValue>(Task<TValue> returnValue, IActionResultTypeMapper mapper)
   {
      var result = await returnValue;
      return result is IActionResult actionResult ? actionResult : mapper.Convert(result, typeof(TValue));
   }

   private static async Task<IActionResult> ConvertFromValueTaskAsync<TValue>( ValueTask<TValue> returnValue, IActionResultTypeMapper mapper)
   {
      var result = await returnValue;
      return result is IActionResult actionResult ? actionResult : mapper.Convert(result, typeof(TValue));
   }

   public async Task InvokeAsync()
   {
      var actionDescriptor = (ControllerActionDescriptor) ActionContext.ActionDescriptor;
      var controllerType = actionDescriptor.ControllerType;
      var requestServies = ActionContext.HttpContext.RequestServices;
      var controllerInstance = ActivatorUtilities.CreateInstance(requestServies, controllerType);

      if (controllerInstance is Controller controller)
      {
         controller.ActionContext = ActionContext;
      }

      var actionMethod = actionDescriptor.Method;
      var returnValue = actionMethod.Invoke(controllerInstance, new object[0]);
      var mapper = requestServies.GetRequiredService<IActionResultTypeMapper>();
      var actionResult = await ToActionResultAsync(returnValue, actionMethod.ReturnType, mapper);

      await actionResult.ExecuteResultAsync(ActionContext);
   }

   private Task<IActionResult> ToActionResultAsync(object returnValue, Type returnType, IActionResultTypeMapper mapper)
   {
       if (returnValue == null || returnType == typeof(Task) || returnType == typeof(ValueTask))
          return Task.FromResult<IActionResult>(NullActionResult.Instance);

       if (returnValue is IActionResult actionResult)
          return Task.FromResult(actionResult);

       if (returnType.IsGenericType && returnType.GetGenericTypeDefinition() == typeof(Task<>))
       {
          var declaredType = returnType.GenericTypeArguments.Single();
          var taskOfResult = _taskConvertMethod.MakeGenericMethod(declaredType).Invoke(null, new object[] { returnValue, mapper });
          return (Task<IActionResult>)taskOfResult;
       }

       if (returnType.IsGenericType && returnType.GetGenericTypeDefinition() == typeof(ValueTask<>))
       {
          var declaredType = returnType.GenericTypeArguments.Single();
          var valueTaskOfResult = _valueTaskConvertMethod.MakeGenericMethod(declaredType).Invoke(null, new object[] { returnValue, mapper });
          return (Task<IActionResult>)valueTaskOfResult;
       }

       return Task.FromResult(mapper.Convert(returnValue, returnType));
   }
}
//----------------------------------Ʌ
```








































compare endpoints.MapControllers() and endpoints.MapDefaultControllerRoute() to see if MapControllers() is still needed when using MapDefaultControllerRoute()




Now whenever you run your application and try to access a view then what the Razor View Engine does is, it converts the View code into a C# class file. It will not convert the view to a C# class until and unless you try to access the view.



-Q
how does output get generated
how does ActionContext get created
how IActionResult get rendered, combined with previous restful web api chapter

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