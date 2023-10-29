## Demystifying Model Binding
=============================

```C#
internal partial class ControllerActionInvoker : ResourceInvoker, IActionInvoker    
{
   private readonly ControllerActionInvokerCacheEntry _cacheEntry;
   private readonly ControllerContext _controllerContext;
   private Dictionary<string, object> _arguments;    // <---------------------
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
        : base(diagnosticListener, logger, actionContextAccessor, mapper, controllerContext, 
               filters, controllerContext.ValueProviderFactories)
   {
      _cacheEntry = cacheEntry;
      _controllerContext = controllerContext;
   }

   private Task Next(ref State next, ref Scope scope, ref object? state, ref bool isCompleted)   
   {
      switch (next)
      {
         case State.ActionBegin:   
            var controllerContext = _controllerContext;
            _cursor.Reset();

            _instance = _cacheEntry.ControllerFactory(controllerContext);   
                                                                            
            _arguments = new Dictionary<string, object?>(StringComparer.OrdinalIgnoreCase);

            var task = BindArgumentsAsync();   // <------------------------------------------------1.0

            if (task.Status != TaskStatus.RanToCompletion)
            {
               next = State.ActionNext;
               return task;
            }

            goto case State.ActionNext;

         case State.ActionNext:
            var current = _cursor.GetNextFilter<IActionFilter, IAsyncActionFilter>();
            // ...
         
         // ...
         
         case State.ActionInside:
            var task = InvokeActionMethodAsync();   
            if (task.Status != TaskStatus.RanToCompletion)
            {
               next = State.ActionEnd;
               return task;
            }
 
            goto case State.ActionEnd;
        
         // ...
      }
   }
   
   private Task InvokeActionMethodAsync()
   {
      var objectMethodExecutor = _cacheEntry.ObjectMethodExecutor;
      var actionMethodExecutor = _cacheEntry.ActionMethodExecutor;
      var orderedArguments = PrepareArguments(_arguments, objectMethodExecutor);
 
      var actionResultValueTask = actionMethodExecutor.Execute(ControllerContext, 
                                                               _mapper, 
                                                               objectMethodExecutor, 
                                                               _instance!, 
                                                               orderedArguments);
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

   // ...

   private Task BindArgumentsAsync()     // <---------------------------------------1.1
   {
      var actionDescriptor = _controllerContext.ActionDescriptor;
      if (actionDescriptor.BoundProperties.Count == 0 && actionDescriptor.Parameters.Count == 0)
      {
         return Task.CompletedTask;
      }
       
      return _cacheEntry.ControllerBinderDelegate(_controllerContext,   // <----------1.2+, which connects to 2.4b
                                                  _instance!,
                                                  _arguments!);  // _arguments is Empty Dictionary
   }

   private static object?[]? PrepareArguments(IDictionary<string, object?>? actionParameters, 
                                              ObjectMethodExecutor actionMethodExecutor)
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
}
```

```C#
//-----------------------------------------------------V
internal sealed class ControllerActionInvokerCacheEntry
{
   internal ControllerActionInvokerCacheEntry(
      FilterItem[] cachedFilters,
      Func<ControllerContext, object> controllerFactory,
      Func<ControllerContext, object, ValueTask>? controllerReleaser,
      ControllerBinderDelegate? controllerBinderDelegate,
      ObjectMethodExecutor objectMethodExecutor,
      ActionMethodExecutor actionMethodExecutor,
      ActionMethodExecutor innerActionMethodExecutor)
   {
      // ...
   }

   public FilterItem[] CachedFilters { get; }

   public Func<ControllerContext, object> ControllerFactory { get; }

   public Func<ControllerContext, object, ValueTask>? ControllerReleaser { get; }
 
   public ControllerBinderDelegate? ControllerBinderDelegate { get; }   // <---------------------
 
   internal ObjectMethodExecutor ObjectMethodExecutor { get; }
 
   internal ActionMethodExecutor ActionMethodExecutor { get; }
 
   internal ActionMethodExecutor InnerActionMethodExecutor { get; }
}
//-----------------------------------------------------Ʌ

internal delegate Task ControllerBinderDelegate(ControllerContext controllerContext, object controller, Dictionary<string, object?> arguments);


//------------------------------------------------V
internal sealed class ControllerActionInvokerCache               
{
   private readonly ParameterBinder _parameterBinder;
   private readonly IModelBinderFactory _modelBinderFactory;
   private readonly IModelMetadataProvider _modelMetadataProvider;
   private readonly IFilterProvider[] _filterProviders;
   private readonly IControllerFactoryProvider _controllerFactoryProvider;
   private readonly MvcOptions _mvcOptions;

   public ControllerActionInvokerCache(
      ParameterBinder parameterBinder,
      IModelBinderFactory modelBinderFactory,
      IModelMetadataProvider modelMetadataProvider,
      IEnumerable<IFilterProvider> filterProviders,
      IControllerFactoryProvider factoryProvider,
      IOptions<MvcOptions> mvcOptions)
   {
      _parameterBinder = parameterBinder;
      _modelBinderFactory = modelBinderFactory;
      _modelMetadataProvider = modelMetadataProvider;
      _filterProviders = filterProviders.OrderBy(item => item.Order).ToArray();
      _controllerFactoryProvider = factoryProvider;
     _mvcOptions = mvcOptions.Value;
   }

   public (ControllerActionInvokerCacheEntry cacheEntry, IFilterMetadata[] filters) GetCachedResult(ControllerContext controllerContext)
   {
      var actionDescriptor = controllerContext.ActionDescriptor;
      IFilterMetadata[] filters;
      var cacheEntry = actionDescriptor.CacheEntry;

      if (cacheEntry is null)
      {
         var filterFactoryResult = FilterFactory.GetAllFilters(_filterProviders, controllerContext);   
         filters = filterFactoryResult.Filters;                                                        
                                                                                                       
         var parameterDefaultValues = ParameterDefaultValues.GetParameterDefaultValues(actionDescriptor.MethodInfo);

         var objectMethodExecutor = ObjectMethodExecutor.Create(
            actionDescriptor.MethodInfo,
            actionDescriptor.ControllerTypeInfo,
            parameterDefaultValues);

         var controllerFactory = _controllerFactoryProvider.CreateControllerFactory(actionDescriptor);  
         var controllerReleaser = _controllerFactoryProvider.CreateAsyncControllerReleaser(actionDescriptor);   
         ControllerBinderDelegate propertyBinderFactory = ControllerBinderDelegateProvider.CreateBinderDelegate(   //----------------------------2.1
            _parameterBinder,
            _modelBinderFactory,
            _modelMetadataProvider,
            actionDescriptor,
            _mvcOptions);
         
         var actionMethodExecutor = ActionMethodExecutor.GetExecutor(objectMethodExecutor);
         var filterExecutor = actionDescriptor.FilterDelegate is not null ? ActionMethodExecutor.GetFilterExecutor(actionDescriptor) : null;

         cacheEntry = new ControllerActionInvokerCacheEntry(
            filterFactoryResult.CacheableFilters,
            controllerFactory,                      
            controllerReleaser,
            propertyBinderFactory,   // <---------------------------------------------2.0
            objectMethodExecutor,
            filterExecutor ?? actionMethodExecutor,
            actionMethodExecutor);

         actionDescriptor.CacheEntry = cacheEntry;   
      }
      else 
      {
         // filter instances from statically defined filter descriptors + from filter providers
         filters = FilterFactory.CreateUncachedFilters(_filterProviders, controllerContext, cacheEntry.CachedFilters);
      }

      return (cacheEntry, filters);   
   }
}
//------------------------------------------------Ʌ
```




## Main


```C#
//----------------------------------------------V
public abstract class ModelBinderProviderContext
{
   public abstract IModelBinder CreateBinder(ModelMetadata metadata);

   public virtual IModelBinder CreateBinder(ModelMetadata metadata, BindingInfo bindingInfo)
   {
      throw new NotSupportedException();
   }

   public abstract BindingInfo BindingInfo { get; }

   public abstract ModelMetadata Metadata { get; }

   public abstract IModelMetadataProvider MetadataProvider { get; }

   public virtual IServiceProvider Services { get; } = default!;
}
//----------------------------------------------Ʌ

//----------------------------------------------------V
private sealed class DefaultModelBinderProviderContext : ModelBinderProviderContext   // a private class of ModelBinderFactory
{
   private readonly ModelBinderFactory _factory;

   public DefaultModelBinderProviderContext(ModelBinderFactory factory, ModelBinderFactoryContext factoryContext)
   {
      _factory = factory;
      Metadata = factoryContext.Metadata;
      BindingInfo bindingInfo;
      if (factoryContext.BindingInfo != null)
      {
         bindingInfo = new BindingInfo(factoryContext.BindingInfo);
      }
      else
      {
         bindingInfo = new BindingInfo();
      }
 
      bindingInfo.TryApplyBindingInfo(Metadata);
      BindingInfo = bindingInfo;
 
      MetadataProvider = _factory._metadataProvider;
      Visited = new Dictionary<Key, IModelBinder?>();
   }

   private DefaultModelBinderProviderContext(DefaultModelBinderProviderContext parent, ModelMetadata metadata, BindingInfo bindingInfo)
   {
      Metadata = metadata;
      _factory = parent._factory;
      MetadataProvider = parent.MetadataProvider;
      Visited = parent.Visited;
      BindingInfo = bindingInfo;
   }

   public override BindingInfo BindingInfo { get; }

   public override ModelMetadata Metadata { get; }

   public override IModelMetadataProvider MetadataProvider { get; }

   public Dictionary<Key, IModelBinder?> Visited { get; }    // <-------------------------

   public override IServiceProvider Services => _factory._serviceProvider;

   public override IModelBinder CreateBinder(ModelMetadata metadata)
   {
      var bindingInfo = new BindingInfo();
      bindingInfo.TryApplyBindingInfo(metadata);

      return CreateBinder(metadata, bindingInfo);
   }

   public override IModelBinder CreateBinder(ModelMetadata metadata, BindingInfo bindingInfo)
   {
      var token = metadata;

      var nestedContext = new DefaultModelBinderProviderContext(this, metadata, bindingInfo);
      return _factory.CreateBinderCoreCached(nestedContext, token);
   }
}
//----------------------------------------------------Ʌ

//----------------------------------------------------V
internal static class ControllerBinderDelegateProvider       
{
   public static ControllerBinderDelegate? CreateBinderDelegate(   // <-------------------2.2
      ParameterBinder parameterBinder,
      IModelBinderFactory modelBinderFactory,
      IModelMetadataProvider modelMetadataProvider,
      ControllerActionDescriptor actionDescriptor,
      MvcOptions mvcOptions)
   {
      BinderItem[] parameterBindingInfo = GetParameterBindingInfo(modelBinderFactory, modelMetadataProvider, actionDescriptor);  // <-------------------2.3->
      BinderItem[] propertyBindingInfo = GetPropertyBindingInfo(modelBinderFactory, modelMetadataProvider, actionDescriptor);
 
      if (parameterBindingInfo == null && propertyBindingInfo == null)
      {
         return null;
      }

      var parameters = actionDescriptor.Parameters switch
      {
         List<ParameterDescriptor> list => list.ToArray(),
         _ => actionDescriptor.Parameters.ToArray()
      };
 
      var properties = actionDescriptor.BoundProperties switch
      {
         List<ParameterDescriptor> list => list.ToArray(),
         _ => actionDescriptor.BoundProperties.ToArray()
      };
 
      return Bind;   // <-------------------2.4a

      async Task Bind(ControllerContext controllerContext, object controller, Dictionary<string, object?> arguments)  // <-------------------2.4b
      {
         var (success, valueProvider) = await CompositeValueProvider
                                              .TryCreateAsync(controllerContext, controllerContext.ValueProviderFactories);    // <-----------4.0
         if (!success)
         {
            return;
         }

         for (var i = 0; i < parameters.Length; i++)
         {
            ParameterDescriptor parameter = parameters[i];
            BinderItem bindingInfo = parameterBindingInfo![i];   // <-------------------4.1
            ModelMetadata modelMetadata = bindingInfo.ModelMetadata;
 
            if (!modelMetadata.IsBindingAllowed)
            {
               continue;
            }
 
            ModelBindingResult result = await parameterBinder.BindModelAsync(    // <------------------4.2a
               controllerContext,
               bindingInfo.ModelBinder,
               valueProvider,
               parameter,
               modelMetadata,
               value: null,
               container: null); // Parameters do not have containers.
 
            if (result.IsModelSet)
            {
               arguments[parameter.Name] = result.Model;   // <-----------------------!4.3
            }
         }

         for (var i = 0; i < properties.Length; i++)
         {
            var property = properties[i];
            var bindingInfo = propertyBindingInfo![i];
            var modelMetadata = bindingInfo.ModelMetadata;
 
            if (!modelMetadata.IsBindingAllowed)
            {
               continue;
            }
 
            var result = await parameterBinder.BindModelAsync(
               controllerContext,
               bindingInfo.ModelBinder,
               valueProvider,
               property,
               modelMetadata,
               value: null,
               container: controller);
 
            if (result.IsModelSet)
            {
               PropertyValueSetter.SetValue(bindingInfo.ModelMetadata, controller, result.Model);
            }
         }
      }
   }
   
   private static BinderItem[]? GetParameterBindingInfo(    // <-----------------2.3.0
      IModelBinderFactory modelBinderFactory,
      IModelMetadataProvider modelMetadataProvider,
      ControllerActionDescriptor actionDescriptor)
   {
      IList<ParameterDescriptor> parameters = actionDescriptor.Parameters;
      if (parameters.Count == 0)
      {
         return null;
      }

      BinderItem[] parameterBindingInfo = new BinderItem[parameters.Count];
      for (var i = 0; i < parameters.Count; i++)
      {
         ParameterDescriptor parameter = parameters[i];

         ModelMetadata metadata;
         if (modelMetadataProvider is ModelMetadataProvider modelMetadataProviderBase && parameter is ControllerParameterDescriptor controllerParameterDescriptor)
         {
            // the default model metadata provider derives from ModelMetadataProvider and can therefore supply information about attributes applied to parameters.
            metadata = modelMetadataProviderBase.GetMetadataForParameter(controllerParameterDescriptor.ParameterInfo);   // <-----------------2.3.1
         }
         else
         {
            // for backward compatibility, if there's a custom model metadata provider that only implements the older IModelMetadataProvider interface, access the more
            // limited metadata information it supplies. In this scenario, validation attributes are not supported on parameters.
            metadata = modelMetadataProvider.GetMetadataForType(parameter.ParameterType);
         }

         IModelBinder binder = modelBinderFactory.CreateBinder(new ModelBinderFactoryContext()    // <-----------------2.3.8->3.0
         {
            BindingInfo = parameter.BindingInfo,  // <--------------- check _BindingInfo_ section in previous Demystifying ControllerActionInvoker
            Metadata = metadata,                  // <----------------this determines which IModelBinder later
            CacheToken = parameter,
         });

         parameterBindingInfo[i] = new BinderItem(binder, metadata);   // <-----------------2.3.9_end
      }

      return parameterBindingInfo;
   }

   private static BinderItem[]? GetPropertyBindingInfo(
      IModelBinderFactory modelBinderFactory,
      IModelMetadataProvider modelMetadataProvider,
      ControllerActionDescriptor actionDescriptor)
   { ... }

   private readonly struct BinderItem
   {
      public BinderItem(IModelBinder modelBinder, ModelMetadata modelMetadata)
      {
         ModelBinder = modelBinder;
         ModelMetadata = modelMetadata;
      }

      public IModelBinder ModelBinder { get; }

      public ModelMetadata ModelMetadata { get; }
   }
}
//----------------------------------------------------Ʌ

//---------------------------------------V
public readonly struct ModelBindingResult : IEquatable<ModelBindingResult>
{
   public static ModelBindingResult Failed()
   {
      return new ModelBindingResult(model: null, isModelSet: false);
   }

   public static ModelBindingResult Success(object? model)  // <----------------
   {
      return new ModelBindingResult(model, isModelSet: true);
   }

   private ModelBindingResult(object? model, bool isModelSet)
   {
      Model = model;
      IsModelSet = isModelSet;
   }

   public object? Model { get; }

   public bool IsModelSet { get; }

   public bool Equals(ModelBindingResult other)
   {
      return IsModelSet == other.IsModelSet && object.Equals(Model, other.Model);
   }
}
//---------------------------------------Ʌ

//---------------------------------V
public abstract class ModelMetadata : IEquatable<ModelMetadata?>, IModelMetadataProvider
{
   public static readonly int DefaultOrder = 10000;

   private static readonly ParameterBindingMethodCache ParameterBindingMethodCache = new(throwOnInvalidMethod: false);

   private int? _hashCode;
   private IReadOnlyList<ModelMetadata>? _boundProperties;
   private IReadOnlyDictionary<ModelMetadata, ModelMetadata>? _parameterMapping;
   private IReadOnlyDictionary<ModelMetadata, ModelMetadata>? _boundConstructorPropertyMapping;
   private Exception? _recordTypeValidatorsOnPropertiesError;
   private bool _recordTypeConstructorDetailsCalculated;

   protected ModelMetadata(ModelMetadataIdentity identity)
   {
      Identity = identity;
      InitializeTypeInformation();
   }

   public Type? ContainerType => Identity.ContainerType;

   public virtual ModelMetadata? ContainerMetadata
   {
      get {
         throw new NotImplementedException();
      }
   }

   public ModelMetadataKind MetadataKind => Identity.MetadataKind;

   public Type ModelType => Identity.ModelType;

   public string? Name => Identity.Name;

   public string? ParameterName => MetadataKind == ModelMetadataKind.Parameter ? Identity.Name : null;

   public string? PropertyName => MetadataKind == ModelMetadataKind.Property ? Identity.Name : null;

   protected internal ModelMetadataIdentity Identity { get; }

   public abstract IReadOnlyDictionary<object, object> AdditionalValues { get; }

   public abstract ModelPropertyCollection Properties { get; }

   internal IReadOnlyList<ModelMetadata> BoundProperties
   {
      get {
         if (BoundConstructor is null)
         {
            return Properties;
         }

         if (_boundProperties is null)
         {
            var boundParameters = BoundConstructor.BoundConstructorParameters!;
            var boundProperties = new List<ModelMetadata>();
 
            foreach (var metadata in Properties)
            {
               if (!boundParameters.Any(p => string.Equals(p.ParameterName, metadata.PropertyName, StringComparison.Ordinal) && p.ModelType == metadata.ModelType))
               {
                  boundProperties.Add(metadata);
               }
            }
 
            _boundProperties = boundProperties;
         }
 
         return _boundProperties;
      }
   }

   internal IReadOnlyDictionary<ModelMetadata, ModelMetadata> BoundConstructorParameterMapping
   {
      get {
         Debug.Assert(BoundConstructor != null, "This API can be only called for types with bound constructors.");
         CalculateRecordTypeConstructorDetails();
 
         return _parameterMapping;
      }
   }

   internal IReadOnlyDictionary<ModelMetadata, ModelMetadata> BoundConstructorPropertyMapping
   {
      get {
         Debug.Assert(BoundConstructor != null, "This API can be only called for types with bound constructors.");
         CalculateRecordTypeConstructorDetails();
 
         return _boundConstructorPropertyMapping;
      }
   }

   public virtual ModelMetadata? BoundConstructor { get; }

   public virtual IReadOnlyList<ModelMetadata>? BoundConstructorParameters { get; }

   public abstract string? BinderModelName { get; }

   public abstract Type? BinderType { get; }

   public abstract BindingSource? BindingSource { get; }

   public abstract bool ConvertEmptyStringToNull { get; }

   public abstract string? DataTypeName { get; }

   public abstract string? DisplayName { get; }

   public abstract bool IsRequired { get; }

   // ...
}
//---------------------------------Ʌ

//--------------------------V
public class BindingMetadata
{
   private Type? _binderType;
   
   private DefaultModelBindingMessageProvider? _messageProvider;
   
   public BindingSource? BindingSource { get; set; }
   
   public string? BinderModelName { get; set; }
   
   public Type? BinderType
   {
      get => _binderType;
      set {
         _binderType = value;
      }
   }

   public bool IsBindingAllowed { get; set; } = true;
   
   public bool IsBindingRequired { get; set; }
   
   public bool? IsReadOnly { get; set; }
   
   public DefaultModelBindingMessageProvider? ModelBindingMessageProvider
   {
      get => _messageProvider;
      set {
         _messageProvider = value;
      }
   }

   public IPropertyFilterProvider? PropertyFilterProvider { get; set; }

   public ConstructorInfo? BoundConstructor { get; set; }
}
//--------------------------Ʌ

//-------------------------------V
public class DefaultModelMetadata : ModelMetadata
{
   private readonly IModelMetadataProvider _provider;
   private readonly ICompositeMetadataDetailsProvider _detailsProvider;
   private readonly DefaultMetadataDetails _details;

   private readonly DefaultModelBindingMessageProvider _modelBindingMessageProvider;

   private ReadOnlyDictionary<object, object>? _additionalValues;
   private ModelMetadata? _elementMetadata;
   private ModelMetadata? _constructorMetadata;
   private bool? _isBindingRequired;
   private bool? _isReadOnly;
   private bool? _isRequired;
   private ModelPropertyCollection? _properties;
   private bool? _validateChildren;
   private bool? _hasValidators;
   private ReadOnlyCollection<object>? _validatorMetadata;

   public DefaultModelMetadata(
      IModelMetadataProvider provider,
      ICompositeMetadataDetailsProvider detailsProvider,
      DefaultMetadataDetails details)
      : this(provider, detailsProvider, details, new DefaultModelBindingMessageProvider())
   {

   }

   public DefaultModelMetadata(
      IModelMetadataProvider provider,
      ICompositeMetadataDetailsProvider detailsProvider,
      DefaultMetadataDetails details,
      DefaultModelBindingMessageProvider modelBindingMessageProvider)
      : base(details.Key)
   {
      _provider = provider;
      _detailsProvider = detailsProvider;
      _details = details;
      _modelBindingMessageProvider = modelBindingMessageProvider;
   }

   public ModelAttributes Attributes => _details.ModelAttributes;

   public override ModelMetadata? ContainerMetadata => _details.ContainerMetadata;

   public BindingMetadata BindingMetadata
   {
      get {
         if (_details.BindingMetadata == null)
         {
            var context = new BindingMetadataProviderContext(Identity, _details.ModelAttributes);
            context.BindingMetadata.ModelBindingMessageProvider = new DefaultModelBindingMessageProvider(_modelBindingMessageProvider);

            _detailsProvider.CreateBindingMetadata(context);
            _details.BindingMetadata = context.BindingMetadata;
         }

         return _details.BindingMetadata;
      }
   }

   public DisplayMetadata DisplayMetadata
   {
      get {
         if (_details.DisplayMetadata == null)
         {
            var context = new DisplayMetadataProviderContext(Identity, _details.ModelAttributes);
            _detailsProvider.CreateDisplayMetadata(context);
            _details.DisplayMetadata = context.DisplayMetadata;
         }

         return _details.DisplayMetadata;
      }
   }

   public ValidationMetadata ValidationMetadata
   {
      get {
         if (_details.ValidationMetadata == null)
         {
            var context = new ValidationMetadataProviderContext(Identity, _details.ModelAttributes);
            _detailsProvider.CreateValidationMetadata(context);
            _details.ValidationMetadata = context.ValidationMetadata;
         }

         return _details.ValidationMetadata;
      }
   }

   public override IReadOnlyDictionary<object, object> AdditionalValues
   {
      get {
         return _additionalValues;
      }
   }

   public override BindingSource? BindingSource => BindingMetadata.BindingSource;

   public override string? BinderModelName => BindingMetadata.BinderModelName;

   public override Type? BinderType => BindingMetadata.BinderType;

   public override ModelMetadata? ElementMetadata
   {
      get {
         if (_elementMetadata == null && ElementType != null)
         {
            _elementMetadata = _provider.GetMetadataForType(ElementType);
         }
         
         return _elementMetadata;
      }
   }

   public override bool IsBindingAllowed
   {
      get {
         if (MetadataKind == ModelMetadataKind.Type)
            return true;
         else
            return BindingMetadata.IsBindingAllowed;
      }
   }

   public override bool IsBindingRequired
   {
      get {
         if (!_isBindingRequired.HasValue)
         {
            if (MetadataKind == ModelMetadataKind.Type)
               _isBindingRequired = false;
            else
               _isBindingRequired = BindingMetadata.IsBindingRequired;
         }

         return _isBindingRequired.Value;
      }
   }

   public override bool IsRequired
   {
      get {
         if (!_isRequired.HasValue)
         {
            if (ValidationMetadata.IsRequired.HasValue)
               _isRequired = ValidationMetadata.IsRequired;
            else
               _isRequired = !IsReferenceOrNullableType;
         }

         return _isRequired.Value;
      }
   }

   public override ModelMetadata? BoundConstructor
   {
      get {
         if (BindingMetadata.BoundConstructor == null)
            return null;

         if (_constructorMetadata == null)
         {
            var modelMetadataProvider = (ModelMetadataProvider)_provider;
            _constructorMetadata = modelMetadataProvider.GetMetadataForConstructor(BindingMetadata.BoundConstructor, ModelType);
         }

         return _constructorMetadata;
      }
   }

   public override IReadOnlyList<ModelMetadata>? BoundConstructorParameters => _details.BoundConstructorParameters;

   public override IPropertyFilterProvider? PropertyFilterProvider => BindingMetadata.PropertyFilterProvider;

   public override Func<object?[], object>? BoundConstructorInvoker => _details.BoundConstructorInvoker;

   // ...
}
//-------------------------------Ʌ

public interface IModelMetadataProvider
{
   ModelMetadata GetMetadataForType(Type modelType);

   IEnumerable<ModelMetadata> GetMetadataForProperties(Type modelType);
}

public abstract class ModelMetadataProvider : IModelMetadataProvider
{
   public abstract IEnumerable<ModelMetadata> GetMetadataForProperties(Type modelType);
   public abstract ModelMetadata GetMetadataForType(Type modelType);
   public abstract ModelMetadata GetMetadataForParameter(ParameterInfo parameter);
   public virtual ModelMetadata GetMetadataForParameter(ParameterInfo parameter, Type modelType) => throw new NotSupportedException();
   public virtual ModelMetadata GetMetadataForProperty(PropertyInfo propertyInfo, Type modelType) => throw new NotSupportedException();
   public virtual ModelMetadata GetMetadataForConstructor(ConstructorInfo constructor, Type modelType) => throw new NotSupportedException();
}

//---------------------------------------V
public class DefaultModelMetadataProvider : ModelMetadataProvider
{
   private readonly ConcurrentDictionary<ModelMetadataIdentity, ModelMetadataCacheEntry> _modelMetadataCache = new();
   private readonly Func<ModelMetadataIdentity, ModelMetadataCacheEntry> _cacheEntryFactory;   // <------------------
   private readonly ModelMetadataCacheEntry _metadataCacheEntryForObjectType;

   public DefaultModelMetadataProvider(ICompositeMetadataDetailsProvider detailsProvider) : this(detailsProvider, new DefaultModelBindingMessageProvider()) { }

   public DefaultModelMetadataProvider(ICompositeMetadataDetailsProvider detailsProvider, IOptions<MvcOptions> optionsAccessor) 
      : this(detailsProvider, GetMessageProvider(optionsAccessor)) { }

   private DefaultModelMetadataProvider(ICompositeMetadataDetailsProvider detailsProvider, DefaultModelBindingMessageProvider modelBindingMessageProvider)
   {
      DetailsProvider = detailsProvider;
      ModelBindingMessageProvider = modelBindingMessageProvider;
 
      _cacheEntryFactory = CreateCacheEntry;    // <---------------------------------------2.3.4
      _metadataCacheEntryForObjectType = GetMetadataCacheEntryForObjectType();
   }

   protected ICompositeMetadataDetailsProvider DetailsProvider { get; }

   protected DefaultModelBindingMessageProvider ModelBindingMessageProvider { get; }

   internal void ClearCache() => _modelMetadataCache.Clear();

   private ModelMetadataCacheEntry GetMetadataCacheEntryForObjectType()
   {
      var key = ModelMetadataIdentity.ForType(typeof(object));
      var entry = CreateCacheEntry(key);
      return entry;
   }

   private ModelMetadataCacheEntry CreateCacheEntry(ModelMetadataIdentity key)   // <--------------2.3.4
   {
      DefaultMetadataDetails details;
 
      if (key.MetadataKind == ModelMetadataKind.Constructor)
      {
         details = CreateConstructorDetails(key);
      }
      else if (key.MetadataKind == ModelMetadataKind.Parameter)
      {
         details = CreateParameterDetails(key);     // <--------------2.3.5
      }
      else if (key.MetadataKind == ModelMetadataKind.Property)
      {
         details = CreateSinglePropertyDetails(key);
      }
      else
      {
         details = CreateTypeDetails(key);
      }
 
      ModelMetadata metadata = CreateModelMetadata(details);   // <---------------2.3.7a, create ModelMetadata instance
            
      return new ModelMetadataCacheEntry(metadata, details);
   }

   protected virtual DefaultMetadataDetails CreateParameterDetails(ModelMetadataIdentity key)   
   {
      return new DefaultMetadataDetails(key, ModelAttributes.GetAttributesForParameter(key.ParameterInfo!, key.ModelType));  // <--------------2.3.6
   }

   protected virtual ModelMetadata CreateModelMetadata(DefaultMetadataDetails entry)
   {
      return new DefaultModelMetadata(this, DetailsProvider, entry, ModelBindingMessageProvider);     // <---------------2.3.7b_end
   }

   public override IEnumerable<ModelMetadata> GetMetadataForProperties(Type modelType)
   {
      var cacheEntry = GetCacheEntry(modelType);

      if (cacheEntry.Details.Properties == null)
      {
         var key = ModelMetadataIdentity.ForType(modelType);
         var propertyDetails = CreatePropertyDetails(key);
 
         var properties = new ModelMetadata[propertyDetails.Length];
         for (var i = 0; i < properties.Length; i++)
         {
            propertyDetails[i].ContainerMetadata = cacheEntry.Metadata;
            properties[i] = CreateModelMetadata(propertyDetails[i]);
         }
 
         cacheEntry.Details.Properties = properties;
      }

      return cacheEntry.Details.Properties;
   }

   public override ModelMetadata GetMetadataForParameter(ParameterInfo parameter)   // <-----------2.3.2a
      => GetMetadataForParameter(parameter, parameter.ParameterType);

   public override ModelMetadata GetMetadataForParameter(ParameterInfo parameter, Type modelType)
   {
      var cacheEntry = GetCacheEntry(parameter, modelType);   // <-----------2.3.2b
      return cacheEntry.Metadata;
   }

   public override ModelMetadata GetMetadataForType(Type modelType)
   {
      var cacheEntry = GetCacheEntry(modelType);
      return cacheEntry.Metadata;
   }

   public override ModelMetadata GetMetadataForProperty(PropertyInfo propertyInfo, Type modelType)
   {
      var cacheEntry = GetCacheEntry(propertyInfo, modelType);
      return cacheEntry.Metadata;
   }

   public override ModelMetadata GetMetadataForConstructor(ConstructorInfo constructorInfo, Type modelType)
   {
      var cacheEntry = GetCacheEntry(constructorInfo, modelType);
      return cacheEntry.Metadata;
   }

   private static DefaultModelBindingMessageProvider GetMessageProvider(IOptions<MvcOptions> optionsAccessor)
   {
      return optionsAccessor.Value.ModelBindingMessageProvider;
   }

   private ModelMetadataCacheEntry GetCacheEntry(Type modelType)
   {
      ModelMetadataCacheEntry cacheEntry;

      if (modelType == typeof(object))
      {
         cacheEntry = _metadataCacheEntryForObjectType;
      }
      else
      {
         var key = ModelMetadataIdentity.ForType(modelType);
 
         cacheEntry = _modelMetadataCache.GetOrAdd(key, _cacheEntryFactory);
      }

      return cacheEntry;
   }

   private ModelMetadataCacheEntry GetCacheEntry(ParameterInfo parameter, Type modelType)   // <----------------2.3.3
   {
      return _modelMetadataCache.GetOrAdd(
         ModelMetadataIdentity.ForParameter(parameter, modelType), _cacheEntryFactory);
   }

   // ...

   private readonly struct ModelMetadataCacheEntry
   {
      public ModelMetadataCacheEntry(ModelMetadata metadata, DefaultMetadataDetails details)
      {
         Metadata = metadata;
         Details = details;
      }
 
      public ModelMetadata Metadata { get; }
 
      public DefaultMetadataDetails Details { get; }
   }
}
//---------------------------------------Ʌ

//------------------------------------------V
public readonly struct ModelMetadataIdentity : IEquatable<ModelMetadataIdentity>
{
   private ModelMetadataIdentity(Type modelType, string? name = null, Type? containerType = null, object? fieldInfo = null, ConstructorInfo? constructorInfo = null)
   {
      ModelType = modelType;
      Name = name;
      ContainerType = containerType;
      FieldInfo = fieldInfo;
      ConstructorInfo = constructorInfo;
   }

   public Type? ContainerType { get; }
   public Type ModelType { get; }
   public string? Name { get; }
   private object? FieldInfo { get; }

   public ParameterInfo? ParameterInfo => FieldInfo as ParameterInfo;
   public PropertyInfo? PropertyInfo => FieldInfo as PropertyInfo;
   public ConstructorInfo? ConstructorInfo { get; }
   
   public ModelMetadataKind MetadataKind
   {
      get {
         if (ParameterInfo != null)
         {   
            return ModelMetadataKind.Parameter;
         }
         else if (ConstructorInfo != null)
         {
            return ModelMetadataKind.Constructor;
         }
         else if (ContainerType != null && Name != null)
         {
            return ModelMetadataKind.Property;
         }
         else
         {
            return ModelMetadataKind.Type;
         }
      }
   }

   public static ModelMetadataIdentity ForType(Type modelType)
   {
      return new ModelMetadataIdentity(modelType);
   }

   public static ModelMetadataIdentity ForProperty(Type modelType, string name, Type containerType)
   {
       return new ModelMetadataIdentity(modelType, name, containerType);
   }

   public static ModelMetadataIdentity ForParameter(ParameterInfo parameter, Type modelType)
   {
      return new ModelMetadataIdentity(modelType, parameter.Name, fieldInfo: parameter);
   }

   public static ModelMetadataIdentity ForConstructor(ConstructorInfo constructor, Type modelType)
   {
      return new ModelMetadataIdentity(modelType, constructor.Name, constructorInfo: constructor);
   }

   public enum ModelMetadataKind
   {
      Type,
      Property,
      Parameter,
      Constructor
   }
}
//------------------------------------------Ʌ

//---------------------------------V
public class DefaultMetadataDetails  // holds associated metadata objects for a DefaultModelMetadata
{
   public DefaultMetadataDetails(ModelMetadataIdentity key, ModelAttributes attributes)
   {
      Key = key;
      ModelAttributes = attributes;
   }

   public ModelAttributes ModelAttributes { get; }
   public BindingMetadata? BindingMetadata { get; set; }
   public DisplayMetadata? DisplayMetadata { get; set; }
   public ModelMetadataIdentity Key { get; }
   public ModelMetadata[]? Properties { get; set; }
   public ModelMetadata[]? BoundConstructorParameters { get; set; }
   public Func<object, object?>? PropertyGetter { get; set; }
   public Action<object, object?>? PropertySetter { get; set; }
   public Func<object?[], object>? BoundConstructorInvoker { get; set; }
   public ValidationMetadata? ValidationMetadata { get; set; }
   public ModelMetadata? ContainerMetadata { get; set; }
}
//---------------------------------Ʌ

//--------------------------V
public class ModelAttributes
{
   internal static readonly ModelAttributes Empty = new ModelAttributes(Array.Empty<object>());

   internal ModelAttributes(IReadOnlyList<object> attributes)
   {
      Attributes = attributes;
   }

   internal ModelAttributes(IEnumerable<object> typeAttributes, IEnumerable<object>? propertyAttributes, IEnumerable<object>? parameterAttributes)
   {
      if (propertyAttributes != null)
      {
         PropertyAttributes = propertyAttributes.ToArray();
         TypeAttributes = typeAttributes.ToArray();
         Attributes = PropertyAttributes.Concat(TypeAttributes).ToArray();
      }
      else if (parameterAttributes != null)
      {
         ParameterAttributes = parameterAttributes.ToArray();
         TypeAttributes = typeAttributes.ToArray();
         Attributes = ParameterAttributes.Concat(TypeAttributes).ToArray();
      }
      else if (typeAttributes != null)
      {
         Attributes = TypeAttributes = typeAttributes.ToArray();
      }
      else
      {
         Attributes = Array.Empty<object>();
      }
   }

   public IReadOnlyList<object> Attributes { get; }
   public IReadOnlyList<object>? PropertyAttributes { get; }
   public IReadOnlyList<object>? ParameterAttributes { get; }
   public IReadOnlyList<object>? TypeAttributes { get; }

   public static ModelAttributes GetAttributesForParameter(ParameterInfo parameterInfo, Type modelType)
   {
      var typeAttributes = GetAttributesForType(modelType).TypeAttributes!;
      var parameterAttributes = parameterInfo.GetCustomAttributes();
 
      return new ModelAttributes(typeAttributes, propertyAttributes: null, parameterAttributes);
   }

   public static ModelAttributes GetAttributesForType(Type type)
   {
      var attributes = type.GetCustomAttributes();
 
      var metadataType = GetMetadataType(type);
      if (metadataType != null)
      {
         attributes = attributes.Concat(metadataType.GetCustomAttributes());
      }
 
      return new ModelAttributes(attributes, propertyAttributes: null, parameterAttributes: null);
   }

   // public static ModelAttributes GetAttributesForProperty(Type containerType, PropertyInfo property, Type modelType);

   private static Type? GetMetadataType(Type type)
   {
      return type.GetCustomAttribute<ModelMetadataTypeAttribute>()?.MetadataType;
   }
}
//--------------------------Ʌ

public class ModelBinderFactoryContext
{
   public BindingInfo? BindingInfo { get; set; }

   public ModelMetadata Metadata { get; set; } = default!;

   public object? CacheToken { get; set; }
}

public interface IModelBinderFactory
{
   IModelBinder CreateBinder(ModelBinderFactoryContext context);
}

//-------------------------------------V
public partial class ModelBinderFactory : IModelBinderFactory
{
   private readonly IModelMetadataProvider _metadataProvider;   // used in DefaultModelBinderProviderContext
   private readonly IModelBinderProvider[] _providers;
   private readonly ConcurrentDictionary<Key, IModelBinder> _cache;
   private readonly IServiceProvider _serviceProvider;

   public ModelBinderFactory(IModelMetadataProvider metadataProvider, IOptions<MvcOptions> options, IServiceProvider serviceProvider)
   {
      _metadataProvider = metadataProvider;
      _providers = options.Value.ModelBinderProviders.ToArray();
      _serviceProvider = serviceProvider;
      _cache = new ConcurrentDictionary<Key, IModelBinder>();
   }

   public IModelBinder CreateBinder(ModelBinderFactoryContext context)   // <---------------------3.0
   {
      if (TryGetCachedBinder(context.Metadata, context.CacheToken, out var binder))
      {
         return binder;
      }

      var providerContext = new DefaultModelBinderProviderContext(this, context);   // <-----------------3.1
      binder = CreateBinderCoreUncached(providerContext, context.CacheToken);       // <-----------------3.2a

      AddToCache(context.Metadata, context.CacheToken, binder);

      return binder;
   }

   private IModelBinder CreateBinderCoreCached(DefaultModelBinderProviderContext providerContext, object? token)   // <-----------------3.2b
   {
      if (TryGetCachedBinder(providerContext.Metadata, token, out var binder))
      {
         return binder;
      }

      // we're definitely creating a binder for an non-root node here, 
      //so it's OK for binder creation to fail
      binder = CreateBinderCoreUncached(providerContext, token)    // <------------------------------3.3
                  ?? NoOpBinder.Instance;   

      if (!(binder is PlaceholderBinder))
      {
         AddToCache(providerContext.Metadata, token, binder);
      }

      return binder;
   }

   private IModelBinder? CreateBinderCoreUncached(DefaultModelBinderProviderContext providerContext, object? token)   // <------------------3.4
   {
      if (!providerContext.Metadata.IsBindingAllowed)
      {
         return NoOpBinder.Instance;
      }

      var key = new Key(providerContext.Metadata, token);

      var visited = providerContext.Visited;

      if (visited.TryGetValue(key, out var binder))
      {
         if (binder != null)
         {
            return binder;
         }
 
         // if we're currently recursively building a binder for this type, just return a PlaceholderBinder.
         // we'll fix it later to point to the 'real' binder when the stack unwinds.
         binder = new PlaceholderBinder();
         visited[key] = binder;
         return binder;
      }

      // OK this isn't a recursive case (yet) so add an entry and then ask the providers to create the binder
      visited.Add(key, null);

      IModelBinder? result = null;

      for (var i = 0; i < _providers.Length; i++)
      {
         var provider = _providers[i];
         result = provider.GetBinder(providerContext);   // <------------------3.5
         if (result != null)
         {
            break;
         }
      }

      // if the PlaceholderBinder was created, then it means we recursed. Hook it up to the 'real' binder.
      if (visited[key] is PlaceholderBinder placeholderBinder)
      {
         placeholderBinder.Inner = result ?? NoOpBinder.Instance;
      }

      if (result != null)
      {
         visited[key] = result;
      }

      return result;
   }

   private void AddToCache(ModelMetadata metadata, object? cacheToken, IModelBinder binder)
   {
      if (cacheToken == null)
         return;
      
      _cache.TryAdd(new Key(metadata, cacheToken), binder);
   }

   private bool TryGetCachedBinder(ModelMetadata metadata, object? cacheToken, [NotNullWhen(true)] out IModelBinder? binder)
   {
      if (cacheToken == null)
      {
         binder = null;
         return false;
      }
 
      return _cache.TryGetValue(new Key(metadata, cacheToken), out binder);
   }
}
//-------------------------------------Ʌ


//-----------------------------------------------------------------------------------------------V
public interface IValueProvider
{
   bool ContainsPrefix(string prefix);
   ValueProviderResult GetValue(string key);
}

public interface IEnumerableValueProvider : IValueProvider
{
   IDictionary<string, string> GetKeysFromPrefix(string prefix);
}

public interface IBindingSourceValueProvider : IValueProvider
{
   IValueProvider? Filter(BindingSource bindingSource);
}

// >
public readonly struct ValueProviderResult : IEquatable<ValueProviderResult>, IEnumerable<string>
{
   private static readonly CultureInfo _invariantCulture = CultureInfo.InvariantCulture;
   public static ValueProviderResult None = new ValueProviderResult(Array.Empty<string>());

   public ValueProviderResult(StringValues values) : this(values, _invariantCulture) { }

   public CultureInfo Culture { get; }
   public StringValues Values { get; }

   public string? FirstValue 
   {
      get {
         if (Values.Count == 0)
            return null;
            
         return Values[0];
      }
   }

   public int Length => Values.Count;

   public IEnumerator<string> GetEnumerator() => ((IEnumerable<string>)Values).GetEnumerator();
} 
// <

// >
public class ValueProviderFactoryContext 
{  
   public ValueProviderFactoryContext(ActionContext context)
   {
      ActionContext = context;
   }

   public ActionContext ActionContext { get; }
   
   public IList<IValueProvider> ValueProviders { get; } = new List<IValueProvider>();
}  
// <

public interface IValueProviderFactory
{
   Task CreateValueProviderAsync(ValueProviderFactoryContext context);
}
//-----------------------------------------------------------------------------------------------Ʌ

public class QueryStringValueProviderFactory : IValueProviderFactory
{
   public Task CreateValueProviderAsync(ValueProviderFactoryContext context)
   {
      var query = context.ActionContext.HttpContext.Request.Query;
      if (query != null && query.Count > 0)
      {
         var valueProvider = new QueryStringValueProvider(BindingSource.Query, query, CultureInfo.InvariantCulture);
         context.ValueProviders.Add(valueProvider);
      }

      return Task.CompletedTask;
   }
}

public class FormValueProviderFactory : IValueProviderFactory
{
   public Task CreateValueProviderAsync(ValueProviderFactoryContext context)
   {
      var request = context.ActionContext.HttpContext.Request;
      if (request.HasFormContentType)
         return AddValueProviderAsync(context);   // allocating a Task only when the body is form data

      return Task.CompletedTask;
   }

   private static async Task AddValueProviderAsync(ValueProviderFactoryContext context)
   {
      var request = context.ActionContext.HttpContext.Request;
      IFormCollection form;

      try {
         form = await request.ReadFormAsync();
      }
      catch (InvalidDataException ex) {  
         throw new ValueProviderException(Resources.FormatFailedToReadRequestForm(ex.Message), ex);
      }
      catch (IOException ex) {
         throw new ValueProviderException(Resources.FormatFailedToReadRequestForm(ex.Message), ex);
      }

      var valueProvider = new FormValueProvider(BindingSource.Form, form, CultureInfo.CurrentCulture);
 
      context.ValueProviders.Add(valueProvider);
   }
}

//----------------------------->>
public interface IValueProvider
{
   bool ContainsPrefix(string prefix);
   ValueProviderResult GetValue(string key);
}

public interface IBindingSourceValueProvider : IValueProvider
{
   IValueProvider? Filter(BindingSource bindingSource);
}
//-----------------------------<<

//----------------------------------------------------------------------------V
public abstract class BindingSourceValueProvider : IBindingSourceValueProvider
{
   public BindingSourceValueProvider(BindingSource bindingSource)
   {
      if (bindingSource.IsGreedy) {
         var message = Resources.FormatBindingSource_CannotBeGreedy(bindingSource.DisplayName, nameof(BindingSourceValueProvider));
         throw new ArgumentException(message, nameof(bindingSource));
      }

      if (bindingSource is CompositeBindingSource) {
         var message = Resources.FormatBindingSource_CannotBeComposite(bindingSource.DisplayName, nameof(BindingSourceValueProvider));
         throw new ArgumentException(message, nameof(bindingSource));
      }

      BindingSource = bindingSource;
   }

   protected BindingSource BindingSource { get; }
   public abstract bool ContainsPrefix(string prefix);
   public abstract ValueProviderResult GetValue(string key);

   public virtual IValueProvider? Filter(BindingSource bindingSource)
   {
      if (bindingSource.CanAcceptDataFrom(BindingSource))
      {
         return this;
      }     
      else
      {
         return null;
      }
   }
}
//----------------------------------------------------------------------------Ʌ

//------------------------------------------------------------------------------------------V
public class QueryStringValueProvider : BindingSourceValueProvider, IEnumerableValueProvider
{
   private readonly IQueryCollection _values;
   private PrefixContainer? _prefixContainer;

   public QueryStringValueProvider(BindingSource bindingSource, IQueryCollection values, CultureInfo? culture) : base(bindingSource)
   {
      _values = values;
      Culture = culture;
   }

   public CultureInfo? Culture { get; }

   protected PrefixContainer PrefixContainer
   {
      get {
         if (_prefixContainer == null)
            _prefixContainer = new PrefixContainer(_values.Keys);
 
         return _prefixContainer;
      }
   }

   public override bool ContainsPrefix(string prefix) => PrefixContainer.ContainsPrefix(prefix);

   public virtual IDictionary<string, string> GetKeysFromPrefix(string prefix) => PrefixContainer.GetKeysFromPrefix(prefix);

   public override ValueProviderResult GetValue(string key)
   {
      if (key.Length == 0)
         return ValueProviderResult.None;
      
      var values = _values[key];
      if (values.Count == 0)
         return ValueProviderResult.None;
      else
         return new ValueProviderResult(values, Culture);
   }
}
//------------------------------------------------------------------------------------------Ʌ

//-----------------------------------------------------------------------------------V
public class FormValueProvider : BindingSourceValueProvider, IEnumerableValueProvider
{
   private readonly IFormCollection _values;
   private readonly HashSet<string?>? _invariantValueKeys;
   private PrefixContainer? _prefixContainer;

   public FormValueProvider(BindingSource bindingSource, IFormCollection values, CultureInfo? culture) : base(bindingSource)
   {
      _values = values;
 
      if (_values.TryGetValue(FormValueHelper.CultureInvariantFieldName, out var invariantKeys) && invariantKeys.Count > 0)
      {
         _invariantValueKeys = new(invariantKeys, StringComparer.OrdinalIgnoreCase);
      }
 
      Culture = culture;
   }

   public CultureInfo? Culture { get; }

   protected PrefixContainer PrefixContainer {
      get {
         if (_prefixContainer == null)
            _prefixContainer = new PrefixContainer(_values.Keys);
         return _prefixContainer;
        }
   }

   public override bool ContainsPrefix(string prefix) => PrefixContainer.ContainsPrefix(prefix);
   
   public virtual IDictionary<string, string> GetKeysFromPrefix(string prefix) => PrefixContainer.GetKeysFromPrefix(prefix);
   
   public override ValueProviderResult GetValue(string key)
   {
      if (key.Length == 0)
         return ValueProviderResult.None;
         
      var values = _values[key];
      if (values.Count == 0)
      {
         return ValueProviderResult.None;
      }
      else
      {
         var culture = _invariantValueKeys?.Contains(key) == true ? CultureInfo.InvariantCulture : Culture;
         return new ValueProviderResult(values, culture);
      }
   }
}
//-----------------------------------------------------------------------------------Ʌ

// public class RouteValueProvider : BindingSourceValueProvider { ... }

//---------------------------------V
public class CompositeValueProvider :  Collection<IValueProvider>, IEnumerableValueProvider, IBindingSourceValueProvider, IKeyRewriterValueProvider
{
   public CompositeValueProvider() { }

   public CompositeValueProvider(IList<IValueProvider> valueProviders) : base(valueProviders) { }

   internal static async ValueTask<(bool success, CompositeValueProvider? valueProvider)> TryCreateAsync(ActionContext actionContext, IList<IValueProviderFactory> factories)
   {
      try
      {
         var valueProvider = await CreateAsync(actionContext, factories);
         return (true, valueProvider);
      }
      catch (ValueProviderException exception)
      {
         actionContext.ModelState.TryAddModelException(key: string.Empty, exception);
         return (false, null);
      }
   }

   public static async Task<CompositeValueProvider> CreateAsync(ControllerContext controllerContext)
   {
      var factories = controllerContext.ValueProviderFactories;
      return await CreateAsync(controllerContext, factories);
   }

   public static async Task<CompositeValueProvider> CreateAsync(ActionContext actionContext, IList<IValueProviderFactory> factories)
   {
      var valueProviderFactoryContext = new ValueProviderFactoryContext(actionContext);

      for (var i = 0; i < factories.Count; i++)
      {
         var factory = factories[i];
         await factory.CreateValueProviderAsync(valueProviderFactoryContext);
      }

      return new CompositeValueProvider(valueProviderFactoryContext.ValueProviders);
   }

   public virtual bool ContainsPrefix(string prefix)
   {
      for (var i = 0; i < Count; i++)
      {
         if (this[i].ContainsPrefix(prefix))
         {
            return true;
         }
      }
      return false;
   }

   public virtual ValueProviderResult GetValue(string key)
   {
      var itemCount = Items.Count;
      for (var i = 0; i < itemCount; i++)
      {
         var valueProvider = Items[i];
         var result = valueProvider.GetValue(key);
         if (result != ValueProviderResult.None)
         {
            return result;
         }
      }

      return ValueProviderResult.None;
   }

   public virtual IDictionary<string, string> GetKeysFromPrefix(string prefix)
   {
      foreach (var valueProvider in this)
      {
         if (valueProvider is IEnumerableValueProvider enumeratedProvider)
         {
            var result = enumeratedProvider.GetKeysFromPrefix(prefix);
            if (result != null && result.Count > 0)
            {
               return result;
            }
         }
      }
      return new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
   }

   protected override void InsertItem(int index, IValueProvider item)
   {
      base.InsertItem(index, item);
   }

   protected override void SetItem(int index, IValueProvider item)
   {
      base.SetItem(index, item);
   }

   public IValueProvider? Filter(BindingSource bindingSource)
   {
      var shouldFilter = false;
      for (var i = 0; i < Count; i++)
      {
         var valueProvider = Items[i];
         if (valueProvider is IBindingSourceValueProvider)
         {
            shouldFilter = true;
            break;
         }
      }

      if (!shouldFilter)
      {
         // no inner IBindingSourceValueProvider implementations. Result will be empty
         return null;
      }

      var filteredValueProviders = new List<IValueProvider>();
      for (var i = 0; i < Count; i++)
      {
         var valueProvider = Items[i];
         if (valueProvider is IBindingSourceValueProvider bindingSourceValueProvider)
         {
            var result = bindingSourceValueProvider.Filter(bindingSource);
            if (result != null)
            {
               filteredValueProviders.Add(result);
            }
         }
      }
 
      if (filteredValueProviders.Count == 0)
      {
         // do not create an empty CompositeValueProvider.
         return null;
      }
 
      return new CompositeValueProvider(filteredValueProviders);
   }

   public IValueProvider? Filter()
   {
      var shouldFilter = false;
      for (var i = 0; i < Count; i++)
      {
         var valueProvider = Items[i];
         if (valueProvider is IKeyRewriterValueProvider)
         {
            shouldFilter = true;
            break;
         }
      }

      if (!shouldFilter)
      {
         // no inner IKeyRewriterValueProvider implementations. Nothing to exclude.
         return this;
      }

      var filteredValueProviders = new List<IValueProvider>();
      for (var i = 0; i < Count; i++)
      {
         var valueProvider = Items[i];
         if (valueProvider is IKeyRewriterValueProvider keyRewriterValueProvider)
         {
            var result = keyRewriterValueProvider.Filter();
            if (result != null)
            {
               filteredValueProviders.Add(result);
            }
         }
         else 
         {
            // assume value providers that aren't rewriter-aware do not rewrite their keys.
            filteredValueProviders.Add(valueProvider);
         }
      }

      if (filteredValueProviders.Count == 0)
      {
         // Do not create an empty CompositeValueProvider.
         return null;
      }
 
      return new CompositeValueProvider(filteredValueProviders);
   }
}
//---------------------------------Ʌ

//------------------------V
public class BindingSource : IEquatable<BindingSource?>
{
   public static readonly BindingSource Body = new BindingSource("Body", Resources.BindingSource_Body, isGreedy: true, isFromRequest: true);
   public static readonly BindingSource Custom = new BindingSource("Custom", Resources.BindingSource_Custom, isGreedy: true, isFromRequest: true);
   public static readonly BindingSource Form = new BindingSource("Form", Resources.BindingSource_Form, isGreedy: false, isFromRequest: true);
   public static readonly BindingSource Header = new BindingSource("Header", Resources.BindingSource_Header, isGreedy: true, isFromRequest: true);
   public static readonly BindingSource ModelBinding = new BindingSource("ModelBinding", Resources.BindingSource_ModelBinding, isGreedy: false, isFromRequest: true);
   public static readonly BindingSource Path = new BindingSource("Path", Resources.BindingSource_Path, isGreedy: false, isFromRequest: true);     // url request url path
   public static readonly BindingSource Query = new BindingSource("Query", Resources.BindingSource_Query, isGreedy: false, isFromRequest: true);  // query-string
   public static readonly BindingSource Services = new BindingSource("Services", Resources.BindingSource_Services, isGreedy: true, isFromRequest: false);
   public static readonly BindingSource Special = new BindingSource("Special", Resources.BindingSource_Special, isGreedy: true, isFromRequest: false);  // when not user input
   public static readonly BindingSource FormFile = new BindingSource("FormFile", Resources.BindingSource_FormFile, isGreedy: true, isFromRequest: true);

   public BindingSource(string id, string displayName, bool isGreedy, bool isFromRequest)
   {
      Id = id;
      DisplayName = displayName;
      IsGreedy = isGreedy;
      IsFromRequest = isFromRequest;
   }

   public string DisplayName { get; }
   public string Id { get; }
   public bool IsGreedy { get; }
   public bool IsFromRequest { get; }

   public virtual bool CanAcceptDataFrom(BindingSource bindingSource)
   {
      if (bindingSource is CompositeBindingSource)
         throw new ArgumentNullException(nameof(bindingSource));

      if (bindingSource is CompositeBindingSource)
      {
         var message = Resources.FormatBindingSource_CannotBeComposite(bindingSource.DisplayName, nameof(CanAcceptDataFrom));
         throw new ArgumentException(message, nameof(bindingSource));
      }

      if (this == bindingSource)
      {
         return true;
      }

      if (this == ModelBinding)
      {
         return bindingSource == Form || bindingSource == Path || bindingSource == Query;
      }
 
      return false;
   }
}
//------------------------Ʌ

//---------------------------------V
public class CompositeBindingSource : BindingSource
{
   public static CompositeBindingSource Create(IEnumerable<BindingSource> bindingSources, string displayName)
   {
      foreach (var bindingSource in bindingSources)
      {
         if (bindingSource.IsGreedy) {
            var message = Resources.FormatBindingSource_CannotBeGreedy(bindingSource.DisplayName, nameof(CompositeBindingSource));
            throw new ArgumentException(message, nameof(bindingSources));
         }

         if (!bindingSource.IsFromRequest) {
            var message = Resources.FormatBindingSource_MustBeFromRequest(bindingSource.DisplayName, nameof(CompositeBindingSource));
            throw new ArgumentException(message, nameof(bindingSources));
         }

         if (bindingSource is CompositeBindingSource) {
            var message = Resources.FormatBindingSource_CannotBeComposite(bindingSource.DisplayName, nameof(CompositeBindingSource));
            throw new ArgumentException(message, nameof(bindingSources));
         }
      }

      var id = string.Join("&", bindingSources.Select(s => s.Id).OrderBy(s => s, StringComparer.Ordinal));
      return new CompositeBindingSource(id, displayName, bindingSources);
   }

   private CompositeBindingSource(string id, string displayName, IEnumerable<BindingSource> bindingSources) : base(id, displayName, isGreedy: false, isFromRequest: true)
   {
      BindingSources = bindingSources;
   }

   public IEnumerable<BindingSource> BindingSources { get; }

   public override bool CanAcceptDataFrom(BindingSource bindingSource)
   {
      if (bindingSource is CompositeBindingSource)
        throw new ArgumentException(message, nameof(bindingSource));

      foreach (var source in BindingSources)
      {
         if (source.CanAcceptDataFrom(bindingSource))
         {
            return true;
         }
      }

      return false;
   }
}
//---------------------------------Ʌ

//----------------------V
public class BindingInfo
{
   private Type? _binderType;

   public BindingInfo() { }

   public BindingInfo(BindingInfo other)
   {
      BindingSource = other.BindingSource;
      BinderModelName = other.BinderModelName;
      BinderType = other.BinderType;
      PropertyFilterProvider = other.PropertyFilterProvider;
      RequestPredicate = other.RequestPredicate;
      EmptyBodyBehavior = other.EmptyBodyBehavior;
   }

   public BindingSource? BindingSource { get; set; }

   public string? BinderModelName { get; set; }

   public Type? BinderType
   {
      get => _binderType;
      set {
         if (value != null && !typeof(IModelBinder).IsAssignableFrom(value))
            throw new ArgumentException(Resources.FormatBinderType_MustBeIModelBinder(value.FullName, typeof(IModelBinder).FullName), nameof(value));
         
         _binderType = value;
      }
   }

   public IPropertyFilterProvider? PropertyFilterProvider { get; set; }

   public Func<ActionContext, bool>? RequestPredicate { get; set; }  // a predicate determines if the model should be bound based on state from the current request

   public EmptyBodyBehavior EmptyBodyBehavior { get; set; }  // gets or sets the value which decides if empty bodies are treated as valid inputs

   public static BindingInfo? GetBindingInfo(IEnumerable<object> attributes)
   {
      var bindingInfo = new BindingInfo();
      var isBindingInfoPresent = false;

      foreach (var binderModelNameAttribute in attributes.OfType<IModelNameProvider>())  // BinderModelName
      {
         isBindingInfoPresent = true;
         if (binderModelNameAttribute?.Name != null)
         {
            bindingInfo.BinderModelName = binderModelNameAttribute.Name;
            break;
         }
      }

      foreach (var binderTypeAttribute in attributes.OfType<IBinderTypeProviderMetadata>())  // BinderType
      {
         isBindingInfoPresent = true;
         if (binderTypeAttribute.BinderType != null)
         {
            bindingInfo.BinderType = binderTypeAttribute.BinderType;
            break;
         }
      }
    
      foreach (var bindingSourceAttribute in attributes.OfType<IBindingSourceMetadata>())  // BindingSource
      {
         isBindingInfoPresent = true;
         if (bindingSourceAttribute.BindingSource != null)
         {
            bindingInfo.BindingSource = bindingSourceAttribute.BindingSource;
            break;
         }
      }

      var propertyFilterProviders = attributes.OfType<IPropertyFilterProvider>().ToArray();  // PropertyFilterProvider
      if (propertyFilterProviders.Length == 1)
      {
         isBindingInfoPresent = true;
         bindingInfo.PropertyFilterProvider = propertyFilterProviders[0];
      }
      else if (propertyFilterProviders.Length > 1)
      {
         isBindingInfoPresent = true;
         bindingInfo.PropertyFilterProvider = new CompositePropertyFilterProvider(propertyFilterProviders);
      }

      foreach (var requestPredicateProvider in attributes.OfType<IRequestPredicateProvider>())  // RequestPredicate
      {
         isBindingInfoPresent = true;
         if (requestPredicateProvider.RequestPredicate != null)
         {
            bindingInfo.RequestPredicate = requestPredicateProvider.RequestPredicate;
            break;
         }
      }

      foreach (var configureEmptyBodyBehavior in attributes.OfType<IConfigureEmptyBodyBehavior>())
      {
         isBindingInfoPresent = true;
         bindingInfo.EmptyBodyBehavior = configureEmptyBodyBehavior.EmptyBodyBehavior;
         break;
      }
 
      return isBindingInfoPresent ? bindingInfo : null;
   }

   public static BindingInfo? GetBindingInfo(IEnumerable<object> attributes, ModelMetadata modelMetadata)
   {
      var bindingInfo = GetBindingInfo(attributes);
      var isBindingInfoPresent = bindingInfo != null;

      if (bindingInfo == null)
      {
         bindingInfo = new BindingInfo();
      }

      isBindingInfoPresent |= bindingInfo.TryApplyBindingInfo(modelMetadata);
      
      return isBindingInfoPresent ? bindingInfo : null;
   }

   public bool TryApplyBindingInfo(ModelMetadata modelMetadata)
   {
      var isBindingInfoPresent = false;
      if (BinderModelName == null && modelMetadata.BinderModelName != null)
      {
         isBindingInfoPresent = true;
         BinderModelName = modelMetadata.BinderModelName;
      }
 
      if (BinderType == null && modelMetadata.BinderType != null)
      {
         isBindingInfoPresent = true;
         BinderType = modelMetadata.BinderType;
      }
 
      if (BindingSource == null && modelMetadata.BindingSource != null)
      {
         isBindingInfoPresent = true;
         BindingSource = modelMetadata.BindingSource;
      }
 
      if (PropertyFilterProvider == null && modelMetadata.PropertyFilterProvider != null)
      {
         isBindingInfoPresent = true;
         PropertyFilterProvider = modelMetadata.PropertyFilterProvider;
      }

      if (EmptyBodyBehavior == EmptyBodyBehavior.Default && BindingSource == BindingSource.Body &&
         (modelMetadata.NullabilityState == NullabilityState.Nullable || modelMetadata.IsNullableValueType || modelMetadata.HasDefaultValue))
      {
         isBindingInfoPresent = true;
         EmptyBodyBehavior = EmptyBodyBehavior.Allow;
      }
 
      return isBindingInfoPresent;
   }

   private sealed class CompositePropertyFilterProvider : IPropertyFilterProvider
   {
      private readonly IEnumerable<IPropertyFilterProvider> _providers;
 
      public CompositePropertyFilterProvider(IEnumerable<IPropertyFilterProvider> providers)
      {
          _providers = providers;
      }
 
      public Func<ModelMetadata, bool> PropertyFilter => CreatePropertyFilter();

      private Func<ModelMetadata, bool> CreatePropertyFilter()
      {
         var propertyFilters = _providers.Select(p => p.PropertyFilter).Where(p => p != null);

         return (m) => 
         {
            foreach (var propertyFilter in propertyFilters) {
               if (!propertyFilter(m))
                  return false;         
            }
 
            return true;
         };
      }
   }
}
//----------------------Ʌ

//---------------------------------------V
public abstract class ModelBindingContext
{
   public abstract ActionContext ActionContext
   public abstract string? BinderModelName { get; set; }
   public abstract BindingSource? BindingSource { get; set; }
   public abstract string FieldName { get; set; }
   public virtual HttpContext HttpContext => ActionContext?.HttpContext!;
   public abstract bool IsTopLevelObject { get; set; }
   public abstract object? Model { get; set; }
   public abstract ModelMetadata ModelMetadata { get; set; }
   public abstract string ModelName { get; set; }
   public string OriginalModelName { get; protected set; } = default!;
   public abstract ModelStateDictionary ModelState { get; set; }
   public virtual Type ModelType => ModelMetadata.ModelType;
   public abstract Func<ModelMetadata, bool>? PropertyFilter { get; set; }
   public abstract ValidationStateDictionary ValidationState { get; set; }
   public abstract IValueProvider ValueProvider { get; set; }
   public abstract ModelBindingResult Result { get; set; }
   public abstract NestedScope EnterNestedScope(ModelMetadata modelMetadata, string fieldName, string modelName, object? model);
   public abstract NestedScope EnterNestedScope();
   protected abstract void ExitNestedScope();
   
   public readonly struct NestedScope : IDisposable
   {
      private readonly ModelBindingContext _context;

      public NestedScope(ModelBindingContext context)
      {
         _context = context;
      }

      public void Dispose()
      {
         _context.ExitNestedScope();
      }
   }
}
//---------------------------------------Ʌ

//-------------------------------------V
public class DefaultModelBindingContext : ModelBindingContext
{
   private static readonly IValueProvider EmptyValueProvider = new CompositeValueProvider();
 
   private IValueProvider _originalValueProvider = default!;
   private ActionContext _actionContext = default!;
   private ModelStateDictionary _modelState = default!;
   private ValidationStateDictionary _validationState = default!;
   private int? _maxModelBindingRecursionDepth;
 
   private State _state;
   private readonly Stack<State> _stack = new Stack<State>();

   public override ActionContext ActionContext { get; set; }
  
   public override string FieldName
   {
      get { return _state.FieldName; }
      set { _state.FieldName = value; }
   }

   public override object? Model
   {
      get { return _state.Model; }
      set { _state.Model = value; }
   }

   public override ModelMetadata ModelMetadata
   {
      get { return _state.ModelMetadata; }
      set { _state.ModelMetadata = value; }     
   }

   public static ModelBindingContext CreateBindingContext(    //<--------------------3.3b
      ActionContext actionContext,
      IValueProvider valueProvider,
      ModelMetadata metadata,
      BindingInfo? bindingInfo,
      string modelName)
   {
      var binderModelName = bindingInfo?.BinderModelName ?? metadata.BinderModelName;
      var bindingSource = bindingInfo?.BindingSource ?? metadata.BindingSource;
      var propertyFilterProvider = bindingInfo?.PropertyFilterProvider ?? metadata.PropertyFilterProvider;

      var bindingContext = new DefaultModelBindingContext()   // <-------------------3.3c
      {
         ActionContext = actionContext,
         BinderModelName = binderModelName,
         BindingSource = bindingSource,
         PropertyFilter = propertyFilterProvider?.PropertyFilter,
         ValidationState = new ValidationStateDictionary(),
 
         // because this is the top-level context, FieldName and ModelName should be the same.
         FieldName = binderModelName ?? modelName,
         ModelName = binderModelName ?? modelName,
         OriginalModelName = binderModelName ?? modelName,
 
         IsTopLevelObject = true,
         ModelMetadata = metadata,
         ModelState = actionContext.ModelState,   // <-----------------------
 
         OriginalValueProvider = valueProvider,
         ValueProvider = FilterValueProvider(valueProvider, bindingSource),
      };
 
      // mvcOptions may be null when this method is called in test scenarios.
      var mvcOptions = actionContext.HttpContext.RequestServices?.GetService<IOptions<MvcOptions>>();
      if (mvcOptions != null)
         bindingContext.MaxModelBindingRecursionDepth = mvcOptions.Value.MaxModelBindingRecursionDepth;
 
      return bindingContext;
   }

   // ...

   private struct State
   {
      public string FieldName;
      public object? Model;
      public ModelMetadata ModelMetadata;
      public string ModelName;
 
      public IValueProvider ValueProvider;
      public Func<ModelMetadata, bool>? PropertyFilter;
 
      public string? BinderModelName;
      public BindingSource? BindingSource;
      public bool IsTopLevelObject;
 
      public ModelBindingResult Result;
   }
}
//-------------------------------------Ʌ

//-------------------V
public abstract class ModelBinderProviderContext
{
   public abstract IModelBinder CreateBinder(ModelMetadata metadata);

   public virtual IModelBinder CreateBinder(ModelMetadata metadata, BindingInfo bindingInfo) => throw new NotSupportedException();

   public abstract BindingInfo BindingInfo { get; }

   public abstract ModelMetadata Metadata { get; }

   public abstract IModelMetadataProvider MetadataProvider { get; }

   public virtual IServiceProvider Services { get; } = default!;
}
//-------------------Ʌ

public interface IModelBinder
{
   Task BindModelAsync(ModelBindingContext bindingContext);
}

public interface IModelBinderProvider
{
   IModelBinder? GetBinder(ModelBinderProviderContext context);
}


//----------------------------------------V
public class SimpleTypeModelBinderProvider : IModelBinderProvider
{
   public IModelBinder? GetBinder(ModelBinderProviderContext context)
   {
      if (!context.Metadata.IsComplexType)
      {
         var loggerFactory = context.Services.GetRequiredService<ILoggerFactory>();
         return new SimpleTypeModelBinder(context.Metadata.ModelType, loggerFactory);
      }

      return null;
   }
}

public class SimpleTypeModelBinder : IModelBinder
{
   private readonly TypeConverter _typeConverter;
   private readonly ILogger _logger;

   public SimpleTypeModelBinder(Type type, ILoggerFactory loggerFactory)
   {
      _typeConverter = TypeDescriptor.GetConverter(type);
      _logger = loggerFactory.CreateLogger<SimpleTypeModelBinder>();
   }

   public Task BindModelAsync(ModelBindingContext bindingContext)
   {
      ValueProviderResult valueProviderResult = bindingContext.ValueProvider.GetValue(bindingContext.ModelName);
      if (valueProviderResult == ValueProviderResult.None) {
         return Task.CompletedTask;
      }

      bindingContext.ModelState.SetModelValue(bindingContext.ModelName, valueProviderResult);

      try
      {
         string value = valueProviderResult.FirstValue;
         
         object? model;
         if (bindingContext.ModelType == typeof(string))
         {
            // already have a string. No further conversion required but handle ConvertEmptyStringToNull.
            if (bindingContext.ModelMetadata.ConvertEmptyStringToNull && string.IsNullOrWhiteSpace(value))
            {
               model = null;
            }
            else
            {
               model = value;
            }
         }
         else if (string.IsNullOrWhiteSpace(value))
         {
            // other than the StringConverter, converters Trim() the value then throw if the result is empty.
            model = null;
         }
         else
         {
            model = _typeConverter.ConvertFrom(context: null, culture: valueProviderResult.Culture, value: value);   // <---------------------
         }
         CheckModel(bindingContext, valueProviderResult, model);

         return Task.CompletedTask;
      }
      catch (Exception exception)
      {
         var isFormatException = exception is FormatException;
         if (!isFormatException && exception.InnerException != null)
         {
            // TypeConverter throws System.Exception wrapping the FormatException, so we capture the inner exception.
            exception = ExceptionDispatchInfo.Capture(exception.InnerException).SourceException;
         }

         bindingContext.ModelState.TryAddModelError(bindingContext.ModelName, exception, bindingContext.ModelMetadata);

         // were able to find a converter for the type but conversion failed.
         return Task.CompletedTask;
      }
   }

   protected virtual void CheckModel(ModelBindingContext bindingContext, ValueProviderResult valueProviderResult, object? model)
   {
      if (model == null && !bindingContext.ModelMetadata.IsReferenceOrNullableType)
      {
         bindingContext.ModelState.TryAddModelError(
            bindingContext.ModelName,
            bindingContext.ModelMetadata.ModelBindingMessageProvider.ValueMustNotBeNullAccessor(valueProviderResult.ToString())
         );
      }
      else
      {
         bindingContext.Result = ModelBindingResult.Success(model);   // <------------------------------
      }
   }
}
//----------------------------------------Ʌ

//-------------------------------------------V
public class ComplexObjectModelBinderProvider : IModelBinderProvider
{
   public IModelBinder? GetBinder(ModelBinderProviderContext context)
   {
      var metadata = context.Metadata;
      if (metadata.IsComplexType && !metadata.IsCollectionType)
      {
         IReadOnlyList<IModelBinder> parameterBinders = GetParameterBinders(context);

         var propertyBinders = new Dictionary<ModelMetadata, IModelBinder>();
         for (var i = 0; i < context.Metadata.Properties.Count; i++)
         {
            var property = context.Metadata.Properties[i];
            propertyBinders.Add(property, context.CreateBinder(property));
         }

         return new ComplexObjectModelBinder(propertyBinders, parameterBinders, logger);
      }

      return null;
   }

   private static IReadOnlyList<IModelBinder> GetParameterBinders(ModelBinderProviderContext context)
   {
      var boundConstructor = context.Metadata.BoundConstructor;
      if (boundConstructor is null)
      {
         return Array.Empty<IModelBinder>();
      }

      var parameterBinders = boundConstructor.BoundConstructorParameters!.Count == 0 
         ? Array.Empty<IModelBinder>() : new IModelBinder[boundConstructor.BoundConstructorParameters.Count];
      
      for (var i = 0; i < parameterBinders.Length; i++)
      {
         parameterBinders[i] = context.CreateBinder(boundConstructor.BoundConstructorParameters[i]);
      }

      return parameterBinders;
   }
}
//-------------------------------------------Ʌ

//--------------------------------------------------V
public sealed partial class ComplexObjectModelBinder : IModelBinder
{
   internal const int NoDataAvailable = 0;
   internal const int GreedyPropertiesMayHaveData = 1;
   internal const int ValueProviderDataAvailable = 2;

   private readonly IDictionary<ModelMetadata, IModelBinder> _propertyBinders;
   private readonly IReadOnlyList<IModelBinder> _parameterBinders;  // <----------------------
   private readonly ILogger _logger;
   private Func<object>? _modelCreator;

   internal ComplexObjectModelBinder(
      IDictionary<ModelMetadata, IModelBinder> propertyBinders, 
      IReadOnlyList<IModelBinder> parameterBinders, 
      ILogger<ComplexObjectModelBinder> logger)
   {
      _propertyBinders = propertyBinders;
      _parameterBinders = parameterBinders;
      _logger = logger;
   }

   public Task BindModelAsync(ModelBindingContext bindingContext)
   {
      var parameterData = CanCreateModel(bindingContext);
      if (parameterData == NoDataAvailable)
      {
         return Task.CompletedTask;
      }

      return BindModelCoreAsync(bindingContext, parameterData);
   }

   private async Task BindModelCoreAsync(ModelBindingContext bindingContext, int propertyData)
   {
      // create model first (if necessary) to avoid reporting errors about properties when activation fails.
      var attemptedBinding = false;
      var bindingSucceeded = false;

      ModelMetadata modelMetadata = bindingContext.ModelMetadata;
      ModelMetadata boundConstructor = modelMetadata.BoundConstructor;

      if (boundConstructor != null)   // // only record types are allowed to have a BoundConstructor
      {
         var values = new object[boundConstructor.BoundConstructorParameters!.Count];
         var (attemptedParameterBinding, parameterBindingSucceeded) = await BindParametersAsync(
            bindingContext,
            propertyData,
            boundConstructor.BoundConstructorParameters,
            values
         );
 
         attemptedBinding |= attemptedParameterBinding;
         bindingSucceeded |= parameterBindingSucceeded;
 
         if (!CreateModel(bindingContext, boundConstructor, values))
         {
            return;
         }
      }
      else if (bindingContext.Model == null)
      {
         CreateModel(bindingContext);
      }

      var (attemptedPropertyBinding, propertyBindingSucceeded) = await BindPropertiesAsync(bindingContext, propertyData, modelMetadata.BoundProperties);

      attemptedBinding |= attemptedPropertyBinding;
      bindingSucceeded |= propertyBindingSucceeded;

      if (!attemptedBinding && bindingContext.IsTopLevelObject && modelMetadata.IsBindingRequired)
      {
         var messageProvider = modelMetadata.ModelBindingMessageProvider;
         var message = messageProvider.MissingBindRequiredValueAccessor(bindingContext.FieldName);
         bindingContext.ModelState.TryAddModelError(bindingContext.ModelName, message);
      }

      if (!bindingContext.IsTopLevelObject && !bindingSucceeded && propertyData == GreedyPropertiesMayHaveData)
      {
         bindingContext.Result = ModelBindingResult.Failed();
         return;
      }
 
      bindingContext.Result = ModelBindingResult.Success(bindingContext.Model);
   }

   internal static bool CreateModel(ModelBindingContext bindingContext, ModelMetadata boundConstructor, object[] values)
   {
      try
      {
         bindingContext.Model = boundConstructor.BoundConstructorInvoker!(values);
         return true;
      }
      catch (Exception ex)
      {
         AddModelError(ex, bindingContext.ModelName, bindingContext);
         bindingContext.Result = ModelBindingResult.Failed();
         return false;
      }
   }

   internal void CreateModel(ModelBindingContext bindingContext)
   {
      if (_modelCreator == null)
      {
         var modelType = bindingContext.ModelType;
         if (modelType.IsAbstract || modelType.GetConstructor(Type.EmptyTypes) == null)
         {
            var metadata = bindingContext.ModelMetadata;
            switch (metadata.MetadataKind)
            {
               case ModelMetadataKind.Parameter:
                  throw new InvalidOperationException(...);
               case ModelMetadataKind.Property:
                  throw new InvalidOperationException(...);
               case ModelMetadataKind.Type:
                  throw new InvalidOperationException(...);
            }
         }
 
         _modelCreator = Expression.Lambda<Func<object>>(Expression.New(bindingContext.ModelType)).Compile();   // <--------
      }

      bindingContext.Model = _modelCreator();
   }

   private async ValueTask<(bool attemptedBinding, bool bindingSucceeded)> BindParametersAsync(
      ModelBindingContext bindingContext,
      int propertyData,
      IReadOnlyList<ModelMetadata> parameters,
      object?[] parameterValues)
   {
      var attemptedBinding = false;
      var bindingSucceeded = false;
 
      if (parameters.Count == 0)
      {
         return (attemptedBinding, bindingSucceeded);
      }

      var postponePlaceholderBinding = false;
      for (var i = 0; i < parameters.Count; i++)
      {
         var parameter = parameters[i];
 
         var fieldName = parameter.BinderModelName ?? parameter.ParameterName!;
         var modelName = ModelNames.CreatePropertyModelName(bindingContext.ModelName, fieldName);
 
         if (!CanBindItem(bindingContext, parameter))
            continue;
 
         IModelBinder parameterBinder = _parameterBinders[i];

         // ...

         ModelBindingResult result = await BindParameterAsync(bindingContext, parameter, parameterBinder, fieldName, modelName);

         if (result.IsModelSet)
         {
            attemptedBinding = true;
            bindingSucceeded = true;
 
            parameterValues[i] = result.Model;
         }
         else if (parameter.IsBindingRequired)
         {
            attemptedBinding = true;
         }
      }
   }

   private async ValueTask<(bool attemptedBinding, bool bindingSucceeded)> BindPropertiesAsync(
      ModelBindingContext bindingContext,
      int propertyData,
      IReadOnlyList<ModelMetadata> boundProperties)
   {
      // ...
   }

   internal static bool CanBindItem(ModelBindingContext bindingContext, ModelMetadata propertyMetadata)
   {
      var metadataProviderFilter = bindingContext.ModelMetadata.PropertyFilterProvider?.PropertyFilter;
      if (metadataProviderFilter?.Invoke(propertyMetadata) == false)
         return false;
 
      if (bindingContext.PropertyFilter?.Invoke(propertyMetadata) == false)
         return false;
 
      if (!propertyMetadata.IsBindingAllowed)
          return false;
 
      if (propertyMetadata.MetadataKind == ModelMetadataKind.Property && propertyMetadata.IsReadOnly)
      {
         // determine if we can update a readonly property (such as a collection).
         return CanUpdateReadOnlyProperty(propertyMetadata.ModelType);
      }
 
      return true;
   }

   private static async ValueTask<ModelBindingResult> BindParameterAsync(
      ModelBindingContext bindingContext,
      ModelMetadata parameter,
      IModelBinder parameterBinder,
      string fieldName,
      string modelName)
   { 
      ModelBindingResult result;
      using (bindingContext.EnterNestedScope(modelMetadata: parameter, fieldName: fieldName, modelName: modelName, model: null))
      {
         await parameterBinder.BindModelAsync(bindingContext);
         result = bindingContext.Result;
      }
 
      if (!result.IsModelSet && parameter.IsBindingRequired)
      {
         var message = parameter.ModelBindingMessageProvider.MissingBindRequiredValueAccessor(fieldName);
         bindingContext.ModelState.TryAddModelError(modelName, message);
      }
 
      return result;
   }

   internal int CanCreateModel(ModelBindingContext bindingContext)
   {
      var isTopLevelObject = bindingContext.IsTopLevelObject;
      var bindingSource = bindingContext.BindingSource;
      if (!isTopLevelObject && bindingSource != null && bindingSource.IsGreedy)
      {
         return NoDataAvailable;
      }
 
      // Create the object if:
      // 1. It is a top level model.
      if (isTopLevelObject)
      {
         return ValueProviderDataAvailable;
      }
 
      // 2. Any of the model properties can be bound.
      return CanBindAnyModelItem(bindingContext);
   }

   private int CanBindAnyModelItem(ModelBindingContext bindingContext)
   {
      // ...
   }

   private static void AddModelError(Exception exception, string modelName, ModelBindingContext bindingContext)
   {
      var targetInvocationException = exception as TargetInvocationException;
      if (targetInvocationException?.InnerException != null)
      {
         exception = targetInvocationException.InnerException;
      }
 
      // do not add an error message if a binding error has already occurred for this property.
      var modelState = bindingContext.ModelState;
      var validationState = modelState.GetFieldValidationState(modelName);
      if (validationState == ModelValidationState.Unvalidated)
      {
         modelState.AddModelError(modelName, exception, bindingContext.ModelMetadata);
      }
   }

   // ...
}
//--------------------------------------------------Ʌ

//----------------------------------V
public class BodyModelBinderProvider : IModelBinderProvider
{
   private readonly IList<IInputFormatter> _formatters;
   private readonly IHttpRequestStreamReaderFactory _readerFactory;
   private readonly ILoggerFactory _loggerFactory;
   private readonly MvcOptions? _options;

   public BodyModelBinderProvider(IList<IInputFormatter> formatters, IHttpRequestStreamReaderFactory readerFactory) : this(...) { }
   
   public BodyModelBinderProvider(IList<IInputFormatter> formatters, IHttpRequestStreamReaderFactory readerFactory, ILoggerFactory loggerFactory, MvcOptions? options)
   {
      _formatters = formatters;
      _readerFactory = readerFactory;
      _loggerFactory = loggerFactory;
      _options = options;
   }

   public IModelBinder? GetBinder(ModelBinderProviderContext context)
   {
      if (context.BindingInfo.BindingSource != null && context.BindingInfo.BindingSource.CanAcceptDataFrom(BindingSource.Body))
      {
         var treatEmptyInputAsDefaultValue = CalculateAllowEmptyBody(context.BindingInfo.EmptyBodyBehavior, _options);

         return new BodyModelBinder(_formatters, _readerFactory, _loggerFactory, _options)
         {
            AllowEmptyBody = treatEmptyInputAsDefaultValue,
         };
      }

      return null;
   }

   internal static bool CalculateAllowEmptyBody(EmptyBodyBehavior emptyBodyBehavior, MvcOptions? options)
   {
      if (emptyBodyBehavior == EmptyBodyBehavior.Default)
      {
         return options?.AllowEmptyInputInBodyModelBinding ?? false;
      }
 
      return emptyBodyBehavior == EmptyBodyBehavior.Allow;
   }
}
//----------------------------------Ʌ

//----------------------------------V
public partial class BodyModelBinder : IModelBinder
{
   private readonly IList<IInputFormatter> _formatters;
   private readonly Func<Stream, Encoding, TextReader> _readerFactory;
   private readonly ILogger _logger;
   private readonly MvcOptions? _options;

   public BodyModelBinder(IList<IInputFormatter> formatters, IHttpRequestStreamReaderFactory readerFactory) : this(formatters, readerFactory, loggerFactory: null) { }

   public BodyModelBinder(IList<IInputFormatter> formatters, IHttpRequestStreamReaderFactory readerFactory, ILoggerFactory? loggerFactory, MvcOptions? options)
   {
      _formatters = formatters;
      _readerFactory = readerFactory.CreateReader;
      _logger = loggerFactory?.CreateLogger<BodyModelBinder>() ?? NullLogger<BodyModelBinder>.Instance;
      _options = options;
   }

   internal bool AllowEmptyBody { get; set; }

   public async Task BindModelAsync(ModelBindingContext bindingContext)
   {
      string modelBindingKey;
      if (bindingContext.IsTopLevelObject)
      {
         modelBindingKey = bindingContext.BinderModelName ?? string.Empty;
      }
      else
      {
         modelBindingKey = bindingContext.ModelName;
      }

      var httpContext = bindingContext.HttpContext;

      var formatterContext = new InputFormatterContext(httpContext, modelBindingKey, bindingContext.ModelState, bindingContext.ModelMetadata, _readerFactory, AllowEmptyBody);
      var formatter = (IInputFormatter?)null;
      for (var i = 0; i < _formatters.Count; i++)
      {
         if (_formatters[i].CanRead(formatterContext))
         {
            formatter = _formatters[i];
            Log.InputFormatterSelected(_logger, formatter, formatterContext);
            break;
         }
         else
         {
               Log.InputFormatterRejected(_logger, _formatters[i], formatterContext);
         }
      }

      if (formatter == null)
      {
         if (AllowEmptyBody)
         {
            var hasBody = httpContext.Features.Get<IHttpRequestBodyDetectionFeature>()?.CanHaveBody;
            hasBody ??= httpContext.Request.ContentLength is not null && httpContext.Request.ContentLength == 0;
            if (hasBody == false)
            {
               bindingContext.Result = ModelBindingResult.Success(model: null);
               return;
            }
         }
  
         var message = Resources.FormatUnsupportedContentType(httpContext.Request.ContentType);
         var exception = new UnsupportedContentTypeException(message);
         bindingContext.ModelState.AddModelError(modelBindingKey, exception, bindingContext.ModelMetadata);
         return;
      }

      try
      {
         var result = await formatter.ReadAsync(formatterContext);
 
         if (result.HasError)
         {
            // formatter encountered an error. Do not use the model it returned.
            _logger.DoneAttemptingToBindModel(bindingContext);
            return;
         }
 
         if (result.IsModelSet)
         {
            var model = result.Model;
            bindingContext.Result = ModelBindingResult.Success(model);
         }
         else
         {        
            var message = bindingContext
               .ModelMetadata
               .ModelBindingMessageProvider
               .MissingRequestBodyRequiredValueAccessor();
            bindingContext.ModelState.AddModelError(modelBindingKey, message);
         }
      }
      catch (Exception exception) when (exception is InputFormatterException || ShouldHandleException(formatter))
      {
         bindingContext.ModelState.AddModelError(modelBindingKey, exception, bindingContext.ModelMetadata);
      }
   }

   private static bool ShouldHandleException(IInputFormatter formatter)
   {
      // Any explicit policy on the formatters overrides the default.
      var policy = (formatter as IInputFormatterExceptionPolicy)?.ExceptionPolicy ?? InputFormatterExceptionPolicy.MalformedInputExceptions;
 
      return policy == InputFormatterExceptionPolicy.AllExceptions;
   }
}
//----------------------------------Ʌ

//----------------------------------V
public partial class ParameterBinder
{
   private readonly IModelMetadataProvider _modelMetadataProvider;   // not being used importantly
   private readonly IModelBinderFactory _modelBinderFactory;         // not being used
   private readonly IObjectModelValidator _objectModelValidator;

   public ParameterBinder(
      IModelMetadataProvider modelMetadataProvider,
      IModelBinderFactory modelBinderFactory,
      IObjectModelValidator validator,
      IOptions<MvcOptions> mvcOptions,
      ILoggerFactory loggerFactory)
   {
      _modelMetadataProvider = modelMetadataProvider;
      _modelBinderFactory = modelBinderFactory;
      _objectModelValidator = validator;
      Logger = loggerFactory.CreateLogger(GetType());
   }

   public virtual Task<ModelBindingResult> BindModelAsync(   // <----------------4.2b
      ActionContext actionContext,
      IModelBinder modelBinder,
      IValueProvider valueProvider,
      ParameterDescriptor parameter,
      ModelMetadata metadata,
      object? value)
   {
      BindModelAsync(actionContext, modelBinder, valueProvider, parameter, metadata, value, container: null).AsTask();
   }

   public virtual async ValueTask<ModelBindingResult> BindModelAsync(
      ActionContext actionContext,
      IModelBinder modelBinder,
      IValueProvider valueProvider,
      ParameterDescriptor parameter,
      ModelMetadata metadata,
      object? value,
      object? container)
   {
      if (parameter.BindingInfo?.RequestPredicate?.Invoke(actionContext) == false)
      {
         Log.ParameterBinderRequestPredicateShortCircuit(Logger, parameter, metadata);
         return ModelBindingResult.Failed();
      }

      var modelBindingContext = DefaultModelBindingContext         // <-----------------------
                                .CreateBindingContext(actionContext, valueProvider, metadata, parameter.BindingInfo, parameter.Name);
      modelBindingContext.Model = value;

      var parameterModelName = parameter.BindingInfo?.BinderModelName ?? metadata.BinderModelName;
      if (parameterModelName != null)
      {
         // the name was set explicitly, always use that as the prefix.
         modelBindingContext.ModelName = parameterModelName;
      }
      else if (modelBindingContext.ValueProvider.ContainsPrefix(parameter.Name))
      {
         // we have a match for the parameter name, use that as that prefix.
         modelBindingContext.ModelName = parameter.Name;
      }
      else
      {
         // no match, fallback to empty string as the prefix.
         modelBindingContext.ModelName = string.Empty;
      }

      await modelBinder.BindModelAsync(modelBindingContext);   // <------------------------3.4

      ModelBindingResult modelBindingResult = modelBindingContext.Result;

      if (_objectModelValidator is ObjectModelValidator baseObjectValidator)
      {
         EnforceBindRequiredAndValidate(baseObjectValidator, actionContext, parameter, metadata, modelBindingContext, modelBindingResult, container);
      }
      else
      {
         // for legacy implementations (which directly implemented IObjectModelValidator), fall back to the
         // back-compatibility logic. In this scenario, top-level validation attributes will be ignored like they were historically.
         if (modelBindingResult.IsModelSet)
         {
            _objectModelValidator.Validate(actionContext, modelBindingContext.ValidationState, modelBindingContext.ModelName, modelBindingResult.Model);
         }
      }

      return modelBindingResult;
   }

   private void EnforceBindRequiredAndValidate(
      ObjectModelValidator baseObjectValidator,
      ActionContext actionContext,
      ParameterDescriptor parameter,
      ModelMetadata metadata,
      ModelBindingContext modelBindingContext,
      ModelBindingResult modelBindingResult,
      object? container)
   {
      RecalculateModelMetadata(parameter, modelBindingResult, ref metadata);
 
      if (!modelBindingResult.IsModelSet && metadata.IsBindingRequired)
      {
         // enforce BindingBehavior.Required (e.g., [BindRequired])
         var modelName = modelBindingContext.FieldName;
         var message = metadata.ModelBindingMessageProvider.MissingBindRequiredValueAccessor(modelName);
         actionContext.ModelState.TryAddModelError(modelName, message);
      }
      else if (modelBindingResult.IsModelSet)
      {
         // enforce any other validation rules
         baseObjectValidator.Validate(actionContext, modelBindingContext.ValidationState, modelBindingContext.ModelName, modelBindingResult.Model, metadata, container);
      }
      else if (metadata.IsRequired)
      {
         var modelName = modelBindingContext.ModelName;
 
         if (string.IsNullOrEmpty(modelBindingContext.ModelName) && parameter.BindingInfo?.BinderModelName == null)
         {
            // if we get here then this is a fallback case. The model name wasn't explicitly set and we ended up with an empty prefix.
            modelName = modelBindingContext.FieldName;
         }
 
         // run validation, we expect this to validate [Required].
         baseObjectValidator.Validate(actionContext, modelBindingContext.ValidationState, modelName, modelBindingResult.Model, metadata, container);
      }
   }

   private void RecalculateModelMetadata(ParameterDescriptor parameter, ModelBindingResult modelBindingResult, ref ModelMetadata metadata)
   {
      if (!modelBindingResult.IsModelSet || modelBindingResult.Model == null || _modelMetadataProvider is not ModelMetadataProvider modelMetadataProvider)
      {
         return;
      }
 
      var modelType = modelBindingResult.Model.GetType();
      if (parameter is IParameterInfoParameterDescriptor parameterInfoParameter)
      {
         var parameterInfo = parameterInfoParameter.ParameterInfo;
         if (modelType != parameterInfo.ParameterType)
         {
            metadata = modelMetadataProvider.GetMetadataForParameter(parameterInfo, modelType);
         }
      }
      else if (parameter is IPropertyInfoParameterDescriptor propertyInfoParameter)
      {
         var propertyInfo = propertyInfoParameter.PropertyInfo;
         if (modelType != propertyInfo.PropertyType)
         {
            metadata = modelMetadataProvider.GetMetadataForProperty(propertyInfo, modelType);
         }
      }
   }
}
//----------------------------------Ʌ
```

```C#
//------------------------V
public class BindAttribute : Attribute, IModelNameProvider, IPropertyFilterProvider
{
   private static readonly Func<ModelMetadata, bool> _default = (m) => true;
   
   private Func<ModelMetadata, bool>? _propertyFilter;

   public BindAttribute(params string[] include)
   {
      var items = new List<string>(include.Length);
      foreach (var item in include)
      {
         items.AddRange(SplitString(item));
      }

      Include = items.ToArray();
   }

   public string[] Include { get; }
   public string? Prefix { get; set; }   // <---------------------
   string? IModelNameProvider.Name => Prefix;

   public Func<ModelMetadata, bool> PropertyFilter
   {
      get {
         if (Include != null && Include.Length > 0)
         {
            _propertyFilter ??= PropertyFilter;
            return _propertyFilter;
         }
         else
         {
            return _default;
         }
 
         bool PropertyFilter(ModelMetadata modelMetadata)
         {
            if (modelMetadata.MetadataKind == ModelMetadataKind.Parameter)
            {
               return Include.Contains(modelMetadata.ParameterName, StringComparer.Ordinal);
            }
 
            return Include.Contains(modelMetadata.PropertyName, StringComparer.Ordinal);
         }
      }
   }

   private static IEnumerable<string> SplitString(string original) 
   {
      return original?.Split(',', StringSplitOptions.TrimEntries | StringSplitOptions.RemoveEmptyEntries) ?? Array.Empty<string>();
   } 
}
//------------------------Ʌ

//-----------------------------------V
public class BindingBehaviorAttribute : Attribute
{
   public BindingBehaviorAttribute(BindingBehavior behavior)
   {
      Behavior = behavior;
   }
}

public BindingBehavior Behavior { get; }
//-----------------------------------Ʌ

//
public enum BindingBehavior
{
   Optional = 0,
   Never,
   Required
}
//

//------------------------------------V
public sealed class BindNeverAttribute : BindingBehaviorAttribute
{
   public BindNeverAttribute() : base(BindingBehavior.Never) { }
}
//------------------------------------Ʌ

public class BindPropertyAttribute : Attribute, IBinderTypeProviderMetadata, IBindingSourceMetadata, IModelNameProvider, IRequestPredicateProvider
{
   public BindPropertyAttribute();

   public Type BinderType { get; set; }
   public virtual BindingSource BindingSource { get; protected set; }
   public string Name { get; set; }
   public bool SupportsGet { get; set; }
}




```