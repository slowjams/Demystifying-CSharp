## Source Code

```C#
//-------------------------------V
public class ModelStateDictionary : IReadOnlyDictionary<string, ModelStateEntry?>  // represents the state of an attempt to bind values from an HTTP Request 
{                                                                                  // to an action method, which includes validation information
   public static readonly int DefaultMaxAllowedErrors = 200;
   
   // internal for testing
   internal const int DefaultMaxRecursionDepth = 32;
 
   private const char DelimiterDot = '.';
   private const char DelimiterOpen = '[';
 
   private readonly ModelStateNode _root;
   private int _maxAllowedErrors;

   public ModelStateDictionary() : this(DefaultMaxAllowedErrors) { }

   // ...

   private ModelStateDictionary(int maxAllowedErrors, int maxValidationDepth, int maxStateDepth)
   {
      MaxAllowedErrors = maxAllowedErrors;
      MaxValidationDepth = maxValidationDepth;
      MaxStateDepth = maxStateDepth;
      var emptySegment = new StringSegment(buffer: string.Empty);
       _root = new ModelStateNode(subKey: emptySegment)
      {
         Key = string.Empty
      };
   }

   public ModelStateEntry Root => _root;

   public int MaxAllowedErrors
   {
      get {
         return _maxAllowedErrors;
      }
      set {
         _maxAllowedErrors = value;
      }
   }

   public bool HasReachedMaxErrors => ErrorCount >= MaxAllowedErrors;
   public int ErrorCount { get; private set; }
   public int Count { get; private set; }
   public KeyEnumerable Keys => new KeyEnumerable(this);

   public bool IsValid
   {
      get {
         var state = ValidationState;
         return state == ModelValidationState.Valid || state == ModelValidationState.Skipped;
      }
   }

   public ModelValidationState ValidationState => GetValidity(_root, currentDepth: 0) ?? ModelValidationState.Valid;

   public ModelStateEntry? this[string key]
   {
      get {          
         TryGetValue(key, out var entry);
         return entry;
      }
   }

   public bool TryGetValue(string key, [NotNullWhen(true)] out ModelStateEntry? value)
   {
      var result = GetNode(key);
      if (result?.IsContainerNode == false)
      {
         value = result;
         return true;
      }
 
      value = null;
      return false;
   }

   private ModelStateNode? GetNode(string key)
   {
      var current = _root;
      if (key.Length > 0)
      {
         var match = default(MatchResult);
         do
         {
            var subKey = FindNext(key, ref match);
            current = current.GetNode(subKey);
 
            // Path not found, exit early
            if (current == null)
               break;
 
         } while (match.Type != Delimiter.None);
      }
 
      return current;
   }

   public void AddModelError(string key, Exception exception, ModelMetadata metadata)
   {
      TryAddModelError(key, exception, metadata);
   }

   public bool TryAddModelException(string key, Exception exception)
   {
      if ((exception is InputFormatterException || exception is ValueProviderException) && !string.IsNullOrEmpty(exception.Message))
      {
         // InputFormatterException, ValueProviderException is a signal that the message is safe to expose to clients
         return TryAddModelError(key, exception.Message);
      }
 
      if (ErrorCount >= MaxAllowedErrors - 1)
      {
         EnsureMaxErrorsReachedRecorded();
         return false;
      }
 
      AddModelErrorCore(key, exception);
      return true;
   }

   public bool TryAddModelError(string key, Exception exception, ModelMetadata metadata) { ... }

   public void AddModelError(string key, string errorMessage)
   {
      TryAddModelError(key, errorMessage);
   }

   public bool TryAddModelError(string key, string errorMessage)
   {     
      if (ErrorCount >= MaxAllowedErrors - 1)
      {
         EnsureMaxErrorsReachedRecorded();
         return false;
      }
 
      var modelState = GetOrAddNode(key);
      Count += !modelState.IsContainerNode ? 0 : 1;
      modelState.ValidationState = ModelValidationState.Invalid;
      modelState.MarkNonContainerNode();
      modelState.Errors.Add(errorMessage);
 
      ErrorCount++;
      return true;
   }

   // continue ...hghghdgfhdgfhdgfg
}
//-------------------------------Ʌ
```