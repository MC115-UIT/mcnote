## Reporting pdf from html razor views

```c#
/Install-Package Razor.Template.Core

app.MapGet('invoice-report',async(InvoiceFactory invoiceFactory)=>{

	Invoice invoice = invoiceFactory.Create();
	var html = await RazorTemplateEngine.RenderAysnc(
	'Views/InvoiceReport.cshtml',
	invoice
	);

	var renderer = new ChormePdfRenderer();
	using var pdfDocument = renderer.RendererHtmlAsPdf(html)

	return Results.File(
	pdfDocument.BinaryData,
	'application/pdf',
	$'invoice-{invoice.number}.pdf'
	);
	
});
```


```c#
try
{
	cts.CancelAfter(Timespan.FromMinutes(15))
}
catch (Exception ex)
{
	Log.Information($"Exception caught : {ex}")
	if (ex is OperationCanceledException)
	{
		Log.Information("One or more tasks were canceled")
	}

}
```

##Health check
Link [more detail](https://medium.com/@jeslurrahman/implementing-health-checks-in-net-8-c3ba10af83c3)
-- Memory health check
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Options;


namespace FeedbackService.Api.HealthCheck
{
	public class MemoryHealthCheck : IHealthCheck
	{
		private readonly IOptions<MemoryCheckOptions> _options;
		public MemoryHealthCheck (IOptions<MemoryCheckOptions> options)
		{
			_options = options;
		}
		public string Name => 'memory_check';
		public Task<HealthCheckResult> CheckHealthAsync (
			HealthCheckContext context,
			CancellationToken cancellationToken = default(CancellationToken)
		)
		{
			var options = _options.Get(context.Registration.Name)
				// Include GC information in the reported diagnostic.
			var allocated = GC.GetTotalMemory(forceFullCollection : false);
			var data = new Dictionary<string, object(0)>
			{
				{"Allocated": allocated},
				{"Gen0Collections": GC.CollectionCount(0)},
				{"Gen1Collections": GC.CollectionCount(1)}.
				{"Gen2Collections": GC.CollectionCount(2)}
			};
			var status = (allocated < options.Threshold) ? HealthStatus.Healthy : HealthStatus.UnHealthy;

			return Task.FromResult(new HealthCheck (
				status,
				description : "Reports degraded status if allocated bytes" +
				$">= {options.Threshold} bytes",
				exception : null,
				data : data
			));
		}
	}
	public class MemoryCheckOptions()
	{
		public string MemoryStatus {get; set;}
		// pulic int Threshold {get; set;}
		// Failure in bytes
		public long Threshold {get; set;} = 1024L*1024L*1024L;
	}
}
```

Configure `RemoteHealthCheck.cs` and `MemoryHealthCheck.cs` inside the `HealthCheck.cs`

```c# title="HealthCheck.cs"
public static void ConfigureHealthCheck(this IServiceCollection services, IConfiguration configuration)
{
	services.AddHealthChecks()
		.AddSqlServer(configuration["Connectionstring:Feedback"], healthQuery : "Select 1", name : "SQL Server", failureStatus :  HealthStatus.UnHealthy, tags : new[] {"Feedback,Database"})
			.AddCheck<RemoteHealthCheck>("Remote endpoints Health Check",failureStatus : HealthStatus.UnHealthy)
			.AddCheck<MemoryHealthCheck>($"Feedback Service memory check", failureStatus : HealtheStatus.UnHealthy ,tags : new[] {"Feedback service"})
			.AddUrlGroup(new Uri("https://localhost:44333/api/v1/heartbeats/ping"), name : "Base URL" , failureStatus : HealthStatus.UnHealthy);

	services.AddHealthCheckUI(opt => {
		opt.SetEvaluationTimeSeconds(10) // time in seconds between check
		opt.MaximumHistoryEntriesPerEnpoint(60) // maximum history of checks
		opt.SetApiMaxActiveRequests(1) // api request concurrency
		opt.AddHealthCheckEndpoint("feedback api","/api/health"); // map health check api
	})
		.AddInMemoryStorage();
}

//configure the headlthCheck in main (program.cs)

// Configuring the HealthCheck
builder.Services.ConfigureHealthCheck(builder.Configuration)

//HealthCheck MiddleWare

app.MapHealthChecks("api/health", new HealthCheckOptions()
{
	Predicate => true,
	ResponeWriter = UIResponseWriter.WriteHealthCheckUIRespone
});
app.UseHealthCheckUI(delegaate (Options options)
{
	options.UIPath = "/healthcheck-ui";
	options.AddCustomStyleSheet("./HealthCheck/Custom.css");
})
```

## Expression builder

```c# title="ExpressionBuilder.cs"
public class ExpressionBuilder<T>
{
	// create expression entity => entity.propertyName
	public Expression CreatePropertyExpression(
		Expression entityParameter,
		string propertyName, object? value
	)
	{
		var property= typeof(T).GetProperty(propertyName);
		var propertyAccess = Expression.MakeMemberAccess(entityParameter, property);
		var valueExpression = Expression.Constant(value, property.PropertyType);
		var equalsExpression = Expression.Equal(propertyAccess, valueExpression);

		return equalsExpression;
	}
	// create expression entity => entity.propertyName1 == value1 && entity.propertyName2 == value2 && ...

	public Func<T,bool> ComplexExpression(List<Tupple<string,object> conditions>)
	{
		Expression expression = null;
		var parameter = Expression.Parameter(typeOf(T), "entity");
		
		foreach(var condition in conditions)
		{
			var currentExpression = CreatePropertyExpression(parameter, condition.Item1, condition.Item2);
			if(expression == null)
			{
				expression = currentExpression;
				continue;
			}
			expression = Expression.AndAlso(expression, currentExpression);
		}
		var lambda = Expression.Lambda<Func<T,bool>>(expression, parameter);
		return lambda.Compile();
	}
}

```

Usage 
```c#
	var builder = new ExpressionBuilder<Foo>();

	// this can be create it at runtime
	var conditions = new List<Tupple<string,object>>{
		new Tupple<string,object>("Condition1", "Value1"),
		new Tupple<string,object>("Condition2", "Value2")
		//etc
	}
	var expression = builder.ComplexExpression(conditions);
	var filterd = list.Where(expression).Tolist()
```

## Comparing two object in effcient way

```c#
	public static bool DeepEquals(this object obj, object another)
	{
		if(ReferenceEqual(obj, another)) return true;
		if(obj == null || another == null) return false;
		if(obj.GetType() != another.GetType()) return false;

		var result = true;
		foreach(var property in obj.GetType().GetProperties())
		{
			var objectValue = property.GetValue(obj);
			var anotherValue = property.GetValue(another);
			if(!objectValue.Equals(anotherValue)) result = false
			return result;
		}
	}
	// if the property is also object 
	if(!objectValue.DeepEquals(anotherValue)) result = false

	// with the Json format
	public static bool JsonEquals(this object obj, object another)
	{
		if(ReferenceEqual(obj, another)) return true;
		if(obj == null || another == null) return false;
		if(obj.GetType() != another.GetType()) return false;

		var objJson = JsonConvert.SerializationObject(obj);  
		var anotherJson = JsonConvert.SerializationObject(another);
		
		return objJson == anotherJson;
	}

	// Comparing Two list object
	public static bool DeepEqual<T>( this IEnumerable<T> obj, IEnumerable<T> another)
	{
		if(ReferenceEquals(obb, another)) return true;
		if(obj == null || another == null) return false;

		bool result = true;
		using(var enumerator1 = obj.GetEnumerator())
		using(var enumerator2 = another.GetEnumerator())
		{
			while(true)
			{
				bool hasNext1 = enumerator1.MoveNext();
				bool hasNext2 = enumerator2.MoveNext();

				if(hasNext1 != hasNext2 || !enumerator1.Current.DeepEquals(enumerator2.Current))
				{
					result = false;
					break;
				}
				if(!hasNext1) break;
			}
		}
		return result;
	}
```
## Hepler funcitons with Func<>

``` C#
	public static class SmartHelper
	{
		// Cache for memoization
		private static readonly ConcurrentDictionary<string, object> _cache = new();

		// 1. Memoization helper for expensive computations
		public static Func<TResult> Memoize<TResult> (Func<TResult> func)
		{
			return () =>
			{
				var key = $"{func.Method.Name}";
				return (TResult)_cache.GetOrAdd(key, _ => func());
			};
		}
		
		public void Demo()
		{
			Func<decimal> expensiveCalculation = () =>
			{
				Thread.Sleep(2000);
				return decimal.Parse("123.213");
			}

			var memoizedCalc = SmartHelper.Memoize(expensiveCalculation);

			// firt call take times
			var result1 = memoizedCalc();
			var result2 = memoizedCalc();
			// second call is instant
		}
		

		// 2.Memoization for functions with parameters
		public static Func<T, TResult> Memoize<T, TResult> (Func<T,TResult> func) where T : notnull
		{
			return arg =>
			{
				var key = $"{func.Method.Name}";
				return (TResult)_cache.GetOrAdd(key, _ => func(arg));
			};
		}

		public void Demo()
		{
			 Func<string, int> stringProcessor = str =>
			{
				Thread.Sleep(1000); // Simulate processing
				return str.GetHashCode();
			};


			var memoizedProcessor = SmartHelpers.Memoize(stringProcessor);
        
			// First call for each unique input takes time
			var hash1 = memoizedProcessor("test"); // Takes 1 second
			var hash2 = memoizedProcessor("test"); // Instant (cached)
			var hash3 = memoizedProcessor("different"); // Takes 1 second
		}
		

		// 3. Retry machanism with exponential backoff
		public static Func<T, TResult> WithRetry<T, TResult>(
			Func<T, TResult> func,
			int maxAttempts = 3,
			int initialDelays = 100
		)
		{
			return arg =>
			{
				for(int attempt = 1, attempt <= maxAttempts, attempt ++)
				{
					try
					{
						return func(arg);
					}
					catch(Exception) when ( attempt < maxAttempts)
					{
						Thread.Sleep(initialDelays * (int)Math.Pow(2, attempt-1));
					}
				}
				return func(arg);
			};
		}

		public void Demo()
		{
			Func<int, string> unreliableService = id =>
			{
				if (Random.Shared.Next(0, 2) == 0)
					throw new Exception("Random failure");
				return $"Data for ID: {id}";
			};


			var reliableService = SmartHelper.WithRetry(unreliableService);
			try
			{
				var data = reliableService(123);
				Console.WriteLine(data);
			}
			catch(Exception ex)
			{
				Console.WriteLine("All retry attempts failed");
			}
		}
		
		// 4. Performance measurement decorator
		public static Func<T, TResult> MeasurePerformance<T, TResult>(
			Func<T, TResutl> func,
			Action<TimeSpan> performanceCallback
		)
		{
			return arg => 
			{
				var sw = StopWatch.StartNew();
				var result = func(arg);
				sw.Stop();
				performanceCallback(sw.Elapsed);
				return result;
			};
		}

		public void Demo()
		{
			Func<int, double> complexCalculation = x =>
			{
				Thread.Sleep(100); // Simulate work
				return Math.Pow(x, 2);
			};

			var measuredFunc = SmartHelpers.MeasurePerformance(
				complexCalculation,
				elapsed => Console.WriteLine($"Operation took: {elapsed.TotalMilliseconds}ms")
			);
			var result = measuredFunc(10); 

		}


		// 5. Expression compiler with caching
		private static readonly ConcurrentDictionary<string, Delegate> _compiledExpressions = new();

		public static Func<T, TResutl> CompileAndCache<T, TResult>(Expression<Func<T, TResult>> expression)
		{
			var key = expression.ToString();
			return (Func<T, TResult>)_compiledExpreesions.GetOrAdd(key, _ => expression.Compile());
		}

		public void Demo()
		{
			Expression<Func<int, bool>> isEvenExpr = num => num % 2 == 0;
			var compiledFunc = SmartHelpers.CompileAndCache(isEvenExpr);
			
			var isEven = compiledFunc(4); // true
			var isOdd = compiledFunc(3);  // false

		}

		// 6. Null-safe function wrapper
		public static Func<T, TResult> NullSafe<T, TResult>(Func<T, TResult> func, TResult defaultValue = default ) where T : class
		{
			return arg => arg == null ? defaultValue : func(arg);
		}

		public void Demo()
		{
			Func<string, int> stringLength = str => str.Length;
			var safeLengthFunc = SmartHelpers.NullSafe(stringLength, -1);

			var len1 = safeLengthFunc("test");  // Returns 4
			var len2 = safeLengthFunc(null);    // Returns -1 instead of throwing
		}

		//7. Async function with timeout
		public static async Task<TResult> WithTimeout<TResult>(
			Func<T, TResult> func,
			TimeSpan timeout
		)
		{
			using var cts = new CancellationTokenSource(timeout);
			try
			{
				return await func().WaitAsync(cts.Token);
			}
			catch(OperationCanceledException)
			{
				throw new TimeoutException($"Operation time out after {timeout.TotalSeconds} seconds");
			}
		}
		async Task DemoTimeoutAsync()
        {
            Func<Task<string>> longRunningTask = async () =>
            {
                await Task.Delay(5000); // Simulate long operation
                return "Done!";
            };

            try
            {
                // This will timeout after 2 seconds
                var result = await SmartHelpers.WithTimeout(
                    longRunningTask,
                    TimeSpan.FromSeconds(2)
                );
            }
            catch (TimeoutException ex)
            {
                Console.WriteLine("Operation timed out!");
            }
        }


		// Example combine with UserService
		public class UserService
		{
			private readonly HttpClient _client = new();

			public async Task<UserProfiel> GetUserProfileAsync(int userId)
			{
				// combine meoization, retry, and performance measurement

				Func<int, UserProfile> fetchUser = async id =>
				{
					var response = await _client.GetAsync($"api/users/{id}");
					response.EnsureSuccessStatusCode()
					return await response.Content.ReadFromJsonAsync<UserProfile>();
				};

				var withRetry = SmartHelper.WithRetry(fetchUser);

				var withPerformance = SmartHelper.Measurement(
					withRetry,
					elapsed => Console.WriteLine($"User fetch took {elapsed.TotalMiliseconds}m")
				);

				var resutl = await SmartHelper.WithTimeout(
					() => withPerformance(userId),
					TimeSpan.FromSecond(10)
				);

				return result;
			}
		}
	}
```

## The Result type
When our function needs to represent two states : a happy path and a failure path, we can model it with a generic type `Result<T, E>` where `T` represent
the value and `E` represent the error response. An example function that gets a user could look like this:

```C#
	publuc async Result<User, string> FindByEmailAsync(string email)
	{
		User user = await context.User.FirstOrDefault(u => EF.functions.Like(u.Email, $"%{email}%"));
		if(user is null)
		{
			return "No user found";
		}
		return user;
	}

	[HttpGet("{email}")]
	public async Task<ActionResult<User>> GetByEmail(string email)
	{
		if(string.IsNullOrEmpty(email)) {
			return BadRequest("email cannot be empty");
		}
		Result<User, string> result = await FindByEmail(email);
		return result.Match<ActionResult<User>>(
			user => Ok(user),
			_ => NotFound());
	}
```

Class `Result` type:
```C#
	public readonly struct Result<T, E>
	{
		private readonly bool _success;
		private readonly T Value;
		private readonly E Error;

		private Result<T, E>(T v, E e, bool success)
		{
			Value = v;
			Error = e;
			_success = success;
		}

		public bool isOk() => _success;
		public static Result<T, E> Ok(T v)
		{
			return new(v, default(E), true);
		}
		public static Result<T, E> Err(E e)
		{
			return new(default(T), e, false);
		}

		public static implicit operator Result<T, E>(T v) = new(v, default(E), true);
		public static implicit operator Result<T, E>(E e) = new(default(T), e, false);

		public R Match<R>(
			Func<T, R> success,
			Func<E, R> failure) = > _success ? success(Value) : failure(Error);


	}
```
## Reflection and Attribute combination
`Reflection` allows us to inspect and manipulate objects at runtime. We can use reflection to:

- Inspect Types: Get metadata about types, including properties, methods, fields, and more.
- Create Instances Dynamically: Create objects dynamically using type information obtained through reflection.
- Invoke Methods: Call methods on objects dynamically.
- Access Fields and Properties: Retrieve and set values of fields and properties at runtime.

Example where we use reflection to inspect a class’s metadata.
```C#
	using System;
	using Reflection;

	public class Person
	{
		public string Name {get; set;}
		public int Age {get; set}
		
		public void Introduce()
		{
			Console.WriteLine($"Hi, I'm {Name} and , I'm {Age} years old");
		}
	}
	class Program
	{
		public static void Main(string[] args)
		{
			// Get the type of the class Person
			Type personType = typeof(Person);

			// Get properties of the Person class
			Properties[] properties = personType.GetProperties();
			foreach(var prop in properties)
			{
				Console.WriteLine(prop.Name);
			}

			// Get the method of the Person class
			MethodInfo[] methods = personType.GetMethods();
			foreach(var method in methods)
			{
				Console.WriteLine(method.Name);
			}

			// Create an instance of Person dynamically
			object personInstance = Activator.CreateInstance(personType);
			PropertyInfo nameProperty = personInstance.GetProperty("Name");
			nameProperty.SetValue(personInstance,"Alex");
			// same with other properties

			// Calling the Introduce method dynamically
			MethodInfo introduceMethod = personInstance.GetMethod("Introduce");
			introduceMethod.Invoke(personInstance, null);
		}
	}

```
Create custome `Attributes`, We can define our own custom attributes by inheriting from System.Attribute. Here’s an example of how to create a simple custom attribute:
```C#
	[AttributeUsage(AttributeTargets.Class|AttributeTargets.Method)]
	public class DocumentationAttribute : Attribute
	{
		public string Author {get; set;}
		public string Description {get; set;}

		public DocumentationAttribute(string author, string description)
		{
			Author = author;
			Description = description;
		}
	}

	[Documentation("Alex","This class represents a person")]
	public class Person
	{
		[Documentation("Alex","This method introduce a person")]
		public void Introduce()
		{
			Console.WriteLine("Hello");
		}
	}

```

Now, let’s retrieve and display the custom attribute data using reflection:
```C#
	using System;
	using System.Reflection;
	
	class Program
	{
		public static void Main(string[] args)
		{
			// Get type of the Person class
			Type personType = typeOf(Person);

			// Get custom attributes applied to the Person class
			var classAttributes = personType.GetCustomAttributes(typeOf(DocumentationAttribute), false);
			foreach(DocumentationAttribute arr in classAttributes)
			{
				Console.WriteLine($"Class Author: {attr.Author}, Description: {attr.Description}");
			}

			// Get custom attributes applied to the Introduce method

			MethodInfo methodInfo = personType.GetMethod("Introduce");
			var methodAttributes = methodInfo.GetCustomAttributes(typeOf(DocumentationAttribute), false);
			foreach(DocumentationAttribute attr in methodAttributes)
			{
				Console.WriteLine($"Method Author: {attr.Author}, Description: {attr.Description}");
			}
		}
	}
```
One of the common use cases for reflection and attributes is creating a dynamic system that behaves differently based on the metadata associated with code elements. For example, we might use attributes to mark certain methods for logging, validation, or permission checking.

Let’s look at an example where we use reflection and attributes for method execution tracking:

```C#
	[AtrributeUage(AtrributeTargets.Method)]
	public class LogMethodAttribute : Attribute
	{
		public string Message {get; set;}
		publicc LogMethodAttribute(string message)
		{
			Message = message;
		}
	}

	public class Calculator
	{
		[LogMethod("Adding two numbers")]
		public int Add(int a, int b) => a + b;

		[LogMethod("Subtract two numbers")]
		public int Subtract(int a, int b) => a - b;
	}

	class Program
	{
		public static void Main(string[] args)
		{
			Type calculatorType = typeOf(Calculator);
			MethodInfo[] methodInfos = calculatorType.Getmethods();

			foreach(var method  in methods)
			{
				var logAttribute = method.GetCustomAttribute<LogMethodAttribute>();
				if(logAttribute != null)
				{
					Console.WriteLine($"Logging: {logAttribute.Message}");
					// call the method dynamically

					var instance = Activator.CreateInstance(calculatorType);
					var result = method.Invoke(instance, new object[] {10, 5});

					Console.WriteLine($"Result of {method.Name}: {result}");

				}
			}
		}
	}
```

